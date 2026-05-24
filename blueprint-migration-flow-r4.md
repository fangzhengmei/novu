# Blueprint 派生工作流的 Origin 分流与可同步判定分析 - R4（事实校准版）

本文档为 `blueprint-migration-flow-r3.md` 的事实校准修订版，重点修正了"v1 入口是历史残留"这一错误结论，重新梳理了完整的、不自相矛盾的判定链路。

---

## 一、事实校准：v1 与 v2 创建入口是并行的活跃路径

### 1.1 v1 入口（`/workflows`）

**Controller 位置** `apps/api/src/app/workflows-v1/workflow-v1.controller.ts:62-67`

```typescript
@ApiExcludeController()
@Controller('/workflows')  // ⚠️ 注意：路径是 /workflows，不是 /v1/workflows
@UseInterceptors(ClassSerializerInterceptor)
@RequireAuthentication()
@ApiTags('Workflows')
export class WorkflowControllerV1 {
```

**创建方法** `apps/api/src/app/workflows-v1/workflow-v1.controller.ts:202-241`

```typescript
@Post('')
@ApiResponse(WorkflowResponse, 201)
@ApiOperation({
  summary: 'Create workflow',
  description: `Workflows were previously named notification templates`,
})
@ExternalApiAccessible()
@UseGuards(RootEnvironmentGuard)
@RequirePermissions(PermissionsEnum.WORKFLOW_WRITE)
create(
  @UserSession() user: UserSessionData,
  @Query() query: CreateWorkflowQuery,
  @Body() body: CreateWorkflowRequestDto
): Promise<WorkflowResponse> {
  return this.createWorkflowUsecaseV0.execute(
    CreateWorkflowCommandV0.create({
      // ...
      type: ResourceTypeEnum.REGULAR,           // ⚠️ 硬编码 REGULAR
      origin: ResourceOriginEnum.NOVU_CLOUD_V1,  // ⚠️ 硬编码 NOVU_CLOUD_V1
    })
  );
}
```

**更新方法** `apps/api/src/app/workflows-v1/workflow-v1.controller.ts:102-136`

```typescript
@Put('/:workflowId')
async updateWorkflowById(...): Promise<WorkflowResponse> {
  return await this.updateWorkflowByIdUsecaseV0.execute(
    UpdateWorkflowCommandV0.create({
      // ...
      type: ResourceTypeEnum.REGULAR,  // ⚠️ 更新时也硬编码 REGULAR
      // 注意：origin 字段未设置，保留原值
    })
  );
}
```

### 1.2 v2 入口（`/workflows-v2`）

**Controller 位置** `apps/api/src/app/workflows-v2/workflow.controller.ts:26-31`

```typescript
@Controller({
  path: 'workflows-v2',
  version: '2',
})
@UseInterceptors(ClassSerializerInterceptor)
@RequireAuthentication()
export class WorkflowControllerV2 {
```

**创建方法** `apps/api/src/app/workflows-v2/workflow.controller.ts:112-121`

```typescript
await this.upsertWorkflowUseCase.execute(
  UpsertWorkflowCommand.create({
    preserveWorkflowId: true,
    workflowDto: {
      ...createWorkflowDto,
      steps: upsertSteps,
      origin: ResourceOriginEnum.NOVU_CLOUD,  // ⚠️ 硬编码 NOVU_CLOUD
    },
    user,
  })
);
```

**UpsertWorkflow 创建分支** `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:140-170`

```typescript
private async buildCreateWorkflowCommand(command: UpsertWorkflowCommand): Promise<CreateWorkflowCommandV0> {
  return {
    // ...
    type: ResourceTypeEnum.BRIDGE,        // ⚠️ 硬编码 BRIDGE
    origin: ResourceOriginEnum.NOVU_CLOUD, // ⚠️ 硬编码 NOVU_CLOUD
    // ...
  };
}
```

**UpsertWorkflow 更新分支** `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:172-200`

