# Blueprint 导入导出与跨环境迁移协同链路分析 - R2 补充版

本文档为 `blueprint-migration-flow.md` 的补充修订版，重点分析容易误判的实现细节。

---

## 一、Promote 与 SyncToEnvironment 的适用边界

### 1.1 核心差异对比表

| 维度 | Promote (变更提升) | SyncToEnvironment (同步到环境) |
|------|-------------------|-------------------------------|
| **触发场景** | Development → Production 环境正式发布 | 手动跨环境同步（如 Production → Production 其他环境） |
| **数据机制** | Change 变更记录 + `applyDiff` 差异合并 | 完整实体复制 + ID 重映射 |
| **入口链路** | `ApplyChange` → `PromoteChangeToEnvironment` → `PromoteNotificationTemplateChange` | `SyncToEnvironmentUseCase` → `UpsertWorkflow` |
| **Origin 限制** | 无 Origin 限制 | **仅限** `ResourceOriginEnum.NOVU_CLOUD` |
| **蓝图相关** | 可将蓝图从 Development 发布到 Production（蓝图组织内部） | 用于同步普通工作流，不涉及蓝图定义本身 |
| **是否创建变更记录** | 是（变更记录是数据源） | 否（直接创建/更新） |
| **事务保证** | 变更应用失败自动回滚 `enabled: false` | 依赖数据库事务 |

### 1.2 Origin 限制详解

**SyncToEnvironment 的白名单机制** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47`

```typescript
export const SYNCABLE_WORKFLOW_ORIGINS = [ResourceOriginEnum.NOVU_CLOUD];

private isSyncable(workflow: WorkflowResponseDto): boolean {
  return SYNCABLE_WORKFLOW_ORIGINS.includes(workflow.origin);
}

async execute(command: SyncToEnvironmentCommand) {
  // ... 前置校验
  if (!this.isSyncable(sourceWorkflow)) {
    throw new WorkflowNotSyncableException();
  }
}
```

**ResourceOriginEnum 定义** `packages/shared/src/types/general.ts:9-14`

```typescript
export enum ResourceOriginEnum {
  NOVU_CLOUD = 'novu-cloud',        // Novu 平台原生创建
  NOVU_CLOUD_V1 = 'novu-cloud-v1',  // Novu 平台 v1 版本
  EXTERNAL = 'external',            // 外部 Bridge 来源
}
```

**设计意图**：
- `SyncToEnvironment` 仅支持 Novu 云原生工作流（非 Bridge 工作流）
- `Promote` 流程支持所有类型工作流的跨环境发布，因为它走的是正式变更发布通道

### 1.3 blueprintId 与 Origin 的创建链路关系

**从蓝图创建工作流的完整链路**：

```
蓝图查询 (GET /blueprints/:id)
    ↓
前端组装创建请求，携带 blueprintId 字段
    ↓
CreateWorkflowV0.execute()
    ↓
processBlueprint()  // 仅当 blueprintId 存在时触发
    ├─ handleGroup()        // 处理 notificationGroup
    ├─ normalizeSteps()     // feedId 归零
    └─ 创建新 Command，设置：
        ├─ blueprintId: command.blueprintId         // 保留来源蓝图ID
        ├─ type: ResourceTypeEnum.REGULAR           // 强制 REGULAR
        └─ origin: NOVU_CLOUD (默认)                // 默认为 Novu 云原生
```

**processBlueprint 入口** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:551-574`

```typescript
private async processBlueprint(command: CreateWorkflowCommandV0) {
  if (!command.blueprintId) return null;

  const group: NotificationGroupEntity = await this.handleGroup(command);
  const steps: NotificationStep[] = this.normalizeSteps(command.steps);

  return CreateWorkflowCommandV0.create({
    // ... 其他字段
    blueprintId: command.blueprintId,          // 保留蓝图关联
    type: ResourceTypeEnum.REGULAR,            // 强制转为普通工作流
    origin: command.origin ?? ResourceOriginEnum.NOVU_CLOUD,  // 默认为 NOVU_CLOUD
  });
}
```

**关键点**：
1. 从蓝图创建的工作流 `type` 被强制设置为 `REGULAR`（不再是蓝图）
2. `origin` 默认为 `NOVU_CLOUD`，这意味着**从蓝图创建的工作流天然支持 SyncToEnvironment**
3. `blueprintId` 字段保留来源关联，用于溯源但不影响运行时行为

---

## 二、蓝图查询序列化真实行为：steps.template 与 steps.variants.template 的差异

### 2.1 虚拟字段定义

