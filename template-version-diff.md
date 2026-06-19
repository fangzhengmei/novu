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

### 1.3 ChangeEntity —— 变更记录实体

定义位置: [change.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/libs/dal/src/repositories/change/change.entity.ts)

这是版本管理的**核心数据结构**，记录 Dev 环境中每次编辑产生的差异补丁:

| 字段 | 类型 | 作用 |
|------|------|------|
| `_entityId` | string | 被修改实体的 ID（即 Dev 环境中模板的 `_id`） |
| `type` | ChangeEntityTypeEnum | 变更实体类型：`NOTIFICATION_TEMPLATE`、`MESSAGE_TEMPLATE`、`LAYOUT`、`FEED`、`NOTIFICATION_GROUP`、`TRANSLATION` 等 |
| `change` | any | **递归 Diff 补丁**（recursive-diff 格式的 rdiffResult[]） |
| `enabled` | boolean | 是否已应用（Promote）到下一级环境 |
| `_parentId` | string? | 父级 Change ID（用于级联变更分组） |
| `_environmentId` | string | 变更所属的源环境 ID（通常是 Dev） |

---

## 二、Diff 机制详解

### 2.1 Diff 的生成

当在 Dev 环境中更新模板时，系统会对修改前后的模板对象做递归差异比对，生成 `rdiffResult[]` 格式的补丁，存入 `ChangeEntity.change` 字段。

### 2.2 Diff 的聚合与应用

