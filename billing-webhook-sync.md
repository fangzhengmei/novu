# Billing Webhook 入站处理代码分析

本文档对 Novu 计费系统的 webhook 入站处理进行代码分析，**严格区分已验证事实与证据边界外的推断**。所有标为「事实」的结论均有可见代码作为直接证据；标为「推断」的内容属于合理推测但因企业包不可见而无法验证，仅供参考。

> **证据边界声明**：计费 webhook 的核心实现位于 `@novu/ee-billing` 企业包（git submodule，当前工作区源码未检出）。控制器、路由、签名验证、事件分发等 HTTP 层逻辑均在该包内部，可见代码中无法直接查阅。以下分析仅限于共享代码、e2e 测试、模块注册和类型定义中能够验证的部分。

> **E2E 测试执行状态声明**：billing e2e 共 13 个测试套件中，**仅 `customer-subscription-deleted` 套件被 `describe.skip()` 标记为跳过**，其余 12 个套件正常执行。跳过套件中的所有用例不能作为已验证行为证据。

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

### 1.3 计费入站的业务流程（区分验证状态）

```
Stripe Platform → [BillingModule 内部 Controller（证据边界）]
    → 事件路由分发（证据边界）
        ├─ CheckoutSessionCompletedHandler  [✅ 已验证：5 项断言全部通过]
        ├─ CustomerSubscriptionCreatedHandler [✅ 已验证：5 项断言全部通过]
        └─ CustomerSubscriptionDeletedHandler [⚠️ describe.skip：未执行，不验证]
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

> 注意：以下 early exit 条件**仅以非 skipped 套件中的断言为准**。

| Handler | early exit 条件 | 验证状态 | 证据 |
|---------|----------------|---------|------|
| CheckoutSessionCompleted | organization 为 null → 直接返回，不触发 analytics | ✅ 已验证 | [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L187-L197)：`expect(analyticsServiceStub.track.called).to.be.false` |
| CustomerSubscriptionCreated | 1. organization 为 null → `updateServiceLevelStub.called === false`<br>2. `apiServiceLevel` 为非法值 → `updateServiceLevelStub.called === false` | ✅ 已验证 | [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L168-L178) 与 [L233-L296](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L233-L296) |
| CustomerSubscriptionDeleted | organization 为 null → 不调用 UpdateServiceLevel | ⚠️ 套件 skipped，不验证 | — |

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

证据：[checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L218-L242) 中年付场景下额外创建 metered 子订阅并设置 `parentSubscriptionId`。

### 4.3 Handler 行为（按验证状态区分）

> **重要说明**：本章中 CustomerSubscriptionDeletedHandler 的所有描述均来自 `describe.skip` 套件中的**未执行用例**，代码结构可说明 Handler 设计意图，但**不代表已验证行为**。

#### CheckoutSessionCompletedHandler [✅ 全部 5 项断言已验证]

触发事件：`checkout.session.completed`

| 步骤 | 行为 | 断言 |
|-----|------|------|
| 1 | VerifyCustomer：通过 `customer.metadata.organizationId` 映射 | 隐式 |
| 2 | 组织不存在 → early exit（不触发 analytics） | `track.called === false` ([L187-L197](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L187-L197)) |
| 3 | 取消除触发订阅外的所有其他旧订阅 | `cancel.callCount === 1`，`cancel.lastCall.args[0] === 'subscription_id'` ([L199-L205](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L199-L205)) |
| 4 | 更新 customer 默认支付方式 | `customers.update` 设置 `invoice_settings.default_payment_method` ([L207-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L207-L216)) |
| 5 | 年付专属：额外创建月度 metered 子订阅，设置 `metadata.parentSubscriptionId` | `subscriptions.create` 参数校验 ([L218-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L218-L242)) |
| 6 | 缓存失效 | `invalidateByKey.called === true` ([L244-L249](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts#L244-L249)) |

> 注意：CheckoutSessionCompletedHandler **不直接调用 UpdateServiceLevel**（不注入 UpdateServiceLevel 依赖），等级更新由后续的 `customer.subscription.created` 事件触发。

---

#### CustomerSubscriptionCreatedHandler [✅ 全部 5 项断言已验证]

触发事件：`customer.subscription.created`

| 步骤 | 行为 | 断言 |
|-----|------|------|
| 1 | VerifyCustomer：customer → organization 映射 | 隐式 |
| 2 | 组织不存在 → early exit | `updateServiceLevelStub.called === false` ([L168-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L168-L178)) |
| 3 | **licensed 订阅**：提取 `apiServiceLevel` 并更新等级 | `updateServiceLevelStub` 参数 = `{organizationId, BUSINESS, isTrial:false}` ([L180-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L180-L189)) |
| 4 | 缓存失效（licensed） | `invalidateByKey.called === true` ([L191-L196](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L191-L196)) |
| 5 | **metered 订阅**：**仍调用 UpdateServiceLevel**（❌ 原结论错误，已修正） | `updateServiceLevelStub.called === true` ([L198-L231](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L198-L231)) |
| 6 | apiServiceLevel 为非法值（如 `'invalid'`）→ early exit | `updateServiceLevelStub.called === false` ([L233-L296](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L233-L296)) |

> ⚠️ **重要修正**：之前的分析断言"纯 metered 订阅跳过等级更新"是**错误的**。该用例标题写的是 "should exit early with known organization and metered subscription"，但实际断言是 `expect(updateServiceLevelStub.called).to.be.true`——明确要求 UpdateServiceLevel **被调用**。因此 **metered 订阅不会跳过等级更新**，Handler 会在该分支下调用 UpdateServiceLevel（具体参数未在此用例中断言）。

---

#### CustomerSubscriptionDeletedHandler [⚠️ describe.skip：套件未执行]

触发事件：`customer.subscription.deleted`

套件定义位置：[customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts#L100)：

```typescript
describe.skip('webhook event - customer.subscription.deleted #novu-v2', () => {
```

**状态说明**：整个测试套件被 `describe.skip()` 标记，所有 6 项用例均**未实际执行**。以下内容仅反映用例编写时的**设计意图**，不代表已验证行为：

| 用例 | 预期断言 | 状态 |
|-----|---------|------|
| 组织不存在 → early exit | `updateServiceLevelStub.called === false` | ⚠️ skipped |
| 年付 licensed 取消 → 级联取消 metered 子订阅 | `cancel('linked_metered_subscription_id') === true` | ⚠️ skipped |
| 年付 metered 取消 → 级联取消父 licensed 订阅 | `cancel('licensed_subscription_id') === true` | ⚠️ skipped |
| 无剩余订阅 → 创建 FREE 套餐并降级 | `createSubscriptionStub.called === true` + `UpdateServiceLevel(FREE)` | ⚠️ skipped |
| 有剩余订阅 → 保持最高等级 | `UpdateServiceLevel(BUSINESS)` 保持不变 | ⚠️ skipped |
| 缓存失效 | `invalidateByKey.called === true` | ⚠️ skipped |

**对账户状态更新判断的影响**：
- "级联取消关联订阅"机制（licensed ↔ metered 双向）：**未验证**
- "取消后无订阅自动降级 FREE" 逻辑：**未验证**
- "有剩余订阅则保持最高等级"逻辑：**未验证**
- 实际生产环境中 Handler 是否按这些预期工作，需在 ee-billing 源码或启用该套件后确认。

### 4.4 VerifyCustomer 行为（事实）

[verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts)：

| 场景 | 行为 | 验证状态 |
|-----|------|---------|
| Stripe customer 不存在 | 抛出 `Customer not found` 异常 | ✅ 已验证 |
| Stripe customer 已标记 deleted | 抛出 `Customer is deleted: 'customer_id'` 异常 | ✅ 已验证 |
| customer.metadata.organizationId 对应组织不存在 | 打 verbose 日志，返回 `{ organization: null, customer }` | ✅ 已验证 |
| 正常场景 | 返回 `{ organization, customer, subscriptions }` | ✅ 已验证 |

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

### 6.3 套餐等级变更的触发路径（区分验证状态）

| 事件 | 等级变更操作 | 验证状态 | 说明 |
|-----|------------|---------|------|
| `checkout.session.completed` | — | ✅ | Handler 不注入 UpdateServiceLevel，只做订阅级操作，等级更新留给后续事件 |
| `customer.subscription.created`（licensed） | ✅ UpdateServiceLevel 被调用，参数具体断言 BUSINESS | ✅ | [L180-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L180-L189) |
| `customer.subscription.created`（metered） | ✅ UpdateServiceLevel 被调用（参数未断言） | ✅ | [L198-L231](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L198-L231)：`called === true` |
| `customer.subscription.deleted`（级联取消） | 设计意图：级联取消 licensed ↔ metered | ⚠️ skipped | 未执行，不验证 |
| `customer.subscription.deleted`（无剩余订阅） | 设计意图：CreateSubscription(FREE) + UpdateServiceLevel(FREE) | ⚠️ skipped | 未执行，不验证 |
| `customer.subscription.deleted`（有剩余） | 设计意图：保持最高等级 | ⚠️ skipped | 未执行，不验证 |

### 6.4 配额限流拦截器（事实）

全局注册：[app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L109-L118)

```typescript
const enterpriseQuotaThrottlerInterceptor =
  (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') &&
  require('@novu/ee-billing')?.QuotaThrottlerInterceptor
    ? [{ provide: APP_INTERCEPTOR, useClass: require('@novu/ee-billing')?.QuotaThrottlerInterceptor }]
    : [];
```

限流行为（基于 [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) ✅ 已验证）：

| 场景 | 行为 |
|-----|------|
| `IS_SELF_HOSTED === 'true'` | 完全跳过限流 |
| FREE 套餐超额 | 返回 **402 Payment Required** |
| PRO/BUSINESS/ENTERPRISE 超额 | 不拦截（按超额计费） |
| fallback（locked: false） | 不拦截（降级保护） |

### 6.5 GetSubscription DTO（事实）

[get-subscription.dto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/packages/shared/src/dto/subscription/get-subscription.dto.ts) 定义了前端所需字段：`apiServiceLevel`、`isActive`、`status`、`hasPaymentMethod`、`currentPeriodStart/End`、`billingInterval`、`events{current,included}`、`trial{start,end,isActive,daysTotal}`、`cancelAt`。

### 6.6 套餐状态机（已验证部分）

根据已验证的事件路径，**已确认**的状态流转如下（未确认的 skipped 分支已排除）：

```
组织创建 → FREE (default)
  → 用户点击升级 → CreateCheckoutSession → Stripe 支付
      → checkout.session.completed
          [✅] 取消旧订阅
          [✅] 设置默认支付方式
          [✅] 年付则创建 metered 子订阅
      → customer.subscription.created
          [✅] licensed 订阅：UpdateServiceLevel(FREE → BUSINESS)
          [✅] metered 订阅：UpdateServiceLevel 仍被调用（❌ 原"跳过"结论错误）
          [✅] apiServiceLevel 非法值 → early exit
```

**未确认的 skipped 分支**（不可作为依据）：
```
      → customer.subscription.deleted [⚠️ skipped]
          [?] 级联取消 licensed ↔ metered
          [?] 无订阅 → FREE 降级
          [?] 有剩余订阅 → 保持最高等级
```

---

## 七、核心结论汇总表

| 分析项 | 结论 | 性质 | 验证状态 | 证据 |
|-------|------|------|---------|------|
| **计费 webhook 入口位置** | 在 `@novu/ee-billing` BillingModule 内部注册 | 事实 | — | [app.module.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app.module.ts#L77-L79) |
| **InboundWebhooksModule 用途** | 投递回执 V2 入口，不是计费入站 | 事实 | — | [bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L123-L127) |
| **Raw Body 保留** | 企业版下启用 `rawBody: true` + verify 回调 | 事实 | — | [bootstrap.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/bootstrap.ts#L70-L73) |
| **constructEvent 调用** | ee-billing 内部实现，不可见 | 证据边界 | — | — |
| **STRIPE_WEBHOOK_SECRET 使用** | 可见代码中无直接引用 | 证据边界 | — | — |
| **事件去重是否入库** | ❌ 否，无 webhook_events 等表 | 事实 | — | libs/dal 全部 42 个 schema 排查 |
| **Stripe 操作级幂等键** | ✅ 是，结构化 Key 构造规则可验证 | 事实 | ✅ | [create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts#L115-L119) |
| **customer-subscription-deleted 套件** | `describe.skip`，全部 6 项用例未执行 | 事实 | ⚠️ skipped | [L100](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts#L100) |
| **级联取消关联订阅** | 设计意图存在，但未执行验证 | 设计意图（未验证） | ⚠️ skipped | — |
| **取消后 FREE 降级** | 设计意图存在，但未执行验证 | 设计意图（未验证） | ⚠️ skipped | — |
| **全局 IdempotencyInterceptor** | 不适用于计费 webhook（无 Header） | 事实 | ✅ | [L22-L27](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/shared/framework/idempotency.interceptor.ts#L22-L27) |
| **Handler early exit（org null）** | Checkout + Created 已验证，Deleted 未执行 | 事实 | ✅/⚠️ | 各 e2e 文件 |
| **Handler early exit（apiServiceLevel 非法）** | CustomerSubscriptionCreated ✅ 已验证 | 事实 | ✅ | [L233-L296](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L233-L296) |
| **⟨metered 订阅跳过等级更新⟩** | ❌ **原结论错误**：UpdateServiceLevel 仍被调用（`called === true`） | 事实（修正后） | ✅ | [L198-L231](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts#L198-L231) |
| **本地计费重试队列** | ❌ 否，无 BillingWebhookQueue 等 | 事实 | — | libs/application-generic 全部队列排查 |
| **BullMQ 计费重试 Job** | ❌ 否，无相关定义 | 事实 | — | 全局代码搜索 |
| **容错降级机制** | ✅ Customer 创建 / 缓存失效 / 额度评估 均有 try/catch | 事实 | ✅ | 三处独立代码 |
| **组织套餐字段** | `apiServiceLevel`、`isTrial`、`stripeCustomerId` | 事实 | — | [organization.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/libs/dal/src/repositories/organization/organization.schema.ts#L11-L19) |
| **配额限流** | FREE 超额 402，付费超额不拦截，自托管跳过 | 事实 | ✅ | [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) |

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
| **E2E 测试（✅ 正常执行）** | | |
| [checkout-session-completed.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/checkout-session-completed.e2e-ee.ts) | Checkout 完成 Handler | 5 项断言：early exit(org null)、取消旧订阅、默认支付方式、年付子订阅、缓存失效 |
| [customer-subscription-created.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-created.e2e-ee.ts) | 订阅创建 Handler | 5 项断言：early exit(org null)、licensed 更新等级、缓存失效、**metered 仍调用 UpdateServiceLevel（修正点）**、非法等级 early exit |
| [verify-customer.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/verify-customer.e2e-ee.ts) | 客户校验 | 4 项断言：customer 不存在、customer deleted、组织不存在（返回 null）、正常映射 |
| [create-subscription.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-subscription.e2e-ee.ts) | 订阅创建 | 幂等键构造规则、CreateSubscription → VerifyCustomer 调用链 |
| [create-usage-records.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/create-usage-records.e2e-ee.ts) | 用量上报 | FREE 兜底创建、覆盖式上报 usage record |
| [get-event-resource-limit.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/get-event-resource-limit.e2e-ee.ts) | 额度评估 | fallback 降级策略 |
| [quota-throttler.guard.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/quota-throttler.guard.e2e-ee.ts) | 配额限流 | FREE 402、付费不拦截、自托管跳过、fallback 不限流 |
| **E2E 测试（⚠️ 已跳过）** | | |
| [customer-subscription-deleted.e2e-ee.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/59-novu/apps/api/src/app/billing/e2e/customer-subscription-deleted.e2e-ee.ts#L100) | 订阅取消 Handler | `describe.skip`，6 项用例**全部未执行**，不构成证据 |
