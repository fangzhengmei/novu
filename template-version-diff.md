# 通知模板版本管理代码分析

## 一、核心数据模型

### 1.1 NotificationTemplateEntity —— 工作流模板实体

定义位置: [notification-template.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/notification-template/notification-template.entity.ts)

关键字段:

| 字段 | 类型 | 作用 |
|------|------|------|
| `_id` | string | 当前环境下模板的唯一 ID |
| `_parentId` | string? | **跨环境关联键**。Dev 环境模板被 Promote 到 Prod 后，Prod 环境副本的 `_parentId` 指向 Dev 环境的原始 `_id` |
| `_environmentId` | string | 所属环境 ID，与 `_id` 共同构成复合主键逻辑 |
| `steps` | NotificationStepEntity[] | 步骤数组，每个步骤含 `_templateId` 关联 MessageTemplate |
| `active` | boolean | 模板是否激活（可被 trigger 触发） |
| `draft` | boolean | 是否为草稿状态 |
| `payloadSchema` | any | 触发时的 payload JSON Schema（用于变量校验） |
| `validatePayload` | boolean | 是否启用 payload 校验 |
| `status` | WorkflowStatusEnum? | 工作流状态（由 active + steps 计算得出） |

跨环境关联示意:

```
Dev Environment:
  NotificationTemplate { _id: "dev_tpl_123", _parentId: undefined, _environmentId: "dev_env" }
     ↓ promote
Production Environment:
  NotificationTemplate { _id: "prod_tpl_456", _parentId: "dev_tpl_123", _environmentId: "prod_env" }
```

### 1.2 MessageTemplateEntity —— 消息模板实体

定义位置: [message-template.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/message-template/message-template.entity.ts)

关键字段:

| 字段 | 类型 | 作用 |
|------|------|------|
| `_id` | string | 当前环境消息模板 ID |
| `_parentId` | string? | 跨环境关联键 |
| `type` | StepTypeEnum | 渠道类型（EMAIL/SMS/IN_APP/PUSH/CHAT 等） |
| `content` | string \| IEmailBlock[] | 模板内容 |
| `variables` | ITemplateVariable[]? | 模板变量定义列表 |
| `controls` | ControlSchemas? | Bridge 工作流的控制 Schema |
| `subject`, `preheader`, `senderName` | string? | Email 专属字段 |

### 1.3 ChangeEntity —— 变更记录实体（V1 版本管理核心）

定义位置: [change.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/change/change.entity.ts)
Schema 定义: [change.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/change/change.schema.ts)

这是 **V1 版本管理**的核心数据结构，记录 Dev 环境中每次编辑产生的差异补丁:

| 字段 | 类型 | 作用 |
|------|------|------|
| `_entityId` | string | 被修改实体的 ID（即 Dev 环境中模板的 `_id`） |
| `type` | ChangeEntityTypeEnum | 变更实体类型：`NOTIFICATION_TEMPLATE`、`MESSAGE_TEMPLATE`、`LAYOUT`、`FEED`、`NOTIFICATION_GROUP`、`TRANSLATION` 等 |
| `change` | any | **递归 Diff 补丁**（recursive-diff 格式的 rdiffResult[]） |
| `enabled` | boolean | 是否已应用（Promote）到下一级环境 |
| `_parentId` | string? | 父级 Change ID（用于级联变更分组，如工作流变更作为父，步骤变更作为子） |
| `_environmentId` | string | 变更所属的源环境 ID（通常是 Dev） |
| `_creatorId` | string | 创建者用户 ID |

rdiffResult 补丁格式示例（来自 `recursive-diff` 库）:

```typescript
// path: 变更路径数组，如 ["steps", "0", "template", "content"]
// op: add / update / delete
// val: 新值（仅 add/update）
interface rdiffResult {
  path: Array<string | number>;
  op: 'add' | 'update' | 'delete';
  val?: any;
}
```

---

## 二、两套版本管理体系

Novu 实际存在 **两套并行的版本管理体系**：

| 体系 | 适用场景 | 核心机制 | 主要 API |
|------|---------|---------|---------|
| **V1 ChangeEntity 模式** | 单工作流粒度的 Promote | 基于 `ChangeEntity` 的增量补丁 + `enabled` 标记 | `POST /changes/:changeId/apply`、`POST /changes/bulk/apply` |
| **V2 Environments 模式** | 环境间全量同步（Publish） | 基于实体直接比对的同步策略（Workflow/Layout/Agent） | `POST /v2/environments/:targetId/diff`、`POST /v2/environments/:targetId/publish` |

### 2.1 V1 与 V2 的关系

- V1 是 **细粒度** 的变更管理（每次编辑一条 Change）
- V2 是 **粗粒度** 的环境同步（按环境维度做整体 diff/publish）
- V2 的 `publishEnvironments` 内部最终也会调用 V1 的 `ApplyChange` 逻辑（对于非 Bridge 工作流）
- Dashboard 的 "Publish changes" 按钮使用 **V2 API**

