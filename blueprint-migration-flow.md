# Blueprint 导入导出与跨环境迁移协同链路分析

## 一、核心实体与数据模型

### 1.1 蓝图实体定义 (`NotificationTemplateEntity`)

蓝图本质上是标记了 `isBlueprint: true` 的通知模板实体，核心定义位于：

**`libs/dal/src/repositories/notification-template/notification-template.entity.ts:29-109`**

```typescript
export class NotificationTemplateEntity {
  _id: string;
  name: string;
  description: string;
  active: boolean;
  draft: boolean;
  tags: string[];
  steps: NotificationStepEntity[];      // 步骤定义，含消息模板引用
  triggers: NotificationTriggerEntity[]; // 触发器定义
  _notificationGroupId: string;          // 通知分组ID
  _organizationId: OrganizationId;       // 组织ID
  _environmentId: EnvironmentId;         // 环境ID
  isBlueprint: boolean;                  // 蓝图标记
  blueprintId?: string;                  // 来源蓝图ID（用于从蓝图创建）
  type?: ResourceTypeEnum;               // 资源类型：REGULAR / BRIDGE
  origin?: ResourceOriginEnum;           // 来源：NOVU_CLOUD / EXTERNAL 等
  payloadSchema?: any;                   // Payload JSON Schema
  validatePayload?: boolean;             // 是否校验 Payload
  // ... 其他字段
}
```

### 1.2 关联实体层级

```
NotificationTemplateEntity (蓝图)
├── NotificationStepEntity[] (步骤)
│   ├── _templateId → MessageTemplateEntity (消息模板)
│   ├── filters: StepFilter[] (过滤条件)
│   ├── metadata: IWorkflowStepMetadata (步骤元数据)
│   └── variants: NotificationStepData[] (变体)
│       └── _templateId → MessageTemplateEntity
├── NotificationTriggerEntity[] (触发器)
│   ├── identifier: string (触发标识符)
│   ├── variables: INotificationTriggerVariable[]
│   └── reservedVariables: ITriggerReservedVariable[]
└── _notificationGroupId → NotificationGroupEntity (通知分组)
```

### 1.3 MessageTemplateEntity 结构

**`libs/dal/src/repositories/message-template/message-template.entity.ts:14-67`**

```typescript
export class MessageTemplateEntity {
  _id?: string;
  type: StepTypeEnum;                    // 步骤类型：EMAIL/SMS/IN_APP/CHAT/PUSH/DIGEST/DELAY 等
  content: string | IEmailBlock[];       // 消息内容
  contentType?: MessageTemplateContentType;
  controls?: ControlSchemas;             // 控制 schema（JSON Schema + UI Schema）
  output?: { schema: JSONSchemaEntity }; // 输出 schema
  _parentId?: string;                    // 父模板ID（跨环境映射用）
  stepResolverHash?: string;             // 步骤解析器哈希
  // ... 其他字段
}
```

## 二、蓝图序列化与导出流程

### 2.1 蓝图查询与数据加载

**查询入口**：`apps/api/src/app/blueprint/usecases/get-blueprint/get-blueprint.usecase.ts:11-29`

```typescript
async execute(command: GetBlueprintCommand): Promise<GetBlueprintResponse> {
  const isInternalId = NotificationTemplateRepository.isInternalId(command.templateIdOrIdentifier);
  let template: NotificationTemplateEntity | null;
  if (isInternalId) {
    template = await this.notificationTemplateRepository.findBlueprintById(command.templateIdOrIdentifier);
  } else {
    template = await this.notificationTemplateRepository.findBlueprintByTriggerIdentifier(
      command.templateIdOrIdentifier
    );
  }
  return template as GetBlueprintResponse;
}
```

**Repository 层查询**：`libs/dal/src/repositories/notification-template/notification-template.repository.ts:220-235`

```typescript
async findBlueprintById(id: string) {
  const requestQuery: NotificationTemplateQuery = {
    isBlueprint: true,
    _organizationId: this.blueprintOrganizationId,  // BLUEPRINT_CREATOR 环境变量
    _id: id,
  };
  const item = await this.MongooseModel.findOne(requestQuery)
    .populate('steps.template')              // 关联加载消息模板
    .populate('notificationGroup')           // 关联加载通知分组
    .lean();
  return this.mapEntity(item);
}
```

