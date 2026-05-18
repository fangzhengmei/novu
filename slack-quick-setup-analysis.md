# Slack 渠道快速接入流程代码分析

## 一、整体流程概览

Slack 渠道首次接入包含四个核心阶段，涉及前后端多个模块协同工作：

```
控制台点开 → 第三方授权 → 回调写入凭据 → 首条消息发出
     ↓            ↓             ↓                ↓
[前端引导]  [Slack OAuth]  [凭据持久化]  [运行时消息发送]
```

## 二、模块架构与关键文件

### 2.1 前端层 (Dashboard)
| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| 安装引导界面 | `apps/dashboard/src/components/agents/slack-setup-guide.tsx` | 引导用户完成 Quick Setup 或 Manual Setup |
| 连接按钮组件 | `packages/react/src/components/slack-connect-button/SlackConnectButton.tsx` | 触发 OAuth 流程，轮询连接状态 |
| 连接按钮实现 | `packages/js/src/ui/components/slack-connect-button/SlackConnectButton.tsx` | SolidJS 版本底层实现 |
| API 客户端 | `apps/dashboard/src/api/integrations.ts` | 调用 slackQuickSetup 等后端接口 |
| 欢迎消息 API | `apps/dashboard/src/api/agents.ts` | 调用 sendAgentWelcomeMessage |

### 2.2 后端 API 层
| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| 集成控制器 | `apps/api/src/app/integrations/integrations.controller.ts` | 暴露 /integrations/slack/quick-setup 等端点 |
| Quick Setup UseCase | `apps/api/src/app/integrations/usecases/slack-quick-setup/slack-quick-setup.usecase.ts` | 自动创建 Slack App 并保存凭据 |
| OAuth URL 生成 | `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-slack-oath-url/generate-slack-oauth-url.usecase.ts` | 生成带签名的 Slack OAuth 授权 URL |
| OAuth 回调处理 | `apps/api/src/app/integrations/usecases/chat-oauth-callback/slack-oauth-callback/slack-oauth-callback.usecase.ts` | 处理 Slack 回调，创建 Channel Connection |
| 欢迎消息发送 | `apps/api/src/app/agents/usecases/send-agent-welcome-message/send-agent-welcome-message.usecase.ts` | OAuth 成功后发送首条欢迎消息 |

### 2.3 消息提供者层
| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| Slack Provider | `packages/providers/src/lib/chat/slack/slack.provider.ts` | 底层 Slack API 调用（chat.postMessage） |
| Slack Handler | `libs/application-generic/src/factories/chat/handlers/slack.handler.ts` | Provider 工厂包装 |

## 三、前端界面引导流程详解

### 3.1 引导入口：`SlackSetupGuide` 组件

**文件**: `apps/dashboard/src/components/agents/slack-setup-guide.tsx`

#### 两种设置模式
组件通过 Feature Flag `IS_SLACK_QUICK_SETUP_ENABLED` 控制是否启用快速设置模式：

```typescript
const isQuickSetupEnabled = useFeatureFlag(FeatureFlagsKeysEnum.IS_SLACK_QUICK_SETUP_ENABLED, false);
const activeSetupMode = isQuickSetupEnabled ? setupMode : 'manual';
```

#### Quick Setup 流程（3步）
1. **自动创建 Slack App** - 用户粘贴 App Configuration Token，调用 `slackQuickSetup` API
2. **安装 App 到工作区** - 点击 `SlackConnectButton` 触发 OAuth
3. **发送首条消息** - 引导用户在 Slack 中 @ 机器人

#### Manual Setup 流程（3步）
1. **通过 Manifest 创建 App** - 生成 pre-filled 的 Slack App Manifest YAML
2. **粘贴凭据** - 用户手动复制 App ID、Client ID、Client Secret、Signing Secret
3. **安装 App 到工作区** - 同上

### 3.2 Slack App Manifest 动态生成

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

## 四、Quick Setup：自动创建 Slack App

### 4.1 前端调用

`QuickSetupStep` 子组件处理 Token 输入和 API 调用：

```typescript
// apps/dashboard/src/components/agents/slack-setup-guide.tsx:196-204
const mutation = useMutation({
  mutationFn: async () => {
    return slackQuickSetup(
      integrationId,
      { configToken: configToken.trim(), agentId, subscriberId, connectionIdentifier },
      environment
    );
  },
  onSuccess: () => {
    setCredentialsSavedLocally(true);
    queryClient.invalidateQueries({ queryKey: [QueryKeys.fetchIntegrations, currentEnvironment?._id] });
  }
});
```

