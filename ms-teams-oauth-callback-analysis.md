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
  identifier?: string;           // ⚠️ 通道连接标识 - 贯穿整个链路的关键
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
        const linkUserUrl = await this.generateMsTeamsOauthUrl.execute(
          GenerateMsTeamsOauthUrlCommand.create({
            environmentId: stateData.environmentId,
            organizationId: stateData.organizationId,
            connectionIdentifier: stateData.identifier,  // ⚠️ 传入相同的 identifier
            subscriberId: stateData.subscriberId,
            integration,
            context: stateData.context,
            mode: 'link_user',
          })
        );
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

## 四、Connection 标识一致性分析

### 4.1 标识生成与传递链路

**⚠️ 关键发现**：`connectionIdentifier` 在整个链路中保持一致，是关联 ChannelConnection 和 ChannelEndpoint 的核心纽带。

#### 前端生成逻辑

**文件**: `packages/js/src/ui/components/constants.ts`

```typescript
export const DEFAULT_MSTEAMS_CONNECTION_IDENTIFIER = 'chconn-msteams-default';

export function buildDefaultConnectionIdentifier(prefix: string, subscriberId: string | undefined): string {
  if (!subscriberId) {
    return prefix;  // 'chconn-msteams-default'
  }
  return `${prefix}-${subscriberId}`;  // 'chconn-msteams-default-{subscriberId}'
}
```

#### 完整传递链路

```
前端 MsTeamsConnectButton 组件
      │
      ├─ 生成 connectionIdentifier:
      │    buildDefaultConnectionIdentifier('chconn-msteams-default', subscriberId)
      │
      ▼
调用 generateConnectOAuthUrl API
      │
      ├─ 参数: connectionIdentifier = 'chconn-msteams-default-{subscriberId}'
      │
      ▼
GenerateConnectOauthUrl.execute()
      │
      ├─ 转发到 GenerateMsTeamsOauthUrl.execute(mode='connect')
      │
      ▼
GenerateMsTeamsOauthUrl.createSecureState()
      │
      ├─ stateData.identifier = connectionIdentifier  ⚠️ 编码到 state 中
      │
      ▼
返回 OAuth URL 给前端，前端打开弹窗
      │
      ▼
用户授权后回调 /v1/integrations/chat/oauth/callback
      │
      ▼
MsTeamsOauthCallback.execute(mode='connect')
      │
      ├─ 1. decodeMsTeamsState(state) → 取出 stateData.identifier
      ├─ 2. createAdminConsentConnection()
      │     └─ CreateChannelConnectionCommand.create({
      │          identifier: stateData.identifier  ⚠️ 保存到 ChannelConnection
      │        })
      │
      └─ 3. 如果 autoLinkUser === true:
            ├─ GenerateMsTeamsOauthUrl.execute(mode='link_user')
            │   └─ 参数: connectionIdentifier = stateData.identifier  ⚠️ 相同的 identifier
            │
            ▼
          生成 link_user OAuth URL 并跳转
            │
            ▼
          用户登录授权后回调
            │
            ▼
          MsTeamsOauthCallback.execute(mode='link_user')
            │
            ├─ 1. decodeMsTeamsState(state) → 取出 stateData.identifier
            └─ 2. linkUserEndpoint()
                 └─ CreateChannelEndpointCommand.create({
                      connectionIdentifier: stateData.identifier  ⚠️ 关联到相同的 Connection
                    })
```

### 4.2 一致性验证

| 阶段 | 位置 | 值 |
|------|------|-----|
| 前端生成 | `MsTeamsConnectButton.tsx:51-53` | `chconn-msteams-default-{subscriberId}` |
| State 编码 | `generate-msteams-oauth-url.usecase.ts:144` | `stateData.identifier = connectionIdentifier` |
| Connection 创建 | `msteams-oauth-callback.usecase.ts:140` | `identifier: stateData.identifier` |
| 链式调用传参 | `msteams-oauth-callback.usecase.ts:90` | `connectionIdentifier: stateData.identifier` |
| Endpoint 创建 | `msteams-oauth-callback.usecase.ts:176` | `connectionIdentifier: stateData.identifier` |

