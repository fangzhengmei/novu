# 发送渠道健康检测与异常降级实现分析（代码精校版 v3）

> 修正点对照：
> 1. ✅ 队列就绪检测的真实检查项
> 2. ✅ MS Teams 中 azureBotCreated 与 permissions 的完整判定链路
> 3. ✅ 主集成切换后 active 与 primary 的状态变化逻辑
> 4. ✅ 健康检查触发节奏的定时常量与停轮询条件
> 5. ✅ 这些差异对排障和运营决策的影响

---

## 一、队列就绪检测的真实检查项

### 1.1 健康检查调用链的真实层级

之前的理解有误：`QueueHealthIndicator.isReady()` 调用的是 `queueService.isReady()`，而 `queueService.isReady()` 调用的是 `bullMqService.isClientReady()`，**最终只检查 Redis 客户端的连接状态**，不检查队列是否暂停或 Worker 是否运行。

| 层级 | 方法 | 实际检查内容 |
|------|------|-------------|
| Health Controller | `indicator.isHealthy()` | 抛出 HealthCheckError 或返回 result |
| QueueHealthIndicator | `handleHealthCheck()` | 调用 `queueService.isReady()` |
| QueueBaseService | `isReady()` | 调用 `bullMqService.isClientReady()` |
| BullMqService | `isClientReady()` | 调用 `workflowInMemoryProviderService.isReady()` |
| InMemoryProvider | `isReady()` | 检查 `status === 'ready'`（ioredis status） |

**关键修正**：队列健康检查的 `isReady()` 方法**不等同于** `getStatus()`。`getStatus()` 是一个独立方法，会并行查询：
- `isQueuePaused()` → 队列是否被暂停
- `isWorkerPaused()` → Worker 是否被暂停
- `isWorkerRunning()` → Worker 是否在运行

但健康检查端点**不调用** `getStatus()`，只调用 `isReady()`。

### 1.2 `isClientReady` 的判定逻辑

`libs/application-generic/src/services/in-memory-provider/providers/redis-provider.ts:121`
```typescript
export const isClientReady = (status: string): boolean => status === CLIENT_READY;
```

`CLIENT_READY = 'ready'`，即 ioredis 的连接状态等于 `ready` 才算健康。其他状态（`connecting`、`connect`、`reconnecting`、`disconnecting`、`disconnected`）都算不健康。

### 1.3 各 Provider 的统一判定

所有 Redis Provider（standalone、cluster、master-slave、AWS ElastiCache、Azure Cache、MemoryDB）都使用相同的 `isClientReady` 判定逻辑：
- `redis-provider.ts:121`
- `redis-cluster-provider.ts:126`
- `redis-master-slave-provider.ts:164`
- `elasticache-cluster-provider.ts:124`
- `azure-cache-for-redis-cluster-provider.ts:129`
- `memory-db-cluster-provider.ts:144`

---

## 二、MS Teams 健康检查的完整判定链路

### 2.1 四个检查点的真实实现

`apps/api/src/app/integrations/usecases/msteams-health-check/msteams-health-check.usecase.ts`

#### 2.1.1 `appRegistration` 检查
```typescript
private async checkAppRegistration(clientId, secretKey, tenantId): Promise<HealthCheckStatus> {
  try {
    const token = await this.msTeamsTokenService.getGraphToken(...);
    return token ? 'ready' : 'pending';
  } catch {
    return 'pending';  // 所有异常都返回 pending，没有 failed 状态
  }
}
```
- **判定**：成功获取 Graph API token → `ready`
- **异常处理**：任何网络错误、凭证错误、权限错误 → 全部 `pending`，**从不返回 `failed`**

