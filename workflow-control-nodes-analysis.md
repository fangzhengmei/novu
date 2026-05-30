# Novu 工作流控制节点运行机制分析

> 本文档梳理 Novu 工作流中 **聚合（Digest）**、**限频（Throttle）**、**延迟（Delay）** 三类控制节点的运行机制，涵盖面板配置到运行时调度的完整映射链路、状态机模型、存储与队列依赖、节点连接方式及触发条件满足后的恢复流程。

---

## 一、全局架构概览

### 1.1 核心流程

```
Dashboard 配置 → API 持久化 → Worker 调度执行
     ↓                ↓               ↓
 controlValues    Job 文档(MongoDB)  StandardQueue(BullMQ)
 UiSchema/Schema  digest/delay/step  AddJob → RunJob → SendMessage
```

### 1.2 节点分类

| 类型 | StepTypeEnum | 是否延迟型(Deferred) | 作用 |
|------|-------------|---------------------|------|
| Digest | `DIGEST` | ✅ | 聚合同一窗口内的事件 |
| Throttle | `THROTTLE` | ✅ | 在时间窗口内限制执行次数 |
| Delay | `DELAY` | ✅ | 将后续步骤推迟到指定时间 |
| Trigger / Channel Steps | 其他 | ❌ | 即时执行 |

三类控制节点均被标记为 **Deferred Job**（`add-job.usecase.ts:1149-1167`），意味着它们在入队时会先经过 `executeDeferredJob` 分支处理延迟逻辑，再通过 BullMQ 的 `delay` 选项投递到队列。

---

## 二、面板配置到运行时的映射

### 2.1 配置定义层（Schema & UiSchema）

每种控制节点在 `libs/application-generic/src/schemas/control/` 下有对应的 Zod Schema 和 UiSchema：

| 节点 | Schema 文件 | UiSchema Group | 核心字段 |
|------|-----------|---------------|---------|
| Delay | `delay-control.schema.ts` | `UiSchemaGroupEnum.DELAY` | `type`（regular/timed/dynamic）、`amount`、`unit`、`cron`、`dynamicKey`、`extendToSchedule` |
| Digest | `digest-control.dto.ts` | `UiSchemaGroupEnum.DIGEST` | `type`（regular/timed）、`amount`、`unit`、`cron`、`digestKey`、`lookBackWindow` |
| Throttle | `throttle-control.schema.ts` | `UiSchemaGroupEnum.THROTTLE` | `type`（fixed/dynamic）、`amount`、`unit`、`dynamicKey`、`threshold`、`throttleKey` |

**映射路径**：Dashboard 前端通过 `UiSchema.properties` 中声明的 `UiComponentEnum` 渲染对应的 UI 组件（如 `UiComponentEnum.DELAY_TYPE` → `DelayWindow`），用户填写后生成 `controlValues`，随 Workflow 创建/更新 API 提交到后端，存入 `step.metadata`。

### 2.2 DTO 传输层

API 层使用 Class Validator DTO 进行校验：

- `DigestControlDto`（`libs/application-generic/src/dtos/workflow/controls/digest-control.dto.ts`）
- `ThrottleControlDto`（`libs/application-generic/src/dtos/workflow/controls/throttle-control.dto.ts`）
- Delay 使用 `delayControlZodSchema` 进行 discriminated union 校验

### 2.3 运行时元数据（Job.digest / Job.step.metadata）

当 `CreateNotificationJobs` 为每个 step 创建 Job 时（`create-notification-jobs.usecase.ts:186-212`），Digest 类型的 `step.metadata` 被展开为 Job 的 `digest` 字段：

```
Job {
  digest: {
    type: 'regular' | 'timed' | 'backoff',
    amount: number,
    unit: string,
    digestKey: string,
    digestValue: string,      // 从 payload 解析
    backoff: boolean,
    backoffAmount: number,
    backoffUnit: string,
    timed: { cronExpression, untilDate, atTime, weekDays, monthDays, ordinal, ordinalValue, monthlyType },
    events: []                 // 运行时填充
  },
  step: {
    metadata: { type, amount, unit, dynamicKey, ... }   // Delay/Throttle 配置
  }
}
```

---

## 三、聚合节点（Digest）

### 3.1 状态机

```
PENDING → DELAYED → RUNNING → COMPLETED
  │         │
  │    ┌────┴────┐
  │    ↓         ↓
  │  MERGED    SKIPPED
  │
  └→ CANCELED
```

**状态流转详解**：

1. **PENDING → DELAYED**：`AddJob.executeDeferredJob()` 检测到 `job.type === DIGEST`，调用 `handleDigest()`，经 `MergeOrCreateDigest` 决策后，若创建新 Digest 则设置 `status=DELAYED`，计算等待时长后入队。
2. **PENDING → MERGED**：若已有相同 `digestKey+digestValue` 的 DELAYED Digest Job 存在，当前 Job 被合并（`status=MERGED`，`_mergedDigestId` 指向主 Job），子 Job 也全部标记 MERGED。
3. **PENDING → SKIPPED**：Backoff 模式下，若当前 Job 不是最早的触发事件，则被跳过。
4. **DELAYED → RUNNING → COMPLETED**：BullMQ 延迟到期后，`RunJob` 取出 Job，调用 `SendMessage → Digest.execute()`，收集聚合事件后返回 SUCCESS。

### 3.2 合并决策逻辑（MergeOrCreateDigest）

位于 `merge-or-create-digest.usecase.ts:150-217`：

```
computeDigestLogicBasedOnExistingDigestState()
  │
  ├─ isBackOffDigestType? ───→ backoffLogic()
  │                              │
  │                              ├─ 无更早的同 Key Job → SKIPPED
  │                              └─ 有更早的同 Key Job → isMasterDigestOrShouldMergeToExisting()
  │
  └─ 非 Backoff ──→ isMasterDigestOrShouldMergeToExisting()
                       │
                       ├─ 无已 DELAYED 的同 digestValue Job → CREATED（标记为 Digest Master）
                       └─ 有已 DELAYED 的同 digestValue Job → MERGED
```

**Backoff 模式**额外逻辑：先查找 backoff 时间窗口内是否有更早的同 Key Job，若有则当前 Job 被跳过；否则继续走常规合并判定。

### 3.3 存储依赖

| 存储 | 用途 | 关键字段 |
|------|------|---------|
| MongoDB `jobs` 集合 | Job 持久化、状态流转、合并关系 | `_mergedDigestId`、`digest.digestKey`、`digest.digestValue`、`status` |
| MongoDB `notifications` 集合 | 记录被聚合的 Notification | `_digestedNotificationId` |
| MongoDB `step_runs` 集合 | 步骤运行记录 | `status` |
| BullMQ `StandardQueue` | 延迟调度 | `delay` 选项控制投递时间 |

