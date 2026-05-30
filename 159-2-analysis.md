# 发送渠道健康检测与异常降级实现分析（代码级精读版）

## 一、健康检查的触发链路

### 1.1 基础设施级健康检查 — 完全被动，外部触发

三个服务各自暴露 `GET /health-check` 端点，由外部监控系统（K8s liveness/readiness、New Relic、自定义 runner）轮询调用。**代码内部没有任何定时器或调度逻辑驱动健康检查的节奏**，频率完全取决于部署配置。

#### API 服务
`apps/api/src/app/health/health.controller.ts:32-51`
```
checks = [
  dalHealthIndicator.isHealthy(),          // MongoDB connection.readyState === 1
  workflowQueueHealthIndicator.isHealthy(),// Redis + BullMQ 队列 isReady
  { apiVersion: { version, status: 'up' } }
]
// 条件性：如果 ELASTICACHE_CLUSTER_SERVICE_HOST 存在 → cacheHealthIndicator.isHealthy()
```

#### Worker 服务
`apps/worker/src/app/health/health.controller.ts:18-32`
```
checks = [
  dalHealthIndicator.isHealthy(),
  ...healthIndicators.map(indicator => indicator.isHealthy()), // 动态注入的所有队列指标
  { apiVersion: { version, status: 'up' } }
]
```
Worker 通过 `@Inject('QUEUE_HEALTH_INDICATORS')` 注入一组 `QueueHealthIndicator[]`，具体注册哪些取决于 SharedModule 的 Provider 配置。

#### WS 服务
`apps/ws/src/health/health.controller.ts:19-35`
```
checks = [
  ...indicators.map(indicator => indicator.isHealthy()), // 队列指标
  dalHealthIndicator.isHealthy(),                        // MongoDB
  wsServerHealthIndicator.isHealthy(),                   // WebSocket Gateway server 实例是否存在
  { apiVersion: { version, status: 'up' } }
]
```
WSServerHealthIndicator 检测逻辑：`!!this.wsGateway.server`（`apps/ws/src/socket/services/ws-server-health-indicator.service.ts:17`），仅验证 WS Gateway 实例已初始化，**不检测客户端连接数或消息吞吐**。

#### 队列健康指标基类的真实检测内容
`libs/application-generic/src/health/queue-health-indicator.service.ts`
```
isReady() = this.bullMqService.isClientReady()
```
底层调用 BullMQ 的 `isClientReady()`，一次性检查三个维度：
1. Redis 客户端连接状态
2. 队列是否暂停
3. Worker 是否运行

**关键发现：此检测只验证队列的「可达性」，不检测积压深度、消费延迟、错误率。** 即使队列积压 10 万条消息，只要 Redis 连接正常，健康检查仍返回 `up`。

### 1.2 集成级健康检查 — 按需手动触发

#### Email 集成凭证校验
`apps/api/src/app/integrations/usecases/check-integration/check-integration.usecase.ts:10-25`

- **仅支持 Email 渠道**，SMS/Push/Chat 没有实现 `CheckIntegration`
- **触发时机**：用户在 Dashboard 创建或更新集成时，如果 `check=true`
- **检测方式**：通过 `MailFactory` 创建 Handler 实例，调用 `mailHandler.check()`
- **错误处理**：捕获 `getaddrinfo ENOTFOUND` 抛出特定错误，其余一律 `BadRequestException`

**关键发现：这是「配置时」的一次性校验，不是「运行时」的周期性探测。保存集成后不会再自动复查。**

#### MS Teams 健康检查
`apps/api/src/app/integrations/usecases/msteams-health-check/msteams-health-check.usecase.ts`

- **触发方式**：前端按需调用 `GET /integrations/:id/msteams-health?checks=appRegistration,azureBotCreated,...`
- **支持按需选择检查点**：通过 `checks` query param 指定，未指定则全部执行
- **四个检查点的真实逻辑**：