**关键序列化机制**：
1. **Mongoose Populate**：通过 `populate('steps.template')` 和 `populate('notificationGroup')` 自动关联加载关联实体
2. **虚拟字段**：在 schema 中定义虚拟字段，实现跨表关联
   - `notificationTemplate.schema.ts:260-285` 定义了 `steps.template`、`steps.variants.template`、`notificationGroup` 等虚拟字段
3. **mapEntity 转换**：将 MongoDB 文档转换为业务实体对象
4. **ClassSerializerInterceptor**：Controller 层使用该拦截器进行最终的响应序列化

### 2.2 对外格式约束 (DTO)

**GetBlueprintResponse**：`apps/api/src/app/blueprint/dtos/get-blueprint.response.dto.ts:3-48`

```typescript
export class GetBlueprintResponse {
  _id: string;
  name: string;
  description: string;
  active: boolean;
  draft: boolean;
  preferenceSettings: IPreferenceChannels;
  critical: boolean;
  tags: string[];
  steps: NotificationStepDto[];
  triggers: INotificationTrigger[];
  _notificationGroupId: string;
  notificationGroup?: INotificationGroup;  // 关联的分组信息
  isBlueprint: boolean;
  blueprintId?: string;
  // ... 其他字段
}
```

**IGroupedBlueprint**：`packages/shared/src/entities/notification-template/notification-template.interface.ts:43-50`

```typescript
export class IGroupedBlueprint {
  name: string;                    // 分组名称
  blueprints: IBlueprint[];        // 该分组下的蓝图列表
}

export interface IBlueprint extends INotificationTemplate {
  notificationGroup: INotificationGroup;
}
```

### 2.3 分组蓝图导出

**`apps/api/src/app/blueprint/usecases/get-grouped-blueprints/get-grouped-blueprints.usecase.ts:33-42`**

```typescript
private async fetchGroupedBlueprints() {
  const groups = await this.notificationTemplateRepository.findAllGroupedByCategory();
  return groups;
}

// Repository 中按分组聚合
async findAllGroupedByCategory(): Promise<{ name: string; blueprints: NotificationTemplateEntity[] }[]> {
  // 查询所有蓝图并按 notificationGroup 分组
  const items = result?.map((item) => this.mapEntity(item));
  const groupedItems = items.reduce((acc, item) => {
    const notificationGroupId = item._notificationGroupId;
    const notificationGroupName = item.notificationGroup?.name;
    // ... 分组逻辑
  }, {});
  return Object.values(groupedItems);
}
```

## 三、导入校验与跨环境迁移链路

### 3.1 两种迁移模式

系统支持两种跨环境迁移模式：

| 模式 | 触发场景 | 核心机制 |
|------|---------|---------|
| **变更提升 (Promote)** | Development → Production 环境发布 | Change 变更记录 + 差异合并 |
| **同步到环境 (Sync)** | 手动同步工作流到其他环境 | 完整实体复制 + ID 重映射 |

### 3.2 变更提升链路 (Promote Flow)

#### 3.2.1 整体架构

```
ApplyChange (应用变更)
    ↓
PromoteChangeToEnvironment (提升到环境)
    ↓
PromoteNotificationTemplateChange (处理通知模板变更)
    ├─ 处理 NotificationGroup 迁移
    ├─ 处理 MessageTemplate ID 重映射
    └─ 创建/更新目标环境的 NotificationTemplate
```

#### 3.2.2 ApplyChange - 应用变更入口

**`apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts:15-83`**

```typescript
async execute(command: ApplyChangeCommand): Promise<ChangeEntity[]> {
  const parentChange = await this.changeRepository.findOne({...});
  const changes = await this.changeRepository.find(
    { _environmentId: parentChange._environmentId, _parentId: parentChange._id },
    '', { sort: { createdAt: 1 } }
  );
  
  for (const change of [...changes, parentChange]) {
    const item = await this.applyChange(change, command);
    items.push(item);
  }
  return items;
}

async applyChange(change, command: ApplyChangeCommand): Promise<ChangeEntity> {
  try {
    await this.changeRepository.update({ _id: change._id, ... }, { enabled: true });
    await this.promoteChangeToEnvironment.execute(
      PromoteChangeToEnvironmentCommand.create({
        itemId: change._entityId,
        type: change.type,
        environmentId: change._environmentId,
        organizationId: change._organizationId,
        userId: command.userId,
      })
    );
  } catch (e) {
    await this.changeRepository.update({ _id: change._id, ... }, { enabled: false });
    throw e;
  }
  return change;
}
```

