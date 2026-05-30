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
              ├─ granted=true  → DELAYED → RUNNING → COMPLETED
              └─ granted=false → SKIPPED（含所有子 Job）
```

**注意**：Throttle 虽然是 Deferred 类型，但其核心决策在 `AddJob.executeDeferredJob()` 阶段通过 Redis 完成，**不经过 BullMQ 延迟**——通过限频的 Job 直接以 `delay=0` 入队立即执行；未通过的 Job 直接 SKIPPED，不会延迟重试。

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
| Redis 限频服务 | `libs/application-generic/src/services/throttle/redis-throttle.service.ts` |
| Throttle 运行时 | `apps/worker/src/app/workflow/usecases/send-message/throttle/throttle.usecase.ts` |
| Delay 运行时 | `apps/worker/src/app/workflow/usecases/send-message/send-message-delay.usecase.ts` |
| 延迟时长计算 | `libs/application-generic/src/services/calculate-delay/compute-job-wait-duration.service.ts` |
| Timed 延迟计算 | `libs/application-generic/src/services/calculate-delay/timed-digest-delay.service.ts` |
| Job 执行与恢复 | `apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts` |
| 消息分发 | `apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts` |
| Job Schema | `libs/dal/src/repositories/job/job.schema.ts` |
| Digest 校验 | `apps/worker/src/app/workflow/usecases/add-job/validation.ts` |
| Delay 控制 Schema | `libs/application-generic/src/schemas/control/delay-control.schema.ts` |
| Throttle 控制 Schema | `libs/application-generic/src/schemas/control/throttle-control.schema.ts` |
| Digest 控制 DTO | `libs/application-generic/src/dtos/workflow/controls/digest-control.dto.ts` |
| Throttle 控制 DTO | `libs/application-generic/src/dtos/workflow/controls/throttle-control.dto.ts` |
| Throttle 类型定义 | `libs/application-generic/src/services/throttle/throttle.types.ts` |
| Job 创建入口 | `libs/application-generic/src/usecases/create-notification-jobs/create-notification-jobs.usecase.ts` |
