# WhatsApp 通道访问令牌全流程分析报告

## 1. 概述

本文档详细分析了 Novu 项目中 WhatsApp 通道的访问令牌从录入、加密保存，到发送消息时取出校验的完整流程，涵盖凭据管理、provider 选择以及失败处理机制。

## 2. 凭据结构定义

### 2.1 WhatsApp Business 凭据字段

WhatsApp Business 集成需要以下凭据字段（定义于 `packages/shared/src/consts/providers/credentials/provider-credentials.ts:1278-1316`）：

| 字段 | 显示名称 | 说明 | 必填 |
|------|----------|------|------|
| `apiToken` | Access API token | WhatsApp Business 访问令牌 | 是 |
| `phoneNumberIdentification` | Phone Number Identification | 电话号码 ID | 是 |
| `businessAccountId` | WhatsApp Business Account ID | WABA 账户 ID | 是 |
| `secretKey` | App Secret | Meta 应用密钥，用于验证 webhook 签名 | 是 |
| `token` | Verify Token | 自动生成的验证令牌，用于 webhook 握手 | 否（自动生成） |

### 2.2 需加密的敏感字段

系统定义了需要加密存储的敏感字段列表（`packages/shared/src/consts/providers/credentials/secure-credentials.ts:1-11`）：

```typescript
export const secureCredentials: CredentialsKeyEnum[] = [
  CredentialsKeyEnum.ApiKey,
  CredentialsKeyEnum.ApiToken,      // WhatsApp apiToken 属于此类
  CredentialsKeyEnum.SecretKey,     // WhatsApp secretKey 属于此类
  CredentialsKeyEnum.Token,         // WhatsApp token 属于此类
  CredentialsKeyEnum.Password,
  CredentialsKeyEnum.ServiceAccount,
  CredentialsKeyEnum.SigningSecret,
];
```

## 3. 凭据录入与加密保存流程

### 3.1 录入流程

#### 3.1.1 自动生成 Verify Token

WhatsApp 的 Verify Token 由系统自动生成，无需用户手动输入。逻辑位于 `apps/api/src/app/integrations/usecases/whatsapp/whatsapp-credentials.utils.ts:11-35`：

```typescript
export function ensureWhatsAppManagedCredentials({
  providerId,
  nextCredentials,
  existingCredentials,
}: {
  providerId: string;
  nextCredentials: ICredentials;
  existingCredentials?: ICredentials;
}): ICredentials {
  if (providerId !== ChatProviderIdEnum.WhatsAppBusiness) {
    return nextCredentials;
  }

  const incomingToken = typeof nextCredentials.token === 'string' ? nextCredentials.token.trim() : '';
  if (incomingToken) {
    return nextCredentials;
  }

  const existingToken = typeof existingCredentials?.token === 'string' ? existingCredentials.token.trim() : '';

  return {
    ...nextCredentials,
    token: existingToken || randomUUID(),  // 首次保存时自动生成 UUID
  };
}
```

#### 3.1.2 集成创建流程

在 `CreateIntegration` usecase 中（`apps/api/src/app/integrations/usecases/create-integration/create-integration.usecase.ts:138-222`）：

1. 调用 `ensureWhatsAppManagedCredentials` 处理自动生成的字段
2. 调用 `encryptCredentials` 加密凭据
3. 保存到数据库

关键代码：
```typescript
const managedCredentials = ensureWhatsAppManagedCredentials({
  providerId: command.providerId,
  nextCredentials: command.credentials ?? {},
});

const query: IntegrationQuery = {
  // ... 其他字段
  credentials: encryptCredentials(managedCredentials),  // 加密后存储
  // ...
};
```

#### 3.1.3 更新集成分支：Verify Token 继承与重新加密

在 `UpdateIntegration` usecase 中（`apps/api/src/app/integrations/usecases/update-integration/update-integration.usecase.ts:183-193`），更新集成时的凭据处理流程：

1. **解密现有凭据**：首先从数据库读取并解密现有凭据
2. **Verify Token 继承**：调用 `ensureWhatsAppManagedCredentials`，如果用户未提供新的 token，则继承现有 token
3. **重新加密**：对最终凭据重新加密后保存

关键代码：
```typescript
if (command.credentials) {
  const existingCredentials = existingIntegration.credentials
    ? decryptCredentials(existingIntegration.credentials)  // 解密现有凭据
    : undefined;
  const managedCredentials = ensureWhatsAppManagedCredentials({
    providerId: existingIntegration.providerId,
    nextCredentials: command.credentials,
    existingCredentials,                                 // 传入现有凭据用于继承
  });
  updatePayload.credentials = encryptCredentials(managedCredentials);  // 重新加密
}
```