**结论**：整个链路中 `connectionIdentifier` 完全一致，确保了 ChannelEndpoint 能正确关联到 ChannelConnection。

### 4.3 自定义 connectionIdentifier 的约束

如果调用方传入自定义的 `connectionIdentifier`，必须满足：

1. **手动绑定时必须传入相同值**：使用 `MsTeamsLinkUser` 组件时，如果之前创建 Connection 时用了自定义 identifier，必须通过 `connectionIdentifier` prop 传入相同的值，否则会创建孤立的 ChannelEndpoint。

2. **唯一性约束**：同一环境下 identifier 必须唯一，否则会抛出 409 Conflict 错误。

3. **默认值回退**：如果未传入，前端会自动生成默认值，但此时必须保证 subscriberId 存在（subscriber 模式）或 context 存在（shared 模式）。

---

## 五、租户凭据持久化：Admin Consent 模式

### 5.1 处理流程

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

### 5.2 ChannelConnection 实体

**文件**: `apps/api/src/app/channel-connections/usecases/create-channel-connection/create-channel-connection.usecase.ts`

| 字段 | 说明 |
|------|------|
| `identifier` | 唯一标识，前端生成：`chconn-msteams-default-{subscriberId}` |
| `integrationIdentifier` | 关联的集成配置 |
| `subscriberId` | 关联的订阅者 (subscriber 模式必填，shared 模式为 null) |
| `workspace` | 存储 `{ id: tenantId }` - **用户租户 ID，来自 admin_consent 回调** |
| `auth` | 加密存储的认证信息，这里是 `{ accessToken: 'app-only' }` |
| `contextKeys` | 上下文标签数组 (shared 模式必填) |

**唯一性约束**: 同一集成 + 订阅者 + 上下文组合只能有一个连接

---

## 六、订阅者通道绑定：link_user 模式

### 6.1 处理流程

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

### 6.2 授权码交换与 oid 提取

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

### 6.3 Bot 自动安装

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

### 6.4 ChannelEndpoint 实体

**文件**: `apps/api/src/app/channel-endpoints/usecases/create-channel-endpoint/create-channel-endpoint.usecase.ts`

```typescript
await this.createChannelEndpoint.execute(
  CreateChannelEndpointCommand.create({
    organizationId: stateData.organizationId,
    environmentId: stateData.environmentId,
    integrationIdentifier: integration.identifier,
    connectionIdentifier: stateData.identifier, // ⚠️ 关联到 ChannelConnection
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

## 七、发送阶段 tenant 信息的来源区分

### 7.1 两个不同的 tenantId

**⚠️ 关键理解修正**：发送阶段存在两个不同来源的 tenantId，用途完全不同：

| 来源 | 存储位置 | 用途 | 说明 |
|------|----------|------|------|
| `subscriberTenantId` | `ChannelConnection.workspace.id` | Bot Framework API 调用时标识**用户所在租户** | 来自 admin_consent 回调参数 `tenant`，每个连接可能不同 |
| `tenantId` | `Integration.credentials.tenantId` | 获取 Bot Framework token 时标识**Bot 所在租户** | 集成配置时预先填写，所有连接共享 |

### 7.2 发送时的数据流

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

### 7.3 Provider 发送时的使用

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

### 7.4 数据模型关系图

```
Integration (MS Teams 配置)
      │
      ├─ credentials: { clientId, secretKey, tenantId }  ← Bot 租户 ID（配置时填写）
      │
      ▼
ChannelConnection (租户级连接)
      │
      ├─ identifier: 'chconn-msteams-default-{subscriberId}'  ⚠️ 前端生成
      ├─ integrationIdentifier: 关联 Integration
      ├─ subscriberId: 绑定到订阅者 (subscriber模式) / null (shared模式)
      ├─ workspace: { id: tenantId }  ← ⚠️ 用户租户 ID（来自 admin_consent 回调）
      └─ auth: { accessToken: 'app-only' }
      │
      ▼
