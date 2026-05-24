# Blueprint 派生工作流的 Origin 分流与可同步判定分析 - R5（事实收敛最终版）

本文档为 `blueprint-migration-flow-r4.md` 的事实收敛修订版，重点修正了两处接口口径偏差：
1. v2 controller 的路由组织方式（path 与 version 的组合，不是静态 `/workflows-v2`）
2. 控制器类名与代码完全一致

所有结论均经过代码事实核对，确保整条判定链路前后一致。

---

## 一、事实校准：Controller 类名与路由配置

### 1.1 NestJS 全局版本配置

**定义位置** `apps/api/src/bootstrap.ts:76-80`

```typescript
app.enableVersioning({
  type: VersioningType.URI,           // URI 版本控制
  prefix: `${CONTEXT_PATH}v`,          // 前缀：通常为 /v
  defaultVersion: '1',                 // 默认版本：1
});
```

**路由组合规则**：`{prefix}{version}/{path}`

### 1.2 v1 Controller（`WorkflowControllerV1`）

**定义位置** `apps/api/src/app/workflows-v1/workflow-v1.controller.ts:63-67`

```typescript
@ApiExcludeController()
@Controller('/workflows')              // path: /workflows
@UseInterceptors(ClassSerializerInterceptor)
@RequireAuthentication()
@ApiTags('Workflows')
export class WorkflowControllerV1 {    // ⚠️ 类名：WorkflowControllerV1
```

**路由推导**：
- 未显式指定 version → 使用默认版本 `1`
- 完整路由前缀：`/v1/workflows`
- 创建接口：`POST /v1/workflows`
- 更新接口：`PUT /v1/workflows/:workflowId`

**创建时 type/origin 硬编码** `apps/api/src/app/workflows-v1/workflow-v1.controller.ts:237-238`

```typescript
type: ResourceTypeEnum.REGULAR,           // ⚠️ 硬编码 REGULAR
origin: ResourceOriginEnum.NOVU_CLOUD_V1,  // ⚠️ 硬编码 NOVU_CLOUD_V1
```

### 1.3 v2 Controller（`WorkflowController`）

**定义位置** `apps/api/src/app/workflows-v2/workflow.controller.ts:77-81`

```typescript
@ThrottlerCategory(ApiRateLimitCategoryEnum.CONFIGURATION)
@ApiCommonResponses()
@Controller({ path: `/workflows`, version: '2' })  // ⚠️ path: /workflows, version: 2
@UseInterceptors(ClassSerializerInterceptor)
@RequireAuthentication()
@ApiTags('Workflows')
export class WorkflowController {         // ⚠️ 类名：WorkflowController（无 V2 后缀）
```

**路由推导**：
- 显式指定 version: `'2'`
- 完整路由前缀：`/v2/workflows`
- 创建接口：`POST /v2/workflows`
- 更新接口：`PUT /v2/workflows/:workflowId`
- 同步接口：`PUT /v2/workflows/:workflowId/sync`（见下文）

**创建时 type/origin 硬编码** `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:155-156`

```typescript
type: ResourceTypeEnum.BRIDGE,        // ⚠️ 硬编码 BRIDGE
origin: ResourceOriginEnum.NOVU_CLOUD, // ⚠️ 硬编码 NOVU_CLOUD
```

### 1.4 SyncToEnvironment 接口路由

**定义位置** `apps/api/src/app/workflows-v2/workflow.controller.ts:124-146`

```typescript
@Put(':workflowId/sync')                // path: :workflowId/sync
@ExternalApiAccessible()
@ApiOperation({
  summary: 'Sync a workflow',
  description: 'Synchronizes a workflow to the target environment',
})
async sync(...): Promise<WorkflowResponseDto> {
  return this.syncToEnvironmentUseCase.execute(...);
}
```

**完整路由**：`PUT /v2/workflows/:workflowId/sync`

### 1.5 Controller 对比表（事实收敛版）

