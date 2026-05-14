# Novu OAuth 集成流程完整分析

**生成日期**: 2026-05-14  
**分析范围**: Chat 渠道 OAuth 完整流程

---

## 目录

1. [整体架构概述](#1-整体架构概述)
2. [聊天 OAuth 链接生成流程](#2-聊天-oauth-链接生成流程)
3. [Slack OAuth 回调处理流程](#3-slack-oauth-回调处理流程)
4. [OAuth State 校验机制](#4-oauth-state-校验机制)
5. [Provider 凭证隔离机制](#5-provider-凭证隔离机制)
6. [主渠道设置条件](#6-主渠道设置条件)
7. [集成选择与默认回退链路](#7-集成选择与默认回退链路)
8. [失败分支完整分析](#8-失败分支完整分析)
9. [关键代码引用](#9-关键代码引用)

---

## 1. 整体架构概述

### 1.1 核心 UseCase 协作图

```
┌─────────────────────────────────────────────────────────────────┐
│                    OAuth 流程入口层                               │
├─────────────────────────────────────────────────────────────────┤
│  GenerateChatOauthUrl          ChatOauthCallback                │
│  (统一分发器)                    (统一分发器)                    │
└─────────────┬───────────────────────────┬──────────────────────┘
              │                           │
              ▼                           ▼
┌─────────────────────────────┐  ┌─────────────────────────────┐
│   Provider 具体实现层        │  │   Provider 具体回调层        │
├─────────────────────────────┤  ├─────────────────────────────┤
│  GenerateSlackOauthUrl      │  │  SlackOauthCallback         │
│  GenerateMsTeamsOauthUrl    │  │  MsTeamsOauthCallback       │
│  GenerateLinkUserOauthUrl   │  │                             │
│  GenerateConnectOauthUrl    │  │                             │
└─────────────┬───────────────┘  └─────────────┬───────────────┘
              │                                │
              ▼                                ▼
┌─────────────────────────────┐  ┌─────────────────────────────┐
│   辅助工具层                 │  │   集成管理层                 │
├─────────────────────────────┤  ├─────────────────────────────┤
│  chat-oauth-state.util.ts   │  │  SetIntegrationAsPrimary    │
│  CHAT_OAUTH_CALLBACK_PATH   │  │  SelectIntegration          │
└─────────────────────────────┘  └─────────────────────────────┘
```

### 1.2 支持的 Provider 和模式

| Provider | 支持模式 | 文件位置 |
|----------|---------|---------|
| **Slack** | `connect`, `link_user`, `incoming_webhook` | `generate-slack-oauth-url.usecase.ts` |
| **MsTeams** | `connect`, `link_user` | `generate-msteams-oauth-url.usecase.ts` |
| **Novu (演示)** | 同 Slack | 使用 Slack provider 逻辑 |

---

## 2. 聊天 OAuth 链接生成流程

### 2.1 主入口: GenerateChatOauthUrl

**代码位置**: `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-chat-oauth-url.usecase.ts`

```typescript
async execute(command: GenerateChatOauthUrlCommand): Promise<string> {
  const integration = await this.getIntegration(command);
  
  switch (integration.providerId) {
    case ChatProviderIdEnum.Slack:
    case ChatProviderIdEnum.Novu:
      return this.generateSlackOAuthUrl.execute(...);
      
    case ChatProviderIdEnum.MsTeams:
      return this.generateMsTeamsOAuthUrl.execute(...);
      
    default:
      throw new BadRequestException(`OAuth not supported for provider: ${integration.providerId}`);
  }
}
```

### 2.2 Slack OAuth URL 生成完整流程

```
┌─────────────────────────────────────────────────────────────┐
│              GenerateSlackOauthUrl.execute()                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. validateSubscriberIdOrContext()                          │
│    ├─ scope 包含 incoming_webhook? → 必须有 subscriberId    │
│    └─ validateConnectionMode() 检查连接模式合法性            │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. assertResourceExists()                                   │
│    └─ subscriberId 存在? → 在 DB 中查找 subscriber           │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. getIntegrationCredentials()                              │
│    ├─ Novu 演示集成 → GetNovuProviderCredentials            │
│    └─ 客户自有集成 → 从 integration.credentials 读取         │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. createSecureState()                                      │
│    ├─ 构建 StateData 对象 (包含环境/组织/集成信息)           │
│    ├─ JSON 序列化 payload                                   │
│    ├─ 用环境 API Key 生成 HMAC 签名                         │
│    └─ base64url 编码: `${payload}.${signature}`             │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. resolveBotScopes()                                       │
│    ├─ 指定了 scope? → 使用传入的 scope                       │
│    ├─ 集成关联 Agent? → 使用 SLACK_AGENT_OAUTH_SCOPES       │
│    └─ 否则 → 使用默认 scope (undefined)                      │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. getOAuthUrl()                                            │
│    ├─ link_user 模式: 设置 user_scope                        │
│    └─ 其他模式: 设置 scope                                   │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 StateData 结构

```typescript
// 位置: generate-slack-oauth-url.usecase.ts:23-35
export type StateData = {
  identifier?: string;              // 连接标识符
  subscriberId?: string;            // 订阅者 ID
  context?: ContextPayload;         // 上下文 (租户等)
  environmentId: string;            // ✓ 环境 ID (必填)
  organizationId: string;           // ✓ 组织 ID (必填)
  integrationIdentifier: string;    // ✓ 集成标识符 (必填)
  providerId: ChatProviderIdEnum;   // ✓ 提供商 ID (必填)
  timestamp: number;                // ✓ 创建时间戳 (必填)
  mode?: 'connect' | 'link_user';   // OAuth 模式
  connectionMode?: ConnectionMode;  // 连接模式
  autoLinkUser?: boolean;           // 是否自动链接用户
};
```

### 2.4 OAuth Scopes 配置

| 场景 | Scopes |
|------|--------|
| **默认** | `chat:write`, `chat:write.public`, `channels:read`, `groups:read`, `users:read`, `users:read.email` |
| **Agent 集成** | `SLACK_AGENT_OAUTH_SCOPES` (扩展权限) |
| **link_user 模式** | `identity.basic` (user_scope) |
| **incoming_webhook** | `incoming-webhook` |

---

## 3. Slack OAuth 回调处理流程

### 3.1 主入口: ChatOauthCallback

**代码位置**: `apps/api/src/app/integrations/usecases/chat-oauth-callback/chat-oauth-callback.usecase.ts`

```typescript
async execute(command: ChatOauthCallbackCommand): Promise<ChatOauthCallbackResult> {
  const providerId = this.extractProviderIdFromState(command.state);
  
  switch (providerId) {
    case ChatProviderIdEnum.Slack:
    case ChatProviderIdEnum.Novu:
      if (!command.providerCode) {
        throw new BadRequestException('Missing required parameter: code');
      }
      return await this.slackOauthCallback.execute(...);
      
    case ChatProviderIdEnum.MsTeams:
      return await this.msTeamsOauthCallback.execute(...);
      
    default:
      throw new BadRequestException(`OAuth callback not supported for provider: ${providerId}`);
  }
}
```

### 3.2 Slack 回调完整流程

```
┌─────────────────────────────────────────────────────────────┐
│             SlackOauthCallback.execute()                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. decodeSlackState()                                       │
│    ├─ peekOAuthStatePayload() → 无验证获取 environmentId    │
│    ├─ 查找环境并获取 API Key                                │
│    └─ validateAndDecodeState() → 完整签名+过期验证           │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. getIntegration()                                         │
│    └─ 按 {environmentId, organizationId, channel=chat,      │
│           providerId, integrationIdentifier} 查找集成        │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. getIntegrationCredentials()                              │
│    ├─ Novu 演示集成 → GetNovuProviderCredentials            │
│    └─ 客户自有集成 → 检查 credentials.clientId/secretKey     │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. exchangeCodeForAuthData()                                │
│    ├─ decryptCredentials() 解密凭证                          │
│    ├─ POST slack.com/api/oauth.v2.access                    │
│    │   └─ body: { redirect_uri, code, client_id, client_secret }
│    └─ 检查 res.data.ok === true                              │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. 分支处理 (根据 mode 和 authData)                         │
│    ├──────────────────────────────────────────────────────┐│
│    │ mode === 'link_user' → linkUserEndpoint()            ││
│    │   └─ 创建 SLACK_USER 类型的 channel-endpoint          ││
│    ├──────────────────────────────────────────────────────┤│
│    │ authData.incoming_webhook → createIncomingWebhookEndpoint()
│    │   └─ 仅使用 webhook_url, 创建 WEBHOOK 类型 endpoint   ││
│    └──────────────────────────────────────────────────────┤│
│    │ 工作区连接模式 → createChannelConnection + autoLinkUser
│    │   ├─ 创建 channel-connection (access_token + workspace)
│    │   └─ autoLinkUser=true 时: 自动创建 SLACK_USER endpoint
│    └──────────────────────────────────────────────────────┘│
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. 返回结果                                                  │
│    ├─ credentials.redirectUrl 存在? → 返回 URL 重定向        │
│    └─ 否则 → 返回 HTML: <script>window.close();</script>     │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 回调结果类型

```typescript
// ChatOauthCallbackResult
{
  type: ResponseTypeEnum.URL | ResponseTypeEnum.HTML;
  result: string;
}
```

---

## 4. OAuth State 校验机制

### 4.1 编码格式设计

**代码位置**: `apps/api/src/app/integrations/usecases/generate-chat-oath-url/chat-oauth-state.util.ts`

```
OAuth State 结构:
base64url( `${json_payload}.${hex_hmac_signature}` )
          └───────┬───────┘   └─────────┬─────────┘
                  │                       │
                  │                       └─ 永不包含 '.'
                  └─ 可能包含 '.' (如邮箱、versioned IDs)

⚠️  关键: 必须在最后一个 '.' 处分割, 不是第一个!
```

### 4.2 核心工具函数

| 函数 | 用途 | 是否验证签名 |
|------|------|-------------|
| `encodeOAuthState(payload, signature)` | 编码 state | - |
| `decodeOAuthStateString(state)` | 解码为原始字符串 | - |
| `splitOAuthState(state)` | 分割 payload 和 signature | - |
| `peekOAuthStatePayload<T>(state)` | **无验证**解析 payload | ❌ |
| `validateAndDecodeState()` | 完整验证 | ✅ |

### 4.3 完整校验流程 (validateAndDecodeState)

```
┌─────────────────────────────────────────────────────────────┐
│        GenerateSlackOauthUrl.validateAndDecodeState()       │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│ 1. splitOAuthState()      │   │ ❌ 失败分支                │
│    └─ 在最后一个 '.' 分割  │   │    "Invalid OAuth state:  │
│                           │   │     missing signature      │
│                           │   │     separator"             │
└─────────────┬─────────────┘   └───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 签名验证                                                 │
│    expected_signature = HMAC_SHA256(环境 API Key, payload)  │
│    if (signature !== expected_signature)                     │
└─────────────┬───────────────────────────────────────────────┘
              │
              ├─ ✅ 匹配 → 继续
              │
              ▼
┌───────────────────────────┐
│ ❌ 失败分支               │
│    "Invalid state         │
│    signature"             │
└───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. JSON 解析 payload                                        │
│    if (解析失败)                                            │
└─────────────┬───────────────────────────────────────────────┘
              │
              ├─ ✅ 成功 → 继续
              │
              ▼
┌───────────────────────────┐
│ ❌ 失败分支               │
│    "Invalid OAuth state   │
│    parameter"             │
└───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 过期检查 (5 分钟)                                        │
│    if (Date.now() - data.timestamp > 5 * 60 * 1000)        │
└─────────────┬───────────────────────────────────────────────┘
              │
              ├─ ✅ 未过期 → 返回 StateData
              │
              ▼
┌───────────────────────────┐
│ ❌ 失败分支               │
│    "OAuth state expired"  │
└───────────────────────────┘
```

### 4.4 "Peek 模式" 的设计意图

`peekOAuthStatePayload()` 是一个**故意设计**的不安全函数:

**场景**: 在回调入口 `ChatOauthCallback.extractProviderIdFromState()` 中
- 我们需要知道 `providerId` 来分发到正确的 provider handler
- 但我们还没有环境 API Key 来验证签名
- 因为 API Key 存储在环境中, 需要 `environmentId` 才能查找

**鸡生蛋问题**:
```
需要 providerId → 才能选择 handler
  ↓
handler 需要 environmentId → 才能获取 API Key
  ↓
API Key 才能验证签名
  ↓
验证后才能信任 environmentId
```

**解决方案**:
1. 先用 `peekOAuthStatePayload()` **无验证**地获取 `providerId` 和 `environmentId`
2. 用 `environmentId` 查找环境并获取 API Key
3. **然后** 调用 `validateAndDecodeState()` 进行完整验证
4. 验证通过后才信任所有字段

---

## 5. Provider 凭证隔离机制

### 5.1 两层凭证架构

```
┌─────────────────────────────────────────────────────────────┐
│                  Integration Credentials                     │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
          ▼                                       ▼
┌───────────────────────────┐       ┌───────────────────────────┐
│ Novu 演示集成             │       │ 客户自有集成               │
│ providerId === 'novu'    │       │ providerId === 'slack'    │
└─────────────┬─────────────┘       └─────────────┬─────────────┘
              │                                   │
              ▼                                   ▼
┌───────────────────────────┐       ┌───────────────────────────┐
│ GetNovuProviderCredentials│       │ integration.credentials   │
│ (中央服务获取)            │       │ (数据库存储, 加密)        │
└───────────────────────────┘       └─────────────┬─────────────┘
                                                  │
                                                  ▼
                                        ┌───────────────────┐
                                        │ decryptCredentials()
                                        │ (回调时解密)
                                        └───────────────────┘
```

### 5.2 凭证结构

```typescript
interface ICredentialsEntity {
  // OAuth 必填字段
  clientId: string;           // OAuth Client ID
  secretKey: string;          // OAuth Client Secret (加密存储)
  
  // 可选配置
  hmac?: boolean;             // 是否启用 subscriber HMAC 验证
  redirectUrl?: string;       // 回调成功后的重定向 URL
  
  // 其他 provider 特定字段...
}
```

### 5.3 HMAC 验证 (旧版兼容)

**代码位置**: `apps/api/src/app/subscribers/usecases/chat-oauth/chat-oauth.usecase.ts`

```typescript
function validateEncryption({ apiKey, subscriberId, externalHmacHash }) {
  const hmacHash = createHash(apiKey, subscriberId);
  if (hmacHash !== externalHmacHash) {
    throw new BadRequestException('Invalid HMAC hash');
  }
}
```

**触发条件**: `integration.credentials.hmac === true`

---

## 6. 主渠道设置条件

### 6.1 支持主渠道的渠道类型

**代码位置**: `packages/shared/src/types/channel.ts:68`

```typescript
export const CHANNELS_WITH_PRIMARY: readonly ChannelTypeEnum[] = [
  ChannelTypeEnum.EMAIL,    // ✅ 支持
  ChannelTypeEnum.SMS       // ✅ 支持
  // ⚠️ Chat 渠道目前不支持主渠道!
];
```

### 6.2 SetIntegrationAsPrimary 流程

**代码位置**: `apps/api/src/app/integrations/usecases/set-integration-as-primary/set-integration-as-primary.usecase.ts`

```
┌─────────────────────────────────────────────────────────────┐
│          SetIntegrationAsPrimary.execute()                   │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
┌───────────────────────────┐       ┌───────────────────────────┐
│ 1. 集成存在?              │       │ ❌ NotFoundException     │
│    findOne({ _id, orgId })│       │    "Integration not found"│
└─────────────┬─────────────┘       └───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 渠道支持 primary?                                         │
│    if (!CHANNELS_WITH_PRIMARY.includes(integration.channel)) │
└─────────────┬───────────────────────────────────────────────┘
              │
              ├─ ✅ 支持 → 继续
              │
              ▼
┌───────────────────────────┐
│ ❌ BadRequestException    │
│    "Channel X does not    │
│    support primary"       │
└───────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 已是 primary?                                            │
│    if (integration.primary === true)                        │
│      → 直接返回 (幂等)                                      │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. updatePrimaryFlag()                                       │
│    ├─ 步骤 A: 将该 channel 下所有 active 集成的 primary 设为 false
│    │      updateMany({ orgId, envId, channel, active: true, primary: true },
│    │                 { $set: { primary: false } })
│    │
│    └─ 步骤 B: 将目标集成设为 primary
│           updateOne({ _id, orgId, envId },
│                     { $set: { 
│                         active: true, 
│                         primary: true,
│                         conditions: []  // ⚠️ 清空条件!
│                       } })
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. recalculatePriorityForAllActive()                        │
│    重新计算所有 active 集成的优先级                           │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. 返回更新后的集成对象                                      │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 ⚠️ 重要: 设置 Primary 会清空 Conditions

**代码位置**: `set-integration-as-primary.usecase.ts:44`

```typescript
$set: {
  active: true,
  primary: true,
  conditions: [],  // ← 清空所有条件!
}
```

**设计意图**: 
- Primary 集成是"默认兜底"集成
- 不应该有条件限制 (否则匹配不到时无兜底)
- 条件集成应作为更高优先级的匹配项

### 6.4 优先级重计算算法

**代码位置**: `libs/dal/src/repositories/integration/integration.repository.ts:106-151`

```typescript
async recalculatePriorityForAllActive({ _id, _organizationId, _environmentId, channel }) {
  // 1. 查找其他 active 集成 (按 priority 降序)
  const otherActiveIntegrations = await this.find(
    { _organizationId, _environmentId, channel, active: true, _id: { $nin: [_id] } },
    '_id',
    { sort: { priority: -1 } }
  );
  
  // 2. 新的 id 顺序: [新 primary, ...其他集成]
  const ids = [_id, ...otherActiveIntegrations.map(i => i._id)];
  
  // 3. 按顺序设置 priority: n, n-1, ..., 1
  //    新 primary 获得最高优先级 (ids.length)
  await Promise.all(ids.map((id, index) =>
    this.update(
      { _id: id, _organizationId, _environmentId },
      { $set: { priority: ids.length - index } }
    )
  ));
}
```

---

## 7. 集成选择与默认回退链路

### 7.1 SelectIntegration 完整流程

**代码位置**: `libs/application-generic/src/usecases/select-integration/select-integration.usecase.ts`

```
┌─────────────────────────────────────────────────────────────┐
│              SelectIntegration.execute()                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. getPrimaryIntegration()  ← 第一优先级: 获取主渠道        │
│    ├─ 指定了 identifier? → 按 identifier 查找               │
│    ├─ 渠道支持 primary? → 查找 { primary: true, active: true }
│    └─ 否则 → 查找最新创建的 active 集成 (sort: { createdAt: -1 })
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 条件过滤 (仅当: 无 identifier + 有 tenant + 有 userId)   │
│    ├─ 查找该 channel 所有 active 集成                        │
│    ├─ 遍历有 conditions 的集成                               │
│    │   ├─ normalizeVariables() → 处理变量替换                │
│    │   └─ conditionsFilter.filter() → 检查条件是否匹配       │
│    └─ 第一个匹配的集成 → 替换为当前选择                      │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. getDecryptedCredentials()                                │
│    └─ 解密集成的 credentials 字段                            │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 返回结果: IntegrationEntity | undefined                   │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 回退链路优先级

```
                     集成选择优先级
              ┌─────────────────────────────┐
              │     1. 条件匹配集成         │ ◄── 最高优先级
              │        (conditions 通过)     │
              └─────────────┬───────────────┘
                            │
              ┌─────────────▼───────────────┐
              │     2. Primary 集成         │
              │        (primary = true)      │
              └─────────────┬───────────────┘
                            │
              ┌─────────────▼───────────────┐
              │     3. 最新活跃集成         │ ◄── 默认回退
              │     (createdAt 降序)        │
              └─────────────┬───────────────┘
                            │
              ┌─────────────▼───────────────┐
              │     4. undefined            │ ◄── 无可用集成
              └─────────────────────────────┘
```

### 7.3 条件过滤机制

**触发条件 (必须同时满足)**:
1. `!command.identifier` - 没有指定具体集成
2. `command.filterData.tenant` - 有租户信息
3. `command.userId` - 有用户 ID

**过滤流程**:
```
遍历所有 active 集成 (有 conditions):
  ├─ normalizeVariables({ tenant, ... })
  └─ conditionsFilter.filter({ filters, variables })
      └─ 匹配 → 使用此集, break 循环
```

---

## 8. 失败分支完整分析

### 8.1 OAuth URL 生成阶段失败点

| # | 位置 | 异常类型 | 触发条件 | 错误信息 |
|---|------|---------|---------|---------|
| 1 | `getIntegration()` | `NotFoundException` | 集成不存在 | `Integration not found: X in environment Y` |
| 2 | 主入口 switch default | `BadRequestException` | Provider 不支持 OAuth | `OAuth not supported for provider: X` |
| 3 | `validateSubscriberIdOrContext()` | `BadRequestException` | incoming_webhook 但无 subscriberId | `subscriberId is required for incoming webhook` |
| 4 | `assertResourceExists()` | `NotFoundException` | Subscriber 不存在 | `Subscriber not found: X` |
| 5 | `getIntegrationCredentials()` | `NotFoundException` | 集成无 credentials | `Slack integration missing credentials` |
| 6 | `getIntegrationCredentials()` | `NotFoundException` | 缺少 clientId | `Slack integration missing required OAuth credentials (clientId)` |
| 7 | `createSecureState()` | `BadRequestException` | 签名生成失败 | `Failed to create OAuth state signature` |
| 8 | `buildRedirectUri()` | `Error` | 缺少环境变量 | `API_ROOT_URL environment variable is required` |
| 9 | `getEnvironmentApiKey()` | `NotFoundException` | 环境无 API Key | `Environment ID: X not found` |

### 8.2 State 校验阶段失败点

| # | 位置 | 异常类型 | 触发条件 | 错误信息 |
|---|------|---------|---------|---------|
| 10 | `splitOAuthState()` | `Error` | State 格式错误 (无 '.') | `Invalid OAuth state: missing signature separator` |
| 11 | `validateAndDecodeState()` | `Error` | 签名不匹配 | `Invalid state signature` |
| 12 | `validateAndDecodeState()` | `Error` | State 过期 (>5分钟) | `OAuth state expired` |
| 13 | `validateAndDecodeState()` | `BadRequestException` | JSON 解析失败 | `Invalid OAuth state parameter` |
| 14 | `extractProviderIdFromState()` | `BadRequestException` | peek 后无 providerId | `Invalid state: missing providerId` |
| 15 | `decodeSlackState()` | `BadRequestException` | peek 后无 environmentId | `Invalid Slack state: missing environmentId` |
| 16 | `decodeSlackState()` | `NotFoundException` | 环境不存在 | `Environment not found: X` |
| 17 | `decodeSlackState()` | `NotFoundException` | 环境无 API Key | `Environment X has no API keys` |
| 18 | `decodeSlackState()` catch-all | `BadRequestException` | 其他错误 | `Invalid or expired Slack OAuth state parameter` |

### 8.3 回调处理阶段失败点

| # | 位置 | 异常类型 | 触发条件 | 错误信息 |
|---|------|---------|---------|---------|
| 19 | 主入口 switch default | `BadRequestException` | Provider 不支持回调 | `OAuth callback not supported for provider: X` |
| 20 | 主入口 Slack 分支 | `BadRequestException` | 缺少 code 参数 | `Missing required parameter: code` |
| 21 | `getIntegration()` | `NotFoundException` | 集成不存在 | `Slack integration not found: X in environment Y` |
| 22 | `getIntegrationCredentials()` | `NotFoundException` | 集成无 credentials | `Slack integration missing credentials` |
| 23 | `getIntegrationCredentials()` | `NotFoundException` | 缺少 clientId/secretKey | `Slack integration missing required OAuth credentials (clientId/clientSecret)` |
| 24 | `exchangeCodeForAuthData()` | `BadRequestException` | Slack API 返回错误 | `Slack OAuth error: X, metadata: Y` |
| 25 | `linkUserEndpoint()` | `BadRequestException` | link_user 但无 subscriberId | `subscriberId is required for link_user mode` |
| 26 | `linkUserEndpoint()` | `BadRequestException` | Slack 无返回 user ID | `Slack did not return a user ID in the OAuth response` |
| 27 | `createIncomingWebhookEndpoint()` | `BadRequestException` | incoming_webhook 但无 subscriberId | `subscriberId is required for incoming webhook` |

### 8.4 主渠道设置阶段失败点

| # | 位置 | 异常类型 | 触发条件 | 错误信息 |
|---|------|---------|---------|---------|
| 28 | `execute()` | `NotFoundException` | 集成不存在 | `Integration with id X not found` |
| 29 | `execute()` | `BadRequestException` | 渠道不支持 primary | `Channel X does not support primary` |
| 30 | `recalculatePriorityForAllActive()` | (DalException) | 环境/组织 ID 缺失 | `Deletion operation blocked for missing...` (类似) |

### 8.5 MsTeams 特有失败点

| # | 位置 | 异常类型 | 触发条件 | 错误信息 |
|---|------|---------|---------|---------|
| 31 | `generate-msteams-oauth-url` | `NotFoundException` | 缺少 clientId | `MS Teams integration missing clientId` |
| 32 | `generate-msteams-oauth-url` | `BadRequestException` | link_user 但无 subscriberId | `subscriberId is required for link_user mode` |
| 33 | `generate-msteams-oauth-url` | `NotFoundException` | 缺少 tenantId | `MS Teams integration missing tenantId` |
| 34 | `generate-msteams-oauth-url` | `BadRequestException` | 无 subscriberId 也无 context | `Either subscriberId or context must be provided` |
| 35 | `generate-msteams-oauth-url` | `NotFoundException` | Subscriber 不存在 | `Subscriber not found: X` |

### 8.6 失败分支流程图

```
OAuth 流程失败点总览:

┌─────────────┐
│ URL 生成    │ ──▶ 失败点 1-9 (集成/凭证/参数验证)
└─────────────┘
       │
       ▼
┌─────────────┐
│ 用户授权    │ ──▶ Slack/MsTeams 端错误 (不在代码控制内)
└─────────────┘
       │
       ▼
┌─────────────┐
│ 回调处理    │ ──▶ 失败点 10-27 (state 验证/token 换取)
└─────────────┘
       │
       ▼
┌─────────────┐
│ 集成管理    │ ──▶ 失败点 28-30 (primary 设置)
└─────────────┘
```

---

## 9. 关键代码引用

| 功能模块 | 文件路径 |
|---------|---------|
| **OAuth URL 生成入口** | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-chat-oauth-url.usecase.ts` |
| **Slack URL 生成** | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-slack-oath-url/generate-slack-oauth-url.usecase.ts` |
| **State 工具函数** | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/chat-oauth-state.util.ts` |
| **回调入口** | `apps/api/src/app/integrations/usecases/chat-oauth-callback/chat-oauth-callback.usecase.ts` |
| **Slack 回调** | `apps/api/src/app/integrations/usecases/chat-oauth-callback/slack-oauth-callback/slack-oauth-callback.usecase.ts` |
| **主渠道设置** | `apps/api/src/app/integrations/usecases/set-integration-as-primary/set-integration-as-primary.usecase.ts` |
| **集成选择** | `libs/application-generic/src/usecases/select-integration/select-integration.usecase.ts` |
| **集成 Repository** | `libs/dal/src/repositories/integration/integration.repository.ts` |
| **Channel 常量** | `packages/shared/src/types/channel.ts` |
| **旧版 Chat OAuth** | `apps/api/src/app/subscribers/usecases/chat-oauth/chat-oauth.usecase.ts` (已废弃) |

---

## 附录: 安全设计总结

| 安全措施 | 位置 | 目的 |
|---------|------|------|
| **HMAC State 签名** | `createSecureState()` | 防止篡改 state 内容 |
| **5 分钟 State 过期** | `validateAndDecodeState()` | 防止重放攻击 |
| **凭证加密存储** | `decryptCredentials()` | 防止数据库泄露导致密钥泄露 |
| **Peek 后强制验证** | `decodeSlackState()` | 确保 peek 获取的信息最终被验证 |
| **可选 HMAC 验证** | `validateEncryption()` | 额外的 subscriber 身份验证 |

---

**分析完成日期**: 2026-05-14  
**分析版本**: v1.0
