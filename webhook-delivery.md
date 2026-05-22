# Webhook 回调机制深度分析

本文档详细分析 Novu 平台向用户回调 webhook 的完整机制，包括事件流转、签名生成、失败重试与去重机制，以及接收方如何验证身份与回溯历史投递。

## 1. 架构概述

Novu 存在 **两条独立的签名链路**，用于不同的通信场景：

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              出站 Webhook (Svix)                          │
│  业务事件 → SendWebhookMessage → Svix API → svix-signature → 用户回调端点   │
│     (MESSAGE_SENT, WORKFLOW_DELETED 等 17 种事件)                          │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                         入站 Bridge (novu-signature)                      │
│  Novu 平台 → ExecuteFrameworkRequest → novu-signature → 用户 Bridge 端点   │
│     (工作流发现、执行、预览等内部通信)                                      │
└───────────────────────────────────────────────────────────────────────────┘
```

**核心代码位置**：
- `libs/application-generic/src/webhooks/` - 核心 webhook 逻辑
- `apps/api/src/app/outbound-webhooks/` - Webhook Portal API
- `packages/shared/src/webhooks/` - 事件类型定义
- `packages/framework/src/handler.ts` - Bridge 签名验证
- `libs/application-generic/src/utils/hmac.ts` - novu-signature 签名生成

---

## 2. 事件语义与触发矩阵

### 2.1 事件类型定义

**文件**：`packages/shared/src/webhooks/webhook-event.enum.ts:1-33`

```typescript
export enum WebhookEventEnum {
  // Workflow
  WORKFLOW_CREATED = 'workflow.created',
  WORKFLOW_UPDATED = 'workflow.updated',
  WORKFLOW_DELETED = 'workflow.deleted',
  WORKFLOW_PUBLISHED = 'workflow.published',

  // Message
  MESSAGE_SENT = 'message.sent',
  MESSAGE_FAILED = 'message.failed',
  MESSAGE_DELIVERED = 'message.delivered',
  MESSAGE_SEEN = 'message.seen',
  MESSAGE_READ = 'message.read',
  MESSAGE_UNREAD = 'message.unread',
  MESSAGE_ARCHIVED = 'message.archived',
  MESSAGE_UNARCHIVED = 'message.unarchived',
  MESSAGE_SNOOZED = 'message.snoozed',
  MESSAGE_UNSNOOZED = 'message.unsnoozed',
  MESSAGE_DELETED = 'message.deleted',

  // Preference
  PREFERENCE_UPDATED = 'preference.updated',

  // Email Inbound
  EMAIL_RECEIVED = 'email.received',
}
```

### 2.2 完整触发矩阵

| 事件类型 | 触发场景 | 触发文件位置 | Payload 字段 |
|---------|---------|-------------|-------------|
| **WORKFLOW_CREATED** | 创建新工作流 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:113-121` | `object` |
| **WORKFLOW_UPDATED** | 更新工作流 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:123-134` | `object`, `previousObject` |
| **WORKFLOW_DELETED** | 删除工作流 | `apps/api/src/app/workflows-v1/usecases/delete-workflow/delete-workflow.usecase.ts:50-58` | `object` |
| **WORKFLOW_PUBLISHED** | 工作流同步/发布到环境 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:149-160` | `object`, `previousObject` |
| **MESSAGE_SENT** | 消息发送成功（全渠道） | Email: `send-message-email.usecase.ts:487-497`<br>SMS: `send-message-sms.usecase.ts:356-370`<br>Push: `send-message-push.usecase.ts:644-655`<br>In-App: `send-message-in-app.usecase.ts:307-318`<br>Chat: `send-message-chat.usecase.ts:805-818` | `object`, `providerResponseId` |
| **MESSAGE_FAILED** | 消息发送失败（仅 Email/SMS/Push） | Email: `send-message-email.usecase.ts:553-564`<br>SMS: `send-message-sms.usecase.ts:372-382`<br>Push: `send-message-push.usecase.ts:726-741` | `object`, `error` |
| **MESSAGE_SENT** (⚠️ Chat 失败特例) | **Chat 发送失败** 时错误使用 | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:854-867` | `object`, `error` |
| **MESSAGE_DELIVERED** | 第三方回调投递成功 | `apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts` (provider 驱动) | - |
| **MESSAGE_SEEN** | 用户查看消息 | `apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts:77-105` | `object` |
| **MESSAGE_READ** / **MESSAGE_UNREAD** | 用户标记消息已读/未读 | 批量: `apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:83-86`<br>单个: `mark-notification-as.usecase.ts` (调用批量)<br>按条件: `update-all-notifications.usecase.ts:121-124` | `object` |
| **MESSAGE_ARCHIVED** / **MESSAGE_UNARCHIVED** | 用户归档/取消归档 | 批量: `mark-many-notifications-as.usecase.ts:88-91`<br>按条件: `update-all-notifications.usecase.ts:126-129` | `object` |
| **MESSAGE_SNOOZED** / **MESSAGE_UNSNOOZED** | 用户稍后提醒/取消 | 批量: `mark-many-notifications-as.usecase.ts:93-97`<br>Snooze: `snooze-notification.usecase.ts` (调用 `markNotificationAsSnoozed` → 最终调用批量)<br>Unsnooze: `unsnooze-notification.usecase.ts` (调用批量)<br>定时唤醒: `process-unsnooze-job.usecase.ts` | `object` |
| **MESSAGE_DELETED** | 用户删除消息 | 批量删除: `apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts:77, 120-137`<br>按条件删除: `delete-all-notifications.usecase.ts:115, 147-164` | `object` |
| **PREFERENCE_UPDATED** | 用户更新通知偏好 | `apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts:78-88` | `object` |
| **EMAIL_RECEIVED** | 入站邮件到达 | `libs/application-generic/src/usecases/inbound-domain-route-delivery/inbound-domain-route-delivery.usecase.ts:121-127` | `object` (domain, route, mail) |

### 2.3 重点路径详细分析

#### 2.3.1 ⚠️ Chat 发送失败事件类型不一致

**文件**：`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:825-873`

**发现问题**：Chat 渠道发送失败时，使用的事件类型是 `MESSAGE_SENT` 而非 `MESSAGE_FAILED`，这与其他渠道（Email/SMS/Push）不一致。

```typescript
// Chat 失败时的代码（错误实现）
private async handleMessageSendError(...) {
  await this.sendWebhookMessage.execute({
    eventType: WebhookEventEnum.MESSAGE_SENT,  // ❌ 应该是 MESSAGE_FAILED
    objectType: WebhookObjectTypeEnum.MESSAGE,
    payload: {
      object: messageWebhookMapper(message, command.subscriberId, { channelData: redactedChannelData }),
      error: {
        message: this.getErrorMessage(error) || 'Error while sending chat with provider',
      },
    },
    organizationId: command.organizationId,
    environmentId: command.environmentId,
  });
}