#### 2.1.2 `azureBotCreated` 检查（关键修正）
```typescript
private checkAzureBotCreated(integration: IntegrationEntity): HealthCheckStatus {
  return integration.provisioning?.status ?? 'pending';
}
```
- **判定**：直接读取 `integration.provisioning.status` 字段值
- **来源**：此字段在 Quick Setup 流程中由 `tryDeployBotService` 写入
- **缺失行为**：如果 `provisioning` 字段不存在 → 返回 `pending`（不是 `failed`）
- **关键发现**：这是**纯数据库读取**，不产生任何网络请求。如果 Azure Bot 资源被手动删除但数据库记录未更新，此检查仍会返回之前存储的 `ready`。

#### 2.1.3 `teamsAppCatalog` 检查
```typescript
private async checkTeamsAppCatalog(clientId, secretKey, tenantId): Promise<HealthCheckStatus> {
  // 1. 获取 Graph token
  // 2. 调用 Graph API: GET /appCatalogs/teamsApps?$filter=externalId eq '{clientId}' and distributionMethod eq 'organization'
  // 3. response.data.value.length > 0 → ready
  // 4. 任何异常 → pending（不返回 failed）
}
```
- 检查 App 是否已发布到组织的 Teams 应用目录
- 超时设置：10 秒

#### 2.1.4 `permissions` 检查（关键修正）
```typescript
private async checkPermissions(clientId, secretKey, tenantId): Promise<HealthCheckStatus> {
  // 1. 获取 Graph token
  // 2. 解析 service principal: GET /servicePrincipals?$filter=appId eq '{clientId}'&$select=id
  // 3. 查询权限分配: GET /servicePrincipals/{spId}/appRoleAssignments
  // 4. 对比 grantedRoleIds 是否包含所有 EXPECTED_ROLE_IDS
  
  const EXPECTED_ROLE_IDS = new Set([
    '7ab1d382-f21e-4acd-a863-ba3e13f7da61', // Directory.Read.All
    '2280dda6-0bfd-44ee-a2f4-cb867cfc4c1e', // Team.ReadBasic.All
    '59a6b24b-4225-4393-8165-ebaec5f55d7a', // Channel.ReadBasic.All
    'e12dae10-5a57-4817-b79d-dfbec5348930', // AppCatalog.Read.All
    '9f67436c-5415-4e7f-8ac1-3014a7132630', // TeamsAppInstallation.ReadWriteSelfForTeam.All
    '908de74d-f8b2-4d6b-a9ed-2a17b3b78179', // TeamsAppInstallation.ReadWriteSelfForUser.All
  ]);
}
```
- **两步网络请求**：先查 SP 再查权限分配
- **判定标准**：6 个权限**全部**存在 → `ready`，缺失任意一个 → `pending`
- **异常处理**：任何网络错误 → `pending`（不返回 `failed`）

### 2.2 `allReady` 判定逻辑
```typescript
const allReady = [appRegistration, azureBotCreated, teamsAppCatalog, permissions]
  .filter((s) => s !== null)      // 只考虑实际请求的检查点
  .every((s) => s === 'ready');   // 全部为 ready 才返回 true
```

### 2.3 `failed` 状态的唯一来源

只有当 `clientId/secretKey/tenantId` 三者任一为空时，四个检查点才会返回 `failed`。检查过程中产生的所有异常都返回 `pending`，**意味着健康检查 UI 上几乎不会显示红色的 failed 状态**。

---

## 三、主集成切换后的 active 与 primary 状态变化

### 3.1 SetIntegrationAsPrimary 的真实行为

`apps/api/src/app/integrations/usecases/set-integration-as-primary/set-integration-as-primary.usecase.ts:18-48`

```typescript
private async updatePrimaryFlag({ existingIntegration }) {
  // Step 1: 将当前所有 active=true 且 primary=true 的集成 → primary=false
  await this.integrationRepository.update(
    {
      _organizationId,
      _environmentId,
      channel,
      active: true,
      primary: true,
    },
    { $set: { primary: false } }  // 注意：只改 primary，不改 active
  );

  // Step 2: 将目标集成设为 primary=true、active=true、清空 conditions
  await this.integrationRepository.update(
    { _id: existingIntegration._id, ... },
    {
      $set: {
        active: true,        // 强制设为 active
        primary: true,       // 设为主集成
        conditions: [],      // 主集成不能有过滤条件
      },
    }
  );
}
```