**唯一性保证**：`jobs` 集合上有复合唯一索引（`job.schema.ts:422-443`），防止同一 `digestKey + digestValue + workflow + subscriber` 出现两个 DELAYED 的 Master Digest Job。

### 3.4 事件收集与恢复流程

当 Delayed Digest Job 到期被执行时（`RunJob`），`SendMessage` 路由到 `Digest.execute()`：

1. **获取当前 Job**：从 MongoDB 读取
2. **收集聚合事件**：
   - **新路径**（`IS_USE_MERGED_DIGEST_ID_ENABLED` 特性开关）：查询所有 `_mergedDigestId === currentJob._id` 且 `status=MERGED` 的 Job，取其 payload
   - **旧路径**：
     - Regular 模式：`GetDigestEventsRegular` → 从当前 Job 创建时间回溯 `amount+unit` 时间段，查找所有同 digestKey/digestValue 的 Trigger Job
     - Backoff 模式：`GetDigestEventsBackoff` → 从当前 Job 创建时间起查找所有已完成的 Trigger Job
3. **写入 events**：将收集到的 events 数组写入当前 Job 及关联 Job 的 `digest.events` 字段
4. **返回 SUCCESS**：`RunJob` 标记 COMPLETED，继续调用 `tryQueueNextJobs()` 调度下游节点

### 3.5 取消与容错

- 若 DELAYED 的 Digest Master Job 被取消，`RunJob.delayedEventIsCanceled()` 会查找同一 digestKey/digestValue 下是否有其他 MERGED 状态的 DELAYED Job（即 "active digest follower"），若有则将其提升为新的执行者继续流程
- 取消操作在 `cancel-delayed.usecase.ts` 中处理

---

## 四、限频节点（Throttle）

### 4.1 状态机

```
PENDING → [Redis 限频判定]
              │
              ├─ granted=true  → DELAYED(delay=0) → RUNNING → COMPLETED
              └─ granted=false → SKIPPED（含所有子 Job）→ 流程终止
```

**注意**：Throttle 虽然是 Deferred 类型，但其核心决策在 `AddJob.executeDeferredJob()` 阶段通过 Redis 完成，**不经过 BullMQ 业务延迟**——通过限频的 Job 以 `delay=0` 入队立即执行；未通过的 Job 直接 SKIPPED，**不会延迟重试，也不会在窗口恢复后自动重新触发**。

### 4.2 限频判定逻辑（AddJob.handleThrottle）

位于 `add-job.usecase.ts:815-912`：

```
handleThrottle()
  │
  ├─ 解析 throttleConfig（Bridge 响应或 step metadata）
  │   ├─ type=fixed → DurationUtils.convertToMilliseconds(amount, unit) 计算 windowMs
  │   └─ type=dynamic → parseDynamicDurationValue() 从 payload 解析 windowMs
  │
  ├─ validateThrottleWindow() → 动态类型校验时长合法性
  │
  ├─ 解析 throttleValue（从 payload 按 throttleKey 提取，默认 'default'）
  │
  └─ redisThrottleService.reserveThrottleSlot()
      │
      ├─ Redis Lua 脚本原子判定
      │   ├─ SCARD < count → SADD + 返回 granted=1
      │   └─ SCARD >= limit → 返回 granted=0, count, ttl
      │
      ├─ granted=true → shouldSkip=false → Job 正常入队
      └─ granted=false → shouldSkip=true → handleThrottleSkip()
```

### 4.3 Redis 限频存储

**Redis Key 结构**：
```
throttle:{environmentId}:{subscriberId}:{workflowId}:{stepId}[:{throttleKey}:{throttleValue}]:set
```

**数据结构**：Redis Set，每个成员为一个 `jobId:timestamp`

**Lua 脚本逻辑**（`redis-throttle.service.ts:14-61`）：
1. 检查 Key 的 TTL，若已过期或无 TTL 则清理
2. `SCARD` 获取当前 Set 大小
3. 若 `count >= limit`，返回 `{granted=0, count, ttl}`
4. `SADD` 添加 jobId
5. 若是第一个成员，设置 `EXPIRE`（`windowMs + TTL_BUFFER_MS`，默认 30s）
6. 添加后若 `count > limit`，回滚 SREM
7. 返回 `{granted=1, count, ttl}`

**TTL 计算**：`Math.ceil((windowMs + 30000) / 1000)` 秒

### 4.4 限频跳过处理（handleThrottleSkip）

位于 `add-job.usecase.ts:1004-1051`：

1. 当前 Job 标记 `status=SKIPPED`，`stepOutput` 写入限频详情
2. 创建 `stepRun` 记录（SKIPPED）
3. 所有子 Job（`_parentId === job._id`）也标记 SKIPPED
4. 写入执行详情 `DetailEnum.THROTTLE_LIMIT_EXCEEDED`

### 4.5 Throttle 运行时（SendMessage.Throttle）

位于 `throttle.usecase.ts`：逻辑极简，仅返回 `{ status: SUCCESS }`。因为实际的限频判定和过滤已在 `AddJob` 阶段完成，到达 `SendMessage` 时意味着已获得执行配额。

### 4.6 配置映射

| 面板字段 | controlValues Key | 运行时用途 |
|---------|------------------|-----------|
| Throttle Type | `type` | fixed / dynamic 分支 |
| Window | `amount` + `unit` | fixed 模式计算 windowMs |
| Dynamic Key | `dynamicKey` | dynamic 模式从 payload 解析 windowMs |
| Threshold | `threshold` | Redis 限频 limit |
| Throttle Key | `throttleKey` | 分组限频的 Key 路径 |

### 4.7 Redis 窗口 TTL 到期后的配额恢复机制

Redis Throttle 的配额恢复完全依赖 **Redis Key 的 TTL 自动过期机制**，而非业务层的延迟重试。以下是完整的代码级分析：

#### 4.7.1 Lua 脚本中的 TTL 检查与清理

`redis-throttle.service.ts:25-33` 的 Lua 脚本在每次 `reserveThrottleSlot` 调用时都会执行前置 TTL 检查：

```lua
-- Manual TTL check: if key exists but has expired, clean it up
local currentTtl = redis.call('TTL', setKey)
if currentTtl == 0 then
  -- Key exists but has no TTL (should not happen) or has expired
  redis.call('DEL', setKey)
elseif currentTtl == -1 then
  -- Key exists but has no expiry set (should not happen with our logic)
  redis.call('DEL', setKey)
end
```

**触发清理的两种异常场景**：
- `currentTtl == 0`：Key 已过期但 Redis 尚未惰性删除（Redis 的 TTL 过期删除是惰性 + 定期采样）
- `currentTtl == -1`：Key 存在但未设置过期时间（防御性检查，正常逻辑不会出现）

#### 4.7.2 TTL 设置时机与窗口起点