// 对比 Email 失败时的正确实现
await this.sendWebhookMessage.execute({
  eventType: WebhookEventEnum.MESSAGE_FAILED,  // ✅ 正确
  ...
});
```

**接收方注意**：处理 Chat 渠道的失败事件时，需要检查 `eventType === 'message.sent'` 且存在 `error` 字段，才能判定为发送失败。

#### 2.3.2 工作流删除

**文件**：`apps/api/src/app/workflows-v1/usecases/delete-workflow/delete-workflow.usecase.ts:39-59`

触发时机：用户调用 API 删除工作流时，在删除相关实体（控制值、消息模板、偏好、翻译组）之后触发。

```typescript
async execute(command: DeleteWorkflowCommand): Promise<void> {
  const workflowEntity = await this.getWorkflowByIdsUseCase.execute(...);

  await this.deleteRelatedEntities(command, workflowEntity);  // 先删除实体

  await this.sendWebhookMessage.execute({
    eventType: WebhookEventEnum.WORKFLOW_DELETED,
    objectType: WebhookObjectTypeEnum.WORKFLOW,
    payload: { object: workflowEntity },  // 包含已删除工作流的完整信息
    organizationId: command.organizationId,
    environmentId: command.environmentId,
  });
}
```

#### 2.3.3 工作流发布/同步

**文件**：`apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:149-160`

触发时机：工作流从一个环境同步到另一个环境（如开发 → 生产）时。

```typescript
// 更新源工作流的发布信息
await this.notificationTemplateRepository.updatePublishFields(
  sourceWorkflow._id,
  command.user.environmentId,
  command.user._id,
  command.session
);

if (this.sendWebhookMessage) {
  await this.sendWebhookMessage.execute({
    eventType: WebhookEventEnum.WORKFLOW_PUBLISHED,
    objectType: WebhookObjectTypeEnum.WORKFLOW,
    payload: {
      object: upsertedWorkflow,        // 目标环境的新工作流状态
      previousObject: sourceWorkflow,  // 源环境的原始工作流状态
    },
    organizationId: command.user.organizationId,
    environmentId: command.user.environmentId,  // 注意：是源环境 ID
  });
}
```

#### 2.3.4 消息归档/取消归档

**文件**：`apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:81-99`

触发流程：
1. `MarkNotificationAs`（单个）→ 调用 `MarkManyNotificationsAs`（批量）
2. 批量更新消息状态
3. 按 `archived` 值决定触发 `MESSAGE_ARCHIVED` 或 `MESSAGE_UNARCHIVED`

```typescript
const eventTypes: WebhookEventEnum[] = [];

if (command.read !== undefined) {
  const eventType = command.read ? WebhookEventEnum.MESSAGE_READ : WebhookEventEnum.MESSAGE_UNREAD;
  eventTypes.push(eventType);
}

if (command.archived !== undefined) {
  const eventType = command.archived ? WebhookEventEnum.MESSAGE_ARCHIVED : WebhookEventEnum.MESSAGE_UNARCHIVED;
  eventTypes.push(eventType);
}

if (command.snoozedUntil !== undefined) {
  const eventType = command.snoozedUntil ? WebhookEventEnum.MESSAGE_SNOOZED : WebhookEventEnum.MESSAGE_UNSNOOZED;
  eventTypes.push(eventType);
}