#### 3.2.3 PromoteChangeToEnvironment - 变更分发

**`apps/api/src/app/change/usecases/promote-change-to-environment/promote-change-to-environment.usecase.ts:40-91`**

```typescript
async execute(command: PromoteChangeToEnvironmentCommand) {
  // 1. 聚合所有变更
  const changes = await this.changeRepository.getEntityChanges(command.organizationId, command.type, command.itemId);
  const aggregatedItem = changes
    .filter((change) => change.enabled)
    .reduce((prev, change) => {
      const sanitized = sanitizeDiff(change.change);
      if (sanitized.length === 0) return prev;
      return applyDiff(prev, sanitized);  // 递归应用差异
    }, {});

  // 2. 查找目标环境（子环境）
  const environment = await this.environmentRepository.findOne({
    _parentId: command.environmentId,
  });

  // 3. 按类型分发处理
  const typeCommand = PromoteTypeChangeCommand.create({
    organizationId: command.organizationId,
    environmentId: environment._id,  // 目标环境ID
    item: aggregatedItem,
    userId: command.userId,
  });

  switch (command.type) {
    case ChangeEntityTypeEnum.NOTIFICATION_TEMPLATE:
      await this.promoteNotificationTemplateChange.execute(typeCommand);
      break;
    case ChangeEntityTypeEnum.MESSAGE_TEMPLATE:
      await this.promoteMessageTemplateChange.execute(typeCommand);
      break;
    // ... 其他类型
  }
}
```

#### 3.2.4 PromoteNotificationTemplateChange - 模板提升核心

**`apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts:60-241`**

**核心处理步骤**：

1. **ID 重映射 - MessageTemplate**
   ```typescript
   const mapNewStepItem = (step: NotificationStepEntity) => {
     const oldMessage = messages.find((message) => {
       return message._parentId === step._templateId;  // 通过 _parentId 查找目标环境中的对应实体
     });
     if (step?._templateId && oldMessage._id) {
       step._templateId = oldMessage._id;  // 替换为目标环境的 ID
     }
     return step;
   };
   ```

2. **NotificationGroup 处理**
   ```typescript
   let notificationGroup = await this.notificationGroupRepository.findOne({
     _environmentId: command.environmentId,
     _organizationId: command.organizationId,
     _parentId: newItem._notificationGroupId,  // 通过 _parentId 查找
   });
   if (!notificationGroup) {
     // 如果目标环境不存在，先应用该分组的变更
     const changes = await this.changeRepository.getEntityChanges(
       command.organizationId, ChangeEntityTypeEnum.NOTIFICATION_GROUP, newItem._notificationGroupId
     );
     for (const change of changes) {
       await this.applyChange.execute(...);
     }
     // 再次查询
   }
   ```

3. **创建或更新目标实体**
   ```typescript
   if (!item) {
     // 目标环境不存在，创建新模板
     const newNotificationTemplate: Partial<NotificationTemplateEntity> = {
       name: newItem.name,
       active: newItem.active,
       steps,  // 已重映射 ID 的步骤
       _parentId: command.item._id,  // 设置父ID，关联源实体
       _environmentId: command.environmentId,
       _organizationId: command.organizationId,
       _notificationGroupId: notificationGroup._id,
       isBlueprint: command.organizationId === this.blueprintOrganizationId,
       blueprintId: newItem.blueprintId,
       // ... 其他字段
     };
     const createdTemplate = await this.notificationTemplateRepository.create(
       newNotificationTemplate as NotificationTemplateEntity
     );
   } else {
     // 目标环境已存在，执行更新
     const updatedTemplate = await this.notificationTemplateRepository.update(
       { _environmentId: command.environmentId, _id: item._id },
       {
         name: newItem.name,
         steps,
         _notificationGroupId: notificationGroup._id,
         // ... 其他字段
       }
     );
   }
   ```