```typescript
private async buildUpdateWorkflowCommand(...): Promise<UpdateWorkflowCommandV0> {
  return {
    // ...
    type: ResourceTypeEnum.BRIDGE,  // ⚠️ 更新时也硬编码 BRIDGE
    // 注意：origin 字段未设置，保留原值
    // ...
  };
}
```

### 1.3 v1/v2 入口对比表（事实校准版）

| 维度 | v1 入口 (`/workflows`) | v2 入口 (`/workflows-v2`) |
|------|----------------------|-------------------------|
| Controller 类 | `WorkflowControllerV1` | `WorkflowControllerV2` |
| Controller 路径 | `apps/api/src/app/workflows-v1/` | `apps/api/src/app/workflows-v2/` |
| 路由前缀 | `/workflows`（无版本号） | `/workflows-v2`（有版本号 v2） |
| 废弃标记 | `@deprecated` + `@ApiExcludeController()` | 无 |
| 是否可调用 | ✅ 是（`@ExternalApiAccessible()`） | ✅ 是 |
| 创建时 type | `REGULAR`（硬编码） | `BRIDGE`（硬编码） |
| 创建时 origin | `NOVU_CLOUD_V1`（硬编码） | `NOVU_CLOUD`（硬编码） |
| 更新时 type | `REGULAR`（硬编码，覆盖原值） | `BRIDGE`（硬编码，覆盖原值） |
| 更新时 origin | 保留原值 | 保留原值 |
| 调用 `CreateWorkflowV0` | ✅ 直接调用 | ✅ 通过 `UpsertWorkflow` 间接调用 |
| 是否支持 `blueprintId` | ✅ 支持（第 234 行） | ✅ 支持 |

---

## 二、processBlueprint 的 origin 传递规则

### 2.1 完整调用链

```
任何入口调用 CreateWorkflowV0.execute()
    ↓
检查 usecaseCommand.blueprintId 是否存在
    ├─ 不存在 → 直接使用 usecaseCommand
    └─ 存在 → 调用 processBlueprint() 覆盖规则
        ├─ type: 强制 REGULAR
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

| 调用入口 | 传入 blueprintId | 传入 type | 传入 origin | processBlueprint 后 type | processBlueprint 后 origin |
|---------|-----------------|-----------|------------|-------------------------|---------------------------|
| v1 `POST /workflows` | 有 | `REGULAR` | `NOVU_CLOUD_V1` | `REGULAR`（不变） | `NOVU_CLOUD_V1`（不变） |
| v1 `POST /workflows` | 无 | `REGULAR` | `NOVU_CLOUD_V1` | `REGULAR`（不变） | `NOVU_CLOUD_V1`（不变） |
| v2 `POST /workflows-v2` | 有 | `BRIDGE` | `NOVU_CLOUD` | ❗ `REGULAR`（被覆盖） | `NOVU_CLOUD`（不变） |
| v2 `POST /workflows-v2` | 无 | `BRIDGE` | `NOVU_CLOUD` | `BRIDGE`（不变） | `NOVU_CLOUD`（不变） |
| Bridge API 注册 | 有 | `BRIDGE` | `EXTERNAL` | ❗ `REGULAR`（被覆盖） | `EXTERNAL`（不变） |
| 手动传 origin=EXTERNAL | 有 | 任意 | `EXTERNAL` | ❗ `REGULAR`（被覆盖） | `EXTERNAL`（不变） |

**关键点**：
1. `processBlueprint` 的 `type` 覆盖优先级最高，**无论传入什么 type，都会被强制改为 REGULAR**
2. `origin` 遵循"有传参用传参，无传参默认 NOVU_CLOUD"
3. v1 入口传入的 `type=REGULAR, origin=NOVU_CLOUD_V1` 不会被 processBlueprint 改变（因为本来就是 REGULAR）

---

## 三、computeOrigin 的推算优先级（事实校准版）

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
| v1 入口创建 | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` |
| v2 入口创建（无 blueprintId） | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | `NOVU_CLOUD` |
| v2 入口从蓝图创建（有 blueprintId） | ❗ `REGULAR` | `NOVU_CLOUD` | ❗ `NOVU_CLOUD_V1` | ❗ `NOVU_CLOUD_V1` |
| v1 入口从蓝图创建（有 blueprintId） | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` |
| Bridge API 注册（无 blueprintId） | `BRIDGE` | `EXTERNAL` | `EXTERNAL` | `EXTERNAL` |
| Bridge API 从蓝图创建 | ❗ `REGULAR` | `EXTERNAL` | ❗ `NOVU_CLOUD_V1` | ❗ `NOVU_CLOUD_V1` |
| 历史遗留数据 | `undefined` | `undefined` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` |
| SyncToEnvironment 同步后 | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | `NOVU_CLOUD` |

