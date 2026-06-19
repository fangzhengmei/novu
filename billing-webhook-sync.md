# Billing Webhook 入站处理代码分析

本文档对 Novu 计费系统的 webhook 入站处理进行代码分析，**严格区分已验证事实与证据边界外的推断**。所有标为「事实」的结论均有可见代码作为直接证据；标为「推断」的内容属于合理推测但因企业包不可见而无法验证，仅供参考。

> **证据边界声明**：计费 webhook 的核心实现位于 `@novu/ee-billing` 企业包（git submodule，当前工作区源码未检出）。控制器、路由、签名验证、事件分发等 HTTP 层逻辑均在该包内部，可见代码中无法直接查阅。以下分析仅限于共享代码、e2e 测试、模块注册和类型定义中能够验证的部分。

---

## 一、整体架构概览

### 1.1 模块分层与加载（事实）

计费系统通过动态加载方式接入主应用：

- **BillingModule**：来自 `@novu/ee-billing` 包，通过 `BillingModule.forRoot()` 注册
  - 注册位置：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L77-L79)
  - 加载条件：`NOVU_ENTERPRISE === 'true' || CI_EE_TEST === 'true'`
  - **包含内容（推断）**：计费 webhook 控制器、事件 Handler、套餐业务逻辑

- **InboundWebhooksModule**：来自 `@novu/ee-api` 包，**用于投递回执 V2**，不是计费 webhook 入口
  - 注册位置：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L81-L83)
  - 路由证据：[bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L123-L127) 中为 `/v2/inbound-webhooks/delivery-providers/` 路径配置了独立的 `text/plain` body parser（用于 AWS SNS 确认）