`redis-throttle.service.ts:49-51` 中，TTL 仅在 **第一个成员加入时设置**：

```lua
count = count + 1
if count == 1 then
  redis.call('EXPIRE', setKey, ttlSec)
end
```

这意味着：
- **窗口起点 = 第一个成功获取配额的事件的时间戳**
- TTL 计算公式（`redis-throttle.service.ts:112-114`）：
  ```typescript
  private computeTtlSeconds(windowMs: number): number {
    return Math.ceil((windowMs + this.ttlBufferMs) / 1000);
  }
  ```
  其中 `ttlBufferMs` 默认 30s（环境变量 `THROTTLE_REDIS_TTL_BUFFER_MS`），确保窗口时间内的所有事件都被正确计数，避免边界 race condition。

#### 4.7.3 TTL 到期后的新事件处理流程

```
新事件到达 → AddJob.executeDeferredJob() → handleThrottle() → reserveThrottleSlot()
     │
     ├─ Lua 脚本执行：
     │   1. TTL 检查：若 Key 已过期（TTL=0 或 -1），DEL 清理
     │   2. SCARD 计数：Key 不存在或已删除 → 返回 0
     │   3. 0 < threshold → 允许通过
     │   4. SADD 添加 jobId
     │   5. count == 1 → 设置新的 EXPIRE（窗口重置）
     │   6. 返回 granted=1
     │
     ├─ shouldSkip=false → 继续 executeDeferredJob 流程
     │   ├─ delay = getExecutionDelayAmount() → Throttle 非 Digest/Delay，所以 delay=0
     │   ├─ stepRun 标记 DELAYED
     │   └─ queueJob({ delay: 0 }) → BullMQ 立即投递
     │
     └─ 后续：StandardWorker 消费 → RunJob → SendMessage.Throttle → SUCCESS → tryQueueNextJobs()
```

**关键点**：
- TTL 到期后，**新窗口的起点是新事件到达的时间**，而非上一个窗口的结束时间
- 配额恢复对**新到达的事件**有效，已被 SKIPPED 的旧事件不会被回溯

### 4.8 为何被限频跳过的父子 Job 不会自动恢复

#### 4.8.1 handleThrottleSkip 的代码逻辑

`add-job.usecase.ts:1004-1051` 中 `handleThrottleSkip` 的完整执行流程：

```typescript
private async handleThrottleSkip(
  command: AddJobCommand,
  job: JobEntity,
  throttleResult: { ... }
) {
  // 1. 当前 Throttle Job 标记为 SKIPPED（终态）
  await this.jobRepository.updateOne(
    { _id: job._id, _environmentId: command.environmentId },
    {
      $set: {
        status: JobStatusEnum.SKIPPED,
        stepOutput: {
          throttled: true,
          executionCount: throttleResult.executionCount,
          threshold: throttleResult.threshold,
          throttledUntil: throttleResult.throttledUntil,
        },
      },
    }
  );

  // 2. 创建 stepRun 记录（SKIPPED）
  await this.stepRunRepository.create(job, { status: JobStatusEnum.SKIPPED });

  // 3. 级联标记所有子 Job 为 SKIPPED
  const childJobsUpdated = await this.jobRepository.updateAllChildJobStatus(
    job,
    JobStatusEnum.SKIPPED,
    job._id
  );

  // 4. 批量创建子 Job 的 stepRun
  if (childJobsUpdated.length > 0) {
    await this.stepRunRepository.createMany(childJobsUpdated, { status: JobStatusEnum.SKIPPED });

    // 5. 写入执行详情（记录限频事件，便于排查）
    await this.createExecutionDetails.execute(
      CreateExecutionDetailsCommand.create({
        ...CreateExecutionDetailsCommand.getDetailsFromJob(job),
        detail: DetailEnum.THROTTLE_LIMIT_EXCEEDED,
        source: ExecutionDetailsSourceEnum.INTERNAL,
        status: ExecutionDetailsStatusEnum.SUCCESS,
        isTest: false,
        isRetry: false,
        raw: JSON.stringify({ ...throttleResult }),
      })
    );
  }
}
```

#### 4.8.2 不自动恢复的三个代码层面原因

**原因 1：SKIPPED 是终态（Terminal State），非中间态**

在 `executeDeferredJob` 中（`add-job.usecase.ts:293-298`），`handleThrottleSkip` 后直接返回，**不会调用 `queueJob()` 投递到队列**：

```typescript
if (throttleResult.shouldSkip) {
  await this.handleThrottleSkip(...);

  return {
    workflowStatus: WorkflowRunStatusEnum.COMPLETED,
    deliveryLifecycleStatus: DeliveryLifecycleStatusEnum.SKIPPED,
  };
}
```

对比 Digest/Delay 的 DELAYED 路径（`add-job.usecase.ts:361-365`）：
```typescript
await this.stepRunRepository.create(updatedJob, { status: JobStatusEnum.DELAYED });
await this.queueJob({ job, delay, untilDate: bridgeDelayAmountDate, timezone: subscriber?.timezone });
```

**原因 2：没有"重试/恢复 Job"的调度机制**

系统中**没有任何组件**会：
- 监听 Redis Key 过期事件
- 扫描 SKIPPED 状态的 Throttle Job
- 在窗口恢复后重新触发被跳过的 Job

BullMQ 的延迟调度（`delay` 选项）只用于 Digest/Delay，Throttle 未通过时不会设置任何延迟。

**原因 3：父子链已被中断**

`updateAllChildJobStatus(job, JobStatusEnum.SKIPPED, job._id)` 将所有下游子 Job 也标记为 SKIPPED。即使后续想恢复，也无法通过 `tryQueueNextJobs()` 找到下一个可执行的 Job（因为子 Job 已处于终态）。

#### 4.8.3 设计意图

Throttle 的设计目标是**保护下游系统不被过载**，而非**保证每个事件都必须送达**。如果在窗口恢复后自动重试被跳过的事件，可能导致：
1. 流量突增（所有被跳过的事件同时重试）
2. 再次触发限频
3. 违反"保护下游"的初衷

如果业务需要保证送达，应使用 Digest（聚合）而非 Throttle（限流）。

### 4.9 与 Digest/Delay 到期调度的本质差异

#### 4.9.1 三节点 AddJob 阶段决策对比

| 决策维度 | Digest | Delay | Throttle |
|---------|--------|-------|----------|
| 决策阶段 | AddJob.executeDeferredJob | AddJob.executeDeferredJob | AddJob.executeDeferredJob |
| 决策依据 | MongoDB 查询同 Key 的 DELAYED Job | metadata 配置计算 delayMs | Redis Set 计数 + TTL |
| 决策结果 | DELAYED / MERGED / SKIPPED | DELAYED | 通过: 继续<br>不通过: SKIPPED |
| 是否调用 queueJob() | ✅ DELAYED 分支调用 | ✅ 调用 | ✅ 通过时调用（delay=0）<br>❌ 不通过时不调用 |
| BullMQ delay 值 | digestAmount（>0） | delayAmount（>0） | 0（立即执行） |
| Job 终态设置 | DELAYED（中间态） | DELAYED（中间态） | SKIPPED（终态） |

