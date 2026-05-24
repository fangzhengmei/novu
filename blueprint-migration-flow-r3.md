# Blueprint 派生工作流的 Origin 分流与可同步判定分析 - R3

本文档聚焦于蓝图派生工作流的 origin 字段在不同创建入口下的分流逻辑，以及 `SyncToEnvironment` 可同步判定的完整判断链。

---

## 一、ResourceOriginEnum 全量定义与语义

**定义位置** `packages/shared/src/types/general.ts:9-13`

```typescript
export enum ResourceOriginEnum {
  NOVU_CLOUD = 'novu-cloud',        // Novu 平台 v2 原生创建（Bridge 架构）
  NOVU_CLOUD_V1 = 'novu-cloud-v1',  // Novu 平台 v1 遗留工作流（Regular 架构）
  EXTERNAL = 'external',            // 外部 Bridge 来源
}
```

**Origin 推算规则（兼容旧数据）** `libs/application-generic/src/utils/notification-template-mapper.ts:120-128`

```typescript
function computeOrigin(template: NotificationTemplateEntity): ResourceOriginEnum {
  // 旧数据兼容：type 和 origin 都未定义 → 判定为 V1
  if (typeof template.type === 'undefined' && typeof template.origin === 'undefined') {
    return ResourceOriginEnum.NOVU_CLOUD_V1;
  }

  // type === REGULAR → 判定为 V1
  return template?.type === ResourceTypeEnum.REGULAR
    ? ResourceOriginEnum.NOVU_CLOUD_V1
    : template.origin || ResourceOriginEnum.EXTERNAL;
}
```

---

## 二、v1 与 v2 创建入口的 Origin 初始化逻辑

### 2.1 v2 工作流创建入口（/workflows）

**Controller 层** `apps/api/src/app/workflows-v2/workflow.controller.ts:112-121`

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

**UpsertWorkflow 内部 - 创建分支** `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:140-170`

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

**UpsertWorkflow 内部 - 更新分支** `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:172-201`

```typescript
private async buildUpdateWorkflowCommand(...): Promise<UpdateWorkflowCommandV0> {
  return {
    // ...
    type: ResourceTypeEnum.BRIDGE,  // ⚠️ 硬编码 BRIDGE
    // ⚠️ 注意：origin 字段未设置，保留数据库中原有值
    // ...
  };
}
```

### 2.2 v1 工作流的来源路径

v1 工作流不会通过当前 API 创建，而是通过以下路径产生：

1. **历史遗留数据**：系统升级前创建的工作流，`type` 为 `REGULAR` 或未定义
2. **从蓝图创建**：通过 `processBlueprint` 创建时 `type` 被强制设为 `REGULAR`（见下文）
3. **外部 Bridge 注册**：通过 Bridge API 注册，`origin` 设为 `EXTERNAL`

### 2.3 各入口 Origin 初始化对比表

| 创建入口 | type | origin | 说明 |
|---------|------|--------|------|
| v2 Controller (`POST /workflows`) | `BRIDGE` (硬编码) | `NOVU_CLOUD` (硬编码) | 标准 v2 创建路径 |
| UpsertWorkflow - 创建分支 | `BRIDGE` (硬编码) | `NOVU_CLOUD` (硬编码) | v2 内部创建 |
| UpsertWorkflow - 更新分支 | `BRIDGE` (硬编码) | **保留原值** | 更新时不修改 origin |
| processBlueprint (蓝图派生) | `REGULAR` (强制) | `command.origin ?? NOVU_CLOUD` | 见下文详细分析 |
| Bridge API 注册 | `BRIDGE` | `EXTERNAL` | 外部 Bridge 注册 |
| 历史遗留数据 | `undefined` / `REGULAR` | `undefined` | computeOrigin 推算为 `NOVU_CLOUD_V1` |

---

## 三、processBlueprint 对 Origin 的传递规则与覆盖逻辑

### 3.1 完整调用链

```
前端发起从蓝图创建请求
    ↓ 携带 blueprintId 和可选的 origin
CreateWorkflowV0.execute()
    ↓ 检查 blueprintId 是否存在
processBlueprint()  // 仅当 blueprintId 存在时触发
    ├─ handleGroup()        // notificationGroup 处理
    ├─ normalizeSteps()     // feedId 归零
    └─ 创建新 Command，覆盖规则：
        ├─ type: REGULAR            // 强制覆盖为 REGULAR
        ├─ blueprintId: 保留        // 保留来源关联
        └─ origin: command.origin ?? NOVU_CLOUD  // 有则用，无则默认
```

### 3.2 processBlueprint 核心代码

**入口判断** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:551-552`

```typescript
private async processBlueprint(command: CreateWorkflowCommandV0) {
  if (!command.blueprintId) return null;  // ⚠️ 只有携带 blueprintId 才进入此分支
  // ...
}
```

**Origin 传递规则** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:557-574`

