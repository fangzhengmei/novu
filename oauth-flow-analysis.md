# Novu OAuth 集成流程完整分析

**生成日期**: 2026-05-14  
**分析范围**: Chat 渠道 OAuth 完整流程

---

## 目录

1. [整体架构概述](#1-整体架构概述)
2. [聊天 OAuth URL 生成流程](#2-聊天-oauth-url-生成流程)
3. [Slack OAuth 回调处理流程](#3-slack-oauth-回调处理流程)
4. [OAuth State 校验机制](#4-oauth-state-校验机制)
5. [Provider 凭证隔离机制](#5-provider-凭证隔离机制)
6. [失败分支完整分析](#6-失败分支完整分析)
7. [关键代码引用与证据](#7-关键代码引用与证据)

---

## 1. 整体架构概述

### 1.1 核心分支优先级

**重要修正**: `incoming-webhook` 不是独立模式，而是由 **scope 参数触发** 的回调分支。OAuth 模式仅有 **2 种**，回调分支按以下优先级判定：

| 优先级 | 判定条件 | 分支类型 | 触发来源 |
|-------|---------|---------|---------|
| **1 (最高)** | `stateData.mode === 'link_user'` | link_user 模式 | State 中的 mode 字段 |
| **2** | `authData.incoming_webhook` 存在 | incoming-webhook 分支 | scope 参数包含 `incoming-webhook` |
| **3 (默认)** | 以上都不满足 | workspace connect | 默认行为 |

### 1.2 涉及的核心文件

| 模块 | 文件路径 | 功能 |
|------|----------|------|
| **URL 生成入口** | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-chat-oauth-url.usecase.ts` | 统一入口，根据 providerId 分发 |
| **Slack URL 生成** | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-slack-oath-url/generate-slack-oauth-url.usecase.ts` | Slack OAuth URL 生成 |
| **回调入口** | `apps/api/src/app/integrations/usecases/chat-oauth-callback/chat-oauth-callback.usecase.ts` | 统一回调入口 |
| **Slack 回调** | `apps/api/src/app/integrations/usecases/chat-oauth-callback/slack-oauth-callback/slack-oauth-callback.usecase.ts` | Slack 回调具体逻辑 |
| **State 工具** | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/chat-oauth-state.util.ts` | State 编码/解码/签名 |

---

## 2. 聊天 OAuth URL 生成流程

### 2.1 OAuth 模式定义

**代码位置**: `generate-slack-oauth-url.usecase.ts:21`

```typescript
export type OAuthMode = 'connect' | 'link_user';
```

> **事实修正**: 只有 **2 种模式**，没有 `incoming-webhook` 模式。incoming-webhook 是 scope 触发的回调分支。

### 2.2 Slack OAuth URL 生成完整流程

```
execute(command)
  │
  ├─ validateSubscriberIdOrContext(command)
  │   └─ scope 包含 incoming-webhook?
  │       └─ 是 → 必须提供 subscriberId
  │           └─ 失败点 #3: subscriberId is required for incoming webhook
  │
  ├─ assertResourceExists(command)
  │   └─ subscriberId 存在? → 在 DB 中查找 subscriber
  │       └─ 失败点 #4: Subscriber not found
  │
  ├─ getIntegrationCredentials(command.integration)
  │   ├─ Novu 演示集成 → GetNovuProviderCredentials
  │   └─ 客户自有集成 → 从 integration.credentials 读取
  │       ├─ 失败点 #5: Slack integration missing credentials
  │       └─ 失败点 #6: missing clientId
  │
  ├─ createSecureState(...)
  │   ├─ 构建 StateData 对象 (含 mode, subscriberId, organizationId, environmentId 等)
  │   ├─ JSON 序列化 payload
  │   ├─ 用环境 API Key 生成 HMAC 签名
  │   └─ base64url 编码: `${payload}.${signature}`
  │       └─ 失败点 #7: Failed to create OAuth state signature
  │
  ├─ resolveBotScopes(command)
  │   ├─ 指定了 scope? → 使用传入的 scope
  │   ├─ 集成关联 Agent? → 使用 SLACK_AGENT_OAUTH_SCOPES
  │   └─ 否则 → 使用默认 scope (undefined)
  │
  └─ getOAuthUrl(clientId, secureState, resolvedScope, userScope, command.mode)
      ├─ mode === 'link_user'? → 设置 user_scope (identity.basic)
      └─ 否则 → 设置 scope (默认或传入的, 可能包含 incoming-webhook)
```

### 2.3 StateData 结构

**代码位置**: `generate-slack-oauth-url.usecase.ts:23-35`

```typescript
export type StateData = {
  identifier?: string;              // 连接标识符
  subscriberId?: string;            // 订阅者 ID (incoming-webhook 必传)
  context?: ContextPayload;         // 上下文
  environmentId: string;            // ✓ 环境 ID (必填)
  organizationId: string;           // ✓ 组织 ID (必填)
  integrationIdentifier: string;    // ✓ 集成标识符 (必填)
  providerId: ChatProviderIdEnum;   // ✓ 提供商 ID (必填)
  timestamp: number;                // ✓ 创建时间戳 (必填)
  mode?: OAuthMode;                 // OAuth 模式: 'connect' | 'link_user'
  connectionMode?: ConnectionMode;  // 连接模式
  autoLinkUser?: boolean;           // 是否自动链接用户
};
```

### 2.4 scope 与 incoming-webhook 的关系

**代码位置**: `generate-slack-oauth-url.usecase.ts:112-115`

```typescript
if (scope?.includes('incoming-webhook')) {
  if (!subscriberId) {
    throw new BadRequestException('subscriberId is required for incoming webhook');
  }
}
```

**关键说明**:
- incoming-webhook 是 **作为 scope 值** 传入，不是模式
- 传入 scope=['incoming-webhook'] 会触发 subscriberId 校验
- 真正的分支判定发生在 **回调阶段**，而不是 URL 生成阶段

---

## 3. Slack OAuth 回调处理流程

### 3.1 三分支判定逻辑（按优先级）

**代码位置**: `slack-oauth-callback.usecase.ts:48-97`

```typescript
if (stateData.mode === 'link_user') {
  // 优先级 1: link_user 模式 (由 state 中的 mode 字段控制)
  await this.linkUserEndpoint(stateData, integration, authData);
} else if (authData.incoming_webhook) {
  // 优先级 2: incoming-webhook 分支 (由 Slack 返回的 authData 控制, 触发条件是 URL 生成时 scope 包含 incoming-webhook)
  await this.createIncomingWebhookEndpoint(stateData, integration, authData);
} else {
  // 优先级 3: 默认 workspace connect
  await this.createChannelConnection.execute(...);
  // + 可选的 autoLinkUser
}
```

### 3.2 分支 1: link_user 模式

**触发条件**: `stateData.mode === 'link_user'` (State 中明确设置)

**行为**: 创建 SLACK_USER 类型的 channel-endpoint

```typescript
linkUserEndpoint(stateData, integration, authData)
  ├─ 检查 subscriberId 存在
  │   └─ 失败点 #21: subscriberId is required for link_user mode
  ├─ 从 Slack authData 获取 authed_user.id
  │   └─ 失败点 #22: Slack did not return a user ID
  └─ createChannelEndpoint.execute({
       type: ENDPOINT_TYPES.SLACK_USER,
       endpoint: { userId: authData.authed_user.id }
     })
```

### 3.3 分支 2: incoming-webhook (scope 触发)

**触发条件**: 
- URL 生成时 scope 包含 `incoming-webhook`
- Slack OAuth 响应中 `authData.incoming_webhook` 存在

**代码位置**: `slack-oauth-callback.usecase.ts:50-63`

```typescript
/*
 * Incoming webhooks are handled differently from workspace connections:
 *
 * - Incoming webhook: Creates a stateless endpoint tied to a specific subscriber
 *   using only the webhook URL. This provides direct message delivery.
 *
 * - Workspace connection: Uses access_token for broader workspace access
 *   and is not tied to a specific subscriber.
 *
 * While authData contains both access_token and channel_id, we intentionally
 * use only the webhook URL to maintain clear separation of concerns.
 */
```

**行为**: 创建 WEBHOOK 类型的 channel-endpoint

```typescript
createIncomingWebhookEndpoint(stateData, integration, authData)
  ├─ 检查 subscriberId 存在
  │   └─ 失败点 #23: subscriberId is required for incoming webhook
  └─ createChannelEndpoint.execute({
       type: ENDPOINT_TYPES.WEBHOOK,
       endpoint: { url: authData.incoming_webhook.url }
     })
```

> **设计意图**: 虽然 authData 同时包含 access_token 和 channel_id，但只使用 webhook URL 以保持关注点分离。

### 3.4 分支 3: workspace connect (默认)

**触发条件**: 不满足以上两个条件

**行为**: 创建 channel-connection + 可选 autoLinkUser

```typescript
else
  ├─ createChannelConnection.execute({
  │    auth: { accessToken: authData.access_token },
  │    workspace: { id: authData.team.id, name: authData.team.name }
  │  })
  │
  └─ autoLinkUser === true && subscriberId 存在 && authed_user.id 存在?
     └─ createChannelEndpoint.execute({
          type: ENDPOINT_TYPES.SLACK_USER,
          endpoint: { userId: authData.authed_user.id }
        })
```

---

## 4. OAuth State 校验机制

### 4.1 编码格式设计

**代码位置**: `chat-oauth-state.util.ts`

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
| `splitOAuthState(state)` | 分割 payload 和 signature | - |
| `peekOAuthStatePayload<T>(state)` | 无验证解析 payload | ❌ |
| `validateAndDecodeState()` | 完整验证 | ✅ |

### 4.3 完整校验流程 (validateAndDecodeState)

**代码位置**: `generate-slack-oauth-url.usecase.ts:202-222`

```typescript
static async validateAndDecodeState(state, environmentApiKey)
  │
  ├─ splitOAuthState(state) → 分离 payload 和 signature
  │   └─ 无 '.'? → 后续在 catch 中抛出: Invalid OAuth state parameter
  │
  ├─ expectedSignature = HMAC_SHA256(环境 API Key, payload)
  ├─ if (signature !== expectedSignature)
  │   └─ throw Error('Invalid state signature')
  │
  ├─ JSON.parse(payload)
  │   └─ 解析失败? → BadRequestException: Invalid OAuth state parameter
  │
  └─ 检查过期 (5 分钟)
      └─ 过期? → Error: OAuth state expired
```

### 4.4 "Peek 模式" 的设计意图

`peekOAuthStatePayload()` 是故意设计的不安全函数，用于解决"鸡生蛋"问题：

**场景**: 在回调入口需要知道 providerId 来分发到正确的 handler
- 但签名验证需要 environment API Key
- environment API Key 需要 environmentId 才能查找
- environmentId 存在于 state payload 中

**解决方案**:
1. 先用 `peekOAuthStatePayload()` 无验证获取 providerId 和 environmentId
2. 用 environmentId 查找环境并获取 API Key
3. **然后** 调用 `validateAndDecodeState()` 进行完整验证
4. 验证通过后才信任所有字段

**代码位置**: `slack-oauth-callback.usecase.ts:232-255`

---

## 5. Provider 凭证隔离机制

### 5.1 两层凭证架构

```
Integration Credentials
        │
  ┌─────┴─────┐
  │           │
  ▼           ▼
Novu 演示集成    客户自有集成
providerId === 'novu'  providerId === 'slack'
  │           │
  ▼           ▼
GetNovuProviderCredentials  integration.credentials
(中央服务获取)       (数据库存储, 加密)
              │
              ▼
        decryptCredentials()
          (回调时解密)
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

### 5.3 凭证解密时机

- **URL 生成阶段**: 不解密，只需要 clientId
- **回调阶段 (exchangeCodeForAuthData)**: 必须解密获取 secretKey 用于调用 Slack token 接口

**代码位置**: `slack-oauth-callback.usecase.ts:205-229`

```typescript
private async exchangeCodeForAuthData(providerCode, integrationCredentials) {
  const credentials = decryptCredentials(integrationCredentials);  // ← 解密
  // 使用 credentials.clientId 和 credentials.secretKey 调用 Slack API
}
```

---

## 6. 失败分支完整分析

### 6.1 URL 生成阶段

| # | 错误信息 | 触发位置 | 触发条件 | 影响说明 |
|---|---------|---------|---------|---------|
| 1 | `Integration not found: X in environment Y` | `getIntegration()` | 集成不存在或不匹配 | 用户无法开始 OAuth 流程 |
| 2 | `OAuth not supported for provider: X` | 主入口 switch | Provider 不支持 OAuth | 不支持的提供商 |
| 3 | `subscriberId is required for incoming webhook` | `validateSubscriberIdOrContext()` | scope 包含 incoming-webhook 但无 subscriberId | incoming-webhook 分支无法使用 |
| 4 | `Subscriber not found: X` | `assertResourceExists()` | Subscriber 在数据库中不存在 | subscriberId 无效 |
| 5 | `Slack integration missing credentials` | `getIntegrationCredentials()` | integration.credentials 为空 | 集成配置不完整 |
| 6 | `Slack integration missing required OAuth credentials (clientId)` | `getIntegrationCredentials()` | credentials.clientId 为空 | 集成配置不完整 |
| 7 | `Failed to create OAuth state signature` | `createSecureState()` | HMAC 签名生成失败 | 安全校验失败 |
| 8 | `API_ROOT_URL environment variable is required` | `buildRedirectUri()` | 环境变量缺失 | 部署配置错误 |
| 9 | `Environment ID: X not found` | `getEnvironmentApiKey()` | 环境无 API Keys | 环境配置异常 |

### 6.2 State 校验阶段

| # | 错误信息 | 触发位置 | 触发条件 | 影响说明 |
|---|---------|---------|---------|---------|
| 10 | `Invalid OAuth state parameter` | `validateAndDecodeState()` catch | JSON 解析失败或其他校验错误 | state 被篡改或格式错误 |
| 11 | `Invalid state signature` | `validateAndDecodeState()` | HMAC 签名不匹配 | state 被篡改 |
| 12 | `OAuth state expired` | `validateAndDecodeState()` | state 超过 5 分钟 | 用户授权耗时过长或重放攻击 |
| 13 | `Invalid Slack state: missing environmentId` | `decodeSlackState()` | peek 后无 environmentId | state 结构损坏 |
| 14 | `Environment not found: X` | `decodeSlackState()` | 环境不存在 | environmentId 无效 |
| 15 | `Environment X has no API keys` | `decodeSlackState()` | 环境无 API Keys | 环境配置异常 |
| 16 | `Invalid or expired Slack OAuth state parameter` | `decodeSlackState()` catch-all | 其他未知错误 | 兜底错误处理 |

### 6.3 回调处理阶段

| # | 错误信息 | 触发位置 | 触发条件 | 影响说明 |
|---|---------|---------|---------|---------|
| 17 | `OAuth callback not supported for provider: X` | 主入口 switch | Provider 不支持回调 | 不支持的提供商 |
| 18 | `Missing required parameter: code` | 主入口 Slack 分支 | code 参数缺失 | Slack 回调参数异常 |
| 19 | `Slack integration not found: X in environment Y` | `getIntegration()` | 集成不存在 | 集成被删除或 state 篡改 |
| 20 | `Slack integration missing required OAuth credentials (clientId/clientSecret)` | `getIntegrationCredentials()` | clientId 或 secretKey 缺失 | 集成配置不完整 |
| 21 | `Slack OAuth error: X` | `exchangeCodeForAuthData()` | Slack API 返回错误 | code 过期、无效、或凭证错误 |
| 22 | `subscriberId is required for link_user mode` | `linkUserEndpoint()` | link_user 模式但无 subscriberId | state 不完整 |
| 23 | `Slack did not return a user ID in the OAuth response` | `linkUserEndpoint()` | Slack 响应无 authed_user.id | Slack API 异常 |
| 24 | `subscriberId is required for incoming webhook` | `createIncomingWebhookEndpoint()` | incoming-webhook 分支但无 subscriberId | state 不完整 |

---

## 7. 关键代码引用与证据

### 7.1 incoming-webhook 是 scope 分支的证据

| 证据点 | 文件 | 行号 | 代码 |
|-------|------|-----|------|
| OAuthMode 只有 2 种 | generate-slack-oauth-url.usecase.ts | 21 | `export type OAuthMode = 'connect' \| 'link_user';` |
| incoming-webhook 作为 scope 校验 | generate-slack-oauth-url.usecase.ts | 112-115 | `if (scope?.includes('incoming-webhook'))` |
| 回调三分支判定 | slack-oauth-callback.usecase.ts | 48-97 | `if (mode === 'link_user') ... else if (incoming_webhook) ... else` |
| incoming-webhook 分支注释 | slack-oauth-callback.usecase.ts | 51-62 | 完整设计意图注释 |

### 7.2 State 校验机制证据

| 证据点 | 文件 | 行号 | 代码 |
|-------|------|-----|------|
| validateAndDecodeState 实现 | generate-slack-oauth-url.usecase.ts | 202-222 | 签名校验 + 过期校验 + JSON 解析 |
| peekOAuthStatePayload 使用 | slack-oauth-callback.usecase.ts | 234 | 用于 preliminaryData 获取 |
| 5 分钟过期设置 | generate-slack-oauth-url.usecase.ts | 214 | `const FIVE_MINUTES = 5 * 60 * 1000;` |

### 7.3 凭证隔离机制证据

| 证据点 | 文件 | 行号 | 代码 |
|-------|------|-----|------|
| decryptCredentials 调用 | slack-oauth-callback.usecase.ts | 206 | 回调阶段解密 |
| 两层凭证分支 | generate-slack-oauth-url.usecase.ts | 235-248 | Novu 演示集成 vs 客户自有集成 |

### 7.4 分支优先级证据

**代码位置**: `slack-oauth-callback.usecase.ts:48-97`

判定顺序（不可改变）:
1. **第一判断**: `mode === 'link_user'` - State 显式控制
2. **第二判断**: `authData.incoming_webhook` 存在 - Slack 响应控制
3. **第三分支**: `else` - 默认 workspace connect

> **影响说明**: 如果同时满足多个条件，优先级高的分支会先执行。例如 mode='link_user' 时，即使 scope 包含 incoming-webhook 也会被忽略。

---

## 附录: 关键发现总结

1. **incoming-webhook 不是模式**: 是由 scope 参数触发的回调分支，不是独立 OAuth 模式
2. **分支优先级明确**: link_user > incoming-webhook > workspace connect，优先级由代码顺序决定
3. **State 双阶段验证**: peek 无验证预读 → 完整签名校验，解决了"鸡生蛋"问题
4. **凭证分层加密**: 演示集成与客户自有集成分离，secretKey 加密存储，仅回调时解密
5. **关注点分离设计**: incoming-webhook 分支只使用 webhook URL，即使 Slack 返回了 access_token 也不使用

---

**分析完成日期**: 2026-05-14  
**分析版本**: v2.0