### 3.3 同步到环境链路 (SyncToEnvironment Flow)

#### 3.3.1 整体流程

**`apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:76-163`**

```
1. 前置校验
   ├─ 不能同步到相同环境
   ├─ 目标环境存在性校验
   └─ 工作流可同步性校验 (origin 必须是 NOVU_CLOUD)
   
2. 数据准备
   ├─ 获取源工作流完整信息
   ├─ 获取源工作流偏好设置
   ├─ 查找目标环境中是否已存在同名工作流
   └─ 构建请求 DTO (区分创建/更新)

3. 关联资源同步
   ├─ Layout 同步 (邮件步骤引用的布局)
   ├─ Layout 翻译组发布
   ├─ StepResolver 同步
   └─ Workflow 翻译组发布

4. 执行 Upsert
   └─ 调用 UpsertWorkflowUseCase 在目标环境创建/更新

5. 后置处理
   ├─ 更新源工作流发布信息
   └─ 发送 Webhook 通知
```

#### 3.3.2 可同步性校验

```typescript
export const SYNCABLE_WORKFLOW_ORIGINS = [ResourceOriginEnum.NOVU_CLOUD];

private isSyncable(workflow: WorkflowResponseDto): boolean {
  return SYNCABLE_WORKFLOW_ORIGINS.includes(workflow.origin);
}
```

#### 3.3.3 DTO 构建 - 创建模式

```typescript
private async mapWorkflowToCreateWorkflowDto(
  sourceWorkflow: WorkflowResponseDto,
  preferences: PreferencesEntity[]
): Promise<UpsertWorkflowDataCommand> {
  return {
    workflowId: sourceWorkflow.workflowId,  // 保留原始 workflowId
    payloadSchema: sourceWorkflow.payloadSchema || null,
    validatePayload: sourceWorkflow.validatePayload,
    isTranslationEnabled: sourceWorkflow.isTranslationEnabled,
    origin: ResourceOriginEnum.NOVU_CLOUD,
    name: sourceWorkflow.name,
    active: sourceWorkflow.active,
    tags: sourceWorkflow.tags,
    description: sourceWorkflow.description,
    severity: sourceWorkflow.severity,
    steps: await this.mapStepsToCreateOrUpdateDto(sourceWorkflow.steps),
    preferences: this.mapPreferences(preferences),
  };
}
```

#### 3.3.4 Step ID 映射策略

```typescript
private async mapStepsToCreateOrUpdateDto(
  sourceSteps: StepResponseDto[],
  targetEnvSteps?: StepResponseDto[]
): Promise<UpsertStepDataCommand[]> {
  return sourceSteps.map((sourceStep) => {
    // 如果目标环境有匹配的 stepId，则携带 _id 进行更新
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
    ...(targetStepInternalId && { _id: targetStepInternalId }),  // 有则更新，无则创建
    stepId: sourceStep.stepId,
    name: sourceStep.name ?? '',
    type: sourceStep.type,
    controlValues: sourceStep.controls?.values ?? {},
  };
}
```

### 3.4 UpsertWorkflow - 通用导入/创建流程

**`libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:69-137`**

#### 3.4.1 执行流程

```
1. 查询现有工作流（通过 workflowIdOrInternalId）
2. 区分创建/更新分支
   ├─ 创建：调用 CreateWorkflowV0
   └─ 更新：调用 UpdateWorkflowV0
3. Upsert ControlValues (步骤控制值)
4. 查询并返回完整工作流信息
5. 发送 Webhook 通知
```

#### 3.4.2 Steps 构建与校验

**`libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:204-302`**

