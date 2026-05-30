# 发送渠道健康检测与异常降级实现分析（代码精校版 v4）

> 新增修正点：
> 1. ✅ MS Teams 健康检查中 `failed` 状态的全部来源路径
> 2. ✅ `provisioning.status` 不同取值的行为差异与排障优先级
> 3. ✅ 状态透传逻辑在前端展示上的影响
> 4. ✅ 避免误判的核查步骤

---

## 一、MS Teams 健康检查中 `failed` 状态的全部来源

### 1.1 源码级精确追溯

`apps/api/src/app/integrations/usecases/msteams-health-check/msteams-health-check.usecase.ts`

#### 1.1.1 唯一会返回 `failed` 的代码路径

```typescript
// 第 79-90 行 —— 这是整个文件中唯一返回 'failed' 的地方
if (!clientId || !secretKey || !tenantId) {
  const failedStatus = (name: string): HealthCheckStatus | null =>
    !command.checks || command.checks.includes(name) ? 'failed' : null;

  return {
    appRegistration: failedStatus('appRegistration'),
    azureBotCreated: failedStatus('azureBotCreated'),
    teamsAppCatalog: failedStatus('teamsAppCatalog'),
    permissions: failedStatus('permissions'),
    allReady: false,
  };
}
```

**触发条件**：`clientId`、`secretKey`、`tenantId` 三者**任一为空字符串或 undefined**。

#### 1.1.2 四个 check 方法永不返回 `failed`

| 检查点 | 正常返回 | 异常返回 |
|--------|---------|---------|
| `checkAppRegistration` | `'ready'` (token 存在) | `'pending'` (所有异常) |
| `checkAzureBotCreated` | `integration.provisioning?.status` | `'pending'` (字段缺失) |
| `checkTeamsAppCatalog` | `'ready'` (找到) | `'pending'` (所有异常) |
| `checkPermissions` | `'ready'` (权限齐全) | `'pending'` (所有异常) |

#### 1.1.3 `pending` 覆盖的异常场景

```typescript
// checkAppRegistration: 127-131 行
catch (error) {
  this.logger.warn(`Health check: appRegistration failed clientId=${clientId} error="${(error as Error).message}"`);
  return 'pending';  // 包括但不限于：
  // - invalid_client (凭据错误)
  // - unauthorized_client (权限不足)
  // - invalid_grant (授权无效)
  // - network timeout
  // - DNS failure
  // - 任何其他 HTTP 错误
}
```

**所有网络异常、权限错误、凭证过期、资源未找到，全部返回 `pending`**。用户在 UI 上永远看到转圈的加载图标，看不到红色错误。

### 1.2 `failed` 场景的完整清单

| 场景 | 哪些检查点返回 `failed` | 其他检查点 |
|------|------------------------|-----------|
| clientId 为空 | 全部 4 个 | N/A |
| secretKey 为空 | 全部 4 个 | N/A |
| tenantId 为空 | 全部 4 个 | N/A |
| 仅指定 `checks=appRegistration` 且 clientId 为空 | `appRegistration: failed` | 其他为 `null` |
| 仅指定 `checks=permissions` 且 secretKey 为空 | `permissions: failed` | 其他为 `null` |

### 1.3 `hasFailed` 前端判定的真实覆盖

`apps/dashboard/src/components/agents/teams-setup-guide.tsx:384`
```typescript
const hasFailed = status?.appRegistration === 'failed' || status?.azureBotCreated === 'failed';
```

**关键发现**：
- 只判断 `appRegistration` 和 `azureBotCreated` 是否为 `failed`
- `teamsAppCatalog` 和 `permissions` 就算能返回 `failed`（实际不会），也不会触发错误提示
- 由于 `provisioning.status` 只可能是 `pending`/`ready`/`failed`，而 `failed` 只有在 ARM 部署写入时才会出现，所以 `azureBotCreated` 为 `failed` 时才会真正显示红色错误 Toast

---

## 二、`provisioning.status` 不同取值的行为差异

### 2.1 字段写入的完整时间线

`apps/api/src/app/integrations/usecases/azure-setup-oauth-callback/azure-setup-oauth-callback.usecase.ts`

