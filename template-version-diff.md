# 通知模板版本管理代码分析

## 一、核心数据模型

### 1.1 NotificationTemplateEntity —— 工作流模板实体

定义位置: [notification-template.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/notification-template/notification-template.entity.ts)

关键字段:

| 字段 | 类型 | 作用 |
|------|------|------|
| `_id` | string | 当前环境下模板的唯一 ID |
| `_parentId` | string? | **V1 跨环境关联键**。V1 Promote 时 Prod 环境副本的 `_parentId` 指向 Dev 原始 `_id` |
| `workflowId` | string? | **V2 跨环境关联键**（external identifier）。V2 Sync 时通过此字段在环境间匹配同一工作流 |
| `_environmentId` | string | 所属环境 ID，与 `_id` 共同构成复合主键逻辑 |
| `steps` | NotificationStepEntity[] | 步骤数组，每个步骤含 `_templateId` 关联 MessageTemplate |
| `active` | boolean | 模板是否激活（可被 trigger 触发） |
| `draft` | boolean | 是否为草稿状态 |
| `payloadSchema` | any | 触发时的 payload JSON Schema（用于变量校验） |
| `validatePayload` | boolean | 是否启用 payload 校验 |
| `origin` | ResourceOriginEnum? | 资源来源（NOVU_CLOUD 等），V2 仅同步 NOVU_CLOUD 来源 |
| `type` | string? | 工作流类型（BRIDGE 等），V2 Diff 仅处理 BRIDGE 类型 |

跨环境关联键对比:

```
V1 (_parentId 关联):                     V2 (workflowId 关联):
Dev Environment                           Dev Environment
  _id: dev_tpl_123                          _id: dev_tpl_123
  _parentId: undefined                      workflowId: "onboarding-email"
     ↓ promote                                 ↓ sync
Production Environment                      Production Environment
  _id: prod_tpl_456                          _id: prod_tpl_456
  _parentId: dev_tpl_123                     workflowId: "onboarding-email"
                                             _parentId: dev_tpl_123 (UpsertWorkflowUseCase 会设置)
```

### 1.2 MessageTemplateEntity —— 消息模板实体

定义位置: [message-template.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/message-template/message-template.entity.ts)

关键字段:

| 字段 | 类型 | 作用 |
|------|------|------|
| `_id` | string | 当前环境消息模板 ID |
| `_parentId` | string? | V1 跨环境关联键 |
| `stepId` | string? | **V2 步骤关联键**（external identifier），V2 通过 stepId 在环境间匹配同一步骤 |
| `type` | StepTypeEnum | 渠道类型（EMAIL/SMS/IN_APP/PUSH/CHAT 等） |
| `content` | string \| IEmailBlock[] | 模板内容 |
| `variables` | ITemplateVariable[]? | 模板变量定义列表 |
| `controls` | ControlSchemas? | Bridge 工作流的控制 Schema |
| `subject`, `preheader`, `senderName` | string? | Email 专属字段 |

### 1.3 ChangeEntity —— 变更记录实体（仅 V1 使用）

定义位置: [change.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/change/change.entity.ts)
Schema 定义: [change.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/change/change.schema.ts)