---

## 四、SyncToEnvironment 白名单判定链（完整可执行口径）

### 4.1 判定链总览

```
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

| 场景 | 数据库 type | 数据库 origin | computeOrigin 返回 | isSyncable | 能否同步 |
|------|------------|--------------|-------------------|------------|---------|
| **场景 1**：v2 手动创建（无 blueprintId） | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | ✅ true | ✅ 可以 |
| **场景 2**：v1 手动创建（无 blueprintId） | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 3**：v2 从蓝图创建（有 blueprintId） | ❗ `REGULAR` | `NOVU_CLOUD` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 4**：v1 从蓝图创建（有 blueprintId） | `REGULAR` | `NOVU_CLOUD_V1` | `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 5**：从蓝图创建（传 origin=EXTERNAL） | `REGULAR` | `EXTERNAL` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 6**：Bridge API 注册（无 blueprintId） | `BRIDGE` | `EXTERNAL` | `EXTERNAL` | ❌ false | ❌ 被拒绝 |
| **场景 7**：Bridge API 从蓝图创建 | `REGULAR` | `EXTERNAL` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 8**：历史遗留数据 | `undefined` | `undefined` | `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 9**：SyncToEnvironment 同步后再次同步 | `BRIDGE` | `NOVU_CLOUD` | `NOVU_CLOUD` | ✅ true | ✅ 可以 |

### 4.4 关键结论

1. **唯一可同步场景**：`type = BRIDGE` 且 `origin = NOVU_CLOUD`（场景 1 和场景 9）
2. **所有从蓝图创建的工作流都不可同步**：因为 `type` 被强制设为 `REGULAR`，`computeOrigin` 必定返回 `NOVU_CLOUD_V1`
3. **所有 v1 入口创建的工作流都不可同步**：因为 `type` 硬编码为 `REGULAR`
4. **同步后可再次同步**：因为同步时重置为 `type=BRIDGE, origin=NOVU_CLOUD`

---

## 五、最小验证路径与关键证据定位

### 5.1 验证路径 1：v1 入口确实可写入 NOVU_CLOUD_V1

```
步骤 1：调用 POST /workflows（v1 入口）
        Body: { "name": "test-v1", "notificationGroupId": "<group-id>", "steps": [...] }
步骤 2：查询数据库 notification_templates 集合
步骤 3：验证：
        - type === "REGULAR"
        - origin === "novu-cloud-v1"

关键证据：
  apps/api/src/app/workflows-v1/workflow-v1.controller.ts:237-238
  → type: ResourceTypeEnum.REGULAR
  → origin: ResourceOriginEnum.NOVU_CLOUD_V1
```

### 5.2 验证路径 2：v2 从蓝图创建会被强制改为 REGULAR

```
步骤 1：调用 POST /workflows-v2（v2 入口）
        Body: { "name": "test-blueprint", "blueprintId": "<blueprint-id>", ... }
步骤 2：查询数据库
步骤 3：验证：
        - type === "REGULAR" （尽管 v2 入口默认 BRIDGE）
        - origin === "novu-cloud"
        - blueprintId === "<blueprint-id>"

关键证据：
  libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:572
  → type: ResourceTypeEnum.REGULAR
```

### 5.3 验证路径 3：type=REGULAR 时 computeOrigin 总是返回 NOVU_CLOUD_V1

```
步骤 1：创建工作流，设置数据库：
        db.notification_templates.updateOne(
          { _id: "<id>" },
          { $set: { type: "REGULAR", origin: "novu-cloud" } }
        )