| 检查点 | 实现方式 | 返回值语义 |
|--------|----------|-----------|
| `appRegistration` | 用 clientId/secretKey/tenantId 向 `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token` 发请求 | `ready` = token 获取成功 |
| `azureBotCreated` | 检查 Integration 实体的 `provisioning.azureBotResourceId` 是否存在 | `ready` = 字段非空（**不是网络探测**） |
| `teamsAppCatalog` | 用 Graph API 调用 `https://graph.microsoft.com/v1.0/appCatalogs/teamsApps?$filter=externalId eq '{clientId}'` | `ready` = 搜索结果非空 |
| `permissions` | 用 Graph API 调用 `https://graph.microsoft.com/v1.0/servicePrincipals(appId='{clientId}')?$select=id,appId,requiredResourceAccess` | `ready` = 有权限配置 |

**关键发现**：`azureBotCreated` 仅检查数据库字段，不是实际网络探测。如果 Azure 资源被手动删除但数据库未更新，此检查仍返回 `ready`。

### 1.3 Bridge 健康检查 — 前端轮询

`apps/dashboard/src/hooks/use-fetch-bridge-health-check.ts`

- **刷新间隔**：硬编码 `BRIDGE_STATUS_REFRESH_INTERVAL_IN_MS = 10 * 1000`（10秒）
- **触发条件**：仅当 `currentEnvironment.bridge.url` 存在时启用
- **检测内容**：向 Bridge URL 发请求，判断 `data.status === 'ok'`
- **状态判定**：

| 条件 | 状态 |
|------|------|
| isLoading | `LOADING` |
| bridgeURL 存在 + 无 error + status=ok | `CONNECTED` |
| 其余一切 | `DISCONNECTED` |

---

## 二、采样覆盖范围

### 2.1 发送失败的全量采样

`apps/worker/src/app/workflow/services/standard.worker.ts:52-58`

BullMQ Worker 注册了 `failed` 和 `completed` 事件监听器，**每一条任务的成功和失败都会被捕获**，不存在采样率的概念。

### 2.2 shouldBackoff 的覆盖范围 — 远比想象中窄

`apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts:721-723`
```typescript
public shouldBackoff(error: Error): boolean {
  return error?.message?.includes(EXCEPTION_MESSAGE_ON_WEBHOOK_FILTER);
}
```
`EXCEPTION_MESSAGE_ON_WEBHOOK_FILTER = 'Exception while performing webhook request.'`

**关键发现：只有错误消息包含「Exception while performing webhook request.」时才触发重试退避。这意味着：**
- Provider 返回 500/429/超时 → **不重试**，直接标记为 FAILED
- Webhook Filter 执行异常 → **重试**，最多 3 次
- 模板编译失败 → **不重试**
- 凭证过期 → **不重试**
- 网络超时 → **不重试**

### 2.3 重试仅针对 Webhook Filter 场景

Standard Worker 的 `DEFAULT_ATTEMPTS = 3`（`queue-base.service.ts:16`），但这个上限**仅对 Webhook Filter 错误有意义**，因为其他类型的错误 `hasToBackoff=false`，在第一次失败时就被 `shouldBeSetAsFailed=true` 终结。

### 2.4 SQS 消费路径的重试行为

`apps/worker/src/app/workflow/services/standard.worker.ts:64-88`

SQS 路径的重试行为与 BullMQ 不同：
- **Workflow Worker**：at-most-once，失败后 ack 丢弃（`return false`），SQS `RedrivePolicy.maxReceiveCount=1`
- **SubscriberProcess Worker**：at-most-once，同上
- **Standard Worker**：`jobHasFailed` 返回 `true` 时 re-throw → SQS 不删除消息 → visibility timeout 后重新投递；返回 `false` 时 ack

SQS 路径的重试节奏是**均匀的**（由 `SQS_DEFAULT_VISIBILITY_TIMEOUT` 控制），不是指数退避。

### 2.5 渠道覆盖矩阵

| 渠道 | 凭证校验（配置时） | 运行时重试 | 运行时主动探测 | 降级到 Novu Provider |
|------|:--:|:--:|:--:|:--:|
| Email | ✅ `CheckIntegration` | ❌ 仅 Webhook Filter | ❌ | ✅ |
| SMS | ❌ 无实现 | ❌ 仅 Webhook Filter | ❌ | ✅ |
| Push | ❌ 无实现 | ❌ 仅 Webhook Filter | ❌ | ❌ |
| Chat | ❌ 无实现 | ❌ 仅 Webhook Filter | ❌ | ✅ Slack only |
| In-App | N/A | ❌ 仅 Webhook Filter | ❌ | ❌ |
| MS Teams | ✅ 4维度健康检查 | ❌ 仅 Webhook Filter | ❌ | ❌ |