ChannelEndpoint (用户级端点)
      │
      ├─ connectionIdentifier: 'chconn-msteams-default-{subscriberId}'  ⚠️ 关联 Connection
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

## 八、手动补绑入口：路由与前端调用映射

### 8.1 后端路由修正

**⚠️ 之前的理解偏差**：手动补绑的后端路由不是 `/chat/oauth/link-user-url`，而是 `/channel-endpoints/oauth`。

**文件**: `apps/api/src/app/integrations/integrations.controller.ts:705`

```typescript
@Post('/channel-endpoints/oauth')
@ApiResponse(GenerateChatOAuthUrlResponseDto, 201)
@ApiOperation({
  summary: 'Generate OAuth URL to link a subscriber user identity',
  description: `Generate an OAuth URL that links a specific subscriber to their chat identity (Slack user ID or MS Teams user OID). 
  The generated URL expires after 5 minutes.`,
})
@SdkMethodName('generateLinkUserOAuthUrl')
@RequirePermissions(PermissionsEnum.INTEGRATION_WRITE)
@ExternalApiAccessible()
@RequireAuthentication()
async generateLinkUserOAuthUrl(
  @UserSession() user: UserSessionData,
  @Body() body: GenerateLinkUserOauthUrlRequestDto
): Promise<GenerateChatOAuthUrlResponseDto> {
  const url = await this.generateLinkUserOauthUrlUsecase.execute(
    GenerateLinkUserOauthUrlCommand.create({
      environmentId: user.environmentId,
      organizationId: user.organizationId,
      subscriberId: body.subscriberId,
      integrationIdentifier: body.integrationIdentifier,
      connectionIdentifier: body.connectionIdentifier,  // ⚠️ 必须与之前的 connection 一致
      context: body.context,
      userScope: body.userScope,
    })
  );
  return { url };
}
```

### 8.2 前端调用链路

**文件**: `packages/js/src/channel-endpoints/helpers.ts:14`

```
前端 MsTeamsLinkUser 组件
      │
      ▼
useChannelEndpoint hook
      │
      ├─ generateLinkUserOAuthUrl()
      │
      ▼
channelEndpoints.generateLinkUserOAuthUrl(args)
      │
      ▼
apiService.generateLinkUserOAuthUrl(args)
      │
      ▼
POST /v1/inbox/channel-endpoints/oauth  ⚠️ 真实 API 路由
```

**前端 SDK 代码** (`packages/js/src/api/inbox-service.ts:596`):

```typescript
generateLinkUserOAuthUrl({
  integrationIdentifier,
  connectionIdentifier,
  subscriberId,
  context,
  userScope,
}: GenerateLinkUserOAuthUrlArgs): Promise<{ url: string }> {
  return this.#httpClient.post(CHANNEL_ENDPOINTS_OAUTH_ROUTE, {
    integrationIdentifier,
    connectionIdentifier,
    subscriberId,
    context,
    userScope,
  });
}
```

**常量定义** (`packages/js/src/api/inbox-service.ts:40-41`):
```typescript
const CHANNEL_ENDPOINTS_ROUTE = `${INBOX_ROUTE}/channel-endpoints`;
const CHANNEL_ENDPOINTS_OAUTH_ROUTE = `${CHANNEL_ENDPOINTS_ROUTE}/oauth`;
```

### 8.3 路由映射表

| 功能 | 后端 Controller 路由 | 前端 SDK 方法 | 说明 |
|------|---------------------|-------------|------|
| 租户级授权 URL | `POST /v1/integrations/channel-connections/oauth` | `channelConnections.generateConnectOAuthUrl()` | 生成 admin_consent URL |
| 用户级绑定 URL | `POST /v1/integrations/channel-endpoints/oauth` | `channelEndpoints.generateLinkUserOAuthUrl()` | 生成 link_user URL |
| 回调处理 | `GET /v1/integrations/chat/oauth/callback` | N/A | 统一处理所有 chat OAuth 回调 |