**继承规则**：
- 如果用户在更新时明确提供了 `token` 字段，使用用户提供的值
- 如果用户未提供 `token`，但数据库中已有 token，继承现有 token
- 如果用户未提供 `token` 且数据库中也没有（罕见场景），生成新的 UUID
- **注意**：无论是否修改凭据，只要 `command.credentials` 存在，所有敏感字段都会被重新加密（IV 会变化，加密结果不同）

### 3.2 加密机制

#### 3.2.1 加密算法

使用 AES-256-CBC 加密算法（`libs/application-generic/src/encryption/cipher.ts:1-28`）：

```typescript
const IV_LENGTH = 16;
const CIPHER_ALGO = 'aes-256-cbc';

export function encrypt(text) {
  const ENCRYPTION_KEY = process.env.STORE_ENCRYPTION_KEY;
  const iv = randomBytes(IV_LENGTH);
  const cipher = createCipheriv(CIPHER_ALGO, Buffer.from(ENCRYPTION_KEY), iv);
  let encrypted = cipher.update(text);
  encrypted = Buffer.concat([encrypted, cipher.final()]);
  return `${iv.toString('hex')}:${encrypted.toString('hex')}`;
}
```

#### 3.2.2 加密标记

加密后的字符串会添加 `NOVU_ENCRYPTION_SUB_MASK` 前缀，用于识别是否已加密（`libs/application-generic/src/encryption/encrypt-provider.ts:1-68`）：

```typescript
export function encryptSecret(text: string): EncryptedSecret {
  const encrypted = encrypt(text);
  return `${NOVU_ENCRYPTION_SUB_MASK}${encrypted}`;
}

export function encryptCredentials(credentials: ICredentialsDto): ICredentialsDto {
  const encryptedCredentials: ICredentialsDto = {};
  for (const key in credentials) {
    encryptedCredentials[key] = isCredentialEncryptionRequired(key)
      ? encryptSecret(credentials[key])  // 敏感字段加密
      : credentials[key];                // 非敏感字段明文
  }
  return encryptedCredentials;
}
```

### 3.3 令牌有效性验证

在创建/更新集成时（如果启用了 `check` 参数），会调用 `WhatsAppValidateToken` usecase 验证令牌有效性（`apps/api/src/app/integrations/usecases/whatsapp/whatsapp-validate-token.usecase.ts:41-269`）。

验证流程：
1. 调用 Meta Graph API 的 `debug_token` 端点验证令牌
2. 检查必需的权限范围（`whatsapp_business_messaging`）
3. 验证电话号码 ID 是否属于该令牌
4. 验证 WABA ID 是否可访问
5. 交叉验证电话号码 ID 是否属于指定的 WABA

可能的错误代码：
- `invalid_token` - 令牌无效
- `expired_token` - 令牌过期
- `phone_not_found` - 电话号码不存在
- `phone_mismatch` - 令牌无法访问该电话号码
- `waba_not_accessible` - WABA 不可访问
- `waba_phone_mismatch` - 电话号码不属于该 WABA
- `missing_messaging_scope` - 缺少消息发送权限

## 4. 消息发送时的令牌取出与使用流程

### 4.1 消息发送主流程

消息发送由 `SendMessage` usecase 统一处理（`apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts:88-211`），根据步骤类型分发到具体的通道处理器。

对于 Chat 类型消息，调用 `SendMessageChat` usecase（`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:92-139`）。

### 4.2 集成选择 (SelectIntegration)

在发送消息前，通过 `SelectIntegration` usecase 选择合适的集成（`libs/application-generic/src/usecases/select-integration/select-integration.usecase.ts:19-80`）。

选择逻辑：
1. 首先获取主集成（`primary: true`）
2. 如果有租户信息，遍历所有集成交互式检查条件
3. 返回第一个满足条件的集成
4. 自动解密凭据（调用 `GetDecryptedIntegrations.getDecryptedCredentials`）

### 4.3 凭据解密

`GetDecryptedIntegrations` usecase 负责获取并解密集成凭据（`libs/application-generic/src/usecases/get-decrypted-integrations/get-decrypted-integrations.usecase.ts:55-59`）：

```typescript
public static getDecryptedCredentials(integration: IntegrationEntity) {
  integration.credentials = decryptCredentials(integration.credentials);
  return integration;
}
```

解密逻辑（`libs/application-generic/src/encryption/encrypt-provider.ts:33-44`）：
```typescript
export function decryptCredentials(credentials: ICredentialsDto): ICredentialsDto {
  const decryptedCredentials: ICredentialsDto = {};
  for (const key in credentials) {
    decryptedCredentials[key] =
      typeof credentials[key] === 'string' && isNovuEncrypted(credentials[key])
        ? decryptSecret(credentials[key])
        : credentials[key];
  }
  return decryptedCredentials;
}
```