| 维度 | v1 Controller | v2 Controller |
|------|---------------|--------------|
| 类名 | `WorkflowControllerV1` | `WorkflowController` |
| 目录 | `apps/api/src/app/workflows-v1/` | `apps/api/src/app/workflows-v2/` |
| @Controller path | `/workflows` | `/workflows` |
| @Controller version | 未指定（默认 `1`） | 显式 `'2'` |
| 完整路由前缀 | `/v1/workflows` | `/v2/workflows` |
| 创建接口 | `POST /v1/workflows` | `POST /v2/workflows` |
| 同步接口 | ❌ 无 | `PUT /v2/workflows/:workflowId/sync` |
| 创建时 type | `REGULAR`（硬编码） | `BRIDGE`（硬编码） |
| 创建时 origin | `NOVU_CLOUD_V1`（硬编码） | `NOVU_CLOUD`（硬编码） |
| 更新时 type | `REGULAR`（硬编码，覆盖原值） | `BRIDGE`（硬编码，覆盖原值） |
| 更新时 origin | 保留原值 | 保留原值 |
| 废弃标记 | `@deprecated` + `@ApiExcludeController()` | 无 |
| 是否支持 blueprintId | ✅ 是 | ✅ 是 |
| 是否对外开放 | ✅ `@ExternalApiAccessible()` | ✅ `@ExternalApiAccessible()` |

---

## 二、processBlueprint 的 origin 传递规则

### 2.1 完整调用链

```
任何入口调用 CreateWorkflowV0.execute()
    ↓
检查 usecaseCommand.blueprintId 是否存在
    ├─ 不存在 → 直接使用 usecaseCommand
    └─ 存在 → 调用 processBlueprint() 覆盖规则
        ├─ type: 强制 REGULAR（优先级最高）
        ├─ blueprintId: 保留
        └─ origin: command.origin ?? NOVU_CLOUD
```

### 2.2 processBlueprint 核心代码

**入口判断** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:61-62`

```typescript
const blueprintCommand = await this.processBlueprint(usecaseCommand);
const command = blueprintCommand ?? usecaseCommand;  // ⚠️ blueprintCommand 优先
```

**覆盖规则** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:551-575`

```typescript
private async processBlueprint(command: CreateWorkflowCommandV0) {
  if (!command.blueprintId) return null;

  // ... handleGroup, normalizeSteps ...

  return CreateWorkflowCommandV0.create({
    // ... 其他字段保留原值
    blueprintId: command.blueprintId,          // ✅ 保留蓝图关联
    type: ResourceTypeEnum.REGULAR,             // ⚠️ 强制覆盖为 REGULAR（优先级最高）
    origin: command.origin ?? ResourceOriginEnum.NOVU_CLOUD,  // ⚠️ 有传参用传参，无传参默认 NOVU_CLOUD
  });
}
```

### 2.3 不同入口调用 processBlueprint 后的结果

| 调用入口 | blueprintId | 传入 type | 传入 origin | processBlueprint 后 type | processBlueprint 后 origin |
|---------|-------------|-----------|------------|-------------------------|---------------------------|
| `POST /v1/workflows` | 有 | `REGULAR` | `NOVU_CLOUD_V1` | `REGULAR`（不变） | `NOVU_CLOUD_V1`（不变） |
| `POST /v1/workflows` | 无 | `REGULAR` | `NOVU_CLOUD_V1` | `REGULAR`（不变） | `NOVU_CLOUD_V1`（不变） |
| `POST /v2/workflows` | 有 | `BRIDGE` | `NOVU_CLOUD` | ❗ `REGULAR`（被覆盖） | `NOVU_CLOUD`（不变） |
| `POST /v2/workflows` | 无 | `BRIDGE` | `NOVU_CLOUD` | `BRIDGE`（不变） | `NOVU_CLOUD`（不变） |
| Bridge API 注册 | 有 | `BRIDGE` | `EXTERNAL` | ❗ `REGULAR`（被覆盖） | `EXTERNAL`（不变） |
| 手动传 origin=EXTERNAL | 有 | 任意 | `EXTERNAL` | ❗ `REGULAR`（被覆盖） | `EXTERNAL`（不变） |

