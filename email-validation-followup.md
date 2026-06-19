# 邮件地址校验 — 代码路径深度分析（续）

> 本文件为 `email-address-validation.md` 的补充，重点梳理四条核心链路的**代码走向**：MX 校验在域名验证中的调用链、订阅者邮箱格式校验的完整流程、缺失邮箱跳过结果与消息状态的层级差异、以及延迟事件的软退信映射关系。

---

## 1. 域名验证中的 MX 校验 — 调用链全景

Novu 的 MX 校验分散在三个独立接口，面向不同场景，代码互不复用。

### 1.1 主路径：域名验证接口

```
POST /domains/:domain/verify
    ↓
DomainsController.verifyDomain() [domains.controller.ts#L173-L197]
    ↓
VerifyDomain.execute() [verify-domain.usecase.ts#L22-L64]
    ├─ resolveDomainName() → 按 name 查 Domain 实体
    ├─ getMailServerDomain() → 从 MAIL_SERVER_DOMAIN env 提取期望的 MX 主机
    ├─ checkMxRecord(domain.name, INBOUND_DOMAIN) → 核心查询
    │   ├─ dnsPromises.resolveMx(lookupDomain) → Node.js 原生 DNS 查询
    │   ├─ records.some(r => r.exchange === expectedExchange) → 比对
    │   └─ 异常处理：
    │       ├─ ENOTFOUND / ENODATA → definitive=false, configured=false
    │       └─ 其他异常 → definitive=false（保留原状态，不误降级）
    ├─ 状态判定：
    │   └─ definitive=true 时才更新 mxRecordConfigured 和 status
    └─ domainRepository.update({ $set: { mxRecordConfigured, status } })
```