### 3.2 关键修正：旧主集成的状态

| 操作 | 旧主集成的 primary | 旧主集成的 active |
|------|-------------------|-------------------|
| Set Primary 执行前 | `true` | `true`（查询条件） |
| Set Primary 执行后 | `false` | **`true`（不变）** |

**之前的结论错误**：旧主集成只是被去掉了 `primary` 标记，**仍然保持 active**，不会自动 deactivated。它仍然会被 `SelectIntegration` 选中（如果条件匹配），只是优先级更低。

### 3.3 UpdateIntegration 中 conditions 对 primary 的影响

`apps/api/src/app/integrations/usecases/update-integration/update-integration.usecase.ts:227-250`

```typescript
const shouldRemovePrimary = haveConditions && existingIntegration.primary;
if (shouldRemovePrimary) {
  updatePayload.primary = false;
}
// ...
if (shouldRemovePrimary) {
  await this.integrationRepository.recalculatePriorityForAllActive({ ... });
}
```

- 如果一个 primary 集成被添加了 `conditions`（过滤规则），系统会自动移除其 `primary` 标记
- 触发 `recalculatePriorityForAllActive` 重新计算优先级

### 3.4 UpdateIntegration 中 active 变化的影响

```typescript
if (isActiveChanged && isChannelSupportsPrimary) {
  const { primary, priority } = await this.calculatePriorityAndPrimary({
    existingIntegration,
    active: !!command.active,
  });
  updatePayload.primary = primary;
  updatePayload.priority = priority;
}
```

当 `active` 从 `true` 改为 `false` 时：
```typescript
await this.integrationRepository.recalculatePriorityForAllActive({
  _id: existingIntegration._id,  // exclude = true 时排除当前
  exclude: true,
});
result = { priority: 0, primary: false };  // 被禁用的集成优先级=0，primary=false
```

---

## 四、健康检查触发节奏的定时常量与停轮询条件

### 4.1 MS Teams 健康检查轮询

`apps/dashboard/src/components/agents/teams-setup-guide.tsx:299`

| 常量 | 值 | 用途 |
|------|----|------|
| `HEALTH_POLL_INTERVAL_MS` | 10,000 ms (10 秒) | 健康检查四个检查点的轮询间隔 |
| `ENDPOINT_POLL_INTERVAL_MS` | 3,000 ms (3 秒) | channel endpoint 连接状态轮询间隔 |
| `ENDPOINT_POLL_GRACE_MS` | 30,000 ms (30 秒) | endpoint 轮询超时窗口，超过后显示手动恢复按钮 |

#### 停轮询条件（`stopPolling` 触发时机）：
1. ✅ `result.allReady === true` → 所有检查点就绪，调用 `onReady()` 回调
2. ✅ 组件 `useEffect` cleanup → 组件卸载时自动清理
3. ✅ `waitingForSetup === true` → Azure 授权弹窗打开期间暂停（但不清理 interval，只是 poll 函数 early return）
4. ❌ 没有超时机制 → 如果权限传播需要 60 分钟，轮询会一直持续直到用户离开页面

### 4.2 Bridge 健康检查轮询

`apps/dashboard/src/hooks/use-fetch-bridge-health-check.ts:9`

| 常量 | 值 | 用途 |
|------|----|------|
| `BRIDGE_STATUS_REFRESH_INTERVAL_IN_MS` | 10,000 ms (10 秒) | Bridge URL 健康状态轮询间隔 |