**关键点**：
1. `processBlueprint` 的 `type` 覆盖优先级最高，**无论传入什么 type，都会被强制改为 REGULAR**
2. `origin` 遵循"有传参用传参，无传参默认 NOVU_CLOUD"
3. v1 入口传入的 `type=REGULAR, origin=NOVU_CLOUD_V1` 不会被 processBlueprint 改变（因为本来就是 REGULAR）

---

## 三、computeOrigin 的推算优先级

### 3.1 完整推算逻辑

**定义位置** `libs/application-generic/src/utils/notification-template-mapper.ts:120-128`

```typescript
function computeOrigin(template: NotificationTemplateEntity): ResourceOriginEnum {
  // 优先级 1：旧数据兼容（type 和 origin 都未定义）
  if (typeof template.type === 'undefined' && typeof template.origin === 'undefined') {
    return ResourceOriginEnum.NOVU_CLOUD_V1;
  }

  // 优先级 2：type === REGULAR → NOVU_CLOUD_V1
  return template?.type === ResourceTypeEnum.REGULAR
    ? ResourceOriginEnum.NOVU_CLOUD_V1  // ⚠️ 只要是 REGULAR，不管 origin 是什么都返回 V1
    // 优先级 3：type !== REGULAR → 使用 origin 字段，默认 EXTERNAL
    : template.origin || ResourceOriginEnum.EXTERNAL;
}
```

### 3.2 优先级总结

| 优先级 | 条件 | 返回值 |
|-------|------|--------|
| 1（最高） | `type === undefined && origin === undefined` | `NOVU_CLOUD_V1` |
| 2 | `type === REGULAR`（无论 origin 是什么） | `NOVU_CLOUD_V1` |
| 3（最低） | `type !== REGULAR` | `origin` 字段值，默认 `EXTERNAL` |

### 3.3 数据库存储值 vs DTO 输出值对比

| 场景 | 数据库 `type` | 数据库 `origin` | `computeOrigin` 返回 | DTO 输出 `origin` |
|------|--------------|----------------|---------------------|------------------|
| `POST /v1/workflows`（无 blueprintId） | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` |
| `POST /v2/workflows`（无 blueprintId） | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | `NOVU_CLOUD` |
| `POST /v2/workflows`（有 blueprintId） | ❗ `REGULAR` | `NOVU_CLOUD` | ❗ `NOVU_CLOUD_V1` | ❗ `NOVU_CLOUD_V1` |
| `POST /v1/workflows`（有 blueprintId） | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` |
| Bridge API 注册（无 blueprintId） | `BRIDGE` | `EXTERNAL` | `EXTERNAL` | `EXTERNAL` |
| Bridge API 从蓝图创建 | ❗ `REGULAR` | `EXTERNAL` | ❗ `NOVU_CLOUD_V1` | ❗ `NOVU_CLOUD_V1` |
| 历史遗留数据 | `undefined` | `undefined` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` |
| SyncToEnvironment 同步后 | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | `NOVU_CLOUD` |

---

## 四、SyncToEnvironment 白名单判定链（可执行口径）

### 4.1 判定链总览

```
PUT /v2/workflows/:workflowId/sync
    ↓
SyncToEnvironment.execute()
    ├─ 前置校验 1：不能同步到相同环境
    ├─ 前置校验 2：目标环境存在
    ├─ 查询源工作流完整信息（返回 DTO）
    │   └─ notification-template-mapper → computeOrigin() 推算 origin
    │
    ├─ 可同步性判定（DTO origin 判定）
    │   └─ isSyncable(sourceWorkflow)
    │       └─ SYNCABLE_WORKFLOW_ORIGINS.includes(workflow.origin)
    │           └─ 白名单 = [NOVU_CLOUD]
    │
    ├─ 判定通过 → 继续同步流程（同步后重置 origin=NOVU_CLOUD, type=BRIDGE）
    └─ 判定失败 → 抛出 WorkflowNotSyncableException
