# Bridge Framework 远程节点调用链路分析

## 总体架构

Novu Bridge Framework 的远程调用是云端（Novu API）与用户自托管服务（Bridge SDK）之间的双向通信机制。核心场景有两类：

1. **Workflow 模式**：Novu API → Bridge SDK（执行工作流步骤）
2. **Agent 模式**：Novu API → Bridge SDK → Novu API（事件投递 → 用户代码处理 → 回复投递）

```
┌─────────────┐         HTTP POST          ┌──────────────────┐
│  Novu API   │ ──────────────────────────→ │  Bridge SDK      │
│  (云端)      │   novu-signature Header    │  (用户服务端)     │
│             │ ←────────────────────────── │                  │
│             │         JSON Response       │  用户代码执行     │
└──────┬──────┘                             └────────┬─────────┘
       │                                              │
       │          HTTP POST (replyUrl)                │
       │←─────────────────────────────────────────────┘
       │  Authorization: ApiKey <secretKey>
       └──────────────────────────────────────────────→
```

---

## 1. 握手（Health Check & Discovery）

Bridge SDK 不存在传统意义上的 TLS 握手，而是通过 **Health Check + Discovery** 实现服务就绪验证和元数据同步。

### 1.1 Health Check（GET 请求，action=health-check）

当 Novu API 向 Bridge URL 发送 GET 请求且 `action=health-check` 时，框架跳过 HMAC 验证，直接返回服务状态。

代码位置：[NovuRequestHandler.handleAction()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L142-L193)

```ts
if (action !== GetActionEnum.HEALTH_CHECK) {
  await this.validateHmac(body, signatureHeader);
}
```

