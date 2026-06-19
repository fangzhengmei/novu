# Billing Webhook 入站处理代码分析

本文档对 Novu 计费系统的 webhook 入站处理进行全面代码分析，涵盖签名验证、事件去重、套餐同步、失败重试、多来源 webhook 和账户状态更新六大核心机制。

---

## 一、整体架构概览

### 1.1 模块分层

计费系统的核心逻辑位于企业版模块 `@novu/ee-billing`，通过动态加载方式接入主应用：

- **入口注册**：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L69-L107) 中 `enterpriseImports()` 函数根据 `NOVU_ENTERPRISE` 环境变量条件加载 `BillingModule.forRoot()`
- **入站 webhook 入口**：`@novu/ee-api` 包中的 `InboundWebhooksModule` 负责接收外部 webhook 请求
- **通用 webhook 服务**：[apps/webhook](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook) 应用处理邮件/SMS 渠道提供商的回调 webhook

### 1.2 核心数据流

```
Stripe Webhook → InboundWebhooksModule (签名验证)
    → 事件路由分发
        → CheckoutSessionCompletedHandler
        → CustomerSubscriptionCreatedHandler
        → CustomerSubscriptionDeletedHandler
        → ...其他事件处理器
            → VerifyCustomer (组织映射)
            → UpdateServiceLevel (套餐更新)
            → InvalidateCacheService (缓存清理)
            → AnalyticsService (埋点上报)
```

---

## 二、签名验证 (Signature Verification)

### 2.1 Stripe 标准签名验证机制

虽然 `@novu/ee-billing` 的具体实现位于企业 submodule（当前工作区未检出），但从 e2e 测试和 Stripe SDK 使用模式可以推断：

- **验证方式**：使用 Stripe 官方 SDK 的 `stripe.webhooks.constructEvent()` 方法
- **所需参数**：
  - 请求原始 Body（必须是原始字符串，不能是 JSON parse 后的对象）
  - HTTP Header `stripe-signature`
  - 环境变量中配置的 Webhook Secret Key

### 2.2 安全要点

从 `apps/webhook` 的通用处理模式可见一斑：[webhook.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts)

- Provider 级别的签名验证由各邮件/SMS Provider 的 `handler.buildProvider()` 在内部完成
- Billing webhook 走独立的 `InboundWebhooksModule` 入口，使用 Stripe 专属签名验证

---

## 三、事件去重 (Event Deduplication)

系统采用**多层级、多策略**的去重方案：

### 3.1 API 层全局幂等拦截器

[IdempotencyInterceptor](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts) 为所有 POST/PATCH 请求提供通用幂等保护：

```typescript
// 关键配置常量
const IDEMPOTENCY_CACHE_TTL = 60 * 60 * 24;   // 结果缓存 24 小时
const IDEMPOTENCY_PROGRESS_TTL = 60 * 5;      // 处理中标记 5 分钟
const ALLOWED_METHODS = ['post', 'patch'];
```

**工作流程**：
1. 从请求 Header 读取 `Idempotency-Key`
2. 构造缓存 Key：`${env}-${organizationId}-${idempotencyKey}`
3. 对 Body 做 `blake2s256` 哈希用于校验请求体一致性
4. 使用 `cacheService.setIfNotExist()` 原子写入 "in-progress" 状态
5. 三种返回状态：
   - 首次请求：执行处理器 → 缓存结果（SUCCESS/ERROR）
   - 正在处理中：返回 409 Conflict + `Retry-After: 1`
   - 请求体不一致：返回 422 Unprocessable Entity
   - 重复请求：返回缓存结果 + Header `Idempotency-Replay: true`

### 3.2 Stripe 操作级幂等 Key