```

### 4.2 关键代码证据

**白名单定义** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47`

```typescript
export const SYNCABLE_WORKFLOW_ORIGINS = [ResourceOriginEnum.NOVU_CLOUD];
```

**判定方法** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:28-30`

```typescript
private isSyncable(workflow: WorkflowResponseDto): boolean {
  return SYNCABLE_WORKFLOW_ORIGINS.includes(workflow.origin);  // ⚠️ 判定使用的是 DTO 的 origin
}
```

**触发判定** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:90-92`

```typescript
if (!this.isSyncable(sourceWorkflow)) {
  throw new WorkflowNotSyncableException(sourceWorkflow);
}
```

**同步后重置** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:271-311`

```typescript
private async mapWorkflowToCreateWorkflowDto(...): Promise<UpsertWorkflowDataCommand> {
  return {
    // ...
    origin: ResourceOriginEnum.NOVU_CLOUD,  // ⚠️ 强制重置为 NOVU_CLOUD
    // ...
  };
}
```

### 4.3 可同步 / 不可同步场景矩阵（完整校准版）

| 场景 | 创建入口 | blueprintId | 数据库 type | 数据库 origin | computeOrigin 返回 | isSyncable | 能否同步 |
|------|---------|-------------|------------|--------------|-------------------|------------|---------|
| **场景 1**：v2 手动创建 | `POST /v2/workflows` | 无 | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | ✅ true | ✅ 可以 |
| **场景 2**：v1 手动创建 | `POST /v1/workflows` | 无 | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | ❌ false | ❌ 拒绝 |
| **场景 3**：v2 从蓝图创建 | `POST /v2/workflows` | 有 | ❗ `REGULAR` | `NOVU_CLOUD` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 拒绝 |
| **场景 4**：v1 从蓝图创建 | `POST /v1/workflows` | 有 | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | ❌ false | ❌ 拒绝 |
| **场景 5**：从蓝图创建（传 origin=EXTERNAL） | 任意 | 有 | `REGULAR` | `EXTERNAL` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 拒绝 |
| **场景 6**：Bridge API 注册（无 blueprintId） | Bridge | 无 | `BRIDGE` | `EXTERNAL` | `EXTERNAL` | ❌ false | ❌ 拒绝 |
| **场景 7**：Bridge API 从蓝图创建 | Bridge | 有 | `REGULAR` | `EXTERNAL` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 拒绝 |
| **场景 8**：历史遗留数据 | - | - | `undefined` | `undefined` | `NOVU_CLOUD_V1` | ❌ false | ❌ 拒绝 |
| **场景 9**：SyncToEnvironment 同步后再次同步 | - | - | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | ✅ true | ✅ 可以 |

### 4.4 关键结论

1. **唯一可同步场景**：`type = BRIDGE` 且 `origin = NOVU_CLOUD`（场景 1 和场景 9）
2. **所有从蓝图创建的工作流都不可同步**：因为 `type` 被强制设为 `REGULAR`，`computeOrigin` 必定返回 `NOVU_CLOUD_V1`
3. **所有 v1 入口创建的工作流都不可同步**：因为 `type` 硬编码为 `REGULAR`
4. **同步后可再次同步**：因为同步时重置为 `type=BRIDGE, origin=NOVU_CLOUD`
5. **SyncToEnvironment 接口仅在 v2 路由可用**：`PUT /v2/workflows/:workflowId/sync`

---

## 五、最小验证路径（路由与类名校准版）

### 5.1 验证路径 1：v1 入口确实可写入 NOVU_CLOUD_V1

```
步骤 1：调用 POST /v1/workflows
        Body: { "name": "test-v1", "notificationGroupId": "<group-id>", "steps": [...] }