Health Check 响应结构（[HealthCheck 类型](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/types/health-check.types.ts#L1-L9)）：

```ts
type HealthCheck = {
  status: 'ok' | 'error';
  sdkVersion: string;
  frameworkVersion: string;
  discovered: { workflows: number; steps: number; };
};
```

### 1.2 Discovery（GET 请求，action=discover）

Discovery 在 Health Check 通过后触发，Bridge SDK 将注册的 Workflow 和 Agent 定义（含 JSON Schema）序列化返回给云端。这一步实现了**类型同步**（详见第 7 节）。

代码位置：[NovuRequestHandler.getGetActionMap()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L273-L291)

---

## 2. HMAC 签名

HMAC 签名是 Bridge 通信的核心安全机制，确保请求来源可信且内容未被篡改。

### 2.1 签名生成（API 端 → SDK 端）

签名的生成在 `@novu/application-generic` 的 [buildNovuSignatureHeader()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/hmac.ts#L6-L12) 中完成：

```ts
export function buildNovuSignatureHeader(secretKey: string, payload: unknown): string {
  const timestamp = Date.now();
  const publicKey = `${timestamp}.${JSON.stringify(payload)}`;
  const hmac = createHmac('sha256', secretKey).update(publicKey).digest('hex');
  return `t=${timestamp},v1=${hmac}`;
}
```

**签名算法**：
- 算法：HMAC-SHA256
- 签名输入：`<unix毫秒时间戳>.<JSON.stringify(payload)>`
- 输出格式：`t=<timestamp>,v1=<hex-hmac>`
- 密钥：Environment 级别的 `secretKey`（通过 `GetDecryptedSecretKey` 获取）

### 2.2 签名生成（Agent Bridge 场景）

在 Agent Bridge 场景中，[BridgeExecutorService.fireWithRetries()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L165-L231) 使用同样的 `buildNovuSignatureHeader()` 生成签名，并通过 `novu-signature` Header 发送：

```ts
const signatureHeader = buildNovuSignatureHeader(secretKey, payload);
// ...
headers: {
  'content-type': 'application/json',
  [HttpHeaderKeysEnum.NOVU_SIGNATURE]: signatureHeader,
},
```

关键常量：[HttpHeaderKeysEnum.NOVU_SIGNATURE = 'novu-signature'](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/constants/http-headers.constants.ts#L2)

### 2.3 签名验证（SDK 端）

SDK 端的签名验证在 [NovuRequestHandler.validateHmac()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L371-L396) 中实现：

```ts
private async validateHmac(payload: unknown, hmacHeader: string | null): Promise<void> {
  if (!this.hmacEnabled) return;              // 开发模式下可关闭
  if (!hmacHeader) throw new SignatureNotFoundError();
  if (!this.client.secretKey) throw new SigningKeyNotFoundError();

  const parsed = parseSignatureHeader(hmacHeader);
  if (!parsed.v1 || parsed.t === undefined) throw new SignatureInvalidError();

  const now = Date.now();
  if (parsed.t < now - SIGNATURE_TIMESTAMP_TOLERANCE || parsed.t > now + SIGNATURE_TIMESTAMP_TOLERANCE) {
    throw new SignatureExpiredError();
  }

  const localHash = await createHmacSubtle(this.client.secretKey, `${parsed.t}.${JSON.stringify(payload)}`);
  if (!timingSafeEqual(localHash, parsed.v1)) throw new SignatureMismatchError();
}
```

### 2.4 HMAC 的密码学实现

SDK 端使用 Web Crypto API（[createHmacSubtle()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/utils/crypto.utils.ts#L11-L32)），兼容浏览器、Node.js 和 Edge Runtime：

```ts
export const createHmacSubtle = async (secretKey: string, data: string): Promise<string> => {
  const encoder = new TextEncoder();
  const keyData = encoder.encode(secretKey);
  const dataBuffer = encoder.encode(data);
  const cryptoKey = await crypto.subtle.importKey('raw', keyData, { name: 'HMAC', hash: { name: 'SHA-256' } }, false, ['sign']);
  const signature = await crypto.subtle.sign('HMAC', cryptoKey, dataBuffer);
  return Array.from(new Uint8Array(signature)).map((byte) => byte.toString(16).padStart(2, '0')).join('');
};
```

API 端使用 Node.js 原生 `crypto.createHmac()`，因为 API 只运行在 Node.js 环境。

### 2.5 时序安全比较

为防止时序攻击，[timingSafeEqual()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/utils/crypto.utils.ts#L50-L60) 使用纯 JS 实现，执行时间仅取决于字符串长度，不取决于差异位置：

```ts
export const timingSafeEqual = (a: string, b: string): boolean => {
  if (typeof a !== 'string' || typeof b !== 'string') return false;
  if (a.length !== b.length) return false;
  let mismatch = 0;
  for (let i = 0; i < a.length; i += 1) { mismatch |= a.charCodeAt(i) ^ b.charCodeAt(i); }
  return mismatch === 0;
};
```

### 2.6 strictAuthentication 开关

HMAC 验证可通过 [Client 构造选项](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/client.ts#L87-L110) 的 `strictAuthentication` 控制：

- 默认值：**非开发环境为 `true`**，开发环境为 `false`
- 环境变量覆盖：`NOVU_STRICT_AUTHENTICATION_ENABLED=true|false`

---

## 3. 防重放

Bridge 框架的防重放机制是一个**单一机制**——即 HMAC 签名中嵌入的时间戳容差检查。`deliveryId` 不是框架层面的防重放手段，它只是请求追踪标识。

### 3.1 唯一的防重放机制：时间戳容差

**机制原理**：签名头中包含请求生成时的 Unix 毫秒时间戳 `t`，接收端验证该时间戳与当前时间的差值在容差范围内。

代码位置：[validateHmac()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L371-L396)

```ts
if (parsed.t < now - SIGNATURE_TIMESTAMP_TOLERANCE || parsed.t > now + SIGNATURE_TIMESTAMP_TOLERANCE) {
  throw new SignatureExpiredError();
}
```

[SIGNATURE_TIMESTAMP_TOLERANCE = ±5 分钟](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/constants/api.constants.ts#L6-L7)

**能防什么**：
- ✅ **过期请求重放**：攻击者截获请求后保存，数小时/数天后再重放 —— 时间戳已过期，直接被拒绝
- ✅ **离线截获的请求重放**：中间人截获后长期保存用于后续攻击 —— 同样被时间窗口拦截
- ✅ **大范围时间漂移攻击**：攻击者尝试用非当前时间戳伪造请求 —— HMAC 签名保护了时间戳的完整性，无法篡改

**不能防什么**：
- ❌ **5 分钟窗口内的重放攻击**：攻击者在签名生成后的 5 分钟内，将同一请求重复发送 N 次 —— 时间戳仍然有效，签名验证通过，框架层面无任何去重逻辑
- ❌ **同一会话内的重复执行**：因网络抖动导致的重试（Novu 端的重试机制）会造成用户代码被执行多次 —— 这是设计上的取舍，重试被视为合法的"至少一次"投递语义
- ❌ **时钟漂移过大的场景**：两端时钟差超过 5 分钟时，合法请求也会被拒绝

**关键设计前提**：HMAC 签名覆盖了时间戳（签名输入是 `<timestamp>.<payload>`），因此攻击者无法在不持有 secretKey 的情况下修改 `t` 来绕过时间检查。

### 3.2 deliveryId 的真实定位

`deliveryId` **不是**框架层面的防重放/去重机制。它只是一个请求追踪标识，被放入 payload 中透传。

生成代码：[buildPayload()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L262-L305)

```ts
deliveryId: `${conversation._id}:${message.id}`                    // 有消息时
deliveryId: `${conversation._id}:${event}:${action.id}:${timestamp}`  // 有 action 时
deliveryId: `${conversation._id}:${event}:${reaction.messageId}:${timestamp}`  // 有 reaction 时
deliveryId: `${conversation._id}:${event}`                          // 其他情况
```

**接收侧行为**：
- SDK 端的 [AgentContextImpl](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/resources/agent/agent.context.ts) 甚至没有将 `deliveryId` 暴露为公共属性
- 框架层面没有任何基于 `deliveryId` 的去重缓存、数据库索引或幂等判断
- 它只是 `AgentBridgeRequest` 类型定义中的一个字段，随 payload 一起透传给用户代码

**deliveryId 的实际用途**：
- ✅ **日志/链路追踪**：用户代码可以通过 `deliveryId` 关联同一次投递的多条日志
- ✅ **业务层幂等（用户自行实现）**：如果用户需要精确一次语义，需要自己基于 `deliveryId` 在业务层做去重（如 Redis SETNX、数据库唯一索引等）
- ✅ **问题排查**：出现重复投递时，可通过 `deliveryId` 确认是否为同一请求的多次到达

**deliveryId 不能做什么**：
- ❌ 不能替代时间戳容差做防重放（框架不校验）
- ❌ 不能保证"恰好一次"语义
- ❌ 不能防止 5 分钟窗口内的重复执行

### 3.3 签名头解析与历史漏洞

[parseSignatureHeader()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L415-L439) 从 `novu-signature` 头中解析 `t`（时间戳）和 `v1`（签名值）：

```ts
function parseSignatureHeader(header: string): ParsedSignatureHeader {
  const fields: Record<string, string> = {};
  for (const rawPart of header.split(',')) {
    const part = rawPart.trim();
    if (!part) continue;
    const eqIdx = part.indexOf('=');   // 只在第一个 '=' 处分割
    if (eqIdx <= 0) continue;
    const key = part.slice(0, eqIdx);
    const value = part.slice(eqIdx + 1);
    if (key && value && !(key in fields)) { fields[key] = value; }
  }
  const tRaw = fields.t;
  const t = tRaw !== undefined ? Number(tRaw) : NaN;
  return { t: Number.isFinite(t) ? t : undefined, v1: fields.v1 };
}
```

**历史安全漏洞**：旧实现使用 `split('=')` 来分割键值对，导致当 value 中包含 `=` 字符时，`timestamp` 会被错误地赋值为字面量字符串 `"t"`。由于 `"t" < now - tolerance` 恒为真，时间戳验证**静默失效**，相当于完全禁用了防重放保护。

当前实现使用 `indexOf('=')` 只在第一个 `=` 处分割，并按名称查找字段，确保了时间戳解析的正确性。

### 3.4 防重放能力总结

| 攻击场景 | 时间戳容差 | deliveryId |
|----------|-----------|------------|
| 截获请求后长期保存再重放 | ✅ 阻止 | ❌ 不阻止 |
| 5 分钟内重复发送同一请求 | ❌ 不阻止 | ❌ 不阻止（框架无校验） |
| 修改时间戳绕过时间窗口 | ✅ 阻止（HMAC 保护完整性） | — |
| 网络重试导致重复执行 | ❌ 不阻止（设计为至少一次） | ❌ 不阻止 |
| 用户代码自行实现幂等 | — | ✅ 可作为幂等键 |

---

## 4. 超时

超时机制分为两层：

### 4.1 Workflow Bridge 超时

[ExecuteFrameworkRequest.execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts#L100) 根据 URL 是否为内部服务选择不同超时：

```ts
const timeout = bridgeUrl?.includes(process.env.API_INTERNAL_ORIGIN) ? 60_000 : DEFAULT_TIMEOUT;
```

- 内部请求（Novu Cloud 内部）：60 秒
- 外部请求（用户 Bridge）：5 秒（[DEFAULT_TIMEOUT = 5_000](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/services/http-client/http-client.types.ts#L106)）

### 4.2 Agent Bridge 超时

Agent Bridge 使用 `safeOutboundJsonRequest`，其默认超时为 30 秒：

[DEFAULT_TIMEOUT_MS = 30_000](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L209)

### 4.3 SSRF 安全出站请求中的超时实现

在 [performPinnedRequest()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L367-L453) 中，使用 Node.js HTTP Agent 的 `timeout` 选项，并在 `timeout` 事件中销毁请求，将错误标记为 `ETIMEDOUT`：

```ts
req.on('timeout', () => {
  const timeoutError = new Error(`Request to ${parsed.hostname} timed out after ${timeoutMs}ms.`);
  timeoutError.code = 'ETIMEDOUT';
  req.destroy(timeoutError);
});
```

### 4.4 Agent Reply 的超时

Agent 回复通过标准 `fetch()` API 发送到 `replyUrl`，没有显式超时控制，依赖运行时默认行为。

---

## 5. 重试

重试机制存在两条路径：

### 5.1 Workflow Bridge 重试（HttpClientService）

[HttpClientService.request()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/services/http-client/http-client.service.ts#L82-L151) 通过 `got` 库或自实现的 SSRF 安全路径进行重试：

- **默认重试次数**：[DEFAULT_RETRIES_LIMIT = 3](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/services/http-client/http-client.types.ts#L107)
- **可重试的 HTTP 状态码**：408, 429, 500, 503, 504, 521, 522, 524
- **可重试的网络错误码**：EAI_AGAIN, ECONNREFUSED, ECONNRESET, EADDRINUSE, EPIPE, ETIMEDOUT, ENOTFOUND, EHOSTUNREACH, ENETUNREACH
- **退避策略**：指数退避 `delay = 2^attempt × RETRY_BASE_INTERVAL_IN_MS`（测试环境 50ms，生产环境 500ms）
- **SSRF 拦截不重试**：`SsrfBlockedError` 是确定性的，重试不会改变结果

### 5.2 Agent Bridge 重试（BridgeExecutorService）

[BridgeExecutorService.fireWithRetries()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L165-L231) 实现了独立的重试逻辑：

- **最大重试次数**：[MAX_RETRIES = 2](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L30)（共 3 次尝试）
- **退避策略**：指数退避 `delay = RETRY_BASE_DELAY_MS × 2^attempt`（基础延迟 500ms → 500ms, 1000ms, 2000ms）
- **每次重试都重新验证 SSRF 和重新计算 HMAC**（签名中包含时间戳，必须重新生成）
- **失败回调**：所有重试耗尽后，触发 `onBridgeFailure` 回调

```ts
for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
  assertSafeOutboundUrl(url);                    // 每次重试都检查 URL 安全
  await resolvePublicAddresses(new URL(url).hostname);  // DNS 解析检查
  const signatureHeader = buildNovuSignatureHeader(secretKey, payload);  // 重新签名
  // ... 发送请求
  if (attempt < MAX_RETRIES) {
    await this.delay(RETRY_BASE_DELAY_MS * 2 ** attempt);  // 指数退避
  }
}
```

### 5.3 重试安全性设计

- **SSRF 先于签名**：URL 合法性检查和 DNS 解析在 HMAC 计算之前执行，确保被阻止的 URL 不会收到签名过的载荷
- **跨域重定向脱敏**：[stripSensitiveHeaders()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L283-L298) 在重定向跨越原始域名时，自动移除 `Authorization`、`Cookie`、`novu-signature` 等敏感头部

---

## 6. 类型同步

### 6.1 Workflow Discovery

Workflow 和 Agent 定义在 Bridge SDK 端注册时，通过 `discover()` 函数收集完整的类型信息：

[Client.discover()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/client.ts#L208-L213) 返回所有注册的 Workflow 和 Agent 的元数据：

```ts
public discover(): DiscoverOutput {
  return {
    workflows: this.getRegisteredWorkflows(),
    agents: Array.from(this.registeredAgents.keys()).map((id) => ({ agentId: id })),
  };
}
```

每个 Workflow 的 Discovery 输出包含（[DiscoverWorkflowOutput](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/types/discover.types.ts#L49-L71)）：
- `workflowId` / `name` / `description` / `tags` / `severity`
- `payload.schema` / `controls.schema` / `env.schema` — JSON Schema
- `steps[]` — 每个步骤的 `controls.schema`、`outputs.schema`、`results.schema`

### 6.2 Schema 转换

用户提供的 Zod/JSON Schema 在 Discovery 阶段通过 [transformSchema()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/validators/index.ts) 统一转换为标准 JSON Schema，使得云端可以在不依赖特定验证库的情况下理解数据结构。

### 6.3 Agent 类型同步

Agent 的类型同步较为轻量，Discovery 仅返回 `agentId`。完整的 `AgentBridgeRequest` 类型（[agent.types.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/resources/agent/agent.types.ts#L383-L400)）是静态的，两端通过 SDK 版本约定保持一致：

```ts
interface AgentBridgeRequest {
  version: number;
  timestamp: string;
  deliveryId: string;
  event: string;
  agentId: string;
  replyUrl: string;
  conversationId: string;
  integrationIdentifier: string;
  action: AgentAction | null;
  message: AgentMessage | null;
  reaction: AgentReaction | null;
  conversation: AgentConversation;
  subscriber: AgentSubscriber | null;
  history: AgentHistoryEntry[];
  platform: string;
  platformContext: AgentPlatformContext;
}
```

### 6.4 版本标识

每个请求/响应都携带版本信息用于兼容性判断：
- Header `novu-framework-version`：框架协议版本
- Header `novu-framework-sdk`：SDK 版本
- `AgentBridgeRequest.version`：Bridge 协议版本（当前为 `1`）

---

## 7. 用户代码执行边界

### 7.1 Workflow 执行边界

Workflow 的用户代码通过 [Client.executeWorkflow()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/client.ts#L430-L548) 调用。执行模型是"逐步骤执行"：

1. Novu API 按步骤逐个调用 Bridge SDK（每步一次 HTTP 请求，`action=execute` 或 `action=preview`）
2. SDK 通过 `stepId` 确定当前执行哪一步
3. 当前步骤之前的步骤从 `event.state` 中恢复（hydrated），不重新执行
4. 当前步骤之后的步骤不执行（`hasResult()` 为 true 时提前退出）
5. 用户代码抛出的错误被包装为 `StepExecutionFailedError` 或 `ProviderExecutionFailedError`

### 7.2 Agent 执行边界

Agent 的用户代码在 [NovuRequestHandler.runAgentHandler()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L319-L348) 中执行：

```ts
private async runAgentHandler(registeredAgent: Agent, event: string, ctx: AgentContextImpl): Promise<void> {
  const replyIfPresent = async (result: MessageContent | void) => {
    if (result != null) await ctx.reply(result);
  };

  switch (event) {
    case AgentEventEnum.ON_MESSAGE:
      await replyIfPresent(await registeredAgent.handlers.onMessage(ctx.message!, ctx as AgentMessageContext));
      break;
    case AgentEventEnum.ON_ACTION:
      if (registeredAgent.handlers.onAction) {
        await replyIfPresent(await registeredAgent.handlers.onAction(ctx.action!, ctx as AgentActionContext));
      }
      break;
    // ... ON_REACTION, ON_RESOLVE
  }

  await ctx.flush();  // 刷新所有未发送的 signals/reactions
}
```

**关键边界设计**：

1. **异步投递模型**：Agent 事件采用 "fire-and-forget + waitUntil" 模式。Bridge SDK 立即返回 `{ status: 'ack' }`，用户代码在后台执行：
   ```ts
   const handlerPromise = this.runAgentHandler(registeredAgent, agentEvent, ctx).catch(...);
   if (waitUntil) { waitUntil(handlerPromise); }
   return this.createResponse(HttpStatusEnum.OK, { status: 'ack' });
   ```
   这意味着框架层不等待用户代码完成即可响应云端，适用于 Serverless 环境（Vercel Edge、Cloudflare Workers）的 `waitUntil` 语义。

2. **错误隔离**：用户代码中的错误被 `catch` 捕获并记录日志，不会导致 Bridge 请求本身失败（因为已经返回了 ack）：
   ```ts
   .catch((err) => {
     if (err instanceof AgentDeliveryError) {
       this.client.logger.error(`[agent:${agentId}] ${err.message}`);
     } else {
       this.client.logger.error(`[agent:${agentId}] Handler error:`, err);
     }
   });
   ```

3. **Reply 边界**：用户代码通过 `ctx.reply()` 将回复 POST 到 `replyUrl`。这是一个**带认证的独立 HTTP 调用**：
   ```ts
   private async _post(body: AgentReplyPayload): Promise<SentMessageInfo | null> {
     const response = await fetch(this._replyUrl, {
       method: 'POST',
       headers: {
         'Content-Type': 'application/json',
         Authorization: `ApiKey ${this._secretKey}`,
       },
       body: JSON.stringify(body),
     });
     if (!response.ok) throw new AgentDeliveryError(response.status, text);
   }
   ```
   回复端点 [AgentReplyController](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/reply/agent-reply.controller.ts#L14-L41) 使用 `@RequireAuthentication()` + `@ExternalApiAccessible()` 守卫。

4. **输出验证**：Workflow 步骤的用户代码返回值会经过 Schema 验证（[Client.validate()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/client.ts#L234-L262)），不符合 Schema 的输出会抛出 `ExecutionStateOutputInvalidError`。

5. **XSS 防护**：Email 和 In-App 类型的步骤输出经过 [sanitizeHtmlInObject()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/client.ts#L390-L417) 清洗，除非用户显式设置 `disableOutputSanitization: true`。

---

## 8. SSRF 防护

SSRF 防护贯穿整个远程调用链路，是 Bridge 安全的关键补充层。

### 8.1 URL 静态检查

[assertSafeOutboundUrl()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L100-L127) 在请求发出前执行：
- 仅允许 `http:` 和 `https:` 协议
- 拒绝包含嵌入凭证的 URL（`user:pass@host`）
- 拒绝 `localhost`、`metadata.google.internal` 等受阻止的主机名

### 8.2 DNS 解析 + 私有 IP 拦截

[resolvePublicAddresses()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L129-L171) 对目标主机进行 DNS 解析，检查所有解析到的 IP 地址是否为私有/保留地址：
- 使用 LRU 缓存（容量 500，TTL 5 分钟）避免重复 DNS 查询
- 拦截 RFC 1918、链路本地、环回等私有地址

### 8.3 DNS Pinning

[safeOutboundRequest()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L455-L536) 将 TCP 连接直接绑定到验证过的 IP 地址（而非主机名），保留原始 `Host` 头和 SNI servername，防止 DNS rebinding 攻击：

```ts
const requestOptions = {
  hostname: address.address,    // 直接使用 IP
  family: address.family,
  servername: parsed.hostname,  // TLS SNI 仍使用原始域名
};
```

### 8.4 重定向策略

- 最大重定向次数：3 次
- 每个重定向目标重新执行 URL 验证和 DNS 检查
- **跨域重定向脱敏**：敏感 Header（Authorization、Cookie、novu-signature）在跨越原始域名时自动移除
- **307/308 跨域拒绝**：方法保持的重定向跨域时直接抛出 `CROSS_ORIGIN_METHOD_PRESERVING_REDIRECT` 错误

### 8.5 Agent Bridge 场景的 SSRF

[BridgeExecutorService.fireWithRetries()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L173-L185) 在**每次重试**前都执行 SSRF 检查：

```ts
for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
  assertSafeOutboundUrl(url);
  await resolvePublicAddresses(new URL(url).hostname);
  const signatureHeader = buildNovuSignatureHeader(secretKey, payload);  // 签名在 SSRF 检查之后
  // ...
}
```

---

## 9. 完整调用链路图

### 9.1 Workflow 执行链路

```
Novu Worker
  └─→ ExecuteFrameworkRequest.execute()
       ├─ getBridgeUrl()          解析 Bridge URL
       ├─ buildRequestSignature() 生成 HMAC 签名
       │    └─ GetDecryptedSecretKey → buildNovuSignatureHeader()
       ├─ HttpClientService.request()
       │    ├─ enforceSsrfProtection?
       │    │    ├─ YES: safeOutboundRequest() (DNS pin + 私有 IP 拦截 + 重定向策略)
       │    │    └─ NO:  got (Node.js HTTP 客户端)
       │    └─ retry (指数退避, 默认3次)
       └─→ Bridge SDK (Express/Nest/Next/...)
            ├─ NovuRequestHandler.handleAction()
            │    ├─ 解析 URL query params (action, workflowId, stepId)
            │    ├─ validateHmac()
            │    │    ├─ parseSignatureHeader()
            │    │    ├─ 时间戳容差检查 (±5min)
            │    │    ├─ createHmacSubtle() 本地计算
            │    │    └─ timingSafeEqual() 比较签名
            │    └─ handlePostAction()
            │         └─ Client.executeWorkflow()
            │              ├─ createExecutionPayload()  验证 payload
            │              ├─ workflow.execute()         用户代码执行
            │              │    └─ executeStepFactory()  按步骤执行
            │              │         ├─ compileControls()    Liquid 模板渲染
            │              │         ├─ step.resolve()       用户步骤函数
            │              │         ├─ validate()           输出 Schema 验证
            │              │         └─ executeProviders()   Provider 执行
            │              └─ 返回 ExecuteOutput
            └─ HTTP Response (JSON)
```

### 9.2 Agent 事件投递链路

```
Novu API (BridgeExecutorService)
  ├─ resolveBridgeUrl()     解析 Bridge URL + query params
  ├─ buildPayload()         构建 AgentBridgeRequest
  │    ├─ resolveAgentReplyApiOrigin()  计算 replyUrl
  │    └─ deliveryId 生成 + timestamp
  └─ fireWithRetries()      发送 + 重试
       ├─ assertSafeOutboundUrl()     SSRF URL 检查
       ├─ resolvePublicAddresses()    DNS + 私有 IP 拦截
       ├─ buildNovuSignatureHeader()  HMAC 签名
       └─ safeOutboundJsonRequest()   DNS Pin + 超时30s
            └─→ Bridge SDK
                 ├─ validateHmac()           HMAC 验证
                 ├─ PostActionEnum.AGENT_EVENT
                 │    ├─ AgentContextImpl(body, secretKey)  构建上下文
                 │    ├─ runAgentHandler()                  用户代码执行
                 │    │    ├─ onMessage / onAction / ...
                 │    │    ├─ ctx.reply()  → fetch(replyUrl) → AgentReplyController
                 │    │    └─ ctx.flush()  → 发送剩余 signals/reactions
                 │    └─ 返回 { status: 'ack' }
                 └─ HTTP 200 (立即返回)

Agent Reply (独立链路):
  Bridge SDK (AgentContextImpl._post())
    └─→ fetch(replyUrl)
         ├─ Authorization: ApiKey <secretKey>
         └─→ Novu API /v1/agents/:agentId/reply
              ├─ @RequireAuthentication()
              ├─ HandleAgentReply.execute()
              └─ 投递消息到聊天平台
```

---

## 10. 关键文件索引

| 关注点 | 文件 | 关键行 |
|--------|------|--------|
| HMAC 签名生成 | [hmac.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/hmac.ts#L6-L12) | L6-L12 |
| HMAC 签名验证 | [handler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L371-L396) | L371-L396 |
| 签名头解析 | [handler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L415-L439) | L415-L439 |
| Web Crypto HMAC | [crypto.utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/utils/crypto.utils.ts#L11-L32) | L11-L32 |
| 时序安全比较 | [crypto.utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/utils/crypto.utils.ts#L50-L60) | L50-L60 |
| 时间戳容差常量 | [api.constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/constants/api.constants.ts#L6-L7) | L6-L7 |
| Agent Bridge 投递 | [bridge-executor.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L115-L163) | L115-L163 |
| Agent 重试逻辑 | [bridge-executor.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/runtime/bridge-executor.service.ts#L165-L231) | L165-L231 |
| SSRF URL 检查 | [ssrf-url-validation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L100-L127) | L100-L127 |
| DNS 解析 + 私有 IP | [ssrf-url-validation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L129-L171) | L129-L171 |
| 安全出站请求 | [ssrf-url-validation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/utils/ssrf-url-validation.ts#L455-L536) | L455-L536 |
| Workflow Bridge 执行 | [execute-framework-request.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts#L56-L143) | L56-L143 |
| HTTP 客户端重试 | [http-client.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/libs/application-generic/src/services/http-client/http-client.service.ts#L82-L151) | L82-L151 |
| Agent 用户代码执行 | [handler.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/handler.ts#L319-L348) | L319-L348 |
| Agent 上下文 (reply) | [agent.context.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/resources/agent/agent.context.ts#L250-L319) | L250-L319 |
| Agent Reply 端点 | [agent-reply.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/apps/api/src/app/agents/conversation-runtime/reply/agent-reply.controller.ts#L14-L41) | L14-L41 |
| Bridge 协议类型 | [agent.types.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/resources/agent/agent.types.ts#L383-L400) | L383-L400 |
| 签名错误类型 | [signature.errors.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/60-novu/packages/framework/src/errors/signature.errors.ts#L1-L48) | L1-L48 |
