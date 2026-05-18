# Microsoft Teams 三方授权回调流程分析

## 一、整体架构概览

Microsoft Teams 授权回调流程贯穿三个核心阶段：**浏览器跳转入口** → **租户凭据持久化** → **订阅者通道绑定**。

```
用户点击连接按钮
      ↓
生成 OAuth URL (GenerateMsTeamsOauthUrl)
      ↓
浏览器跳转到 Microsoft 授权页面
      ↓
用户授权后回调 → /v1/integrations/chat/oauth/callback
      ↓
Controller 接收参数 (handleChatOAuthCallback)
      ↓
通用 ChatOauthCallback 根据 state 路由到 MsTeamsOauthCallback
      ↓
┌─────────────────────────────────────────────────────────┐
│ 两种 OAuth 模式                                         │
│  ┌──────────────┐        ┌──────────────────────────┐  │
│  │ admin_consent│        │ link_user                │  │
│  │ 租户级授权   │───────▶│ 用户级授权               │  │
│  │ 保存 tenantId│        │ 保存用户 oid + 安装 Bot │  │
│  └──────────────┘        └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
      ↓
创建 ChannelConnection / ChannelEndpoint
      ↓
后续发送消息时通过 MsTeamsTokenService 获取 token
```

---

## 二、入口层：浏览器跳转与 URL 生成

### 2.1 授权 URL 生成器

**文件**: `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-msteams-oath-url/generate-msteams-oauth-url.usecase.ts`

**核心方法**: `execute(command: GenerateMsTeamsOauthUrlCommand)`

#### 两种授权模式

| 模式 | 授权类型 | 端点 | 作用域 | 用途 |
|------|---------|------|--------|------|
| `connect` (默认) | Admin Consent | `/adminconsent` | `https://graph.microsoft.com/.default` | 租户管理员一次性授予应用权限 |
| `link_user` | Authorization Code | `/oauth2/v2.0/authorize` | `openid profile User.Read` | 单个用户登录并获取其 AAD Object ID |

#### State 安全机制

```typescript
// 状态数据结构 (StateData)
{
  identifier?: string;           // 通道连接标识
  subscriberId?: string;         // 订阅者 ID
  context?: ContextPayload;      // 上下文 payload
  environmentId: string;         // 环境 ID
  organizationId: string;        // 组织 ID
  integrationIdentifier: string; // 集成标识
  providerId: ChatProviderIdEnum; // 提供者 ID (MsTeams)
  timestamp: number;             // 时间戳 (5 分钟过期)
  mode?: OAuthMode;              // 'connect' | 'link_user'
  autoLinkUser?: boolean;        // 是否自动链式调用 link_user
}
```

**安全保障**:
1. 使用环境 API Key 对 payload 签名 (HMAC)
2. State 编码格式: `base64url(${jsonPayload}.${signature})`
3. 签名验证防止篡改，时间戳防止重放攻击

### 2.2 Controller 回调入口

**文件**: `apps/api/src/app/integrations/integrations.controller.ts:735`

```typescript
@Get('/chat/oauth/callback')
async handleChatOAuthCallback(
  @Query('code') providerCode?: string,      // link_user 模式的授权码
  @Query('tenant') tenant?: string,          // admin_consent 模式的租户 ID
  @Query('admin_consent') adminConsent?: string, // 是否管理员同意
  @Query('state') state?: string,            // 安全状态参数
)
```

**路由分发** (`chat-oauth-callback.usecase.ts`):
- 通过 `peekOAuthStatePayload` 从 state 中提取 `providerId`
- 根据 providerId 分发到具体处理器: `MsTeamsOauthCallback`

---

## 三、租户凭据持久化：Admin Consent 模式

### 3.1 处理流程

**文件**: `apps/api/src/app/integrations/usecases/chat-oauth-callback/msteams-oauth-callback/msteams-oauth-callback.usecase.ts:44`

```typescript
async execute(command: MsTeamsOauthCallbackCommand): Promise<ChatOauthCallbackResult> {
  const stateData = await this.decodeMsTeamsState(command.state);
  const integration = await this.getIntegration(stateData);
  
  if (stateData.mode === 'link_user') {
    await this.linkUserEndpoint(...);
  } else {
    await this.createAdminConsentConnection(command, stateData, integration);
    
    // 自动链式调用 link_user (如果 autoLinkUser === true)
    if (stateData.autoLinkUser === true && stateData.subscriberId) {
      const linkUserUrl = await this.generateMsTeamsOauthUrl.execute(...);
      return { type: ResponseTypeEnum.URL, result: linkUserUrl };
    }
  }
}
```

### 3.2 租户凭据持久化机制

**核心方法**: `createAdminConsentConnection()` (行 117-150)