### 4.4 发送链路二次校验说明

**重要结论**：发送链路中**不存在**二次 token 校验。

- Worker 进程中没有调用 `WhatsAppValidateToken` 或 Meta `debug_token` 端点
- 令牌仅在集成创建/更新时（API 层）进行有效性验证
- 发送时直接使用解密后的 token 调用 Meta Graph API
- 如果 token 已失效或权限不足，Meta API 会直接返回错误，由失败处理机制处理

### 4.5 Provider 初始化

#### 4.5.1 ChatFactory 选择 Handler

通过 `ChatFactory` 根据 providerId 选择对应的 Handler（`libs/application-generic/src/factories/chat/chat.factory.ts:16-42`）：

```typescript
export class ChatFactory implements IChatFactory {
  handlers: IChatHandler[] = [
    // ... 其他 handler
    new WhatsAppBusinessHandler(),
  ];

  getHandler(integration) {
    const handler = this.handlers.find(
      (handlerItem) => handlerItem.canHandle(integration.providerId, integration.channel)
    ) ?? null;
    if (!handler) return null;
    handler.buildProvider(integration.credentials);  // 使用解密后的凭据构建 provider
    return handler;
  }
}
```

#### 4.5.2 WhatsAppBusinessHandler

WhatsApp Business 的 Handler 实现（`libs/application-generic/src/factories/chat/handlers/whatsapp-business.handler.ts:1-16`）：

```typescript
export class WhatsAppBusinessHandler extends BaseChatHandler {
  constructor() {
    super(ChatProviderIdEnum.WhatsAppBusiness, ChannelTypeEnum.CHAT);
  }

  buildProvider(credentials: ICredentials) {
    this.provider = new WhatsappBusinessChatProvider({
      accessToken: credentials.apiToken,              // 使用解密后的 apiToken
      phoneNumberIdentification: credentials.phoneNumberIdentification,
    });
  }
}
```

#### 4.5.3 WhatsappBusinessChatProvider

实际发送消息的 Provider（`packages/providers/src/lib/chat/whatsapp-business/whatsapp-business.provider.ts:15-116`）：

```typescript
export class WhatsappBusinessChatProvider extends BaseProvider implements IChatProvider {
  constructor(
    private config: {
      accessToken: string;
      phoneNumberIdentification: string;
    }
  ) {
    super();
    this.axiosClient = Axios.create({
      headers: {
        Authorization: `Bearer ${this.config.accessToken}`,  // Bearer 令牌认证
        'Content-Type': 'application/json',
      },
    });
  }

  async sendMessage(options: IChatOptions): Promise<ISendMessageSuccessResponse> {
    const payload = this.transform(/* ... */);
    const { data } = await this.axiosClient.post<ISendMessageRes>(
      `${this.baseUrl + this.config.phoneNumberIdentification}/messages`,
      payload
    );
    return {
      id: data.messages[0].id,
      date: new Date().toISOString(),
    };
  }
}
```

## 5. Provider 选择机制

### 5.1 通道解析流程

在 `SendMessageChat` 中，通过以下步骤解析通道（`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:188-211`）：

1. **新集成通道**：调用 `resolveChannelEndpoints` 获取按集成分组的通道端点
2. **遗留通道**：从 subscriber 的 `channels` 字段中获取，并自动为有电话号码的 subscriber 添加 WhatsApp Business 通道

```typescript
private getLegacyChatChannels(command: SendMessageChannelCommand): IChannelSettings[] {
  const { subscriber } = command.compileContext;
  const chatChannels = subscriber.channels?.filter(/* ... */) || [];
  
  // 如果 subscriber 有电话号码，自动添加 WhatsApp Business 通道
  if (subscriber.phone) {
    chatChannels.push({
      providerId: ChatProviderIdEnum.WhatsAppBusiness,
      credentials: {
        phoneNumber: subscriber.phone,
      },
    });
  }
  
  return chatChannels;
}
```

### 5.2 多通道尝试机制

系统会尝试向所有可用的通道发送消息，只要有一个成功即视为成功（`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:216-257`）：

```typescript
private async sendToAllChannels(channels: UnifiedChannel[], messageContext: MessageContext): Promise<SendMessageStatus> {
  let status: SendMessageStatus = SendMessageStatus.FAILED;
  
  for (const channel of channels) {
    try {
      const result = await this.sendChannelMessage(/* ... */);
      status = this.updateStatus(status, result.status);
    } catch (e) {
      // 单个通道失败不影响其他通道
      Logger.error(e, `Sending chat message to ${channel.type} channel failed`);
    }
  }
  
  return status;
}
```