**Schema 虚拟字段** `libs/dal/src/repositories/notification-template/notification-template.schema.ts:260-278`

```typescript
notificationTemplateSchema.virtual('steps.template', {
  ref: 'MessageTemplate',
  localField: 'steps._templateId',
  foreignField: '_id',
  justOne: true,
});

notificationTemplateSchema.virtual('steps.variants.template', {
  ref: 'MessageTemplate',
  localField: 'steps.variants._templateId',
  foreignField: '_id',
  justOne: true,
});

// 确保序列化时包含虚拟字段
notificationTemplateSchema.path('steps')?.schema?.set('toJSON', { virtuals: true });
notificationTemplateSchema.path('steps')?.schema?.set('toObject', { virtuals: true });
notificationTemplateSchema.path('steps.variants')?.schema?.set('toJSON', { virtuals: true });
notificationTemplateSchema.path('steps.variants')?.schema?.set('toObject', { virtuals: true });
```

### 2.2 不同查询路径的 Populate 差异

| 查询方法 | populate('steps.template') | populate('steps.variants.template') | populate('notificationGroup') |
|---------|---------------------------|-----------------------------------|-------------------------------|
| `findBlueprintById` | ✅ 是 | ❌ **否** | ✅ 是 |
| `findBlueprintByTriggerIdentifier` | ✅ 是 | ❌ **否** | ✅ 是 |
| `findBlueprintTemplates` | ✅ 是 | ❌ **否** | ✅ 是 |
| `findAllGroupedByCategory` | ✅ 是 | ❌ **否** | ✅ 是 |
| `findById` (普通查询) | ✅ 是 | ✅ 是 | ❌ 否 |
| `findWithTemplates` | ✅ 是 | ✅ 是 | ❌ 否 |

**蓝图查询示例** `libs/dal/src/repositories/notification-template/notification-template.repository.ts:220-235`

```typescript
async findBlueprintById(id: string) {
  const requestQuery: NotificationTemplateQuery = {
    isBlueprint: true,
    _organizationId: this.blueprintOrganizationId,
    _id: id,
  };

  const item = await this.MongooseModel.findOne(requestQuery)
    .populate('steps.template')              // ✅ 只 populate 主模板
    .populate('notificationGroup')           // ✅ populate 通知分组
    .lean();

  return this.mapEntity(item);
}
```

**普通工作流查询对比** `libs/dal/src/repositories/notification-template/notification-template.repository.ts:153-172`

```typescript
async findById(id: string, environmentId: string, ...) {
  const query = this.MongooseModel.findOne({ ... })
    .populate('steps.template')              // ✅ 主模板
    .populate('steps.variants.template');    // ✅ **同时 populate 变体模板**

  const item = await query;
  return this.mapEntity(item);
}
```

### 2.3 实际影响

**蓝图导出时 `steps.variants[].template` 为 `null` 或 `undefined`**：

```json
// 蓝图导出时的 steps 结构
{
  "steps": [
    {
      "_templateId": "65abc123...",
      "template": { ... },       // ✅ 有数据
      "variants": [
        {
          "_templateId": "65def456...",
          "template": null       // ❌ 未 populate，为 null
        }
      ]
    }
  ]
}
```

**导入时的处理**：
- 蓝图导入流程中，variants 的 template 会被重新创建（不依赖导入数据）
- 导入流程主要依赖 `_templateId` 以外的字段（filters, metadata 等）
- 但如果业务逻辑需要变体的完整 template 数据，需要额外查询或在查询时手动添加 populate

---

## 三、Blueprint 导入门禁

### 3.1 notificationGroup.name 校验