// 批量发送 webhook（每批 100 条）
await this.processWebhooksInBatches(eventTypes, updatedMessages, command, environment);
```

#### 2.3.5 消息稍后提醒（Snooze/Unsnooze）

**触发场景**：
1. **用户主动 Snooze**：`SnoozeNotification.execute()` → 创建定时 Unsnooze Job → 调用 `markNotificationAsSnoozed()` → `MarkManyNotificationsAs` 触发 `MESSAGE_SNOOZED`
   - 文件：`apps/api/src/app/inbox/usecases/snooze-notification/snooze-notification.usecase.ts:60-64`

2. **用户主动 Unsnooze**：`UnsnoozeNotification.execute()` → 删除定时 Job → 调用 `markNotificationAs()` → `MarkManyNotificationsAs` 触发 `MESSAGE_UNSNOOZED`
   - 文件：`apps/api/src/app/inbox/usecases/unsnooze-notification/unsnooze-notification.usecase.ts:58-77`

3. **定时自动唤醒**：`ProcessUnsnoozeJob.execute()` → 更新 `snoozedUntil: null` → 触发 `MESSAGE_UNSNOOZED`
   - 文件：`apps/worker/src/app/workflow/usecases/process-unsnooze-job/process-unsnooze-job.usecase.ts`

#### 2.3.6 消息删除

**批量删除**：`DeleteManyNotifications.execute()` → 删除数据库记录 → 触发 `MESSAGE_DELETED`
- 文件：`apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts:47-77`

**按条件删除**：`DeleteAllNotifications.execute()` → 按过滤器删除 → 触发 `MESSAGE_DELETED`
- 文件：`apps/api/src/app/inbox/usecases/delete-all-notifications/delete-all-notifications.usecase.ts:69-115`

**批量处理策略**：每 100 条消息为一批，并发发送 webhook。

#### 2.3.7 入站邮件

**文件**：`libs/application-generic/src/usecases/inbound-domain-route-delivery/inbound-domain-route-delivery.usecase.ts:112-133`

触发时机：用户回复邮件（Reply-to 或入站域名路由）时。

```typescript
async deliverToWebhook(params: {
  environmentId: string;
  organizationId: string;
  domain: RoutableDomain;
  route: DomainRouteEntity;
  mail: InboundDomainRouteMailInput;
}): Promise<{ latencyMs: number; skipped: boolean }> {
  const payload = this.buildDomainRouteWebhookPayload(params.domain, params.route, params.mail);
  const result = await this.sendWebhookMessage.execute({
    environmentId: params.environmentId,
    organizationId: params.organizationId,
    eventType: WebhookEventEnum.EMAIL_RECEIVED,
    objectType: WebhookObjectTypeEnum.EMAIL_INBOUND,
    payload: { object: payload },
  });
  return { latencyMs: Date.now() - started, skipped: result === undefined };
}
```

**入站邮件 Payload 结构**：
```typescript
{
  domain: { id, name, data },       // 域名配置
  route: { address, data },         // 路由配置
  mail: {                           // 邮件内容
    from, to, subject, html, text,
    headers, attachments, messageId,
    inReplyTo, references, date, cc
  }
}
```

### 2.4 事件投递流程

**核心文件**：`libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts:20-80`

```typescript
async execute(command: SendWebhookMessageCommand): Promise<{ eventId: string } | undefined> {
  // 1. 检查 Svix 客户端是否可用
  if (!this.svix) return;

  // 2. 获取环境配置，检查 webhookAppId（优先使用传入的 environment，减少 DB 查询）
  const environment = command.environment || await this.environmentRepository.findOne({ _id: command.environmentId });
  const appId = environment.webhookAppId;
  if (!appId) return;

  // 3. 生成唯一事件 ID
  const eventId = `evt_${generateObjectId()}`;

  // 4. 构造标准 payload
  const webhookPayload: WrapperDto<any> = {
    id: eventId,
    type: command.eventType,
    object: command.objectType,
    data: command.payload,
    timestamp: new Date().toISOString(),
    environmentId: environment.identifier,
  };

  // 5. 通过 Svix 发送
  await this.svix.message.create(appId, {
    eventType: command.eventType,
    eventId,
    payload: webhookPayload,
  });

  return { eventId };
}
```

### 2.5 Payload 结构

**文件**：`libs/application-generic/src/webhooks/dtos/webhook-payload.dto.ts:4-40`

```typescript
export class WrapperDto<T> {
  id: string;                    // 事件唯一标识 evt_xxx
  type: WebhookEventEnum;        // 事件类型
  object: WebhookObjectTypeEnum; // 对象类型
  data: T;                       // 事件数据
  timestamp: string;             // ISO 时间戳
  environmentId: string;         // 环境标识
}
```

### 2.6 消息 Mapper

**文件**：`libs/application-generic/src/webhooks/mappers/message.mapper.ts:5-79`

`messageWebhookMapper` 函数负责将内部 `MessageEntity` 映射为对外暴露的 `MessageWebhookResponseDto`，包含字段：
- 基础字段：`_id`, `_templateId`, `_environmentId`, `_organizationId`, `_notificationId`
- 状态字段：`status`, `seen`, `read`, `archived`, `snoozedUntil`
- 时间字段：`createdAt`, `updatedAt`, `deliveredAt`, `lastSeenDate`, `lastReadDate`
- 其他：`subscriberId`, `transactionId`, `channel`, `providerId`, `errorId`, `errorText`, `contextKeys`

---

## 3. 两条签名链路深度对比

### 3.1 链路概览

| 维度 | Svix 出站签名 | Framework novu-signature 验签 |
|-----|--------------|-----------------------------|
| **使用场景** | Novu 向用户回调业务事件（17 种 webhook 事件） | Novu 调用用户 Bridge 端点（工作流发现、执行、预览） |
| **签名头部** | `svix-signature` | `novu-signature` |
| **签名格式** | `t=<timestamp>,v1=<signature>` | `t=<unix-ms>,v1=<hex-hmac>` |
| **签名算法** | HMAC-SHA256 | HMAC-SHA256 |
| **签名密钥** | Svix 为每个 endpoint 生成的 Signing Secret | 用户环境的 `secretKey`（Novu 平台存储，加密保存） |
| **签名生成方** | Svix 服务自动生成 | Novu 平台（`buildNovuSignatureHeader`） |
| **签名验证方** | 用户代码（使用 Svix SDK） | Novu Framework Handler（用户 Bridge 端） |
| **时间戳容忍度** | Svix 默认 5 分钟 | Novu Framework 固定 5 分钟 |
| **重放防护** | ✅ 时间戳 + 事件 ID | ✅ 时间戳 |
| **恒时比较** | ✅ Svix SDK 实现 | ✅ `timingSafeEqual` 手动实现 |
| **开关控制** | `SVIX_API_KEY` 环境变量 | `strictAuthentication` 开关 |

### 3.2 链路一：Svix 出站签名（用户接收 Novu 事件）

**签名生成**：由 Svix 服务在投递时自动生成，使用用户在 Svix Portal 中配置的 Signing Secret。

**签名验证**（用户端实现）：
```javascript
import { Webhook } from 'svix';

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post('/webhook', (req, res) => {
  const payload = JSON.stringify(req.body);
  const headers = req.headers;

  try {
    const verified = webhook.verify(payload, headers);
    res.status(200).send('OK');
  } catch (err) {
    res.status(400).send('Invalid signature');
  }
});
```

**Svix 客户端初始化**：`libs/application-generic/src/webhooks/services/svix-provider.service.ts:1-18`

```typescript
export const SvixProviderService: Provider<SvixClient> = {
  provide: 'SVIX_CLIENT',
  useFactory: (): SvixClient => {
    const apiKey = process.env.SVIX_API_KEY;
    if (!apiKey) return null;  // 未配置则禁用 webhook 功能
    return new Svix(apiKey);
  },
};
```

### 3.3 链路二：novu-signature 验签（Novu 调用用户 Bridge）

#### 3.3.1 签名生成（Novu 平台端）

**文件**：`libs/application-generic/src/utils/hmac.ts:6-12`

```typescript
export function buildNovuSignatureHeader(secretKey: string, payload: unknown): string {
  const timestamp = Date.now();
  const publicKey = `${timestamp}.${JSON.stringify(payload)}`;
  const hmac = createHmac('sha256', secretKey).update(publicKey).digest('hex');

  return `t=${timestamp},v1=${hmac}`;
}
```

**调用点**：`libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts:140-148`

```typescript
private async buildRequestHeaders(command: ExecuteBridgeRequestCommand) {
  const novuSignatureHeader = await this.buildRequestSignature(command);

  return {
    [HttpRequestHeaderKeysEnum.BYPASS_TUNNEL_REMINDER]: 'true',
    [HttpRequestHeaderKeysEnum.CONTENT_TYPE]: 'application/json',
    [HttpHeaderKeysEnum.NOVU_SIGNATURE]: novuSignatureHeader,  // 注入签名
  };
}