---

## 三、自动降级与恢复路径

### 3.1 队列后端降级（SQS → BullMQ）

`libs/application-generic/src/services/queues/queue-base.service.ts:186-264`

#### 降级触发条件
**每次入队操作**实时评估 `QUEUE_BACKEND_MODE` Feature Flag，降级是**请求级瞬时决策**，不是持久状态。

#### 各模式的降级和恢复路径

| 模式 | 正常路径 | 降级路径 | 自动恢复 |
|------|---------|---------|---------|
| `BULLMQ` | 全走 BullMQ | 无降级 | N/A |
| `SHADOW` | BullMQ 主 + SQS 影子（skipProcessing=true） | SQS 写入失败仅 warn，不影响主路径 | SQS 自愈后下次请求影子写入自然恢复 |
| `LIVE` | SQS 主 + BullMQ 影子（skipProcessing=true） | SQS 失败 → 降级到 BullMQ 主路径 | **无自动恢复**：当次请求已降级到 BullMQ，后续请求仍重新评估 Flag |
| `COMPLETE` | 全走 SQS | SQS 失败 → BullMQ | **无自动恢复**：同上 |

**关键发现**：
1. 降级是**请求级瞬时行为**，不是全局状态切换。SQS 短暂抖动只会影响出错的那条消息，下一条消息仍会先尝试 SQS。
2. 没有熔断器（Circuit Breaker）模式，不存在「连续 N 次失败后打开断路器」的机制。
3. `skipProcessing=true` 的影子消息会被 Worker 在入口处直接跳过（`standard.worker.ts:151-155`），仅用于验证消息能否成功入队和出队。
4. 延迟超过 SQS 最大延迟（15分钟 = `SQS_MAX_DELAY_SECONDS * 1000`）的任务无条件走 BullMQ，因为 SQS 不支持超过 15 分钟的延迟。

### 3.2 SQS DeleteMessage 的恢复逻辑

`libs/application-generic/src/services/sqs/sqs-consumer.service.ts:333-395`

消息处理成功后删除 SQS 消息时，如果 DeleteMessage 失败，有独立的三次重试机制：

```
DELETE_MAX_ATTEMPTS = 3
DELETE_BACKOFF_BASE_MS = 200
DELETE_BACKOFF_JITTER_FACTOR = 0.25
```

退避计算：`baseBackoff = 200 * 2^(attempt-1)`，加 `0~25%` 的 jitter。

**不可重试的错误**（立即放弃）：
- `ReceiptHandleIsInvalid`：handle 过期
- `InvalidParameterValue` / `InvalidAddress`：请求格式错误
- `AccessDenied` / `AccessDeniedException`：权限问题

**恢复路径**：如果三次 DeleteMessage 都失败，SQS 会在 visibility timeout 后重新投递消息，导致**重复处理**。代码用 `updateOne({ status: RUNNING }, { $set: { status: COMPLETED } })` 的条件更新来保证幂等性。

### 3.3 Novu 内置 Provider 降级

`libs/application-generic/src/usecases/get-novu-provider-credentials/get-novu-provider-credentials.usecase.ts`

**这不是运行时自动降级，而是集成选择链的最后一环。** `SelectIntegration` 在找不到用户自定义集成时，会返回 Novu 内置集成。这不是「主 Provider 失败后切到备用」，而是「没有配置主 Provider 时使用默认」。

#### 限额与恢复

| 渠道 | 默认限额 | 统计方式 | 恢复时机 |
|------|---------|---------|---------|
| Email | 300 条/月 | `messageRepository.count`，按自然月 | **月初自动重置** |
| SMS | 20 条/月 | 同上 | 同上 |
| Chat | 300 条/月 | 同上 | 同上 |

**额外约束**：当 `IS_TEST_PROVIDER_LIMITS_ENABLED=true` 时，Email 渠道只能发送给当前登录用户自己的邮箱，否则抛出 `ForbiddenException`。

### 3.4 发送失败后的恢复路径

完整的发送失败处理链路：