---

## 九、autoLinkUser 未触发或失败后的绑定闭环

### 9.1 触发条件与失败场景

**触发条件矩阵**:

| autoLinkUser | subscriberId | 行为 |
|--------------|--------------|------|
| `true` (显式) | 存在 | 链式调用 link_user，成功则跳转，失败则回退到正常返回 |
| `true` (显式) | 不存在 | 不触发链式调用，执行正常返回 |
| `false` / `undefined` | 任意 | 不触发链式调用，执行正常返回 |

**失败场景**（链式调用可能失败的原因）：
1. 集成配置中缺少 `tenantId`（link_user 模式的前置条件）
2. State 过期（5 分钟有效期）
3. 其他生成 URL 时的异常

### 9.2 前端补齐方案：MsTeamsLinkUser 组件

**文件**: `packages/js/src/ui/components/msteams-link-user/MsTeamsLinkUser.tsx`

当 autoLinkUser 未触发或失败时，前端可以使用独立的 `MsTeamsLinkUser` 组件手动完成用户绑定：

```
组件初始化
      │
      ├─ 1. 生成 connectionIdentifier（与 ConnectButton 相同的算法）
      │    buildDefaultConnectionIdentifier('chconn-msteams-default', subscriberId)
      │
      ├─ 2. 查询 channelEndpoints.list()
      │    过滤条件: integrationIdentifier + connectionIdentifier
      │    检查是否已有 type='ms_teams_user' 的端点
      │
      ▼
显示状态：已绑定 / 未绑定
      │
      ▼
用户点击 "Link Teams User"
      │
      ├─ 1. 调用 generateLinkUserOAuthUrl API
      │    参数: integrationIdentifier, connectionIdentifier, subscriberId
      │
      ├─ 2. 打开 OAuth 弹窗
      │
      └─ 3. 开始轮询 channelEndpoints.list()（2.5s 间隔，超时 2 分钟）
            │
            ├─ 找到 type='ms_teams_user' 的端点 → 绑定成功
            └─ 超时 → 提示错误
```

### 9.3 后端补齐 API

#### 单独的 link_user URL 生成接口

**文件**: `apps/api/src/app/integrations/integrations.controller.ts:705`

```typescript
@Post('/channel-endpoints/oauth')
@SdkMethodName('generateLinkUserOAuthUrl')
async generateLinkUserOAuthUrl(
  @UserSession() user: UserSessionData,
  @Body() body: GenerateLinkUserOauthUrlRequestDto
): Promise<GenerateChatOAuthUrlResponseDto> {
  // ⚠️ 关键: connectionIdentifier 必须与之前创建 ChannelConnection 时使用的标识完全一致
  // 否则创建的 ChannelEndpoint 无法关联到正确的 Connection
}
```

#### ChannelEndpoints 查询接口

**文件**: `apps/api/src/app/channel-endpoints/channel-endpoints.controller.ts:97`

```typescript
@Get()
@SdkMethodName('list')
async listChannelEndpoints(
  @UserSession() user: UserSessionData,
  @Query() query: ListChannelEndpointsQueryDto
): Promise<ListChannelEndpointsResponseDto> {
  // 支持按 integrationIdentifier + connectionIdentifier 过滤
  // 前端轮询时使用这两个参数定位特定连接下的端点
}
```

### 9.4 绑定闭环时序图