### 4.2 后端实现：`SlackQuickSetup` UseCase

**文件**: `apps/api/src/app/integrations/usecases/slack-quick-setup/slack-quick-setup.usecase.ts`

#### 执行流程
1. **查询 Integration** - 验证 Integration 存在且为 Slack 类型
2. **构建 Manifest** - 调用 `buildManifest()` 生成 Slack App 配置
3. **调用 Slack API** - POST 到 `https://slack.com/api/apps.manifest.create`
4. **保存凭据** - 加密后写入 Integration 文档

#### 核心代码分析

```typescript
async execute(command: SlackQuickSetupCommand): Promise<SlackQuickSetupResult> {
  // 1. 验证 Integration
  const integration = await this.integrationRepository.findOne({...});
  
  // 2. 构建 Manifest（与前端逻辑一致，前后端双保险）
  const manifest = this.buildManifest(integration.name ?? 'Novu Bot', integration.identifier, command.agentId);
  
  // 3. 调用 Slack API 创建 App
  const slackResponse = await this.callManifestCreate(command.configToken, manifest);
  
  // 4. 加密并保存凭据
  const { client_id, client_secret, signing_secret } = slackResponse.credentials;
  await this.saveCredentials(command, client_id, client_secret, signing_secret, slackResponse.app_id);
  
  return {};
}
```

#### 凭据保存逻辑

```typescript
private async saveCredentials(
  command: SlackQuickSetupCommand,
  clientId: string,
  clientSecret: string,
  signingSecret: string,
  applicationId?: string
): Promise<void> {
  const credentials = encryptCredentials({
    clientId,
    secretKey: clientSecret,
    signingSecret,
    ...(applicationId && { applicationId }),
  });

  await this.integrationRepository.update(
    { _id: command.integrationId, _environmentId: command.environmentId, _organizationId: command.organizationId },
    {
      $set: {
        credentials,
        active: true,  // 自动激活 Integration
      },
    }
  );
}
```

**数据流向**:
```
Slack API (apps.manifest.create) 
    → client_id, client_secret, signing_secret, app_id
        → encryptCredentials() 
            → Integration.credentials (加密存储)
```

## 五、OAuth 授权流程

### 5.1 前端触发：`SlackConnectButton` 组件

**文件**: `packages/js/src/ui/components/slack-connect-button/SlackConnectButton.tsx`

#### 点击处理流程
```typescript
const handleClick = async () => {
  setActionLoading(true);
  
  // 1. 请求后端生成 OAuth URL
  const result = await generateConnectOAuthUrl({
    integrationIdentifier: integrationIdentifier(),
    connectionIdentifier: connectionIdentifier(),
    subscriberId: resolvedSubscriberId,
    context: ctx,
    scope: props.scope,
    connectionMode: mode,
    autoLinkUser: mode === 'subscriber' ? (props.autoLinkUser ?? true) : false,
  });

  if (result.data?.url) {
    // 2. 打开新窗口进行 Slack 授权
    window.open(result.data.url, '_blank', 'noopener,noreferrer');
    // 3. 开始轮询连接状态
    startPolling();
  }
};
```

#### 连接状态轮询

```typescript
const startPolling = () => {
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
    } catch { /* 忽略 transient errors */ }

    if (Date.now() - startedAt >= POLL_TIMEOUT_MS) {  // 120秒超时
      clearInterval(intervalIdRef.current);
      props.onConnectError?.(new Error('Slack OAuth timed out.'));
    }
  }, POLL_INTERVAL_MS);  // 2.5秒轮询一次
};
```

### 5.2 OAuth URL 生成：`GenerateSlackOauthUrl` UseCase

**文件**: `apps/api/src/app/integrations/usecases/generate-chat-oath-url/generate-slack-oath-url/generate-slack-oauth-url.usecase.ts`

#### 核心逻辑
1. **验证资源** - 检查 Subscriber 是否存在
2. **获取凭据** - 从 Integration 或 Novu 托管 Provider 获取 clientId
3. **创建安全 State** - 包含上下文信息并用 API Key 签名
4. **确定 Scope** - Agent 模式使用 `SLACK_AGENT_OAUTH_SCOPES`，普通模式使用默认 Scope
5. **构建 URL** - 生成最终的 Slack OAuth 授权 URL