步骤 2：查询数据库 notification_templates 集合
步骤 3：验证：
        - type === "REGULAR"
        - origin === "novu-cloud-v1"

关键证据：
  类名：WorkflowControllerV1
  文件：apps/api/src/app/workflows-v1/workflow-v1.controller.ts:237-238
  → type: ResourceTypeEnum.REGULAR
  → origin: ResourceOriginEnum.NOVU_CLOUD_V1
```

### 5.2 验证路径 2：v2 从蓝图创建会被强制改为 REGULAR

```
步骤 1：调用 POST /v2/workflows
        Body: { "name": "test-blueprint", "blueprintId": "<blueprint-id>", ... }
步骤 2：查询数据库
步骤 3：验证：
        - type === "REGULAR" （尽管 v2 入口默认 BRIDGE）
        - origin === "novu-cloud"
        - blueprintId === "<blueprint-id>"

关键证据：
  类名：WorkflowController
  文件：libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:572
  → type: ResourceTypeEnum.REGULAR
```

### 5.3 验证路径 3：type=REGULAR 时 computeOrigin 总是返回 NOVU_CLOUD_V1

```
步骤 1：创建工作流，设置数据库：
        db.notification_templates.updateOne(
          { _id: "<id>" },
          { $set: { type: "REGULAR", origin: "novu-cloud" } }
        )
步骤 2：调用 GET /v2/workflows/<id> 或 GET /v1/workflows/<id>
步骤 3：验证响应中 origin === "novu-cloud-v1"（不是 novu-cloud）

关键证据：
  文件：libs/application-generic/src/utils/notification-template-mapper.ts:126-128
  → template?.type === REGULAR ? NOVU_CLOUD_V1 : ...
```

### 5.4 验证路径 4：从蓝图创建的工作流不能同步

```
步骤 1：从蓝图创建一个工作流（场景 3）
步骤 2：调用 PUT /v2/workflows/<workflow-id>/sync
        Body: { "targetEnvironmentId": "<env-id>" }
步骤 3：验证返回 400 Bad Request
        错误码：WorkflowNotSyncableException

关键证据：
  路由：PUT /v2/workflows/:workflowId/sync
  类名：WorkflowController
  文件：apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47
  → SYNCABLE_WORKFLOW_ORIGINS = [NOVU_CLOUD]
  文件：apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:90-92
  → if (!this.isSyncable(sourceWorkflow)) { throw ... }
```

### 5.5 验证路径 5：同步后的工作流可以再次同步

```
步骤 1：POST /v2/workflows 创建一个工作流（场景 1，可同步）
步骤 2：PUT /v2/workflows/<id>/sync 同步到另一个环境（成功）
步骤 3：在目标环境查询工作流 GET /v2/workflows/<id>
        - 验证 type === "BRIDGE"
        - 验证 origin === "novu-cloud"
步骤 4：再次 PUT /v2/workflows/<id>/sync 同步到第三个环境（成功）

关键证据：
  文件：apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:277, 298
  → origin: ResourceOriginEnum.NOVU_CLOUD
  文件：libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:155-156
  → type: BRIDGE, origin: NOVU_CLOUD
```

### 5.6 验证路径 6：v1 与 v2 路由正确性

```
步骤 1：POST /v1/workflows 创建（成功）
        验证响应头 X-Route-Matched 或日志中路由为 /v1/workflows
步骤 2：POST /v2/workflows 创建（成功）
        验证路由为 /v2/workflows
步骤 3：PUT /v2/workflows/<id>/sync 调用（存在）
步骤 4：PUT /v1/workflows/<id>/sync 调用（404，因为 v1 controller 没有 sync 方法）