```typescript
private async buildSteps(
  command: UpsertWorkflowCommand,
  existingWorkflow?: NotificationTemplateEntity
): Promise<NotificationStep[]> {
  // 1. 预加载现有 ControlValues
  // 2. 生成唯一 StepId
  // 3. 为每个步骤执行 Issue 检测
  const stepsWithIssues = await Promise.all(
    command.workflowDto.steps.map(async (step, index) => {
      const controlSchemaKey = step.type;
      const controlSchemas: ControlSchemas =
        existingStep?.template?.controls || stepTypeToControlSchema[controlSchemaKey];
      
      // 步骤问题校验
      const issues: StepIssuesDto = await this.buildStepIssuesUsecase.execute({
        workflowOrigin,
        user,
        stepType: step.type,
        controlSchema: controlSchemas.schema,
        controlsDto: step.controlValues,
        optimisticSteps,
        preloadedControlValues,
        optimisticPayloadSchema,
      });

      return {
        template: {
          type: step.type,
          name: step.name,
          controls: controlSchemas,
          content: '',
        },
        stepId: stepIds[index],
        name: step.name,
        issues,
      };
    })
  );

  return stepsWithIssues;
}
```

#### 3.4.3 StepId 生成策略

```typescript
private generateUniqueStepId(step: UpsertStepDataCommand, previousSteps: NotificationStep[]): string {
  const slug = slugifyOrRandom(step.name);
  let finalStepId = slug;
  let attempts = 0;
  const maxAttempts = 5;

  const previousStepIds = previousSteps.reduce<string[]>((acc, { stepId }) => {
    if (stepId) acc.push(stepId);
    return acc;
  }, []);

  const isStepIdUnique = (stepId: string) => !previousStepIds.includes(stepId);

  while (attempts < maxAttempts) {
    if (isStepIdUnique(finalStepId)) break;
    finalStepId = `${slug}-${shortId()}`;
    attempts += 1;
  }
  // ...
}
```

### 3.5 CreateWorkflowV0 - 创建流程核心校验

**`libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts:59-137`**

#### 3.5.1 前置校验

```typescript
async execute(usecaseCommand: CreateWorkflowCommandV0): Promise<WorkflowWithPreferencesResponseDto> {
  const blueprintCommand = await this.processBlueprint(usecaseCommand);
  const command = blueprintCommand ?? usecaseCommand;
  
  await this.validatePayload(command);
  await this.resourceValidatorService.validateWorkflowLimit(command.environmentId);
  // ...
}

private async validatePayload(command: CreateWorkflowCommandV0) {
  // 步骤数量限制校验
  if (command.steps) {
    await this.resourceValidatorService.validateStepsLimit(
      command.environmentId,
      command.organizationId,
      command.steps
    );
  }

  // 变体非空校验
  const variants = command.steps ? command.steps?.flatMap((step) => step.variants || []) : [];
  for (const variant of variants) {
    if (isVariantEmpty(variant)) {
      throw new BadRequestException(
        `Variant conditions are required, variant name ${variant.name} id ${variant._id}`
      );
    }
  }
}
```

#### 3.5.2 触发器标识符唯一性校验

```typescript
private async generateUniqueIdentifier(command: CreateWorkflowCommandV0, triggerIdentifier: string) {
  const maxAttempts = 3;
  let identifier = '';

  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const candidateIdentifier = attempt === 0 ? triggerIdentifier : `${triggerIdentifier}-${shortId()}`;
    const isIdentifierExist = await this.notificationTemplateRepository.findByTriggerIdentifier(
      command.environmentId,
      candidateIdentifier
    );
    if (!isIdentifierExist) {
      identifier = candidateIdentifier;
      break;
    }
  }
  // ...
}
```

#### 3.5.3 蓝图导入处理

```typescript
private async processBlueprint(command: CreateWorkflowCommandV0) {
  if (!command.blueprintId) return null;

  const group: NotificationGroupEntity = await this.handleGroup(command);
  const steps: NotificationStep[] = this.normalizeSteps(command.steps);

  return CreateWorkflowCommandV0.create({
    // ... 字段映射
    blueprintId: command.blueprintId,  // 保留来源蓝图ID
    type: ResourceTypeEnum.REGULAR,
    origin: command.origin ?? ResourceOriginEnum.NOVU_CLOUD,
  });
}

private normalizeSteps(commandSteps: NotificationStep[]): NotificationStep[] {
  const steps = JSON.parse(JSON.stringify(commandSteps)) as NotificationStep[];
  return steps.map((step) => {
    const { template } = step;
    if (template) {
      template.feedId = undefined;  // 清除 feedId，导入后重新分配
    }
    return { ...step, ...(template ? { template } : {}) };
  });
}
```

