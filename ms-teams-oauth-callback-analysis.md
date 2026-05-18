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

## 三、回调核心逻辑：MsTeamsOauthCallback.execute()

**文件**: `apps/api/src/app/integrations/usecases/chat-oauth-callback/msteams-oauth-callback/msteams-oauth-callback.usecase.ts:44`

### 3.1 执行流程总览

```typescript
async execute(command: MsTeamsOauthCallbackCommand): Promise<ChatOauthCallbackResult> {
  const stateData = await this.decodeMsTeamsState(command.state);
  const integration = await this.getIntegration(stateData);
  const credentials = await this.getIntegrationCredentials(integration);

  if (stateData.mode === 'link_user') {
    // 用户级授权流程
    try {
      await this.linkUserEndpoint(command, stateData, integration, credentials);
    } catch (error) {
      // 关键：Bot 安装失败时返回友好 HTML 错误页，而非抛出异常
      if (message.includes('MS Teams bot installation failed')) {
        return { type: ResponseTypeEnum.HTML, result: this.buildErrorHtml(message) };
      }
      throw error;
    }
  } else {
    // 租户级授权流程
    await this.createAdminConsentConnection(command, stateData, integration);

    // 关键：autoLinkUser 链式调用逻辑
    if (stateData.autoLinkUser === true && stateData.subscriberId) {
      try {
        const linkUserUrl = await this.generateMsTeamsOauthUrl.execute(...);
        return { type: ResponseTypeEnum.URL, result: linkUserUrl };
      } catch (error) {
        // 关键：链式调用失败只打 warn 日志，不中断流程
        this.logger.warn(`Could not chain link_user redirect after admin consent: ...`);
      }
    }
  }

  // 关键：返回分流逻辑（仅在没有提前 return 时才执行到这里）
  if (credentials.redirectUrl) {
    return { type: ResponseTypeEnum.URL, result: credentials.redirectUrl };
  }

  return {
    type: ResponseTypeEnum.HTML,
    result: '<script>window.close();</script>',
  };
}
```

### 3.2 返回分流优先级

| 优先级 | 条件 | 返回类型 | 说明 |
|--------|------|----------|------|
| 1 | `autoLinkUser === true && subscriberId 存在` 且链式调用成功 | `URL` | 跳转到 link_user OAuth URL |
| 2 | `credentials.redirectUrl 存在` | `URL` | 跳转到集成配置的自定义回调地址 |
| 3 | 其他情况 | `HTML` | 返回 `<script>window.close()</script>` 关闭弹窗 |

**注意**：
- autoLinkUser 链式调用成功时，会直接 return，不会执行后面的 redirectUrl 检查
- autoLinkUser 链式调用失败时（catch 住），会继续执行后面的 redirectUrl 检查
- link_user 模式完成后，也会执行相同的返回分流逻辑

### 3.3 link_user 失败回退分支

**文件**: 行 53-64

```typescript
if (stateData.mode === 'link_user') {
  try {
    await this.linkUserEndpoint(command, stateData, integration, credentials);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);

    // 关键：只有 Bot 安装相关错误才返回友好 HTML 页面
    if (message.includes('MS Teams bot installation failed')) {
      return { type: ResponseTypeEnum.HTML, result: this.buildErrorHtml(message) };
    }

    // 其他错误（如授权码无效、oid 提取失败等）直接抛出
    throw error;
  }
}
```

**Bot 安装失败的具体场景**（`buildErrorHtml` 会渲染友好提示）：
- 403 Forbidden: 缺少 `AppCatalog.Read.All` 或 `TeamsAppInstallation.ReadWriteSelfForUser.All` 权限
- 404 Not Found: 应用未发布到组织目录，或用户不存在
- 其他 Graph API 调用错误

**权限传播提示**: 错误信息包含特定权限名称时，会额外提示用户 "Azure permission changes may take up to 60 minutes to propagate"

---

## 四、租户凭据持久化：Admin Consent 模式