**校验位置** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:593-623`

```typescript
private async handleGroup(command: CreateWorkflowCommandV0): Promise<NotificationGroupEntity> {
  // ⚠️ 严格校验：notificationGroup.name 必须存在
  if (!command.notificationGroup?.name) {
    throw new NotFoundException(`Notification group was not provided`);
  }

  // 按名称在当前环境中查找
  let notificationGroup = await this.notificationGroupRepository.findOne({
    name: command.notificationGroup.name,
    _environmentId: command.environmentId,
    _organizationId: command.organizationId,
  });

  // 不存在则创建（按名称匹配，而非 ID）
  if (!notificationGroup) {
    notificationGroup = await this.notificationGroupRepository.create({
      _environmentId: command.environmentId,
      _organizationId: command.organizationId,
      name: command.notificationGroup.name,  // 保留蓝图中的分组名称
    });

    // 非 Bridge 工作流需要创建变更记录
    if (!isBridgeWorkflow(command.type)) {
      await this.createChange.execute(...);
    }
  }

  return notificationGroup;
}
```

**关键点**：
1. **按名称匹配而非 ID**：不同环境的 notificationGroup._id 不同，但 name 保持一致
2. **不存在自动创建**：导入时如果目标环境没有对应名称的分组，自动创建
3. **严格的空值校验**：`!command.notificationGroup?.name` 直接抛出异常，不降级

### 3.2 feedId 归零机制

**归零位置** `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:577-591`

```typescript
private normalizeSteps(commandSteps: NotificationStep[]): NotificationStep[] {
  // 深拷贝，避免修改原始数据
  const steps = JSON.parse(JSON.stringify(commandSteps)) as NotificationStep[];

  return steps.map((step) => {
    const { template } = step;
    if (template) {
      // ⚠️ 强制清除 feedId，导入后在目标环境重新分配
      template.feedId = undefined;
    }

    return {
      ...step,
      ...(template ? { template } : {}),
    };
  });
}
```

**设计意图**：
- `feedId` 关联的是 In-App Feed，不同环境的 Feed 配置不同
- 清除后由目标环境的默认 Feed 或后续配置接管
- **仅在从蓝图创建时调用**：`normalizeSteps` 仅在 `processBlueprint` 中被调用

---

## 四、Grouped 蓝图读取侧的 Production 固定环境和 Popular 组筛选逻辑

### 4.1 Production 固定环境读取

**Controller 层入口** `apps/api/src/app/blueprint/blueprint.controller.ts:23-30`

```typescript
@Get('/group-by-category')
async getGroupedBlueprints(): Promise<GroupedBlueprintResponse> {
  // ⚠️ 强制从 Production 环境读取
  const prodEnvironmentId = await this.getProdEnvironmentId();

  return this.getGroupedBlueprintsUsecase.execute(
    GetGroupedBlueprintsCommand.create({ environmentId: prodEnvironmentId })
  );
}
```

**Production 环境查找逻辑** `apps/api/src/app/blueprint/blueprint.controller.ts:32-44`

```typescript
private async getProdEnvironmentId() {
  const productionEnvironmentId = (
    await this.environmentRepository.findOrganizationEnvironments(
      NotificationTemplateRepository.getBlueprintOrganizationId() || ''
    )
  )?.find((env) => env.name === 'Production')?._id;  // 按名称匹配

  if (!productionEnvironmentId) {
    throw new Error('Production environment id was not found');
  }

  return productionEnvironmentId;
}
```

**Repository 层重复校验** `libs/dal/src/repositories/notification-template/notification-template.repository.ts:275-296`

```typescript
async findAllGroupedByCategory(): Promise<{ name: string; blueprints: NotificationTemplateEntity[] }[]> {
  const organizationId = this.blueprintOrganizationId;

  // 再次查找 Production 环境（双重校验）
  const productionEnvironmentId = (
    await this.environmentRepository.findOrganizationEnvironments(organizationId)
  )?.find((env) => env.name === 'Production')?._id;

  if (!productionEnvironmentId) {
    throw new DalException(
      `Production environment id for BLUEPRINT_CREATOR ${process.env.BLUEPRINT_CREATOR} was not found`
    );
  }

  const requestQuery: NotificationTemplateQuery = {
    isBlueprint: true,
    _environmentId: productionEnvironmentId,  // 强制查询 Production 环境
    _organizationId: organizationId,
  };
  // ...
}
```

**关键点**：
1. **硬编码环境名称**：通过 `env.name === 'Production'` 匹配，不依赖环境 ID
2. **双重校验**：Controller 和 Repository 各查一次，确保一致性
3. **蓝图组织隔离**：始终从 `BLUEPRINT_CREATOR` 指定的组织读取

### 4.2 Popular 组筛选逻辑

**Popular 组 ID 配置** `apps/api/src/app/blueprint/usecases/get-grouped-blueprints/consts.ts:1-4`

```typescript
import { getPopularTemplateIds } from '@novu/shared';