```
autoLinkUser 失败或未触发
      │
      ▼
ChannelConnection 已创建（有 tenantId），但 ChannelEndpoint 不存在
      │
      ▼
前端显示 "未绑定" 状态，展示 MsTeamsLinkUser 组件
      │
      ▼
用户点击 "Link Teams User"
      │
      ├─ 前端生成相同的 connectionIdentifier: 'chconn-msteams-default-xxx'
      ├─ 调用 generateLinkUserOAuthUrl API
      │   └─ POST /v1/integrations/channel-endpoints/oauth
      │      参数: connectionIdentifier='chconn-msteams-default-xxx'
      ├─ 打开 OAuth 弹窗
      └─ 开始轮询 channelEndpoints.list(integrationId, connectionId)
            │
            ▼
用户完成授权
      │
      ▼
回调创建 ChannelEndpoint（关联到已有 Connection）
      │
      ▼
前端轮询发现新的 ms_teams_user 端点
      │
      ▼
绑定完成，显示 "已绑定" 状态
```

---

## 十、subscriber 与 shared 两种模式的绑定差异与风险

### 10.1 connectionMode 验证规则

**文件**: `apps/api/src/app/channel-connections/usecases/channel-connection.utils.ts:16`

```typescript
export function validateConnectionMode({
  connectionMode,
  subscriberId,
  context,
}: {
  connectionMode?: ConnectionMode;
  subscriberId?: string;
  context?: ContextPayload;
}): void {
  if (connectionMode === 'shared') {
    if (!context) {
      throw new BadRequestException('context is required when connectionMode is "shared"');
    }
    return;
  }

  if (connectionMode === 'subscriber') {
    if (!subscriberId) {
      throw new BadRequestException('subscriberId is required when connectionMode is "subscriber"');
    }
    return;
  }

  if (!subscriberId && !context) {
    throw new BadRequestException('Either subscriberId or context must be provided');
  }
}
```

### 10.2 两种模式对比

| 维度 | subscriber 模式 | shared 模式 |
|------|---------------|------------|
| **必填字段** | `subscriberId` | `context` |
| **connectionIdentifier 生成** | `chconn-msteams-default-{subscriberId}` | `chconn-msteams-default` (无 subscriberId 时) |
| **ChannelConnection.subscriberId** | 存储 subscriberId | `null` |
| **ChannelConnection.contextKeys** | 可选 | 必填（存储 context key 列表）|
| **autoLinkUser 默认值** | `true`（前端组件默认） | `false` |
| **适用场景** | 单个用户独立绑定 | 多用户共享同一个租户连接 |

### 10.3 subscriber 模式详解

**特点**:
- 每个 subscriber 有独立的 ChannelConnection
- connectionIdentifier 包含 subscriberId，天然隔离
- autoLinkUser 默认开启，一次点击完成两步绑定
- ChannelEndpoint 直接绑定到 subscriberId

**数据流**:
```
Subscriber A → Connection A (subscriberId=A) → Endpoint A (userId=oid_A)
Subscriber B → Connection B (subscriberId=B) → Endpoint B (userId=oid_B)
```

**风险**:
- 每个 subscriber 都需要管理员授权一次（如果是不同租户）
- 管理员需要为每个用户单独授权，操作繁琐

### 10.4 shared 模式详解

**特点**:
- 多个 subscriber 共享同一个 ChannelConnection（同一个租户）
- connectionIdentifier 不包含 subscriberId（通常基于 context 生成）
- autoLinkUser 默认关闭，需要单独为每个用户执行 link_user
- 所有 ChannelEndpoint 关联到同一个 Connection

**数据流**:
```
Connection X (shared, context={team: 'sales'})
      ├─ Endpoint A (subscriberId=A, userId=oid_A)
      ├─ Endpoint B (subscriberId=B, userId=oid_B)
      └─ Endpoint C (subscriberId=C, userId=oid_C)
```

**风险**:
1. **autoLinkUser 不适用**: shared 模式下 autoLinkUser 默认为 false，即使设为 true 也无法正确绑定（因为 connection 的 subscriberId 为 null）
2. **必须手动绑定**: 每个用户都需要单独使用 MsTeamsLinkUser 组件完成绑定
3. **context 一致性**: 所有用户必须使用相同的 context 才能关联到同一个 shared connection
4. **connectionIdentifier 匹配**: 手动绑定时必须传入与创建 connection 时相同的 connectionIdentifier