```
SendMessageEmail.send() 抛出异常
  → 返回 { status: 'failed', errorMessage: 'PROVIDER_ERROR' }
  → RunJob 捕获后：
     1. 标记 Job 为 FAILED
     2. 创建 stepRun 记录
     3. 判断 shouldHaltOnStepFailure：
        - Action Step（DELAY/DIGEST/THROTTLE）→ 总是 halt
        - 其他 Step → 看 step.shouldStopOnFail（默认 true）
     4. 如果 halt → 取消同 workflow 后续所有 PENDING job
     5. 如果不 halt → 继续排下一个 job
```

**关键发现：发送失败后没有任何自动切换 Provider 的逻辑。** 一条消息选择了一个 Integration 后，如果该 Integration 的 Provider 返回错误，消息直接失败，不会尝试同渠道的其他 Integration。

### 3.5 Kill Switch 的恢复路径

`apps/worker/src/app/workflow/services/standard.worker.ts:131-149`

Kill Switch 通过 `IS_ORG_KILLSWITCH_FLAG_ENABLED` Feature Flag 控制，作用于三个 Worker：
- `StandardWorker`：`standard.worker.ts:143`
- `WorkflowWorker`：`workflow.worker.ts:87-93`
- `SubscriberProcessWorker`：`subscriber-process.worker.ts:93-99`

**行为**：Kill Switch 启用时，Worker 消费消息后直接 return（不做业务处理），消息被 ack。

**恢复路径**：修改 LaunchDarkly 上的 Flag 即可，下次 Feature Flag SDK 刷新后生效。但**已 ack 丢弃的消息不会重发**，属于永久丢失。

---

## 四、运营侧与终端用户侧控制点的真实生效边界

### 4.1 运营侧控制点

#### 4.1.1 Feature Flags（LaunchDarkly）
**文件**：`packages/shared/src/types/feature-flags.ts`

| Flag | 控制范围 | 生效延迟 | 真实边界 |
|------|---------|---------|---------|
| `IS_ORG_KILLSWITCH_FLAG_ENABLED` | 组织级 | SDK 刷新周期 | 仅阻止新消息进入处理，已入队的消息仍会被消费（只是跳过业务逻辑后 ack） |
| `QUEUE_BACKEND_MODE` | 组织级 | SDK 刷新周期 | 仅控制**入队路由**，不影响已在 SQS/BullMQ 中的消息 |
| `IS_TEST_PROVIDER_LIMITS_ENABLED` | 用户级 | SDK 刷新周期 | 控制 Novu Email Provider 是否限制收件人必须是当前登录用户 |
| `IS_EMAIL_INLINE_CSS_DISABLED` | 环境级 | SDK 刷新周期 | 控制 Email 模板是否内联 CSS |
| `IS_DELIVERY_LIFECYCLE_TRANSITION_ENABLED` | 组织级 | SDK 刷新周期 | 控制工作流状态追踪方式（优化跳过无 action step 的中间更新） |

**Flag 评估的实际颗粒度**：大多数 Flag 传入了 `organization` 和 `environment`，可以做到组织级和环境级控制。但 `QUEUE_BACKEND_MODE` 的评估还读取了 `organization.apiServiceLevel`，所以同一个组织在不同 API 服务等级下可能有不同的默认值。

#### 4.1.2 集成管理 API
`apps/api/src/app/integrations/integrations.controller.ts`

| 操作 | 生效边界 |
|------|---------|
| 创建集成 | 立即生效，但新集成不会影响已在队列中的消息 |
| 更新集成（含凭证） | 立即生效，后续发送使用新凭证 |
| 删除集成 | 立即生效，后续发送找不到该集成时降级到 Novu Provider 或失败 |
| 设为主集成 | 立即生效，**但会禁用同渠道之前的主集成**（`set-integration-as-primary.usecase.ts`） |
| 更新集成 active=false | 立即生效，该集成不再被 `SelectIntegration` 选中 |

**主集成（Primary）机制的真实边界**：
- **仅 Email 和 SMS 渠道支持 Primary**（`CHANNELS_WITH_PRIMARY = [EMAIL, SMS]`，`packages/shared/src/types/channel.ts:68`）
- 切换 Primary 时，旧 Primary 自动设为 `active=false`（不是仅去掉 primary 标记）
- 如果同一渠道没有 Primary 且有多个 active 集成，`SelectIntegration` 返回**第一个匹配的**，顺序不可控