#### 2.1.1 写入时机与取值

| 时机 | 写入代码位置 | 写入值 | 附加字段 |
|------|-------------|--------|---------|
| ARM 部署开始前 | 第 431 行 | `'pending'` | `startedAt: ISO string` |
| 无 Azure 订阅 | 第 451-455 行 | `'failed'` | `completedAt`, `errorMessage` |
| ARM 部署成功 | 第 553 行 | `'ready'` | `completedAt` |
| ARM 部署异常 | 第 565-569 行 | `'failed'` | `completedAt`, `errorMessage` |
| 无 refresh token | 第 101-105 行 | `'failed'` | `completedAt`, `errorMessage` |
| App Catalog 上传成功 | 第 710-713 行 | `'pending'` | `teamsAppCatalogId` |

**关键发现**：第 710-713 行在上传 App Catalog 成功后，会**覆盖** `status` 为 `'pending'`。如果 ARM 部署先完成写入了 `'ready'`，App Catalog 上传后会把状态**回退**为 `'pending'`。

#### 2.1.2 `provisioning` 完整结构

```typescript
// 第 578-583 行 — writeProvisioning 类型定义
provisioning: {
  status: 'pending' | 'ready' | 'failed';
  startedAt?: string;           // ISO timestamp
  completedAt?: string;         // ISO timestamp
  errorMessage?: string;        // ARM 部署失败原因
  teamsAppCatalogId?: string;   // Teams App Catalog 中的 ID
}
```

### 2.2 各取值的行为差异与排障优先级

| status 值 | 来源 | 前端展示 | 对 `allReady` 的影响 | 排障优先级 | 建议操作 |
|-----------|------|---------|-------------------|-----------|---------|
| **`undefined`** | 字段不存在（手动配置或旧数据） | 灰圈 pending | ❌ 阻止 allReady | ⭐⭐⭐⭐⭐ 最高 | 检查是否走了 Quick Setup 流程；如果是手动配置，忽略此项或补全 |
| **`'pending'`** | 1. ARM 部署中<br>2. 部署完成但被 App Catalog 上传覆盖 | 转圈 | ❌ 阻止 allReady | ⭐⭐⭐ 中 | 查看 startedAt 判断等待了多久；>10 分钟需查后端日志 |
| **`'ready'`** | ARM 部署全部成功 | 绿勾 | ✅ allReady 条件满足 | ⭐ 低 | 正常状态，无需操作 |
| **`'failed'`** | 1. 无 Azure 订阅<br>2. ARM 部署失败<br>3. 无 refresh token | 红叉 | ❌ 阻止 allReady | ⭐⭐⭐⭐ 高 | 查看 `errorMessage` 字段；点击「Retry Azure setup」按钮 |

### 2.3 状态覆盖的竞态条件

`tryDeployBotService` 是**异步 fire-and-forget**（第 92 行 `void this.tryDeployBotService(...)`），而 `tryUploadTeamsApp` 是**同步等待**（第 88 行 `await this.tryUploadTeamsApp(...)`）。

**典型时序**：
1. OAuth callback 到达 → 同步执行 `tryUploadTeamsApp`
2. `tryUploadTeamsApp` 成功 → 写入 `status: 'pending'` + `teamsAppCatalogId`
3. 异步启动 `tryDeployBotService` → 写入 `status: 'pending'` + `startedAt`
4. ARM 部署完成 → 写入 `status: 'ready'`

**异常时序（竞态）**：
1. OAuth callback 到达
2. 异步启动 `tryDeployBotService` → 快速执行完成 → 写入 `status: 'ready'`
3. 同步 `tryUploadTeamsApp` 稍晚完成 → **覆盖为 `'pending'`**
4. 状态永久停留在 `pending`，因为没有后续写入

**影响**：健康检查一直显示 `azureBotCreated: pending`，`allReady` 永远为 false，用户被卡住无法进入下一步。

---

## 三、状态透传逻辑在前端展示上的影响

### 3.1 两套健康检查轮询逻辑

#### 3.1.1 Quick Setup 模式 — `HealthCheckView` 组件

`teams-setup-guide.tsx:338-411`