### 4.1 处理流程

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
    workspace: { id: command.tenant },  // ⚠️ 持久化的是用户租户 ID（来自回调参数）
  })
);
```

**关键设计决策**:
- 不存储 token，只存储从回调参数 `command.tenant` 获取的租户 ID
- 发送消息时通过 `client_credentials` 流程实时获取 token
- 避免了 refresh token 管理的复杂性

### 4.2 ChannelConnection 实体

**文件**: `apps/api/src/app/channel-connections/usecases/create-channel-connection/create-channel-connection.usecase.ts`

| 字段 | 说明 |
|------|------|
| `identifier` | 唯一标识，由调用方传入或自动生成 `chconn_xxx` |
| `integrationIdentifier` | 关联的集成配置 |
| `subscriberId` | 关联的订阅者 (可选，shared 模式为 null) |
| `workspace` | 存储 `{ id: tenantId }` - **用户租户 ID，来自 admin_consent 回调** |
| `auth` | 加密存储的认证信息，这里是 `{ accessToken: 'app-only' }` |
| `contextKeys` | 上下文标签数组 |

**唯一性约束**: 同一集成 + 订阅者 + 上下文组合只能有一个连接

---

## 五、订阅者通道绑定：link_user 模式

### 5.1 处理流程

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

### 5.2 授权码交换与 oid 提取

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
// ⚠️ 这里的 tenantId 来自 integration.credentials.tenantId（Bot 租户 ID）
// 从返回的 id_token 中提取 oid claim
```

**oid 提取**: JWT 的中间段 base64url 解码后取 `oid` 字段

### 5.3 Bot 自动安装

**方法**: `installBotForUser()` (行 185-203)

```
1. 获取 Graph API token (client_credentials 模式)
   ↓  MsTeamsTokenService.getGraphToken()
   ⚠️ 使用 integration.credentials.tenantId（Bot 租户 ID）
2. 解析 Teams App ID
   ↓  GET /appCatalogs/teamsApps?$filter=externalId eq '{clientId}'
3. 为用户安装 App
   ↓  POST /users/{oid}/teamwork/installedApps
```

**错误处理**:
- 409 Conflict: Bot 已安装，静默跳过
- 403 Forbidden: 权限未传播，提示用户等待
- 404 Not Found: 用户或应用不存在

### 5.4 ChannelEndpoint 实体

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

## 六、发送阶段 tenant 信息的来源区分

### 6.1 两个不同的 tenantId

**⚠️ 关键理解修正**：发送阶段存在两个不同来源的 tenantId，用途完全不同：

| 来源 | 存储位置 | 用途 | 说明 |
|------|----------|------|------|
| `subscriberTenantId` | `ChannelConnection.workspace.id` | Bot Framework API 调用时标识**用户所在租户** | 来自 admin_consent 回调参数 `tenant`，每个连接可能不同 |
| `tenantId` | `Integration.credentials.tenantId` | 获取 Bot Framework token 时标识**Bot 所在租户** | 集成配置时预先填写，所有连接共享 |

### 6.2 发送时的数据流

**文件**: `apps/worker/src/app/workflow/usecases/send-message/channel-endpoint-resolution/resolve-channel-endpoints.usecase.ts:191-229`

```typescript
private async extractMsTeamsToken(endpoint, connectionMap): Promise<Record<string, unknown>> {
  // 1. 从 ChannelConnection 获取用户租户 ID
  const connection = endpoint.connectionIdentifier ? connectionMap.get(endpoint.connectionIdentifier) : undefined;
  const subscriberTenantId = connection?.workspace?.id;  // ⚠️ 来自 admin_consent 回调

  if (!subscriberTenantId) {
    throw new Error(`MS Teams endpoint ${endpoint.identifier} requires a connection with tenant ID`);
  }

  // 2. 从 Integration 获取 Bot 租户 ID 和密钥
  const integration = await this.integrationRepository.findOne({...});
  const decryptedCredentials = decryptCredentials(integration.credentials);
  const { clientId, secretKey, tenantId } = decryptedCredentials;  // ⚠️ 来自集成配置

  // 3. 使用 Bot 租户 ID 获取 token
  const token = await this.msTeamsTokenService.getBotFrameworkToken(clientId, secretKey, tenantId);

  // 4. 返回时携带用户租户 ID（用于 Bot Framework API 调用）
  if (endpoint.type === ENDPOINT_TYPES.MS_TEAMS_USER) {
    return { subscriberTenantId, token, clientId };
  }

  return { subscriberTenantId, token };
}
```

### 6.3 Provider 发送时的使用