**关键文件**：
- 控制器入口：[domains.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/domains.controller.ts#L173-L197)
- 核心用例：[verify-domain.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/usecases/verify-domain/verify-domain.usecase.ts#L66-L92)

**设计要点**：
- **`definitive` 标志位**：只有 DNS 返回明确的"记录存在/不存在"时才更新状态；网络超时等瞬态失败时保留原有状态，避免已验证域名被误降级回 pending。
- **状态映射**：`mxRecordConfigured: true` → `status: VERIFIED`；`false` → `status: PENDING`。
- **DNS 查询方式**：直接调用 `node:dns` 的 `promises.resolveMx()`，无缓存、无重试。

### 1.2 次路径：入站解析 MX 状态接口

```
GET /inbound-parse/mx/status
    ↓
InboundParseController.getMxRecordStatus() [inbound-parse.controller.ts#L27-L39]
    ↓
GetMxRecord.execute() [get-mx-record.usecase.ts#L36-L62]
    ├─ 从 Environment 实体取 dns.inboundParseDomain 作为待查域名
    ├─ dnsPromises.resolveMx(domain)
    ├─ 检查是否存在指向 INBOUND_DOMAIN (MAIL_SERVER_DOMAIN) 的 MX 记录
    └─ 写回 Environment.dns.mxRecordConfigured（仅状态变化时写库）
```

**关键文件**：
- 控制器：[inbound-parse.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/inbound-parse/inbound-parse.controller.ts#L27-L39)
- 用例：[get-mx-record.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/inbound-parse/usecases/get-mx-record/get-mx-record.usecase.ts#L1-L64)

**与主路径的区别**：
- 查询对象不同：主路径查 `Domain` 实体（自定义域名），次路径查 `Environment` 实体的 `inboundParseDomain`。
- 错误处理不同：主路径有 `definitive` 保护，次路径 catch 直接返回空数组（`[]`），不区分瞬态/确定失败。

### 1.3 诊断路径：域名诊断接口

```
POST /domains/:domain/diagnose
    ↓
DomainsController.diagnoseDomain()
    ↓
DiagnoseDomain.execute() [diagnose-domain.usecase.ts#L46-L316]
    ├─ Phase 1: MX 记录检查 (MX_MISSING / MX_WRONG_TARGET / MX_LOW_PRIORITY)
    │   ├─ withDnsTimeout(resolveMx(domain), 5000) → 5s 超时
    │   ├─ 记录是否存在
    │   ├─ 是否指向 Novu 邮件服务器（normalize 后比对）
    │   └─ 优先级是否最高
    ├─ Phase 2: Apex CNAME 冲突检查
    │   └─ CNAME 与 MX 不能共存于 zone apex
    └─ Phase 3: DNS 黑名单检查
        ├─ resolveHostnameToIpv4 → 解析邮件服务器 IP
        ├─ 跳过私有/回环地址
        └─ 查 zen.spamhaus.org / b.barracudacentral.org / dnsbl.sorbs.net
```

**关键文件**：[diagnose-domain.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/usecases/diagnose-domain/diagnose-domain.usecase.ts#L46-L316)

**三条路径对比**：

| 路径 | 接口 | 查询目标 | 结果用途 | 错误处理 |
|---|---|---|---|---|
| 主路径 | `POST /domains/:domain/verify` | Domain 实体 | 更新域名验证状态 | definitive 保护，不误降级 |
| 次路径 | `GET /inbound-parse/mx/status` | Environment.inboundParseDomain | 更新环境级 MX 配置标记 | 直接吞异常，返回空数组 |
| 诊断路径 | `POST /domains/:domain/diagnose` | 入站域名 | 返回结构化诊断报告 | 5s 超时，详细 issue 列表 |

---

## 2. 订阅者邮箱格式校验 — 完整流程

邮箱格式校验**只发生在 API 入口层**（DTO 验证），业务逻辑层不做格式校验。

### 2.1 校验发生位置

```
HTTP Request → NestJS ValidationPipe → class-validator
    ↓
BaseSubscriberFieldsDto.email [base-subscriber-fields.dto.ts#L28-L37]
    @IsOptional()
    @ValidateIf((obj) => obj.email !== null)
    @IsEmail()
    email?: string | null;
```

**关键文件**：[base-subscriber-fields.dto.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/shared/dtos/base-subscriber-fields.dto.ts#L28-L37)

**校验规则**：
- 使用 `class-validator` 的 `@IsEmail()` 装饰器（底层依赖 `validator` 库）。
- `@IsOptional()` + `@ValidateIf(obj.email !== null)` —— `undefined` 和 `null` 都跳过校验（email 可选，且支持显式 null 赋值）。
- `@IsEmail()` 使用默认选项，遵循 RFC 5322 的宽松模式。

### 2.2 调用链：V2 创建订阅者

```
POST /subscribers
    ↓
SubscribersController.createSubscriber() [subscribers.controller.ts#L188-L236]
    ├─ DTO: CreateSubscriberRequestDto extends BaseSubscriberFieldsDto
    │   └─ @IsEmail() 验证（请求进入 controller 前由 ValidationPipe 执行）
    └─ CreateOrUpdateSubscriberUseCase.execute()
        ├─ getExistingSubscriber() → 查库判断是否存在
        ├─ 存在 → updateSubscriber() → UpdateSubscriber.execute()
        └─ 不存在 → createSubscriber() → subscriberRepository.create()
```

**注意**：`CreateOrUpdateSubscriberUseCase`（[create-or-update-subscriber.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/usecases/create-or-update-subscriber/create-or-update-subscriber.usecase.ts#L25-L49)）和 `UpdateSubscriber`（[update-subscriber.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/usecases/update-subscriber/update-subscriber.usecase.ts#L17-L104)）**不做任何 email 格式校验**，直接把 `command.email` 写入 update payload。

### 2.3 调用链：V2 部分更新

```
PATCH /subscribers/:subscriberId
    ↓
SubscribersController.patchSubscriber()
    ├─ DTO: PatchSubscriberRequestDto extends BaseSubscriberFieldsDto
    │   └─ @IsEmail() 验证
    └─ PatchSubscriber.execute()
        ├─ 校验 subscriberId 格式
        └─ UpdateSubscriber.execute() → 直接写库
```

### 2.4 批量创建

V1 的 `BulkSubscriberCreateDto`（[create-subscriber-request.dto.ts#L72-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/subscribers/dtos/create-subscriber-request.dto.ts#L72-L82)）中每个 subscriber 都继承 `BaseSubscriberFieldsDto`，因此批量创建时每个 email 都会被 `@IsEmail()` 校验。

### 2.5 其他入口的 email 校验

- **agents 模块**：[email-normalization.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/agents/shared/util/email-normalization.ts#L1-L9) 使用简易正则 `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` 做查询前格式检查，**不经过** `class-validator`。
- **domain routes**：[email-local-part.validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/validators/email-local-part.validator.ts#L1-L42) 只校验 local-part（@ 左边），用于入站邮件路由地址配置。

### 2.6 校验层级结论

| 层级 | 是否校验 | 方式 | 适用范围 |
|---|---|---|---|
| API DTO 层 | ✅ | `@IsEmail()` (class-validator) | V1/V2 所有订阅者创建/更新接口 |
| application-generic 层 | ❌ | 无 | 直接透传 |
| DAL 层 | ❌ | 无 | 直接写入 MongoDB |
| agents 模块 | ⚠️ | 简易正则 | 邮箱查找前快速检查 |
| domain routes | ⚠️ | RFC 5321 atom 正则 | 仅校验 local-part |

---

## 3. 缺失邮箱的跳过结果与消息状态差异

"订阅者没有邮箱"在 Novu 中会穿过四个独立的状态体系，每个体系有自己的语义和枚举值。

### 3.1 四层状态体系

```
订阅者无邮箱
    │
    ├─ SendMessageStatus.SKIPPED              ← 用例返回值（worker 内部）
    │   (send-message-type.usecase.ts#L6-L11)
    │
    ├─ DeliveryLifecycleStatusEnum.SKIPPED    ← Job 级别状态（工作流维度）
    │   + detail: USER_MISSING_EMAIL
    │   (delivery-lifecycle-status.enum.ts#L1-L10)
    │
    ├─ Message.status = 'warning'             ← Message 实体（数据库）
    │   + errorId: 'mail_unexpected_error'
    │   + errorText: 'Subscriber does not have an email address'
    │   (message.entity.ts#L113)
    │
    └─ ExecutionDetails.status = FAILED       ← 执行详情（活动日志）
        + detail: SUBSCRIBER_MISSING_EMAIL_ADDRESS
        (create-execution-details types)
```

### 3.2 代码走向：sendErrors 方法

**文件**：[send-message-email.usecase.ts#L430-L493](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L430-L493)

```typescript
private async sendErrors(email, integration, message, command): SendMessageResult {
  const status = 'warning';          // Message.status 用的值
  const errorId = 'mail_unexpected_error';

  if (!email) {
    // 1. 更新 Message 实体
    await this.sendErrorStatus(message, status, errorId, 
      'Subscriber does not have an email address', command);

    // 2. 写入执行详情
    await this.createExecutionDetails.execute({
      detail: DetailEnum.SUBSCRIBER_MISSING_EMAIL_ADDRESS,
      status: ExecutionDetailsStatusEnum.FAILED,
      source: ExecutionDetailsSourceEnum.INTERNAL,
      ...
    });

    // 3. 返回 SKIPPED 给上层
    return {
      status: SendMessageStatus.SKIPPED,
      deliveryLifecycleState: {
        status: DeliveryLifecycleStatusEnum.SKIPPED,
        detail: DeliveryLifecycleDetail.USER_MISSING_EMAIL,
      },
    };
  }
  ...
}
```

### 3.3 状态差异对照表

| 状态体系 | 枚举值 | 含义 | 存储位置 |
|---|---|---|---|
| **SendMessageStatus** | `SKIPPED` | 步骤跳过，非失败也非成功 | 函数返回值，不持久化 |
| **DeliveryLifecycleStatusEnum** | `SKIPPED` + `USER_MISSING_EMAIL` | 工作流维度的递送生命周期 | `Job.deliveryLifecycleState`（MongoDB） |
| **Message.status** | `'warning'` | 消息实体的状态标记 | `Message.status`（MongoDB） |
| **ExecutionDetailsStatusEnum** | `FAILED` | 执行详情的成功/失败判定 | `ExecutionDetails` 集合 + ClickHouse |

### 3.4 关键发现

1. **Message 没有 SKIPPED 状态**：Message 实体的 `status` 字段只有 `'sent' | 'error' | 'warning'` 三值（[message.entity.ts#L113](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/dal/src/repositories/message/message.entity.ts#L113)）。跳过的消息被标记为 `warning`，这是一个语义模糊的中间状态。

2. **同样是跳过，但状态不同**：
   - 缺少邮箱 → `Message.status = 'warning'` + `SendMessageStatus.SKIPPED`
   - 没有活跃集成 → `Message.status = 'warning'` + `SendMessageStatus.FAILED` + `errorMessage: SUBSCRIBER_NO_ACTIVE_INTEGRATION`
   - 区别：一个返回 SKIPPED，一个返回 FAILED。SKIPPED 表示"条件不满足，跳过"；FAILED 表示"应发送但失败"。

3. **Job 级状态才是权威的跳过记录**：`Job.deliveryLifecycleState` 包含 `status: SKIPPED` + `detail: USER_MISSING_EMAIL`，这是最完整的跳过原因记录。

4. **执行详情标记为 FAILED**：在 ExecutionDetails 中，缺失邮箱被记录为 `FAILED` 状态，这与 SendMessage/Job 层的 `SKIPPED` 存在语义不一致 —— 跳过不等于失败。

### 3.5 状态传播路径

```
SendMessageResult.SKIPPED
    ↓ （由 send-message.usecase.ts 消费）
Job.deliveryLifecycleState = { status: SKIPPED, detail: USER_MISSING_EMAIL }
    ↓
后续步骤是否继续？ → 取决于工作流调度器对 SKIPPED 的处理
```

---

## 4. 延迟事件的软退信映射

Novu 的邮件事件体系中有两个与"延迟/暂缓"相关的状态：`DEFERRED` 和 `DELAYED`。软退信（soft bounce）则没有独立状态，被并入 `BOUNCED`。

### 4.1 Provider 层事件映射总览

| Provider | 软退信 / 延迟事件 | 映射到 Novu 的 EmailEventStatusEnum |
|---|---|---|
| **Mandrill** | `soft_bounce` | `BOUNCED` |
| **Mandrill** | `deferral` | `DEFERRED` |
| **SendGrid** | `deferred`（未映射！） | — |
| **SES** | `DeliveryDelay` | `DELAYED` |
| **Resend** | `email.delivery_delayed` | `DELAYED` |
| **Mailgun** | `failed`（含软/硬） | `REJECTED` |
| **Postmark** | 无延迟事件 | — |
| **Mailjet** | 无延迟事件 | — |
| **NetCore** | 无延迟事件 | — |
| **Brevo** | 待查 | — |

**关键文件**：
- Mandrill: [mandrill.provider.ts#L190-L215](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/mandrill/mandrill.provider.ts#L190-L215)
- SES: [ses.provider.ts#L173-L194](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/ses/ses.provider.ts#L173-L194)
- Resend: [resend.provider.ts#L172-L193](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/resend/resend.provider.ts#L172-L193)
- SendGrid: [sendgrid.provider.ts#L341-L358](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/sendgrid/sendgrid.provider.ts#L341-L358)

### 4.2 `DEFERRED` vs `DELAYED`：两个相似但不同的枚举

**EmailEventStatusEnum**（[provider.interface.ts#L129-L143](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/stateless/src/lib/provider/provider.interface.ts#L129-L143)）定义了两个相关值：

```typescript
DEFERRED = 'deferred',   // Mandrill deferral 事件
DELAYED = 'delayed',     // SES DeliveryDelay + Resend delivery_delayed
```

**语义差异**：
- `DEFERRED`：服务商暂时性无法投递，正在内部重试（如收件服务器返回 4xx 响应）。
- `DELAYED`：投递被延迟，预计稍后送达。

两者本质上都是"软退信"范畴的暂时性失败，但来源不同、命名不同。

### 4.3 执行详情映射：DEFERRED → PENDING，DELAYED → ???

**文件**：[create-execution-details.usecase.ts (webhook)](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/webhook/src/webhooks/usecases/execution-details/create-execution-details.usecase.ts#L74-L101)

```typescript
private mapEmailStatus(eventStatus: EmailEventStatusEnum): ExecutionDetailsStatusEnum {
  switch (eventStatus) {
    case EmailEventStatusEnum.DEFERRED:
      return ExecutionDetailsStatusEnum.PENDING;   // 暂缓，等待后续事件
    case EmailEventStatusEnum.BOUNCED:
      return ExecutionDetailsStatusEnum.FAILED;    // 硬退信 = 失败
    // ... 其他状态
    default:
      return ExecutionDetailsStatusEnum.SUCCESS;   // ⚠ DELAYED 会落到这里！
  }
}
```

**关键发现：DELAYED 没有显式处理，默认落到 SUCCESS**

- `DEFERRED` → `PENDING`（正确：暂缓，等待后续送达或退信）
- `DELAYED` → `SUCCESS`（**不正确**：延迟不应视为成功）

这是一个隐性 Bug：SES 和 Resend 的投递延迟事件会被错误地标记为执行成功。

### 4.4 DetailEnum 有对应值，但事件类型映射缺漏

DetailEnum 中确实定义了 `MESSAGE_DEFERRED` 和 `MESSAGE_DELAYED`（[types/index.ts#L26-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/usecases/create-execution-details/types/index.ts#L26-L28)）：

```typescript
MESSAGE_DEFERRED = 'Message deferred',
MESSAGE_DELAYED = 'Message delayed',
```

但在 webhook 的执行详情创建流程中，事件到 DetailEnum 的映射存在缺失。

### 4.5 软退信（soft bounce）的消失

Mandrill 明确区分 `hard_bounce` 和 `soft_bounce`（[mandrill.provider.ts#L17-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/mandrill/mandrill.provider.ts#L17-L28)）：

```typescript
HARD_BOUNCED = 'hard_bounce',
SOFT_BOUNCED = 'soft_bounce',
```

但在 `getStatus` 映射中**两者都映射为 `BOUNCED`**：

```typescript
case MandrillStatusEnum.HARD_BOUNCED:
  return EmailEventStatusEnum.BOUNCED;  // 硬退信
case MandrillStatusEnum.SOFT_BOUNCED:
  return EmailEventStatusEnum.BOUNCED;  // 软退信 → 也当成硬退信！
```

**后果**：
- 软退信（邮箱满、对方限流等暂时性问题）和硬退信（地址不存在）在 Novu 内部无法区分。
- 基于 bounce 的 suppression 策略如果按"所有 bounce 都拉黑"来做，会误伤暂时性问题的地址。
- 实际上当前版本还没有自动 suppression，所以这个问题目前只影响数据统计。

### 4.6 SendGrid 的 deferred 事件未映射

SendGrid 的 `deferred` 事件（收件服务器暂时拒绝，服务商内部重试）在 `sendgrid.provider.ts` 的 `getStatus` 方法中**没有 case**，会返回 `undefined`，导致事件被丢弃。

### 4.7 软退信事件处理总结

```
Provider 事件 → EmailEventStatusEnum → ExecutionDetailsStatusEnum → 业务影响
────────────   ─────────────────────   ──────────────────────────   ────────
Mandrill hard_bounce → BOUNCED    → FAILED  → 记录为退信
Mandrill soft_bounce → BOUNCED    → FAILED  → 记录为退信（误归类）
Mandrill deferral   → DEFERRED   → PENDING → 正确，等待后续
SES DeliveryDelay    → DELAYED    → SUCCESS → ⚠ 被错误地视为成功
Resend delayed       → DELAYED    → SUCCESS → ⚠ 被错误地视为成功
SendGrid deferred    → (未映射)   → (丢弃)  → ⚠ 事件丢失
Mailgun failed       → REJECTED   → FAILED  → 记录为拒绝（不分软/硬）
```

---

## 四链路汇总图

```
┌───────────────────────────────────────────────────────────────────────┐
│                    1. MX 校验（域名验证层）                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ verify      │  │ inbound-     │  │ diagnose     │                │
│  │ domain      │  │ parse/mx     │  │ domain       │                │
│  │ (更新状态)   │  │ (环境级标记) │  │ (诊断报告)    │                │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                │                  │                        │
│         └────────────────┼──────────────────┘                        │
│                          │                                           │
│                    dnsPromises.resolveMx                             │
│                    (无缓存，实时查询)                                 │
└───────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│                   2. 订阅者邮箱格式校验（API 入口层）                    │
│  POST /subscribers                                                    │
│    → BaseSubscriberFieldsDto.email                                    │
│    → @IsEmail() (class-validator)                                    │
│    → 直接进入 application-generic 层（不再校验）                       │
│    → 直接写入 MongoDB                                                 │
└───────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│                   3. 缺失邮箱 → 四层状态体系                           │
│                                                                       │
│  SendMessageStatus.SKIPPED  （用例返回值，内存中）                     │
│         │                                                             │
│  DeliveryLifecycleStatus.SKIPPED + USER_MISSING_EMAIL （Job 实体）    │
│         │                                                             │
│  Message.status = 'warning'  （Message 实体，只有 sent/error/warning）│
│         │                                                             │
│  ExecutionDetails.FAILED  （执行详情，语义不一致）                     │
└───────────────────────────────────────────────────────────────────────┘
                                     │
┌───────────────────────────────────────────────────────────────────────┐
│                   4. 延迟 / 软退信映射（Webhook 层）                    │
│                                                                       │
│  soft_bounce → BOUNCED → FAILED  (误归类为硬退信)                     │
│  deferral    → DEFERRED → PENDING (正确的暂缓状态)                    │
│  delayed     → DELAYED → SUCCESS (⚠ Bug: 被误判为成功)               │
│  deferred    → (未映射) → 丢弃 (⚠ SendGrid 事件丢失)                 │
└───────────────────────────────────────────────────────────────────────┘
```
