# Webhook 回调机制深度分析

本文档详细分析 Novu 平台向用户回调 webhook 的完整机制，包括事件流转、签名生成、失败重试与去重机制，以及接收方如何验证身份与回溯历史投递。

## 1. 架构概述

Novu 的出站 Webhook 系统采用 **Svix** 作为第三方 webhook 服务提供商，负责处理签名、重试、去重等底层复杂度。平台本身专注于业务事件的触发和 payload 构造。

```
业务事件触发 → SendWebhookMessage → Svix API → 用户回调端点
     ↓                ↓
  生成 eventId    构造 WrapperDto
                   注入签名
                   管理重试
```

**核心代码位置**：
- `libs/application-generic/src/webhooks/` - 核心 webhook 逻辑
- `apps/api/src/app/outbound-webhooks/` - Webhook Portal API
- `packages/shared/src/webhooks/` - 事件类型定义

---

## 2. 事件流转机制

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

### 2.2 事件触发点

#### 2.2.1 工作流变更
**文件**：`libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:113-134`

- **创建工作流**：触发 `WORKFLOW_CREATED`，payload 包含 `object`（当前状态）
- **更新工作流**：触发 `WORKFLOW_UPDATED`，payload 包含 `object`（当前状态）和 `previousObject`（变更前状态）

#### 2.2.2 消息发送成功/失败
各渠道发送用例中触发：
- **Email**：`apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts:487-497, 553-564`
- **SMS**：`apps/worker/src/app/workflow/usecases/send-message/send-message-sms.usecase.ts:356-382`
- **Push**：`apps/worker/src/app/workflow/usecases/send-message/send-message-push.usecase.ts:644-655, 726-741`
- **In-App**：`apps/worker/src/app/workflow/usecases/send-message/send-message-in-app.usecase.ts:307-318`
- **Chat**：`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:805-855`

**成功时 payload 示例**：
```typescript
{
  eventType: WebhookEventEnum.MESSAGE_SENT,
  objectType: WebhookObjectTypeEnum.MESSAGE,
  payload: {
    object: messageWebhookMapper(message, subscriberId, {
      providerResponseId: result.id,
    }),
  },
  organizationId,
  environmentId,
}
```

**失败时 payload 示例（Push）**：
```typescript
{
  eventType: WebhookEventEnum.MESSAGE_FAILED,
  objectType: WebhookObjectTypeEnum.MESSAGE,
  payload: {
    object: messageWebhookMapper(message, subscriberId),
    error: {
      push: {
        reason: isTokenInvalid ? 'token_invalid' : 'generic_error',
        deviceToken: deviceToken,
      },
      message: e.message || 'Error while sending push',
    },
  },
  organizationId,
  environmentId,
}
```

#### 2.2.3 消息状态变更（已读/已看等）
**文件**：`apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts:77-105`

- 用户标记消息为已看 → `MESSAGE_SEEN`
- 用户标记消息为已读/未读 → `MESSAGE_READ` / `MESSAGE_UNREAD`

#### 2.2.4 偏好更新
**文件**：`apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts:78-88`

- 用户更新通知偏好 → `PREFERENCE_UPDATED`

### 2.3 事件投递流程

**核心文件**：`libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts:20-80`