```typescript
await this.createChannelConnection.execute(
  CreateChannelConnectionCommand.create({
    identifier: stateData.identifier,
    organizationId: stateData.organizationId,
    environmentId: stateData.environmentId,
    integrationIdentifier: integration.identifier,
    subscriberId: stateData.subscriberId,
    context: stateData.context,
    auth: { accessToken: 'app-only' },  // 标记为 app-only 模式
    workspace: { id: command.tenant },  // 持久化租户 ID
  })
);
```

**关键设计决策**:
- 不存储 token，只存储 `tenantId`
- 发送消息时通过 `client_credentials` 流程实时获取 token
- 避免了 refresh token 管理的复杂性

### 3.3 ChannelConnection 实体

**文件**: `apps/api/src/app/channel-connections/usecases/create-channel-connection/create-channel-connection.usecase.ts`

| 字段 | 说明 |
|------|------|
| `identifier` | 唯一标识，由调用方传入或自动生成 `chconn_xxx` |
| `integrationIdentifier` | 关联的集成配置 |
| `subscriberId` | 关联的订阅者 (可选，shared 模式为 null) |
| `workspace` | 存储 `{ id: tenantId }` |
| `auth` | 加密存储的认证信息，这里是 `{ accessToken: 'app-only' }` |
| `contextKeys` | 上下文标签数组 |

**唯一性约束**: 同一集成 + 订阅者 + 上下文组合只能有一个连接

---

## 四、订阅者通道绑定：link_user 模式

### 4.1 处理流程

**核心方法**: `linkUserEndpoint()` (行 152-183)

```
1. 验证 subscriberId 存在
2. 获取授权码 providerCode
3. 交换授权码换取 id_token
   ↓
4. 从 id_token 提取 oid (AAD Object ID)
   ↓
5. 为用户安装 Teams Bot
   ↓
6. 创建 ChannelEndpoint 绑定订阅者
```

### 4.2 授权码交换与 oid 提取

**方法**: `exchangeCodeForAadObjectId()` (行 343-380)

```typescript
const tokenParams = new URLSearchParams({
  grant_type: 'authorization_code',
  client_id: clientId,
  client_secret: secretKey,
  code: code,
  redirect_uri: callbackUrl,
  scope: 'openid profile User.Read',
});

// POST 到 /{tenantId}/oauth2/v2.0/token
// 从返回的 id_token 中提取 oid claim
```

**oid 提取**: JWT 的中间段 base64url 解码后取 `oid` 字段

### 4.3 Bot 自动安装

**方法**: `installBotForUser()` (行 185-203)

```
1. 获取 Graph API token (client_credentials 模式)
   ↓  MsTeamsTokenService.getGraphToken()
2. 解析 Teams App ID
   ↓  GET /appCatalogs/teamsApps?$filter=externalId eq '{clientId}'
3. 为用户安装 App
   ↓  POST /users/{oid}/teamwork/installedApps
```

**错误处理**:
- 409 Conflict: Bot 已安装，静默跳过
- 403 Forbidden: 权限未传播，提示用户等待
- 404 Not Found: 用户或应用不存在

### 4.4 ChannelEndpoint 实体

**文件**: `apps/api/src/app/channel-endpoints/usecases/create-channel-endpoint/create-channel-endpoint.usecase.ts`

```typescript
await this.createChannelEndpoint.execute(
  CreateChannelEndpointCommand.create({
    organizationId: stateData.organizationId,
    environmentId: stateData.environmentId,
    integrationIdentifier: integration.identifier,
    connectionIdentifier: stateData.identifier, // 关联到 ChannelConnection
    subscriberId: stateData.subscriberId,       // 绑定订阅者
    context: stateData.context,
    type: ENDPOINT_TYPES.MS_TEAMS_USER,         // 端点类型
    endpoint: { userId: oid },                  // 存储用户 AAD Object ID
  })
);
```

**关键关联**:
- `connectionIdentifier` 关联到之前创建的租户级 ChannelConnection
- `subscriberId` 绑定到具体订阅者
- `endpoint.userId` 存储用户的 AAD Object ID，用于后续发送消息

---

## 五、凭据服务：MsTeamsTokenService

**文件**: `libs/application-generic/src/services/ms-teams-token.service.ts`

### 5.1 两种 Token 类型

| 方法 | 用途 | 作用域 | 缓存 Key |
|------|------|--------|----------|
| `getGraphToken()` | Graph API 调用 (安装 Bot、查询应用等) | `https://graph.microsoft.com/.default` | `msteams:graph-token:{clientId}:{tenantId}:{secretHash}` |
| `getBotFrameworkToken()` | Bot Framework 发送消息 | `https://api.botframework.com/.default` | `msteams:bot-token:{clientId}:{tenantId}:{secretHash}` |

### 5.2 缓存设计

- 使用 `@CachedResponse` 装饰器，TTL 55 分钟 (1 小时 token 减 5 分钟缓冲)
- 缓存 Key 包含 `secretHash` (SHA-256 前 8 位)，密钥轮换后自动失效
- 获取失败时 `getBotFrameworkToken()` 返回空字符串，优雅降级