## 6. 失败处理与回退机制

### 6.1 真正生效的失败处理机制

#### 6.1.1 多通道回退机制（已接入）

这是 WhatsApp 消息发送最主要的失败处理机制：

- **位置**：`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:216-257`
- **机制**：遍历所有可用通道，依次尝试发送
- **行为**：单个通道失败不抛出异常，继续尝试下一个通道
- **成功条件**：只要有一个通道成功，整体状态即为成功
- **适用场景**：配置了多个 Chat 集成（如同时配置 Slack 和 WhatsApp）

#### 6.1.2 队列级重试（部分接入）

- **位置**：`apps/worker/src/app/workflow/services/standard.worker.ts:230-285`
- **触发条件**：仅当错误消息包含 `EXCEPTION_MESSAGE_ON_WEBHOOK_FILTER` 时才会重试
- **重试逻辑**：
  - 最大重试次数：`DEFAULT_ATTEMPTS`（默认 3 次）
  - 退避策略：随机指数退避 `Math.round(Math.random() * 2 ** attemptsMade * 1000)`
  - 定义于 `apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts:35`
- **限制**：WhatsApp API 返回的常规错误（如 token 无效、权限不足）不会触发此重试

#### 6.1.3 错误记录与通知（已接入）

失败时会：
1. 更新消息状态为 `error`
2. 创建执行详情记录（`ExecutionDetailsStatusEnum.FAILED`）
3. 发送 webhook 通知（`WebhookEventEnum.MESSAGE_SENT`），包含错误信息

### 6.2 未接入该链路的通用重试能力

#### 6.2.1 RetryOnError 装饰器（未使用）

系统提供了通用的重试装饰器 `RetryOnError`（`libs/application-generic/src/decorators/retry-on-error-decorator.ts:16-82`），支持：

- 配置最大重试次数（默认 3 次）
- 指数退避延迟（默认 100ms 起始）
- 自定义错误过滤逻辑
- 自定义日志记录

**实际使用情况**：
- 该装饰器**未在 WhatsApp 消息发送链路中使用**
- 仅在两个地方使用：
  - `create-or-update-subscriber.usecase.ts:20` - 处理 `MongoServerError`
  - `merge-or-create-digest.usecase.ts:146` - 处理 `MongoServerError`
- WhatsApp Provider 的 `sendMessage` 方法没有使用此装饰器

#### 6.2.2 Provider 级重试（未实现）

`WhatsappBusinessChatProvider` 的 `sendMessage` 方法没有内置重试逻辑，Meta API 调用失败直接抛出异常。

### 6.3 消息发送失败处理流程

在 `SendMessageChat.sendMessage` 中（`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:542-574`）：

```typescript
try {
  const result = await chatHandler.send({
    channelData: overriddenChannelData,
    bridgeProviderData: combinedOverrides,
    customData: overrides,
    content,
  });
  return await this.handleMessageSendSuccess(result, message, command, overriddenChannelData);
} catch (error) {
  return await this.handleMessageSendError(error, message, command, overriddenChannelData);
}
```

### 6.4 失败处理机制对比表

| 机制 | 位置 | 状态 | 适用场景 |
|------|------|------|----------|
| 多通道回退 | `send-message-chat.usecase.ts` | ✅ 已接入 | 多 Chat 集成配置 |
| 队列级重试 | `standard.worker.ts` | ⚠️ 部分接入 | 仅 webhook filter 错误 |
| 错误记录通知 | `send-message-chat.usecase.ts` | ✅ 已接入 | 所有错误 |
| RetryOnError 装饰器 | `retry-on-error-decorator.ts` | ❌ 未使用 | 数据库操作错误 |
| Provider 级重试 | `whatsapp-business.provider.ts` | ❌ 未实现 | - |

## 7. 关键流程总结

### 7.1 凭据录入流程（创建）

```
用户输入凭据 (apiToken, phoneNumberIdentification, businessAccountId, secretKey)
        ↓
ensureWhatsAppManagedCredentials() - 自动生成 token (Verify Token)
        ↓
encryptCredentials() - 加密敏感字段 (apiToken, secretKey, token)
        ↓
保存到集成表 (IntegrationRepository.create)
```

### 7.2 凭据录入流程（更新）

```
用户提交更新 (可能包含部分凭据字段)
        ↓
decryptCredentials() - 解密数据库中现有凭据
        ↓
ensureWhatsAppManagedCredentials() - 继承或生成 token
        ↓
encryptCredentials() - 重新加密所有敏感字段
        ↓
更新集成表 (IntegrationRepository.update)
```