```typescript
async execute(command: SendWebhookMessageCommand): Promise<{ eventId: string } | undefined> {
  // 1. 检查 Svix 客户端是否可用
  if (!this.svix) return;

  // 2. 获取环境配置，检查 webhookAppId
  const environment = await this.environmentRepository.findOne({ _id: command.environmentId });
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

### 2.4 Payload 结构

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

### 2.5 消息 Mapper

**文件**：`libs/application-generic/src/webhooks/mappers/message.mapper.ts:5-79`

`messageWebhookMapper` 函数负责将内部 `MessageEntity` 映射为对外暴露的 `MessageWebhookResponseDto`，包含字段：
- 基础字段：`_id`, `_templateId`, `_environmentId`, `_organizationId`, `_notificationId`
- 状态字段：`status`, `seen`, `read`, `archived`, `snoozedUntil`
- 时间字段：`createdAt`, `updatedAt`, `deliveredAt`, `lastSeenDate`, `lastReadDate`
- 其他：`subscriberId`, `transactionId`, `channel`, `providerId`, `errorId`, `errorText`, `contextKeys`

---

## 3. 签名生成与验证机制

### 3.1 出站 Webhook 签名（Svix 处理）

Novu 使用 **Svix** 作为 webhook 服务，签名由 Svix 自动生成和附加。

**签名头部**：`svix-signature`
**签名算法**：HMAC-SHA256
**签名格式**：`t=<timestamp>,v1=<signature>`

Svix 签名机制特性：
- 包含时间戳防止重放攻击
- 多版本签名支持（v1 为当前版本）
- 用户使用 Svix 提供的签名密钥进行验证

**Svix 客户端初始化**：`libs/application-generic/src/webhooks/services/svix-provider.service.ts:1-18`

```typescript
export const SvixProviderService: Provider<SvixClient> = {
  provide: 'SVIX_CLIENT',
  useFactory: (): SvixClient => {
    const apiKey = process.env.SVIX_API_KEY;
    if (!apiKey) return null;
    return new Svix(apiKey);
  },
};
```

### 3.2 入站请求签名验证（Framework Handler）

**文件**：`packages/framework/src/handler.ts:371-396`

用于验证来自 Novu 平台的入站请求签名（如 Bridge 回调）。

**签名头部**：`novu-signature`
**签名格式**：`t=<unix-ms>,v1=<hex-hmac>`

**验证流程**：
```typescript
private async validateHmac(payload: unknown, hmacHeader: string | null): Promise<void> {
  // 1. 检查是否启用 HMAC
  if (!this.hmacEnabled) return;
  
  // 2. 检查签名头部是否存在
  if (!hmacHeader) throw new SignatureNotFoundError();
  
  // 3. 检查签名密钥是否配置
  if (!this.client.secretKey) throw new SigningKeyNotFoundError();

  // 4. 解析签名头部
  const parsed = parseSignatureHeader(hmacHeader);
  if (!parsed.v1 || parsed.t === undefined) throw new SignatureInvalidError();

  // 5. 检查时间戳（防重放攻击）
  const now = Date.now();
  if (parsed.t < now - SIGNATURE_TIMESTAMP_TOLERANCE || parsed.t > now + SIGNATURE_TIMESTAMP_TOLERANCE) {
    throw new SignatureExpiredError();
  }

  // 6. 计算本地签名
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

### 3.3 核心加密工具

**文件**：`packages/framework/src/utils/crypto.utils.ts:11-60`

#### HMAC 生成（跨平台兼容）
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

#### 恒定时间比较（防时序攻击）
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

#### 签名头部解析
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

### 3.4 签名错误类型

**文件**：`packages/framework/src/errors/signature.errors.ts:4-57`

| 错误类型 | 场景 |
|---------|------|
| `SignatureNotFoundError` | 请求缺少 `novu-signature` 头部 |
| `SignatureInvalidError` | 签名格式不正确 |
| `SignatureExpiredError` | 签名时间戳超出 5 分钟容忍范围 |
| `SignatureMismatchError` | 签名验证失败 |
| `SigningKeyNotFoundError` | 服务端未配置签名密钥 |
| `SignatureVersionInvalidError` | 签名版本不支持 |

---

## 4. 失败重试与去重机制

### 4.1 失败重试（Svix 自动处理）

Svix 作为专业的 webhook 服务提供完善的重试机制：

**重试策略**：指数退避（Exponential Backoff）
**重试次数**：Svix 默认最多重试 25 次，时间跨度约 3 天
**成功判定**：接收方返回 2xx 状态码视为成功

### 4.2 内部 Webhook Filter 重试策略

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

### 4.3 去重机制

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

## 5. 接收方身份验证与历史投递回溯

### 5.1 Webhook Portal 访问管理

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

### 5.2 App ID 生成规则

**文件**：`libs/application-generic/src/webhooks/utils/app-id.ts:7-9`

```typescript
export function generateWebhookAppId(organizationId: OrganizationId, environmentId: EnvironmentId): string {
  return `o-${organizationId}-e-${environmentId}`;
}
```

### 5.3 企业版 vs 社区版差异

**文件**：`apps/api/src/app/outbound-webhooks/outbound-webhooks.module.ts:14-50`

- **企业版**：完整的 Svix 集成，包含所有 webhook 功能
- **社区版**：使用 `NoopSendWebhookMessage` 空实现，webhook 功能被禁用

### 5.4 历史投递回溯

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

## 6. 入站 Webhook 处理（平台接收第三方回调）

### 6.1 入口端点

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

### 6.2 处理流程

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

### 6.3 Provider 接口要求

**文件**：`packages/stateless/src/lib/provider/provider.interface.ts`

支持 webhook 的 provider 必须实现：
- `getMessageId(body)` - 从回调中提取消息 ID
- `parseEventBody(body, messageId)` - 解析事件类型和详情

---

## 7. 关键配置项

### 环境变量
- `SVIX_API_KEY` - Svix API 密钥，未配置时 webhook 功能禁用
- `NOVU_ENTERPRISE` - 是否为企业版，决定是否启用完整 webhook 功能
- `STORE_NOTIFICATION_CONTENT` - 是否存储通知内容

### 数据库字段
- `environment.webhookAppId` - 环境对应的 Svix 应用 ID

---

## 8. 接收方集成指南

### 8.1 验证 Svix 签名（接收方实现）

使用 Svix 官方 SDK 验证签名：

```javascript
import { Webhook } from 'svix';

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post('/webhook', (req, res) => {
  const payload = JSON.stringify(req.body);
  const headers = req.headers;

  try {
    const verified = webhook.verify(payload, headers);
    // 处理验证通过的请求
    res.status(200).send('OK');
  } catch (err) {
    res.status(400).send('Invalid signature');
  }
});
```

### 8.2 去重处理（接收方实现）

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

### 8.3 回溯历史

1. 通过 Novu API 获取 Svix Portal 访问令牌
2. 登录 Svix Portal 查看完整投递历史
3. 使用 `eventId` 搜索特定事件
4. 查看每次投递的请求/响应详情和重试记录

---

## 9. 核心文件索引

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
| 签名验证 | `packages/framework/src/handler.ts` | HMAC 签名验证逻辑 |
| 加密工具 | `packages/framework/src/utils/crypto.utils.ts` | HMAC 生成和恒时比较 |
| 签名错误 | `packages/framework/src/errors/signature.errors.ts` | 签名相关错误类型 |