### 10.5 shared 模式的绑定闭环

```
管理员创建 shared Connection (context={team: 'sales'})
      │
      ├─ connectionIdentifier = 'chconn-msteams-default-sales-team'
      └─ 存储 tenantId 到 workspace.id
      │
      ▼
用户 A 打开应用
      │
      ├─ 前端使用相同的 connectionIdentifier 查询 Endpoints
      ├─ 发现未绑定，显示 MsTeamsLinkUser 组件
      └─ 用户点击绑定 → 完成授权 → 创建 Endpoint A (subscriberId=A)
      │
      ▼
用户 B 打开应用
      │
      ├─ 前端使用相同的 connectionIdentifier 查询 Endpoints
      ├─ 发现未绑定，显示 MsTeamsLinkUser 组件
      └─ 用户点击绑定 → 完成授权 → 创建 Endpoint B (subscriberId=B)
      │
      ▼
所有用户共享同一个 Connection，但有各自的 Endpoint
```

### 10.6 模式选择建议

| 场景 | 推荐模式 | 理由 |
|------|---------|------|
| 单租户、用户较少 | subscriber 模式 | 简单，一次点击完成绑定 |
| 单租户、用户较多 | shared 模式 | 管理员只需授权一次 |
| 多租户 | subscriber 模式 | 每个租户单独授权 |
| 需要按团队/部门隔离 | shared 模式 | 每个团队一个 shared connection |

---

## 十一、未绑定风险分析

### 11.1 风险场景

| 场景 | 原因 | 后果 |
|------|------|------|
| autoLinkUser 链式调用失败 | 集成缺少 tenantId、state 过期等 | Connection 存在但无 Endpoint |
| autoLinkUser 设为 false | 业务需求只做租户级授权 | Connection 存在但无 Endpoint |
| 用户在 link_user 步骤关闭弹窗 | 用户主动中断 | Connection 存在但无 Endpoint |
| Bot 安装失败 | 权限不足、应用未发布等 | 抛出错误或返回错误页，无 Endpoint |
| 前端未正确实现轮询 | 未检测到绑定完成 | 前端状态显示异常 |
| shared 模式下未手动绑定 | 用户不知道需要第二步 | Connection 存在但用户无 Endpoint |
| connectionIdentifier 不匹配 | 手动绑定时传入错误的 identifier | 创建孤立的 Endpoint，无法关联 Connection |

### 11.2 风险影响

1. **消息发送失败**: 发送消息时找不到 ChannelEndpoint，导致投递失败
2. **数据不一致**: ChannelConnection 存在但无关联的 ChannelEndpoint，形成"僵尸连接"
3. **孤立 Endpoint**: connectionIdentifier 不匹配时，Endpoint 无法关联到 Connection，发送时缺少 tenantId
4. **用户体验差**: 用户可能不知道需要手动完成第二步绑定
5. **shared 模式扩散风险**: 一个 shared connection 下多个用户未绑定，影响范围大

### 11.3 缓解措施

1. **前端状态检测**: 组件初始化时查询 ChannelEndpoints，明确显示绑定状态
2. **引导手动绑定**: 未绑定时显示 MsTeamsLinkUser 组件，引导用户完成绑定
3. **connectionIdentifier 一致性保障**:
   - 推荐使用默认生成算法，避免自定义 identifier
   - 如必须自定义，确保在 ConnectButton 和 LinkUser 组件中传入相同值
4. **连接清理机制**: 定期清理无 Endpoint 的僵尸 Connection（可选）
5. **错误监控**: 监控 autoLinkUser 失败率和孤立 Endpoint 数量，及时发现配置问题
6. **文档说明**: 明确告知用户两种模式的绑定流程差异，避免误解
7. **shared 模式特殊处理**: 为 shared 模式提供批量绑定功能或管理员视角的绑定状态监控

---

## 十二、凭据服务：MsTeamsTokenService

