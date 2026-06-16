# 定时排程与延时通知代码链路分析

## 一、概述

Novu 的延时通知和定时排程体系涉及三个核心环节：**调度入队** → **到期触发** → **失败重试**。整个链路横跨 API 层、Worker 层、队列层（BullMQ/SQS）以及可选的 Cloudflare Durable Object 调度器。

---

## 二、调度入队链路

### 2.1 入口：AddJob Usecase

**核心文件**：[add-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts)

#### 2.1.1 作业类型分流

```typescript
// add-job.usecase.ts#L168-L170
const result = isJobDeferredType(job.type)
  ? await this.executeDeferredJob(command)
  : await this.executeNoneDeferredJob(command);
```

**Deferred 类型**（需要入队等待）：
- `StepTypeEnum.DELAY` - 延时步骤
- `StepTypeEnum.DIGEST` - 聚合步骤
- `StepTypeEnum.THROTTLE` - 限流步骤

**Non-Deferred 类型**（立即执行）：
- Trigger、Email、SMS、In-App 等

#### 2.1.2 延时类型处理入口（executeDeferredJob）

**关键时机点**：
1. 条件过滤（conditionsFilter）- 如果过滤不通过，直接返回 SKIPPED
2. Bridge 数据获取（fetchBridgeData）- V2 框架从 Bridge 获取动态配置
3. 各类型专属处理：
   - DIGEST：[handleDigest](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L914-L967) - 计算聚合延迟，处理三态级联逻辑（MERGED/SKIPPED/CREATED）
   - THROTTLE：[handleThrottle](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L815-L912) - Redis 限流槽位预留，不走队列延迟
   - DELAY：[handleDelay](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L428-L471) - 计算延时时长

### 2.2 延迟计算服务

**核心文件**：[compute-job-wait-duration.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/calculate-delay/compute-job-wait-duration.service.ts)

#### 2.2.1 延迟类型分支（共 5 个分支 + 兜底 return 0）

```typescript
// compute-job-wait-duration.service.ts#L35-L126
if (digestType === DelayTypeEnum.SCHEDULED) {
  // 分支1: 从 payload 指定路径读取目标时间，计算与当前时间差值
} else if (digestType === DelayTypeEnum.DYNAMIC) {
  // 分支2: 动态延迟，支持 ISO8601 时间戳或 {amount, unit} 对象
} else if (
  digestType &&
  (digestType === DigestTypeEnum.REGULAR ||
   digestType === DigestTypeEnum.BACKOFF ||
   digestType === DelayTypeEnum.REGULAR) &&
  isRegularDigest(digestType)
) {
  // 分支3: 固定时长延迟（含 overrides 优先级）
} else if (digestType === DigestTypeEnum.TIMED) {
  // 分支4: 定时排程，通过 RRule 计算下一个触发时间点
} else if ((stepMetadata as IDelayRegularMetadata)?.unit && (stepMetadata as IDelayRegularMetadata)?.amount) {
  // 分支5: 兜底检查 unit 和 amount 字段（处理 type 缺失但有 unit/amount 的情况）
  // 同样支持 overrides 优先级
}

// 所有分支都不匹配时返回 0（无延迟）
return 0;
```

#### 2.2.2 定时排程计算（TIMED 类型）

**核心文件**：[timed-digest-delay.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/calculate-delay/timed-digest-delay.service.ts)

```typescript
// timed-digest-delay.service.ts#L67-L109
public static calculate({ dateStart, unit, amount, timeConfig, timezone }): number {
  // 1. 解析 atTime → hours, minutes, seconds
  // 2. 转换时区（toZonedTime）
  // 3. 计算 bysetpos/byweekday/bymonthday 字段
  // 4. 构建 RRule 规则
  // 5. rule.after(dateStartTz) 获取下一个执行时间
  // 6. 转回 UTC 计算与当前时间的差值（毫秒）
}
```

**RRule 配置**：
- `dtstart`: 起始时间（按时区转换后）
- `freq`: 频率（分钟/小时/天/周/月）
- `interval`: 间隔次数
- `byhour/byminute/bysecond`: 指定时间点
- `byweekday/bymonthday/bysetpos`: 复杂排程规则

#### 2.2.3 DIGEST 三态级联逻辑

**核心文件**：[merge-or-create-digest.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/merge-or-create-digest.usecase.ts)

DIGEST 作业在 handleDigest 中调用 `MergeOrCreateDigest.execute()`，返回三态之一，并在 AddJob 中进行级联处理：

```typescript
// add-job.usecase.ts#L264-L278
if (isShouldHaltJobExecution(digestResult.digestCreationResult)) {
  if (digestResult.digestCreationResult === DigestCreationResultEnum.MERGED) {
    // 级联1: MERGED - 已合并到现有 digest 作业
    // 标记当前 job 为 MERGED，返回 COMPLETED + MERGED 状态
    return {
      workflowStatus: WorkflowRunStatusEnum.COMPLETED,
      deliveryLifecycleStatus: DeliveryLifecycleStatusEnum.MERGED,
    };
  }

  if (digestResult.digestCreationResult === DigestCreationResultEnum.SKIPPED) {
    // 级联2: SKIPPED - 不创建新 digest
    // 标记当前 job 为 SKIPPED，返回 COMPLETED + SKIPPED 状态
    return {
      workflowStatus: WorkflowRunStatusEnum.COMPLETED,
      deliveryLifecycleStatus: DeliveryLifecycleStatusEnum.SKIPPED,
    };
  }
}

// 级联3: CREATED - 创建了新 digest 作业
// 继续执行，获取 digestAmount，后续走 queueJob() 入队延迟
digestAmount = digestResult.digestAmount;
```