```typescript
return CreateWorkflowCommandV0.create({
  // ...
  blueprintId: command.blueprintId,          // ✅ 保留蓝图关联
  __source: command.__source,
  type: ResourceTypeEnum.REGULAR,             // ⚠️ 强制覆盖为 REGULAR（无论原值是什么）
  origin: command.origin ?? ResourceOriginEnum.NOVU_CLOUD,  // ⚠️ 有传参用传参，无传参默认 NOVU_CLOUD
});
```

### 3.3 关键覆盖逻辑分析

| 字段 | 覆盖行为 | 设计意图 |
|------|---------|---------|
| `type` | **强制 REGULAR** | 从蓝图创建的工作流永远是 REGULAR 类型，而非 BRIDGE |
| `origin` | `command.origin ?? NOVU_CLOUD` | 允许调用方指定 origin，默认 NOVU_CLOUD |
| `blueprintId` | 原样保留 | 溯源用，不影响运行时 |

**实际影响**：
- 从蓝图创建的工作流 `type = REGULAR`，根据 `computeOrigin` 规则，**对外展示时会被推算为 NOVU_CLOUD_V1**
- 但数据库中存储的 `origin` 字段值取决于创建时是否传参

---

## 四、SyncToEnvironment 的 SYNCABLE_WORKFLOW_ORIGINS 白名单判定链

### 4.1 判定链总览

```
SyncToEnvironment.execute()
    ├─ 前置校验 1：不能同步到相同环境
    ├─ 前置校验 2：目标环境存在
    ├─ 查询源工作流完整信息
    │
    ├─ ⚠️ 可同步性判定（第 90-92 行）
    │   └─ isSyncable(sourceWorkflow)
    │       └─ SYNCABLE_WORKFLOW_ORIGINS.includes(workflow.origin)
    │           └─ [NOVU_CLOUD]  ⚠️ 白名单仅此一个
    │
    ├─ 判定通过 → 继续同步流程
    └─ 判定失败 → 抛出 WorkflowNotSyncableException
```

### 4.2 白名单定义与判定逻辑

**白名单常量** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47`

```typescript
export const SYNCABLE_WORKFLOW_ORIGINS = [ResourceOriginEnum.NOVU_CLOUD];
```

**判定方法** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:28-30`

```typescript
private isSyncable(workflow: WorkflowResponseDto): boolean {
  return SYNCABLE_WORKFLOW_ORIGINS.includes(workflow.origin);
}
```

**触发判定** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:90-92`

```typescript
if (!this.isSyncable(sourceWorkflow)) {
  throw new WorkflowNotSyncableException(sourceWorkflow);
}
```

### 4.3 前端双重校验

**前端 Hook** `apps/dashboard/src/hooks/use-sync-workflow.tsx:30-32`

```typescript
const isSyncable = useMemo(
  () => workflow.origin === ResourceOriginEnum.NOVU_CLOUD && workflow.status !== WorkflowStatusEnum.ERROR,
  [workflow.origin, workflow.status]
);
```

**前端 UI 禁用** `apps/dashboard/src/components/workflow-row.tsx:565-570`

```typescript
if (!isSyncable) {
  return (
    <Tooltip>
      <TooltipTrigger>
        <DropdownMenuItem disabled>
          <LuBookUp2 />
          Sync to environment
        </DropdownMenuItem>
      </TooltipTrigger>
      {/* ... */}
    </Tooltip>
  );
}
```

### 4.4 同步后 Origin 重置

无论源工作流 origin 是什么，同步到目标环境后都会被重置为 `NOVU_CLOUD`：

**创建 DTO** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:271-290`

```typescript
private async mapWorkflowToCreateWorkflowDto(...): Promise<UpsertWorkflowDataCommand> {
  return {
    // ...
    origin: ResourceOriginEnum.NOVU_CLOUD,  // ⚠️ 强制重置为 NOVU_CLOUD
    // ...
  };
}
```

**更新 DTO** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:292-311`

```typescript
private async mapWorkflowToUpdateWorkflowDto(...): Promise<UpsertWorkflowDataCommand> {
  return {
    origin: ResourceOriginEnum.NOVU_CLOUD,  // ⚠️ 强制重置为 NOVU_CLOUD
    // ...
  };
}
```

---

## 五、可同步 / 不可同步场景矩阵

### 5.1 场景判定表

| 场景 | 数据库 origin | 数据库 type | computeOrigin 结果 | isSyncable 结果 | 能否同步 |
|------|--------------|------------|-------------------|----------------|---------|
| **场景 1**：v2 手动创建 | `NOVU_CLOUD` | `BRIDGE` | `NOVU_CLOUD` | ✅ true | ✅ 可以 |
| **场景 2**：从蓝图创建（不传 origin） | `NOVU_CLOUD` | `REGULAR` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 3**：从蓝图创建（传 origin=NOVU_CLOUD） | `NOVU_CLOUD` | `REGULAR` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 4**：从蓝图创建（传 origin=EXTERNAL） | `EXTERNAL` | `REGULAR` | ❗ `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 5**：v1 历史遗留 | `undefined` | `REGULAR` | `NOVU_CLOUD_V1` | ❌ false | ❌ 被拒绝 |
| **场景 6**：外部 Bridge 注册 | `EXTERNAL` | `BRIDGE` | `EXTERNAL` | ❌ false | ❌ 被拒绝 |
| **场景 7**：同步后再次同步 | `NOVU_CLOUD` | `BRIDGE` | `NOVU_CLOUD` | ✅ true | ✅ 可以 |