#### 4.9.2 到期/恢复时的调度链路对比

**Digest/Delay 到期调度链路（自动恢复）**：
```
BullMQ 内部调度（delay 到期）
    ↓
StandardWorker.getWorkerProcessor() 消费
    ↓
RunJob.execute()
    ├─ delayedEventIsCanceled() 检查
    ├─ Subscriber Schedule 检查（可重新入队）
    ├─ SendMessage 执行（Digest 收集 events / Delay 仅记录）
    ├─ 标记 COMPLETED
    └─ tryQueueNextJobs() → 查找 _parentId = 当前 JobId 的下一个 Job → AddJob.execute()
```

**Throttle 不通过链路（无自动恢复）**：
```
AddJob.executeDeferredJob()
    ↓
handleThrottle() → Redis 判定 granted=false
    ↓
handleThrottleSkip()
    ├─ 更新当前 Job.status = SKIPPED
    ├─ updateAllChildJobStatus() → 所有子 Job.status = SKIPPED
    └─ return { workflowStatus: COMPLETED, deliveryLifecycleStatus: SKIPPED }
    ↓
流程终止。不会投递到 BullMQ，没有后续调度。
新事件到达时重新走完整链路，与已 SKIPPED 的旧 Job 无关。
```

#### 4.9.3 状态机性质差异

| 节点 | 状态 | 性质 | 后续动作 |
|------|------|------|---------|
| Digest | DELAYED | 中间态 | BullMQ 到期自动调度 |
| Delay | DELAYED | 中间态 | BullMQ 到期自动调度 |
| Throttle（通过） | DELAYED（delay=0） | 伪中间态 | BullMQ 立即调度 |
| Throttle（不通过） | SKIPPED | 终态 | 无任何后续动作 |

**核心区别**：DELAYED 状态意味着"等待条件满足后继续执行"，而 SKIPPED 意味着"放弃执行，流程到此为止"。BullMQ 只会为 DELAYED 状态的 Job 提供自动调度能力。

### 4.10 限频执行语义的代码级深入分析

#### 4.10.1 配置异常时为何直接放行而非限流失败

在 `handleThrottle()`（`add-job.usecase.ts:815-857`）中，**所有配置解析异常分支都返回 `shouldSkip: false`**，即直接放行而非限频失败。代码中共有 5 处异常退出点：

| 异常场景 | 代码位置 | 处理方式 |
|---------|---------|---------|
| Fixed 类型缺少 amount/unit | `add-job.usecase.ts:828-831` | `logger.warn()` + `return { shouldSkip: false }` |
| Fixed 类型 unit 非法（DurationUtils 抛错） | `add-job.usecase.ts:835-838` | `logger.warn()` + `return { shouldSkip: false }` |
| Dynamic 类型缺少 dynamicKey | `add-job.usecase.ts:841-844` | `logger.warn()` + `return { shouldSkip: false }` |
| Dynamic 类型解析失败（parseDynamicDurationValue 返回 null） | `add-job.usecase.ts:847-851` | `logger.warn()` + `return { shouldSkip: false }` |
| 未知 type（非 fixed/dynamic） | `add-job.usecase.ts:854-857` | `logger.warn()` + `return { shouldSkip: false }` |

**代码示例**：
```typescript
if (type === 'fixed') {
  const { amount, unit } = throttleConfig;
  if (!amount || !unit) {
    this.logger.warn(`Fixed throttle configuration missing amount or unit for job ${job._id}`);
    return { shouldSkip: false };  // 直接放行
  }
}
```

**设计意图分析**：
1. **Fail-open 而非 Fail-close**：Throttle 的设计哲学是"最好的情况是限频，最坏的情况是不限频"，而非因为配置错误导致业务完全不可用。
2. **区别于 Delay 的异常处理**：Delay 类型在动态延迟解析失败时会 `throw error`（`compute-job-wait-duration.service.ts`）并走 `handleStepValidationError` 标记 FAILED，因为 Delay 是"保证送达"型节点，错误配置导致延迟不成立时应失败而非跳过。
3. **监控兜底**：仅打 `logger.warn()`，依赖日志监控发现配置问题，而不是阻塞业务流程。

#### 4.10.2 throttledUntil 与 Redis TTL 缓冲不一致的原因

**现象**：
- `throttledUntil` 计算：`new Date(reservationResult.windowStartMs + windowMs).toISOString()`（`add-job.usecase.ts:901, 910`）
- Redis 实际 TTL：`Math.ceil((windowMs + ttlBufferMs) / 1000)` 秒（`redis-throttle.service.ts:112-114`），其中 `ttlBufferMs` 默认 30s

即：`Redis TTL = windowMs + 30s`，而 `throttledUntil = windowStartMs + windowMs`，两者相差约 30 秒。

**代码层面的不一致原因**：

1. **用途不同**：
   - `throttledUntil` 是**用户可见的提示信息**，写入 `Job.stepOutput.throttledUntil`，在 Activity Feed 中展示给用户，告知"限频到 X 时间为止"
   - Redis TTL 是**内部一致性保障**，确保窗口内的所有事件都被正确计数，防止边界 race condition

2. **窗口起点的注释说明**：
   ```typescript
   // redis-throttle.service.ts:201
   windowStartMs: params.nowMs, // For sliding windows, window starts when first request arrives
   ```
   代码注释表明 `windowStartMs` 是"窗口起点"，但实际上**固定窗口的起点是第一个成功获取配额的事件的时间**，而非每个事件的 `nowMs`。这导致：
   - 当窗口已有配额被占用时，`windowStartMs = 当前事件的 nowMs`（并非真实窗口起点）
   - `throttledUntil = windowStartMs + windowMs` 会比真实的窗口结束时间晚

3. **Buffer 存在的原因**：
   - 防止 Redis TTL 过期与新事件到达之间的边界 race condition
   - 防止时钟漂移导致窗口提前结束
   - Lua 脚本中已有前置 TTL 检查（`TTL == 0 或 -1 时 DEL`），所以 buffer 不会导致窗口延长

**潜在的用户体验问题**：
- 用户看到的 `throttledUntil` 可能与实际可重新获取配额的时间不一致
- 如果配置了 1 分钟窗口，用户看到 `throttledUntil` 是 1 分钟后，但 Redis 实际要 1 分 30 秒后才会重置

##### 4.10.2.1 throttledUntil 与 Redis 实际配额恢复时刻的精确时序关系