这是 **V1 版本管理**的核心数据结构，V2 体系**完全不读取或写入**此集合:

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
interface rdiffResult {
  path: Array<string | number>;  // 变更路径，如 ["steps", "0", "template", "content"]
  op: 'add' | 'update' | 'delete';
  val?: any;                     // 新值（仅 add/update）
}
```

---

## 二、两套版本管理体系（完全独立、无相互调用）

### 2.1 体系概览与边界

Novu 实际存在 **两套完全独立、互不调用** 的版本管理体系。以下事实来自代码核准：

| 维度 | **V1 ChangeEntity 模式** | **V2 Environments 模式** |
|------|--------------------------|--------------------------|
| **API 控制器** | [ChangesController](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/changes.controller.ts) 路由: `/changes/*` | [EnvironmentsController](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/environments.controller.ts) 路由: `/v2/environments/*` |
| **核心入口** | ApplyChange / BulkApplyChange | PublishEnvironmentUseCase / DiffEnvironmentUseCase |
| **版本数据载体** | ChangeEntity（`changes` MongoDB 集合） | 无中间存储，直接操作实体集合 |
| **跨环境匹配键** | `_parentId`（MongoDB ObjectId 引用） | `workflowId` / `stepId`（business identifier） |
| **Diff 计算时机** | 编辑保存时生成，持久化 | 查询 API 时实时计算，不存储 |
| **同步单位** | 单条 Change（单实体单字段级） | 单资源整体（Workflow/Layout/Agent） |
| **中间状态** | `enabled: false`（待发布） | 无，直接同步 |
| **适用工作流** | 全部传统工作流 | `origin = NOVU_CLOUD` 且 `type = BRIDGE` 的工作流 |
| **Dashboard 入口** | Changes 列表页 | PublishButton / PublishModal |
| **是否互相调用** | —— | **否**，V2 完全不经过 V1 Change 体系 |

### 2.2 两套体系的独立调用链（关键：V2 不调用 V1）

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            V1 ChangeEntity 模式                                 │
│  (传统工作流 / Single Workflow Promote / Changes 列表页)                         │
└─────────────────────────────────────────────────────────────────────────────────┘
  PUT /notification-templates/:id
      │
      ▼
  UpdateWorkflowV0
      ├─► getChangeId() —— 复用或生成 Change ID
      ├─► UpdateMessageTemplate → CreateChange (MESSAGE_TEMPLATE, enabled=false)
      └─► CreateChange (NOTIFICATION_TEMPLATE, enabled=false)
                                     │
                                     ▼
                          POST /changes/:changeId/apply
                                     │
                                     ▼
                          ApplyChange.execute()
                              ├─► enabled = true (先标记)
                              ├─► PromoteChangeToEnvironment
                              │     ├─► 聚合 enabled=true 的 Change
                              │     ├─► applyDiff({}, ...) → 还原目标状态
                              │     └─► 按 type 分发:
                              │           ├─► PromoteNotificationTemplateChange
                              │           ├─► PromoteMessageTemplateChange
                              │           ├─► PromoteLayoutChange
                              │           ├─► PromoteNotificationGroupChange
                              │           └─► ... 其他类型
                              └─► 失败 → enabled = false (回滚标记)


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         V2 Environments 模式（完全独立）                         │
│  (Bridge 工作流 / 环境级 Publish / Dashboard PublishButton)                      │
└─────────────────────────────────────────────────────────────────────────────────┘
  POST /v2/environments/:targetId/publish
      │
      ▼
  PublishEnvironmentUseCase.execute()
      │
      └─► 按顺序执行三种 SyncStrategy:
           │
           ├─► 1. WorkflowSyncStrategy.execute()
           │       └─► WorkflowSyncOperation.execute()   ← BaseSyncOperation
           │             ├─► fetchSyncableResources (源 + 目标)
           │             ├─► determineSyncDecisions (ComparatorAdapter)
           │             ├─► syncResources:
           │             │     └─► WorkflowSyncAdapter.syncResourceToTarget()
           │             │           └─► SyncToEnvironmentUseCase.execute()
           │             │                 ├─► getWorkflowUseCase (源环境)
           │             │                 ├─► 先递归同步引用的 Layout → LayoutSyncToEnvironmentUseCase
           │             │                 ├─► buildRequestDto (map steps, preferences)
           │             │                 ├─► findWorkflowInTarget (通过 workflowId 匹配)
           │             │                 ├─► UpsertWorkflowUseCase.execute() ← 核心写入
           │             │                 ├─► syncStepResolver (if feature flag)
           │             │                 └─► publishTranslationGroup (if enterprise)
           │             └─► handleDeletedResources:
           │                   └─► WorkflowDeleteAdapter.deleteResourceFromTarget()
           │
           ├─► 2. LayoutSyncStrategy.execute()
           │       └─► LayoutSyncOperation.execute()
           │             ├─► LayoutSyncAdapter → LayoutSyncToEnvironmentUseCase
           │             └─► LayoutDeleteAdapter
           │
           └─► 3. AgentSyncStrategy.execute()
                 └─► AgentSyncOperation.execute()
                       ├─► AgentSyncAdapter → SyncAgentToEnvironment
                       └─► AgentDeleteAdapter
```

**代码事实核准确认**:
- [WorkflowSyncAdapter](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/sync-strategies/adapters/workflow-sync.adapter.ts) L9-L21: 直接注入 `SyncToEnvironmentUseCase`，无任何 ApplyChange 引用
- [LayoutSyncAdapter](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/sync-strategies/adapters/layout-sync.adapter.ts) L10-L22: 直接注入 `LayoutSyncToEnvironmentUseCase`
- [AgentSyncAdapter](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/sync-strategies/adapters/agent-sync.adapter.ts) L12-L25: 直接注入 `SyncAgentToEnvironment`
- [PublishEnvironmentUseCase](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/publish-environment/publish-environment.usecase.ts) L57-L59: 直接注入三种 Strategy，无 V1 引用

---

## 三、变更创建（仅 V1 Change 生成流程）

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

**设计意图**:
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

---

## 四、变更应用与撤销（仅 V1 Apply / Rollback）

### 4.1 ChangesController API 入口

代码位置: [changes.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/changes.controller.ts)

| 路由 | 方法 | 用途 |
|------|------|------|
| `GET /changes/` | GetChanges | 分页获取变更列表（按 promoted 过滤） |
| `GET /changes/count` | CountChanges | 获取未发布变更数量 |
| `POST /changes/:changeId/apply` | ApplyChange | 应用单条变更 |
| `POST /changes/bulk/apply` | BulkApplyChange | 批量应用变更 |

### 4.2 ApplyChange 主流程

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

### 4.3 单条 Change 的应用与失败处理

```typescript
async applyChange(change, command: ApplyChangeCommand): Promise<ChangeEntity> {
  if (!change) throw new NotFoundException();

  try {
    // 阶段 1: 先将 Change 标记为 enabled=true（状态标记）
    await this.changeRepository.update(
      {
        _id: change._id,
        _environmentId: command.environmentId,
        _organizationId: command.organizationId,
      },
      { enabled: true }
    );

    // 阶段 2: 执行实际的 Promote 逻辑（将变更同步到目标环境）
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
    // ⚠️  失败回滚：将 enabled 重新置为 false（仅回滚标记，不回滚已 Promote 的实体）
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
2. **自动回滚仅限状态标记**: 若同步失败，**自动将 `enabled` 置回 false**，但不回滚已写入目标环境的实际数据
3. **无部分成功自动补偿**: 父变更 + 子变更串行执行，若子变更 A 成功后子变更 B 失败，A 的目标环境实体变更**不会被自动回滚**
4. **无分布式事务**: 两阶段之间没有数据库事务绑定，极端情况下可能出现状态不一致

### 4.4 PromoteChangeToEnvironment 的聚合与分发

代码位置: [promote-change-to-environment.usecase.ts L40-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-change-to-environment/promote-change-to-environment.usecase.ts#L40-L91)

```typescript
async execute(command: PromoteChangeToEnvironmentCommand) {
  // 1. 从源环境拉取该实体的所有 Change 记录
  const changes = await this.changeRepository.getEntityChanges(command.organizationId, command.type, command.itemId);

  // 2. 仅聚合已 enabled 的变更，按顺序 applyDiff 到空对象
  const aggregatedItem = changes
    .filter((change) => change.enabled)
    .reduce((prev, change) => {
      const sanitized = sanitizeDiff(change.change);
      if (sanitized.length === 0) return prev;
      return applyDiff(prev, sanitized);
    }, {});

  // 3. 通过源环境的 _parentId 查找目标环境（假设源是 Dev，目标是 Prod）
  const environment = await this.environmentRepository.findOne({
    _parentId: command.environmentId,
  });
  if (!environment) throw new NotFoundException(...);

  // 4. 按类型分发给具体的 Promote 处理器
  const typeCommand = PromoteTypeChangeCommand.create({
    organizationId: command.organizationId,
    environmentId: environment._id,  // 目标环境 ID
    item: aggregatedItem,            // 聚合后的目标状态
    userId: command.userId,
  });

  switch (command.type) {
    case ChangeEntityTypeEnum.NOTIFICATION_TEMPLATE:
      await this.promoteNotificationTemplateChange.execute(typeCommand);
    case ChangeEntityTypeEnum.MESSAGE_TEMPLATE:
      await this.promoteMessageTemplateChange.execute(typeCommand);
    case ChangeEntityTypeEnum.LAYOUT:
    case ChangeEntityTypeEnum.DEFAULT_LAYOUT:
      await this.promoteLayoutChange.execute(typeCommand);
    case ChangeEntityTypeEnum.NOTIFICATION_GROUP:
      await this.promoteNotificationGroupChange.execute(typeCommand);
    // ... FEED / TRANSLATION / TRANSLATION_GROUP
  }
}
```

### 4.5 撤销（回滚）的实现方式

当前代码中**没有显式的 "撤销/回滚" API**，可通过以下方式实现：

**方式 A: enabled 标记撤销（最常用）**
- 将已 enabled 的 Change 重新置为 `enabled=false`
- 下次聚合时该 Change 会被过滤掉
- **但这不会自动撤销已经 Promote 到目标环境的实体数据**，需要通过新的反向 Change 覆盖

**方式 B: 生成反向 Diff**
- 对目标环境实体和源环境历史状态做 `getDiff`，生成反向补丁
- 创建新的 Change 记录并 Apply/Promote

**方式 C: 使用 V2 Environments API 重新同步**
- 调用 `POST /v2/environments/:targetId/publish`，V2 会将源环境的最新整体状态重新写入目标环境（整体覆盖，粒度更粗）

---

## 五、V2 环境发布链路详解（完全独立）

### 5.1 DiffEnvironmentUseCase —— 后台 Diff 展示计算

代码位置: [diff-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/diff-environment/diff-environment.usecase.ts)

**完整流程**:

```
POST /v2/environments/:targetId/diff
    │
    ▼
DiffEnvironmentUseCase.execute()
    │
    ├─► validateEnvironments() —— 校验源/目标环境合法性
    │
    ├─► 预加载数据（性能优化）
    │     └─► WorkflowDataContainer.loadWorkflowsWithControlValues()
    │           └─► 从 notification-template.repository.findWithTemplates()
    │                 加载条件:
    │                   - _environmentId: { $in: [sourceId, targetId] }
    │                   - origin: NOVU_CLOUD
    │                   - type: BRIDGE
    │
    ├─► 并行执行三种 Strategy.diff():
    │     │
    │     ├─► WorkflowSyncStrategy.diff()
    │     │     └─► WorkflowDiffOperation.execute()
    │     │           ├─► 从 WorkflowDataContainer 获取两边环境工作流
    │     │           ├─► 通过 triggers[0].identifier (即 workflowId) 配对源/目标
    │     │           ├─► WorkflowComparatorAdapter.compareResources()
    │     │           │     └─► 生成 IResourceDiffResult (含 changes + summary)
    │     │           └─► 处理新增/修改/删除三种情况
    │     │
    │     ├─► LayoutSyncStrategy.diff()
    │     │     └─► LayoutDiffOperation → LayoutComparatorAdapter
    │     │
    │     └─► AgentSyncStrategy.diff()
    │           └─► AgentDiffOperation → AgentComparatorAdapter
    │
    ├─► DependencyAnalyzerService.analyzeDependencies()
    │     └─► 分析资源间依赖（如 workflow → layout），补充到结果
    │
    └─► 计算 summary: { totalEntities, totalChanges, hasChanges }
```

**与 V1 Change 的关系**：V2 Diff **完全不读取 Change 集合**，直接比对实体实际状态。若存在 V1 未 enabled 的 Change（Dev 环境实体已被修改但 Change 未发布），V2 Diff **会把这些修改也纳入比对**——因为它看的是 Dev 环境实体的最新状态，而不是 Change 的 enabled 标记。

### 5.2 PublishEnvironmentUseCase —— 环境同步执行入口

代码位置: [publish-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/publish-environment/publish-environment.usecase.ts)

```typescript
async execute(command: PublishEnvironmentCommand): Promise<IPublishResult> {
  // 1. 环境校验 + 确定 sourceEnvironmentId（默认 Dev）
  // 2. 构造 ISyncContext
  // 3. 按固定顺序串行执行三种 Strategy:
  const strategies = [this.workflowSyncStrategy, this.layoutSyncStrategy, this.agentSyncStrategy];
  const results = await this.executeSync(strategies, syncContext);
  // 4. 汇总 summary
}
```

**注意**：当前代码中 `executeSync` 方法虽然有注释提到 `use transactions for atomicity`，但实际实现中**并未开启 MongoDB 事务**，各资源同步之间是独立的，没有全局事务保证。

### 5.3 BaseSyncOperation —— 同步决策通用框架

代码位置: [base-sync.operation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/sync-strategies/base/operations/base-sync.operation.ts)

这是所有资源类型同步的通用基类，流程如下:

```
BaseSyncOperation.execute(context)
    │
    ├─► fetchSyncableResources(sourceEnv) —— 获取源环境需同步的资源
    │
    ├─► filterResourcesForSelectiveSync() —— 如果指定了 resources，按 resourceType+resourceId 过滤
    │
    ├─► syncResources(context, sourceResources, resultBuilder)
    │     │
    │     ├─► fetchSyncableResources(targetEnv)
    │     ├─► createResourceMap(targetResources) —— Map<identifier, resource>
    │     │
    │     ├─► determineSyncDecisions() （分批次，每批 5 个并发）
    │     │     │
    │     │     └─► shouldSyncResource(resource, targetResource?)
    │     │           │
    │     │           ├─► if (!targetResource) → { sync: true, action: CREATED }
    │     │           │
    │     │           └─► ComparatorAdapter.compareResources(resource, targetResource)
    │     │                 ├─► resourceChanges !== null
    │     │                 ├─► otherDiffs.length > 0
    │     │                 └─► { sync: true, action: UPDATED } 或
    │     │                     { sync: false, reason: NO_CHANGES }
    │     │
    │     └─► 按决策串行执行:
    │           │
    │           ├─► sync=true  → SyncAdapter.syncResourceToTarget() → resultBuilder.addSuccess()
    │           ├─► sync=false → resultBuilder.addSkipped(reason)
    │           └─► 异常 → resultBuilder.addFailure() → throw
    │
    └─► handleDeletedResources(context, sourceResources, resultBuilder)
          │
          ├─► fetchSyncableResources(targetEnv)
          ├─► createResourceMap(sourceResources)
          └─► 对每个在目标环境但不在源环境中的资源:
                DeleteAdapter.deleteResourceFromTarget()
```

### 5.4 WorkflowSyncAdapter → SyncToEnvironmentUseCase

代码位置: [sync-to-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts)

这是 V2 工作流同步的**核心执行单元**，完全独立于 V1 体系:

```typescript
async execute(command: SyncToEnvironmentCommand): Promise<WorkflowResponseDto> {
  // 1. 校验: 不能同步到同一环境；目标环境必须存在
  // 2. getWorkflowUseCase() —— 从源环境加载完整工作流（含 steps、controlValues）
  // 3. isSyncable() 校验: 仅 NOVU_CLOUD 来源可同步
  // 4. getWorkflowPreferences() —— 加载 USER_WORKFLOW + WORKFLOW_RESOURCE 偏好
  // 5. findWorkflowInTargetEnvironment() —— 按 workflowId (external ID) 在目标环境查找
  //       ↑ 这是 V2 与 V1 的关键区别：V2 用 workflowId 匹配，V1 用 _parentId 匹配
  // 6. buildRequestDto() —— 根据是创建还是更新，映射为 UpsertWorkflowDataCommand
  //
  // 7. ⭐ 先递归同步依赖的 Layout（Email 步骤引用的 layoutId）
  const layoutsToSyncPromises = layoutsToSync.map((layoutId) =>
    this.layoutSyncToEnvironmentUseCase.execute(...)
  );
  await Promise.all(layoutsToSyncPromises);
  //
  // 8. 同步 Layout 的翻译（仅企业版）
  //
  // 9. ⭐ UpsertWorkflowUseCase.execute() —— 核心写入：创建或更新目标环境的工作流
  const upsertedWorkflow = await this.upsertWorkflowUseCase.execute(
    UpsertWorkflowCommand.create({
      preserveWorkflowId: true,           // 保持 workflowId 不变
      user: { ...user, environmentId: targetEnvId },  // 切换用户上下文到目标环境
      workflowIdOrInternalId: targetWorkflow?._id,   // 目标环境内部 ID（更新时用）
      workflowDto,
      session: command.session,
    })
  );
  // 10. syncStepResolver() —— 如果 feature flag 开启，同步步骤解析器
  // 11. publishTranslationGroup() —— 同步工作流翻译（仅企业版）
  // 12. updatePublishFields() —— 更新源工作流的发布元信息（lastPublishedAt 等）
  // 13. sendWebhookMessage() —— 发送 WORKFLOW_PUBLISHED webhook（可选）

  return upsertedWorkflow;
}
```

**V2 同步步骤 ID 映射逻辑** (mapStepsToCreateOrUpdateDto):

```typescript
sourceSteps.map((sourceStep) => {
  // 在目标环境步骤中，通过 stepId (external identifier) 匹配找到对应的内部 _id
  const targetStepInternalId = targetEnvSteps?.find(
    (targetStep) => targetStep.stepId === sourceStep.stepId
  )?._id;

  return {
    ...(targetStepInternalId && { _id: targetStepInternalId }),
    stepId: sourceStep.stepId,  // 保持 stepId 不变
    name: sourceStep.name ?? '',
    type: sourceStep.type,
    controlValues: sourceStep.controls?.values ?? {},
  };
});
```

**与 V1 PromoteNotificationTemplateChange 的对比**:

| 维度 | V1 PromoteNotificationTemplateChange | V2 SyncToEnvironmentUseCase |
|------|--------------------------------------|------------------------------|
| 工作流匹配 | 通过 `_parentId` 查找目标环境副本 | 通过 `workflowId` (external) 查找目标环境副本 |
| 步骤匹配 | 通过 `_parentId` 查找目标环境 MessageTemplate | 通过 `stepId` (external) 查找目标环境步骤，用 `_id` 写入 Upsert 命令 |
| 依赖处理 | PromoteNotificationGroup（递归 Apply NotificationGroup Change） | 同步引用的 Layout（调用 LayoutSyncToEnvironmentUseCase） |
| 偏好处理 | 手动同步 UserWorkflowPreferences + WorkflowResourcePreferences | 偏好作为 UpsertWorkflowCommand 的一部分由 UpsertWorkflowUseCase 统一处理 |
| 控制值同步 | 不涉及（Change 模式增量覆盖） | 通过 WorkflowDataContainer 预加载，在 UpsertWorkflowUseCase 中处理 |
| 翻译同步 | TRANSLATION Change 单独发布 | 企业版内置 publishTranslationGroup() |
| Webhook 通知 | 无 | 发送 WORKFLOW_PUBLISHED webhook 事件 |
| 事务 | 无 | 无（虽有 session 参数但未绑定完整事务） |

---

## 六、后台 Diff 展示与发送链路的关系

### 6.1 展示与发送的数据流关系

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                       后台展示层（Dashboard）                                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                                    │
  PublishButton 组件                                                                               │
    ├─► useDiffEnvironments() Hook                                                                 │
    │     └─► POST /v2/environments/:targetId/diff                                                 │
    │           └─► DiffEnvironmentUseCase                                                        │
    │                 └─► 直接比对源/目标环境实体（不经过 Change）                                   │
    │                                                                                               │
    └─► 确认发布 → usePublishEnvironments()                                                        │
          └─► POST /v2/environments/:targetId/publish                                              │
                └─► PublishEnvironmentUseCase                                                     │
                      └─► WorkflowSyncStrategy → SyncToEnvironmentUseCase                          │
                            └─► UpsertWorkflowUseCase → 写入 Prod 环境 NotificationTemplate       │
                                                                                                    │
                                                                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              发送链路（Worker）                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                                    │
  Trigger API /events/trigger                                                                       │
    │                                                                                               │
    ▼                                                                                               │
  ParseEventRequest                                                                                 │
    ├─► getNotificationTemplateByTriggerIdentifier()                                               │
    │     └─► 按 (envId=prod, triggerIdentifier) 查找 Prod 环境 NotificationTemplate               │
    │           ↑ 这个就是 V2 刚刚写入的目标环境实体！                                               │
    │                                                                                               │
    ├─► payloadSchema 校验（AJV + 默认值填充）                                                     │
    │                                                                                               │
    └─► 创建 Notification + Jobs                                                                   │
          ├─► job._templateId = Prod 环境模板的 _id  ← **版本在此刻固化**                          │
          └─► job.step.template = 嵌入模板内容快照                                                  │
                                                                                                    │
    ▼                                                                                               │
  Workflow Queue (BullMQ)                                                                           │
    │                                                                                               │
    ▼                                                                                               │
  RunJob                                                                                            │
    ├─► getWorkflow(envId, templateId)                                                             │
    │     └─► InMemoryLRUCacheStore.WORKFLOW —— key = `${envId}:${templateId}`                     │
    │                                                                                               │
    ▼                                                                                               │
  SendMessage                                                                                       │
    └─► 使用 job.step.template 快照内容渲染（不依赖 DB 实时版本）                                   │
```

**关键耦合点**:

1. **V2 Publish → Prod 环境实体写入**：SyncToEnvironmentUseCase 通过 UpsertWorkflowUseCase 直接在目标环境写入 NotificationTemplate，这是发送链路的数据源
2. **triggerIdentifier 匹配**：ParseEventRequest 通过 `triggers[0].identifier`（即 workflowId）查找模板，V2 同步时 `preserveWorkflowId: true` 保证匹配成功
3. **版本固化时机**：Job 创建时 `job._templateId` 和 `job.step.template` 快照固化，**V2 Publish 之后的变更不会影响已入队的 Job**
4. **LRU 缓存风险**：RunJob 的 `getWorkflow()` 有缓存，若 V2 Publish 是原地更新（同一 `_id` 覆盖内容），旧缓存需等失效

### 6.2 V1 vs V2 发布后发送链路的差异

| 场景 | V1 Change Promote 后 | V2 Environment Publish 后 |
|------|----------------------|--------------------------|
| 工作流匹配方式 | `_parentId` 关联，查找目标环境副本 | `workflowId` 匹配，查找目标环境副本 |
| trigger 能否立即找到 | ✅ 可以（目标环境副本 triggerIdentifier 不变） | ✅ 可以（`preserveWorkflowId: true` 保持不变） |
| 步骤内容映射 | V1 需重新映射 `step._templateId`（用 `_parentId` 找目标 MessageTemplate） | V2 的 UpsertWorkflowUseCase 通过 `stepId` 匹配并复用目标 MessageTemplate `_id` |
| LRU 缓存影响 | 是，若 `_id` 不变需失效缓存 | 是，同上 |
| 已入队 Job 受影响 | 否，Job 快照已固化 | 否，Job 快照已固化 |

---

## 七、发送链路中的模板版本选择（两层队列模型）

发送链路实际采用 **两层队列模型**，模板版本在两个阶段分别被查找和固化：

```
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                              API 进程（触发阶段）                                          │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                               │
  Trigger API /events/trigger                                                                 │
    │                                                                                         │
    ▼                                                                                         │
  ParseEventRequest.execute()                                                                 │
    ├─► getNotificationTemplateByTriggerIdentifier()  ◄─── 第1次模板查找：按 (envId, triggerId)
    │     └─► 找到后模板对象挂到 command.workflow 上                                          │
    ├─► payloadSchema 校验（AJV + 默认值填充）                                               │
    └─► dispatchEventToWorkflowQueue()                                                       │
          └─► workflowQueueService.add(jobData)                                              │
                └─► 入队 Workflow Queue (BullMQ/SQS)                                          │
                     jobData 中不含完整模板，只有 identifier + payload + actor 等元数据         │
                                                                                               │
                                                                                               ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                           Worker 进程（工作流队列消费阶段）                                  │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                               │
  WorkflowWorker (消费 Workflow Queue)                                                        │
    └─► triggerEventUsecase.execute()                                                         │
          ├─► getAndUpdateWorkflowById()  ◄─── 第2次模板查找：从 DB 取完整模板
          │     └─► 用 InMemoryLRUCacheService？❌ API 层用的是直接查询                       │
          ├─► verifyPayload（再次校验）                                                       │
          ├─► triggerMulticast / triggerBroadcast                                            │
          │     └─► 拆分为订阅者，分批送入 SubscriberProcessQueue                              │
          │                                                                                   │
          ▼                                                                                   │
  SubscriberProcessQueue → （消费后调用 CreateNotificationJobs）                               │
    └─► CreateNotificationJobs.execute()                                                      │
          ├─► 从 command.template 获取完整模板对象                                             │
          ├─► createNotification()  ← notification._templateId = template._id                 │
          ├─► filterActiveSteps()                                                             │
          └─► buildJobFromStep()                                                              │
                ├─► step: buildStepForJob(step, command) ← 模板内容嵌入 job.step               │
                ├─► _templateId: notification._templateId    ← 模板 ID 固化                   │
                └─► payload + overrides + ...                                                  │
                                                                                               │
          └─► StoreSubscriberJobs.execute()                                                   │
                ├─► jobRepository.storeJobs(jobs)  ← 写入 MongoDB jobs 集合                    │
                └─► addJob.execute()                                                          │
                      └─► standardQueueService.add(job)  ← 入队 Standard Queue                 │
                                                                                               │
                                                                                               ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                          Worker 进程（标准队列消费阶段）                                    │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                                                                               │
  Standard Queue Worker → RunJob.execute()                                                    │
    ├─► 从 DB 取 Job (jobRepository.findOne)                                                  │
    ├─► 从 DB 取 Notification                                                                 │
    ├─► getWorkflow()  ◄─── 第3次模板查找：用 LRU 缓存
    │     └─► key = `${environmentId}:${templateId}`                                          │
    │         TTL = 30 秒, max = 1000 条                                                      │
    │                                                                                         │
    └─► SendMessage.execute()                                                                 │
          └─► 使用 job.step.template 快照渲染（不依赖 DB 实时版本）                              │
                                                                                               │
                                                                                               ▼
                                                                         各渠道 Provider 发送
```

### 7.1 三层模板查找与版本固化时机

| 阶段 | 位置 | 查找方式 | 缓存 | 版本固化效果 |
|------|------|---------|------|-------------|
| **第1次: API 入口** | `ParseEventRequest` | `getNotificationTemplateByTriggerIdentifier(envId, triggerId)` | 无 | 确认模板存在，仅用于 payload 校验；入队时**不携带完整模板** |
| **第2次: Workflow Worker** | `TriggerEvent` → `getAndUpdateWorkflowById` | `notificationTemplateRepository.findById` | 无（直接查 DB） | 模板对象挂在 command 上，后续订阅者拆分和 Job 构建都用这份 |
| **第3次: Standard Queue** | `RunJob.getWorkflow()` | `notificationTemplateRepository.findById` | LRU 缓存（30s TTL） | 用于偏好判断和步骤调度；SendMessage 实际用 `job.step.template` 快照 |

**版本固化的真正发生点**：

```typescript
// CreateNotificationJobs.buildJobFromStep() [create-notification-jobs.usecase.ts L186-L212]
private buildJobFromStep(step, command, notification): NotificationJob {
  return {
    identifier: command.identifier,
    payload: command.payload,
    step: this.buildStepForJob(step, command),  // ⭐ 模板内容嵌入 job.step
    _templateId: notification._templateId,       // ⭐ 模板 ID 固化到 job 上
    // ... 其他字段
  };
}
```

- **`job._templateId`**：模板的环境内 ID，后续 RunJob 用此 ID 再查一次模板（带缓存）
- **`job.step.template`**：模板内容快照，SendMessage 直接使用此快照渲染
- **`notification._templateId`**：工作流级别的模板引用，保留触发时的版本

### 7.2 LRU 内存缓存的换新与失效机制

代码位置:
- 服务实现: [in-memory-lru-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.service.ts)
- 存储配置: [in-memory-lru-cache.store.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.store.ts)

**缓存配置**（WORKFLOW 存储）:

```typescript
// in-memory-lru-cache.store.ts L48-L52
[InMemoryLRUCacheStore.WORKFLOW]: {
  max: 1000,              // 最大缓存 1000 条
  ttl: THIRTY_SECONDS_MS, // TTL = 30 秒
  featureFlagComponent: 'workflow',
}
```

**核心 API**：

```typescript
// 读取缓存，未命中则调用 fetchFn 并填充
async get(storeName, key, fetchFn, opts): Promise<T>

// 主动失效指定 key（同时失效所有变体）
invalidate(storeName: InMemoryLRUCacheStore, key: string): void

// 清空整个 store
invalidateAll(storeName: InMemoryLRUCacheStore): void

// 缓存变体 key（用于多版本并存）
// key = `${baseKey}:v:${cacheVariant}`
```

**失效逻辑**（invalidate 方法）:

```typescript
// in-memory-lru-cache.service.ts L82-L93
invalidate(storeName, key): void {
  const store = STORES.get(storeName);
  if (!store) return;

  for (const cacheKey of store.cache.keys()) {
    // 删除精确匹配的 key，以及所有带变体后缀的 key
    // 形如 `${key}` 或 `${key}:v:${variant}`
    if (cacheKey === key || cacheKey.startsWith(`${key}:v:`)) {
      store.cache.delete(cacheKey);
    }
  }
}
```

**换新机制总结**：
1. **TTL 自动失效**：30 秒后缓存自动过期，下一次 get 会重新 fetch
2. **主动失效**：通过 `invalidate(WORKFLOW, templateId)` 立即清除指定模板的缓存
3. **缓存变体**：支持 `cacheVariant` 参数，可用于同一模板多版本并存场景（如渐进切换）
4. **inflight 请求合并**：同一时间对同 key 的多次 get 会复用同一个 fetch Promise，避免缓存击穿
5. **进程内隔离**：缓存是进程内的 Map，多实例部署时各实例独立失效

**注意**：目前代码中 V2 Publish 后**没有主动调用 invalidate 清理缓存**，新模板版本发布后，最长可能需要等待 30 秒（TTL）才能在 RunJob 中生效。

### 7.3 触发 API 到运行队列的数据流（版本视角）

```
Trigger Request
    │  identifier = "onboarding-email"
    │  payload = { name: "John" }
    │  environmentId = "prod_env"
    ▼
┌─────────────────────────────────────────┐
│ ParseEventRequest (API 进程)             │
│  ├─ 按 triggerId + envId 查模板          │
│  ├─ 找到 template = { _id: "prod_123", … } │
│  └─ payloadSchema 校验通过                │
│                                          │
│  ⚠️  入队 Workflow Queue 时，jobData 中  │
│     只有 identifier/payload/actor 等，    │
│     没有完整模板对象！                     │
│     Job 固化尚未发生                      │
└─────────────────────────────────────────┘
    │
    ▼  Workflow Queue (BullMQ + SQS)
    │
┌─────────────────────────────────────────┐
│ TriggerEvent (Worker 进程)               │
│  ├─ getAndUpdateWorkflowById(prod_123?) │
│  │   └─ 等等，这里用什么 ID 查？          │
│  │                                      │
│  └─ 实际: 用 identifier 再查一次模板      │ ← 第二次查找
│         └─ 拿到完整 NotificationTemplate
│                                          │
│  triggerMulticast → 拆分订阅者           │
│                                          │
│  CreateNotificationJobs:                 │
│    ├─ 用 command.template 构建 step      │
│    ├─ 模板内容嵌入 job.step.template      │ ← ⭐ 内容固化
│    └─ job._templateId = template._id     │ ← ⭐ ID 固化
│                                          │
│  StoreSubscriberJobs:                    │
│    ├─ jobRepository.storeJobs(jobs)      │ ← 写入 MongoDB
│    └─ standardQueueService.add(job)      │ ← 入队 Standard Queue
└─────────────────────────────────────────┘
    │
    ▼  Standard Queue
    │
┌─────────────────────────────────────────┐
│ RunJob (Worker 进程)                     │
│  ├─ 从 DB 取 Job 实体                     │
│  ├─ getWorkflow(job._templateId)         │ ← 第三次查找，带 LRU 缓存
│  │   └─ 用于偏好判断、步骤元数据           │
│  │                                      │
│  └─ SendMessage                           │
│       └─ 使用 job.step.template 渲染      │ ← 不查 DB，用快照
└─────────────────────────────────────────┘
```

**关键事实核准**：
- ParseEventRequest 的 `dispatchEventToWorkflowQueue` 入队时 **不包含完整模板**，只有 identifier + payload 等元数据
- 模板版本的真正固化发生在 **CreateNotificationJobs** 阶段（Worker 进程内）
- RunJob 中的 `getWorkflow()` 是**第三次**查找模板，用于偏好判断等辅助功能，而非获取发送内容
- SendMessage 最终使用的是 `job.step.template` 快照，不依赖 DB 实时版本

### 7.4 消息发送阶段（SendMessage）

代码位置: [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts#L334-L340)

```typescript
// 偏好评估时可能再次获取模板（优先使用 command 传入的 workflow，否则从 DB 查）
const workflow = command.workflow ??
  (await this.getWorkflow({ _id: job._templateId, environmentId: job._environmentId }));
```

此阶段主要用模板做 **偏好判断（Subscriber Preference）**，不直接读取内容。实际模板内容（subject、content 等）已在 Job 创建时嵌入 `job.step.template`。

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
                    V1 Change Promote
Dev Environment  ───────────────────────────►  Production Environment
     │                                               │
     │  V2 Environment Publish                      │
     └──────────────────────────────────────────────►│
                                                     ▼
                                              所有 trigger 使用
                                              最新 Promote/Publish 版本
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
| **回滚原子性** | V1: `ApplyChange` / V2: `SyncToEnvironmentUseCase` | V1 需解决部分成功回滚问题；V2 天然是整体覆盖，回滚即重新发布源环境的旧版本 |

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

### 10.1 关系矩阵（按代码事实核准）

| 概念 | 核心机制 | 影响发送链路 | 影响后台展示 | V1 体系 | V2 体系 | 涉及主要文件 |
|------|---------|-------------|-------------|----------|----------|-------------|
| **Diff (V1)** | `recursive-diff` 的 rdiffResult 补丁，存于 `ChangeEntity.change`，编辑时相对于 baseline 计算 | 间接影响：V1 Promote 聚合后成为新版本模板内容 | 间接：仅通过 V1 Changes 列表展示（非 Dashboard PublishModal） | ✅ 核心 | ❌ 不涉及 | [create-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/usecases/create-change/create-change.usecase.ts) |
| **Diff (V2)** | 调用 API 时实时比对源/目标环境实体，生成 `IResourceDiffResult`，不存储 | 无直接影响（仅展示用） | ✅ 直接：Dashboard PublishModal 展示变更列表 | ❌ 不读取 | ✅ 核心 | [diff-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/diff-environment/diff-environment.usecase.ts) |
| **变更创建** | `UpdateWorkflowV0` → `UpdateMessageTemplate` → `CreateChange`，多次编辑合并为同一条 Change | 无（仅创建待发布变更，V2 不受影响） | 展示为 Changes 列表中的待发布项 | ✅ 核心 | ❌ 不涉及 | [update-workflow.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/application-generic/src/usecases/update-workflow-v0/update-workflow.usecase.ts) |
| **变更应用 (V1)** | `ApplyChange` 先置 enabled=true 再 Promote，失败自动回滚 enabled=false | 直接影响：V1 Promote 后目标环境 trigger 使用新版本 | V1 Changes 列表 enabled 状态变 true | ✅ 核心 | ❌ 不涉及 | [apply-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts) |
| **环境发布 (V2)** | 各 SyncStrategy → BaseSyncOperation → 各自 SyncAdapter → Upsert 用例 | 直接影响：V2 Publish 写入 Prod 环境，后续 trigger 用新版本 | V2 Publish 按钮/Modal，展示 success/failure/skipped | ❌ 不涉及 | ✅ 核心 | [publish-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/environments-v2/usecases/publish-environment/publish-environment.usecase.ts) |
| **工作流同步 (V2)** | `SyncToEnvironmentUseCase` → `UpsertWorkflowUseCase`，按 workflowId/stepId 匹配 | V2 发布的实际执行者，写入 Prod 环境实体 | 作为 V2 Publish 的子流程无独立展示 | ❌ 不涉及 | ✅ 核心 | [sync-to-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts) |
| **回滚** | V1: 将 Change.enabled 置 false + 重新发布反向 Change<br>V2: 重新调用 V2 Publish（源环境回退到旧状态后整体覆盖） | 影响后续新 trigger 使用的模板版本（已入队 Job 不受影响） | V1: Changes 列表 enabled 状态切换<br>V2: 重新展示 Diff 结果 | V1 部分 | V2 部分 | 无独立 API |
| **变量兼容** | `payloadSchema` JSON Schema + AJV 默认值填充 + 模板引擎兜底 | Trigger 入口校验失败直接拒绝；运行时渲染缺变量为空字符串 | Dashboard 编辑器提供 Schema 定义 UI | ✅ 共用 | ✅ 共用 | [parse-event-request.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts) |
| **影子发布** | 当前未原生实现，需双版本并行触发并对比 | 需在 SendMessage 层面增加"仅记录不真实发送"分支 | 需新增影子发布结果对比页面 | 需新建 | 需新建 | 需新建 |
| **渐进切换** | 当前未原生实现，需按流量比例路由 templateId | ParseEventRequest 中选择版本 → 写入 Job._templateId → 后续链路天然跟随 | 需新增版本流量配置 UI + 指标看板 | 需新建 | 需新建 | 需新建 |
| **发送链路** | Trigger → ParseEvent → Queue → RunJob → SendMessage → Provider | **版本选择的实际发生处**，所有发布策略的最终落点 | 展示 Activity Feed / Execution Details 中的执行结果 | ✅ 共用 | ✅ 共用 | [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts) |

### 10.2 V1 Change 的使用边界总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        V1 ChangeEntity 使用范围                               │
└─────────────────────────────────────────────────────────────────────────────┘
  ✅  以下流程使用 V1 Change:
    ├─► UpdateWorkflowV0 / UpdateMessageTemplate —— 保存时创建 Change
    ├─► CreateChange —— 生成 rdiffResult 补丁并写入 changes 集合
    ├─► GetChanges / CountChanges —— Changes 列表页展示未发布/已发布变更
    ├─► ApplyChange / BulkApplyChange —— POST /changes/*/apply API
    ├─► PromoteChangeToEnvironment —— 聚合 enabled=true 的 Change 并分发
    ├─► PromoteNotificationTemplateChange / PromoteMessageTemplateChange
    ├─► PromoteLayoutChange / PromoteNotificationGroupChange
    ├─► PromoteFeedChange / PromoteTranslationChange / PromoteTranslationGroupChange
    └─► UpdateChange —— 修改 Change 元信息（V1 流程内部使用）

  ❌  以下流程完全不使用 V1 Change:
    ├─► DiffEnvironmentUseCase —— V2 Diff API（直接比对实体）
    ├─► PublishEnvironmentUseCase —— V2 Publish API
    ├─► WorkflowSyncStrategy / LayoutSyncStrategy / AgentSyncStrategy
    ├─► WorkflowSyncAdapter / LayoutSyncAdapter / AgentSyncAdapter
    ├─► SyncToEnvironmentUseCase —— V2 工作流同步核心
    ├─► LayoutSyncToEnvironmentUseCase
    ├─► SyncAgentToEnvironment
    ├─► UpsertWorkflowUseCase
    ├─► ParseEventRequest —— 发送链路入口
    ├─► RunJob —— 发送链路执行
    └─► Dashboard PublishButton / PublishModal —— 仅调用 V2 API

  ⚠️  半隔离场景:
    └─► 若某工作流被 V1 修改并创建了 Change (enabled=false)，但未 V1 Apply:
          V2 Diff 会直接比对 Dev 实体的最新状态（包含未发布修改）
          V2 Publish 会直接将包含未发布修改的 Dev 实体写入 Prod
          V1 的 Change.enabled 标记不会阻止 V2 同步