#### 停轮询条件：
1. ✅ `enabled: !!bridgeURL` → 环境没有配置 Bridge URL 时不启动
2. ✅ 组件卸载 → React Query 自动清理
3. ❌ 没有成功后停止逻辑 → Bridge 连接成功后仍会每 10 秒轮询一次
4. ✅ `refetchOnWindowFocus: true` → 窗口重新获得焦点时额外触发一次

### 4.3 域名验证轮询

`apps/dashboard/src/hooks/use-domain.ts:16`

| 常量 | 值 | 用途 |
|------|----|------|
| `VERIFICATION_POLL_INTERVAL_MS` | 5,000 ms (5 秒) | DKIM/SPF 验证状态轮询间隔 |

#### 停轮询条件：
1. ✅ `enabled: isPending` → 只有当状态为 `PENDING` 时才轮询
2. ✅ `refetchIntervalInBackground: false` → 标签页后台运行时不轮询
3. ✅ 状态变为非 PENDING 后自动停止（由 enabled 条件控制）

### 4.4 Telegram 设置轮询

`apps/dashboard/src/components/agents/telegram-mobile-setup-card.tsx:12`

| 常量 | 值 | 用途 |
|------|----|------|
| `REFRESH_INTERVAL_MS` | 300,000 ms (5 分钟) | Telegram bot webhook 状态轮询 |

### 4.5 Analytics 图表刷新

`apps/dashboard/src/components/analytics/constants/analytics-page.consts.ts:41`

| 常量 | 值 | 用途 |
|------|----|------|
| `CHART_CONFIG.refetchInterval` | 300,000 ms (5 分钟) | Analytics 图表数据刷新间隔 |

---

## 五、这些差异对排障和运营决策的影响

### 5.1 排障指导修正

| 现象 | 之前的理解 | 真实代码行为 | 排障建议 |
|------|-----------|-------------|---------|
| 健康检查返回 `up` 但消息堆积 | 健康检查应该检测到队列问题 | 健康检查只看 Redis 连接，不检测积压 | 不能依赖 `/health-check` 判断系统健康，需要额外监控队列长度 |
| MS Teams 健康检查全绿但发不出消息 | 健康检查通过应该能发消息 | `azureBotCreated` 是数据库快照，Azure 资源可能已被删除 | 不要只看 UI 绿灯，实际发送一条测试消息验证 |
| 旧主集成仍然被选中 | 设新主集成后旧的应该被禁用 | 旧主集成 `primary=false` 但 `active=true`，条件匹配时仍会被选中 | 如果要彻底弃用旧集成，手动将其 `active=false` |
| 集成添加条件后突然不工作了 | 应该同时支持 primary 和 conditions | 添加 conditions 自动移除 primary 标记 | 需要 primary 的集成不要加条件，条件集成用优先级控制 |
| MS Teams 健康检查一直 pending | 可能是网络问题 | 所有异常都返回 pending，包括凭证错误和权限不足 | 查看后端日志 `logger.warn("Health check: xxx failed")` 才能知道真实原因 |

### 5.2 运营决策影响

#### 5.2.1 监控告警设计
- ❌ 不要对 `/health-check` 返回 200 过度自信，它只测连通性
- ✅ 需要额外监控：队列深度、消费延迟、错误率、发送成功率
- ✅ Provider 级健康状态需要自己构建（基于 ExecutionDetails 统计），Novu 不提供现成的

#### 5.2.2 故障恢复流程
- **Kill Switch 启用后**：消息被永久丢弃，不要指望能自动恢复
- **SQS 降级到 BullMQ**：是请求级的，SQS 恢复后下一条消息自动切回去，不需要人工干预
- **Novu Provider 限额耗尽**：自然月自动重置，期间需要配置自定义 Provider
- **MS Teams 权限传播慢**：健康检查会一直 pending，用户可以关闭页面稍后回来，也可以保持页面打开一直等（最长可能 60 分钟）