#### 安全 State 设计

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

#### Scope 动态解析

```typescript
private async resolveBotScopes(command: GenerateSlackOauthUrlCommand): Promise<string[] | undefined> {
  if (command.scope !== undefined) return command.scope;

  const isAgentLinked = await this.isIntegrationLinkedToAgent(command.integration);
  
  if (isAgentLinked) {
    return [...SLACK_AGENT_OAUTH_SCOPES];  // Agent 模式需要更多权限
  }
  
  return undefined;  // 使用默认 Scope
}
```

### 5.3 OAuth 回调处理：`SlackOauthCallback` UseCase

**文件**: `apps/api/src/app/integrations/usecases/chat-oauth-callback/slack-oauth-callback/slack-oauth-callback.usecase.ts`

#### 执行流程
1. **解码 State** - 验证签名和时效性
2. **查询 Integration** - 根据 State 中的信息定位
3. **交换 Code** - 使用 OAuth code 换取 access_token
4. **创建连接** - 根据模式创建 Channel Connection 或 Endpoint

#### 核心代码

```typescript
async execute(command: SlackOauthCallbackCommand): Promise<ChatOauthCallbackResult> {
  // 1. 解码并验证 State
  const stateData = await this.decodeSlackState(command.state);
  
  // 2. 查询 Integration
  const integration = await this.getIntegration(stateData);
  const credentials = await this.getIntegrationCredentials(integration);
  
  // 3. 交换授权 Code 为 Access Token
  const authData = await this.exchangeCodeForAuthData(command.providerCode, credentials);

  // 4. 根据模式创建不同类型的连接
  if (stateData.mode === 'link_user') {
    await this.linkUserEndpoint(stateData, integration, authData);
  } else if (authData.incoming_webhook) {
    await this.createIncomingWebhookEndpoint(stateData, integration, authData);
  } else {
    // 创建 Workspace Connection（Agent 模式）
    const connection = await this.createChannelConnection.execute(
      CreateChannelConnectionCommand.create({
        identifier: stateData.identifier,
        organizationId: stateData.organizationId,
        environmentId: stateData.environmentId,
        integrationIdentifier: integration.identifier,
        subscriberId: isSharedMode ? undefined : stateData.subscriberId,
        connectionMode: stateData.connectionMode,
        auth: { accessToken: authData.access_token },
        workspace: { id: authData.team.id, name: authData.team.name },
      })
    );
    
    // 自动链接当前用户为 Slack User Endpoint
    if (stateData.autoLinkUser === true && stateData.subscriberId && authData.authed_user?.id) {
      await this.createChannelEndpoint.execute(
        CreateChannelEndpointCommand.create({
          type: ENDPOINT_TYPES.SLACK_USER,
          endpoint: { userId: authData.authed_user.id },
          // ... 其他字段
        })
      );
    }
  }

  return { type: ResponseTypeEnum.HTML, result: '<script>window.close();</script>' };
}
```

**数据流向**:
```
Slack OAuth 回调 (code, state)
    → 解码 State → 验证签名
        → oauth.v2.access (换取 access_token)
            → 创建 ChannelConnection (access_token 存入)
                → 可选: 创建 ChannelEndpoint (userId)
                    → 返回关闭窗口脚本
```

## 六、首条消息发送流程

### 6.1 触发时机

OAuth 成功后，前端在 `handleSlackOAuthSuccess` 回调中触发欢迎消息：

```typescript
// apps/dashboard/src/components/agents/slack-setup-guide.tsx:312-336
const handleSlackOAuthSuccess = useCallback(() => {
  handleSlackWorkspaceConnected();
  
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
      .catch((err) => console.warn('Failed to send agent welcome message:', err));
  }
}, [...]);
```

### 6.2 后端实现：`SendAgentWelcomeMessage` UseCase

**文件**: `apps/api/src/app/agents/usecases/send-agent-welcome-message/send-agent-welcome-message.usecase.ts`

#### 执行流程
1. **查询 Agent** - 根据 identifier 定位
2. **查询 Integration** - 验证集成存在
3. **查询 Endpoint** - 查找用户的 Slack User Endpoint
4. **发送消息** - 通过 Chat SDK 发送 DM
5. **创建会话** - 保存 Conversation 和 Message 记录