让我们通过一个**精确的时序示例**来理解两者的差异：

**场景**：配置 `windowMs = 60000ms`（1 分钟），`threshold = 1`（每窗口只允许 1 条）

```
时间轴 (ms):
t=1000      [事件 E1 到达] → 成功获取配额，Redis Set Key 创建，TTL=ceil((60000+30000)/1000)=90s
            │
            ├─ Lua 执行: count=0 < 1 → granted=1, SADD, count==1 → EXPIRE 90
            └─ Redis Key TTL 到期时间 = t=1000 + 90000 = t=91000

t=2000      [事件 E2 到达] → 被拒绝
            │
            ├─ params.nowMs = 2000 (在 add-job.usecase.ts:859 调用 Date.now())
            ├─ reservationResult.windowStartMs = 2000 (这是 E2 的到达时间，不是窗口起点!)
            ├─ throttledUntil = 2000 + 60000 = t=62000 (显示给用户)
            └─ 实际 Redis 配额恢复时刻 = t=91000
               ↑
               差异: 91000 - 62000 = 29000ms (≈ 30s buffer + E1/E2 时间差)

t=62000     [用户看到 throttledUntil 已过，以为配额恢复了]
            [事件 E3 到达] → 仍被拒绝 (Redis Key 还需 29s 才到期)

t=91000     [Redis Key TTL 到期，真正可重新获取配额]
            [事件 E4 到达] → Lua TTL 检查发现 TTL<=0 → DEL Key → 重新计数
            └─ 新窗口起点 = t=91000
```

**精确数学关系**：

| 变量 | 计算方式 |
|------|---------|
| 窗口真实起点 `W₀` | 第一个成功事件的到达时间（Lua 设置 TTL 的时刻） |
| 用户可见 `throttledUntil` | `E_arrivalTime + windowMs`（当前被拒绝事件的到达时间 + 窗口大小） |
| 实际配额恢复时刻 `T_actual` | `W₀ + windowMs + bufferMs` |
| 展示误差 `Δ` | `T_actual - throttledUntil = (W₀ - E_arrivalTime) + bufferMs` |

**误差范围**：
- 最小误差：`bufferMs`（事件刚好在窗口起点之后到达，`W₀ ≈ E_arrivalTime`）
- 最大误差：`windowMs + bufferMs`（事件刚好在上一个窗口起点之后到达）

##### 4.10.2.2 为什么不使用 Redis 返回的 ttlSecRemaining 计算 throttledUntil？

`redis-throttle.service.ts:200` 实际上从 Redis 获取了 `ttlSecRemaining`：
```typescript
ttlMs: ttlSecRemaining > 0 ? ttlSecRemaining * 1000 : 0,
```

但 `add-job.usecase.ts:901` 并没有使用这个值，而是使用了 `windowStartMs + windowMs`。这可能是一个**历史遗留问题**或**设计选择**：
- 使用 `windowStartMs + windowMs` 给用户的感觉是"限频到 X 时间为止"，是一个绝对时间点
- 使用 `ttlMs` 会给用户"还需要等待 X 毫秒"，是一个相对时长
- 从用户体验角度，绝对时间点可能更友好，但牺牲了精确性

**另外一个隐藏问题**：即使使用 `ttlSecRemaining` 计算，也仍然有 `bufferMs` 的差异，因为 TTL 包含了 30s buffer。

---

#### 4.10.3 SKIPPED 子链写入 _mergedDigestId 对 delivery lifecycle 判定优先级的影响

`updateAllChildJobStatus()`（`job.repository.ts:262-301`）在更新子 Job 状态时，会**同时写入 `_mergedDigestId: activeDigestId`**：

```typescript
async updateAllChildJobStatus(job: JobEntity, status: JobStatusEnum, activeDigestId: string): Promise<JobEntity[]> {
  // ...
  {
    $set: {
      status,
      _mergedDigestId: activeDigestId,  // 无论 status 是 MERGED 还是 SKIPPED，都会写入
    },
  }
  // ... 循环遍历整条子链
}
```

这个方法被**三处调用**，传入的 `activeDigestId` 含义不同：

| 调用方 | 传入的 status | 传入的 activeDigestId | 含义 |
|--------|-------------|----------------------|------|
| `processMergedDigest`（merge-or-create-digest.usecase.ts:87） | `MERGED` | Master Digest Job 的 _id | 子 Job 被合并到主 Digest Job |
| `handleThrottleSkip`（add-job.usecase.ts:1032） | `SKIPPED` | 当前 Throttle Job 的 _id | 子 Job 因限频被跳过 |
| `handleDigestSkip` 无直接调用，但 Digest Backoff 跳过也会走级联 | 各种 | 不同 |

**对 Delivery Lifecycle 判定的影响**：

在 `WorkflowRunService.buildDeliveryLifecycle()`（`workflow-run.service.ts:841-1017`）中，SKIPPED 和 MERGED 的判定优先级如下（从高到低）：

```
Priority 4: SKIPPED
  ↓ 条件：
  - allStepsFinished（所有 channel job 都是终态）
  - skippedJobs = channelJobs 中 deliveryLifecycleState.status='skipped' 且 !_mergedDigestId
  - skippedJobs.length > 0

Priority 7: MERGED
  ↓ 条件：
  - allStepsMerged = 所有 channel job 都是 MERGED 或 (SKIPPED 且 !!_mergedDigestId)
```

**关键优先级逻辑**（`workflow-run.service.ts:914-917`）：
```typescript
const skippedJobs = channelJobs.filter(
  (job) =>
    job.deliveryLifecycleState?.status && job.deliveryLifecycleState.status === 'skipped' && !job._mergedDigestId
);
```

**判定优先级的代码路径**：
1. **Priority 4（SKIPPED）先于 Priority 7（MERGED）执行**
2. SKIPPED 判定会**排除** `_mergedDigestId` 存在的 Job（即 Throttle SKIPPED 的子链不会被算作 SKIPPED）
3. 如果所有 channel job 都是 `SKIPPED 且有 _mergedDigestId`，则不满足 Priority 4，继续向下走
4. Priority 7 判定 `allStepsMerged`：`MERGED || (SKIPPED && _mergedDigestId)` → 返回 MERGED

**这意味着**：
- Throttle 限频跳过的子 Job 链（`status=SKIPPED, _mergedDigestId=throttleJobId`）在 delivery lifecycle 中会被判定为 **MERGED**，而非 SKIPPED
- 只有真正因条件过滤、用户偏好等原因跳过且 `_mergedDigestId=null` 的 Job，才会被判定为 SKIPPED