**文件**: `libs/application-generic/src/services/ms-teams-token.service.ts`

### 12.1 两种 Token 类型

| 方法 | 用途 | 作用域 | 缓存 Key |
|------|------|--------|----------|
| `getGraphToken()` | Graph API 调用 (安装 Bot、查询应用等) | `https://graph.microsoft.com/.default` | `msteams:graph-token:{clientId}:{tenantId}:{secretHash}` |
| `getBotFrameworkToken()` | Bot Framework 发送消息 | `https://api.botframework.com/.default` | `msteams:bot-token:{clientId}:{tenantId}:{secretHash}` |

### 12.2 缓存设计

- 使用 `@CachedResponse` 装饰器，TTL 55 分钟 (1 小时 token 减 5 分钟缓冲)
- 缓存 Key 包含 `secretHash` (SHA-256 前 8 位)，密钥轮换后自动失效
- 获取失败时 `getBotFrameworkToken()` 返回空字符串，优雅降级

---

## 十三、完整时序与数据流

### 13.1 Admin Consent + autoLinkUser 完整流程

```
用户点击 SDK MsTeamsConnectButton (autoLinkUser=true, subscriberId='xxx')
      │
      ├─ 生成 connectionIdentifier: 'chconn-msteams-default-xxx'
      │
      ▼
调用 generateConnectOAuthUrl API
      │
      ▼
GenerateMsTeamsOauthUrl.execute(mode='connect', autoLinkUser=true, subscriberId='xxx')
      │
      ├─► 创建 StateData (含 identifier='chconn-msteams-default-xxx', subscriberId, autoLinkUser=true)
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
      │     └─► CreateChannelConnection(identifier='chconn-msteams-default-xxx', workspace={id: subscriberTenantId})
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
            │     ├─► 1. decodeMsTeamsState(state) → identifier='chconn-msteams-default-xxx'
            │     ├─► 2. getIntegration()
            │     ├─► 3. linkUserEndpoint()
            │     │     ├─► exchangeCodeForAadObjectId(code) → 获取 oid
            │     │     │     └─► 使用 Bot tenantId 调用 token 端点
            │     │     ├─► installBotForUser(oid) → 调用 Graph API 安装 Bot
            │     │     │     └─► 使用 Bot tenantId 获取 Graph token
            │     │     └─► createChannelEndpoint(connectionIdentifier='chconn-msteams-default-xxx')
            │     │
            │     └─► 4. 返回分流：redirectUrl 或 window.close()
            │
            └─ 失败：catch 住，打 warn 日志 → 继续执行返回分流
                  │
                  ▼
                返回分流：redirectUrl 或 window.close()
```

### 13.2 手动补齐绑定流程

```
autoLinkUser 失败 → Connection 存在但无 Endpoint
      │
      ▼
前端 MsTeamsLinkUser 组件显示 "未绑定"
      │
      ▼
用户点击 "Link Teams User"
      │
      ├─ 前端生成相同的 connectionIdentifier: 'chconn-msteams-default-xxx'
      ├─ 调用 generateLinkUserOAuthUrl API
      │   └─ POST /v1/integrations/channel-endpoints/oauth
      │      参数: connectionIdentifier='chconn-msteams-default-xxx'
      ├─ 打开 OAuth 弹窗
      └─ 开始轮询 channelEndpoints.list(integrationId, connectionId)
            │
            ▼
用户完成授权 → 回调创建 ChannelEndpoint
      │
      ▼
前端轮询发现 ms_teams_user 端点 → 绑定成功
```

---

## 十四、关键设计亮点与注意事项

### 14.1 设计亮点