#### 核心代码

```typescript
private async sendWelcomeMessage(command: SendAgentWelcomeMessageCommand): Promise<{ sent: boolean; conversationId?: string }> {
  // 1. 查询 Agent 和 Integration
  const agent = await this.agentRepository.findOne({ identifier: command.agentIdentifier, ... });
  const integration = await this.integrationRepository.findOne({ identifier: command.integrationIdentifier, ... });
  
  // 2. 查找用户的 Slack Endpoint
  const platform = resolveAgentPlatform(integration.providerId);  // SLACK
  const endpointConfig = PLATFORM_ENDPOINT_CONFIG[platform];
  const endpoint = await this.channelEndpointRepository.findOne({
    integrationIdentifier: command.integrationIdentifier,
    type: endpointConfig.endpointType,  // SLACK_USER
  });
  
  const platformUserId = (endpoint.endpoint as Record<string, string>)[endpointConfig.identityField];  // userId
  
  // 3. 发送欢迎消息
  const welcomeText = getWelcomeText(platform);  // "Your Slack app is connected!..."
  const sent = await this.chatSdkService.sendDirectMessage(
    agent._id,
    command.integrationIdentifier,
    platformUserId,
    { markdown: welcomeText }
  );
  
  // 4. 创建 Conversation 记录
  const conversation = await this.conversationService.createOrGetConversation({
    agentId: agent._id,
    platform,
    integrationId: integration._id,
    platformThreadId: sent.platformThreadId,
    participantId: `${platform}:${platformUserId}`,
    platformUserId,
    firstMessageText: welcomeText,
  });
  
  // 5. 持久化 Agent 消息
  await this.conversationService.persistAgentMessage({
    conversationId: conversation._id,
    platformMessageId: sent.messageId,
    agentIdentifier: command.agentIdentifier,
    content: welcomeText,
    ...
  });
  
  return { sent: true, conversationId: conversation._id };
}
```

### 6.3 底层消息发送：`SlackProvider`

**文件**: `packages/providers/src/lib/chat/slack/slack.provider.ts`

```typescript
async sendMessage(data: IChatOptions, ...): Promise<ISendMessageSuccessResponse> {
  const response = await this.sendMessageToEndpoint(data, data.channelData, ...);
  
  return {
    id: response.headers['x-slack-req-id'] || `webhook-id-${Date.now()}`,
    date: new Date().toISOString(),
  };
}

private async sendAppMessageToUser(data: IChatOptions, channelData: SlackUserData, ...) {
  const { endpoint, token } = channelData;
  
  return await this.axiosInstance.post(
    `${this.slackAPI}/chat.postMessage`,
    {
      text: data.content,
      blocks: data.blocks,
      channel: endpoint.userId,  // 直接发送到用户 ID（DM）
      ...(data.customData || {}),
    },
    {
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,  // 使用 ChannelConnection 中的 accessToken
      },
    }
  );
}
```

## 七、完整时序图

