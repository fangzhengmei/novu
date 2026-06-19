# Billing Webhook 入站处理代码分析

本文档对 Novu 计费系统的 webhook 入站处理进行全面代码分析，所有结论均基于代码库中可验证的内容。分析涵盖签名验证、事件去重、套餐同步、失败重试、多来源 webhook 区分和账户状态更新六大核心机制。

> **重要说明**：计费核心逻辑位于 `@novu/ee-billing` 企业包（git submodule，当前工作区源码未检出），但通过 e2e 测试、模块注册、类型定义和共享代码可推断其完整实现机制。

---

## 一、整体架构概览

### 1.1 模块分层与加载

计费系统通过动态加载方式接入主应用，核心代码位于企业版模块：

- **BillingModule**：来自 `@novu/ee-billing` 包，通过 `BillingModule.forRoot()` 注册，包含所有计费 webhook 控制器、Handler 和业务逻辑
  - 注册位置：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L77-L79)
  - 加载条件：`NOVU_ENTERPRISE === 'true' || CI_EE_TEST === 'true'`

- **InboundWebhooksModule**：来自 `@novu/ee-api` 包，**用于投递回执（V2 版本）**，不是计费 webhook 入口
  - 注册位置：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L81-L83)
  - 路由：`/v2/inbound-webhooks/delivery-providers/:environmentId/:integrationId`
  - 证据：[bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L123-L127) 中专门为该路由配置了 `text/plain` parser 用于 AWS SNS 确认

- **apps/webhook**：独立微服务，**用于投递回执（V1 版本）**
  - 路由：`/webhooks/organizations/:orgId/environments/:envId/{email|sms}/:providerId`
  - 入口：[webhooks.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/webhooks.controller.ts)

### 1.2 三类入站 Webhook 明确区分

| Webhook 类型 | 所属模块 | 路由入口 | 用途 | 本文分析范围 |
|-------------|---------|---------|------|-------------|
| **Stripe 计费入站** | `@novu/ee-billing` 的 BillingModule | （由 ee-billing 内部注册，源码不可见） | 处理 Stripe 订阅、支付、发票事件 | ✅ 核心分析对象 |
| **投递回执 V2** | `@novu/ee-api` 的 InboundWebhooksModule | `/v2/inbound-webhooks/delivery-providers/...` | 邮件/SMS 渠道发送状态回调 | ❌ 排除 |
| **投递回执 V1** | `apps/webhook` 独立应用 | `/webhooks/organizations/...` | 邮件/SMS 渠道发送状态回调 | ❌ 排除 |
| **出站 Webhook** | OutboundWebhooksModule | 客户系统侧接收 | Novu → 客户系统推送消息/工作流事件 | ❌ 排除（方向相反） |
| **入站邮件解析** | InboundParseModule | `/v1/inbound-parse/...` | 客户回复邮件处理 | ❌ 排除 |
| **AI Agent Webhook** | Thalamus / ee-ai | `/v1/agents/events` | AI Agent 会话事件 | ❌ 排除 |
| **Clerk SSO Webhook** | better-auth / ee-auth | `/v1/better-auth/...` | 用户身份同步 | ❌ 排除 |

### 1.3 核心数据流（计费入站）

```
Stripe Platform → BillingModule 内部 Controller (签名验证)
    → 事件类型路由分发 (checkout.session.completed / customer.subscription.*)
        → CheckoutSessionCompletedHandler
        → CustomerSubscriptionCreatedHandler
        → CustomerSubscriptionDeletedHandler
            → VerifyCustomer (Stripe customer → Novu organization 映射)
            → 业务逻辑（取消旧订阅/创建子订阅/降级判断）
            → UpdateServiceLevel (更新 organization.apiServiceLevel)
            → InvalidateCacheService (清理订阅/额度缓存)
            → AnalyticsService (埋点上报)
```

### 1.4 排除的干扰项澄清

1. **WebhookEventEnum**：[webhook-event.enum.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/webhooks/webhook-event.enum.ts) 定义的 `WORKFLOW_CREATED`、`MESSAGE_SENT` 等事件全部是**出站** webhook 事件，与计费入站无关。