```

---

## 十一、关键风险与注意事项

### 11.1 LRU 缓存导致的版本不一致

`RunJob.getWorkflow()` 使用了 `InMemoryLRUCacheStore.WORKFLOW`，缓存 key 为 `${environmentId}:${templateId}`。

- 若模板内容原地更新（同一 templateId），旧缓存会导致短时间内使用旧版本
- V2 Publish 操作应配合 `InvalidateCacheService` 清理缓存
- **渐进切换场景下必须禁用此缓存或加版本号后缀**

### 11.2 Job 快照与模板内容不一致

Job 创建时会将模板内容嵌入 `job.step.template`，后续 SendMessage 直接使用该快照。

- 已在队列中等待的 Job 不受模板更新影响
- 这保证了**单条通知的原子性**（不会出现步骤 1 用旧版本、步骤 2 用新版本）
- 但也意味着**无法对已入队通知紧急切换模板版本**

### 11.3 V1 _parentId 断裂风险（V1 特有）

V1 Promote 流程依赖 `_parentId` 做 Dev/Prod 消息模板 ID 映射。若 Dev 环境中新增了步骤但未先 Promote 消息模板子变更，`missingMessages` 日志会记录缺失，该步骤会被静默过滤。

- 关键代码: [promote-notification-template-change.usecase.ts L80-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts#L80-L132)
- 风险: Prod 环境模板步骤数 < Dev 环境，且不会报错中断
- **V2 无此风险**，因 V2 UpsertWorkflowUseCase 按 stepId 匹配并重新创建/更新步骤

### 11.4 payloadSchema 版本漂移

若 Dev 环境更新了 `payloadSchema` 但未发布（V1 Change 未 enabled / V2 未 Publish），Prod 环境仍用旧 Schema 校验。

- Trigger API 在哪个环境调用，就用哪个环境的模板 Schema
- 这是**环境隔离的设计初衷**，但需要 CI/CD 流程确保 Schema 变更与业务调用方升级同步

### 11.5 V1 ApplyChange 部分成功风险

父变更 + 子变更串行执行，若子变更 A 成功后子变更 B 失败：
- A 的 Change.enabled 保持 true，B 被自动置回 false
- 已 Promote 到目标环境的 A 变更**不会被自动回滚**
- 需人工判断是否需要重新发布或做补偿操作

### 11.6 V1 / V2 混用一致性风险（最关键）

```
场景: 某工作流在 Dev 环境被修改
  ├─► 产生 V1 Change (enabled=false)
  ├─► 用户未调用 V1 Apply（Change 仍是未发布状态）
  ├─► 用户在 Dashboard 点击 V2 Publish
  │     └─► V2 直接比对 Dev 实体最新状态 vs Prod 实体
  │           会把 Dev 上的修改（包括未 V1 发布的修改）全部同步到 Prod
  │
  └─► 结果:
        ✅ Prod 环境已更新为最新
        ❌ V1 Change.enabled 仍为 false（V2 不修改 Change 集合）
        ❌ V1 Changes 列表页仍显示"待发布变更"，但内容实际上已在 Prod 生效
```

**建议**：对于 BRIDGE / NOVU_CLOUD 工作流，统一使用 V2 Environments API 进行发布管理，避免 V1/V2 混用导致的状态不一致。

### 11.7 V2 Publish 无全局事务

- V2 Publish 内部各 Strategy 串行执行，每个资源的同步独立
- 若 Workflow A 同步成功、Workflow B 同步失败：
  - Workflow A 的 Prod 变更**不会被回滚**
  - Publish API 会抛出异常，调用方需自行处理部分成功场景
- SyncToEnvironmentUseCase 虽有 `session` 参数（MongoDB ClientSession），但未在整个 Publish 链路中绑定事务