- **检查点**：全部 4 个（appRegistration、azureBotCreated、teamsAppCatalog、permissions）
- **停轮询条件**：`result.allReady === true`
- **错误 Toast 触发**：`hasFailed = appRegistration==='failed' || azureBotCreated==='failed'`
- **状态默认值**：`status` 为 null 时，全部显示 `'pending'`（第 377 行 `allPending: HealthCheckStatus = 'pending'`）

#### 3.1.2 Manual Setup 模式 — `useManualHealthPoll` hook

`teams-setup-guide.tsx:592-640`

```typescript
const MANUAL_HEALTH_CHECKS = ['teamsAppCatalog', 'permissions'] as const;
```

- **检查点**：仅 2 个（teamsAppCatalog、permissions），`azureBotCreated` 强制为 `null`
- **停轮询条件**：`teamsAppCatalog === 'ready' && permissions === 'ready'`
- **Azure Bot Created**：在手动模式下不检查，UI 上该行不显示或强制通过

### 3.2 `CheckpointRow` 状态映射

`teams-setup-guide.tsx:308-328`

| status | 图标 | 文字颜色 | 语义 |
|--------|------|---------|------|
| `'ready'` | ✅ RiCheckLine 绿色 | `text-text-strong` | 检查通过 |
| `'pending'` | ⏳ RiLoader4Line 旋转灰色 | `text-text-sub` | 检查中 / 等待传播 |
| `'failed'` | ❌ RiCloseLine 红色 | `text-error-base` | 配置错误 |

### 3.3 状态透传导致的误判场景

#### 3.3.1 场景 1：空状态默认 pending

```typescript
// 第 381 行 — 如果 status 为 null，所有检查点显示 pending
status: (status ? (status[key] ?? allPending) : allPending) as HealthCheckStatus
```

**影响**：
- 第一次请求还没返回时，用户看到 4 个检查点都在转圈
- 网络中断导致请求失败（被 catch 吞掉），UI 永远转圈不会报错
- 用户可能误以为系统还在检查，实际上请求已经失败

#### 3.3.2 场景 2：异常吞掉导致状态冻结

```typescript
// 第 365-367 行 — 所有异常被忽略
catch {
  // ignore transient errors and keep polling
}
```

**影响**：
- 后端返回 500、401、403 等错误，UI 毫无反应
- 状态停留在上一次成功返回的值
- 如果上一次是 pending，用户永远看不到错误，只有去查后端日志才能发现

#### 3.3.3 场景 3：`azureBotCreated` 与实际资源状态不一致

`checkAzureBotCreated` 只读取数据库字段，不做网络探测。

**误判示例**：
1. Quick Setup 成功，`provisioning.status = 'ready'`，UI 显示绿勾
2. 用户在 Azure Portal 手动删除了 Bot Service 资源
3. 健康检查仍显示 `azureBotCreated: ready`
4. 实际发送消息时失败，UI 与实际状态严重脱节

#### 3.3.4 场景 4：`permissions` pending 的多义性

`checkPermissions` 返回 `pending` 可能代表：
- Graph API 调用网络超时
- Service Principal 还没创建完成（Azure 传播延迟）
- 权限还在传播（正常，可能需要 60 分钟）
- 权限确实缺失（管理员没批准）
- clientId/secretKey 错误（但应该在 appRegistration 阶段失败）

**影响**：用户无法区分是「等等就好」还是「需要管理员操作」，只能一直等。

### 3.4 `allReady` 计算逻辑

```typescript
// 第 103-105 行
const allReady = [appRegistration, azureBotCreated, teamsAppCatalog, permissions]
  .filter((s) => s !== null)      // 跳过未请求的检查点
  .every((s) => s === 'ready');   // 剩余全部为 ready 才返回 true
```

**手动模式下**：
- `azureBotCreated = null` 被过滤掉
- 只需要 `teamsAppCatalog` 和 `permissions` 为 `ready` 即可通过

---

## 四、避免误判的核查步骤

### 4.1 健康检查显示 pending 超过 10 分钟的排查流程