**文件**: `packages/providers/src/lib/chat/msTeams/msTeams.provider.ts`

```typescript
// 发送用户消息时
private async sendUserMessage(channelData: MsTeamsUserData, content: string) {
  const { endpoint, subscriberTenantId, token, clientId } = channelData;
  const { userId } = endpoint;

  const conversationPayload = {
    isGroup: false,
    bot: { id: clientId },
    members: [{ id: userId }],
    channelData: {
      tenant: { id: subscriberTenantId },  // ⚠️ 使用用户租户 ID
    },
  };

  // POST 到 Bot Framework 使用 token 认证
  const response = await this.axiosInstance.post(
    `${MsTeamsProvider.BOT_FRAMEWORK_SERVICE_URL}/teams/v3/conversations`,
    conversationPayload,
    { headers: { Authorization: `Bearer ${token}` } }
  );
}
```

### 6.4 数据模型关系图

```
Integration (MS Teams 配置)
      │
      ├─ credentials: { clientId, secretKey, tenantId }  ← Bot 租户 ID（配置时填写）
      │
      ▼
ChannelConnection (租户级连接)
      │
      ├─ integrationIdentifier: 关联 Integration
      ├─ subscriberId: 可选 (shared 模式为 null)
      ├─ workspace: { id: tenantId }  ← ⚠️ 用户租户 ID（来自 admin_consent 回调）
      └─ auth: { accessToken: 'app-only' }
      │
      ▼
ChannelEndpoint (用户级端点)
      │
      ├─ connectionIdentifier: 关联 ChannelConnection
      ├─ subscriberId: 绑定到具体订阅者
      ├─ type: 'ms-teams-user'
      └─ endpoint: { userId: oid }  ← 用户 AAD Object ID
      │
      ▼
发送消息时 (resolve-channel-endpoints.usecase.ts):
  1. 通过 subscriberId 查找 ChannelEndpoint
  2. 通过 connectionIdentifier 查找 ChannelConnection
  3. 从 Connection 取 subscriberTenantId（用户租户 ID）
  4. 从 Integration 取 clientId/secretKey/tenantId（Bot 租户 ID）
  5. MsTeamsTokenService.getBotFrameworkToken(clientId, secretKey, tenantId) → 获取 token
  6. 组装 channelData: { subscriberTenantId, token, clientId, endpoint: { userId } }
  7. Provider 使用 subscriberTenantId 调用 Bot Framework API
```

---

## 七、凭据服务：MsTeamsTokenService

**文件**: `libs/application-generic/src/services/ms-teams-token.service.ts`

### 7.1 两种 Token 类型

| 方法 | 用途 | 作用域 | 缓存 Key |
|------|------|--------|----------|
| `getGraphToken()` | Graph API 调用 (安装 Bot、查询应用等) | `https://graph.microsoft.com/.default` | `msteams:graph-token:{clientId}:{tenantId}:{secretHash}` |
| `getBotFrameworkToken()` | Bot Framework 发送消息 | `https://api.botframework.com/.default` | `msteams:bot-token:{clientId}:{tenantId}:{secretHash}` |

### 7.2 缓存设计

- 使用 `@CachedResponse` 装饰器，TTL 55 分钟 (1 小时 token 减 5 分钟缓冲)
- 缓存 Key 包含 `secretHash` (SHA-256 前 8 位)，密钥轮换后自动失效
- 获取失败时 `getBotFrameworkToken()` 返回空字符串，优雅降级

---

## 八、完整时序与数据流

### 8.1 Admin Consent + autoLinkUser 完整流程