#### 4.1.3 集成条件过滤
`libs/application-generic/src/usecases/select-integration/select-integration.usecase.ts`

集成可配置 `conditions`（过滤规则），`SelectIntegration` 遍历集成列表时找到第一个 `passed=true` 的即返回。**这是按消息维度动态选择 Provider 的唯一机制**。

**生效边界**：
- 条件在每次发送时实时评估，修改立即生效
- 遍历顺序取决于数据库返回顺序，没有显式排序
- 如果所有集成的条件都不匹配，会走到 Novu 内置集成（如果存在）或返回 undefined（后续发送失败）

### 4.2 终端用户侧控制点

#### 4.2.1 触发时 Override

发送请求中的 `overrides` 字段可以在运行时动态选择集成：

```typescript
// send-message-email.usecase.ts:92
const overrideSelectedIntegration = command.overrides?.email?.integrationIdentifier;
```

**生效边界**：
- 仅支持 Email 渠道的 `integrationIdentifier` override
- 如果指定的 identifier 不存在，`SelectIntegration` 找不到匹配集成，发送失败
- **不支持**跨渠道降级（如 Email 失败自动切到 SMS）

#### 4.2.2 订阅者偏好

`send-message.usecase.ts:322-419`

每个发送前会评估订阅者偏好：

```
stepPreferred = workflowPreferred && channelPreferred
```

**生效边界**：
- 如果订阅者关闭了某渠道偏好，该步骤会被 SKIP（不是 FAILED）
- 偏好评估失败会抛出 `PlatformException`，导致任务异常终止
- 偏好评估是**每个消息实时查询**，修改后立即生效

#### 4.2.3 步骤条件过滤

`send-message.usecase.ts:228-261`

Workflow 步骤上的 `filters` 在发送时评估，不通过则 SKIP。

#### 4.2.4 步骤级 shouldStopOnFail

`apps/worker/src/app/shared/utils/should-halt-on-step-failure.ts`

```typescript
export const shouldHaltOnStepFailure = (job: JobEntity): boolean => {
  // Action steps（DELAY/DIGEST/THROTTLE）总是 halt
  if (job.type && isActionStepType(job.type)) return true;
  // 其他步骤看用户配置
  return job.step.shouldStopOnFail === true;
};
```

**生效边界**：
- `shouldStopOnFail` 默认值在代码中是 `undefined`，但在 `standard.worker.ts:259` 处理为：`jobEntity.step?.shouldStopOnFail === undefined ? true : job.step.shouldStopOnFail`
- 所以**默认行为是失败后停止整个 Workflow**，用户必须显式设置 `shouldStopOnFail=false` 才允许跳过失败继续
- 这仅影响 Workflow 内后续步骤是否执行，不影响当前步骤的重试逻辑

#### 4.2.5 Dashboard UI 控制点的实际作用

| UI 操作 | API 调用 | 真实效果 |
|---------|---------|---------|
| 集成列表页切换 active | `PUT /integrations/:id { active }` | 立即生效，集成不再被选中 |
| 设为主集成 | `POST /integrations/:id/set-primary` | 切换主集成，旧主集成自动 deactivated |
| 更新凭证 | `PUT /integrations/:id { credentials, check: true }` | 更新并可选校验，后续发送使用新凭证 |
| 删除集成 | `DELETE /integrations/:id` | 如果是 Primary，需先选择新的 Primary |
| MS Teams 健康检查 | `GET /integrations/:id/msteams-health` | 只读，不影响运行时行为 |
| Bridge 状态检测 | 前端 10s 轮询 Bridge URL | 只读，仅用于 UI 展示 |

**关键发现：Dashboard 没有提供「Provider 级健康看板」或「发送成功率监控」页面。** 运营人员只能通过 Activity Feed（执行详情记录）逐条查看发送结果，无法在 Dashboard 上看到某 Provider 的实时健康状态或错误率趋势。

---