```
用户操作                          前端 (Dashboard)                     后端 (API)                         Slack
   │                                  │                                   │                                │
   │ 打开 Agent 集成页面               │                                   │                                │
   ├─────────────────────────────────►│                                   │                                │
   │                                  │ 渲染 SlackSetupGuide              │                                │
   │                                  │  (显示 Quick Setup / Manual)      │                                │
   │                                  │                                   │                                │
   │ 粘贴 App Config Token            │                                   │                                │
   ├─────────────────────────────────►│                                   │                                │
   │                                  │ POST /integrations/{id}/slack/quick-setup                        │
   │                                  ├──────────────────────────────────►│                                │
   │                                  │                                   │ 验证 Integration                │
   │                                  │                                   │ 构建 Manifest                   │
   │                                  │                                   ├───────────────────────────────►│ apps.manifest.create
   │                                  │                                   │                                │
   │                                  │                                   │◄───────────────────────────────┤ 返回 credentials
   │                                  │                                   │ 加密保存 credentials            │
   │                                  │◄──────────────────────────────────┤                                │
   │                                  │ 显示 "App created!"               │                                │
   │                                  │                                   │                                │
   │ 点击 "Install App" 按钮           │                                   │                                │
   ├─────────────────────────────────►│                                   │                                │
   │                                  │ POST /integrations/chat/oauth/url                                │
   │                                  ├──────────────────────────────────►│                                │
   │                                  │                                   │ 生成带签名的 State              │
   │                                  │                                   │ 构建 OAuth URL                 │
   │                                  │◄──────────────────────────────────┤                                │
   │                                  │ window.open(Slack OAuth URL)      │                                │
   │                                  ├───────────────────────────────────┼───────────────────────────────►│
   │                                  │ 开始轮询 channelConnections        │                                │
   │                                  │  (每 2.5s 一次，超时 120s)        │                                │
   │                                  │                                   │                                │
   │                                  │              (用户在 Slack 授权)  │                                │
   │                                  │                                   │◄───────────────────────────────┤ OAuth Callback (code, state)
   │                                  │                                   │ 解码验证 State                  │
   │                                  │                                   │ oauth.v2.access               │
   │                                  │                                   ├───────────────────────────────►│
   │                                  │                                   │                                │
   │                                  │                                   │◄───────────────────────────────┤ 返回 access_token
   │                                  │                                   │ 创建 ChannelConnection        │
   │                                  │                                   │ 创建 ChannelEndpoint (user)    │
   │                                  │                                   │ 返回 <script>close()</script> │
   │                                  │ 轮询命中 connection                │                                │
   │                                  │ 触发 onConnectSuccess             │                                │
   │                                  │ POST /agents/{id}/welcome-message │                                │
   │                                  ├──────────────────────────────────►│                                │
   │                                  │                                   │ 查询 ChannelEndpoint           │
   │                                  │                                   │ chat.postMessage               │
   │                                  │                                   ├───────────────────────────────►│
   │                                  │                                   │                                │
   │                                  │                                   │◄───────────────────────────────┤ 消息发送成功
   │                                  │                                   │ 创建 Conversation              │
   │                                  │                                   │ 持久化 Message                  │
   │                                  │◄──────────────────────────────────┤                                │
   │                                  │ 显示欢迎消息已发送                 │                                │
   │                                  │                                   │                                │
```

## 八、关键数据流转与存储

### 8.1 Integration 凭据结构

```javascript
// Integration.credentials (加密存储)
{
  clientId: "xxxx.xxxx",           // Slack App Client ID
  secretKey: "xxxxxx",             // Slack App Client Secret
  signingSecret: "xxxxxx",         // Slack App Signing Secret (验证 Webhook)
  applicationId: "A01XXXXXX"       // Slack App ID (可选)
}
```

### 8.2 ChannelConnection 结构

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

### 8.3 ChannelEndpoint 结构

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

## 九、错误处理与边界情况

### 9.1 Quick Setup 错误处理
- **Invalid Token**: 提示用户 Token 无效或过期
- **Invalid Manifest**: Slack API 拒绝的 manifest 格式错误
- **Integration 不存在**: 404 错误

### 9.2 OAuth 错误处理
- **State 验证失败**: 签名不匹配或过期（5分钟超时）
- **Token 交换失败**: Slack 返回错误信息
- **凭据缺失**: Integration 缺少 clientId/clientSecret

### 9.3 欢迎消息发送失败
- **Endpoint 不存在**: 用户未完成链接（静默失败，返回 sent: false）
- **Slack API 错误**: 捕获异常并记录日志，不阻塞流程

## 十、设计亮点与可优化点

### 10.1 设计亮点
1. **前后端 Manifest 双生成** - 前端用于显示，后端用于实际创建，确保一致性
2. **安全 State 设计** - HMAC 签名 + 超时机制，防止 CSRF 和重放攻击
3. **轮询 + 回调双机制** - OAuth 回调写入数据，前端轮询感知状态变化
4. **自动用户链接** - OAuth 回调中自动创建 SLACK_USER Endpoint，减少用户操作
5. **Provider 抽象** - SlackProvider 与业务逻辑分离，便于替换和测试

### 10.2 潜在优化点
1. **轮询效率** - 当前 2.5s 轮询可考虑使用 WebSocket 推送替代
2. **错误重试** - 欢迎消息发送失败没有重试机制
3. **幂等性** - Quick Setup 和 OAuth 回调需要考虑重复调用的幂等处理
4. **Manifest 版本** - Slack Manifest API 可能有版本变更，需要监控兼容性