```
用户点击 SDK MsTeamsConnectButton (autoLinkUser=true, subscriberId='xxx')
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
管理员授权后回调 /v1/integrations/chat/oauth/callback?tenant={subscriberTenantId}&admin_consent=True&state=...
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
      ├─► 2. getIntegration() → 查找集成配置（含 Bot tenantId）
      ├─► 3. getIntegrationCredentials() → 验证 clientId/secretKey/tenantId 存在
      ├─► 4. createAdminConsentConnection()
      │     └─► CreateChannelConnection 保存 subscriberTenantId 到 workspace.id
      │
      └─► 5. 检查 autoLinkUser === true && subscriberId 存在
            │
            ├─ 成功：生成 linkUserUrl 并 return → 跳转到 link_user OAuth
            │     │
            │     ▼
            │   用户登录 Microsoft 账号授权
            │     │
            │     ▼
            │   回调 /v1/integrations/chat/oauth/callback?code={authCode}&state=...
            │     │
            │     ▼
            │   MsTeamsOauthCallback.execute(mode=link_user)
            │     │
            │     ├─► 1. decodeMsTeamsState(state)
            │     ├─► 2. getIntegration()
            │     ├─► 3. linkUserEndpoint()
            │     │     ├─► exchangeCodeForAadObjectId(code) → 获取 oid
            │     │     │     └─► 使用 Bot tenantId 调用 token 端点
            │     │     ├─► installBotForUser(oid) → 调用 Graph API 安装 Bot
            │     │     │     └─► 使用 Bot tenantId 获取 Graph token
            │     │     └─► createChannelEndpoint() → 绑定 subscriberId + oid
            │     │
            │     └─► 4. 返回分流：redirectUrl 或 window.close()
            │
            └─ 失败：catch 住，打 warn 日志 → 继续执行返回分流
                  │
                  ▼
                返回分流：redirectUrl 或 window.close()
```

### 8.2 autoLinkUser 条件触发矩阵

| autoLinkUser | subscriberId | 行为 |
|--------------|--------------|------|
| `true` (显式) | 存在 | 链式调用 link_user，成功则跳转，失败则回退到正常返回 |
| `true` (显式) | 不存在 | 不触发链式调用，执行正常返回 |
| `false` / `undefined` | 任意 | 不触发链式调用，执行正常返回 |

**注意**：`autoLinkUser` 必须**显式等于 `true`**，`undefined` 或 `false` 都不会触发链式调用

---

## 九、关键设计亮点与注意事项

### 9.1 设计亮点

1. **双模式授权**: Admin Consent 做租户级授权，link_user 做用户级绑定，职责分离清晰
2. **双 tenantId 设计**: Bot 租户 ID（配置）与用户租户 ID（回调）分离，支持多租户场景
3. **无状态 Token 管理**: 不存储 refresh token，通过 client_credentials 实时获取，降低复杂度
4. **链式授权**: `autoLinkUser` 参数允许一次点击完成两步授权，用户体验好
5. **安全 State 机制**: 签名 + 过期时间双重保障，防止 CSRF 和重放攻击
6. **优雅降级**: Bot Framework token 获取失败返回空，不阻塞整个发送流程
7. **缓存感知密钥轮换**: 缓存 Key 包含密钥哈希，密钥更新后缓存自动失效
8. **友好错误处理**: Bot 安装失败时返回 HTML 错误页，而非直接抛出异常

### 9.2 关键注意事项

1. **tenantId 来源混淆**: 必须区分 Bot 租户 ID（集成配置）和用户租户 ID（回调参数），两者用途不同
2. **autoLinkUser 严格相等**: 只有 `autoLinkUser === true` 才触发链式调用，注意是严格相等
3. **链式调用失败不中断**: autoLinkUser 失败只打日志，不影响主流程完成
4. **link_user 前置条件**: link_user 模式要求集成配置中已存在 tenantId，必须先完成 admin_consent
5. **返回分流优先级**: autoLinkUser 跳转 > redirectUrl > window.close()，注意提前 return 的情况

---

## 十、核心文件索引

| 文件 | 职责 |
|------|------|
| `integrations.controller.ts:735` | 回调入口 Controller |
| `chat-oauth-callback.usecase.ts` | 按 providerId 路由回调 |
| `generate-msteams-oauth-url.usecase.ts` | 生成授权 URL、State 编解码验证 |
| `msteams-oauth-callback.usecase.ts` | MS Teams 回调核心逻辑、返回分流、错误处理 |
| `create-channel-connection.usecase.ts` | 创建租户级通道连接 |
| `create-channel-endpoint.usecase.ts` | 创建用户级通道端点 |
| `resolve-channel-endpoints.usecase.ts` | 发送时解析端点、双 tenantId 组装 |
| `ms-teams-token.service.ts` | Graph/Bot Framework Token 服务 |
| `msTeams.provider.ts` | MS Teams 消息发送 Provider |
| `chat-oauth-state.util.ts` | State 编解码工具 |