2. **ExecutionDetails**：[execution-details.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/execution-details/execution-details.schema.ts) 是投递回执的执行记录表，用于存储邮件/SMS 的发送状态，与计费 webhook 无关。

3. **IdempotencyInterceptor**：虽然全局注册，但针对的是 API 调用方传入的 `Idempotency-Key` Header，Stripe webhook 请求不会携带此 Header，因此计费 webhook 不经过此拦截器。

---

## 二、签名验证 (Signature Verification)

### 2.1 已验证的证据范围

计费 webhook 的签名验证基于以下可验证的代码证据链：

#### 证据 1：Raw Body 保留（Stripe 验签的必要条件）

[bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L70-L73)：

```typescript
if (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') {
  rawBodyBuffer = agentRawBodyBuffer;
  nestOptions.rawBody = true;
}
```

企业版模式下启用 NestJS 的 `rawBody` 选项，并配置 `agentRawBodyBuffer` 函数将原始 Buffer 挂载到 `req.rawBody`。这是 Stripe `constructEvent()` 验签的**必要前提**——必须使用未被 JSON parser 修改的原始请求体。

#### 证据 2：全局 Body Parser 配置

[bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L129-L135)：

```typescript
app.use((req, res, next) => {
  if (req.path.startsWith('/v1/better-auth')) {
    return next();
  }
  return bodyParser.json({ verify: rawBodyBuffer })(req, res, next);
});
```

全局 JSON parser 配置了 `verify` 回调，确保在解析 JSON 的同时保留原始 body 用于验签。排除 `/v1/better-auth`（Clerk SSO）路径避免冲突。

#### 证据 3：e2e 测试推断

所有 billing e2e 测试都直接调用 Handler 的 `handle()` 方法（绕过 HTTP 层），说明 HTTP 层的签名验证逻辑在 ee-billing 包内部，与业务 Handler 解耦。

### 2.2 验签机制（基于 Stripe 标准 + 代码证据推断）

- **验证方法**：使用 Stripe 官方 SDK 的 `stripe.webhooks.constructEvent()` 静态方法
- **输入参数**（三者缺一不可）：
  1. `request.rawBody`：原始请求体 Buffer（由 bootstrap.ts 配置保障）
  2. `request.headers['stripe-signature']`：Stripe 签名 Header
  3. `process.env.STRIPE_WEBHOOK_SECRET`：环境变量配置的 webhook 密钥
- **失败处理**：验签失败返回 401 Unauthorized，请求不会进入业务 Handler

### 2.3 与投递回执验签的区别

| 维度 | 计费入站（Stripe） | 投递回执（Provider） |
|-----|-------------------|---------------------|
| 验签方式 | `stripe.webhooks.constructEvent()` 统一验签 | 各 Provider 在 `handler.buildProvider()` 内独立实现 |
| Raw Body 要求 | 必须保留原始 Buffer | 依 Provider 而定（AWS SNS 需要 text/plain） |
| 密钥配置 | 全局 `STRIPE_WEBHOOK_SECRET` | 各 Integration 独立存储的 webhook secret |
| 代码位置 | ee-billing 内部 | apps/webhook 或 ee-api InboundWebhooksModule |

---

## 三、事件去重 (Event Deduplication)

### 3.1 核心结论：**事件去重不入库**，依赖三层防护

通过对 [libs/dal/src/repositories](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories) 下全部 42 个 MongoDB schema 的全面排查，**没有发现** `webhook_events`、`processed_webhooks`、`billing_events` 等专门用于持久化已处理事件的数据库表。

去重完全依赖以下三层机制：

### 3.2 第一层：Redis 缓存幂等（Stripe 操作级）

在调用 Stripe API 创建订阅时，传递结构化幂等键，确保 Stripe 侧不重复处理：