1. **双模式授权**: Admin Consent 做租户级授权，link_user 做用户级绑定，职责分离清晰
2. **双 tenantId 设计**: Bot 租户 ID（配置）与用户租户 ID（回调）分离，支持多租户场景
3. **Connection 标识一致性**: 前端生成的 identifier 贯穿整个链路，确保关联正确
4. **无状态 Token 管理**: 不存储 refresh token，通过 client_credentials 实时获取，降低复杂度
5. **链式授权**: `autoLinkUser` 参数允许一次点击完成两步授权，用户体验好
6. **安全 State 机制**: 签名 + 过期时间双重保障，防止 CSRF 和重放攻击
7. **优雅降级**: Bot Framework token 获取失败返回空，不阻塞整个发送流程
8. **缓存感知密钥轮换**: 缓存 Key 包含密钥哈希，密钥更新后缓存自动失效
9. **友好错误处理**: Bot 安装失败时返回 HTML 错误页，而非直接抛出异常
10. **独立补齐机制**: MsTeamsLinkUser 组件支持手动绑定，形成完整闭环
11. **模式灵活**: 支持 subscriber 和 shared 两种模式，适应不同业务场景

### 14.2 关键注意事项

1. **tenantId 来源混淆**: 必须区分 Bot 租户 ID（集成配置）和用户租户 ID（回调参数），两者用途不同
2. **connectionIdentifier 一致性**: 手动绑定时必须使用与创建 Connection 时相同的 identifier
3. **路由修正**: 手动补绑的后端路由是 `/channel-endpoints/oauth`，不是 `/chat/oauth/link-user-url`
4. **autoLinkUser 严格相等**: 只有 `autoLinkUser === true` 才触发链式调用，注意是严格相等
5. **链式调用失败不中断**: autoLinkUser 失败只打日志，不影响主流程完成
6. **link_user 前置条件**: link_user 模式要求集成配置中已存在 tenantId，必须先完成 admin_consent
7. **返回分流优先级**: autoLinkUser 跳转 > redirectUrl > window.close()，注意提前 return 的情况
8. **未绑定风险**: autoLinkUser 失败后需要前端引导用户手动完成绑定，否则消息发送会失败
9. **shared 模式特殊处理**: shared 模式下 autoLinkUser 不适用，每个用户必须单独绑定
10. **connectionIdentifier 自定义风险**: 自定义 identifier 时必须确保在 ConnectButton 和 LinkUser 组件中一致

---

## 十五、核心文件索引

| 文件 | 职责 |
|------|------|
| `integrations.controller.ts:705` | generateLinkUserOAuthUrl 接口（手动绑定用，路由: `/channel-endpoints/oauth`） |
| `integrations.controller.ts:684` | generateConnectOAuthUrl 接口（路由: `/channel-connections/oauth`） |
| `integrations.controller.ts:735` | 回调入口 Controller（路由: `/chat/oauth/callback`） |
| `chat-oauth-callback.usecase.ts` | 按 providerId 路由回调 |
| `generate-msteams-oauth-url.usecase.ts` | 生成授权 URL、State 编解码验证 |
| `msteams-oauth-callback.usecase.ts` | MS Teams 回调核心逻辑、返回分流、错误处理 |
| `create-channel-connection.usecase.ts` | 创建租户级通道连接 |
| `create-channel-endpoint.usecase.ts` | 创建用户级通道端点 |
| `list-channel-endpoints.usecase.ts` | 查询通道端点列表（前端轮询用） |
| `resolve-channel-endpoints.usecase.ts` | 发送时解析端点、双 tenantId 组装 |
| `channel-connection.utils.ts` | connectionMode 验证规则 |
| `ms-teams-token.service.ts` | Graph/Bot Framework Token 服务 |
| `msTeams.provider.ts` | MS Teams 消息发送 Provider |
| `chat-oauth-state.util.ts` | State 编解码工具 |
| `MsTeamsConnectButton.tsx` | 前端连接按钮组件 |
| `MsTeamsLinkUser.tsx` | 前端手动绑定组件 |
| `constants.ts` | connectionIdentifier 生成逻辑 |
| `inbox-service.ts` | 前端 SDK API 封装 |
| `helpers.ts` (channel-endpoints) | 前端 channelEndpoints 方法实现 |