核心逻辑位于 [promote-change-to-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-change-to-environment/promote-change-to-environment.usecase.ts#L40-L91)

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
1. **sanitizeDiff**: 过滤掉非数组、path 不合法的 diff 项，保证补丁格式安全
2. **applyDiff**: 来自 `recursive-diff` 库，将补丁顺序应用到空对象 `{}` 上，最终还原出"目标状态"的完整实体
3. 这意味着 **Change 记录是增量补丁，但 Promote 时是全量覆盖** —— 聚合后得到的是最终状态

### 2.3 Diff 与回滚

回滚本质上是 **反方向的 Promote** 或 **生成反向 Diff**:

- 方案 A: 对目标环境实体和源环境某个历史状态再做一次 diff，生成反向补丁
- 方案 B: 利用 Change 记录的 enabled=false 状态，重新聚合时跳过该变更

当前实现中，`ApplyChange` usecase（[apply-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts#L41-L83)）通过 `enabled: false` 标记变更为"已撤回"，这是最接近回滚语义的实现。

---

## 三、发送链路中的模板版本选择

完整发送链路: **Trigger API → ParseEventRequest → WorkflowQueue → RunJob → SendMessage → 各渠道 Provider**

### 3.1 Trigger 入口阶段

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

### 3.2 Job 执行阶段（RunJob）

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

### 3.3 消息发送阶段（SendMessage）

代码位置: [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts#L334-L340)

```typescript
// 偏好评估时可能再次获取模板
const workflow = command.workflow ??
  (await this.getWorkflow({ _id: job._templateId, environmentId: job._environmentId }));
```

此阶段主要用模板做 **偏好判断（Subscriber Preference）**，不直接读取内容。实际模板内容（subject、content 等）已在 Job 创建时嵌入 `job.step.template`。

### 3.4 链路总结

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

## 四、后台展示中的变更与版本

### 4.1 获取变更列表

代码位置: [get-changes.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/get-changes/get-changes.usecase.ts#L45-L91)

```typescript
async execute(command: GetChangesCommand): Promise<ChangesResponseDto> {
  // 分页获取 Change 列表（按 promoted 过滤）
  const { data: changeItems, totalCount } = await this.changeRepository.getList(
    command.organizationId, command.environmentId,
    command.promoted, command.page * command.limit, command.limit
  );

  // 对每条 Change 补充展示数据（模板名、消息类型等）
  const changes = await changeItems.reduce(async (prev, change) => {
    let item = {};
    if (change.type === ChangeEntityTypeEnum.MESSAGE_TEMPLATE) {
      item = await this.getTemplateDataForMessageTemplate(change._entityId, command.environmentId);
    }
    if (change.type === ChangeEntityTypeEnum.NOTIFICATION_TEMPLATE) {
      item = await this.getTemplateDataForNotificationTemplate(change._entityId, command.environmentId);
    }
    // ... 其他类型
    list.push({ ...change, ...item });
    return list;
  }, Promise.resolve([]));
}
```

展示层补充的元数据:
- `templateName`: 所属工作流名称
- `templateId`: 所属工作流 ID
- `messageType`: 若为消息模板变更，显示渠道类型（EMAIL/SMS 等）

### 4.2 变更应用（Promote）

代码位置: [apply-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts#L15-L39)

```typescript
async execute(command: ApplyChangeCommand): Promise<ChangeEntity[]> {
  const parentChange = await this.changeRepository.findOne({ _id: command.changeId, ... });

  // 连同子变更一起按时间升序 apply
  const changes = await this.changeRepository.find(
    { _environmentId: parentChange._environmentId, _parentId: parentChange._id },
    '', { sort: { createdAt: 1 } }
  );

  for (const change of [...changes, parentChange]) {
    await this.applyChange(change, command);
  }
}
```

### 4.3 通知模板 Promote 的特殊逻辑

代码位置: [promote-notification-template-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts#L60-L241)

关键点:
1. **MessageTemplate ID 映射**: Dev 环境步骤的 `_templateId` 需要替换成 Prod 环境中对应消息模板的 `_id`（通过 `_parentId` 关联查找）
2. **NotificationGroup 依赖**: 若所属通知组在 Prod 环境尚不存在，会先递归 Promote 通知组变更
3. **偏好同步**: Promote 后会同步更新 UserWorkflowPreferences 和 WorkflowResourcePreferences
4. **软删除处理**: 若源模板已删除且目标环境存在副本，会删除 Prod 环境副本（级联删除消息模板）

---

## 五、变量兼容机制

### 5.1 三层变量校验

| 层级 | 位置 | 机制 |
|------|------|------|
| L1 Trigger 入口 | `ParseEventRequest` | `payloadSchema` + AJV JSON Schema 校验 + 默认值填充 |
| L2 模板变量列表 | `MessageTemplateEntity.variables` | 定义模板所需变量，但仅用于编辑器提示 |
| L3 运行时渲染 | `SendMessage*` 各渠道 | Handlebars/Liquid 模板引擎渲染，缺失变量默认渲染为空字符串 |

### 5.2 payloadSchema 的设计意图

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

### 5.3 变量不兼容风险场景

1. **删除必填变量**: 旧 trigger 仍在传该变量不会出错，但模板渲染逻辑可能隐含依赖
2. **改变变量类型**: 如 `amount` 从 number 改为 string，Schema 校验会拦截旧 payload
3. **修改默认值**: 旧 trigger 不传该变量时行为会变化，需要在 Release Notes 中明确

---

## 六、影子发布、渐进切换与发送链路的关系

### 6.1 当前实现中的版本切换模型

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

### 6.2 若实现影子发布/渐进切换需要改造的链路节点

| 功能 | 需改造位置 | 改造思路 |
|------|-----------|---------|
| **影子发布（Shadow Release）** | `ParseEventRequest`、`SendMessage` | trigger 时同时向新旧两个版本各发一份，旧版本真实发送、新版本仅日志/埋点对比输出 |
| **渐进切换（Canary Rollout）** | `ParseEventRequest.getNotificationTemplateByTriggerIdentifier` | 按比例/租户/用户标签选择 templateId（旧版 or 新版），写入 Job |
| **版本绑定 Job** | 无需改动 | 当前 `job._templateId` 已天然具备 Job 级版本绑定语义 |
| **缓存失效** | `RunJob.getWorkflow` 的 LRU Cache | 渐进切换期间需关闭缓存或加版本号做 key，避免旧版本缓存命中 |
| **多版本存储** | `NotificationTemplateEntity` | 新增 `version` 或 `variant` 字段，同一 trigger identifier 下可存在多个版本 |
| **回滚原子性** | `PromoteChangeToEnvironment` | 当前是全量聚合 applyDiff，回滚需生成反向 Change 并重新 Promote |

### 6.3 渐进切换与发送链路的时序关系

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

## 七、各概念的关系矩阵

| 概念 | 核心机制 | 影响发送链路 | 影响后台展示 | 涉及主要文件 |
|------|---------|-------------|-------------|-------------|
| **Diff** | `recursive-diff` 的 rdiffResult 补丁，存于 `ChangeEntity.change` | 间接影响：聚合后成为新版本模板内容 | 直接：Dashboard Changes 列表展示每条变更 | [promote-change-to-environment.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-change-to-environment/promote-change-to-environment.usecase.ts) |
| **回滚** | 将 Change.enabled 置 false 或生成反向 Diff 重新 Promote | 影响后续新 trigger 使用的模板版本（已入队 Job 不受影响） | 展示为 Changes 列表中 enabled 状态切换 | [apply-change.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/apply-change/apply-change.usecase.ts) |
| **变量兼容** | `payloadSchema` JSON Schema + AJV 默认值填充 + 模板引擎兜底 | Trigger 入口校验失败直接拒绝；运行时渲染缺变量为空字符串 | Dashboard 编辑器提供 Schema 定义 UI | [parse-event-request.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts) |
| **影子发布** | 当前未原生实现，需双版本并行触发并对比 | 需在 SendMessage 层面增加"仅记录不真实发送"分支 | 需新增影子发布结果对比页面 | 需新建 |
| **渐进切换** | 当前未原生实现，需按流量比例路由 templateId | ParseEventRequest 中选择版本 → 写入 Job._templateId → 后续链路天然跟随 | 需新增版本流量配置 UI + 指标看板 | 需新建 |
| **发送链路** | Trigger → ParseEvent → Queue → RunJob → SendMessage → Provider | **版本选择的实际发生处**，是所有版本策略的落点 | 展示 Activity Feed / Execution Details 中的执行结果 | [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts), [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts) |

---

## 八、关键风险与注意事项

### 8.1 LRU 缓存导致的版本不一致

`RunJob.getWorkflow()` 使用了 `InMemoryLRUCacheStore.WORKFLOW`，缓存 key 为 `${environmentId}:${templateId}`。

- 若模板内容原地更新（同一 templateId），旧缓存会导致短时间内使用旧版本
- Promote 操作应配合 `InvalidateCacheService` 清理缓存
- **渐进切换场景下必须禁用此缓存或加版本号后缀**

### 8.2 Job 快照与模板内容不一致

Job 创建时会将模板内容嵌入 `job.step.template`，后续 SendMessage 直接使用该快照。

- 已在队列中等待的 Job 不受模板更新影响
- 这保证了**单条通知的原子性**（不会出现步骤 1 用旧版本、步骤 2 用新版本）
- 但也意味着**无法对已入队通知紧急切换模板版本**

### 8.3 跨环境 _parentId 断裂风险

Promote 流程依赖 `_parentId` 做 Dev/Prod 消息模板 ID 映射。若 Dev 环境中新增了步骤但未先 Promote 消息模板子变更，`missingMessages` 日志会记录缺失，该步骤会被静默过滤。

- 关键代码: [promote-notification-template-change.usecase.ts L80-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/56-novu/apps/api/src/app/change/usecases/promote-notification-template-change/promote-notification-template-change.usecase.ts#L80-L132)
- 风险: Prod 环境模板步骤数 < Dev 环境，且不会报错中断

### 8.4 payloadSchema 版本漂移

若 Dev 环境更新了 `payloadSchema` 但未 Promote，Prod 环境仍用旧 Schema 校验。

- Trigger API 在哪个环境调用，就用哪个环境的模板 Schema
- 这是**环境隔离的设计初衷**，但需要 CI/CD 流程确保 Schema 变更与业务调用方升级同步