export const POPULAR_GROUPED_NAME = 'Popular';
// 根据 NODE_ENV 返回不同的 ID 列表
export const POPULAR_TEMPLATES_ID_LIST = getPopularTemplateIds({
  production: process.env.NODE_ENV === 'production'
});
```

**getPopularTemplateIds 实现** `packages/shared/src/consts/template-store/index.ts:27-29`

```typescript
export function getPopularTemplateIds({ production }: { production: boolean }) {
  return production ? popularProductionIds : popularDevelopmentIds;
}
```

**Popular 组组装逻辑** `apps/api/src/app/blueprint/usecases/get-grouped-blueprints/get-grouped-blueprints.usecase.ts:20-31,48-73`

```typescript
async execute(command: GetGroupedBlueprintsCommand): Promise<GroupedBlueprintResponse> {
  const generalGroups = await this.fetchGroupedBlueprints();

  // 从 generalGroups 中筛选出 Popular 组的蓝图
  const updatePopularBlueprints = this.getPopularGroupBlueprints(generalGroups);

  const popularGroup = { name: POPULAR_GROUPED_NAME, blueprints: updatePopularBlueprints };

  return {
    general: generalGroups as unknown as IGroupedBlueprint[],
    popular: popularGroup as unknown as IGroupedBlueprint,
  };
}

private getPopularGroupBlueprints(
  groups: { name: string; blueprints: NotificationTemplateEntity[] }[]
): NotificationTemplateEntity[] {
  const storedBlueprints = this.groupedToBlueprintsArray(groups);
  const localPopularIds = [...POPULAR_TEMPLATES_ID_LIST];
  const result: NotificationTemplateEntity[] = [];

  for (const localPopularId of localPopularIds) {
    // 按 ID 精确匹配
    const storedBlueprint = storedBlueprints.find((blueprint) => blueprint._id === localPopularId);

    if (!storedBlueprint) {
      this.logger.warn(
        `Could not find stored popular blueprint id: ${localPopularId}, BLUEPRINT_CREATOR: 
        ${NotificationTemplateRepository.getBlueprintOrganizationId()}`
      );
      continue;  // 找不到则跳过，不抛出异常
    }

    result.push(storedBlueprint);
  }

  return result;
}
```

**关键点**：
1. **硬编码 ID 列表**：Popular 组不是动态计算的，而是配置好的固定 ID 列表
2. **环境区分**：生产环境和开发环境使用不同的 ID 列表
3. **容错降级**：找不到对应 ID 时只打 warn 日志，继续处理下一个
4. **顺序保持**：结果顺序与配置的 ID 列表顺序一致

### 4.3 缓存机制与 TTL

**注意**：`get-grouped-blueprints.usecase.ts` 第 9 行定义了：
```typescript
const WEEK_IN_SECONDS = 60 * 60 * 24 * 7;
```

但**当前代码中并未实际使用**该常量，也没有 `@CachedResponse` 装饰器。缓存是在 Controller 层之上可能由其他机制处理，或者该常量为预留字段。

---

## 五、发布侧 Grouped 蓝图缓存失效触发条件

### 5.1 失效触发入口

**PromoteNotificationTemplateChange 执行流程** `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:60-62`

```typescript
async execute(command: PromoteTypeChangeCommand) {
  // ⚠️ **第一步**：先失效缓存，再执行业务逻辑
  await this.invalidateBlueprints(command);

  const item = await this.notificationTemplateRepository.findOne({
    _environmentId: command.environmentId,
    _parentId: command.item._id,
  });
  // ... 后续业务逻辑
}
```

### 5.2 失效触发条件

**invalidateBlueprints 实现** `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:307-317`

```typescript
private async invalidateBlueprints(command: PromoteTypeChangeCommand) {
  // ⚠️ 条件 1：只有当操作组织 == 蓝图组织时才失效
  if (command.organizationId === this.blueprintOrganizationId) {
    // 获取蓝图组织的 Production 环境 ID
    const productionEnvironmentId = await this.getProductionEnvironmentId(
      this.blueprintOrganizationId
    );

    if (productionEnvironmentId) {
      // ⚠️ 失效的是 Production 环境的 grouped 蓝图缓存
      await this.invalidateCache.invalidateByKey({
        key: buildGroupedBlueprintsKey(productionEnvironmentId),
      });
    }
  }
}
```

**蓝图组织 ID 获取** `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:303-305`

```typescript
private get blueprintOrganizationId() {
  return NotificationTemplateRepository.getBlueprintOrganizationId();
}
```

**缓存 Key 构建** `libs/application-generic/src/services/cache/key-builders/entities.ts:62-68`

```typescript
export const buildGroupedBlueprintsKey = (environmentId: string): string =>
  buildEnvironmentScopedKeyById({
    type: CacheKeyTypeEnum.ENTITY,
    keyEntity: CacheKeyPrefixEnum.GROUPED_BLUEPRINTS,
    environmentId,
    identifierPrefix: IdentifierPrefixEnum.GROUPED_BLUEPRINT,
  });