- **apps/webhook**：独立微服务，**用于投递回执 V1**
  - 路由：[webhooks.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/webhooks.controller.ts#L16-L32)

### 1.2 计费入站 vs 投递回执的对比（事实）

本分析仅关注计费入站，但为避免混淆，明确对比两类入站 webhook：

| 维度 | 计费入站（Stripe） | 投递回执 V2（Provider） | 投递回执 V1（Provider） |
|-----|-------------------|------------------------|------------------------|
| **触发源** | Stripe 平台 | 邮件/SMS Provider | 邮件/SMS Provider |
| **所属模块** | `@novu/ee-billing` BillingModule | `@novu/ee-api` InboundWebhooksModule | `apps/webhook` 独立应用 |
| **路由入口** | （证据边界：ee-billing 内部注册，不可见） | `/v2/inbound-webhooks/delivery-providers/:envId/:integrationId` | `/webhooks/organizations/:orgId/environments/:envId/{email\|sms}/:providerId` |
| **核心处理** | 套餐变更、支付事件、账户状态 | 更新 ExecutionDetails 状态 | 更新 ExecutionDetails 状态 |
| **业务影响** | 组织套餐等级、额度限流、账单 | 仅通知投递状态记录 | 仅通知投递状态记录 |

### 1.3 计费入站的业务流程（基于 e2e 测试验证的事实）

```
Stripe Platform → [BillingModule 内部 Controller（证据边界）]
    → 事件路由分发（证据边界）
        → CheckoutSessionCompletedHandler
        → CustomerSubscriptionCreatedHandler
        → CustomerSubscriptionDeletedHandler
            → VerifyCustomer (customer → organization 映射)
            → 业务逻辑（取消旧订阅 / 创建子订阅 / 降级判断）
            → UpdateServiceLevel (更新 organization.apiServiceLevel)
            → InvalidateCacheService (清理订阅/额度缓存)
            → AnalyticsService (埋点上报)
```

---

## 二、签名验证 (Signature Verification)

### 2.1 已验证事实

#### 事实 1：企业版模式下启用 Raw Body 保留

[bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L70-L73)：

```typescript
if (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') {
  rawBodyBuffer = agentRawBodyBuffer;
  nestOptions.rawBody = true;
}
```

企业版/测试模式下启用 NestJS 的 `rawBody` 选项。这是 Stripe 等签名验证机制的必要前提——必须使用未被 JSON parser 修改的原始请求体。

#### 事实 2：全局 Body Parser 配置了 Verify 回调

[bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L129-L135)：

```typescript
app.use((req, res, next) => {
  if (req.path.startsWith('/v1/better-auth')) {
    return next();
  }
  return bodyParser.json({ verify: rawBodyBuffer })(req, res, next);
});
```

全局 JSON parser 配置了 `verify` 回调，将原始 Buffer 挂载到 `req.rawBody`。排除 `/v1/better-auth`（SSO 路径）避免冲突。

### 2.2 证据边界（不可验证）

以下内容在可见代码中**没有直接证据**，属于 Stripe webhook 的标准实现模式，但 Novu 内部是否采用无法确认：

- `stripe.webhooks.constructEvent()` 的具体调用位置和参数
- `STRIPE_WEBHOOK_SECRET` 环境变量的读取和使用
- `stripe-signature` Header 的校验逻辑
- 验签失败的 HTTP 状态码返回逻辑

### 2.3 与投递回执验签的对比（事实）

| 维度 | 计费入站（Stripe） | 投递回执（Provider） |
|-----|-------------------|---------------------|
| **Raw Body 配置** | 企业版下 `rawBody: true`（全局生效） | V2 路由额外配置了 `text/plain` parser（AWS SNS 需要） |
| **验签代码位置** | 证据边界（ee-billing 内部） | V1：各 Provider `handler.buildProvider()` 内；V2：ee-api 内部 |
| **密钥存储** | 证据边界 | 各 Integration 独立存储的 webhook secret |

---

## 三、事件去重 (Event Deduplication)

### 3.1 已验证事实

#### 事实 1：无数据库持久化去重

对 [libs/dal/src/repositories](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories) 下全部 42 个 MongoDB schema 逐一排查，**不存在** `webhook_events`、`processed_webhooks`、`billing_events` 等专门用于持久化已处理 webhook 事件的数据库表。

排查范围包括 organization、execution-details、notification-template、integration 等所有核心 schema。

#### 事实 2：Stripe API 调用使用操作级幂等键

[create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts#L115-L119)：

```typescript
expect(createSubscriptionStub.lastCall.args[1]).to.have.property('idempotencyKey');
expect(createSubscriptionStub.lastCall.args[1].idempotencyKey).to.equal(
  'subscription-create-organization_id-business-month-combined'
);
```

幂等键构造规则（通过 e2e 验证）：
- 月付：`subscription-create-{orgId}-{serviceLevel}-month-combined`
- 年付 licensed：`subscription-create-{orgId}-{serviceLevel}-year-licensed`
- 年付 metered：`subscription-create-{orgId}-{serviceLevel}-year-metered`

该机制确保 **Stripe 侧的写操作**不重复执行，但不覆盖 Stripe → Novu 方向的 webhook 事件去重。

#### 事实 3：Handler 具备业务幂等（early exit 机制）

多个检查点提前返回，避免重复处理产生副作用：

| Handler | early exit 条件 | 证据 |
|---------|----------------|------|
| CheckoutSessionCompleted | organization 为 null → 直接返回，不执行任何变更，不触发 analytics | [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L187-L197) |
| CustomerSubscriptionCreated | 1. apiServiceLevel 不是合法枚举值<br>2. 纯 metered 订阅 | [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L104-L111) |
| CustomerSubscriptionDeleted | 组织不存在 → 直接返回 | [customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts#L116-L125) |

#### 事实 4：全局 IdempotencyInterceptor 不适用于计费 webhook

[IdempotencyInterceptor](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts#L22-L27) 从请求 Header 读取 `Idempotency-Key`：

```typescript
const idempotencyKey = request.headers['idempotency-key'] as string;
if (!idempotencyKey) return next.handle();
```

Stripe webhook 请求不携带 `Idempotency-Key` Header，因此计费 webhook 不会经过该拦截器的幂等保护。

### 3.2 证据边界（不可验证）

以下去重机制在可见代码中**没有直接证据**，属于常见实现但无法确认 Novu 是否采用：

- 基于 Stripe `event.id` 的 Redis 缓存去重
- 基于数据库唯一索引（如 `eventId` 唯一约束）的去重
- 计费 webhook 控制器层面的任何重复请求检测逻辑

---

## 四、套餐同步 (Plan Sync)

### 4.1 服务等级与计费周期定义（事实）

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

### 4.2 双订阅模型（事实）

通过 e2e 测试验证，Novu 采用**扁平费 (licensed) + 按量费 (metered)** 的双轨计费模式：

| 计费周期 | 订阅结构 | 说明 |
|---------|---------|------|
| 月付 (month) | 单一 Subscription，包含 2 个 Item | Item 1: flat licensed；Item 2: usage metered |
| 年付 (year) | 两个独立 Subscription | Sub A: 年付 licensed（一次性收费）；Sub B: 月付 metered（追踪用量），通过 `metadata.parentSubscriptionId` 关联 |

证据：[checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L235-L241) 中年付场景下额外创建 metered 子订阅并设置 `parentSubscriptionId`。

### 4.3 三大核心 Handler 行为（事实，基于 e2e 测试）

#### CheckoutSessionCompletedHandler

触发事件：`checkout.session.completed`

流程：
1. **VerifyCustomer**：通过 `customer.metadata.organizationId` 映射到 Novu 组织
2. **early exit**：组织不存在 → 返回，不触发 analytics
3. **取消旧订阅**：调用 `stripe.subscriptions.cancel()` 取消除本次触发订阅外的所有其他订阅
4. **更新默认支付方式**：`stripe.customers.update()` 设置 `invoice_settings.default_payment_method`
5. **年付专属逻辑**：若为年付，额外创建月度 metered 订阅，设置 `metadata.parentSubscriptionId`
6. **缓存失效**：`InvalidateCacheService.invalidateByKey()`
7. **埋点上报**：AnalyticsService

#### CustomerSubscriptionCreatedHandler

触发事件：`customer.subscription.created`

流程：
1. **VerifyCustomer**：customer → organization 映射校验
2. **套餐解析**：从 `subscription.items.data[].price.product.metadata.apiServiceLevel` 提取等级
3. **early exit**：apiServiceLevel 非法 或 纯 metered 订阅 → 跳过等级更新
4. **UpdateServiceLevel**：更新 `apiServiceLevel` 和 `isTrial`
5. **缓存失效 + 埋点**

#### CustomerSubscriptionDeletedHandler

触发事件：`customer.subscription.deleted`

流程：
1. **VerifyCustomer**：组织映射校验
2. **级联取消关联订阅**：licensed ↔ metered 双向关联，取消一个自动取消另一个
3. **降级判断**：仍有有效订阅则保持最高等级；否则创建 FREE 套餐并降级
4. **缓存失效 + 埋点**

### 4.4 VerifyCustomer 行为（事实）

[verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts)：

| 场景 | 行为 |
|-----|------|
| Stripe customer 不存在 | 抛出 `Customer not found` 异常 |
| Stripe customer 已标记 deleted | 抛出 `Customer is deleted: 'customer_id'` 异常 |
| customer.metadata.organizationId 对应组织不存在 | 打 verbose 日志，返回 `{ organization: null, customer }` |
| 正常场景 | 返回 `{ organization, customer, subscriptions }` |

### 4.5 用量记录同步（事实）

[create-usage-records.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-usage-records.e2e-ee.ts)：

1. 查询各组织通知发送量
2. 无订阅 → 自动创建 FREE 套餐
3. 找到 metered subscription item（lookup_key 含 `usage_notifications`）
4. 调用 `stripe.subscriptionItems.createUsageRecord()`，`action = 'set'`（覆盖式上报）

---

## 五、失败重试 (Failure Retry)

### 5.1 已验证事实

#### 事实 1：本地无专用计费 webhook 重试队列

对 [libs/application-generic/src/services/queues](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/queues) 目录下所有队列服务逐一排查，**不存在** `StripeWebhookQueue`、`BillingWebhookQueue` 或任何与计费 webhook 重试相关的队列定义。

所有数据库 schema 中也不存在 webhook 重试记录表。

#### 事实 2：无本地重试相关代码

在 `apps/api/src`、`libs/application-generic`、`packages/shared` 中搜索 `stripe.*retry`、`attempts`、`backoff`、`maxAttempts` 等关键词，在 billing 相关代码路径下**没有任何匹配**。

> 注：`attempts` 字段在 `packages/providers` 中有大量匹配，但全部属于**投递回执**（如 SendGrid、Twilio 等 Provider 的事件 `attempt` 字段），与计费 webhook 无关。

#### 事实 3：关键路径具备容错降级

##### 3a. Billing Customer 创建不阻塞组织创建

[sync-external-organization.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/organization/usecases/create-organization/sync-external-organization/sync-external-organization.usecase.ts#L185-L202)：

```typescript
private async createCustomer(billingEmail: string, organizationId: string) {
  try {
    if (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') {
      // 调用 GetOrCreateCustomer
    }
  } catch (e) {
    this.logger.error({ err: e }, `Unexpected error while importing enterprise modules`);
  }
}
```

使用 try/catch 静默降级，仅记录错误日志，不中断主流程。

##### 3b. 缓存失效失败不抛异常

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

##### 3c. 额度评估超时返回 FREE 套餐 fallback

[get-event-resource-limit.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-event-resource-limit.e2e-ee.ts#L103-L155)：`GetSubscription` 超时或异常时返回 fallback 结果（FREE 套餐，`locked: false`，不限流）。

### 5.2 证据边界（不可验证）

以下重试机制在可见代码中**没有直接证据**：

- Stripe webhook 平台侧的重试策略（指数退避间隔、最大重试次数、超时窗口）
- ee-billing 内部是否实现了任何主动重试逻辑（如 BullMQ job、定时任务补偿）
- webhook 处理失败时的告警或人工通知机制

---

## 六、账户状态更新 (Account Status Update)

### 6.1 数据模型（事实）

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
});
```

### 6.2 套餐等级与特性映射（事实）

[feature-tiers-constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/consts/feature-tiers-constants.ts) 定义了 70+ 项特性额度。关键配额：

| 特性 | FREE | PRO | BUSINESS | ENTERPRISE |
|-----|------|-----|----------|------------|
| 月事件额度 | 10,000 | 30,000 | 250,000 | 5,000,000 |
| 工作流数量 | 20 | 20 | Unlimited | Unlimited |
| 团队成员 | 3 | 3 | Unlimited | Unlimited |
| 出站 Webhook | ❌ | ❌ | ✅ | ✅ |
| 自动翻译 | ❌ | ❌ | ✅ | ✅ |
| RBAC | ❌ | ❌ | ✅ | ✅ |

### 6.3 配额限流拦截器（事实）

全局注册：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L109-L118)

```typescript
const enterpriseQuotaThrottlerInterceptor =
  (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') &&
  require('@novu/ee-billing')?.QuotaThrottlerInterceptor
    ? [{ provide: APP_INTERCEPTOR, useClass: require('@novu/ee-billing')?.QuotaThrottlerInterceptor }]
    : [];
```

限流行为（基于 [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts)）：

| 场景 | 行为 |
|-----|------|
| `IS_SELF_HOSTED === 'true'` | 完全跳过限流 |
| FREE 套餐超额 | 返回 **402 Payment Required** |
| PRO/BUSINESS/ENTERPRISE 超额 | 不拦截（按超额计费） |
| fallback（locked: false） | 不拦截（降级保护） |

### 6.4 GetSubscription DTO（事实）

[get-subscription.dto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts) 定义了前端所需字段：`apiServiceLevel`、`isActive`、`status`、`hasPaymentMethod`、`currentPeriodStart/End`、`billingInterval`、`events{current,included}`、`trial{start,end,isActive,daysTotal}`、`cancelAt`。

---

## 七、核心结论汇总表

| 分析项 | 结论 | 性质 | 证据 |
|-------|------|------|------|
| **计费 webhook 入口位置** | 在 `@novu/ee-billing` BillingModule 内部注册 | 事实 | [app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L77-L79) |
| **InboundWebhooksModule 用途** | 投递回执 V2 入口，不是计费入站 | 事实 | [bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L123-L127) |
| **Raw Body 保留** | 企业版下启用 `rawBody: true` + verify 回调 | 事实 | [bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L70-L73) |
| **constructEvent 调用** | ee-billing 内部实现，不可见 | 证据边界 | — |
| **STRIPE_WEBHOOK_SECRET 使用** | 可见代码中无直接引用 | 证据边界 | — |
| **事件去重是否入库** | ❌ 否，无 webhook_events 等表 | 事实 | libs/dal 全部 42 个 schema 排查 |
| **Stripe 操作级幂等键** | ✅ 是，结构化 Key 构造规则可验证 | 事实 | [create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts#L115-L119) |
| **Handler early exit 业务幂等** | ✅ 是，多个检查点提前返回 | 事实 | 各 Handler e2e 测试 |
| **event.id 缓存去重** | 可见代码中无证据 | 证据边界 | — |
| **本地计费重试队列** | ❌ 否，无 BillingWebhookQueue 等 | 事实 | libs/application-generic 全部队列排查 |
| **BullMQ 计费重试 Job** | ❌ 否，无相关定义 | 事实 | 全局代码搜索 |
| **Stripe 平台重试策略** | 可见代码中无配置，依赖 Stripe 默认行为 | 证据边界 | — |
| **容错降级机制** | ✅ 是，Customer 创建/缓存失效/额度评估 均有 try/catch | 事实 | sync-external-organization、invalidate-cache、get-event-resource-limit |
| **组织套餐字段** | `apiServiceLevel`、`isTrial`、`stripeCustomerId` | 事实 | [organization.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/organization/organization.schema.ts#L11-L19) |
| **配额限流** | FREE 超额 402，付费超额不拦截，自托管跳过 | 事实 | [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) |

---

## 八、关键代码文件索引

| 文件 | 作用 | 可验证内容 |
|-----|------|-----------|
| [app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L69-L118) | 应用模块 | BillingModule、InboundWebhooksModule、QuotaThrottlerInterceptor 注册位置 |
| [bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L70-L135) | 应用启动 | rawBody 配置、投递回执 V2 路由 body parser |
| [organization.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/organization/organization.schema.ts#L11-L19) | 组织数据模型 | apiServiceLevel、isTrial、stripeCustomerId 字段 |
| [organization.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/types/organization.ts#L3-L14) | 类型定义 | ApiServiceLevelEnum、StripeBillingIntervalEnum |
| [get-subscription.dto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts) | DTO | 订阅信息结构 |
| [feature-tiers-constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/consts/feature-tiers-constants.ts) | 配额矩阵 | 70+ 项特性额度 |
| [idempotency.interceptor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts#L22-L27) | 幂等拦截器 | 验证不适用于计费 webhook（无 Idempotency-Key Header） |
| [invalidate-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/application-generic/src/services/cache/invalidate-cache.service.ts#L12-L19) | 缓存失效 | 容错降级 |
| [sync-external-organization.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/organization/usecases/create-organization/sync-external-organization/sync-external-organization.usecase.ts#L185-L202) | 组织创建 | Billing Customer 创建容错降级 |
| [webhooks.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/webhook/src/webhooks/webhooks.controller.ts#L16-L32) | 投递回执 V1 入口 | 对比用，与计费入站无关 |
| **E2E 测试** | | |
| [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts) | Checkout 完成 Handler | early exit、级联取消、年付子订阅、缓存失效 |
| [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts) | 订阅创建 Handler | 套餐解析、early exit 条件、UpdateServiceLevel |
| [customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts) | 订阅取消 Handler | 级联取消、FREE 降级 |
| [verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts) | 客户校验 | customer → organization 映射逻辑、异常场景 |
| [create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts) | 订阅创建 | idempotencyKey 构造规则 |
| [create-usage-records.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-usage-records.e2e-ee.ts) | 用量上报 | 定期用量同步逻辑 |
| [get-event-resource-limit.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-event-resource-limit.e2e-ee.ts) | 额度评估 | 降级策略 |
| [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) | 配额限流 | 运行时限流行为 |