## 五、关键代码位置速查表

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| API 健康检查端点 | `apps/api/src/app/health/health.controller.ts` | 32-51 |
| Worker 健康检查端点 | `apps/worker/src/app/health/health.controller.ts` | 18-32 |
| WS 健康检查端点 | `apps/ws/src/health/health.controller.ts` | 19-35 |
| 队列健康指标基类 | `libs/application-generic/src/health/queue-health-indicator.service.ts` | 全文件 |
| WS Server 健康指标 | `apps/ws/src/socket/services/ws-server-health-indicator.service.ts` | 16-25 |
| DB 健康指标 | `libs/application-generic/src/health/dal.health-indicator.ts` | 14-23 |
| shouldBackoff 判定 | `apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts` | 721-723 |
| Webhook Filter 异常常量 | `apps/worker/src/app/shared/utils/constants.ts` | 1 |
| shouldHaltOnStepFailure | `apps/worker/src/app/shared/utils/should-halt-on-step-failure.ts` | 4-17 |
| isActionStepType | `libs/application-generic/src/utils/digest.ts` | 24-28 |
| Standard Worker 失败处理 | `apps/worker/src/app/workflow/services/standard.worker.ts` | 230-285 |
| 指数退避策略 | `apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/` | 全文件 |
| SQS→BullMQ 降级路由 | `libs/application-generic/src/services/queues/queue-base.service.ts` | 186-264 |
| SQS Delete 重试 | `libs/application-generic/src/services/sqs/sqs-consumer.service.ts` | 333-395 |
| SQS Consumer pause/resume | `libs/application-generic/src/services/sqs/sqs-consumer.service.ts` | 447-467 |
| Novu Provider 凭证获取 | `libs/application-generic/src/usecases/get-novu-provider-credentials/` | 全文件 |
| Novu Provider 限额计算 | `libs/application-generic/src/usecases/calculate-limit-novu-integration/` | 全文件 |
| 集成选择逻辑 | `libs/application-generic/src/usecases/select-integration/` | 全文件 |
| CHANNELS_WITH_PRIMARY | `packages/shared/src/types/channel.ts` | 68 |
| Email 发送异常处理 | `apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts` | 535-584 |
| SendMessage 主入口 | `apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts` | 88-211 |
| Kill Switch 检查 | `apps/worker/src/app/workflow/services/standard.worker.ts` | 131-149 |
| HandleLastFailedJob | `apps/worker/src/app/workflow/usecases/handle-last-failed-job/` | 全文件 |
| Workflow Worker at-most-once | `apps/worker/src/app/workflow/services/workflow.worker.ts` | 50-66 |
| Feature Flag 枚举 | `packages/shared/src/types/feature-flags.ts` | 28-130 |
| Bridge 健康检查 hook | `apps/dashboard/src/hooks/use-fetch-bridge-health-check.ts` | 全文件 |
| MS Teams 健康检查 | `apps/api/src/app/integrations/usecases/msteams-health-check/` | 全文件 |
| Email 凭证校验 | `apps/api/src/app/integrations/usecases/check-integration/` | 全文件 |
| Dashboard 集成更新侧边栏 | `apps/dashboard/src/components/integrations/components/update-integration-sidebar.tsx` | 63-108 |
| Primary 集成选择弹窗 | `apps/dashboard/src/components/integrations/components/modals/select-primary-integration-modal.tsx` | 全文件 |
| RunJob 完整处理流程 | `apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts` | 84-420 |

---

## 六、总结：系统设计特点与不足

### 设计特点
1. **无状态降级**：队列后端降级是请求级瞬时决策，不维护全局健康状态，避免了状态同步的复杂性
2. **全量可观测**：所有发送事件都有 ExecutionDetails 记录，支持事后分析
3. **Feature Flag 驱动**：关键行为通过 LaunchDarkly 控制，支持灰度和紧急操作

### 不足
1. **无运行时 Provider 健康感知**：发送路径上没有任何 Provider 级别的健康状态缓存或熔断机制，每条消息独立尝试，Provider 持续故障时会产生大量无意义的失败请求
2. **重试范围极窄**：只有 Webhook Filter 异常才重试，Provider 错误（500/429/超时）直接失败，不重试也不降级到其他 Provider
3. **无跨 Provider 降级**：同渠道配置了多个 Integration 时，不会在主 Provider 失败后自动切换到备 Provider
4. **检查≠ 探测**：集成级健康检查（CheckIntegration / MS Teams）是配置时的一次性动作，不是运行时的周期性探测
5. **Kill Switch 消息丢失**：启用 Kill Switch 后消息被 ack 丢弃，无法恢复