## 四、数据库 Schema 约束

### 4.1 NotificationTemplate Schema

**`libs/dal/src/repositories/notification-template/notification-template.schema.ts:105-258`**

关键字段约束：
- `isBlueprint: { type: Schema.Types.Boolean, default: false }`
- `blueprintId: { type: Schema.Types.String }`
- `type: { type: Schema.Types.String, default: ResourceTypeEnum.REGULAR }`
- `origin: { type: Schema.Types.String }`
- `payloadSchema: Schema.Types.Mixed`
- `_parentId: { type: Schema.Types.ObjectId, ref: 'NotificationTemplate' }`

索引约束：
```javascript
notificationTemplateSchema.index({ _environmentId: 1, 'triggers.identifier': 1 });
notificationTemplateSchema.index({ _environmentId: 1, _id: 1 });
```

### 4.2 MessageTemplate Schema

**`libs/dal/src/repositories/message-template/message-template.schema.ts:9-88`**

关键字段约束：
- `_parentId: { type: Schema.Types.ObjectId, ref: 'NotificationTemplate' }`
- `controls: { schema: Schema.Types.Mixed, uiSchema: Schema.Types.Mixed }`
- `stepResolverHash: { type: Schema.Types.String }`

索引约束：
```javascript
messageTemplateSchema.index({ _parentId: 1 });
```

## 五、关键设计要点

### 5.1 ID 映射机制

系统通过 `_parentId` 字段维护跨环境实体关联：

| 实体 | 源环境 | 目标环境 | 映射关系 |
|------|--------|----------|---------|
| NotificationTemplate | `_id` | `_parentId = 源._id` | 1:1 |
| MessageTemplate | `_id` | `_parentId = 源._id` | 1:1 |
| NotificationGroup | `_id` | `_parentId = 源._id` | 1:1 |

### 5.2 事务处理

- `CreateWorkflowV0` 支持外部传入 session 或内部创建事务
- `notificationTemplateRepository.withTransaction()` 确保原子性
- 变更应用失败时自动回滚 `enabled: false`

### 5.3 蓝图组织隔离

- 蓝图存储在特定组织中，通过 `BLUEPRINT_CREATOR` 环境变量指定
- 查询时自动过滤 `_organizationId = blueprintOrganizationId`
- 普通用户导入蓝图时，自动设置 `blueprintId` 关联来源

### 5.4 资源限制校验

- `validateWorkflowLimit` - 工作流数量限制
- `validateStepsLimit` - 单工作流步骤数量限制
- `ResourceValidatorService` 统一处理资源限额校验

## 六、核心代码文件索引

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 实体定义 | `libs/dal/src/repositories/notification-template/notification-template.entity.ts` | NotificationTemplateEntity 定义 |
| 实体定义 | `libs/dal/src/repositories/message-template/message-template.entity.ts` | MessageTemplateEntity 定义 |
| Repository | `libs/dal/src/repositories/notification-template/notification-template.repository.ts` | 蓝图查询与数据访问 |
| 蓝图导出 | `apps/api/src/app/blueprint/usecases/get-blueprint/get-blueprint.usecase.ts` | 单个蓝图导出 |
| 蓝图导出 | `apps/api/src/app/blueprint/usecases/get-grouped-blueprints/get-grouped-blueprints.usecase.ts` | 分组蓝图导出 |
| 变更应用 | `apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts` | 应用变更入口 |
| 变更提升 | `apps/api/src/app/change/usecases/promote-change-to-environment/promote-change-to-environment.usecase.ts` | 变更分发到目标环境 |
| 模板提升 | `apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts` | 通知模板跨环境提升 |
| 环境同步 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | 工作流同步到环境 |
| 通用Upsert | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts` | 工作流创建/更新通用逻辑 |
| 创建工作流 | `libs/application-generic/src/usecases/create-workflow-v0/create-workflow.usecase.ts` | 工作流创建与校验 |
| DTO定义 | `apps/api/src/app/blueprint/dtos/get-blueprint.response.dto.ts` | 蓝图响应DTO |
| 接口定义 | `packages/shared/src/entities/notification-template/notification-template.interface.ts` | 共享接口定义 |