private async buildRequestSignature(command: ExecuteBridgeRequestCommand) {
  const secretKey = await this.getDecryptedSecretKey.execute(
    GetDecryptedSecretKeyCommand.create({ environmentId: command.environmentId })
  );

  return buildNovuSignatureHeader(secretKey, command.event || {});
}
```

#### 3.3.2 签名验证（用户 Bridge 端）

**文件**：`packages/framework/src/handler.ts:371-396`

```typescript
private async validateHmac(payload: unknown, hmacHeader: string | null): Promise<void> {
  // 1. 检查是否启用 HMAC（受 strictAuthentication 控制）
  if (!this.hmacEnabled) return;

  // 2. 检查签名头部是否存在
  if (!hmacHeader) throw new SignatureNotFoundError();

  // 3. 检查签名密钥是否配置
  if (!this.client.secretKey) throw new SigningKeyNotFoundError();

  // 4. 解析签名头部
  const parsed = parseSignatureHeader(hmacHeader);
  if (!parsed.v1 || parsed.t === undefined) throw new SignatureInvalidError();

  // 5. 检查时间戳（防重放攻击，5分钟容忍度）
  const now = Date.now();
  if (parsed.t < now - SIGNATURE_TIMESTAMP_TOLERANCE || parsed.t > now + SIGNATURE_TIMESTAMP_TOLERANCE) {
    throw new SignatureExpiredError();
  }

  // 6. 计算本地签名（使用 Web Crypto API，跨平台兼容）
  const localHash = await createHmacSubtle(
    this.client.secretKey,
    `${parsed.t}.${JSON.stringify(payload)}`
  );

  // 7. 恒定时间比较（防时序攻击）
  if (!timingSafeEqual(localHash, parsed.v1)) {
    throw new SignatureMismatchError();
  }
}
```

**时间戳容忍度**：`packages/framework/src/constants/api.constants.ts:6-7`
```typescript
export const SIGNATURE_TIMESTAMP_TOLERANCE_MINUTES = 5;
export const SIGNATURE_TIMESTAMP_TOLERANCE = SIGNATURE_TIMESTAMP_TOLERANCE_MINUTES * 60 * 1000; // 5分钟
```

#### 3.3.3 核心加密工具

**文件**：`packages/framework/src/utils/crypto.utils.ts:11-60`

##### HMAC 生成（跨平台兼容 Web Crypto API）
```typescript
export const createHmacSubtle = async (secretKey: string, data: string): Promise<string> => {
  const encoder = new TextEncoder();
  const keyData = encoder.encode(secretKey);
  const dataBuffer = encoder.encode(data);

  const cryptoKey = await crypto.subtle.importKey(
    'raw', keyData,
    { name: 'HMAC', hash: { name: 'SHA-256' } },
    false, ['sign']
  );

  const signature = await crypto.subtle.sign('HMAC', cryptoKey, dataBuffer);
  return Array.from(new Uint8Array(signature))
    .map((byte) => byte.toString(16).padStart(2, '0'))
    .join('');
};
```

##### 恒定时间比较（防时序攻击）
```typescript
export const timingSafeEqual = (a: string, b: string): boolean => {
  if (typeof a !== 'string' || typeof b !== 'string') return false;
  if (a.length !== b.length) return false;

  let mismatch = 0;
  for (let i = 0; i < a.length; i += 1) {
    mismatch |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return mismatch === 0;
};
```

##### 签名头部解析
**文件**：`packages/framework/src/handler.ts:415-439`

```typescript
function parseSignatureHeader(header: string): ParsedSignatureHeader {
  const fields: Record<string, string> = {};
  for (const rawPart of header.split(',')) {
    const part = rawPart.trim();
    if (!part) continue;
    const eqIdx = part.indexOf('=');
    if (eqIdx <= 0) continue;
    const key = part.slice(0, eqIdx);
    const value = part.slice(eqIdx + 1);
    if (key && value && !(key in fields)) fields[key] = value;
  }
  return { t: Number.isFinite(Number(fields.t)) ? Number(fields.t) : undefined, v1: fields.v1 };
}
```

#### 3.3.4 签名错误类型

**文件**：`packages/framework/src/errors/signature.errors.ts:4-57`

| 错误类型 | 场景 | HTTP 状态码 |
|---------|------|------------|
| `SignatureNotFoundError` | 请求缺少 `novu-signature` 头部 | 401 |
| `SignatureInvalidError` | 签名格式不正确 | 401 |
| `SignatureExpiredError` | 签名时间戳超出 5 分钟容忍范围 | 401 |
| `SignatureMismatchError` | 签名验证失败 | 401 |
| `SigningKeyNotFoundError` | 服务端未配置签名密钥 | 500 |
| `SignatureVersionInvalidError` | 签名版本不支持 | 401 |

---

## 4. strictAuthentication 开关深度分析

### 4.1 开关定义

**文件**：`packages/framework/src/types/config.types.ts:15-25`

```typescript
export type ClientOptions = {
  /**
   * Explicitly use HMAC signature verification.
   * Setting this to `false` will enable Novu to communicate with your Bridge API
   * without requiring a valid HMAC signature.
   * This is useful for local development and testing.
   *
   * In production you must specify an `secretKey` and set this to `true`.
   *
   * Defaults to true.
   */
  strictAuthentication?: boolean;
};
```

### 4.2 开关优先级与默认值

**文件**：`packages/framework/src/client.ts:53-95`

```typescript
function isRuntimeInDevelopment() {
  return ['development', undefined, 'dev'].includes(process.env.NODE_ENV);
}

private buildOptions(providedOptions?: ClientOptions) {
  const builtConfiguration: Required<ClientOptions> = {
    apiUrl: resolveApiUrl(providedOptions?.apiUrl),
    secretKey: resolveSecretKey(providedOptions?.secretKey),
    strictAuthentication: !isRuntimeInDevelopment(),  // 默认：开发环境 false，生产 true
    verbose: isRuntimeInDevelopment(),
  };

  // 优先级 1：显式传入的参数
  if (providedOptions?.strictAuthentication !== undefined) {
    builtConfiguration.strictAuthentication = providedOptions.strictAuthentication;
  }
  // 优先级 2：环境变量 NOVU_STRICT_AUTHENTICATION_ENABLED
  else if (process.env.NOVU_STRICT_AUTHENTICATION_ENABLED !== undefined) {
    builtConfiguration.strictAuthentication = process.env.NOVU_STRICT_AUTHENTICATION_ENABLED === 'true';
  }
  // 优先级 3：默认值（!isRuntimeInDevelopment()）

  return builtConfiguration;
}
```

**优先级总结**：
1. **最高**：显式传入 `strictAuthentication` 参数
2. **其次**：环境变量 `NOVU_STRICT_AUTHENTICATION_ENABLED`
3. **最低**：默认值（`NODE_ENV` 为 `development`/`dev`/`undefined` 时为 `false`，否则为 `true`）

### 4.3 开关影响范围

**文件**：`packages/framework/src/handler.ts:80-88`

```typescript
constructor(options: INovuRequestHandlerOptions<Input, Output>) {
  this.handler = options.handler;
  this.client = options.client ? options.client : new Client();
  this.workflows = options.workflows || [];
  this.agents = options.agents || [];
  this.http = initApiClient(this.client.secretKey, this.client.apiUrl);
  this.frameworkName = options.frameworkName;
  this.hmacEnabled = this.client.strictAuthentication;  // ⚠️ 直接关联
  this.client.addAgents(this.agents);
}
```

**当 `strictAuthentication = false` 时**：
- `this.hmacEnabled = false`
- `validateHmac()` 方法直接 `return`，跳过所有签名验证
- 即使请求不带 `novu-signature` 头部也能通过
- **仅用于开发/测试环境**

**当 `strictAuthentication = true` 时**：
- `this.hmacEnabled = true`
- 所有请求必须携带有效的 `novu-signature` 头部
- 缺少头部、签名过期、签名不匹配都会抛出相应错误

### 4.4 生产环境强制启用

**文件**：`apps/api/src/app/environments-v1/novu-bridge-client.ts:127-132`

Novu 托管的 Bridge 端点强制启用严格认证：

```typescript
const novuRequestHandler = new NovuRequestHandler({
  frameworkName,
  workflows,
  client: new Client({ secretKey, strictAuthentication: true, verbose: false }),  // 强制 true
  handler: this.novuHandler.handler,
});
```

### 4.5 测试环境禁用

**文件**：`apps/dashboard/tests/utils/test-bridge-server.ts:14`

```typescript
this.client = new Client({ strictAuthentication: false, secretKey, apiUrl });
```

**文件**：`apps/api/e2e/test-bridge-server.ts:9`

```typescript
public client = new Client({ strictAuthentication: false });
```

---

## 5. 失败重试与去重机制

### 5.1 出站 Webhook 重试（Svix 自动处理）

Svix 作为专业的 webhook 服务提供完善的重试机制：

**重试策略**：指数退避（Exponential Backoff）
**重试次数**：Svix 默认最多重试 25 次，时间跨度约 3 天
**成功判定**：接收方返回 2xx 状态码视为成功

### 5.2 内部 Webhook Filter 重试策略

**文件**：`apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts:11-36`

用于工作流执行中的 webhook filter 步骤重试：

```typescript
public async execute(command: WebhookFilterBackoffStrategyCommand): Promise<number> {
  const { attemptsMade, eventError: error, eventJob } = command;

  // 记录重试执行详情
  await this.createExecutionDetails.execute(
    CreateExecutionDetailsCommand.create({
      ...CreateExecutionDetailsCommand.getDetailsFromJob(job),
      detail: DetailEnum.WEBHOOK_FILTER_FAILED_RETRY,
      source: ExecutionDetailsSourceEnum.WEBHOOK,
      status: ExecutionDetailsStatusEnum.PENDING,
      isTest: false,
      isRetry: true,
      raw: JSON.stringify({ message: JSON.parse(error?.message).message, attempt: attemptsMade }),
    })
  );

  // 指数退避 + 随机抖动
  return Math.round(Math.random() * 2 ** attemptsMade * 1000); // 毫秒
}
```

**重试间隔计算**：`随机因子 * 2^重试次数 * 1000ms`
- 第 1 次重试：0-2 秒
- 第 2 次重试：0-4 秒
- 第 3 次重试：0-8 秒
- ...

### 5.3 Bridge 请求重试

**文件**：`libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts:116-118`

```typescript
retry: {
  limit: retriesLimit,  // 默认 DEFAULT_RETRIES_LIMIT = 3
},
```

### 5.4 去重机制

#### 事件 ID 去重
**文件**：`libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts:49`

```typescript
const eventId = `evt_${generateObjectId()}`;
```

- 每个 webhook 事件生成全局唯一的 `eventId`
- 格式：`evt_` + ObjectId（24 位十六进制）
- Svix 使用 `eventId` 作为幂等键，确保同一事件不会被重复投递

#### 幂等性保证
- 发送时明确指定 `eventId` 给 Svix：`svix.message.create(appId, { eventId, ... })`
- Svix 确保对于相同的 `eventId`，即使多次调用 API 也只会投递一次

---

## 6. 接收方身份验证与历史投递回溯

### 6.1 Webhook Portal 访问管理

**文件**：`apps/api/src/app/outbound-webhooks/outbound-webhooks.controller.ts:13-57`

用户通过 Svix App Portal 管理 webhook 配置，Novu 提供 API 生成访问令牌。

#### 初始化 Webhook Portal
**API**：`POST /v2/outbound-webhooks/portal/token`

**文件**：`apps/api/src/app/outbound-webhooks/usecases/create-webhook-portal-token/create-webhook-portal.usecase.ts:16-55`

```typescript
async execute(command: CreateWebhookPortalCommand): Promise<CreateWebhookPortalResponseDto> {
  // 在 Svix 创建应用
  const app = await this.svix.application.create({
    name: organization.name,
    uid: generateWebhookAppId(command.organizationId, command.environmentId),
    metadata: {
      environmentId: command.environmentId,
      organizationId: command.organizationId,
    },
  });

  // 保存 webhookAppId 到环境配置
  await this.environmentRepository.updateOne(
    { _id: command.environmentId },
    { $set: { webhookAppId: app.uid } }
  );

  return { appId: app.uid! };
}
```

#### 获取 Portal 访问令牌
**API**：`GET /v2/outbound-webhooks/portal/token`

**文件**：`apps/api/src/app/outbound-webhooks/usecases/get-webhook-portal-token/get-webhook-portal-token.usecase.ts:15-50`

```typescript
async execute(command: GetWebhookPortalTokenCommand): Promise<GetWebhookPortalTokenResponseDto> {
  const svixResponse = await this.svix.authentication.appPortalAccess(
    environment.webhookAppId,
    {}
  );

  return {
    url: svixResponse.url,
    token: svixResponse.token,
    appId: environment.webhookAppId,
  };
}
```

### 6.2 App ID 生成规则

**文件**：`libs/application-generic/src/webhooks/utils/app-id.ts:7-9`

```typescript
export function generateWebhookAppId(organizationId: OrganizationId, environmentId: EnvironmentId): string {
  return `o-${organizationId}-e-${environmentId}`;
}
```

### 6.3 企业版 vs 社区版差异

**文件**：`apps/api/src/app/outbound-webhooks/outbound-webhooks.module.ts:14-50`

- **企业版**：完整的 Svix 集成，包含所有 webhook 功能
- **社区版**：使用 `NoopSendWebhookMessage` 空实现，webhook 功能被禁用

### 6.4 历史投递回溯

#### 通过 Event ID 追踪
每个 webhook payload 包含唯一的 `id` 字段（即 `eventId`），可用于：
1. 去重处理：接收方可以记录已处理的 `eventId`，避免重复处理
2. 问题排查：通过 `eventId` 在 Svix Portal 中查询具体的投递记录
3. 审计追踪：完整记录每次回调的时间和内容

#### Svix Portal 功能
用户通过 Svix App Portal 可以：
- 查看所有 webhook 事件的投递历史
- 检查每次投递的状态（成功/失败/重试中）
- 查看请求和响应的详细内容
- 手动重发失败的事件
- 配置回调端点 URL 和订阅的事件类型

#### Payload 中的追踪字段
```typescript
{
  "id": "evt_65abc123def456...",      // 事件唯一标识
  "type": "message.sent",              // 事件类型
  "timestamp": "2024-01-15T10:30:00.000Z", // 事件发生时间
  "environmentId": "prod-env-123",     // 环境标识
  "object": "message",                 // 对象类型
  "data": { ... }                      // 业务数据
}
```

---

## 7. 入站 Webhook 处理（平台接收第三方回调）

### 7.1 入口端点

**文件**：`apps/webhook/src/webhooks/webhooks.controller.ts:12-46`

```typescript
@Controller('/webhooks')
export class WebhooksController {
  @Post('/organizations/:organizationId/environments/:environmentId/email/:providerOrIntegrationId')
  public emailWebhook(...) { ... }

  @Post('/organizations/:organizationId/environments/:environmentId/sms/:providerOrIntegrationId')
  public smsWebhook(...) { ... }
}
```

### 7.2 处理流程

**文件**：`apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts:24-141`

```typescript
async execute(command: WebhookCommand): Promise<IWebhookResult[]> {
  // 1. 查找集成配置
  const integration = await this.integrationRepository.findOne(query);

  // 2. 创建对应 provider handler
  this.createProvider(integration, command.type);

  // 3. 从回调中提取消息 ID
  const messageIdentifiers = this.provider.getMessageId(body);

  // 4. 逐条处理事件
  for (const messageIdentifier of messageIdentifiers) {
    // 查找消息
    const message = await this.messageRepository.findOne({ identifier: messageIdentifier });

    // 解析事件类型（delivered, bounced, clicked 等）
    const event = this.provider.parseEventBody(body, messageIdentifier);

    // 创建执行详情
    await this.createExecutionDetails.execute({ message, webhook, webhookEvent, channel });
  }
}
```

### 7.3 Provider 接口要求

**文件**：`packages/stateless/src/lib/provider/provider.interface.ts`

支持 webhook 的 provider 必须实现：
- `getMessageId(body)` - 从回调中提取消息 ID
- `parseEventBody(body, messageId)` - 解析事件类型和详情

---

## 8. 关键配置项

### 环境变量
- `SVIX_API_KEY` - Svix API 密钥，未配置时 webhook 功能禁用
- `NOVU_ENTERPRISE` - 是否为企业版，决定是否启用完整 webhook 功能
- `STORE_NOTIFICATION_CONTENT` - 是否存储通知内容
- `NOVU_STRICT_AUTHENTICATION_ENABLED` - Bridge 签名验证开关（默认生产环境 true）
- `NODE_ENV` - 影响 `strictAuthentication` 默认值

### 数据库字段
- `environment.webhookAppId` - 环境对应的 Svix 应用 ID
- `environment.bridge.url` / `environment.echo.url` - Bridge 端点 URL
- `environment.apiKeys[].key` - 环境 API 密钥（加密存储）

---

## 9. 接收方集成指南

### 9.1 验证 Svix 签名（接收 Novu 出站事件）

使用 Svix 官方 SDK 验证签名：

```javascript
import { Webhook } from 'svix';

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post('/webhook', (req, res) => {
  const payload = JSON.stringify(req.body);
  const headers = req.headers;

  try {
    const verified = webhook.verify(payload, headers);

    // 处理不同事件类型
    const { type, data } = verified;
    switch (type) {
      case 'message.sent':
        // ⚠️ 注意：Chat 失败时也是 message.sent，但有 error 字段
        if (data.error) {
          console.log('消息发送失败:', data.error.message);
        } else {
          console.log('消息发送成功:', data.object._id);
        }
        break;
      case 'message.failed':
        // Email/SMS/Push 失败（注意 Chat 不会触发这个）
        console.log('消息发送失败:', data.error.message);
        break;
      case 'workflow.deleted':
        console.log('工作流已删除:', data.object._id);
        break;
      case 'workflow.published':
        console.log('工作流已发布:', data.object.name);
        break;
      case 'message.archived':
        console.log('消息已归档:', data.object._id);
        break;
      case 'email.received':
        console.log('收到入站邮件:', data.object.mail.subject);
        break;
      // ... 其他事件类型
    }

    res.status(200).send('OK');
  } catch (err) {
    res.status(400).send('Invalid signature');
  }
});
```

### 9.2 去重处理（接收方实现）

```javascript
const processedEventIds = new Set();

app.post('/webhook', (req, res) => {
  const eventId = req.body.id;

  if (processedEventIds.has(eventId)) {
    return res.status(200).send('Already processed');
  }

  processedEventIds.add(eventId);
  // 处理事件...
});
```

### 9.3 Bridge 端签名验证（使用 Novu Framework）

```typescript
import { Client, NovuHandler, NovuRequestHandler } from '@novu/framework/nest';

// 生产环境：强制启用严格认证
const client = new Client({
  secretKey: process.env.NOVU_SECRET_KEY,
  strictAuthentication: process.env.NODE_ENV === 'production',
});

const handler = new NovuRequestHandler({
  frameworkName: 'express',
  workflows: [myWorkflow],
  client,
  handler: ({ step, payload }) => {
    // 工作流逻辑
  },
});

// 所有请求会自动验证 novu-signature 头部
app.post('/bridge', handler.createHandler());
```

### 9.4 回溯历史

1. 通过 Novu API 获取 Svix Portal 访问令牌
   ```bash
   GET /v2/outbound-webhooks/portal/token
   ```
2. 登录 Svix Portal 查看完整投递历史
3. 使用 `eventId` 搜索特定事件
4. 查看每次投递的请求/响应详情和重试记录

---

## 10. 已知问题与注意事项

### 10.1 ⚠️ Chat 发送失败事件类型不一致

**问题**：`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:854`

Chat 渠道发送失败时，使用的事件类型是 `MESSAGE_SENT` 而非 `MESSAGE_FAILED`，与其他渠道不一致。

**接收方处理建议**：
```typescript
if (event.type === 'message.sent') {
  if (event.data.error) {
    // Chat 发送失败
    handleChatFailure(event.data);
  } else {
    // 发送成功
    handleSuccess(event.data);
  }
} else if (event.type === 'message.failed') {
  // Email/SMS/Push 发送失败
  handleOtherFailure(event.data);
}
```

### 10.2 两条签名链路使用不同密钥

- **Svix 出站**：使用 Svix Signing Secret（在 Svix Portal 中获取）
- **Bridge 入站**：使用 Novu 环境 Secret Key（在 Novu 管理后台获取）

不要混淆这两个密钥。

### 10.3 社区版不支持出站 Webhook

社区版使用 `NoopSendWebhookMessage` 空实现，所有 `sendWebhookMessage.execute()` 调用直接返回 `undefined`，不会发送任何 webhook。

---

## 11. 核心文件索引

| 模块 | 文件路径 | 功能 |
|-----|---------|-----|
| 事件定义 | `packages/shared/src/webhooks/webhook-event.enum.ts` | 事件类型枚举 |
| 发送用例 | `libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts` | webhook 发送核心逻辑 |
| Payload 结构 | `libs/application-generic/src/webhooks/dtos/webhook-payload.dto.ts` | 标准 payload 定义 |
| 消息 Mapper | `libs/application-generic/src/webhooks/mappers/message.mapper.ts` | 消息字段映射 |
| Svix 集成 | `libs/application-generic/src/webhooks/services/svix-provider.service.ts` | Svix 客户端初始化 |
| App ID 生成 | `libs/application-generic/src/webhooks/utils/app-id.ts` | webhookAppId 生成规则 |
| Portal API | `apps/api/src/app/outbound-webhooks/outbound-webhooks.controller.ts` | Portal 访问 API |
| 重试策略 | `apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts` | webhook filter 重试 |
| 入站处理 | `apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts` | 第三方回调处理 |
| Bridge 签名生成 | `libs/application-generic/src/utils/hmac.ts` | novu-signature 签名生成 |
| Bridge 请求执行 | `libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts` | Bridge 请求执行与签名注入 |
| Bridge 签名验证 | `packages/framework/src/handler.ts` | HMAC 签名验证逻辑 |
| 加密工具 | `packages/framework/src/utils/crypto.utils.ts` | HMAC 生成和恒时比较 |
| 签名错误 | `packages/framework/src/errors/signature.errors.ts` | 签名相关错误类型 |
| Client 配置 | `packages/framework/src/client.ts` | strictAuthentication 开关逻辑 |
| 配置类型 | `packages/framework/src/types/config.types.ts` | ClientOptions 类型定义 |
| Chat 发送 | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts` | Chat 渠道发送（含失败处理） |
| 工作流删除 | `apps/api/src/app/workflows-v1/usecases/delete-workflow/delete-workflow.usecase.ts` | 工作流删除触发 |
| 工作流发布 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | 工作流发布触发 |
| 消息批量操作 | `apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts` | 归档/已读/稍后 触发 |
| 消息删除 | `apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts` | 消息删除触发 |
| 入站邮件 | `libs/application-generic/src/usecases/inbound-domain-route-delivery/inbound-domain-route-delivery.usecase.ts` | 入站邮件触发 |