步骤 2：调用 GET /workflows/<id>
步骤 3：验证响应中 origin === "novu-cloud-v1"（不是 novu-cloud）

关键证据：
  libs/application-generic/src/utils/notification-template-mapper.ts:126-128
  → template?.type === REGULAR ? NOVU_CLOUD_V1 : ...
```

### 5.4 验证路径 4：从蓝图创建的工作流不能同步

```
步骤 1：从蓝图创建一个工作流（场景 3）
步骤 2：调用 POST /workflows-v2/<workflow-id>/sync-to-environment
步骤 3：验证返回 400 Bad Request
        错误码：WorkflowNotSyncableException

关键证据：
  apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47
  → SYNCABLE_WORKFLOW_ORIGINS = [NOVU_CLOUD]
  apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:90-92
  → if (!this.isSyncable(sourceWorkflow)) { throw ... }
```

### 5.5 验证路径 5：同步后的工作流可以再次同步

```
步骤 1：v2 手动创建一个工作流（场景 1，可同步）
步骤 2：同步到另一个环境（成功）
步骤 3：在目标环境查询工作流
        - 验证 type === "BRIDGE"
        - 验证 origin === "novu-cloud"
步骤 4：再次同步到第三个环境（成功）

关键证据：
  apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:277, 298
  → origin: ResourceOriginEnum.NOVU_CLOUD
  libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:155-156
  → type: BRIDGE, origin: NOVU_CLOUD
```

---

## 六、不自相矛盾的可执行判定口径

### 6.1 Origin 字段三层模型

| 层级 | 取值来源 | 决定因素 | 用途 |
|------|---------|---------|------|
| **L1 数据库存储** | `notification_templates.type` + `notification_templates.origin` | 创建入口 + processBlueprint 规则 | 原始数据 |
| **L2 DTO 输出** | `computeOrigin(L1)` | L1 的 type 优先级高于 origin | 对外展示、前端判定 |
| **L3 业务判定** | `SYNCABLE_WORKFLOW_ORIGINS.includes(L2)` | L2 的 origin 值 | SyncToEnvironment 白名单 |

### 6.2 SyncToEnvironment 可同步判定流程图

```
开始
  ↓
获取工作流 DTO（L2 origin）
  ↓
L2 origin === NOVU_CLOUD ?
  ├─ 是 → 检查其他前置条件 → 允许同步 → 同步后重置 L1: type=BRIDGE, origin=NOVU_CLOUD
  └─ 否 → 拒绝同步，抛出 WorkflowNotSyncableException
```

### 6.3 快速判断法则

**只要数据库中 `type = REGULAR`，无论 `origin` 是什么，都不能同步。**

**原因**：`computeOrigin` 看到 `type=REGULAR` 会直接返回 `NOVU_CLOUD_V1`，不在白名单中。

---

## 七、核心代码文件索引（校准版）

| 模块 | 文件路径 | 职责 | 关键行 |
|------|---------|------|--------|
| v1 Controller | `apps/api/src/app/workflows-v1/workflow-v1.controller.ts` | v1 入口实现 | 237-238 |
| v2 Controller | `apps/api/src/app/workflows-v2/workflow.controller.ts` | v2 入口实现 | 112-121 |
| Upsert 创建 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts` | v2 创建分支 | 155-156 |
| Upsert 更新 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts` | v2 更新分支 | 190 |
| processBlueprint | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts` | 蓝图派生逻辑 | 572-573 |
| computeOrigin | `libs/application-generic/src/utils/notification-template-mapper.ts` | origin 推算 | 126-128 |
| 同步白名单 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | 白名单定义 | 47 |
| 同步判定 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | 判定逻辑 | 28-30, 90-92 |
| 同步重置 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | 重置 origin | 277, 298 |
| Origin 定义 | `packages/shared/src/types/general.ts` | 枚举定义 | 9-13 |