#### 5.2.3 集成管理策略
- 同渠道多集成时，建议显式配置 conditions 或 priority，不要依赖数据库返回顺序
- 切换主集成后，检查旧集成是否需要手动禁用
- Email/SMS 渠道可以用 primary，其他渠道（Push/Chat）只能靠 conditions 和 priority 控制优先级

#### 5.2.4 容量规划
- Novu Provider 限额比较低（Email 300/月、SMS 20/月），生产环境必须配置自定义 Provider
- 高流量场景建议配置多个同渠道集成，通过 conditions 做灰度或分流
- 没有熔断机制意味着 Provider 故障时不会自动切流量，需要：
  - 要么人工介入改集成配置
  - 要么通过 Feature Flag 改队列后端
  - 要么启用 Kill Switch 停止该组织

### 5.3 与设计预期的 Gap

| 设计预期 | 实际实现 | Gap 风险 |
|---------|---------|---------|
| 健康检查 = 系统可正常服务 | 健康检查 = Redis 可连接 | 健康检查通过但系统可能已瘫痪 |
| Provider 失败自动降级到备用 | 没有 Provider 级降级逻辑 | 单个 Provider 故障可能导致全渠道失败 |
| 错误应该重试提高成功率 | 只有 Webhook Filter 错误才重试 | 大部分发送失败直接放弃 |
| 旧主集成应该被清理 | 旧主集成仍然 active | 流量可能走向预期外的 Provider |
| 健康检查 failed 表示配置错误 | 几乎不会返回 failed，都是 pending | 用户不知道问题出在哪里，只能一直等 |

---

## 六、关键代码位置速查表（v3 修正版）

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 队列健康指标基类（真实 isReady 链路） | `libs/application-generic/src/health/queue-health-indicator.service.ts` | 18-31 |
| BullMqService.isClientReady | `libs/application-generic/src/services/bull-mq/bull-mq.service.ts` | 245-247 |
| BullMqService.getStatus（三个状态检查） | `libs/application-generic/src/services/bull-mq/bull-mq.service.ts` | 223-243 |
| Redis isClientReady 判定 | `libs/application-generic/src/services/in-memory-provider/providers/redis-provider.ts` | 121 |
| MS Teams 健康检查全量实现 | `apps/api/src/app/integrations/usecases/msteams-health-check/` | 全文件 |
| azureBotCreated 纯数据库读取 | `apps/api/src/app/integrations/usecases/msteams-health-check/msteams-health-check.usecase.ts` | 139-141 |
| permissions 两步 Graph API 调用 | `apps/api/src/app/integrations/usecases/msteams-health-check/msteams-health-check.usecase.ts` | 177-213 |
| SetIntegrationAsPrimary（真实行为） | `apps/api/src/app/integrations/usecases/set-integration-as-primary/` | 18-48 |
| UpdateIntegration conditions 移除 primary | `apps/api/src/app/integrations/usecases/update-integration/update-integration.usecase.ts` | 227-250 |
| UpdateIntegration active 变化处理 | `apps/api/src/app/integrations/usecases/update-integration/update-integration.usecase.ts` | 217-225 |
| HEALTH_POLL_INTERVAL_MS | `apps/dashboard/src/components/agents/teams-setup-guide.tsx` | 299 |
| MS Teams 停轮询条件（allReady） | `apps/dashboard/src/components/agents/teams-setup-guide.tsx` | 359-364 |
| BRIDGE_STATUS_REFRESH_INTERVAL_IN_MS | `apps/dashboard/src/hooks/use-fetch-bridge-health-check.ts` | 9 |
| VERIFICATION_POLL_INTERVAL_MS | `apps/dashboard/src/hooks/use-domain.ts` | 16 |
| ENDPOINT_POLL_GRACE_MS（超时窗口） | `apps/dashboard/src/components/agents/teams-setup-guide.tsx` | 418 |
| recalculatePriorityForAllActive 触发 | `apps/api/src/app/integrations/usecases/set-integration-as-primary.usecase.ts` | 79-84 |