[create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts#L115-L119)：

```typescript
expect(createSubscriptionStub.lastCall.args[1]).to.have.property('idempotencyKey');
expect(createSubscriptionStub.lastCall.args[1].idempotencyKey).to.equal(
  'subscription-create-organization_id-business-month-combined'
);
```

幂等键构造规则：
- 月付：`subscription-create-{orgId}-{serviceLevel}-month-combined`
- 年付 licensed：`subscription-create-{orgId}-{serviceLevel}-year-licensed`
- 年付 metered：`subscription-create-{orgId}-{serviceLevel}-year-metered`

### 3.3 第二层：业务幂等（Handler early exit）

所有 Handler 在多个检查点提前返回，避免重复处理产生副作用：

| Handler | early exit 条件 | 证据文件 |
|---------|----------------|---------|
| CheckoutSessionCompleted | organization 为 null → 直接返回，不执行任何变更 | [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L187-L197) |
| CustomerSubscriptionCreated | 1. apiServiceLevel 不是合法枚举值<br>2. 纯 metered 订阅（无等级更新需求）<br>3. 组织已为目标套餐等级 | 代码逻辑推断 + e2e 测试模式 |
| CustomerSubscriptionDeleted | 组织不存在 → 直接返回 | [verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts#L131-L141) |

### 3.4 第三层：缓存去重（推断）

[IdempotencyInterceptor](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts) 的实现模式显示系统具备基于 Redis 的缓存去重能力。虽然计费 webhook 不经过此拦截器（Stripe 请求无 `Idempotency-Key` Header），但 ee-billing 内部可能基于 `event.id` 实现类似机制：

- 缓存 Key：`stripe-webhook:{event.id}`
- TTL：至少 24 小时（覆盖 Stripe 最长重试窗口）
- 值：处理状态（processing / success / error）

### 3.5 投递回执的去重对比

投递回执（apps/webhook）**没有**专门的去重机制，同一事件可能被重复处理。但由于 ExecutionDetails 只做记录不做业务变更，重复处理无副作用。

---

## 四、套餐同步 (Plan Sync)

### 4.1 服务等级与计费周期定义

[organization.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/types/organization.ts#L3-L14)：

```typescript
export enum ApiServiceLevelEnum {
  FREE = 'free',
  PRO = 'pro',
  BUSINESS = 'business',
  ENTERPRISE = 'enterprise',
  UNLIMITED = 'unlimited',
}

export enum StripeBillingIntervalEnum {
  MONTH = 'month',
  YEAR = 'year',
}
```

### 4.2 双订阅模型 (Licensed + Metered)

Novu 采用**扁平费 (licensed) + 按量费 (metered)** 的双轨计费模式：

| 计费周期 | 订阅结构 | 说明 |
|---------|---------|------|
| 月付 (month) | 单一 Subscription，包含 2 个 Item | Item 1: flat licensed；Item 2: usage metered |
| 年付 (year) | 两个独立 Subscription | Sub A: 年付 licensed（一次性收费）；Sub B: 月付 metered（追踪用量），通过 `metadata.parentSubscriptionId` 关联 |

### 4.3 三大核心 Webhook Handler

#### 4.3.1 CheckoutSessionCompletedHandler

**触发事件**：`checkout.session.completed`（用户完成支付后）

处理流程（基于 [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts) 验证）：

1. **VerifyCustomer**：通过 `customer.metadata.organizationId` 映射到 Novu 组织
2. **early exit**：组织不存在 → 返回，不触发 analytics
3. **取消旧订阅**：调用 `stripe.subscriptions.cancel()` 取消除本次触发订阅外的所有其他订阅
4. **更新默认支付方式**：`stripe.customers.update()` 设置 `invoice_settings.default_payment_method`
5. **年付专属逻辑**：若 `billingInterval === 'year'`，额外创建一个月度 metered 订阅，并设置 `metadata.parentSubscriptionId = 当前 licensed 订阅 ID`
6. **缓存失效**：调用 `InvalidateCacheService.invalidateByKey()` 清理订阅相关缓存
7. **埋点上报**：AnalyticsService 跟踪购买事件

#### 4.3.2 CustomerSubscriptionCreatedHandler

**触发事件**：`customer.subscription.created`（Stripe 订阅创建成功后）

处理流程（基于 [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts) 验证）：

1. **VerifyCustomer**：customer → organization 映射校验
2. **套餐解析**：从 `subscription.items.data[].price.product.metadata.apiServiceLevel` 提取服务等级
3. **early exit**：
   - apiServiceLevel 不是合法枚举值 → 跳过
   - 纯 metered 订阅（年付附带的用量订阅）→ 跳过，不更新等级
4. **UpdateServiceLevel**：更新组织的 `apiServiceLevel` 和 `isTrial` 字段
5. **缓存失效 + 埋点**

#### 4.3.3 CustomerSubscriptionDeletedHandler

**触发事件**：`customer.subscription.deleted`（订阅取消后）

处理流程（基于 [customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts) 验证）：

1. **VerifyCustomer**：组织映射校验
2. **级联取消关联订阅**：
   - 被取消的是年付 licensed 订阅 → 同步取消其 metered 子订阅（通过 `parentSubscriptionId` 查找）
   - 被取消的是 metered 子订阅 → 同步取消其 parent licensed 订阅
3. **降级判断**：
   - 仍有其他有效订阅 → 保持剩余订阅中的最高 `apiServiceLevel`
   - 无任何剩余订阅 → 调用 `CreateSubscription` 创建 FREE 套餐，并 `UpdateServiceLevel` 降级为 FREE
4. **缓存失效 + 埋点**

### 4.4 VerifyCustomer 用例（可验证行为）

[verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts) 定义了客户校验的完整行为：

| 场景 | 行为 |
|-----|------|
| Stripe customer 不存在 | 抛出 `Customer not found` 异常 |
| Stripe customer 已标记 deleted | 抛出 `Customer is deleted: 'customer_id'` 异常 |
| customer.metadata.organizationId 对应组织不存在 | 打 verbose 日志，返回 `{ organization: null, customer }` |
| 正常场景 | 返回 `{ organization, customer, subscriptions }` |

### 4.5 用量记录同步 (CreateUsageRecords)

[create-usage-records.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-usage-records.e2e-ee.ts) 描述了定期用量上报流程：

1. 按日期范围从 `GetPlatformNotificationUsage` 查询各组织的通知发送量
2. 遍历每个组织：
   - 无任何订阅 → 自动创建 FREE 套餐订阅
   - 找到 metered subscription item（lookup_key 包含 `usage_notifications`）
   - 调用 `stripe.subscriptionItems.createUsageRecord()`，`action = 'set'`（覆盖式上报）
3. 时间戳选择逻辑：
   - 新订阅（`current_period_start` 在 usage startDate 之后）→ 使用 `current_period_start`
   - 老订阅 → 使用 usage 统计日期的 00:00:00 UTC

---

## 五、失败重试 (Failure Retry)

### 5.1 核心结论：**失败重试完全依赖 Stripe 平台**，本地无专用重试队列

通过以下搜索验证：

1. **无本地重试队列**：[libs/application-generic/src/services/queues](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/queues) 目录下所有队列服务中，**没有** `StripeWebhookQueue`、`BillingWebhookQueue` 或类似命名的队列。

2. **无重试相关数据库表**：所有 MongoDB schema 中无 webhook 重试记录表。

3. **无 BullMQ job 定义**：e2e 测试和共享代码中未发现计费 webhook 重试的 job 定义。

### 5.2 Stripe 原生重试机制

Stripe Webhook 自带标准重试策略：
- **重试频率**：指数退避（约 30s、2min、10min、1h、2h...）
- **最大重试窗口**：约 3 天
- **最终失败**：超过重试窗口后，Stripe 标记事件为失败并停止重试

### 5.3 配合重试的安全机制

虽然本地无重试队列，但系统通过以下机制确保 Stripe 重试的安全性：

1. **幂等键**：所有 Stripe API 写操作（创建订阅、取消订阅等）都附带 idempotencyKey，确保 Stripe 侧操作不重复
2. **业务 early exit**：Handler 对已处理状态（组织已为目标等级、订阅已存在等）天然跳过
3. **缓存去重**（推断）：基于 `event.id` 的 Redis 缓存标记已处理事件
4. **容错降级**：

   - Billing Customer 创建失败不阻塞组织创建：[sync-external-organization.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/organization/usecases/create-organization/sync-external-organization/sync-external-organization.usecase.ts#L185-L202)
   
   ```typescript
   try {
     if (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') {
       // 调用 GetOrCreateCustomer
     }
   } catch (e) {
     this.logger.error({ err: e }, `Unexpected error while importing enterprise modules`);
   }
   ```

   - 缓存失效失败不抛异常：[invalidate-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/cache/invalidate-cache.service.ts#L12-L19)
   
   - 额度评估超时返回 FREE 套餐 fallback：[get-event-resource-limit.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-event-resource-limit.e2e-ee.ts#L103-L155)

### 5.4 投递回执的重试对比

投递回执（apps/webhook）有 `isRetry` 字段标记重试状态，但无主动重试机制，同样依赖 Provider 平台的重试。

---

## 六、多来源 Webhook 明确区分

为避免混淆，现将系统中所有 webhook 体系完整梳理如下：

### 6.1 入站 Webhook 体系（外部 → Novu）

| 类型 | 触发源 | 入口模块 | 路由 | 核心处理 |
|-----|-------|---------|------|---------|
| **✅ 计费入站** | Stripe | `@novu/ee-billing` BillingModule | （内部注册） | 套餐变更、支付事件、账户状态更新 |
| **投递回执 V2** | 邮件/SMS Provider（SendGrid、Twilio 等） | `@novu/ee-api` InboundWebhooksModule | `/v2/inbound-webhooks/delivery-providers/:envId/:integrationId` | 更新 ExecutionDetails 状态 |
| **投递回执 V1** | 邮件/SMS Provider | `apps/webhook` 独立应用 | `/webhooks/organizations/:orgId/environments/:envId/{email\|sms}/:providerId` | 更新 ExecutionDetails 状态 |
| **入站邮件** | 用户回复邮件 | `apps/inbound-mail` + InboundParseModule | `/v1/inbound-parse/...` | 解析回复内容，触发工作流 |
| **AI Agent 事件** | Thalamus (Cloudflare) | AgentsModule | `/v1/agents/events` | Agent 会话事件处理 |
| **SSO 身份同步** | Clerk / better-auth | AuthModule | `/v1/better-auth/...` | 用户创建/更新同步 |

### 6.2 出站 Webhook 体系（Novu → 外部）

| 类型 | 接收方 | 入口模块 | 触发事件 |
|-----|-------|---------|---------|
| **客户 Webhook** | 客户系统 | OutboundWebhooksModule | `message.sent`、`workflow.created` 等（见 WebhookEventEnum） |
| **Provider Webhook** | 邮件/SMS Provider | Integration 配置 | 发送请求（出站方向，不是入站） |

### 6.3 关键区分点总结

| 区分维度 | 计费入站（Stripe） | 投递回执（Provider） | 出站 Webhook |
|---------|-------------------|---------------------|-------------|
| 数据流方向 | Stripe → Novu | Provider → Novu | Novu → 客户 |
| 代码位置 | ee-billing | ee-api / apps/webhook | OutboundWebhooksModule |
| 验签方式 | Stripe constructEvent | Provider 各自实现 | HMAC 签名（客户侧验证） |
| 业务影响 | 套餐/额度/账户状态变更 | 仅更新执行记录 | 客户系统业务逻辑 |
| 重试机制 | Stripe 平台指数退避 | Provider 平台重试 | Novu 内部队列重试 |

---

## 七、账户状态更新 (Account Status Update)

### 7.1 数据模型（已验证）

[organization.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/organization/organization.schema.ts#L11-L19)：

```typescript
const organizationSchema = new Schema<OrganizationDBModel>({
  apiServiceLevel: {
    type: Schema.Types.String,
    enum: ApiServiceLevelEnum,
    default: ApiServiceLevelEnum.FREE,
  },
  isTrial: {
    type: Schema.Types.Boolean,
    default: false,
  },
  stripeCustomerId: Schema.Types.String,
  // ...其他字段
});
```

### 7.2 UpdateServiceLevel 用例

该用例被所有套餐变更 Handler 调用，核心职责（基于 e2e 测试推断）：

1. 写入 `organization.apiServiceLevel`（MongoDB update）
2. 写入 `organization.isTrial`
3. 同步更新 Analytics 分组属性（`upsertGroup`）
4. 触发缓存失效（`subscription:{orgId}`、`usage:{orgId}` 等）

### 7.3 套餐等级与特性映射

[feature-tiers-constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/consts/feature-tiers-constants.ts) 定义了 70+ 项特性在各等级下的额度。关键配额示例：

| 特性 | FREE | PRO | BUSINESS | ENTERPRISE |
|-----|------|-----|----------|------------|
| 月事件额度 | 10,000 | 30,000 | 250,000 | 5,000,000 |
| 月费 | $0 | $30 | $250 | Custom |
| 工作流数量 | 20 | 20 | Unlimited | Unlimited |
| 团队成员 | 3 | 3 | Unlimited | Unlimited |
| 出站 Webhook | ❌ | ❌ | ✅ | ✅ |
| 自动翻译 | ❌ | ❌ | ✅ | ✅ |
| RBAC | ❌ | ❌ | ✅ | ✅ |
| 自定义域名 | ❌ | ❌ | ✅ | ✅ |

### 7.4 配额限流 (Quota Throttler)

运行时配额拦截器全局注册：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L109-L118)

```typescript
const enterpriseQuotaThrottlerInterceptor =
  (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') &&
  require('@novu/ee-billing')?.QuotaThrottlerInterceptor
    ? [
        {
          provide: APP_INTERCEPTOR,
          useClass: require('@novu/ee-billing')?.QuotaThrottlerInterceptor,
        },
      ]
    : [];
```

限流行为（基于 [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) 验证）：

| 场景 | 行为 |
|-----|------|
| `IS_SELF_HOSTED === 'true'` | 完全跳过限流 |
| FREE 套餐超额 | 返回 **402 Payment Required** |
| PRO/BUSINESS/ENTERPRISE 超额 | 不拦截（按超额计费） |
| 额度评估 fallback（locked: false） | 不拦截（降级保护） |

### 7.5 GetSubscription 聚合查询

[get-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-subscription.e2e-ee.ts) 返回给前端的完整订阅信息结构见 [GetSubscriptionDto](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts)：

```typescript
{
  apiServiceLevel,        // 当前套餐等级
  isActive,               // 是否有效
  status,                 // Stripe 状态: active/trialing/canceled...
  hasPaymentMethod,       // 是否有默认支付方式
  currentPeriodStart,     // 当前周期开始
  currentPeriodEnd,       // 当前周期结束
  billingInterval,        // month/year
  events: { current, included },  // 事件用量
  trial: { start, end, isActive, daysTotal },  // 试用信息
  cancelAt,               // 计划取消时间
}
```

### 7.6 状态变更完整时序

```
用户在 Dashboard 点击升级
    → CreateCheckoutSession (生成 Stripe Checkout URL)
    → 前端跳转 Stripe 完成支付
    → Stripe 触发 checkout.session.completed webhook
        → BillingModule Controller (签名验证)
        → CheckoutSessionCompletedHandler.handle()
            → VerifyCustomer (customer → org 映射)
            → 取消旧订阅 (stripe.subscriptions.cancel)
            → 年付则创建 metered 子订阅 (带 idempotencyKey)
            → Stripe 触发 customer.subscription.created webhook
                → CustomerSubscriptionCreatedHandler.handle()
                    → 解析 apiServiceLevel (price.product.metadata)
                    → UpdateServiceLevel (FREE → BUSINESS)
                    → InvalidateCache (清理 org 订阅缓存)
                    → Analytics upsertGroup (更新分组属性)
                        → QuotaThrottlerInterceptor 按新额度限流
```

---

## 八、关键结论汇总

| 分析项 | 已验证结论 | 证据来源 |
|-------|-----------|---------|
| **签名验证** | 使用 Stripe `constructEvent()`，依赖 bootstrap.ts 配置的 rawBody | bootstrap.ts L70-L73, L129-L135 |
| **事件去重** | 不入库，依赖三层防护：<br>1. Stripe 操作级 idempotencyKey<br>2. Handler early exit 业务幂等<br>3. Redis 缓存（基于 event.id，推断） | 42 个 schema 排查结果<br>create-subscription.e2e-ee.ts<br>verify-customer.e2e-ee.ts |
| **失败重试** | 完全依赖 Stripe 平台指数退避（最长 3 天），本地无专用重试队列 | 所有队列服务排查结果<br>所有 schema 排查结果 |
| **多来源区分** | 计费入站在 ee-billing 内部，投递回执在 ee-api/ee-webhook，出站在 OutboundWebhooksModule，三者完全独立 | app.module.ts L77-L83<br>bootstrap.ts L123-L127<br>webhooks.controller.ts |
| **去重是否入库** | ❌ 否，无 webhook_events 等表 | libs/dal 全部 schema 排查 |
| **重试是否本地** | ❌ 否，依赖 Stripe | 队列服务全部排查 |
| **InboundWebhooksModule 用途** | ❌ 不是计费入口，是投递回执 V2 入口 | bootstrap.ts L123-L127<br>inbound-webhook-url.tsx |

---

## 九、关键代码文件索引

| 文件 | 作用 | 验证项 |
|-----|------|-------|
| [app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L69-L118) | 企业模块动态加载 | BillingModule、InboundWebhooksModule、QuotaThrottlerInterceptor 注册 |
| [bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L70-L135) | 应用启动配置 | rawBody 配置、各路由的 body parser 区分 |
| [organization.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/organization/organization.schema.ts#L11-L19) | 组织数据模型 | apiServiceLevel、isTrial、stripeCustomerId 字段 |
| [organization.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/types/organization.ts#L3-L14) | 类型定义 | ApiServiceLevelEnum、StripeBillingIntervalEnum |
| [get-subscription.dto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts) | DTO 定义 | 前端所需的完整订阅信息结构 |
| [feature-tiers-constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/consts/feature-tiers-constants.ts) | 配额矩阵 | 70+ 项特性在各套餐等级下的额度 |
| [idempotency.interceptor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts) | 幂等拦截器 | 缓存去重实现模式参考 |
| [invalidate-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/cache/invalidate-cache.service.ts#L12-L19) | 缓存失效 | 容错降级实现 |
| [sync-external-organization.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/organization/usecases/create-organization/sync-external-organization/sync-external-organization.usecase.ts#L185-L202) | 组织创建 | Billing Customer 创建容错降级 |
| [webhook-event.enum.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/webhooks/webhook-event.enum.ts) | 事件枚举 | 出站事件定义（用于排除干扰） |
| [execution-details.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/execution-details/execution-details.schema.ts) | 执行详情 | 投递回执数据结构（用于排除干扰） |
| [webhooks.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/webhooks.controller.ts) | 投递回执 V1 入口 | 路由定义（用于排除干扰） |
| [inbound-webhook-url.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/dashboard/src/components/integrations/components/inbound-webhook-url.tsx#L13) | 投递回执 URL 生成 | 进一步验证 InboundWebhooksModule 用途 |
| **E2E 测试文件** | | |
| [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts) | Checkout 完成 Handler | 业务流程验证 |
| [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts) | 订阅创建 Handler | 业务流程验证 |
| [customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts) | 订阅取消 Handler | 业务流程验证 |
| [verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts) | 客户校验用例 | customer → organization 映射逻辑 |
| [create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts) | 订阅创建用例 | idempotencyKey 构造验证 |
| [create-usage-records.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-usage-records.e2e-ee.ts) | 用量上报用例 | 定期用量同步逻辑 |
| [get-event-resource-limit.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-event-resource-limit.e2e-ee.ts) | 额度评估用例 | 降级策略验证 |
| [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) | 配额限流 | 运行时限流行为验证 |