### 7.3 消息发送流程

```
触发消息发送
        ↓
SelectIntegration.execute() - 选择合适的集成
        ↓
GetDecryptedIntegrations.getDecryptedCredentials() - 解密凭据
        ↓
ChatFactory.getHandler() - 选择 WhatsAppBusinessHandler
        ↓
WhatsAppBusinessHandler.buildProvider() - 使用解密后的凭据初始化 Provider
        ↓
WhatsappBusinessChatProvider.sendMessage() - 调用 Meta Graph API 发送消息
        ↓
成功 → 更新消息状态，发送成功 webhook
失败 → 记录错误，尝试其他通道，发送失败 webhook
```

### 7.4 数据流向

| 阶段 | 数据位置 | 状态 |
|------|----------|------|
| 录入时 | 内存 | 明文 |
| 保存前 | 内存 | 加密（敏感字段） |
| 数据库 | Integration.credentials | 加密（敏感字段） |
| 读取时 | 内存 | 加密 |
| 使用前 | 内存 | 解密 |
| API 调用 | HTTP 请求头 | 明文 (Bearer token) |

### 7.5 校验时机与范围

| 校验类型 | 时机 | 位置 | 说明 |
|----------|------|------|------|
| Token 有效性 | 集成创建/更新时 | API 层 | 调用 Meta debug_token |
| 权限范围检查 | 集成创建/更新时 | API 层 | 检查 whatsapp_business_messaging |
| 电话号码归属 | 集成创建/更新时 | API 层 | 验证 phoneNumberId |
| WABA 可访问性 | 集成创建/更新时 | API 层 | 验证 businessAccountId |
| 二次校验 | 发送消息时 | - | 不存在，直接调用 Meta API |

## 8. 安全特性

1. **字段级加密**：仅敏感字段加密，非敏感字段明文存储，平衡安全性和可查询性
2. **AES-256-CBC**：使用标准加密算法，随机 IV 保证相同明文加密后结果不同
3. **加密标记**：通过前缀识别加密状态，支持透明加解密
4. **环境变量密钥**：加密密钥存储在环境变量 `STORE_ENCRYPTION_KEY` 中，不硬编码
5. **令牌验证**：录入时即验证令牌有效性，避免无效配置
6. **自动生成**：Verify Token 自动生成，减少用户操作和安全风险
7. **更新时重新加密**：更新集成时重新加密所有敏感字段，保证加密随机性

## 9. 代码引用位置汇总

| 功能 | 文件位置 |
|------|----------|
| WhatsApp 凭据定义 | `packages/shared/src/consts/providers/credentials/provider-credentials.ts:1278-1316` |
| 敏感字段列表 | `packages/shared/src/consts/providers/credentials/secure-credentials.ts:1-11` |
| 自动生成/继承 Verify Token | `apps/api/src/app/integrations/usecases/whatsapp/whatsapp-credentials.utils.ts:11-35` |
| 令牌验证 | `apps/api/src/app/integrations/usecases/whatsapp/whatsapp-validate-token.usecase.ts:41-269` |
| 加密算法 | `libs/application-generic/src/encryption/cipher.ts:1-28` |
| 凭据加解密 | `libs/application-generic/src/encryption/encrypt-provider.ts:1-68` |
| 集成创建 | `apps/api/src/app/integrations/usecases/create-integration/create-integration.usecase.ts:138-222` |
| 集成更新 | `apps/api/src/app/integrations/usecases/update-integration/update-integration.usecase.ts:183-193` |
| 解密集成 | `libs/application-generic/src/usecases/get-decrypted-integrations/get-decrypted-integrations.usecase.ts:55-59` |
| 选择集成 | `libs/application-generic/src/usecases/select-integration/select-integration.usecase.ts:19-80` |
| Chat Factory | `libs/application-generic/src/factories/chat/chat.factory.ts:16-42` |
| WhatsApp Handler | `libs/application-generic/src/factories/chat/handlers/whatsapp-business.handler.ts:1-16` |
| WhatsApp Provider | `packages/providers/src/lib/chat/whatsapp-business/whatsapp-business.provider.ts:15-116` |
| 发送 Chat 消息 | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:92-139` |
| 多通道回退 | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:216-257` |
| 队列级重试 | `apps/worker/src/app/workflow/services/standard.worker.ts:230-285` |
| 退避策略 | `apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts:35` |
| 重试装饰器（未使用） | `libs/application-generic/src/decorators/retry-on-error-decorator.ts:16-82` |