关键证据：
  全局版本配置：apps/api/src/bootstrap.ts:76-80
  v1 Controller：@Controller('/workflows') + 默认版本 1 → /v1/workflows
  v2 Controller：@Controller({ path: '/workflows', version: '2' }) → /v2/workflows
```

---

## 六、最终可执行判定口径（前后一致）

### 6.1 Origin 字段三层模型

| 层级 | 取值来源 | 决定因素 | 用途 |
|------|---------|---------|------|
| **L1 数据库存储** | `notification_templates.type` + `notification_templates.origin` | 创建入口（v1/v2/Bridge） + processBlueprint 规则 | 原始数据 |
| **L2 DTO 输出** | `computeOrigin(L1)` | L1 的 type 优先级高于 origin | 对外展示、前端判定、API 响应 |
| **L3 业务判定** | `SYNCABLE_WORKFLOW_ORIGINS.includes(L2)` | L2 的 origin 值 | SyncToEnvironment 白名单判定 |

### 6.2 SyncToEnvironment 可同步判定流程图

```
PUT /v2/workflows/:workflowId/sync
    ↓
获取工作流 DTO（L2 origin）
    ↓
L2 origin === NOVU_CLOUD ?
    ├─ 是 → 检查其他前置条件 → 允许同步 → 同步后重置 L1: type=BRIDGE, origin=NOVU_CLOUD
    └─ 否 → 拒绝同步，抛出 WorkflowNotSyncableException
```

### 6.3 快速判断法则

**只要数据库中 `type = REGULAR`，无论 `origin` 是什么，都不能通过 SyncToEnvironment 同步。**

**原因**：`computeOrigin` 看到 `type=REGULAR` 会直接返回 `NOVU_CLOUD_V1`，不在白名单 `[NOVU_CLOUD]` 中。

### 6.4 路由判定法则

| 操作 | 可用路由 | 不可用路由 |
|------|---------|-----------|
| 创建工作流（v1 模式） | `POST /v1/workflows` | `POST /workflows-v1` |
| 创建工作流（v2 模式） | `POST /v2/workflows` | `POST /workflows-v2` |
| 同步工作流 | `PUT /v2/workflows/:id/sync` | `PUT /v1/workflows/:id/sync`、`PUT /workflows-v2/:id/sync` |

---

## 七、核心代码文件索引（事实收敛版）

| 模块 | 文件路径 | 类名 / 方法 | 关键行 | 职责 |
|------|---------|------------|--------|------|
| 全局版本配置 | `apps/api/src/bootstrap.ts` | `bootstrap()` | 76-80 | NestJS 版本控制配置 |
| v1 Controller | `apps/api/src/app/workflows-v1/workflow-v1.controller.ts` | `WorkflowControllerV1` | 63-67, 237-238 | v1 路由 + 创建时 type/origin |
| v2 Controller | `apps/api/src/app/workflows-v2/workflow.controller.ts` | `WorkflowController` | 77-81, 124-146 | v2 路由 + sync 接口 |
| Upsert 创建 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts` | `buildCreateWorkflowCommand()` | 155-156 | v2 创建时 type/origin |
| Upsert 更新 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts` | `buildUpdateWorkflowCommand()` | 190 | v2 更新时 type |
| processBlueprint | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts` | `processBlueprint()` | 572-573 | 蓝图派生覆盖规则 |
| computeOrigin | `libs/application-generic/src/utils/notification-template-mapper.ts` | `computeOrigin()` | 120-128 | origin 推算优先级 |
| 同步白名单 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | `SYNCABLE_WORKFLOW_ORIGINS` | 47 | 白名单定义 |
| 同步判定 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | `isSyncable()` | 28-30, 90-92 | 同步判定逻辑 |
| 同步重置 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | `mapWorkflowToCreateWorkflowDto()` | 277, 298 | 同步后重置 origin |
| Origin 枚举 | `packages/shared/src/types/general.ts` | `ResourceOriginEnum` | 9-13 | 枚举值定义 |
