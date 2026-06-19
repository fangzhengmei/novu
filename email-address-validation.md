# 邮件地址校验 — 代码路径分析

> 分析范围：Novu Community Edition（含 Enterprise 子模块目录），覆盖 syntax 校验、MX 查询、DNS 缓存、无效地址清理、黑名单批处理与软退信反馈六大链路。

---

## 1. Syntax 校验

Novu 在多处对邮件地址做格式校验，职责分散在不同层级。

### 1.1 邮件本地部分（local-part）校验

**文件**: [email-local-part.validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/validators/email-local-part.validator.ts#L1-L42)

- 使用 `class-validator` 的装饰器模式，自定义 `IsEmailLocalPart` 约束。
- 核心正则：`/^[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*$/i`，遵循 RFC 5321 允许的 atom-dot 结构。
- 长度约束：`MAX_LOCAL_PART_LENGTH = 64`（RFC 规定）。
- 特殊规则：允许通配符 `*`（用于 catch-all 收件路由），禁止包含 `@`，禁止前后空白。
- 注册为 NestJS `ValidatorConstraint`，可在 DTO 上直接用 `@IsEmailLocalPart()` 装饰。

### 1.2 Agent 共享收件箱的简易校验

**文件**: [email-normalization.ts (agents)](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/agents/shared/util/email-normalization.ts#L1-L9)

- 正则：`/^[^\s@]+@[^\s@]+\.[^\s@]+$/` — 最基本的 "有 @、有点、无空格" 校验。
- `normalizeEmailForLookup`：仅做 trim + toLowerCase，用于 agent 查找 subscriber。
- **不负责**完整的 RFC 校验，仅作为查询前的格式防护。

### 1.3 邮件地址规范化（去重用）

**文件**: [email-normalization.ts (application-generic)](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/utils/email-normalization.ts#L1-L48)

- 按域名做 provider-specific 规范化：
  - `gmail.com` / `googlemail.com`：去除 `+` 别名和 `.`（Gmail 忽略点号），`googlemail.com` 别名归一到 `gmail.com`。
  - `hotmail.com` / `outlook.com`：仅去除 `+` 别名。
  - `live.com`：去除 `+` 和 `.`。
- 目的：将 `john.doe+promo@gmail.com` 和 `johndoe@gmail.com` 归一为同一地址，避免重复发送。
- 不做格式校验，仅做标准化。

### 1.4 发送前缺失地址检查

**文件**: [send-message-email.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L430-L464)

- 在 `sendErrors` 方法中，如果 subscriber 没有 email 字段，记录 `DetailEnum.SUBSCRIBER_MISSING_EMAIL_ADDRESS` 执行详情，消息状态标记为 `SKIPPED`。
- **不做** SMTP 级别的邮箱存在性校验 — 这是正确做法，避免 RCPT TO 枚举攻击。

---

## 2. MX 查询

MX 记录查询主要服务于两个场景：**入站邮件解析路由** 和 **域名诊断**。

### 2.1 入站 MX 记录校验

**文件**: [get-mx-record.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/inbound-parse/usecases/get-mx-record/get-mx-record.usecase.ts#L1-L64)

- 使用 `node:dns` 的 `promises.resolveMx(domain)` 查询 MX 记录。
- 流程：
  1. 从 Environment 实体的 `dns.inboundParseDomain` 取出待查域名。
  2. 调用 `checkMxRecordExistence` → `getMxRecords` 解析 MX 记录列表。
  3. 比对是否存在 `record.exchange === INBOUND_DOMAIN`（`MAIL_SERVER_DOMAIN` 环境变量）。
  4. 将结果写回 `Environment.dns.mxRecordConfigured`，仅在状态变化时写库。
- 作用域 `Scope.REQUEST`，每次请求创建新实例。
- 查询失败时返回空数组（`catch` 吞掉异常），不抛出错误。

### 2.2 预期 DNS 记录构建

**文件**: [dns-records.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/utils/dns-records.ts#L1-L25)

- `getMailServerDomain`：从 `MAIL_SERVER_DOMAIN` 环境变量提取，去除协议前缀和尾部斜杠。
- `buildExpectedDnsRecords`：为给定域名生成期望的 MX 记录（priority 10, TTL Auto），用于指导用户在 DNS 服务商处配置。

### 2.3 域名诊断 MX 检查

**文件**: [diagnose-domain.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/usecases/diagnose-domain/diagnose-domain.usecase.ts#L59-L202)

- 三阶段 MX 检查：
  1. **MX 是否存在** (`MX_MISSING`)：`resolveMx` 查询，超时通过 `withDnsTimeout` 控制（5s）。
  2. **MX 是否指向 Novu** (`MX_WRONG_TARGET`)：比对 normalized exchange 与期望值。
  3. **MX 优先级是否最高** (`MX_LOW_PRIORITY`)：如果存在比 Novu MX 优先级更高的记录，发出 WARN。
- 还检查 **Apex CNAME 冲突** (`APEX_CNAME_COLLISION`)：CNAME 不能与 MX 共存于 zone apex。
- 所有检查结果以结构化 `checks[]` + `issues[]` 返回，issue 包含 severity（ERROR/WARN）和修复建议（fix）。

---

## 3. DNS 缓存

### 3.1 InMemoryLRUCacheService（通用缓存层）

**文件**: [in-memory-lru-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.service.ts#L1-L161)

**文件**: [in-memory-lru-cache.store.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.store.ts#L1-L89)

- 基于 `lru-cache` 包实现的通用 LRU 缓存服务，受 Feature Flag `IS_LRU_CACHE_ENABLED` 控制。
- **存储类型**：`InMemoryLRUCacheStore` 枚举定义了 8 种缓存 store（WORKFLOW, ORGANIZATION, ENVIRONMENT, ENVIRONMENT_VARIABLES, API_KEY_USER, VALIDATOR, ACTIVE_WORKFLOWS, WORKFLOW_PREFERENCES），**没有专门的 DNS 缓存 store**。
- 关键特性：
  - **Inflight 请求去重**：同一 key 的并发请求会共享同一个 Promise，防止缓存击穿。
  - **Feature Flag 门控**：每次 `get` 前检查 flag，可按 environment/organization 粒度开关。
  - **TTL 配置**：最短 30s（WORKFLOW），最长 1h（VALIDATOR，且 skipFeatureFlag=true）。
  - **Variant key**：支持 `key:v:variant` 格式，用于同一 key 的不同版本缓存。

### 3.2 DNS 查询无显式缓存

- 当前所有 DNS 查询（`resolveMx`, `resolveNs`, `resolve4`, `resolveCname`）均为**直接调用 `node:dns`**，未经过 `InMemoryLRUCacheService`。
- `dns-diagnostics.ts` 中仅实现了**超时控制**（`withDnsTimeout`），没有缓存层。
- 这意味着每次入站邮件解析或域名诊断都会实时查询 DNS，可能在高并发场景下产生大量重复查询。
- **潜在改进方向**：可为 MX 查询引入一个短 TTL（如 60s）的 LRU 缓存 store，减少 DNS 解析延迟和上游压力。

### 3.3 JS SDK 侧缓存

**文件**: [in-memory-cache.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/js/src/cache/in-memory-cache.ts#L1-L37)

- 客户端 SDK 提供了一个简单的 `Map<string, T>` 缓存，无 TTL、无 LRU 淘汰。
- 用于 SDK 内部状态缓存，与 DNS 无关。

---

## 4. 无效地址清理

### 4.1 发送前缺失地址检测

**文件**: [send-message-email.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L430-L464)

- 在 `sendErrors` 中检测 subscriber 是否缺少 email 字段：
  - 缺少 → 记录 `SUBSCRIBER_MISSING_EMAIL_ADDRESS`，消息状态 `SKIPPED`。
  - 无活跃集成 → 记录 `SUBSCRIBER_NO_ACTIVE_INTEGRATION`，消息状态 `FAILED`。

### 4.2 消息状态追踪

**文件**: [message.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/dal/src/repositories/message/message.entity.ts#L113-L113)

- Message 实体的 `status` 字段取值：`'sent' | 'error' | 'warning'`。
- 发送失败时 status 被设为 `error`，errorId 和 errorText 记录具体原因。

### 4.3 无自动清理机制

- Community Edition 中**没有**自动化的无效地址清理流程：
  - 没有 bounce 后自动标记 subscriber 为 suppressed 的逻辑。
  - 没有 bounce 阈值触发自动停发的机制。
  - 没有 bounce 地址定期清理的定时任务。
- Subscriber Schema 中**没有** `suppressed` / `blocklisted` 字段（参见 [subscriber.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/dal/src/repositories/subscriber/subscriber.schema.ts#L8-L36)）。
- Enterprise Edition 目录中也**没有**发现 blocklist / suppression 相关模块。
- **结论**：当前版本中，bounce 事件仅作为执行详情记录，不会自动触发地址清理或停发。这是社区版与典型 ESP（如 SendGrid 的 Suppression List）之间的功能差距。

---

## 5. 黑名单批处理

### 5.1 DNS 黑名单（DNSBL）检查

**文件**: [dns-diagnostics.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/utils/dns-diagnostics.ts#L1-L91)

- **DNSBL 区域**：`zen.spamhaus.org`、`b.barracudacentral.org`、`dnsbl.sorbs.net`。
- 查询方式：将 IP 反转后拼接 zone 名称（如 `1.0.0.127.zen.spamhaus.org`），解析 A 记录，若返回 `127.*` 地址则视为已列入黑名单。
- 流程：
  1. `resolveHostnameToIpv4`：将邮件服务器域名解析为 IPv4 地址列表。
  2. `isPrivateOrLoopbackIpv4`：跳过私有/回环地址（10.x, 172.16-31.x, 192.168.x, 127.x, 169.254.x, 0.x）。
  3. `isIpv4ListedOnDnsblZone`：逐 IP 逐 zone 查询，带 5s 超时保护。
  4. `checkMailServerIpsOnDnsbl`：汇总所有命中结果。

### 5.2 诊断中的 DNSBL 检查

**文件**: [diagnose-domain.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/api/src/app/domains/usecases/diagnose-domain/diagnose-domain.usecase.ts#L243-L305)

- 作为域名诊断的第三阶段检查项（`DNSBL_LISTED`）。
- 仅在 MX 记录存在且邮件服务器 IP 非私有地址时执行。
- 命中 DNSBL 时 severity 为 **WARN**（非 ERROR），建议联系邮件基础设施提供商处理。
- **不触发自动下线**，仅作为诊断信息返回给用户。

### 5.3 收件人级别黑名单

- Community Edition 中**不存在**收件人级别的邮箱黑名单或 suppression list。
- 不存在批量导入/导出黑名单的 API。
- 不存在基于 bounce 频率自动添加黑名单的逻辑。

---

## 6. 软退信反馈

### 6.1 事件状态枚举

**文件**: [provider.interface.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/stateless/src/lib/provider/provider.interface.ts#L129-L143)

```typescript
export enum EmailEventStatusEnum {
  OPENED = 'opened',
  REJECTED = 'rejected',
  SENT = 'sent',
  DEFERRED = 'deferred',      // 软退信的中间状态
  DELIVERED = 'delivered',
  BOUNCED = 'bounced',         // 硬退信 + 软退信统一映射
  DROPPED = 'dropped',
  CLICKED = 'clicked',
  BLOCKED = 'blocked',
  SPAM = 'spam',
  UNSUBSCRIBED = 'unsubscribed',
  DELAYED = 'delayed',         // Resend 的 delivery_delayed
  COMPLAINT = 'complaint',
}
```

**关键发现**：Novu **不区分**硬退信和软退信 — 两者都映射到 `BOUNCED`。

### 6.2 Provider 级别事件映射

| Provider | 硬退信 | 软退信 | 映射目标 |
|---|---|---|---|
| **Mandrill** | `hard_bounce` → `BOUNCED` | `soft_bounce` → `BOUNCED` | [mandrill.provider.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/mandrill/mandrill.provider.ts#L190-L215) |
| **SendGrid** | `bounce` → `BOUNCED` | 无独立软退信事件 | [sendgrid.provider.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/sendgrid/sendgrid.provider.ts#L341-L358) |
| **Mailgun** | `permanent_fail` → `REJECTED` | `failed` → `REJECTED` | [mailgun.provider.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/mailgun/mailgun.provider.ts#L320-L338) |
| **Mailjet** | `bounce` → `BOUNCED` | 无独立软退信事件 | [mailjet.provider.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/mailjet/mailjet.provider.ts#L166-L185) |
| **Resend** | `email.bounced` → `BOUNCED` | `email.delivery_delayed` → `DELAYED` | [resend.provider.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/resend/resend.provider.ts#L172-L193) |

**Mandrill 的 MandrillStatusEnum**（[mandrill.provider.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/packages/providers/src/lib/email/mandrill/mandrill.provider.ts#L17-L28)）是唯一一个在 provider 层面区分 hard_bounce 和 soft_bounce 的实现，但映射到 Novu 内部时仍然统一为 `BOUNCED`。

### 6.3 Webhook 事件处理链路

```
Provider 回调 → apps/webhook → Webhook.usecase → parseEvents → CreateExecutionDetails → 写入 ExecutionDetails 集合
```

**文件**: [webhook.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts#L24-L71)

1. **Webhook 入口**：根据 `providerOrIntegrationId` 查找 Integration，创建对应 Provider 实例。
2. **解析事件**：调用 `provider.getMessageId(body)` 获取消息标识列表，逐条 `provider.parseEventBody(body, messageId)` 解析事件。
3. **记录执行详情**：每条事件调用 `CreateExecutionDetails.execute`，写入 `ExecutionDetails` 集合（MongoDB）。

**文件**: [create-execution-details.usecase.ts (webhook)](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/apps/webhook/src/webhooks/usecases/execution-details/create-execution-details.usecase.ts#L60-L101)

- `mapEmailStatus` 将事件状态映射为 `ExecutionDetailsStatusEnum`：
  - `BOUNCED` / `REJECTED` / `DROPPED` / `BLOCKED` / `SPAM` / `UNSUBSCRIBED` → **FAILED**
  - `DEFERRED` → **PENDING**（软退信中间状态保留待定语义）
  - `DELIVERED` / `SENT` / `OPENED` / `CLICKED` → **SUCCESS**

**文件**: [create-execution-details.usecase.ts (application-generic)](file:///d:/fz/0601-2/solo-dogfeeding/code/57-novu/libs/application-generic/src/usecases/create-execution-details/create-execution-details.usecase.ts#L150-L237)

- 双写策略：受 `IS_EXECUTION_DETAILS_CLICKHOUSE_ONLY_ENABLED` Feature Flag 控制：
  - **MongoDB** (`ExecutionDetails` 集合)：默认写入，flag 开启后跳过。
  - **ClickHouse** (`trace_log` 表)：始终写入，用于分析查询。
- 事件类型映射：`MESSAGE_BOUNCED` → `'message_bounced'`，`MESSAGE_DEFERRED` → `'message_deferred'`。

### 6.4 软退信处理的局限性

1. **不区分硬/软退信**：Mandrill 的 `soft_bounce` 和 `hard_bounce` 在 Novu 内部都被映射为 `BOUNCED`，无法区分暂时性失败（邮箱满、服务商限流）和永久性失败（地址不存在）。
2. **无重试逻辑差异化**：`DEFERRED` 状态映射为 `PENDING`，理论上支持重试，但当前 webhook 处理链路仅做记录，不触发重试。
3. **无自动 suppression**：bounce 事件不会自动将 subscriber 标记为不可达或加入停发列表。
4. **无退信反馈环**：没有将退信信息回传给上游触发方的机制（仅通过 Activity Feed 展示）。

---

## 整体数据流总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Syntax 校验层                                 │
│  ┌──────────────────┐  ┌────────────────────┐  ┌─────────────────┐ │
│  │ email-local-part │  │ agents email       │  │ email           │ │
│  │ .validator.ts    │  │ -normalization.ts  │  │ -normalization  │ │
│  │ (RFC 5321 regex) │  │ (简易格式检查)       │  │ (provider去重)  │ │
│  └──────────────────┘  └────────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     发送前检查 (send-message-email)                   │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ SUBSCRIBER_MISSING_EMAIL_ADDRESS → SKIPPED                    │ │
│  │ SUBSCRIBER_NO_ACTIVE_INTEGRATION → FAILED                     │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Provider 发送 & 回调                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────────┐  │
│  │SendGrid │ │Mandrill │ │Mailgun  │ │Mailjet  │ │Resend      │  │
│  │bounce→  │ │hard→    │ │perm_fail│ │bounce→  │ │bounced→    │  │
│  │BOUNCED  │ │BOUNCED  │ │→REJECTED│ │BOUNCED  │ │BOUNCED     │  │
│  │         │ │soft→    │ │failed→  │ │         │ │delayed→    │  │
│  │         │ │BOUNCED  │ │REJECTED │ │         │ │DELAYED     │  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └─────┬──────┘  │
└───────┼───────────┼───────────┼───────────┼─────────────┼─────────┘
        │           │           │           │             │
        └───────────┴───────────┴───────────┴─────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Webhook 事件处理 (apps/webhook)                      │
│  Webhook.usecase → parseEvents → CreateExecutionDetails             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ BOUNCED/REJECTED/DROPPED/BLOCKED/SPAM → FAILED             │   │
│  │ DEFERRED → PENDING                                         │   │
│  │ DELIVERED/SENT/OPENED/CLICKED → SUCCESS                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  双写：MongoDB (ExecutionDetails) + ClickHouse (trace_log)          │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  入站域名验证 (apps/api/domains)                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ get-mx-record    │  │ diagnose-domain  │  │ dns-diagnostics  │ │
│  │ .usecase.ts      │  │ .usecase.ts      │  │ .ts              │ │
│  │ (入站MX校验)      │  │ (域名诊断三阶段)   │  │ (DNSBL/超时/IP) │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│  ⚠ DNS 查询无缓存层，每次实时调用 node:dns                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键发现与改进建议

| 领域 | 现状 | 建议 |
|---|---|---|
| **Syntax** | 分散在 3 处，职责清晰但标准不统一 | 统一为单一 `isValidEmail()` 工具函数，支持 full address + local-part 两种模式 |
| **MX 查询** | 每次实时查询，无缓存 | 引入短 TTL LRU 缓存（60s），减少 DNS 延迟 |
| **DNS 缓存** | `InMemoryLRUCacheService` 存在但未用于 DNS | 新增 `DNS_MX` / `DNS_NS` store 配置，复用现有 inflight 去重机制 |
| **无效地址清理** | 仅记录执行详情，无自动清理 | 引入 bounce 后自动 suppression 机制（可由 Enterprise 门控） |
| **黑名单批处理** | 仅 DNSBL 诊断检查（出站 IP），无收件人级别黑名单 | 增加 subscriber-level suppression list，支持批量导入和 bounce 自动填充 |
| **软退信** | hard/soft bounce 统一为 `BOUNCED`，无差异化处理 | 在 `EmailEventStatusEnum` 中新增 `SOFT_BOUNCED`，对 DEFERRED 实现自动重试策略 |