**设计意图**：
- `_mergedDigestId` 在这里被用作"跳过原因"的隐式标记
- Digest 合并的跳过 → 最终会由 Master Digest Job 送达 → 判定为 MERGED
- Throttle 限频的跳过 → 不会送达 → 但因代码复用 `updateAllChildJobStatus` 也被写入了 `_mergedDigestId`，导致判定为 MERGED
- 这可能是一个**设计上的不一致**：Throttle 限频跳过不应被标记为 MERGED，但因为复用了 `updateAllChildJobStatus` 方法而意外获得了 `_mergedDigestId`

**三种调用场景下的 delivery lifecycle 判定**：

| 场景 | Job.status | _mergedDigestId | deliveryLifecycle 判定 |
|------|-----------|-----------------|----------------------|
| Digest 合并子 Job | MERGED | masterDigestJobId | MERGED（Priority 7） |
| Throttle 限频子 Job | SKIPPED | throttleJobId | MERGED（Priority 7，因 _mergedDigestId 存在） |
| 条件过滤子 Job | SKIPPED | null | SKIPPED（Priority 4） |

---

#### 4.10.4 Mixed Channel 状态下 SKIPPED/MERGED/PENDING 最终分类判定表

当工作流有多个 Channel Step（如 Email + SMS），且各步骤状态不一致时，`buildDeliveryLifecycle()` 会按照优先级链进行判定。以下是**完整的 mixed 状态判定表**（仅包含 SKIPPED、MERGED、PENDING 三种结果的触发条件）：

##### 4.10.4.1 判定表（按优先级从高到低）

> **注意**：Priority 1-3（INTERACTED/DELIVERED/SENT）优先级更高，只要满足就不会走到下面的判定。本表聚焦 SKIPPED/MERGED/PENDING，因此假设不满足 Priority 1-3。

| 最终结果 | 判定优先级 | 触发条件（最小集合） | channelJobs 状态组合示例 |
|---------|-----------|---------------------|-------------------------|
| **SKIPPED** | Priority 4 | 1. 所有 channel job 都是终态（COMPLETED/FAILED/CANCELED/MERGED/SKIPPED）<br>2. 至少 1 个 channel job 的 `deliveryLifecycleState.status='skipped'` **且** `_mergedDigestId=null` | [SKIPPED(null), COMPLETED]<br>[SKIPPED(null), MERGED]<br>[SKIPPED(null), FAILED]<br>[SKIPPED(null), SKIPPED(id)] |
| **MERGED** | Priority 7 | 1. 不满足 Priority 4、5、6<br>2. 所有 channel job 满足：<br>`status=MERGED` **或** (`status=SKIPPED` **且** `_mergedDigestId != null`) | [MERGED, MERGED]<br>[MERGED, SKIPPED(id)]<br>[SKIPPED(id), SKIPPED(id)] |
| **PENDING** | Priority 8 | 1. 不满足 Priority 1-7<br>2. 任一 channel job 状态为：PENDING / QUEUED / RUNNING / DELAYED | [PENDING, COMPLETED]<br>[DELAYED, SKIPPED(null)]<br>[RUNNING, MERGED]<br>[QUEUED, SKIPPED(id)] |

##### 4.10.4.2 触发各分支的最小条件集合

**✅ 触发 SKIPPED（Priority 4）的最小条件**：
```
必须同时满足：
  [A] 所有 channel jobs ∈ {COMPLETED, FAILED, CANCELED, MERGED, SKIPPED}
  [B] ∃ 至少 1 个 job:
        job.deliveryLifecycleState?.status === 'skipped'
        AND job._mergedDigestId == null
  [C] 不满足 Priority 1-3（无 INTERACTED/DELIVERED/SENT）

最小反例（不触发）：
  [SKIPPED(id)] → 不满足 [B]（有 _mergedDigestId）→ 走到 Priority 7 → MERGED
  [SKIPPED(null), PENDING] → 不满足 [A]（有非终态）→ 走到 Priority 8 → PENDING
```

**✅ 触发 MERGED（Priority 7）的最小条件**：
```
必须同时满足：
  [A] 不满足 Priority 1-6
  [B] ∀ channel jobs:
        job.status === MERGED
        OR (job.status === SKIPPED AND job._mergedDigestId != null)

最小正例：
  [MERGED] → MERGED
  [SKIPPED(throttleJobId)] → MERGED （这是 Throttle 限频跳过的典型场景）

最小反例（不触发）：
  [MERGED, SKIPPED(null)] → 不满足 [B]（有 SKIPPED 无 _mergedDigestId）→ Priority 4 → SKIPPED
  [MERGED, COMPLETED] → 不满足 [B]（有 COMPLETED）→ 走到 Fallback → ERRORED
```

**✅ 触发 PENDING（Priority 8）的最小条件**：
```
必须同时满足：
  [A] 不满足 Priority 1-7
  [B] ∃ 至少 1 个 job:
        job.status ∈ {PENDING, QUEUED, RUNNING, DELAYED}

最小正例：
  [PENDING] → PENDING
  [DELAYED, SKIPPED(null)] → PENDING

最小反例（不触发）：
  [PENDING, MERGED] → 不满足 [A]（有 PENDING，也不满足 Priority 7 的"所有 job"条件）→ 仍走到 Priority 8 → PENDING
  [COMPLETED, FAILED] → 不满足 [B]（无进行中状态）→ 不满足 Priority 1-7 → Fallback ERRORED
```

##### 4.10.4.3 复杂 mixed 状态判定示例

| channelJobs 状态组合 | 判定过程 | 最终结果 |
|---------------------|---------|---------|
| [SKIPPED(null), MERGED, SKIPPED(id)] | Priority 4: all终态=true, 有 SKIPPED(null) → 满足 | **SKIPPED** |
| [SKIPPED(id), SKIPPED(id), DELAYED] | Priority 4: all终态=false（有 DELAYED）<br>Priority 8: 有 DELAYED → 满足 | **PENDING** |
| [MERGED, COMPLETED] | Priority 4: 无 SKIPPED(null)<br>Priority 5: 无 CANCELED<br>Priority 6: 非 all FAILED<br>Priority 7: COMPLETED 不满足 `MERGED || SKIPPED(id)`<br>Priority 8: 无进行中<br>→ Fallback | **ERRORED** |
| [SKIPPED(id), PENDING, SKIPPED(id)] | Priority 4: all终态=false（有 PENDING）<br>Priority 8: 有 PENDING → 满足 | **PENDING** |
| [FAILED, SKIPPED(null)] | Priority 4: all终态=true, 有 SKIPPED(null) → 满足 | **SKIPPED** |
| [MERGED, SKIPPED(id)] | Priority 7: 所有都满足 `MERGED || SKIPPED(id)` → 满足 | **MERGED** |
| [SKIPPED(id), SKIPPED(id), SKIPPED(id)] | Priority 7: 所有都是 SKIPPED(id) → 满足 | **MERGED**（Throttle 全限频场景） |