---

## 六、完整时序与数据流

### 6.1 Admin Consent + autoLinkUser 完整流程

```
用户点击 SDK MsTeamsConnectButton
      │
      ▼
GenerateMsTeamsOauthUrl.execute(mode='connect', autoLinkUser=true, subscriberId='xxx')
      │
      ├─► 创建 StateData (含 subscriberId, autoLinkUser=true)
      ├─► 签名并编码 state
      └─► 返回 adminconsent URL
      │
      ▼
浏览器跳转到 https://login.microsoftonline.com/organizations/v2.0/adminconsent?...
      │
      ▼
管理员授权后回调 /v1/integrations/chat/oauth/callback?tenant={tenantId}&admin_consent=True&state=...
      │
      ▼
ChatOauthCallback.execute()
      │
      ├─► peek 出 providerId=MsTeams
      └─► 转发到 MsTeamsOauthCallback.execute()
      │
      ▼
MsTeamsOauthCallback.execute(mode=connect)
      │
      ├─► 1. decodeMsTeamsState(state) → 验证签名、过期时间
      ├─► 2. getIntegration() → 查找集成配置
      ├─► 3. createAdminConsentConnection()
      │     └─► CreateChannelConnection 保存 tenantId
      │
      └─► 4. 检查 autoLinkUser === true && subscriberId 存在
            │
            ▼
            GenerateMsTeamsOauthUrl.execute(mode='link_user', subscriberId='xxx')
            │
            ├─► 验证 tenantId 已存在于集成凭据
            └─► 返回 link_user OAuth URL
            │
            ▼
Controller 返回 302 跳转到 link_user URL
      │
      ▼
用户登录 Microsoft 账号授权
      │
      ▼
回调 /v1/integrations/chat/oauth/callback?code={authCode}&state=...
      │
      ▼
MsTeamsOauthCallback.execute(mode=link_user)
      │
      ├─► 1. decodeMsTeamsState(state)
      ├─► 2. getIntegration()
      ├─► 3. linkUserEndpoint()
      │     ├─► exchangeCodeForAadObjectId(code) → 获取 oid
      │     ├─► installBotForUser(oid) → 调用 Graph API 安装 Bot
      │     └─► createChannelEndpoint() → 绑定 subscriberId + oid
      │
      └─► 5. 返回 HTML <script>window.close()</script>
```

### 6.2 数据模型关系

```
Integration (MS Teams 配置)
      │
      ├─ credentials: { clientId, secretKey, tenantId }
      │
      ▼
ChannelConnection (租户级连接)
      │
      ├─ integrationIdentifier: 关联 Integration
      ├─ subscriberId: 可选 (shared 模式为 null)
      ├─ workspace: { id: tenantId }
      └─ auth: { accessToken: 'app-only' }
      │
      ▼
ChannelEndpoint (用户级端点)
      │
      ├─ connectionIdentifier: 关联 ChannelConnection
      ├─ subscriberId: 绑定到具体订阅者
      ├─ type: 'ms-teams-user'
      └─ endpoint: { userId: oid }
      │
      ▼
发送消息时:
  1. 通过 subscriberId 查找 ChannelEndpoint
  2. 通过 connectionIdentifier 查找 ChannelConnection
  3. 从 Connection 取 tenantId，从 Integration 取 clientId/secretKey
  4. MsTeamsTokenService.getBotFrameworkToken() 获取 token
  5. 使用 endpoint.userId 作为收件人发送消息
```

---

## 七、关键设计亮点

1. **双模式授权**: Admin Consent 做租户级授权，link_user 做用户级绑定，职责分离清晰
2. **无状态 Token 管理**: 不存储 refresh token，通过 client_credentials 实时获取，降低复杂度
3. **链式授权**: `autoLinkUser` 参数允许一次点击完成两步授权，用户体验好
4. **安全 State 机制**: 签名 + 过期时间双重保障，防止 CSRF 和重放攻击
5. **优雅降级**: Bot Framework token 获取失败返回空，不阻塞整个发送流程
6. **缓存感知密钥轮换**: 缓存 Key 包含密钥哈希，密钥更新后缓存自动失效

---

## 八、核心文件索引

| 文件 | 职责 |
|------|------|
| `integrations.controller.ts:735` | 回调入口 Controller |
| `chat-oauth-callback.usecase.ts` | 按 providerId 路由回调 |
| `generate-msteams-oauth-url.usecase.ts` | 生成授权 URL、State 编解码验证 |
| `msteams-oauth-callback.usecase.ts` | MS Teams 回调核心逻辑 |
| `create-channel-connection.usecase.ts` | 创建租户级通道连接 |
| `create-channel-endpoint.usecase.ts` | 创建用户级通道端点 |
| `ms-teams-token.service.ts` | Graph/Bot Framework Token 服务 |
| `chat-oauth-state.util.ts` | State 编解码工具 |