**三态判定逻辑**：
| 状态 | 判定条件 | 级联行为 |
|------|---------|---------|
| `CREATED` | 无同 digestKey 的活跃作业 | 继续执行，入队延迟 |
| `MERGED` | 找到同 digestKey 的活跃 DELAYED 作业 | 标记 MERGED，更新 `_mergedDigestId`，终止当前 job |
| `SKIPPED` | backoff 类型且已有合并记录 | 标记 SKIPPED，终止当前 job |

---

### 2.3 THROTTLE 限流机制（不走队列延迟）

**核心文件**：[add-job.usecase.ts#L815-L912](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L815-L912)

THROTTLE 与 DELAY/DIGEST 不同，**不通过队列延迟实现**，而是通过 Redis 槽位预留机制：

```typescript
// handleThrottle() 核心逻辑
// 1. 解析配置：type (fixed/dynamic)、threshold、windowMs
// 2. Redis 槽位预留
const reservationResult = await this.redisThrottleService.reserveThrottleSlot({
  environmentId, subscriberId, workflowId, stepId, jobId,
  windowMs, limit: threshold, nowMs,
  throttleKey, throttleValue,
});

if (!reservationResult.granted) {
  // 预留失败：窗口内请求数已达阈值
  return { shouldSkip: true, executionCount, threshold, throttledUntil };
}

// 预留成功：返回 shouldSkip: false，不设置 delay，立即继续执行
return { shouldSkip: false, executionCount, threshold, throttledUntil };
```

**级联处理**（[add-job.usecase.ts#L283-L308](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L283-L308)）：
- `shouldSkip: true` → 调用 `handleThrottleSkip()`，返回 `COMPLETED + SKIPPED`
- `shouldSkip: false` → 继续执行，由于 `delayAmount` 始终为 undefined，最终 `delay = 0`，立即执行后续步骤

**四条 warn + early-return 兜底路径**：

| 路径 | 触发条件 | 源码行号 | 行为 |
|------|----------|----------|------|
| ① | fixed 类型缺 amount 或 unit | [add-job.usecase.ts#L828-L831](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L828-L831) | `logger.warn` → `return { shouldSkip: false }`，不预留，跳过限流 |
| ② | dynamic 类型缺 dynamicKey | [add-job.usecase.ts#L841-L844](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L841-L844) | `logger.warn` → `return { shouldSkip: false }`，不预留，跳过限流 |
| ③ | throttle type 为 unknown（非 fixed/dynamic） | [add-job.usecase.ts#L854-L857](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L854-L857) | `logger.warn` → `return { shouldSkip: false }`，不预留，跳过限流 |
| ④ | validateThrottleWindow 校验抛出异常 | [add-job.usecase.ts#L862](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L862) → 被外层 [add-job.usecase.ts#L299-L307](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L299-L307) 捕获 | 走 `handleStepValidationError()` → `setJobAsFailed`，标记 FAILED，终止执行 |

**路径 ① 完整代码**（fixed 缺 amount/unit）：
```typescript
// add-job.usecase.ts#L826-L831
if (type === 'fixed') {
  const { amount, unit } = throttleConfig;
  if (!amount || !unit) {
    this.logger.warn(`Fixed throttle configuration missing amount or unit for job ${job._id}`);
    return { shouldSkip: false };
  }
```

**路径 ② 完整代码**（dynamic 缺 dynamicKey）：
```typescript
// add-job.usecase.ts#L839-L844
} else if (type === 'dynamic') {
  const { dynamicKey } = throttleConfig;
  if (!dynamicKey) {
    this.logger.warn(`Dynamic throttle configuration missing dynamicKey for job ${job._id}`);
    return { shouldSkip: false };
  }
```

**路径 ③ 完整代码**（unknown type）：
```typescript
// add-job.usecase.ts#L854-L857
} else {
  this.logger.warn(`Unknown throttle type '${type}' for job ${job._id}`);
  return { shouldSkip: false };
}
```

**路径 ④ 完整代码链路**（validateThrottleWindow 异常）：

沿着调用栈逐层展开，异常有**两个源头**，最终被外层 try/catch 捕获，走统一的失败处理链路：

```
handleThrottle() L862
  └─ validateThrottleWindow() L1137-L1146
        └─ [仅 dynamic 类型] validateDynamicDuration() L473-L528
              ├─ 源头 A: durationMs <= 0（窗口落在过去）L483-L499
              └─ 源头 B: tier 限制校验不通过 L501-L527
executeDeferredJob() L299-L307 catch
  └─ handleStepValidationError() L530-L574
        ├─ 写执行详情（DELAY_MISCONFIGURATION）
        ├─ 标记 job.status = FAILED，写入 error 字段
        ├─ 写 StepRun FAILED 记录
        └─ 返回 workflowStatus = ERROR
```

**源头 A：durationMs <= 0（窗口落在过去）** [add-job.usecase.ts#L483-L499](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L483-L499)：
```typescript
// validateDynamicDuration() 内
if (durationMs <= 0) {
  this.logger.error(`Dynamic throttle must be in the future. durationMs: ${durationMs}, jobId: ${job._id}`);
  await this.createExecutionDetails.execute({
    detail: DetailEnum.THROTTLE_WINDOW_IN_PAST,   // 与 DELAY 区分的专属 detail
    source: ExecutionDetailsSourceEnum.INTERNAL,
    status: ExecutionDetailsStatusEnum.FAILED,
    raw: JSON.stringify({ error: `throttle must be in the future` }),
  });
  throw new Error(`Dynamic throttle must be in the future. durationMs: ${durationMs}`);
}
```

**源头 B：tier 限制校验不通过** [add-job.usecase.ts#L501-L527](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L501-L527)：
```typescript
// validateDynamicDuration() 内
const tierValidationErrors = await this.tierRestrictionsValidateUsecase.execute({
  stepType: StepTypeEnum.THROTTLE,
  deferDurationMs: windowMs,
});

if (tierValidationErrors && tierValidationErrors.length > 0) {
  await this.createExecutionDetails.execute({
    detail: DetailEnum.DEFER_DURATION_LIMIT_EXCEEDED,  // 与 DELAY/DIGEST 共用
    source: ExecutionDetailsSourceEnum.INTERNAL,
    status: ExecutionDetailsStatusEnum.FAILED,
    raw: JSON.stringify({ errorMessage }),
  });
  throw new Error(`throttle duration exceeds tier limits: ${errorMessage}`);
}
```

**外层捕获点：executeDeferredJob 中 THROTTLE try/catch** [add-job.usecase.ts#L283-L307](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L283-L307)：
```typescript
if (job.type === StepTypeEnum.THROTTLE) {
  try {
    const throttleResult = await this.handleThrottle(command, job, bridgeResponse);
    if (throttleResult.shouldSkip) {
      // ... handleThrottleSkip() 正常路径
    }
  } catch (error) {
    // validateThrottleWindow 抛出的异常统一进入此分支
    // 默认 detail = DetailEnum.DELAY_MISCONFIGURATION（与实际写入的 detail 不同，实际以 validateDynamicDuration 内写的为准）
    return await this.handleStepValidationError(
      command, job, error, StepTypeEnum.THROTTLE, DetailEnum.DELAY_MISCONFIGURATION
    );
  }
}
```

**统一失败处理：handleStepValidationError** [add-job.usecase.ts#L530-L574](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L530-L574)：
```typescript
// 1. 写入执行详情（DELAY_MISCONFIGURATION，FAILED 状态）
await createExecutionDetails.execute(...);

// 2. 更新 job 为 FAILED，保存完整 error 对象（message/name/stack）
await jobRepository.updateOne({ _id: job._id }, {
  $set: { status: JobStatusEnum.FAILED, error: { message, name, stack } }
});

// 3. 创建 StepRun FAILED 记录
await stepRunRepository.create(job, { status: JobStatusEnum.FAILED });

// 4. 返回 ERROR 工作流状态，后续不入队
return { workflowStatus: WorkflowRunStatusEnum.ERROR, deliveryLifecycleStatus: DeliveryLifecycleStatusEnum.ERRORED };
```

---

### 2.4 队列路由机制

**核心文件**：[queue-base.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts)

#### 2.4.1 入队入口

```typescript
// add-job.usecase.ts#L1061-L1097
public async queueJob({ job, delay, untilDate, timezone }) {
  const options: JobsOptions = { delay };
  
  // Webhook Filter 特殊处理：启用重试和退避
  if (stepContainsWebhookFilter) {
    options.backoff = { type: 'webhookFilterBackoff' };
    options.attempts = 3;
  }

  await this.standardQueueService.add({
    name: job._id,
    data: { _environmentId, _id, _organizationId, _userId },
    groupId: job._organizationId,
    options,
  });
}
```

#### 2.4.2 StandardQueueService 延迟路由

**核心文件**：[standard-queue.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/standard-queue.service.ts)

```typescript
// standard-queue.service.ts#L40-L51
public async add(data: IStandardJobDto) {
  const delay = data.options?.delay || 0;
  const hasDelay = delay > 0;

  // 有延迟的作业走 CF Scheduler + BullMQ 双轨
  if (hasDelay) {
    return await this.handleDelayedJob(data, delay);
  }

  // 无延迟走 SQS/BullMQ 常规路由
  return await super.add(data);
}
```

#### 2.4.3 Cloudflare Scheduler 模式

**核心文件**：[scheduler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/enterprise/workers/scheduler/src/scheduler.ts)

四种模式（`CF_SCHEDULER_MODE` 特性开关控制）：

| 模式 | 行为 |
|------|------|
| `OFF` | 仅 BullMQ |
| `SHADOW` | BullMQ 为主，CF Scheduler 同步写入用于验证 |
| `LIVE` | CF Scheduler 为主，BullMQ 写入 skipProcessing 标记的影子作业 |
| `COMPLETE` | 仅 CF Scheduler |

#### 2.4.3.1 isInPast 立即执行逻辑

**核心代码**：[scheduler.ts#L68-L90](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/enterprise/workers/scheduler/src/scheduler.ts#L68-L90)

```typescript
private async scheduleJob(request: ScheduleJobRequest): Promise<void> {
  const now = Date.now();
  const isInPast = request.scheduledFor <= now;

  if (isInPast) {
    // 调度时间已过 → 立即执行，不设置 alarm
    const job: ScheduledJob = { ... };
    await this.executeJob(job);
    return;
  }

  // 未来时间 → 持久化 + 设置 alarm
  await Promise.all([
    this.state.storage.put(JOB_KEY, job),
    this.state.storage.setAlarm(request.scheduledFor)
  ]);
}
```

**关键点**：`isInPast` 是 Durable Object 内的本地判断，不通过队列延迟。

#### 2.4.3.2 cancel 取消逻辑

**核心代码**：[scheduler.ts#L29-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/enterprise/workers/scheduler/src/scheduler.ts#L29-L32)、[scheduler.ts#L105-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/enterprise/workers/scheduler/src/scheduler.ts#L105-L115)

```typescript
// HTTP 入口
case 'cancel': {
  const cancelled = await this.cancelJob();
  return Response.json({ success: cancelled });
}

// 取消实现
private async cancelJob(): Promise<boolean> {
  const job = await this.state.storage.get<ScheduledJob>(JOB_KEY);
  if (!job) return false;

  // 同时删除持久化数据和 alarm
  await Promise.all([
    this.state.storage.deleteAll(),
    this.state.storage.deleteAlarm()
  ]);
  return true;
}
```

**取消触发点**：在 [cancel-delayed.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/api/src/app/events/usecases/cancel-delayed/cancel-delayed.usecase.ts) 中，通过 `X-Action: cancel` 请求头调用 CF Scheduler。

#### 2.4.4 QueueBaseService 后端路由

**核心逻辑**：[queue-base.service.ts#L109-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L109-L264)

```
add(params)
  │
  ├─ 延迟 > 15 分钟 → 强制 BullMQ（SQS 限制）
  │
  ├─ 无 organizationId → BullMQ 兜底
  │
  └─ getQueueBackendMode() → 四种模式：
     ├─ BULLMQ → 仅 BullMQ
     ├─ SHADOW → BullMQ + SQS(skipProcessing)
     ├─ LIVE → SQS 为主，失败回退 BullMQ
     └─ COMPLETE → 仅 SQS，失败回退 BullMQ
```

**关键兜底分支**：
- SQS 写入失败时，自动回退到 BullMQ（LIVE/COMPLETE 模式）
- 组织查询失败 → 跳过作业（返回 null）
- 未知模式 → BullMQ 兜底

---

## 三、到期触发链路

### 3.1 触发源分类

#### 3.1.1 BullMQ 内部延迟队列

**核心文件**：[bull-mq.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/bull-mq/bull-mq.service.ts)

BullMQ 原生支持 `delay` 选项，内部基于 Redis sorted set 实现：
- 入队时：`ZADD delayed: {timestamp} {jobId}`
- 轮询时：`ZRANGEBYSCORE delayed: -inf {now}` 取出到期作业
- 到期后移入 waiting 队列等待 Worker 消费

#### 3.1.2 SQS 延迟消息

SQS 原生支持 `DelaySeconds`（最大 15 分钟），到期后消息变为可见，消费者拉取处理。

#### 3.1.3 Cloudflare Durable Object Alarm

**核心文件**：[scheduler.ts#L47-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/enterprise/workers/scheduler/src/scheduler.ts#L47-L66)

```typescript
// 入队时设置 alarm
await this.state.storage.setAlarm(request.scheduledFor);

// alarm 到期自动触发
async alarm(): Promise<void> {
  const job = await this.state.storage.get<ScheduledJob>(JOB_KEY);
  try {
    await this.executeJob(job);  // 回调 Novu API
  } finally {
    await this.state.storage.deleteAll();
  }
}
```

**回调目标**：`/v1/internal/scheduler/callback`

#### 3.1.4 订阅者定时排程（Subscriber Schedule）

**核心文件**：[schedule-validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/schedule-validator.ts)

在 RunJob 执行时检查：
```typescript
// run-job.usecase.ts#L195-L256
const isOutsideSubscriberSchedule = schedule?.isEnabled
  ? !isWithinSchedule(schedule, new Date(), timezone)
  : false;

if (isOutsideSubscriberSchedule) {
  if (shouldExtendToSubscriberSchedule) {
    // 延长到下一个可用时间窗口
    await extendJobToNextAvailableSchedule(job, schedule, timezone);
  } else {
    // 取消作业
    await jobRepository.updateStatus(..., CANCELED);
  }
}
```

**排程延长逻辑**：[extendJobToNextAvailableSchedule](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L911-L1020)
- 最大延长次数：3 次
- 计算下一个可用时间：`calculateNextAvailableTime()`
- 更新 `scheduleExtensionsCount` 并重新入队

### 3.2 消费入口：StandardWorker

**核心文件**：[standard.worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/services/standard.worker.ts)

#### 3.2.1 Worker 初始化

```typescript
// standard.worker.ts#L48-L89
this.initWorker(processor, options, true);

// 监听 BullMQ 事件
this.bullMqWorker.on('failed', async (job, error) => {
  await this.jobHasFailed(job, error);
});
this.bullMqWorker.on('completed', async (job) => {
  await this.jobHasCompleted(job);
});

// SQS 处理器
this.setSqsFailedHandler(async (job, error) => {
  return await this.jobHasFailed(job, error);
});
```

#### 3.2.2 处理器核心

```typescript
// standard.worker.ts#L141-L199
private getWorkerProcessor() {
  return async ({ data }) => {
    // 1. Kill Switch 检查（组织级熔断）
    if (await isKillSwitchEnabled(data)) return;
    
    // 2. skipProcessing 标记检查（迁移影子流量）
    if (data.skipProcessing) return;
    
    // 3. 组织存在性校验
    if (!await organizationExist(data)) return;
    
    // 4. 执行 RunJob
    return await this.runJob.execute(RunJobCommand.create(minimalJobData));
  };
}
```

### 3.3 RunJob 执行链路

**核心文件**：[run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts)

#### 3.3.1 执行前检查

```typescript
// run-job.usecase.ts#L84-L114
public async execute(command: RunJobCommand) {
  // 1. 加载 Job
  const job = await jobRepository.findOne(...);
  
  // 2. 延迟事件取消检查
  const { canceled, activeDigestFollower } = await delayedEventIsCanceled(job);
  if (canceled && !activeDigestFollower) {
    await stepRunRepository.create(..., CANCELED);
    return;
  }
  
  // 3. Digest 取消兜底：找到 follower 顶替执行
  if (activeDigestFollower) {
    job = assignNewDigestExecutor(activeDigestFollower);
  }
}
```

**延迟取消检查**：[delayedEventIsCanceled](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L682-L698)
- 针对 DELAY/DIGEST/THROTTLE 类型做取消检查
- **follower 顶替仅 DIGEST 适用**：[activeDigestMainFollowerExist](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L701-L723) 第一行判断 `if (job.type !== StepTypeEnum.DIGEST) return null;`
- 仅当 Job 类型为 DIGEST 且状态为 CANCELED 时，才查找同 digestKey 的 follower 顶替执行

#### 3.3.2 订阅者排程检查

```typescript
// run-job.usecase.ts#L176-L256
// 1. 获取订阅者排程配置
const schedule = await getSubscriberSchedule.execute(...);

// 2. 检查是否在排程窗口内
const isOutsideSubscriberSchedule = schedule?.isEnabled
  ? !isWithinSchedule(schedule, new Date(), timezone)
  : false;

// 3. 分支处理
if (isOutsideSubscriberSchedule) {
  if (shouldExtendToSubscriberSchedule) {
    // 延迟步骤且非紧急 → 延长到下一个窗口
    const extended = await extendJobToNextAvailableSchedule(job, schedule, timezone);
    if (extended) return;  // 已重新入队，终止当前执行
  }
  
  if (!shouldSkipScheduleCheck) {
    // 非延迟/非紧急 → 取消作业
    await jobRepository.updateStatus(..., CANCELED);
    return;
  }
}
```

**跳过排程检查的条件**：
- TRIGGER / IN_APP / DELAY / DIGEST / HTTP_REQUEST 类型
- critical 标记为 true 的消息

#### 3.3.3 消息发送与后续链式调度

```typescript
// run-job.usecase.ts#L274-L370
const sendMessageResult = await this.sendMessage.execute(...);

if (sendMessageResult.status === 'success') {
  await jobRepository.updateStatus(..., COMPLETED);
} else if (sendMessageResult.status === 'failed') {
  await jobRepository.update(..., { status: FAILED, error: ... });
  
  if (shouldHaltOnStepFailure(job)) {
    shouldQueueNextJob = false;
    await cancelPendingJobs(...);  // 取消后续作业
  }
}

// finally 块：链式调度下一个作业
finally {
  if (shouldQueueNextJob && !isJobExtendedToSubscriberSchedule) {
    await tryQueueNextJobs(job, notification, !!error);
  }
}
```

**SendMessageDelay 特殊说明**：[send-message-delay.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-delay.usecase.ts)

DELAY 类型的消息发送**只写完成记录，不实际发送消息**：
```typescript
// send-message-delay.usecase.ts#L23-L38
public async execute(command: SendMessageCommand): Promise<SendMessageResult> {
  // 仅写入 DELAY_FINISHED 执行详情
  await this.createExecutionDetails.execute(
    CreateExecutionDetailsCommand.create({
      ...CreateExecutionDetailsCommand.getDetailsFromJob(command.job),
      detail: DetailEnum.DELAY_FINISHED,
      source: ExecutionDetailsSourceEnum.INTERNAL,
      status: ExecutionDetailsStatusEnum.SUCCESS,
      ...
    })
  );

  return { status: SendMessageStatus.SUCCESS };
}
```

**链式调度**：[tryQueueNextJobs](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L468-L637)
- 通过 `_parentId` 找到下一个作业
- 循环调用 `addJobUsecase.execute()` 直到遇到需要延迟的步骤
- 如果下一个作业被条件过滤 SKIPPED，继续查找后续作业

---

## 四、失败重试链路

### 4.1 重试触发入口

**核心文件**：[standard.worker.ts#L230-L285](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/services/standard.worker.ts#L230-L285)

```typescript
private async jobHasFailed(job: Job, error: Error): Promise<boolean> {
  const minimalData = this.extractMinimalJobData(job.data);
  
  // 1. 判断是否需要退避重试（Webhook Filter 场景）
  const hasToBackoff = this.runJob.shouldBackoff(error);
  
  // 2. 判断是否达到最大尝试次数（DEFAULT_ATTEMPTS = 3）
  const hasReachedMaxAttempts = job.attemptsMade >= this.DEFAULT_ATTEMPTS;
  
  // 3. 最后一次失败处理
  const shouldHandleLastFailedJob = hasToBackoff && hasReachedMaxAttempts;
  
  // 4. 标记 Job 失败（非退避场景 或 最后一次失败）
  const shouldBeSetAsFailed = !hasToBackoff || shouldHandleLastFailedJob;
  if (shouldBeSetAsFailed) {
    await setJobAsFailed.execute(...);
  }
  
  // 5. 最后一次失败的特殊处理
  if (shouldHandleLastFailedJob) {
    await handleLastFailedJob.execute(...);
  }
  
  // 6. 返回值决定是否重试：需要退避且未达最大次数 → 重试
  return hasToBackoff && !hasReachedMaxAttempts;
}
```

**返回值语义**（SQS 场景）：
- `true` → 抛出错误，SQS 保持消息，可见性超时后重新投递
- `false` → 不抛出，SQS 删除消息（确认消费）

### 4.2 退避条件判断

```typescript
// run-job.usecase.ts#L725-L727
public shouldBackoff(error: Error): boolean {
  return error?.message?.includes(EXCEPTION_MESSAGE_ON_WEBHOOK_FILTER);
}
```

**仅 Webhook Filter 失败场景会触发重试**。这是因为 Webhook Filter 是用户自定义的外部接口，可能因网络波动等临时故障失败。

#### 4.2.1 webhookFilterBackoff 退避策略注册机制

退避策略通过 BullMQ Worker 的 `settings.backoffStrategy` 注册，`getBackoffStrategies()` 返回的是**单函数签名**，与 BullMQ 原生约定完全一致。入队时标记的 `backoff.type` 参数在实际执行时被忽略。

**注册点 1 - 入队时标记退避类型**：
```typescript
// add-job.usecase.ts#L96-L99
if (stepContainsWebhookFilter) {
  options.backoff = { type: 'webhookFilterBackoff' };  // 标记（运行时被忽略）
  options.attempts = 3;                                  // 最大尝试次数
}
```

**注册点 2 - Worker 初始化时注册策略**：
```typescript
// standard.worker.ts#L91-L98
private getWorkerOptions(): WorkerOptions {
  return {
    ...getStandardWorkerOptions(),
    settings: {
      backoffStrategy: this.getBackoffStrategies(),  // 传入单函数
    },
  };
}
```

**注册点 3 - 单函数签名实现**（`type` 参数被忽略）：
```typescript
// standard.worker.ts#L287-L298
private getBackoffStrategies = () => {
  return async (attemptsMade: number, type: string, eventError: Error, eventJob: Job): Promise<number> => {
    // 注：参数 type（即入队时标记的 'webhookFilterBackoff'）在此函数体中未被引用
    // 所有启用了 attempts 的作业统一走同一策略
    return await this.webhookFilterBackoffStrategy.execute({
      attemptsMade,
      environmentId: eventJob?.data?._environmentId,
      eventError,
      eventJob,
      organizationId: eventJob?.data?._organizationId,
      userId: eventJob?.data?._userId,
    });
  };
};
```

**完整调用链路**：
1. 入队时设置 `backoff = { type: 'webhookFilterBackoff' }` + `attempts = 3`
2. Worker 初始化时将 `settings.backoffStrategy` 设为单函数（4 参数签名）
3. 作业失败且有剩余 attempts 时，BullMQ 调用 `backoffStrategy(attemptsMade, type, err, job)`
4. 函数体内忽略 `type`，统一调用 `WebhookFilterBackoffStrategy.execute()` 计算退避延迟
5. 按计算出的延迟重新调度作业

### 4.3 退避策略计算

**核心文件**：[webhook-filter-backoff-strategy.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts)

```typescript
// webhook-filter-backoff-strategy.usecase.ts#L11-L36
public async execute(command): Promise<number> {
  // 1. 记录执行详情
  await createExecutionDetails.execute({
    detail: DetailEnum.WEBHOOK_FILTER_FAILED_RETRY,
    isRetry: true,
    raw: { message: ..., attempt: attemptsMade }
  });
  
  // 2. 指数退避 + 随机抖动
  // delay = random(0, 1) * 2^attempts * 1000 ms
  return Math.round(Math.random() * 2 ** attemptsMade * 1000);
}
```

**退避曲线**（近似值，含随机抖动）：
| 尝试次数 | 延迟范围 |
|---------|---------|
| 第1次重试 | 0 - 2 秒 |
| 第2次重试 | 0 - 4 秒 |
| 第3次重试 | 0 - 8 秒 |

### 4.4 最后一次失败处理

**核心文件**：[handle-last-failed-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/handle-last-failed-job/handle-last-failed-job.usecase.ts)

```typescript
// handle-last-failed-job.usecase.ts#L33-L66
public async execute(command) {
  // 1. 记录最后一次重试失败
  await createExecutionDetails.execute({
    detail: DetailEnum.WEBHOOK_FILTER_FAILED_LAST_RETRY,
    isRetry: true,
    raw: { message: ... }
  });
  
  // 2. 如果配置为不中止流程，继续调度下一个作业
  if (!shouldHaltOnStepFailure(job)) {
    await queueNextJob.execute({ parentId: job._id, ... });
  }
}
```

### 4.5 SQS 与 BullMQ 重试差异

| 维度 | BullMQ | SQS |
|------|--------|-----|
| 重试触发 | `worker.on('failed')` 事件 | `setSqsFailedHandler` 返回值 |
| 重试间隔 | 退避策略计算的精确延迟 | 统一的 `VisibilityTimeout`（默认 30 秒） |
| 最大次数 | `attempts` 选项（Webhook Filter 为 3） | `RedrivePolicy.maxReceiveCount`（标准队列为 3） |
| 死信队列 | BullMQ 内置 `failed` 状态 | SQS RedrivePolicy 配置的 DLQ |
| attemptsMade | BullMQ 原生维护 | 通过 `createSqsJobAdapter` 从 `meta.receiveCount` 映射 |

### 4.6 失败标记：SetJobAsFailed

**核心文件**：[set-job-as-failed.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/update-job-status/set-job-as-failed.usecase.ts)

```typescript
// set-job-as-failed.usecase.ts#L24-L55
public async execute(command, error) {
  // 1. 更新状态为 FAILED
  const jobEntity = await updateJobStatus.execute({ status: FAILED });
  
  // 2. 保存错误信息
  await jobRepository.setError(organizationId, jobId, error);
  
  // 3. 写入 StepRun 记录
  await stepRunRepository.create(..., {
    status: FAILED,
    errorCode: 'job_failed',
    errorMessage: error.message,
  });
  
  // 4. 更新工作流交付生命周期
  await workflowRunService.updateDeliveryLifecycle({
    workflowStatus: isLastJobFailed ? COMPLETED : PROCESSING,
  });
}
```

---

## 五、兜底分支与边界情况汇总

### 5.1 入队阶段兜底

**THROTTLE 四条 warn + early-return 兜底路径**（与 2.3 节结构呼应）：

#### 5.1.1 路径① - fixed 类缺 amount 或 unit

**源码位置**：[add-job.usecase.ts#L828-L831](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L828-L831)

**触发条件**：`type === 'fixed'` 时，`throttleConfig` 中缺少 `amount` 或 `unit` 字段。

**完整代码**：
```typescript
// add-job.usecase.ts#L826-L831
if (type === 'fixed') {
  const { amount, unit } = throttleConfig;
  if (!amount || !unit) {
    this.logger.warn(`Fixed throttle configuration missing amount or unit for job ${job._id}`);
    return { shouldSkip: false };
  }
```

**行为**：`logger.warn` 记录配置缺失 → `return { shouldSkip: false }`，不做 Redis 槽位预留，跳过限流，`delayAmount` 为 undefined，最终 `delay = 0` 立即继续执行后续步骤。

---

#### 5.1.2 路径② - dynamic 类缺 dynamicKey

**源码位置**：[add-job.usecase.ts#L841-L844](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L841-L844)

**触发条件**：`type === 'dynamic'` 时，`throttleConfig` 中缺少 `dynamicKey` 字段。

**完整代码**：
```typescript
// add-job.usecase.ts#L839-L844
} else if (type === 'dynamic') {
  const { dynamicKey } = throttleConfig;
  if (!dynamicKey) {
    this.logger.warn(`Dynamic throttle configuration missing dynamicKey for job ${job._id}`);
    return { shouldSkip: false };
  }
```

**行为**：`logger.warn` 记录配置缺失 → `return { shouldSkip: false }`，不做 Redis 槽位预留，跳过限流，`delayAmount` 为 undefined，最终 `delay = 0` 立即继续执行后续步骤。

---

#### 5.1.3 路径③ - unknown type

**源码位置**：[add-job.usecase.ts#L854-L857](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L854-L857)

**触发条件**：`type` 既不是 `'fixed'` 也不是 `'dynamic'`，即未知类型。

**完整代码**：
```typescript
// add-job.usecase.ts#L854-L857
} else {
  this.logger.warn(`Unknown throttle type '${type}' for job ${job._id}`);
  return { shouldSkip: false };
}
```

**行为**：`logger.warn` 记录未知类型 → `return { shouldSkip: false }`，不做 Redis 槽位预留，跳过限流，`delayAmount` 为 undefined，最终 `delay = 0` 立即继续执行后续步骤。

---

#### 5.1.4 路径④ - validateThrottleWindow 异常（含两个 throw 源头）

**调用链路**（与 2.3 节路径④完全一致）：
```
handleThrottle() L862
  └─ validateThrottleWindow() L1137-L1146
        └─ [仅 dynamic 类型] validateDynamicDuration() L473-L528
              ├─ 源头 A: durationMs <= 0（窗口落在过去）L483-L499
              └─ 源头 B: tier 限制校验不通过 L501-L527
executeDeferredJob() L299-L307 catch
  └─ handleStepValidationError() L530-L574
        ├─ 写执行详情（DELAY_MISCONFIGURATION）
        ├─ 标记 job.status = FAILED，写入 error 字段
        ├─ 写 StepRun FAILED 记录
        └─ 返回 workflowStatus = ERROR
```

**调用点**：[add-job.usecase.ts#L862](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L862)

```typescript
// handleThrottle() 内
await this.validateThrottleWindow(command, job, windowMs, type);
```

**源头 A：durationMs <= 0（窗口落在过去）** [add-job.usecase.ts#L483-L499](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L483-L499)：
```typescript
// validateDynamicDuration() 内
if (durationMs <= 0) {
  this.logger.error(`Dynamic throttle must be in the future. durationMs: ${durationMs}, jobId: ${job._id}`);
  await this.createExecutionDetails.execute({
    detail: DetailEnum.THROTTLE_WINDOW_IN_PAST,
    source: ExecutionDetailsSourceEnum.INTERNAL,
    status: ExecutionDetailsStatusEnum.FAILED,
    raw: JSON.stringify({ error: `throttle must be in the future` }),
  });
  throw new Error(`Dynamic throttle must be in the future. durationMs: ${durationMs}`);
}
```

**源头 B：tier 限制校验不通过** [add-job.usecase.ts#L501-L527](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L501-L527)：
```typescript
// validateDynamicDuration() 内
const tierValidationErrors = await this.tierRestrictionsValidateUsecase.execute({
  stepType: StepTypeEnum.THROTTLE,
  deferDurationMs: windowMs,
});

if (tierValidationErrors && tierValidationErrors.length > 0) {
  await this.createExecutionDetails.execute({
    detail: DetailEnum.DEFER_DURATION_LIMIT_EXCEEDED,
    source: ExecutionDetailsSourceEnum.INTERNAL,
    status: ExecutionDetailsStatusEnum.FAILED,
    raw: JSON.stringify({ errorMessage }),
  });
  throw new Error(`throttle duration exceeds tier limits: ${errorMessage}`);
}
```

**外层捕获点** [add-job.usecase.ts#L299-L307](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L299-L307)：
```typescript
} catch (error) {
  return await this.handleStepValidationError(
    command, job, error, StepTypeEnum.THROTTLE, DetailEnum.DELAY_MISCONFIGURATION
  );
}
```

**统一失败处理** [add-job.usecase.ts#L530-L574](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/add-job/add-job.usecase.ts#L530-L574)：
```typescript
// 1. 写入执行详情（DELAY_MISCONFIGURATION，FAILED 状态）
await createExecutionDetails.execute(...);

// 2. 更新 job 为 FAILED，保存完整 error 对象
await jobRepository.updateOne({ _id: job._id }, {
  $set: { status: JobStatusEnum.FAILED, error: { message, name, stack } }
});

// 3. 创建 StepRun FAILED 记录
await stepRunRepository.create(job, { status: JobStatusEnum.FAILED });

// 4. 返回 ERROR 工作流状态，后续不入队
return { workflowStatus: WorkflowRunStatusEnum.ERROR, ... };
```

**行为**：异常沿调用栈向上抛出，被 `executeDeferredJob()` 中的 try/catch 捕获，走统一的 `handleStepValidationError()` 失败处理，最终标记 job 为 FAILED，不入队，工作流终止。

---

**其他入队兜底**：

5. **延迟 > 15 分钟**：[queue-base.service.ts#L112-L120](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L112-L120)
   - SQS 不支持 > 15 分钟延迟 → 强制走 BullMQ

6. **无 organizationId**：[queue-base.service.ts#L134-L138](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L134-L138)
   - 无法路由 → BullMQ 兜底

7. **组织不存在**：[queue-base.service.ts#L169-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L169-L173)
   - 跳过作业（返回 null，不入队）

8. **SQS 写入失败**：[queue-base.service.ts#L226-L238](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L226-L238)
   - LIVE/COMPLETE 模式 → 自动回退 BullMQ

9. **未知队列模式**：[queue-base.service.ts#L261-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L261-L264)
   - BullMQ 兜底

### 5.2 执行阶段兜底

1. **Job 已取消**：[run-job.usecase.ts#L102-L114](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L102-L114)
   - 到期触发时 Job 已被取消 → 标记 CANCELED 并退出

2. **Digest 主作业取消，follower 顶替**：[run-job.usecase.ts#L116-L119](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L116-L119)
   - 仅 DIGEST 类型适用，找到同 digestKey 的 follower 继续执行

3. **排程窗口外**：[run-job.usecase.ts#L195-L256](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L195-L256)
   - 可延长 → 重新入队到下一个窗口
   - 不可延长 → 取消作业

4. **Kill Switch 熔断**：[standard.worker.ts#L143-L149](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/services/standard.worker.ts#L143-L149)
   - 组织级熔断开启 → 跳过作业

5. **影子流量标记**：[standard.worker.ts#L151-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/services/standard.worker.ts#L151-L155)
   - `skipProcessing: true` → 跳过执行

### 5.3 失败阶段兜底

1. **非 Webhook Filter 错误**：[standard.worker.ts#L243](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/services/standard.worker.ts#L243)
   - 不重试，直接标记 FAILED

2. **最后一次重试失败**：[handle-last-failed-job.usecase.ts#L55-L65](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/handle-last-failed-job/handle-last-failed-job.usecase.ts#L55-L65)
   - 不中止流程 → 继续调度下一个作业

3. **SQS 永久性客户端错误**：[worker-base.service.ts#L227-L249](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/workers/worker-base.service.ts#L227-L249)
   - **仅 sqsFailedHandler 未注册时生效**：如果已注册 handler，先走 handler 返回值；只有未注册时才走兜底
   - 4xx 错误（排除 408、429）→ 确认消费（ack），不重试，避免进入 DLQ
   - 四个生产 SQS worker（workflow/subscriber-process/ws/standard）都已注册 handler，此分支仅保护未来新增忘记注册的 worker

4. **排程延长超限**：[run-job.usecase.ts#L919-L942](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L919-L942)
   - 超过最大延长次数（3 次）→ 不再延长，立即发送

---

## 六、完整链路时序图

```
API 触发事件
    ↓
TriggerEvent Usecase
    ↓
创建 Job（DELAY/DIGEST/THROTTLE 类型）
    ↓
AddJob.execute()
    ├─ 条件过滤
    ├─ Bridge 数据获取（V2）
    ├─ 类型分支处理
    │   ├─ DELAY → handleDelay() → 计算 delay
    │   ├─ DIGEST → handleDigest() → 三态级联
    │   │   ├─ MERGED → 标记合并，返回
    │   │   ├─ SKIPPED → 标记跳过，返回
    │   │   └─ CREATED → 计算 digestAmount，继续
    │   └─ THROTTLE → handleThrottle() → Redis 槽位预留
    │       ├─ 预留失败 → handleThrottleSkip() → 返回
    │       └─ 预留成功 → delay = 0，立即继续
    └─ queueJob()
        └─ StandardQueueService.add()
            ├─ delay > 0 → handleDelayedJob()
            │   ├─ CF Scheduler.isInPast → 立即执行
            │   ├─ CF Scheduler.cancel → 删除存储和 alarm
            │   └─ CF Scheduler 模式检查
            │       ├─ OFF → BullMQ
            │       ├─ SHADOW → BullMQ + CF(验证)
            │       ├─ LIVE → CF 为主, BullMQ 影子
            │       └─ COMPLETE → 仅 CF
            └─ delay = 0 → QueueBaseService.add()
                ├─ delay > 15min → BullMQ
                ├─ 无 orgId → BullMQ
                └─ 按 QUEUE_BACKEND_MODE 路由
                    ├─ BULLMQ → BullMQ
                    ├─ SHADOW → BullMQ + SQS(skip)
                    ├─ LIVE → SQS 失败回退 BullMQ
                    └─ COMPLETE → SQS 失败回退 BullMQ

============= 到期等待 ============
BullMQ: Redis sorted set 轮询 → 到期移入 waiting
SQS: DelaySeconds → 到期变为可见
CF Scheduler: Durable Object Alarm → 到期回调 API

============= 到期触发 ============
StandardWorker.getWorkerProcessor()
    ├─ Kill Switch 检查
    ├─ skipProcessing 检查
    ├─ 组织存在性检查
    └─ RunJob.execute()
        ├─ 加载 Job
        ├─ 延迟取消检查（仅 DIGEST 有 follower 顶替）
        ├─ 订阅者排程检查
        │   ├─ 在窗口内 → 继续
        │   ├─ 可延长 → extendJobToNextAvailableSchedule() → 重新入队
        │   └─ 不可延长 → 标记 CANCELED
        ├─ SendMessage.execute()
        │   └─ DELAY 类型 → SendMessageDelay.execute()（仅写完成记录，不实际发消息）
        └─ tryQueueNextJobs() → 链式调度下一个作业

============= 失败处理 ============
jobHasFailed()
    ├─ hasToBackoff = error 含 EXCEPTION_MESSAGE_ON_WEBHOOK_FILTER
    ├─ hasReachedMaxAttempts = attemptsMade >= 3
    ├─ !hasToBackoff → 标记 FAILED, 不重试
    ├─ hasToBackoff && !max → webhookFilterBackoff 策略(指数退避) → 重试
    │   └─ 注册机制: backoff.type → backoffStrategy 映射 → execute() 计算延迟
    └─ hasToBackoff && max → 标记 FAILED + HandleLastFailedJob → 继续下一个作业
```

---

## 七、关键配置参数

| 参数 | 值 | 位置 | 说明 |
|------|-----|------|------|
| `DEFAULT_ATTEMPTS` | 3 | [queue-base.service.ts#L16](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L16) | 最大重试次数 |
| `SQS_MAX_DELAY_SECONDS` | 900 (15min) | [queue-base.service.ts#L9](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/libs/application-generic/src/services/queues/queue-base.service.ts#L9) | SQS 最大延迟 |
| `MAX_EXTENSIONS` | 3 | [run-job.usecase.ts#L916](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L916) | 排程最大延长次数 |
| 退避公式 | `random() * 2^attempts * 1000` | [webhook-filter-backoff-strategy.usecase.ts#L35](file:///d:/fz/0601-2/solo-dogfeeding/code/5-novu/apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts#L35) | 指数退避 + 抖动 |
