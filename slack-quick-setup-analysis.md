# Slack 渠道快速接入流程代码分析

## 一、整体流程概览

Slack 渠道首次接入包含四个核心阶段，涉及前后端多个模块协同工作：

```
控制台点开 → 第三方授权 → 回调写入凭据 → 首条消息发出
     ↓            ↓             ↓                ↓
[前端引导]  [Slack OAuth]  [凭据持久化]  [运行时消息发送]
```

## 二、模块架构与关键文件

### 2.1 前端层 (Dashboard + React Package + JS Package)

Slack 连接按钮采用三层架构设计（React → SolidJS 桥接 → SolidJS 原生实现）：

| 模块层级 | 文件路径 | 核心职责 |
|----------|----------|----------|
| 安装引导界面 | `apps/dashboard/src/components/agents/slack-setup-guide.tsx` | 引导用户完成 Quick Setup 或 Manual Setup，触发欢迎消息 |
| React 按钮包装器 | `packages/react/src/components/slack-connect-button/SlackConnectButton.tsx` | React 对外导出组件，包裹 NovuUI Provider |
| React 桥接层 | `packages/react/src/components/slack-connect-button/DefaultSlackConnectButton.tsx` | 通过 `Mounter` 将 React Props 传递给 SolidJS 实现 |
| SolidJS 底层实现 | `packages/js/src/ui/components/slack-connect-button/SlackConnectButton.tsx` | 核心逻辑：生成 OAuth URL、打开弹窗、轮询连接状态 |
| SolidJS Hook | `packages/js/src/ui/api/hooks/useChannelConnection.ts` | 封装 channelConnections API，暴露 `generateConnectOAuthUrl` 方法 |
| ChannelConnections 模块 | `packages/js/src/channel-connections/channel-connections.ts` | SDK 层 channelConnections 模块类定义 |
| 核心 API 服务 | `packages/js/src/api/inbox-service.ts` | 定义 Inbox API 端点，发送实际 HTTP 请求 |
| 集成 API 客户端 | `apps/dashboard/src/api/integrations.ts` | `slackQuickSetup()` 调用后端快速设置接口 |
| Agent API 客户端 | `apps/dashboard/src/api/agents.ts` | `sendAgentWelcomeMessage()` 调用欢迎消息接口 |

### 2.2 后端 API 层

| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| Inbox 控制器 | `apps/api/src/app/inbox/inbox.controller.ts` | **核心**：暴露 `/inbox/channel-connections/oauth`（生成 OAuth URL）和 `/inbox/channel-connections/:identifier`（轮询）端点 |
| 集成控制器 | `apps/api/src/app/integrations/integrations.controller.ts` | 暴露 `POST /integrations/:integrationId/slack-quick-setup` 端点（后端 API 用） |
| Agent 控制器 | `apps/api/src/app/agents/agents.controller.ts` | 暴露 `POST /agents/:identifier/welcome-message` 端点 |
| Quick Setup UseCase | `apps/api/src/app/integrations/usecases/slack-quick-setup/slack-quick-setup.usecase.ts` | 调用 Slack API 创建 App 并保存凭据 |
| OAuth URL 生成 | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-connect-oauth-url/generate-connect-oauth-url.usecase.ts` | 生成带签名的 Slack OAuth 授权 URL（被 InboxController 调用） |
| OAuth 回调处理 | `apps/api/src/app/integrations/usecases/chat-oauth-callback/slack-oauth-callback/slack-oauth-callback.usecase.ts` | 处理 Slack 回调，创建 ChannelConnection 和 ChannelEndpoint |
| 欢迎消息发送 | `apps/api/src/app/agents/usecases/send-agent-welcome-message/send-agent-welcome-message.usecase.ts` | OAuth 成功后发送首条欢迎消息 |

### 2.3 消息提供者层

| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| Slack Provider | `packages/providers/src/lib/chat/slack/slack.provider.ts` | 底层 Slack API 调用（`chat.postMessage`） |
| Slack Handler | `libs/application-generic/src/factories/chat/handlers/slack.handler.ts` | Provider 工厂包装 |

---

## 三、三段核心调用关系详解

### 3.1 第一段：Quick Setup 调用链

**代码证据链：**

```
用户粘贴 Token 并点击 "Create app" 按钮
        ↓