### 5.2 关键场景详解

#### 场景 2-4：从蓝图创建的工作流（全部不可同步）

**为什么即使 origin=NOVU_CLOUD 也不能同步？**

因为 `computeOrigin` 的推算规则优先级更高：
1. 数据库中 `type = REGULAR`（processBlueprint 强制设置）
2. `computeOrigin` 看到 `type === REGULAR` → 返回 `NOVU_CLOUD_V1`
3. `isSyncable` 检查返回的 `NOVU_CLOUD_V1` → 不在白名单中 → 拒绝

**代码证据** `libs/application-generic/src/utils/notification-template-mapper.ts:126-128`

```typescript
return template?.type === ResourceTypeEnum.REGULAR
  ? ResourceOriginEnum.NOVU_CLOUD_V1  // ⚠️ 只要是 REGULAR，不管 origin 是什么都返回 V1
  : template.origin || ResourceOriginEnum.EXTERNAL;
```

#### 场景 7：同步后的工作流可以再次同步

因为同步时 `mapWorkflowToCreateWorkflowDto` 强制设置：
- `origin: NOVU_CLOUD`
- `type: BRIDGE`（UpsertWorkflow 创建分支硬编码）

所以同步后的工作流 `type = BRIDGE`，不会被推算为 V1，可以正常同步。

### 5.3 最小验证路径

**验证场景 2：从蓝图创建的工作流不可同步**

```
1. 调用 GET /blueprints/:id 获取蓝图
2. 调用 POST /workflows 从蓝图创建工作流，携带 blueprintId
3. 查询新创建的工作流：
   - 验证 type === 'REGULAR'
   - 验证 origin === 'novu-cloud'
   - 验证对外展示的 origin 为 'novu-cloud-v1'（computeOrigin 推算）
4. 调用 POST /workflows/:id/sync-to-environment
5. 预期返回 400 Bad Request，WorkflowNotSyncableException
```

**验证场景 7：同步后的工作流可以再次同步**

```
1. 创建一个 v2 工作流（origin=NOVU_CLOUD, type=BRIDGE）
2. 同步到另一个环境（成功）
3. 在目标环境查询同步后的工作流：
   - 验证 type === 'BRIDGE'
   - 验证 origin === 'novu-cloud'
4. 再次同步到第三个环境
5. 预期成功
```

---

## 六、Origin 字段在不同层级的取值对比

| 层级 | 取值来源 | 可能值 | 用途 |
|------|---------|--------|------|
| **数据库存储** | `notification_templates.origin` | `NOVU_CLOUD` / `NOVU_CLOUD_V1` / `EXTERNAL` / `undefined` | 原始存储 |
| **数据库存储** | `notification_templates.type` | `REGULAR` / `BRIDGE` / `undefined` | 架构类型标识 |
| **DTO 输出** | `computeOrigin()` 推算 | `NOVU_CLOUD` / `NOVU_CLOUD_V1` / `EXTERNAL` | 对外展示、前端判定 |
| **Sync 判定** | DTO 中的 origin 字段 | `NOVU_CLOUD` → 通过，其他 → 拒绝 | 同步白名单校验 |

**关键注意**：
- `SyncToEnvironment` 判定使用的是 **DTO 输出的 origin**（经过 `computeOrigin` 推算），而非数据库原始字段
- 这意味着即使数据库中 `origin = NOVU_CLOUD`，只要 `type = REGULAR`，推算结果就是 `NOVU_CLOUD_V1`，同步会被拒绝

---

## 七、核心代码文件索引

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| Origin 定义 | `packages/shared/src/types/general.ts:9-13` | `ResourceOriginEnum` 定义 |
| Origin 推算 | `libs/application-generic/src/utils/notification-template-mapper.ts:120-128` | `computeOrigin` 实现 |
| v2 创建入口 | `apps/api/src/app/workflows-v2/workflow.controller.ts:112-121` | Controller 层 origin 硬编码 |
| Upsert 创建 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:140-170` | `buildCreateWorkflowCommand` 实现 |
| Upsert 更新 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:172-201` | `buildUpdateWorkflowCommand` 实现 |
| 蓝图派生 | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:551-574` | `processBlueprint` 实现 |
| 同步白名单 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47` | `SYNCABLE_WORKFLOW_ORIGINS` 定义 |
| 同步判定 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:28-30,90-92` | `isSyncable` 判定 |
| 同步重置 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:271-311` | DTO 构建时 origin 重置 |
| 前端判定 | `apps/dashboard/src/hooks/use-sync-workflow.tsx:30-32` | 前端 `isSyncable` 判定 |
| 前端禁用 | `apps/dashboard/src/components/workflow-row.tsx:565-570` | UI 禁用逻辑 |