在 [create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts#L115-L207) 中，创建 Stripe Subscription 时传递了结构化的幂等键：

```
月度订阅：  subscription-create-{organizationId}-{apiServiceLevel}-month-combined
年度订阅：  subscription-create-{organizationId}-{apiServiceLevel}-year-licensed
            subscription-create-{organizationId}-{apiServiceLevel}-year-metered
```

这种构造方式确保：
- 同一组织同一套餐的创建操作不会重复扣费
- 年度订阅拆分为两个独立订阅（licensed 年付 + metered 月付），各自拥有独立幂等键

### 3.3 Stripe 事件 ID 级去重

Stripe 官方推荐使用事件对象的 `event.id` 字段做业务去重。结合 ee-billing Handler 的 early exit 模式（如组织不存在时直接返回），系统在业务逻辑层也具备天然的防重复处理能力。

---

## 四、套餐同步 (Plan Sync)

### 4.1 服务等级定义

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
| 年付 (year) | 两个独立 Subscription | Sub A: 年付 licensed（一次性收费）；Sub B: 月付 metered（追踪用量）|

年付模式下，两个订阅通过 `metadata.parentSubscriptionId` 建立关联，参考 [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L235-L241)。

### 4.3 Price Lookup Key 映射

[get-prices.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-prices.e2e-ee.ts#L38-L95) 定义了 lookup_key 到套餐等级的映射：

```
FREE      → free_flat_monthly, free_usage_notifications_10k
PRO       → pro_flat_monthly/annually, pro_usage_notifications
BUSINESS  → business_flat_monthly/annually, business_usage_notifications
ENTERPRISE → enterprise_flat_monthly/annually, enterprise_usage_notifications
```

### 4.4 三大核心 Webhook Handler

#### 4.4.1 CheckoutSessionCompletedHandler

触发事件：`checkout.session.completed`

处理流程（见 [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts)）：

1. **VerifyCustomer**：通过 Stripe customer.metadata.organizationId 映射到 Novu 组织，组织不存在则 early exit
2. **清理旧订阅**：取消除当前事件触发的订阅外的所有其他订阅（`stripe.subscriptions.cancel`）
3. **更新默认支付方式**：`stripe.customers.update` 设置 `invoice_settings.default_payment_method`
4. **年付专属逻辑**：若为年付，额外创建一个**月度 metered 订阅**，并通过 `metadata.parentSubscriptionId` 关联到年付 licensed 订阅
5. **缓存失效**：调用 `InvalidateCacheService.invalidateByKey()` 清理订阅相关缓存
6. **埋点上报**：AnalyticsService 跟踪购买事件

#### 4.4.2 CustomerSubscriptionCreatedHandler

触发事件：`customer.subscription.created`

处理流程（见 [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts)）：

1. **VerifyCustomer**：customer → organization 映射校验
2. **套餐解析**：从 `subscription.items.data[].price.product.metadata.apiServiceLevel` 提取服务等级
3. **类型校验**：
   - 仅对 licensed 类型订阅触发 `UpdateServiceLevel`
   - 对纯 metered 订阅（如年付附带的月度用量订阅）跳过等级更新
   - `apiServiceLevel` 必须是合法的枚举值，否则 early exit
4. **UpdateServiceLevel**：更新组织的 `apiServiceLevel` 和 `isTrial` 字段
5. **缓存失效 + 埋点**

#### 4.4.3 CustomerSubscriptionDeletedHandler

触发事件：`customer.subscription.deleted`

处理流程（见 [customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts)）：

1. **VerifyCustomer**：组织映射校验
2. **级联取消关联订阅**：
   - 被取消的是年付 licensed 订阅 → 同步取消其 metered 子订阅（通过 `parentSubscriptionId` 查找）
   - 被取消的是 metered 子订阅 → 同步取消其 parent licensed 订阅
3. **降级判断**：
   - 仍有其他有效订阅 → 保持剩余订阅中的最高 `apiServiceLevel`
   - 无任何剩余订阅 → 调用 `CreateSubscription` 创建 **FREE 套餐**，并 `UpdateServiceLevel` 降级为 FREE
4. **缓存失效 + 埋点**

### 4.5 VerifyCustomer 用例

[verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts) 定义了客户校验的关键行为：

- Stripe customer 不存在 → 抛出异常 `Customer not found`
- Stripe customer 已标记 deleted → 抛出异常 `Customer is deleted`
- customer.metadata.organizationId 对应的 Novu 组织不存在 → 打 verbose 日志，返回 `organization: null`，允许上层 Handler 做 early exit

### 4.6 用量记录同步 (CreateUsageRecords)

[create-usage-records.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-usage-records.e2e-ee.ts) 描述了定期用量上报流程：

1. 按日期范围从 `GetPlatformNotificationUsage` 查询各组织的通知发送量
2. 遍历每个组织：
   - 无任何订阅 → 自动创建 FREE 套餐订阅
   - 找到 metered subscription item（lookup_key 包含 `usage_notifications`）
   - 调用 `stripe.subscriptionItems.createUsageRecord()`，action=`set`（覆盖式上报）
3. 时间戳选择逻辑：
   - 新订阅（`current_period_start` 在 usage startDate 之后）→ 使用 `current_period_start`
   - 老订阅 → 使用 usage 统计日期的 00:00:00 UTC

---

## 五、失败重试 (Failure Retry)

### 5.1 Stripe Webhook 自动重试

Stripe Webhook 自带标准重试策略（指数退避，最长约 3 天），Novu 通过以下机制确保重试安全：

- **幂等键**：所有 Stripe API 写操作都附带 idempotencyKey
- **业务 early exit**：Handler 对已处理状态（如组织已为目标套餐等级）天然跳过
- **事件 ID 去重**（推断）：Stripe event.id 做处理标记

### 5.2 组织创建时的 Billing 容错

[sync-external-organization.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/organization/usecases/create-organization/sync-external-organization/sync-external-organization.usecase.ts#L185-L202)：

```typescript
private async createCustomer(billingEmail: string, organizationId: string) {
  try {
    if (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') {
      // ...调用 GetOrCreateCustomer
    }
  } catch (e) {
    this.logger.error({ err: e }, `Unexpected error while importing enterprise modules`);
  }
}
```

- Billing Customer 创建失败不阻塞组织创建流程
- 使用 try/catch 静默降级，仅记录错误日志

### 5.3 缓存失效容错

[invalidate-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/cache/invalidate-cache.service.ts#L12-L19)：

```typescript
public async invalidateByKey({ key }: { key: string }): Promise<number> {
  try {
    return await this.cacheService.del(key);
  } catch (err) {
    Logger.error(err, `An error has occurred when deleting "key: ${key}",`, LOG_CONTEXT);
  }
}
```

缓存删除失败不抛出异常，保证主流程不被缓存故障阻塞。

### 5.4 套餐额度评估降级 (GetEventResourceUsage)

[get-event-resource-limit.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-event-resource-limit.e2e-ee.ts#L103-L155) 展示了降级策略：

- `GetSubscription` 调用超时 → 返回 fallback 结果（FREE 套餐，不限流）
- 订阅无 included events 配置 → 返回 fallback（视为 FREE 无限制）
- fallback 响应标记 `locked: false`，确保不阻塞用户核心流程

---

## 六、多来源 Webhook (Multi-source Webhooks)

Novu 系统中存在三类独立的入站 webhook 体系，各有不同的入口和用途：

### 6.1 Billing Webhook（Stripe 计费）

- **模块归属**：`@novu/ee-billing` 的 `BillingModule` + `@novu/ee-api` 的 `InboundWebhooksModule`
- **加载位置**：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L77-L83)
- **触发源**：Stripe 平台（订阅、发票、支付等）
- **核心事件**：
  - `checkout.session.completed` → CheckoutSessionCompletedHandler
  - `customer.subscription.created` → CustomerSubscriptionCreatedHandler
  - `customer.subscription.deleted` → CustomerSubscriptionDeletedHandler
  - `invoice.paid` / `invoice.payment_failed`（推断存在）

### 6.2 Provider Webhook（邮件/SMS 投递回执）

- **应用归属**：独立的 [apps/webhook](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook) 服务
- **入口 Controller**：[webhooks.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/webhooks.controller.ts)
- **路由模式**：
  ```
  POST /webhooks/organizations/:orgId/environments/:envId/email/:providerId
  POST /webhooks/organizations/:orgId/environments/:envId/sms/:providerId
  ```
- **核心处理**：[Webhook use case](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts)
  - 通过 `MailFactory` / `SmsFactory` 获取对应 Provider Handler
  - 调用 `provider.getMessageId()` 提取消息标识
  - 调用 `provider.parseEventBody()` 解析投递状态
  - 创建 Execution Details 记录投递回执

### 6.3 Outbound Webhook（Novu → 客户系统）

在 [app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L103) 中通过 `OutboundWebhooksModule.forRoot()` 加载，这是 Novu 向外部客户系统推送事件的出站 webhook 模块，与入站计费 webhook 方向相反。

### 6.4 渠道差异总结

| Webhook 类型 | 部署位置 | 验证方式 | 处理目标 |
|-------------|---------|---------|---------|
| Stripe Billing | apps/api (ee-api) | Stripe 签名 + webhook secret | 套餐变更、支付事件 |
| Provider Callback | apps/webhook | 各 Provider 独立签名 | 邮件/SMS 投递状态 |
| Clerk / 其他 SSO | apps/api (ee-auth) | JWT / Provider 签名 | 用户身份同步 |

---

## 七、账户状态更新 (Account Status Update)

### 7.1 数据模型

核心字段定义在 [IOrganizationEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/entities/organization/organization.interface.ts#L27-L57)：

```typescript
interface IOrganizationEntity {
  _id: string;
  name: string;
  apiServiceLevel?: ApiServiceLevelEnum;   // 当前套餐等级
  isTrial?: boolean;                         // 是否处于试用期
  stripeCustomerId?: string;                  // Stripe Customer ID
  externalId?: string;                        // 外部系统 ID
  // ...其他字段
}
```

### 7.2 UpdateServiceLevel 用例

该用例被所有套餐变更 Handler 调用，核心职责：

- 写入 `organization.apiServiceLevel`
- 写入 `organization.isTrial`
- 同步更新 Analytics 分组属性（`upsertGroup`）
- 触发缓存失效

### 7.3 套餐等级与特性映射

[feature-tiers-constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/consts/feature-tiers-constants.ts) 定义了 70+ 项特性在各等级下的额度。关键配额示例：

| 特性 | FREE | PRO | BUSINESS | ENTERPRISE |
|-----|------|-----|----------|------------|
| 月事件额度 | 10,000 | 30,000 | 250,000 | 5,000,000 |
| 月费 | $0 | $30 | $250 | Custom |
| 工作流数量 | 20 | 20 | Unlimited | Unlimited |
| 团队成员 | 3 | 3 | Unlimited | Unlimited |
| Webhook 出站 | ❌ | ❌ | ✅ | ✅ |
| 自动翻译 | ❌ | ❌ | ✅ | ✅ |
| RBAC | ❌ | ❌ | ✅ | ✅ |
| 自定义域名 | ❌ | ❌ | ✅ | ✅ |

### 7.4 配额限流 (Quota Throttler)

[quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) 描述了运行时限流行为：

- **拦截器注册**：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L109-L118) 中全局注册 `QuotaThrottlerInterceptor`
- **限流逻辑**：
  - `IS_SELF_HOSTED=true` → 完全跳过限流
  - FREE 套餐超额 → 返回 **402 Payment Required**
  - PRO/BUSINESS/ENTERPRISE 超额 → 不拦截（按超额计费）
  - fallback 降级状态（locked: false）→ 不拦截

### 7.5 GetSubscription 聚合查询

[get-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-subscription.e2e-ee.ts) 返回给前端的完整订阅信息结构见 [GetSubscriptionDto](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts)：

```typescript
{
  apiServiceLevel,        // 套餐等级
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

### 7.6 状态变更时序

```
用户点击升级 → CreateCheckoutSession (生成 Stripe Checkout URL)
    → 前端跳转 Stripe 完成支付
    → Stripe 触发 checkout.session.completed webhook
        → CheckoutSessionCompletedHandler
            → 取消旧订阅
            → 年付则创建 metered 子订阅
            → Stripe 再触发 customer.subscription.created
                → CustomerSubscriptionCreatedHandler
                    → UpdateServiceLevel (FREE → BUSINESS)
                    → InvalidateCache
                    → Analytics upsertGroup
                        → QuotaThrottlerInterceptor 按新额度限流
```

---

## 八、关键代码文件索引

| 文件 | 作用 |
|-----|------|
| [app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts) | 企业模块加载、拦截器注册 |
| [organization.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/types/organization.ts) | 服务等级枚举、计费周期枚举 |
| [organization.interface.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/entities/organization/organization.interface.ts) | 组织实体接口（含套餐字段） |
| [get-subscription.dto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts) | 订阅信息 DTO |
| [feature-tiers-constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/consts/feature-tiers-constants.ts) | 套餐特性额度矩阵 |
| [idempotency.interceptor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts) | API 层幂等拦截器 |
| [invalidate-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/cache/invalidate-cache.service.ts) | 缓存失效服务 |
| [sync-external-organization.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/organization/usecases/create-organization/sync-external-organization/sync-external-organization.usecase.ts) | 组织创建时的 Billing Customer 创建 |
| [webhooks.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/webhooks.controller.ts) | Provider webhook 入口控制器 |
| [webhook.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts) | Provider webhook 核心处理逻辑 |
| [billing.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/types/billing.ts) | 产品特性枚举 |
| E2E: checkout-session-completed | Checkout 完成事件 Handler 行为 |
| E2E: customer-subscription-created | 订阅创建事件 Handler 行为 |
| E2E: customer-subscription-deleted | 订阅取消事件 Handler 行为 |
| E2E: verify-customer | Stripe Customer ↔ Organization 映射校验 |
| E2E: create-subscription | 订阅创建（含幂等键构造） |
| E2E: create-usage-records | 用量定期上报 |
| E2E: get-event-resource-limit | 额度评估 + 降级 |
| E2E: quota-throttler.guard | 运行时配额限流 |