[slack-setup-guide.tsx:247] QuickSetupStep 子组件 onClick 触发 mutation.mutate()
        ↓
[slack-setup-guide.tsx:196-204] useMutation 调用 slackQuickSetup()
        ↓
[apps/dashboard/src/api/integrations.ts:102-111] slackQuickSetup() 发送 POST 请求
        ↓
POST /v1/integrations/:integrationId/slack-quick-setup
        ↓
[integrations.controller.ts:808-812] @Post('/:integrationId/slack-quick-setup') 路由
        ↓
[slack-quick-setup.usecase.ts] SlackQuickSetup.execute()
        ↓
Slack API: POST https://slack.com/api/apps.manifest.create
        ↓
返回 credentials → encryptCredentials() → 更新 Integration.credentials
        ↓
前端 mutation onSuccess: setCredentialsSavedLocally(true)
```

**关键代码定位：**

`apps/dashboard/src/components/agents/slack-setup-guide.tsx:177-268` — `QuickSetupStep` 子组件：
```typescript
function QuickSetupStep({ integrationId, agentId, subscriberId, user, onSuccess }) {
  const mutation = useMutation({
    mutationFn: async () => {
      return slackQuickSetup(
        integrationId,
        { configToken: configToken.trim(), agentId, subscriberId, connectionIdentifier },
        environment
      );
    },
    onSuccess: () => {
      setConfigToken('');
      queryClient.invalidateQueries({ queryKey: [QueryKeys.fetchIntegrations, currentEnvironment?._id] });
      onSuccess();  // 调用父组件的 handleQuickSetupSuccess
    },
  });

  return (
    <Input
      // ...
      trailingNode={
        <button onClick={() => mutation.mutate()} disabled={!configToken.trim() || mutation.isPending}>
          {mutation.isPending ? 'Creating…' : 'Create app'}
        </button>
      }
    />
  );
}
```

`apps/api/src/app/integrations/integrations.controller.ts:808-812` — 路由定义：
```typescript
@Post('/:integrationId/slack-quick-setup')
@ApiResponse(SlackQuickSetupResponseDto, 201)
@ApiOperation({ summary: 'Quick-setup a Slack integration' })
async slackQuickSetup(...) {
  return await this.slackQuickSetup.execute(...);
}
```

---

### 3.2 第二段：OAuth 授权与轮询调用链

**⚠️ 关键修正：OAuth URL 生成和轮询都是通过 `/inbox/*` 端点，由 `InboxController` 处理，而非 `IntegrationsController`。**

**代码证据链：**

```
用户点击 "Install AgentName ↗" 按钮
        ↓
[slack-setup-guide.tsx:368-399] SlackConnectButton 被渲染（包裹在 NovuProvider 中）
        ↓
[packages/react/src/components/slack-connect-button/SlackConnectButton.tsx:30-32]
  React.memo 包装的 SlackConnectButton → SlackConnectButtonInternal
        ↓
[packages/react/src/components/slack-connect-button/SlackConnectButton.tsx:9-26]
  withRenderer → NovuUI → DefaultSlackConnectButton
        ↓
[packages/react/src/components/slack-connect-button/DefaultSlackConnectButton.tsx:23-82]
  Mounter 调用 novuUI.mountComponent({ name: 'SlackConnectButton', props, element })
        ↓
[packages/js/src/ui/components/slack-connect-button/SlackConnectButton.tsx:129-173]
  SolidJS handleClick() 被触发
        ↓
1. 调用 generateConnectOAuthUrl() 获取 OAuth URL
2. window.open(url, '_blank') 打开 Slack 授权弹窗
3. startPolling() 开始轮询
        ↓
[packages/js/src/ui/api/hooks/useChannelConnection.ts:36-38]
  generateConnectOAuthUrl = (args) => novuAccessor().channelConnections.generateConnectOAuthUrl(args)
        ↓
[packages/js/src/channel-connections/channel-connections.ts:45-53]
  ChannelConnections.generateConnectOAuthUrl() → 调用 helpers.generateConnectOAuthUrl()
        ↓
[packages/js/src/channel-connections/helpers.ts:36-56]
  generateConnectOAuthUrl() → 调用 apiService.generateConnectOAuthUrl(args)
        ↓
[packages/js/src/api/inbox-service.ts:576-594]
  InboxService.generateConnectOAuthUrl() → POST /inbox/channel-connections/oauth
        ↓
[apps/api/src/app/inbox/inbox.controller.ts:816-836]
  @Post('/channel-connections/oauth') → GenerateConnectOauthUrl.execute()
        ↓
返回 OAuth URL → window.open() 打开弹窗 → startPolling() 开始
        ↓
[packages/js/src/ui/components/slack-connect-button/SlackConnectButton.tsx:85-127]
  startPolling() — 每 2.5 秒调用 novuAccessor().channelConnections.get()
        ↓
[packages/js/src/api/inbox-service.ts:622-624]
  InboxService.getChannelConnection() → GET /inbox/channel-connections/:identifier
        ↓
[apps/api/src/app/inbox/inbox.controller.ts:700-709]
  @Get('/channel-connections/:identifier') → GetChannelConnection.execute()
        ↓
轮询命中 → mutate(response.data) → props.onConnectSuccess?.(connId)
        ↓
[slack-setup-guide.tsx:312-336] handleSlackOAuthSuccess 回调被触发
```

**关键代码定位：**

`apps/dashboard/src/components/agents/slack-setup-guide.tsx:368-399` — 按钮渲染：
```typescript
const slackInstallConnectControl =
  user?.externalId && currentEnvironment?.identifier ? (
    <NovuProvider subscriber={{...}} applicationIdentifier={currentEnvironment.identifier} {...}>
      <SlackConnectButton
        integrationIdentifier={selectedIntegrationIdentifier}
        connectionIdentifier={`${user.externalId}:agent-quickstart:${agent._id}`}
        connectionMode="subscriber"
        connectLabel={`Install ${agent.name} ↗`}
        connectedLabel="Connected to Slack"
        onConnectSuccess={handleSlackOAuthSuccess}  // 成功后触发欢迎消息
        // ...
      />
    </NovuProvider>
  ) : null;
```

`packages/js/src/api/inbox-service.ts:35-41` — 端点常量定义：
```typescript
const INBOX_ROUTE = '/inbox';
const CHANNEL_CONNECTIONS_ROUTE = `${INBOX_ROUTE}/channel-connections`;
const CHANNEL_CONNECTIONS_OAUTH_ROUTE = `${CHANNEL_CONNECTIONS_ROUTE}/oauth`;  // /inbox/channel-connections/oauth
```

`packages/js/src/api/inbox-service.ts:576-594` — OAuth URL 生成 API 调用：
```typescript
generateConnectOAuthUrl({
  integrationIdentifier,
  connectionIdentifier,
  subscriberId,
  context,
  scope,
  connectionMode,
  autoLinkUser,
}: GenerateConnectOAuthUrlArgs): Promise<{ url: string }> {
  return this.#httpClient.post(CHANNEL_CONNECTIONS_OAUTH_ROUTE, {  // POST /inbox/channel-connections/oauth
    integrationIdentifier,
    connectionIdentifier,
    subscriberId,
    context,
    scope,
    connectionMode,
    autoLinkUser,
  });
}
```

`packages/js/src/api/inbox-service.ts:622-624` — 轮询 API 调用：
```typescript
getChannelConnection(identifier: string): Promise<ChannelConnectionResponse> {
  return this.#httpClient.get(`${CHANNEL_CONNECTIONS_ROUTE}/${identifier}`);  // GET /inbox/channel-connections/:identifier
}
```

`apps/api/src/app/inbox/inbox.controller.ts:816-836` — 后端 OAuth URL 生成路由：
```typescript
@UseGuards(AuthGuard('subscriberJwt'))
@Post('/channel-connections/oauth')  // 注意：前缀是 /inbox，完整路径是 /inbox/channel-connections/oauth
async generateConnectOAuthUrl(
  @SubscriberSession() subscriberSession: SubscriberSession,
  @Body() body: GenerateConnectOauthUrlRequestDto
): Promise<GenerateChatOAuthUrlResponseDto> {
  const url = await this.generateConnectOauthUrlUsecase.execute(
    GenerateConnectOauthUrlCommand.create({
      environmentId: subscriberSession._environmentId,
      organizationId: subscriberSession._organizationId,
      subscriberId: subscriberSession.subscriberId,
      integrationIdentifier: body.integrationIdentifier,
      connectionIdentifier: body.connectionIdentifier,
      context: body.context,
      scope: body.scope,
      connectionMode: body.connectionMode,
      autoLinkUser: body.autoLinkUser,
    })
  );

  return { url };
}
```

`apps/api/src/app/inbox/inbox.controller.ts:700-709` — 后端轮询路由：
```typescript
@UseGuards(AuthGuard('subscriberJwt'))
@Get('/channel-connections/:identifier')  // GET /inbox/channel-connections/:identifier
async getChannelConnection(
  @SubscriberSession() subscriberSession: SubscriberSession,
  @Param('identifier') identifier: string
): Promise<InboxChannelConnectionResponseDto> {
  const channelConnection = await this.loadChannelConnectionForSubscriber(subscriberSession, identifier);

  return mapChannelConnectionToInboxDto(channelConnection);
}
```

`packages/js/src/ui/components/slack-connect-button/SlackConnectButton.tsx:85-127` — 轮询实现：
```typescript
const POLL_INTERVAL_MS = 2500;   // 2.5 秒轮询一次
const POLL_TIMEOUT_MS = 120_000; // 120 秒超时

const startPolling = () => {
  const connId = connectionIdentifier();
  const startedAt = Date.now();

  intervalIdRef.current = setInterval(async () => {
    try {
      const response = await novuAccessor().channelConnections.get({ identifier: connId });
      if (response.data) {
        clearInterval(intervalIdRef.current);
        setActionLoading(false);
        mutate(response.data);
        props.onConnectSuccess?.(connId);  // 触发成功回调
        return;
      }
    } catch {
      // ignore transient errors during polling
    }

    if (Date.now() - startedAt >= POLL_TIMEOUT_MS) {
      clearInterval(intervalIdRef.current);
      setActionLoading(false);
      props.onConnectError?.(new Error('Slack OAuth timed out. Please try again.'));
    }
  }, POLL_INTERVAL_MS);
};
```

**后端回调链路（与前端轮询并行）：**

```
用户在 Slack 弹窗中授权
        ↓
Slack 重定向到: /v1/integrations/chat/oauth/callback?code=xxx&state=xxx
        ↓
[integrations.controller.ts] 路由 → SlackOauthCallback.execute()
        ↓
1. decodeSlackState() 验证签名和时效性（5分钟超时）
2. exchangeCodeForAuthData() 调用 Slack oauth.v2.access
3. createChannelConnection.execute() 创建 ChannelConnection
4. autoLinkUser=true 时，createChannelEndpoint.execute() 创建 SLACK_USER Endpoint
5. 返回 <script>window.close();</script> 关闭弹窗
        ↓
ChannelConnection 已写入数据库 → 前端下一次轮询命中 GET /inbox/channel-connections/:identifier
```

---

### 3.3 第三段：欢迎消息发送调用链

**代码证据链：**

```
onConnectSuccess 回调触发 → handleSlackOAuthSuccess()
        ↓
[slack-setup-guide.tsx:312-336] 调用 sendAgentWelcomeMessage()
        ↓
[apps/dashboard/src/api/agents.ts:385-399] sendAgentWelcomeMessage() 发送 POST 请求
        ↓
POST /v1/agents/:identifier/welcome-message
        ↓
[agents.controller.ts:383-386] @Post('/:identifier/welcome-message') 路由
        ↓
[send-agent-welcome-message.usecase.ts] SendAgentWelcomeMessage.execute()
        ↓
1. 查询 Agent 和 Integration
2. 查询 ChannelEndpoint（SLACK_USER 类型）
3. chatSdkService.sendDirectMessage() → SlackProvider.sendMessage()
4. conversationService.createOrGetConversation() 创建会话记录
5. conversationService.persistAgentMessage() 保存消息记录
        ↓
返回 { sent: true, conversationId }
        ↓
前端设置 URL 参数: onboardingConversationId=xxx
```

**关键代码定位：**

`apps/dashboard/src/components/agents/slack-setup-guide.tsx:312-336` — 触发欢迎消息：
```typescript
const handleSlackOAuthSuccess = useCallback(() => {
  handleSlackWorkspaceConnected();  // 设置 isSlackWorkspaceConnected = true

  if (currentEnvironment && selectedIntegrationIdentifier) {
    sendAgentWelcomeMessage(currentEnvironment, agent.identifier, selectedIntegrationIdentifier)
      .then((res) => {
        if (res.conversationId) {
          setSearchParams((prev) => {
            prev.set('onboardingConversationId', res.conversationId as string);
            return prev;
          });
        }
      })
      .catch((err) => {
        console.warn('Failed to send agent welcome message after Slack OAuth:', err);
      });
  }
}, [handleSlackWorkspaceConnected, currentEnvironment, agent.identifier, selectedIntegrationIdentifier, setSearchParams]);
```

`apps/dashboard/src/api/agents.ts:385-399` — API 客户端：
```typescript
type WelcomeMessageResponse = { sent: boolean; conversationId?: string };

export async function sendAgentWelcomeMessage(
  environment: IEnvironment,
  agentIdentifier: string,
  integrationIdentifier: string,
  conversationId?: string
): Promise<WelcomeMessageResponse> {
  const response = await post<{ data: WelcomeMessageResponse }>(
    `/agents/${encodeURIComponent(agentIdentifier)}/welcome-message`,
    { environment, body: { integrationIdentifier, conversationId } }
  );

  return response.data;
}
```

`apps/api/src/app/agents/agents.controller.ts:383-386` — 路由定义：
```typescript
@Post('/:identifier/welcome-message')
@HttpCode(HttpStatus.OK)
@ApiOperation({ summary: 'Send onboarding welcome message' })
async sendWelcomeMessage(...) {
  return await this.sendAgentWelcomeMessage.execute(...);
}
```

---

## 四、前端界面引导流程

### 4.1 引导入口：`SlackSetupGuide` 组件

**文件**: `apps/dashboard/src/components/agents/slack-setup-guide.tsx`

#### 两种设置模式
组件通过 Feature Flag `IS_SLACK_QUICK_SETUP_ENABLED` 控制是否启用快速设置模式：

```typescript
const isQuickSetupEnabled = useFeatureFlag(FeatureFlagsKeysEnum.IS_SLACK_QUICK_SETUP_ENABLED, false);
const [setupMode, setSetupMode] = useState<SetupMode>('quick');
const activeSetupMode = isQuickSetupEnabled ? setupMode : 'manual';
```

#### Quick Setup 流程（3步）
1. **自动创建 Slack App** — 用户粘贴 App Configuration Token，调用 `slackQuickSetup` API
2. **安装 App 到工作区** — 点击 `SlackConnectButton` 触发 OAuth
3. **发送首条消息** — 引导用户在 Slack 中 @ 机器人

#### Manual Setup 流程（3步）
1. **通过 Manifest 创建 App** — 生成 pre-filled 的 Slack App Manifest YAML
2. **粘贴凭据** — 用户手动复制 App ID、Client ID、Client Secret、Signing Secret
3. **安装 App 到工作区** — 同上

### 4.2 Slack App Manifest 动态生成

`buildSlackManifestYaml()` 函数根据 Agent 信息动态生成 Slack App 配置：

```typescript
function buildSlackManifestYaml(agent: AgentResponse, webhookHandlerUrl: string, chatOAuthCallbackUrl: string): string {
  const botName = escapeYamlDoubleQuoted(sanitizeBotDisplayName(agent.name));
  // 自动配置:
  // - display_information: 使用 agent.name 作为 Bot 名称
  // - oauth_config.redirect_urls: 指向 /v1/integrations/chat/oauth/callback
  // - settings.event_subscriptions.request_url: 指向 Agent Webhook
  // - bot_events: 订阅 app_mention, message.im, assistant_thread_started 等
}
```

**关键点**:
- Bot 名称会自动过滤 "slack" 关键词（Slack API 限制）
- OAuth 回调地址和 Webhook 地址动态根据环境生成
- 事件订阅包含完整的 Agent 交互所需事件

---

## 五、OAuth 安全机制

### 5.1 安全 State 设计

`apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-connect-oauth-url/generate-connect-oauth-url.usecase.ts`

```typescript
private async createSecureState(...): Promise<string> {
  const stateData: StateData = {
    identifier: connectionIdentifier,
    subscriberId,
    context,
    environmentId: _environmentId,
    organizationId: _organizationId,
    integrationIdentifier: identifier,
    providerId: providerId as ChatProviderIdEnum,
    timestamp: Date.now(),
    mode,
    connectionMode,
    autoLinkUser,
  };

  const payload = JSON.stringify(stateData);
  const secret = await this.getEnvironmentApiKey(_environmentId);
  const signature = createHash(secret, payload);  // HMAC 签名
  
  return encodeOAuthState(payload, signature);  // Base64 编码
}
```

**State 验证**（回调时使用）:
```typescript
static async validateAndDecodeState(state: string, environmentApiKey: string): Promise<StateData> {
  const { payload, signature } = splitOAuthState(state);
  const expectedSignature = createHash(environmentApiKey, payload);
  
  if (signature !== expectedSignature) throw new Error('Invalid state signature');
  
  const data = JSON.parse(payload);
  if (Date.now() - data.timestamp > FIVE_MINUTES) throw new Error('OAuth state expired');
  
  return data;
}
```

**安全特性**:
- HMAC 签名防止篡改
- 5分钟超时防止重放攻击
- 包含完整上下文（environmentId, organizationId, integrationIdentifier）

### 5.2 Scope 动态解析

```typescript
private async resolveBotScopes(command: GenerateConnectOauthUrlCommand): Promise<string[] | undefined> {
  if (command.scope !== undefined) return command.scope;

  const isAgentLinked = await this.isIntegrationLinkedToAgent(command.integration);
  
  if (isAgentLinked) {
    return [...SLACK_AGENT_OAUTH_SCOPES];  // Agent 模式需要更多权限
  }
  
  return undefined;  // 使用默认 Scope
}
```

---

## 六、关键数据流转与存储

### 6.1 Integration 凭据结构

```javascript
// Integration.credentials (加密存储)
{
  clientId: "xxxx.xxxx",           // Slack App Client ID
  secretKey: "xxxxxx",             // Slack App Client Secret
  signingSecret: "xxxxxx",         // Slack App Signing Secret (验证 Webhook)
  applicationId: "A01XXXXXX"       // Slack App ID (可选)
}
```

### 6.2 ChannelConnection 结构

```javascript
// ChannelConnection (存储工作区连接)
{
  identifier: "user_123:agent-quickstart:agent_456",
  integrationIdentifier: "slack-prod",
  subscriberId: "user_123:agent-quickstart:agent_456",
  connectionMode: "subscriber",
  auth: {
    accessToken: "xoxb-xxxx-xxxx-xxxx"  // Slack Bot Token
  },
  workspace: {
    id: "T01XXXXXX",
    name: "My Workspace"
  }
}
```

### 6.3 ChannelEndpoint 结构

```javascript
// ChannelEndpoint (存储用户级端点)
{
  type: "slack-user",
  integrationIdentifier: "slack-prod",
  connectionIdentifier: "user_123:agent-quickstart:agent_456",
  subscriberId: "user_123:agent-quickstart:agent_456",
  endpoint: {
    userId: "U01XXXXXX"  // Slack 用户 ID，用于发送 DM
  }
}
```

---

## 七、完整时序图

```
用户操作                          前端 (Dashboard/SDK)                    后端 (API)                           Slack
   │                                  │                                        │                                │
   │ 打开 Agent 集成页面               │                                        │                                │
   ├─────────────────────────────────►│                                        │                                │
   │                                  │ 渲染 SlackSetupGuide                   │                                │
   │                                  │  (显示 Quick Setup / Manual)           │                                │
   │                                  │                                        │                                │
   │ 粘贴 App Config Token            │                                        │                                │
   ├─────────────────────────────────►│                                        │                                │
   │                                  │ QuickSetupStep onClick                 │                                │
   │                                  │ ── mutation.mutate() ──                │                                │
   │                                  │ ── slackQuickSetup() ──                │                                │
   │                                  │ POST /integrations/{id}/slack-quick-setup                              │
   │                                  ├────────────────────────────────────────►│                                │
   │                                  │                                        │ 验证 Integration               │
   │                                  │                                        │ 构建 Manifest                  │
   │                                  │                                        ├────────────────────────────────►│ apps.manifest.create
   │                                  │                                        │                                │
   │                                  │                                        │◄────────────────────────────────┤ 返回 credentials
   │                                  │                                        │ encryptCredentials()           │
   │                                  │                                        │ 更新 Integration               │
   │                                  │◄────────────────────────────────────────┤                                │
   │                                  │ setCredentialsSavedLocally(true)        │                                │
   │                                  │ 刷新 integrations 查询                  │                                │
   │                                  │ 显示 "App created!"                     │                                │
   │                                  │                                        │                                │
   │ 点击 "Install AgentName ↗"       │                                        │                                │
   ├─────────────────────────────────►│                                        │                                │
   │                                  │ SlackConnectButton.handleClick()        │                                │
   │                                  │ ── generateConnectOAuthUrl() ──         │                                │
   │                                  │   useChannelConnection hook             │                                │
   │                                  │   → ChannelConnections.generateConnectOAuthUrl                         │
   │                                  │   → InboxService.generateConnectOAuthUrl                               │
   │                                  │ POST /inbox/channel-connections/oauth                                  │
   │                                  ├────────────────────────────────────────►│                                │
   │                                  │                                        │ InboxController                │
   │                                  │                                        │ @Post('/channel-connections/oauth')                           │
   │                                  │                                        │ GenerateConnectOauthUrl.execute()                             │
   │                                  │                                        │ 生成带签名的 State             │
   │                                  │                                        │ 构建 OAuth URL                │
   │                                  │◄────────────────────────────────────────┤                                │
   │                                  │ window.open(Slack OAuth URL)           │                                │
   │                                  ├─────────────────────────────────────────┼────────────────────────────────►│
   │                                  │ startPolling()                          │                                │
   │                                  │ 每 2.5s GET /inbox/channel-connections/{id}                           │
   │                                  │                                        │                                │
   │                                  │              (用户在 Slack 弹窗中授权)  │                                │
   │                                  │                                        │◄────────────────────────────────┤ GET /chat/oauth/callback
   │                                  │                                        │ 解码验证 State                 │
   │                                  │                                        │ POST oauth.v2.access          │
   │                                  │                                        ├────────────────────────────────►│
   │                                  │                                        │                                │
   │                                  │                                        │◄────────────────────────────────┤ 返回 access_token
   │                                  │                                        │ 创建 ChannelConnection        │
   │                                  │                                        │ 创建 ChannelEndpoint (user)   │
   │                                  │                                        │ 返回 <script>close()</script> │
   │                                  │ 轮询命中 connection                     │                                │
   │                                  │   GET /inbox/channel-connections/{id}  │                                │
   │                                  │   → InboxController.getChannelConnection()                           │
   │                                  │ onConnectSuccess → handleSlackOAuthSuccess                              │
   │                                  │ POST /agents/{id}/welcome-message                                      │
   │                                  ├────────────────────────────────────────►│                                │
   │                                  │                                        │ 查询 ChannelEndpoint          │
   │                                  │                                        │ POST chat.postMessage         │
   │                                  │                                        ├────────────────────────────────►│
   │                                  │                                        │                                │
   │                                  │                                        │◄────────────────────────────────┤ 消息发送成功
   │                                  │                                        │ 创建 Conversation             │
   │                                  │                                        │ 持久化 Message                 │
   │                                  │◄────────────────────────────────────────┤                                │
   │                                  │ URL 增加 onboardingConversationId       │                                │
   │                                  │                                        │                                │
```

---

## 八、错误处理与边界情况

### 8.1 Quick Setup 错误处理
- **Invalid Token**: 提示用户 Token 无效或过期
- **Invalid Manifest**: Slack API 拒绝的 manifest 格式错误
- **Integration 不存在**: 404 错误

### 8.2 OAuth 错误处理
- **State 验证失败**: 签名不匹配或过期（5分钟超时）
- **Token 交换失败**: Slack 返回错误信息
- **凭据缺失**: Integration 缺少 clientId/clientSecret

### 8.3 欢迎消息发送失败
- **Endpoint 不存在**: 用户未完成链接（静默失败，返回 `sent: false`）
- **Slack API 错误**: 捕获异常并记录日志，不阻塞流程

---

## 九、设计亮点与可优化点

### 9.1 设计亮点
1. **前后端 Manifest 双生成** - 前端用于显示，后端用于实际创建，确保一致性
2. **安全 State 设计** - HMAC 签名 + 超时机制，防止 CSRF 和重放攻击
3. **轮询 + 回调双机制** - OAuth 回调写入数据，前端轮询感知状态变化
4. **自动用户链接** - OAuth 回调中自动创建 `SLACK_USER` Endpoint，减少用户操作
5. **三层按钮架构** - React → SolidJS 桥接 → SolidJS 原生，兼顾框架兼容性和性能
6. **Inbox 统一入口** - 所有前端 SDK 调用通过 `/inbox/*` 端点，使用 subscriberJwt 鉴权，权限隔离清晰
7. **Provider 抽象** - SlackProvider 与业务逻辑分离，便于替换和测试

### 9.2 潜在优化点
1. **轮询效率** - 当前 2.5s 轮询可考虑使用 WebSocket 推送替代
2. **错误重试** - 欢迎消息发送失败没有重试机制
3. **幂等性** - Quick Setup 和 OAuth 回调需要考虑重复调用的幂等处理
4. **Manifest 版本** - Slack Manifest API 可能有版本变更，需要监控兼容性

---

## 附录：关键端点对照表

| 操作 | 前端 SDK 端点 | 后端 Controller | 说明 |
|------|-------------|-----------------|------|
| 生成 OAuth URL | `POST /inbox/channel-connections/oauth` | InboxController | 前端 SDK 实际使用 |
| 轮询连接状态 | `GET /inbox/channel-connections/:identifier` | InboxController | 前端 SDK 实际使用 |
| 生成 OAuth URL (后端 API) | `POST /integrations/channel-connections/oauth` | IntegrationsController | 后端 API 专用，已标记旧端点为 deprecated |
| Slack OAuth 回调 | `GET /integrations/chat/oauth/callback` | IntegrationsController | Slack 重定向地址 |
| Quick Setup | `POST /integrations/:id/slack-quick-setup` | IntegrationsController | 创建 Slack App |
| 发送欢迎消息 | `POST /agents/:identifier/welcome-message` | AgentsController | OAuth 成功后触发 |