**Step 1：检查 API 响应真实状态**
```bash
# 直接调用 API 看原始返回
curl "https://api.novu.co/v1/integrations/{integrationId}/msteams-health" \
  -H "Authorization: ApiKey {key}"
```

**Step 2：查看后端日志**
```bash
# 搜索健康检查警告日志
grep "Health check:.*failed" /var/log/novu/api.log
```
可能看到：
- `Health check: appRegistration failed clientId=xxx error="invalid_client"`
- `Health check: permissions failed clientId=xxx error="Request failed with status code 403"`

**Step 3：检查 provisioning 字段**
```javascript
// 在 MongoDB 中查询
db.integrations.findOne(
  { _id: ObjectId("integrationId") },
  { provisioning: 1, credentials: 1 }
)
```
- 如果 `provisioning` 字段不存在 → 说明没走 Quick Setup 或旧数据
- 如果 `provisioning.status === 'failed'` → 查看 `errorMessage`
- 如果 `provisioning.startedAt` 超过 30 分钟前 → ARM 部署可能卡住了

**Step 4：验证 credentials 是否完整**
- 解密后检查 `clientId`、`secretKey`、`tenantId` 是否都有有效值
- 特别注意 `secretKey` 可能只存了部分（被截断）

### 4.2 健康检查显示 ready 但发不出消息的核查

**Step 1：验证 Azure Bot 资源是否存在**
```powershell
# 使用 Azure CLI 检查
az bot show --name {botName} --resource-group rg-{botName}
```

**Step 2：验证 Teams 通道是否启用**
```powershell
az bot msteams show --name {botName} --resource-group rg-{botName}
```

**Step 3：验证 Graph API 权限是否真正生效**
```bash
# 用相同的 clientId/secretKey 获取 token，然后手动调用 Graph
curl "https://graph.microsoft.com/v1.0/me" \
  -H "Authorization: Bearer {token}"
```

**Step 4：检查 messaging endpoint 是否可达**
```bash
# Bot Service 的 webhook 地址是否能被 Azure 访问
curl -X POST {webhookUrl} -H "Content-Type: application/json" -d '{}'
```

### 4.3 手动配置场景的特殊核查

手动配置模式下，`azureBotCreated` 检查被跳过。如果手动配置后发不出消息：

**Step 1：确认 Bot Service 的 App ID 是否匹配**
- Azure Portal → Bot Service → Configuration → Microsoft App ID
- 应等于 Novu 集成中配置的 `clientId`

**Step 2：确认 Client Secret 是否有效**
- Azure Portal → App Registration → Certificates & secrets
- 检查 Secret 是否过期（手动配置的用户常忘记设置有效期）

**Step 3：确认 Teams 通道已启用**
- Azure Portal → Bot Service → Channels → Microsoft Teams
- Status 应为 "Running"

**Step 4：检查权限配置**
- 确保 6 个 Graph API 权限都已授予**管理员同意**
- 权限类型必须是「Application」，不是「Delegated」

### 4.4 前端卡住无法进入下一步的紧急处理

如果 `allReady` 一直为 false 但实际配置已就绪：

**方法 1：使用浏览器 DevTools 临时修改**
1. 打开 DevTools → Application → Session Storage
2. 设置 `novu:bot-deployed:{integrationId} = 'true'`
3. 刷新页面，健康检查门将被跳过

**方法 2：手动更新数据库**
```javascript
// 强制标记为 ready
db.integrations.updateOne(
  { _id: ObjectId("integrationId") },
  { $set: { "provisioning.status": "ready" } }
)
```

**方法 3：重新走 Quick Setup**
- 删除当前集成
- 重新创建并授权 Azure
- 确保浏览器不拦截 OAuth 弹窗

---

## 五、provisioning.status 排障决策树