##### 4.10.4.4 Throttle 限频场景下的典型判定链

**场景**：工作流有 Email 和 SMS 两个 Channel Step，均被 Throttle 限频

```
Email Job: status=SKIPPED, _mergedDigestId=throttleJobId
SMS Job:   status=SKIPPED, _mergedDigestId=throttleJobId

判定过程:
  Priority 1-3: 无 message，不满足
  Priority 4: all终态=true ✓, 但所有 SKIPPED 都有 _mergedDigestId → 无 SKIPPED(null) → 不满足
  Priority 5: 无 CANCELED → 不满足
  Priority 6: 非 all FAILED → 不满足
  Priority 7: 所有都是 SKIPPED(id) → 满足 ✓

最终结果: MERGED
```

**用户视角的误导**：用户在 Activity Feed 中看到的是"已聚合（MERGED）"，但实际上两个 Channel 都被限频跳过，**没有任何消息会被发送**。这是一个潜在的 UX 问题。

---

## 五、延迟节点（Delay）

### 5.1 状态机

```
PENDING → DELAYED → RUNNING → COMPLETED
  │                   │
  └→ CANCELED         └→ FAILED（动态延迟配置错误）
```

### 5.2 延迟类型与计算

Delay 有三种类型，由 `ComputeJobWaitDurationService.calculateDelay()` 统一计算等待毫秒数：

| 类型 | metadata.type | 计算方式 |
|------|-------------|---------|
| Regular | `DelayTypeEnum.REGULAR` | `amount × unit` 换算为毫秒 |
| Timed | `DelayTypeEnum.TIMED` | `TimedDigestDelayService.calculate()` 基于 RRule 计算下次 cron 触发时间 |
| Dynamic | `DelayTypeEnum.DYNAMIC` | 从 payload 解析 ISO-8601 时间戳或 `{amount, unit}` 对象 |

**Timed 类型详细计算**（`timed-digest-delay.service.ts`）：
- 使用 `rrule` 库根据 `atTime`、`weekDays`、`monthDays`、`ordinal`/`ordinalValue` 等参数构建 RRule
- 支持时区（`timezone`），通过 `date-fns-tz` 进行时区转换
- 计算当前时间到下次 cron 触发时间的毫秒差

**Dynamic 类型详细计算**（`compute-job-wait-duration.service.ts:52-90`）：
- 从 `payload` 中按 `dynamicKey` 路径提取值
- 若为 ISO-8601 字符串 → 解析为时间戳，计算与当前时间的差值
- 若为 `{ amount, unit }` 对象 → 通过 `DurationUtils.convertToMilliseconds()` 转换

### 5.3 Bridge V2 的元数据更新

对于通过 Novu Framework（Bridge）创建的工作流，`AddJob` 会在 `executeDeferredJob` 阶段调用 `fetchBridgeData()` 获取 Bridge 执行结果，并根据返回的 `outputs` 更新 Job 的 `step.metadata`（Delay）或 `digest`（Digest）字段（`add-job.usecase.ts:597-767`）。

Bridge 可以动态改变延迟参数，例如：
- Delay step 的 `outputs` 可以返回 `{ type: 'regular', amount: 5, unit: 'minutes' }`
- 也可以返回 `{ type: 'timed', cron: '0 9 * * 1-5' }` 切换到定时模式

### 5.4 存储依赖

| 存储 | 用途 |
|------|------|
| MongoDB `jobs` 集合 | Job 状态、`step.metadata`（含 delay 配置）、`expireAt`（TTL 索引） |
| BullMQ `StandardQueue` | 通过 `delay` 选项实现延迟投递 |
| MongoDB `step_runs` 集合 | 步骤运行记录 |

### 5.5 延迟到期后的恢复流程

1. BullMQ 延迟到期，Worker 消费消息，调用 `RunJob.execute()`
2. `RunJob` 检查 Job 是否已取消（`delayedEventIsCanceled`）
3. 检查订阅者日程（Subscriber Schedule），若在日程外且非关键通知：
   - 若 `extendToSchedule=true`，计算下一个可用时间并重新入队（最多扩展 3 次）
   - 否则取消 Job
4. 更新 Job 状态为 RUNNING
5. `SendMessage` 路由到 `SendMessageDelay.execute()`，仅记录 `DetailEnum.DELAY_FINISHED` 后返回 SUCCESS
6. `RunJob` 标记 COMPLETED，调用 `tryQueueNextJobs()` 调度下游节点

### 5.6 取消延迟

通过 `cancel-delayed.usecase.ts`，可按 `transactionId` 查找并取消 DELAYED 状态的 Job。

---

## 六、节点连接方式

### 6.1 Job 链式结构

所有控制节点与执行节点通过 **Job 父子链** 串联：

```
Job A (_parentId: null)
  └→ Job B (_parentId: Job A._id)
       └→ Job C (_parentId: Job B._id)
```

`RunJob.tryQueueNextJobs()` 通过 `_parentId` 索引查找下一个 Job，然后调用 `AddJob.execute()` 投递。

### 6.2 控制节点在链中的角色

```
Trigger → [Digest] → Email → [Delay] → SMS
                ↑                  ↑
          聚合窗口内的多个触发    延迟到指定时间后执行
```

- **Digest**：在链中充当"聚合门"，同一窗口内的新事件被 MERGED 到主 Digest Job，到期后主 Job 带着聚合的 events 传给下游
- **Delay**：在链中充当"延迟门"，到期后直接放行
- **Throttle**：在链中充当"限流门"，超过阈值则跳过自身及所有子 Job

### 6.3 条件过滤（Conditions Filter）

每个步骤在 `AddJob.executeDeferredJob()` 和 `SendMessage.execute()` 阶段都会经过条件过滤：

1. **AddJob 阶段**：`conditionsFilter.filter()` 判断是否应跳过此步骤，若 `filtered=true` 则 `delay=0`（立即执行跳过逻辑）
2. **SendMessage 阶段**：再次评估条件和用户偏好，不满足则返回 SKIPPED

---

## 七、执行详情（Execution Details）追踪

每类控制节点在关键状态变更时均会写入 `ExecutionDetails` 记录：

| 节点 | DetailEnum | 触发时机 |
|------|-----------|---------|
| Digest | `STEP_DIGESTED` | Job 延迟入队 |
| Digest | `DIGEST_MERGED` | Job 被合并 |
| Digest | `DIGEST_SKIPPED` | Job 被跳过 |
| Digest | `DIGEST_TRIGGERED_EVENTS` | 收集聚合事件 |
| Delay | `STEP_DELAYED` | Job 延迟入队 |
| Delay | `DELAY_FINISHED` | 延迟到期完成 |
| Delay | `DELAY_MISCONFIGURATION` | 动态延迟配置错误 |
| Throttle | `THROTTLE_LIMIT_EXCEEDED` | 限频跳过 |
| Throttle | `THROTTLE_WINDOW_IN_PAST` | 动态限频窗口在过去 |
| 通用 | `DEFER_DURATION_LIMIT_EXCEEDED` | 延迟时长超出租户限制 |
| 通用 | `STEP_CANCELED` | Job 被取消 |
| 通用 | `SKIPPED_STEP_BY_CONDITIONS` | 条件过滤跳过 |