```

### 5.3 触发条件总结

| 条件 | 是否触发失效 | 说明 |
|------|-------------|------|
| 普通组织发布工作流 | ❌ 否 | `command.organizationId !== blueprintOrganizationId` |
| 蓝图组织发布非蓝图 | ✅ 是 | 组织匹配即触发，不检查是否是蓝图变更 |
| 蓝图组织发布蓝图 | ✅ 是 | 标准场景 |
| 蓝图组织 Development → Development | ❌ 否 | `getProductionEnvironmentId` 返回 undefined |
| 蓝图组织 Development → Production | ✅ 是 | 标准发布场景 |

**关键点**：
1. **以组织为判断维度**：只要是蓝图组织在发布，不管发布什么内容都失效缓存
2. **失效的是 Production 环境缓存**：即使发布是从 Dev 到其他环境，失效的也是 Production 的缓存
3. **前置失效**：先失效缓存再执行业务逻辑，保证一致性
4. **仅在 Promote 链路失效**：`SyncToEnvironment` 链路不触发蓝图缓存失效（因为它不操作蓝图组织的内容）

### 5.4 isBlueprint 设置时机

**创建时** `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:190`

```typescript
const newNotificationTemplate: Partial<NotificationTemplateEntity> = {
  // ...
  isBlueprint: command.organizationId === this.blueprintOrganizationId,
  blueprintId: newItem.blueprintId,
  // ...
};
```

**更新时** `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:233`

```typescript
const updatedTemplate = await this.notificationTemplateRepository.update(
  { _environmentId: command.environmentId, _id: item._id },
  {
    // ...
    isBlueprint: command.organizationId === this.blueprintOrganizationId,
    // ...
  }
);
```

**设计意图**：
- `isBlueprint` 字段不依赖源数据，而是根据当前操作组织动态计算
- 只有蓝图组织发布的工作流才会被标记为蓝图
- 这保证了蓝图只能在蓝图组织的 Production 环境中存在

---

## 六、补充：SyncToEnvironment 中 Step ID 映射的细节

**`mapStepsToCreateOrUpdateDto` 实现** `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:231-263`

```typescript
private async mapStepsToCreateOrUpdateDto(
  sourceSteps: StepResponseDto[],
  targetEnvSteps?: StepResponseDto[]
): Promise<UpsertStepDataCommand[]> {
  return sourceSteps.map((sourceStep) => {
    // 按 stepId（业务标识符）而非 _id（数据库 ID）匹配
    const targetStepInternalId = targetEnvSteps?.find(
      (targetStep) => targetStep.stepId === sourceStep.stepId
    )?._id;

    return this.buildStepCreateOrUpdateDto(sourceStep, targetStepInternalId);
  });
}

private buildStepCreateOrUpdateDto(
  sourceStep: StepResponseDto,
  targetStepInternalId?: string
): UpsertStepDataCommand {
  return {
    // 有匹配的 _id 则是更新，无则是创建
    ...(targetStepInternalId && { _id: targetStepInternalId }),
    stepId: sourceStep.stepId,  // 保留业务 stepId
    name: sourceStep.name ?? '',
    type: sourceStep.type,
    controlValues: sourceStep.controls?.values ?? {},
  };
}
```

**关键点**：
1. **stepId 是业务标识符**：跨环境保持一致，用于匹配
2. **_id 是数据库标识符**：每个环境不同，用于更新时定位
3. **Upsert 语义**：携带 `_id` 则更新，否则创建

---

## 七、核心代码文件索引（补充）

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 源限制 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:47` | `SYNCABLE_WORKFLOW_ORIGINS` 定义 |
| 蓝图创建处理 | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:551-574` | `processBlueprint` 入口 |
| 分组校验 | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:593-623` | `handleGroup` 校验逻辑 |
| feedId 归零 | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:577-591` | `normalizeSteps` 实现 |
| Production 环境 | `apps/api/src/app/blueprint/blueprint.controller.ts:32-44` | `getProdEnvironmentId` 实现 |
| Popular 组筛选 | `apps/api/src/app/blueprint/usecases/get-grouped-blueprints/get-grouped-blueprints.usecase.ts:48-73` | `getPopularGroupBlueprints` 实现 |
| 缓存失效 | `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:307-317` | `invalidateBlueprints` 实现 |
| Populate 差异 | `libs/dal/src/repositories/notification-template/notification-template.repository.ts:220-235` | `findBlueprintById` 实现 |
| Origin 定义 | `packages/shared/src/types/general.ts:9-14` | `ResourceOriginEnum` 定义 |
| Popular ID 配置 | `packages/shared/src/consts/template-store/index.ts:27-29` | `getPopularTemplateIds` 实现 |