---

## 三、变更创建（V1 Change 生成流程）

### 3.1 完整调用链

```
用户在 Dashboard 编辑模板并保存
    │
    ▼
PUT /notification-templates/:id  (API 入口)
    │
    ▼
UpdateWorkflowV0.execute() [update-workflow.usecase.ts]
    │
    ├─► 更新 NotificationTemplate 主记录（MongoDB $set）
    │
    ├─► 遍历 steps，对每个步骤调用 UpdateMessageTemplate.execute()
    │       │
    │       ├─► 更新 MessageTemplate 记录
    │       ├─► 调用 changeRepository.getChangeId() 生成或复用 Change ID
    │       └─► 调用 CreateChange.execute() 创建 MESSAGE_TEMPLATE 类型的 Change
    │
    ├─► 调用 changeRepository.getChangeId() 生成父 Change ID
    │
    └─► 调用 CreateChange.execute() 创建 NOTIFICATION_TEMPLATE 类型的 Change（父变更）
```

### 3.2 getChangeId 的关键逻辑

代码位置: [change.repository.ts L33-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/change/change.repository.ts#L33-L46)

```typescript
public async getChangeId(environmentId: string, entityType: ChangeEntityTypeEnum, entityId: string): Promise<string> {
  // 查找该实体在当前环境下是否存在未启用（enabled=false）的 Change
  const change = await this.findOne({
    _environmentId: environmentId,
    _entityId: entityId,
    type: entityType,
    enabled: false,
  });

  // 若存在，复用该 Change ID（多次编辑合并为同一条待发布变更）
  if (change?._id) {
    return change._id;
  }

  // 不存在则生成新的 ObjectId
  return BaseRepository.createObjectId();
}
```

**关键设计意图**:
- 同一实体在未发布前的多次编辑会**合并为同一条 Change**（通过复用 changeId 实现）
- 避免产生大量细碎的变更记录，保证发布时的原子性

### 3.3 CreateChange 的 Diff 计算逻辑

代码位置: [create-change.usecase.ts L17-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/usecases/create-change/create-change.usecase.ts#L17-L66)

```typescript
async execute(command: CreateChangeCommand) {
  const itemId = command.item._id;

  // 1. 拉取该实体所有已 enabled 的 Change 记录，按时间升序聚合出"当前已发布状态"
  const changes = await this.changeRepository.getEntityChanges(command.organizationId, command.type, itemId);
  const aggregatedItem = changes
    .filter((change) => change.enabled)
    .reduce((prev, change) => {
      const sanitized = sanitizeDiff(change.change);
      if (sanitized.length === 0) return prev;
      return applyDiff(prev, sanitized);
    }, {});

  // 2. 将"当前已发布状态"与"新提交的完整对象"做 diff，得到增量补丁
  const changePayload = getDiff(aggregatedItem, command.item, true);

  // 3. 若指定的 changeId 已存在（即复用场景），则更新其 change 字段（覆盖）
  const existingChange = await this.changeRepository.findOne({
    _environmentId: command.environmentId,
    _id: command.changeId,
  });

  if (existingChange) {
    existingChange.change = changePayload;
    await this.changeRepository.update(
      { _environmentId: command.environmentId, _id: command.changeId },
      { $set: existingChange }
    );
    return existingChange;
  }

  // 4. 否则创建新的 Change 记录，enabled=false（待发布）
  const item = await this.changeRepository.create({
    _organizationId: command.organizationId,
    _environmentId: command.environmentId,
    _creatorId: command.userId,
    change: changePayload,
    type: command.type,
    _entityId: itemId,
    enabled: false,
    _parentId: command.parentChangeId,
    _id: command.changeId,
  });

  return item;
}
```

**核心算法解析**:
- **baseline = 所有 enabled=true 的 Change 聚合结果**（代表当前已发布状态）
- **diff = getDiff(baseline, newState)**（得到从已发布状态到新状态的增量）
- 这意味着 **Change.change 字段始终是相对于"最后一次发布"的全量增量**，而非相对于上一次编辑
- 多次编辑合并时，新的 diff 会**覆盖**旧的 diff 字段（因为每次都是从 baseline 重新计算）

### 3.4 sanitizeDiff 安全过滤

```typescript
function sanitizeDiff(diff: unknown): rdiffResult[] {
  if (!Array.isArray(diff)) return [];
  return diff.filter((item) => item && Array.isArray(item.path));
}
```

过滤掉格式非法的 diff 项，防止恶意构造的补丁导致 `applyDiff` 出错。

---

## 四、变更应用与撤销（V1 Apply / Rollback）

### 4.1 ApplyChange 主流程

代码位置: [apply-change.usecase.ts L15-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts#L15-L83)

```typescript
async execute(command: ApplyChangeCommand): Promise<ChangeEntity[]> {
  // 1. 查找父 Change（通常是 NOTIFICATION_TEMPLATE 类型）
  const parentChange = await this.changeRepository.findOne({
    _id: command.changeId,
    _environmentId: command.environmentId,
    _organizationId: command.organizationId,
  });
  if (!parentChange) throw new NotFoundException('Parent Change not found');

  // 2. 查找所有子 Change（按 createdAt 升序，保证应用顺序正确）
  const changes = await this.changeRepository.find(
    { _environmentId: parentChange._environmentId, _parentId: parentChange._id },
    '',
    { sort: { createdAt: 1 } }
  );

  // 3. 先应用所有子 Change，最后应用父 Change
  const items: ChangeEntity[] = [];
  for (const change of [...changes, parentChange]) {
    const item = await this.applyChange(change, command);
    items.push(item);
  }

  return items;
}
```

### 4.2 单条 Change 的应用与失败处理

```typescript
async applyChange(change, command: ApplyChangeCommand): Promise<ChangeEntity> {
  if (!change) throw new NotFoundException();

  try {
    // 步骤 1: 先将 Change 标记为 enabled=true
    await this.changeRepository.update(
      {
        _id: change._id,
        _environmentId: command.environmentId,
        _organizationId: command.organizationId,
      },
      { enabled: true }
    );

    // 步骤 2: 执行实际的 Promote 逻辑（将变更同步到目标环境）
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
    // ⚠️  失败回滚：将 enabled 重新置为 false
    await this.changeRepository.update(
      {
        _id: change._id,
        _environmentId: command.environmentId,
        _organizationId: command.organizationId,
      },
      { enabled: false }
    );

    // 抛出原始异常，由上层处理
    throw e;
  }

  return change;
}
```

**失败处理机制的关键细节**:
1. **两阶段提交模型**: 先标记 `enabled=true`（变更状态），再执行实际同步
2. **自动回滚**: 若同步失败，**自动将 `enabled` 置回 false**，保证状态一致性
3. **无部分成功**: 父变更 + 子变更串行执行，任何一步失败都会终止流程
4. **已应用的子变更不会自动回滚**: 若父变更 Promote 失败，已成功的子变更仍保持 `enabled=true`，这是一个潜在的一致性风险

### 4.3 BulkApplyChange —— 批量应用

代码位置: [bulk-apply-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/bulk-apply-change/bulk-apply-change.usecase.ts)

```typescript
async execute(command: BulkApplyChangeCommand): Promise<ChangeEntity[][]> {
  const changes = await this.changeRepository.find(
    { _id: { $in: command.changeIds }, ... },
    '',
    { sort: { createdAt: 1 } }
  );

  const results: ChangeEntity[][] = [];
  for (const change of changes) {
    // 逐条调用 ApplyChange，串行执行
    const item = await this.applyChange.execute(
      ApplyChangeCommand.create({ changeId: change._id, ... })
    );
    results.push(item);
  }

  return results;
}
```

### 4.4 撤销（回滚）的实现方式

当前代码中**没有显式的 "撤销/回滚" API**，但可以通过以下方式实现：

**方式 A: 利用 enabled 标记**
- 将已 enabled 的 Change 重新置为 `enabled=false`
- 下次 Promote 时，该 Change 会被过滤掉（只聚合 enabled=true 的）
- **但这不会自动撤销已经 Promote 到目标环境的实体**，需要重新 Promote 其他变更来覆盖

**方式 B: 生成反向 Diff**
- 对目标环境实体和源环境历史状态做 `getDiff`，生成反向补丁
- 创建新的 Change 记录并 Promote

**方式 C: 使用 V2 Environments API 重新发布**
- 调用 `POST /v2/environments/:targetId/publish`，让源环境的最新状态覆盖目标环境

### 4.5 PromoteChangeToEnvironment 的聚合逻辑

代码位置: [promote-change-to-environment.usecase.ts L40-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-change-to-environment/promote-change-to-environment.usecase.ts#L40-L91)

```typescript
// 从 ChangeRepository 拉取该实体的所有 Change 记录
const changes = await this.changeRepository.getEntityChanges(
  command.organizationId, command.type, command.itemId
);

// 仅聚合已 enabled 的变更，按顺序 applyDiff
const aggregatedItem = changes
  .filter((change) => change.enabled)
  .reduce((prev, change) => {
    const sanitized = sanitizeDiff(change.change);
    if (sanitized.length === 0) return prev;
    return applyDiff(prev, sanitized);
  }, {});
```

关键要点:
1. `applyDiff` 来自 `recursive-diff` 库，将补丁顺序应用到空对象 `{}` 上
2. **Change 记录是增量补丁，但 Promote 时是全量覆盖** —— 聚合后得到的是最终状态
3. 空对象 `{}` 作为起点意味着每个字段的首次出现必须是 `op: 'add'`，否则会丢失

---

## 五、后台 Diff 展示（Dashboard V2 流程）

### 5.1 完整调用链

```
Dashboard PublishButton 组件
    │
    ├─► useDiffEnvironments() Hook
    │      │
    │      └─► POST /v2/environments/:targetId/diff
    │                 │
    │                 ▼
    │       DiffEnvironmentUseCase.execute()
    │                 │
    │                 ├─► WorkflowSyncStrategy.diff()
    │                 │      └─► WorkflowDiffOperation.execute()
    │                 │           ├─► 从 WorkflowDataContainer 加载两边环境的工作流
    │                 │           ├─► 按 _parentId/triggers[0].identifier 配对
    │                 │           └─► WorkflowComparatorAdapter.compareResources()
    │                 │                 └─► 生成 IResourceDiffResult[]（含 changes 数组 + summary）
    │                 ├─► LayoutSyncStrategy.diff()
    │                 └─► AgentSyncStrategy.diff()
    │
    └─► 展示变更列表（PublishModal 组件）
          ├─► 显示每个资源的变更类型（新增/修改/删除）
          ├─► 显示变更数量统计（added/modified/deleted）
          └─► 用户勾选后调用 usePublishEnvironments() 执行发布
```

### 5.2 PublishButton 组件逻辑

代码位置: [publish-button.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/dashboard/src/components/header-navigation/publish-button.tsx)

```typescript
// 自动获取与目标环境的 diff
const { data: diffData, isLoading: isDiffLoading } = useDiffEnvironments({
  sourceEnvironmentId: currentEnvironment?._id,
  targetEnvironmentId: targetEnvironment?._id,
  enabled: !!targetEnvironment?._id && !!currentEnvironment?._id,
});

// 计算变更总数
const changesCount = calculateChangesCount(diffData);

// 发布时调用 V2 API
const handlePublish = async (selectedResources?: ResourceToPublish[]) => {
  const result = await publishMutation.mutateAsync({
    sourceEnvironmentId: currentEnvironment._id,
    targetEnvironmentId: state.selectedEnvironment._id,
    resources: selectedResources,
  });
};
```

### 5.3 V2 Diff API 返回结构

定义位置: [environments.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/dashboard/src/api/environments.ts)

```typescript
interface IResourceDiffResult {
  resourceType: string;                     // 'workflow' | 'layout' | 'step' | 'agent'
  sourceResource?: IResourceInfo | null;    // 源环境资源信息
  targetResource?: IResourceInfo | null;    // 目标环境资源信息
  changes: any[];                           // 具体的变更补丁数组
  summary: IDiffSummary;                    // { added, modified, deleted, unchanged }
  dependencies?: IResourceDependency[];     // 依赖关系（如 workflow 依赖的 layout）
}

interface IEnvironmentDiffResponse {
  sourceEnvironmentId: string;
  targetEnvironmentId: string;
  resources: IResourceDiffResult[];
  summary: {
    totalEntities: number;
    totalChanges: number;
    hasChanges: boolean;
  };
}
```

### 5.4 V2 Diff 与 V1 Change 的区别

| 维度 | V1 Change | V2 Diff |
|------|-----------|---------|
| 计算时机 | 编辑时实时计算（保存即生成） | 查询时按需计算（调用 API 时才比对） |
| 存储方式 | 持久化到 MongoDB `changes` 集合 | 不存储，每次调用实时计算 |
| 粒度 | 单实体单字段级增量 | 资源级（工作流/布局/代理）整体比对 |
| 格式 | `rdiffResult[]` 递归补丁 | 自定义 `IResourceDiffResult` 结构 |
| 用途 | 增量发布（按变更发布） | 展示变更预览 + 全量/部分发布 |

### 5.5 WorkflowDiffOperation 的比对逻辑

代码位置: [workflow-diff.operation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/sync-strategies/operations/workflow-diff.operation.ts)

核心流程:
1. 从 `WorkflowDataContainer` 预加载源/目标环境的所有 Bridge 工作流
2. 按 `triggers[0].identifier`（即 workflowId）配对源/目标工作流
3. 调用 `WorkflowComparatorAdapter.compareResources()` 做详细比对
4. 处理新增（源有目标无）、修改（两边都有）、删除（源无目标有）三种情况
5. 分析依赖关系（如 workflow 依赖的 layout 是否在目标环境存在）

---

## 六、通知模板 Promote 的特殊逻辑（V1 核心）

代码位置: [promote-notification-template-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts)

### 6.1 MessageTemplate ID 映射

```typescript
const mapNewStepItem = (step: NotificationStepEntity) => {
  // 通过 _parentId 查找目标环境中对应的消息模板
  const oldMessage = messages.find((message) => message._parentId === step._templateId);

  if (!oldMessage) {
    missingMessages.push(step._templateId);
    return undefined;  // 静默过滤缺失的步骤
  }

  // 将 Dev 环境的 _templateId 替换为 Prod 环境的 _id
  if (step?._templateId && oldMessage._id) {
    step._templateId = oldMessage._id;
  }

  return step;
};
```

### 6.2 NotificationGroup 依赖处理

```typescript
let notificationGroup = await this.notificationGroupRepository.findOne({
  _environmentId: command.environmentId,
  _organizationId: command.organizationId,
  _parentId: newItem._notificationGroupId,
});

// 若通知组在目标环境不存在，先递归 Promote 通知组的所有变更
if (!notificationGroup) {
  const changes = await this.changeRepository.getEntityChanges(
    command.organizationId,
    ChangeEntityTypeEnum.NOTIFICATION_GROUP,
    newItem._notificationGroupId
  );

  for (const change of changes) {
    await this.applyChange.execute(ApplyChangeCommand.create({ changeId: change._id, ... }));
  }

  // 重新查找
  notificationGroup = await this.notificationGroupRepository.findOne({ ... });
}
```

### 6.3 创建 vs 更新分支

```typescript
if (!item) {
  // 目标环境不存在对应模板 → 创建新副本
  if (newItem.deleted) return;  // 源已删除则无需创建

  const newNotificationTemplate: Partial<NotificationTemplateEntity> = {
    name: newItem.name,
    active: newItem.active,
    steps,
    _parentId: command.item._id,  // 设置跨环境关联
    ...
  };

  const createdTemplate = await this.notificationTemplateRepository.create(newNotificationTemplate);
  await this.updateWorkflowPreferences(createdTemplate._id, command, ...);

  return createdTemplate;
} else {
  // 目标环境已存在 → 全量更新
  await this.notificationTemplateRepository.update(
    { _id: item._id, _environmentId: command.environmentId },
    { $set: { name: newItem.name, active: newItem.active, steps, ... } }
  );
}
```

---

## 七、发送链路中的模板版本选择

完整发送链路: **Trigger API → ParseEventRequest → WorkflowQueue → RunJob → SendMessage → 各渠道 Provider**

### 7.1 Trigger 入口阶段

代码位置: [parse-event-request.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts#L89-L230)

```typescript
// 按 trigger identifier + 当前 environmentId 查找模板
const template = await this.getNotificationTemplateByTriggerIdentifier({
  environmentId: command.environmentId,
  triggerIdentifier: command.identifier,
});

if (!template) throw new UnprocessableEntityException('workflow_not_found');

// 若模板启用了 payloadSchema 校验，则在此阶段验证变量合法性
if (template.validatePayload && template.payloadSchema) {
  const validatedPayload = this.validateAndApplyPayloadDefaults(
    command.payload, template.payloadSchema
  );
  command.payload = validatedPayload;
}
```

**关键点**：
- 模板查找严格依赖 `environmentId`，这确保了 Dev/Prod 环境天然隔离
- `payloadSchema` 校验是**变量兼容**的第一道防线，使用 AJV 做 JSON Schema 校验并自动填充默认值
- 找到模板后，模板的 `_id` 被写入 Job 数据，供后续链路使用

### 7.2 Job 执行阶段（RunJob）

代码位置: [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L427-L453)

```typescript
private async getWorkflow(templateId, environmentId, organizationId, source?) {
  // 带 LRU 缓存: key = `${environmentId}:${templateId}`
  const workflow = await this.inMemoryLRUCacheService.get(
    InMemoryLRUCacheStore.WORKFLOW,
    `${environmentId}:${templateId}`,
    async () => await this.notificationTemplateRepository.findById(templateId, environmentId)
  );
  if (!workflow) throw new NotFoundException(`Workflow ${templateId} not found`);
  return workflow;
}
```

**关键点**：
- 查找键是 `(environmentId, templateId)`，这是模板在**当前环境下的实际 ID**
- 使用了内存 LRU 缓存，意味着刚 Promote 的新模板版本需要等缓存失效或主动失效（见 `InvalidateCacheService`）
- **若模板正在被渐进切换，此处缓存会导致旧版本继续被使用一段时间**

### 7.3 消息发送阶段（SendMessage）

代码位置: [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts#L334-L340)

```typescript
// 偏好评估时可能再次获取模板
const workflow = command.workflow ??
  (await this.getWorkflow({ _id: job._templateId, environmentId: job._environmentId }));
```

此阶段主要用模板做 **偏好判断（Subscriber Preference）**，不直接读取内容。实际模板内容（subject、content 等）已在 Job 创建时嵌入 `job.step.template`。

### 7.4 链路总结

```
Trigger Request
    │
    ▼
ParseEventRequest  ──►  按 (envId, triggerIdentifier) 查 Template ──► 校验 payloadSchema
    │
    ▼
Create Notification + Jobs  ──►  job._templateId = 模板当前环境 _id
    │                                 job.step.template = 嵌入模板内容快照
    ▼
Workflow Queue (BullMQ)
    │
    ▼
RunJob  ──►  getWorkflow(envId, templateId) 带 LRU 缓存
    │         └──► 用于偏好判断、步骤调度
    ▼
SendMessage  ──►  使用 job.step.template 的快照内容渲染
    │
    ▼
各渠道 Provider (SendGrid/Twilio/APNs 等)
```

**模板版本与发送链路的耦合点**:
1. `job._templateId` 在 Job 创建时就已固化，**Job 生命周期内不会改变模板版本**
2. 因此，正在队列中等待的 Job 不会受后续模板更新/Promote 影响
3. 新模板版本只影响 **Job 创建之后** 触发的新通知

---

## 八、变量兼容机制

### 8.1 三层变量校验

| 层级 | 位置 | 机制 |
|------|------|------|
| L1 Trigger 入口 | `ParseEventRequest` | `payloadSchema` + AJV JSON Schema 校验 + 默认值填充 |
| L2 模板变量列表 | `MessageTemplateEntity.variables` | 定义模板所需变量，但仅用于编辑器提示 |
| L3 运行时渲染 | `SendMessage*` 各渠道 | Handlebars/Liquid 模板引擎渲染，缺失变量默认渲染为空字符串 |

### 8.2 payloadSchema 的设计意图

```typescript
// parse-event-request.usecase.ts L138-L157
if (template.validatePayload && template.payloadSchema) {
  const validatedPayload = this.validateAndApplyPayloadDefaults(command.payload, template.payloadSchema);
  command.payload = validatedPayload;
}
```

- 模板编辑者可在 Dashboard 定义 `payloadSchema`（JSON Schema）
- `validatePayload` 开关控制是否强制校验
- 校验通过后，AJV 的 `useDefaults: true` 会自动为缺失字段填充默认值
- **这是版本升级时变量兼容的核心保障**：新版本新增变量时，只要在 Schema 中定义了 `default`，旧 trigger 调用方不传新字段也能正常工作

### 8.3 变量不兼容风险场景

1. **删除必填变量**: 旧 trigger 仍在传该变量不会出错，但模板渲染逻辑可能隐含依赖
2. **改变变量类型**: 如 `amount` 从 number 改为 string，Schema 校验会拦截旧 payload
3. **修改默认值**: 旧 trigger 不传该变量时行为会变化，需要在 Release Notes 中明确

---

## 九、影子发布、渐进切换与发送链路的关系

### 9.1 当前实现中的版本切换模型

Novu 当前采用 **双环境（Dev → Prod）硬切换** 模型，而非逐步流量迁移的灰度模型：

```
Dev Environment  ──────promote/apply──────►  Production Environment
     │                                            │
     ▼                                            ▼
  编辑产生 Change                           所有 trigger 使用
  （enabled=false 未应用）                  最新 Promote 版本
```

**现状限制**:
- 没有"同一环境内多版本并存"的机制
- 没有"按百分比/按用户标签路由到不同模板版本"的能力
- 模板 ID 在 Job 创建时已固化，无法中途切换

### 9.2 若实现影子发布/渐进切换需要改造的链路节点

| 功能 | 需改造位置 | 改造思路 |
|------|-----------|---------|
| **影子发布（Shadow Release）** | `ParseEventRequest`、`SendMessage` | trigger 时同时向新旧两个版本各发一份，旧版本真实发送、新版本仅日志/埋点对比输出 |
| **渐进切换（Canary Rollout）** | `ParseEventRequest.getNotificationTemplateByTriggerIdentifier` | 按比例/租户/用户标签选择 templateId（旧版 or 新版），写入 Job |
| **版本绑定 Job** | 无需改动 | 当前 `job._templateId` 已天然具备 Job 级版本绑定语义 |
| **缓存失效** | `RunJob.getWorkflow` 的 LRU Cache | 渐进切换期间需关闭缓存或加版本号做 key，避免旧版本缓存命中 |
| **多版本存储** | `NotificationTemplateEntity` | 新增 `version` 或 `variant` 字段，同一 trigger identifier 下可存在多个版本 |
| **回滚原子性** | `PromoteChangeToEnvironment` | 当前是全量聚合 applyDiff，回滚需生成反向 Change 并重新 Promote |

### 9.3 渐进切换与发送链路的时序关系

```
                          ┌──────────────────────────────────────┐
                          │     Canary Router (新增逻辑)          │
 Trigger Request ───────► │  按 tenantId / subscriberId / 百分比  │
                          │  路由到 template_v1 或 template_v2    │
                          └──────────────┬───────────────────────┘
                                         │
                                         ▼
                              Create Job (_templateId=vX)
                                         │
                                         ▼
                              Workflow Queue (BullMQ)
                                         │
                      ┌──────────────────┴──────────────────┐
                      │                                     │
                      ▼                                     ▼
             RunJob (getWorkflow v1)              RunJob (getWorkflow v2)
                      │                                     │
                      ▼                                     ▼
             SendMessage (v1 content)             SendMessage (v2 content)
```

---

## 十、各概念的关系矩阵与完整关系图

### 10.1 关系矩阵

| 概念 | 核心机制 | 影响发送链路 | 影响后台展示 | 涉及主要文件 |
|------|---------|-------------|-------------|-------------|
| **Diff (V1)** | `recursive-diff` 的 rdiffResult 补丁，存于 `ChangeEntity.change`，编辑时相对于 baseline 计算 | 间接影响：聚合后成为新版本模板内容 | 间接：通过 V2 Diff API 展示 | [create-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/usecases/create-change/create-change.usecase.ts) |
| **Diff (V2)** | 调用 API 时实时比对源/目标环境实体，生成 `IResourceDiffResult` | 无直接影响（仅展示用） | 直接：Dashboard PublishModal 展示变更列表 | [diff-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/diff-environment/diff-environment.usecase.ts) |
| **变更创建** | `UpdateWorkflowV0` → `UpdateMessageTemplate` → `CreateChange`，多次编辑合并为同一条 Change | 无（仅创建待发布变更） | 展示为 Changes 列表中的待发布项 | [update-workflow.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/usecases/update-workflow-v0/update-workflow.usecase.ts) |
| **变更应用** | `ApplyChange` 先置 enabled=true 再 Promote，失败自动回滚 enabled=false | 直接影响：Promote 后目标环境 trigger 使用新版本 | 展示为 Changes 列表中 enabled 状态变为 true | [apply-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts) |
| **回滚** | 将 Change.enabled 置 false 或生成反向 Diff 重新 Promote | 影响后续新 trigger 使用的模板版本（已入队 Job 不受影响） | 展示为 Changes 列表中 enabled 状态切换 | 无独立 API，通过 ApplyChange + enabled 标记实现 |
| **变量兼容** | `payloadSchema` JSON Schema + AJV 默认值填充 + 模板引擎兜底 | Trigger 入口校验失败直接拒绝；运行时渲染缺变量为空字符串 | Dashboard 编辑器提供 Schema 定义 UI | [parse-event-request.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts) |
| **影子发布** | 当前未原生实现，需双版本并行触发并对比 | 需在 SendMessage 层面增加"仅记录不真实发送"分支 | 需新增影子发布结果对比页面 | 需新建 |
| **渐进切换** | 当前未原生实现，需按流量比例路由 templateId | ParseEventRequest 中选择版本 → 写入 Job._templateId → 后续链路天然跟随 | 需新增版本流量配置 UI + 指标看板 | 需新建 |
| **发送链路** | Trigger → ParseEvent → Queue → RunJob → SendMessage → Provider | **版本选择的实际发生处**，是所有版本策略的落点 | 展示 Activity Feed / Execution Details 中的执行结果 | [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts), [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts) |

### 10.2 完整关系图

```
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     DEV ENVIRONMENT                                        │
└────────────────────────────────────────────────────────────────────────────────────────────┘
                                            │
┌───────────────────────────────────────────┼────────────────────────────────────────────────┐
│  Dashboard 编辑模板                        │                                                │
│    │                                       │                                                │
│    ▼                                       │                                                │
│  UpdateWorkflowV0.execute()                │                                                │
│    ├─► 更新 NotificationTemplate           │                                                │
│    ├─► 遍历 steps → UpdateMessageTemplate  │                                                │
│    │     └─► getChangeId()                 │                                                │
│    │          └─► 复用或生成 Change ID     │                                                │
│    └─► CreateChange.execute()              │                                                │
│         ├─► 聚合 enabled=true 的 Change    │                                                │
│         ├─► getDiff(baseline, newState)   │                                                │
│         └─► 保存 Change (enabled=false)    │                                                │
│                                            │                                                │
│  Dashboard Publish 按钮                     │                                                │
│    │                                       │                                                │
│    ├─► useDiffEnvironments()               │                                                │
│    │    └─► POST /v2/environments/diff      │                                                │
│    │         └─► DiffEnvironmentUseCase    │                                                │
│    │              ├─► WorkflowSyncStrategy │                                                │
│    │              ├─► LayoutSyncStrategy   │                                                │
│    │              └─► AgentSyncStrategy    │                                                │
│    │                                          │                                            │
│    └─► 确认发布 → usePublishEnvironments()   │                                            │
│         └─► POST /v2/environments/publish    │                                            │
│              └─► PublishEnvironmentUseCase   │                                            │
│                   └─► 各 SyncStrategy.execute │                                            │
│                        └─► ApplyChange.execute() ───────────┐                               │
│                             ├─► enabled=true                │                               │
│                             ├─► PromoteChangeToEnvironment  │                               │
│                             │    └─► 聚合 Change → applyDiff │                               │
│                             └─► 失败 → enabled=false (回滚)  │                               │
└──────────────────────────────────────────────────────────────┼───────────────────────────────┘
                                                               │
                                                               │ 跨环境 Promote
                                                               │
┌──────────────────────────────────────────────────────────────┼───────────────────────────────┐
│                          PRODUCTION ENVIRONMENT              │                               │
└──────────────────────────────────────────────────────────────┼───────────────────────────────┘
                                                               │
                                                               ▼
                                                 NotificationTemplate (Prod)
                                                 ├─► _id = prod_tpl_456
                                                 ├─► _parentId = dev_tpl_123
                                                 └─► steps[]._templateId 已映射为 Prod 环境 ID
                                                               │
┌──────────────────────────────────────────────────────────────┼───────────────────────────────┐
│  发送链路                                                     │                               │
│    │                                                         │                               │
│    ▼                                                         │                               │
│  Trigger API /events/trigger                                 │                               │
│    │                                                         │                               │
│    ▼                                                         │                               │
│  ParseEventRequest                                            │                               │
│    ├─► getNotificationTemplateByTriggerIdentifier() ◄────────┘                               │
│    │     └─► 按 (envId=prod, triggerIdentifier) 查找                                         │
│    ├─► payloadSchema 校验 (AJV)                                                              │
│    └─► 创建 Notification + Jobs                                                              │
│         ├─► job._templateId = prod_tpl_456 (固化)                                            │
│         └─► job.step.template = 嵌入模板内容快照                                              │
│                                                                                               │
│    ▼                                                                                          │
│  Workflow Queue (BullMQ)                                                                      │
│    │                                                                                          │
│    ▼                                                                                          │
│  RunJob                                                                                       │
│    ├─► getWorkflow(envId, templateId) 带 LRU 缓存                                             │
│    └─► 调度步骤执行                                                                           │
│                                                                                               │
│    ▼                                                                                          │
│  SendMessage                                                                                  │
│    ├─► 使用 job.step.template 快照渲染                                                        │
│    └─► 调用各渠道 Provider                                                                    │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十一、关键风险与注意事项

### 11.1 LRU 缓存导致的版本不一致

`RunJob.getWorkflow()` 使用了 `InMemoryLRUCacheStore.WORKFLOW`，缓存 key 为 `${environmentId}:${templateId}`。

- 若模板内容原地更新（同一 templateId），旧缓存会导致短时间内使用旧版本
- Promote 操作应配合 `InvalidateCacheService` 清理缓存
- **渐进切换场景下必须禁用此缓存或加版本号后缀**

### 11.2 Job 快照与模板内容不一致

Job 创建时会将模板内容嵌入 `job.step.template`，后续 SendMessage 直接使用该快照。

- 已在队列中等待的 Job 不受模板更新影响
- 这保证了**单条通知的原子性**（不会出现步骤 1 用旧版本、步骤 2 用新版本）
- 但也意味着**无法对已入队通知紧急切换模板版本**

### 11.3 跨环境 _parentId 断裂风险

Promote 流程依赖 `_parentId` 做 Dev/Prod 消息模板 ID 映射。若 Dev 环境中新增了步骤但未先 Promote 消息模板子变更，`missingMessages` 日志会记录缺失，该步骤会被静默过滤。

- 关键代码: [promote-notification-template-change.usecase.ts L80-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts#L80-L132)
- 风险: Prod 环境模板步骤数 < Dev 环境，且不会报错中断

### 11.4 payloadSchema 版本漂移

若 Dev 环境更新了 `payloadSchema` 但未 Promote，Prod 环境仍用旧 Schema 校验。

- Trigger API 在哪个环境调用，就用哪个环境的模板 Schema
- 这是**环境隔离的设计初衷**，但需要 CI/CD 流程确保 Schema 变更与业务调用方升级同步

### 11.5 ApplyChange 部分成功风险

父变更 + 子变更串行执行，若子变更 A 成功、子变更 B 失败：
- A 保持 `enabled=true`，B 被自动置回 `enabled=false`
- 已 Promote 到目标环境的 A 变更不会被自动回滚
- 需人工判断是否需要重新发布或做补偿操作

### 11.6 V1 与 V2 混用的一致性风险

- V1 Change 是**编辑时**生成，V2 Diff 是**查询时**计算
- 若存在 V1 未 enabled 的 Change，V2 Diff 会忽略这些变更（直接比对实体当前状态）
- 建议统一使用 V2 流程进行环境同步，避免状态不一致