---

## 八、Tier 限制校验

三类控制节点的延迟/等待时长均受租户等级限制校验（`TierRestrictionsValidateUsecase`），在 `AddJob` 阶段执行：

- **Defer Duration**：`deferDurationMs` 不得超过租户允许的最大延迟时长
- **Cron Expression**：某些定时表达式的频率受限制
- 校验失败抛出 `Defer duration limit exceeded` 错误，Job 标记 FAILED

---

## 九、Subscriber Schedule 扩展机制

Delay 和 Digest 节点支持 **extendToSchedule** 选项，允许将延迟到期时间扩展到订阅者的下一个可用日程时间：

1. `RunJob` 检测到当前时间不在订阅者日程内
2. 若 Job 为 Delay/Digest 且非关键通知，查询 Bridge 的 `extendToSchedule` 输出
3. 若 `extendToSchedule=true`，计算下一个可用时间并重新入队
4. 最多扩展 3 次（`MAX_EXTENSIONS = 3`），超过后直接发送

---

## 十、关键源码索引

| 功能 | 文件路径 |
|------|---------|
| Job 创建与链式调度 | `apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts` |
| Digest 合并决策 | `apps/worker/src/app/workflow/usecases/add-job/merge-or-create-digest.usecase.ts` |
| Digest 事件收集 | `apps/worker/src/app/workflow/usecases/send-message/digest/digest.usecase.ts` |
| Digest Regular 事件 | `apps/worker/src/app/workflow/usecases/send-message/digest/get-digest-events-regular.usecase.ts` |
| Digest Backoff 事件 | `apps/worker/src/app/workflow/usecases/send-message/digest/get-digest-events-backoff.usecase.ts` |
| Throttle 限频判定 | `apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts` (handleThrottle) |
| Throttle 限频跳过 | `apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts` (handleThrottleSkip) |
| Redis 限频服务 | `libs/application-generic/src/services/throttle/redis-throttle.service.ts` |
| Throttle 运行时 | `apps/worker/src/app/workflow/usecases/send-message/throttle/throttle.usecase.ts` |
| Delay 运行时 | `apps/worker/src/app/workflow/usecases/send-message/send-message-delay.usecase.ts` |
| 延迟时长计算 | `libs/application-generic/src/services/calculate-delay/compute-job-wait-duration.service.ts` |
| Timed 延迟计算 | `libs/application-generic/src/services/calculate-delay/timed-digest-delay.service.ts` |
| Job 执行与恢复 | `apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts` |
| 消息分发 | `apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts` |
| BullMQ 队列服务 | `libs/application-generic/src/services/bull-mq/bull-mq.service.ts` |
| 队列基础服务（含 SQS/BullMQ 路由） | `libs/application-generic/src/services/queues/queue-base.service.ts` |
| 标准 Worker（消费入口） | `apps/worker/src/app/workflow/services/standard.worker.ts` |
| Job Repository（updateAllChildJobStatus） | `libs/dal/src/repositories/job/job.repository.ts` |
| Delivery Lifecycle 判定 | `libs/application-generic/src/services/workflow-run.service.ts` |
| Job Schema | `libs/dal/src/repositories/job/job.schema.ts` |
| Digest 校验 | `apps/worker/src/app/workflow/usecases/add-job/validation.ts` |
| Delay 控制 Schema | `libs/application-generic/src/schemas/control/delay-control.schema.ts` |
| Throttle 控制 Schema | `libs/application-generic/src/schemas/control/throttle-control.schema.ts` |
| Digest 控制 DTO | `libs/application-generic/src/dtos/workflow/controls/digest-control.dto.ts` |
| Throttle 控制 DTO | `libs/application-generic/src/dtos/workflow/controls/throttle-control.dto.ts` |
| Throttle 类型定义 | `libs/application-generic/src/services/throttle/throttle.types.ts` |
| Job 创建入口 | `libs/application-generic/src/usecases/create-notification-jobs/create-notification-jobs.usecase.ts` |

---

## 十一、核心差异总览

### 11.1 三节点调度全景对比

| 维度 | Digest（聚合） | Delay（延迟） | Throttle（限频） |
|-----|-------------|------------|-----------------|
| **核心目的** | 聚合同窗口事件，减少下游通知次数 | 推迟下游执行到指定时间 | 保护下游不被过载，牺牲送达 |
| **决策存储** | MongoDB（Job 状态 + 合并关系 | MongoDB + BullMQ delay | Redis（Set + TTL） |
| **AddJob 决策** | 合并/创建/跳过 | 计算延迟时长 | 过/不通过 |
| **未通过结果** | MERGED / SKIPPED | 无（必然 DELAYED） | SKIPPED（级联子 Job） |
| **BullMQ delay | 入队时 delay>0 | 入队时 delay>0 | 通过时 delay=0<br>不通过不入队 |
| **状态性质** | DELAYED（中间态，自动恢复 | DELAYED（中间态，自动恢复 | SKIPPED（终态，无恢复） |
| **到期恢复** | BullMQ 到期自动调度 + 收集聚合 events | BullMQ 到期自动调度 + 放行 | 无恢复，新事件重新走完整链路 |
| **父子链** | 合并到主 Job，子 Job MERGED | 单 Job 延迟 | 级联 SKIPPED，链中断 |
| **送达保证** | ✅ 保证送达（聚合后） | ✅ 保证送达（延迟后） | ❌ 超限即丢弃 |
| **适用场景** | 批量通知、降噪 | 定时发送、错峰 | 保护下游 API/第三方服务 |

### 11.2 自动恢复能力矩阵

| 触发场景 | Digest | Delay | Throttle |
|---------|--------|-------|----------|
| 延迟/窗口到期 | ✅ 自动恢复 | ✅ 自动恢复 | ❌ 无自动恢复 |
| 主 Job 被取消 | ✅ 提升 Follower 继续 | ❌ 直接取消 | ❌ 无此概念 |
| 订阅者日程扩展 | ✅ 重新入队（最多 3 次） | ✅ 重新入队（最多 3 次） | ❌ 不涉及 |
| 新事件到达 | ✅ 合并到现有窗口 | ❌ 无窗口概念 | ✅ 新窗口重新计数 |
| 已跳过的旧事件 | ❌ 不会追溯 | ❌ 不会追溯 | ❌ 不会追溯 |