```
azureBotCreated 状态 = ?
├─  undefined / null
│   ├─  是手动配置吗？
│   │   ├─  是 → 忽略此项，看其他三项
│   │   └─  否 → ⚠️  Quick Setup 没完成，需要重新授权
│   └─  排障优先级：⭐⭐⭐⭐⭐ 最高
│
├─  "pending"
│   ├─  startedAt 存在吗？
│   │   ├─  不存在 → ⚠️  ARM 部署可能根本没启动，检查 refresh token
│   │   └─  存在 → 等待了多久？
│   │       ├─  < 5 分钟 → ✅ 正常，继续等待
│   │       ├─  5-30 分钟 → ⚠️  可以继续等，或查后端日志
│   │       └─  > 30 分钟 → ⚠️  ARM 部署可能卡住了，查 errorMessage
│   └─  排障优先级：⭐⭐⭐ 中等
│
├─  "ready"
│   ├─  实际发送测试了吗？
│   │   ├─  发送成功 → ✅ 正常
│   │   └─  发送失败 → ⚠️  Azure 资源可能被手动删除了，去 Azure Portal 核查
│   └─  排障优先级：⭐ 低
│
└─  "failed"
    ├─  errorMessage 是什么？
    │   ├─  "No enabled Azure subscription found" → 用户需要有效的 Azure 订阅
    │   ├─  "No refresh token available" → OAuth 授权不完整，需要重新授权
    │   ├─  "ARM step [create-bot-service] failed" → ARM API 调用失败，检查权限
    │   ├─  "ARM step [enable-teams-channel] failed" → Teams 通道启用失败
    │   └─  其他 → 根据具体错误信息排查
    └─  排障优先级：⭐⭐⭐⭐ 高
```

---

## 六、关键代码位置速查表（v4 新增）

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 唯一返回 failed 的代码路径 | `msteams-health-check.usecase.ts` | 79-90 |
| 4 个 check 方法全部返回 pending | `msteams-health-check.usecase.ts` | 118-213 |
| provisioning 写入 pending（ARM 开始） | `azure-setup-oauth-callback.usecase.ts` | 431 |
| provisioning 写入 failed（无订阅） | `azure-setup-oauth-callback.usecase.ts` | 451-455 |
| provisioning 写入 ready（ARM 成功） | `azure-setup-oauth-callback.usecase.ts` | 553 |
| provisioning 写入 failed（ARM 异常） | `azure-setup-oauth-callback.usecase.ts` | 565-569 |
| provisioning 写入 failed（无 refresh token） | `azure-setup-oauth-callback.usecase.ts` | 101-105 |
| provisioning 覆盖为 pending（App Catalog） | `azure-setup-oauth-callback.usecase.ts` | 710-713 |
| writeProvisioning 字段级更新 | `azure-setup-oauth-callback.usecase.ts` | 591-596 |
| 前端 hasFailed 判断 | `teams-setup-guide.tsx` | 384 |
| CheckpointRow 状态映射 | `teams-setup-guide.tsx` | 308-328 |
| 空状态默认 pending | `teams-setup-guide.tsx` | 377, 381 |
| 异常吞掉保持 polling | `teams-setup-guide.tsx` | 365-367 |
| 手动模式跳过 azureBotCreated | `teams-setup-guide.tsx` | 588, 610 |
| allReady 计算逻辑 | `msteams-health-check.usecase.ts` | 103-105 |
| tryDeployBotService fire-and-forget | `azure-setup-oauth-callback.usecase.ts` | 92 |
| tryUploadTeamsApp 同步等待 | `azure-setup-oauth-callback.usecase.ts` | 88 |
| HEALTH_POLL_INTERVAL_MS | `teams-setup-guide.tsx` | 299 |

---

## 七、设计缺陷总结

| 问题 | 影响 | 严重程度 |
|------|------|---------|
| 几乎所有异常都返回 pending，不区分临时性错误和永久性错误 | 用户无法判断是该等还是该排查 | 🔴 高 |
| azureBotCreated 只查数据库不做网络探测 | Azure 资源被删后 UI 仍显示正常 | 🔴 高 |
| App Catalog 上传可能覆盖 ready 为 pending | 竞态导致状态永久卡住 | 🟡 中 |
| 前端吞掉所有网络异常 | 请求失败用户看不到任何提示 | 🟡 中 |
| 手动模式不检查 azureBotCreated | 手动配置的 Bot 资源有问题无法通过 UI 发现 | 🟡 中 |
| permissions pending 语义模糊 | 用户不知道是传播延迟还是真正缺少权限 | 🟡 中 |
