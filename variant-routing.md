# Variant 个性化路由匹配机制

本文档梳理 Novu 中 Variant 个性化路由匹配的完整运转流程，重点覆盖条件判断、受众选择与消息内容分支三者的协作方式。

---

## 1. 数据模型：Variant 在 Schema 中的位置

### 1.1 Step 与 Variant 共享同一结构

在 [notification-template.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/dal/src/repositories/notification-template/notification-template.schema.ts#L9-L103) 中，`variantSchemePart` 定义了 Step 和 Variant 共用的字段结构：

```
steps: [
  {
    ...variantSchemePart,          // Step 本身的字段
    variants: [variantSchemePart], // Step 下挂载的 Variant 列表
  },
]
```

每个 `variantSchemePart` 包含以下关键字段：

| 字段 | 作用 |
|---|---|
| `active` | 是否启用 |
| `filters` | 条件过滤器数组（决定路由匹配） |
| `_templateId` | 指向 `MessageTemplate` 的 ObjectId（决定消息内容） |
| `_parentId` | 父级 Step 的 ObjectId |
| `shouldStopOnFail` | 失败时是否中断 |
| `metadata` | Digest/Delay 等步骤元数据 |
| `name` / `uuid` / `stepId` | 标识信息 |

### 1.2 TypeScript 接口

在 [notification-template.interface.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/packages/shared/src/entities/notification-template/notification-template.interface.ts#L57-L92) 中：

```typescript
interface IStepVariant {
  _id?: string;
  uuid?: string;
  stepId?: string;
  name?: string;
  filters?: IMessageFilter[];
  _templateId?: string;
  template?: IMessageTemplate;
  active?: boolean;
  shouldStopOnFail?: boolean;
  // ...
}

interface INotificationTemplateStep extends IStepVariant {
  variants?: IStepVariant[];
}
```

### 1.3 Filter 结构

每个 `filters` 元素是一个 `IMessageFilter`（[notification-template.interface.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/packages/shared/src/entities/notification-template/notification-template.interface.ts#L94-L99)）：

```typescript
interface IMessageFilter {
  isNegated?: boolean;
  type?: BuilderFieldType;       // 如 'GROUP'
  value: BuilderGroupValues;     // 'AND' | 'OR'，表示组内逻辑
  children: FilterParts[];       // 具体条件列表
}
```

`FilterParts` 是联合类型（[builder.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/packages/shared/src/types/builder.ts#L121-L127)），包含：

| 类型 | `on` 字段值 | 用途 |
|---|---|---|
| `IFieldFilterPart` | `subscriber` / `payload` | 订阅者属性或触发载荷字段匹配 |
| `IWebhookFilterPart` | `webhook` | 外部 Webhook 返回结果判定 |
| `ITenantFilterPart` | `tenant` | 租户属性匹配 |
| `IRealtimeOnlineFilterPart` | `isOnline` | 用户是否在线 |
| `IOnlineInLastFilterPart` | `isOnlineInLast` | 用户最近在线时间 |
| `IPreviousStepFilterPart` | `previousStep` | 前置步骤的 read/seen 状态 |

每个字段条件包含 `field`、`value`、`operator`，支持 `EQUAL`、`NOT_EQUAL`、`LARGER`、`SMALLER`、`IN`、`NOT_IN`、`IS_DEFINED` 等运算符。

---

## 2. 整体流程：从 Trigger 到 Variant 路由

```
Trigger Event
  → CreateNotificationJobs (创建 Job)
    → Worker 消费 Job
      → SendMessage.execute()
        ① evaluateStepCondition()   — Step 级 filters 判定（该步骤是否执行）
        ② 分发到具体 Channel (Email/SMS/InApp/...)
          → SendMessageBase.processVariants()  — Variant 级 filters 判定
            → SelectVariant.execute()
              → NormalizeVariables    — 准备变量
              → ConditionsFilter      — 条件评估
              → MessageTemplate 读取  — 内容分支
          → 用选中的 MessageTemplate 替换 step.template
          → 编译 & 发送
```

### 2.1 Step 级条件 vs Variant 级条件

**Step 级条件**（Step 的 `filters`）：决定该步骤是否对当前订阅者执行。在 [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts#L215-L263) 中评估：

```typescript
private async evaluateStepCondition(command, variables) {
  const stepCondition = await this.conditionsFilter.filter(
    ConditionsFilterCommand.create({
      filters: command.job.step.filters || [],
      // ...
      variables,
    })
  );
  // stepCondition.passed === false → 跳过整个步骤
}
```

**Variant 级条件**（Variant 的 `filters`）：在 Step 级条件通过后，决定使用哪个 Variant 的消息模板。在 [select-variant.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts) 中评估。

两者使用同一个 `ConditionsFilter`，但作用域不同：Step filters 控制「是否执行」，Variant filters 控制「执行哪个内容变体」。

---

## 3. 核心路由逻辑：SelectVariant

### 3.1 入口与短路返回

[select-variant.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts#L20-L30)：

```typescript
async execute(command: SelectVariantCommand) {
  // 短路 1：Step 没有 variants → 直接返回 Step 默认模板
  if (!command.step.variants?.length) {
    return { messageTemplate: command.step.template };
  }

  // 短路 2：没有任何 filterData（无 tenant/payload/subscriber/webhook）→ 返回默认模板
  if (!this.isFilterDataExist(command.filterData)) {
    return { messageTemplate: command.step.template };
  }
  // ...
}
```

### 3.2 遍历匹配 Variant

[select-variant.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts#L32-L89)：

```typescript
for (const variant of command.step.variants) {
  // 跳过：无 filters 或未激活的 variant
  if (!variant.filters?.length || !variant.active) {
    continue;
  }

  // ① 变量归一化：确保 subscriber/tenant/payload 等数据就绪
  const variables = await this.normalizeVariablesUsecase.execute(/*...*/);

  // ② 条件过滤：评估 variant 的 filters
  const { passed, conditions } = await this.conditionsFilter.filter(/*...*/);

  // ③ 匹配成功：加载 variant 对应的消息模板
  if (passed) {
    const messageTemplate = await this.messageTemplateRepository.findOne({
      _id: variant._templateId,
    });
    return { messageTemplate, conditions };
  }
}

// 全部 variant 都未匹配 → 回退到 Step 默认模板
return { messageTemplate: command.step.template };
```

**关键设计**：Variant 列表是有序的，第一个匹配的 Variant 生效（first-match-wins）。

### 3.3 变量归一化：NormalizeVariables

[normalize-variables.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts) 在条件评估前准备所有必要数据：

```typescript
async execute(command) {
  // 合并 step 及其所有 variants 的 filters，统一判断需要哪些变量
  const combinedFilters = [command.step, ...(command.step?.variants || [])]
    .flatMap(variant => variant?.filters ?? []);

  // 按需从 DB 加载 subscriber（仅当存在 subscriber 类型 filter）
  filterVariables.subscriber = await this.fetchSubscriberIfMissing(command, combinedFilters);

  // 按需从 DB 加载 tenant（仅当存在 tenant 类型 filter）
  filterVariables.tenant = await this.fetchTenantIfMissing(command, combinedFilters);

  // payload 直接取自 job 或 command
  filterVariables.payload = command.variables?.payload ?? command.job?.payload;
}
```

归一化产生的 `IFilterVariables` 结构（[filter-processing-details.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/utils/filter-processing-details.ts#L5-L17)）：

```typescript
interface IFilterVariables {
  payload?: ITriggerPayload;
  subscriber?: SubscriberEntity;
  actor?: SubscriberEntity;
  webhook?: Record<string, unknown>;
  tenant?: TenantEntity;
  context?: ContextResolved;
  step?: { digest: boolean; events: any[]; total_count: number; };
}
```

---

## 4. 条件判断引擎：ConditionsFilter

### 4.1 顶层逻辑：OR 语义

[conditions-filter.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L63-L112)：

`filters` 数组中的每个元素是一个**条件组**（`IMessageFilter`），组与组之间是 **OR** 关系：

```typescript
const foundFilter = await this.findAsync(filters, async (filter) => {
  // 任意一个 filter 组通过即可
});
return { passed: !!foundFilter };
```

### 4.2 组内逻辑：AND / OR

[conditions-filter.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L378-L393)：

每个条件组的 `value` 字段决定组内子条件的逻辑关系：

- `FieldLogicalOperatorEnum.AND` → 所有子条件必须通过
- `FieldLogicalOperatorEnum.OR` → 任一子条件通过即可

```typescript
private async handleGroupFilters(filter, variables, command, details) {
  if (filter.value === FieldLogicalOperatorEnum.OR) {
    return await this.handleOrFilters(filter, variables, command, details);
  }
  if (filter.value === FieldLogicalOperatorEnum.AND) {
    return await this.handleAndFilters(filter, variables, command, details);
  }
}
```

### 4.3 AND 组内的优化：Webhook 延后

[conditions-filter.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L395-L423)：

AND 组中，将子条件分为 `webhookFilters` 和 `otherFilters`，先评估其他条件，若已失败则跳过 Webhook 调用（避免不必要的网络请求）：

```typescript
private async handleAndFilters(filter, variables, command, details) {
  const { webhookFilters, otherFilters } = this.splitFilters(filter);

  // 先评非 webhook 条件
  const matchedOtherFilters = await this.filterAsync(otherFilters, /*...*/);
  if (otherFilters.length !== matchedOtherFilters.length) {
    return false;  // 非 webhook 条件有失败的，整体 AND 不通过
  }

  // 再评 webhook 条件
  const matchedWebhookFilters = await this.filterAsync(webhookFilters, /*...*/);
  return matchedWebhookFilters.length === webhookFilters.length;
}
```

### 4.4 各类 Filter 的处理

[conditions-filter.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L343-L377) 的 `processFilter` 方法按 `on` 字段分派：

| `on` 值 | 处理方法 | 说明 |
|---|---|---|
| `payload` / `subscriber` / `tenant` | `processFilterEquality` | 从 `IFilterVariables` 中用 lodash `_.get(variables, 'on.field')` 取实际值，与 `value` 比较 |
| `webhook` | `getWebhookResponse` → `processFilterEquality` | 先 POST 请求外部 URL，用返回结果进行比较 |
| `isOnline` | `processIsOnline` | 检查 `subscriber.isOnline` 布尔值 |
| `isOnlineInLast` | `processIsOnline` | 计算 `lastOnlineAt` 与当前时间的差值 |
| `previousStep` | `processPreviousStep` | 查询前置步骤 Job 对应 Message 的 seen/read 状态 |

### 4.5 值比较：processFilterEquality

[filter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/utils/filter.ts#L7-L68) 中的核心比较方法：

```typescript
protected processFilterEquality(variables, fieldFilter, details) {
  // 从变量中取值：如 variables.tenant.name, variables.subscriber.email
  const actualValue = _.get(variables, `${fieldFilter.on}.${fieldFilter.field}`);
  const filterValue = this.parseValue(actualValue, fieldFilter.value);

  switch (fieldFilter.operator) {
    case FieldOperatorEnum.EQUAL:       result = actualValue === filterValue; break;
    case FieldOperatorEnum.NOT_EQUAL:   result = actualValue !== filterValue; break;
    case FieldOperatorEnum.LARGER:      result = actualValue > filterValue;   break;
    case FieldOperatorEnum.SMALLER:     result = actualValue < filterValue;   break;
    case FieldOperatorEnum.LARGER_EQUAL: result = actualValue >= filterValue; break;
    case FieldOperatorEnum.SMALLER_EQUAL: result = actualValue <= filterValue; break;
    case FieldOperatorEnum.IN:          result = actualValue.includes(filterValue); break;
    case FieldOperatorEnum.NOT_IN:      result = !actualValue.includes(filterValue); break;
    case FieldOperatorEnum.IS_DEFINED:  result = actualValue !== undefined;   break;
  }
}
```

`parseValue` 方法根据 `actualValue` 的类型将 `filterValue` 做类型转换，确保比较类型一致。

### 4.6 Handlebars 模板编译

在 `processFilter` 中，`tenant`/`payload`/`subscriber`/`webhook` 类型的 filter value 会先经过 Handlebars 编译：

```typescript
child.value = await this.compileFilter(child.value, variables, command.job);
```

这意味着 filter value 中可以使用模板语法，如 `{{subscriber.firstName}}`，在运行时被替换为实际值后再做比较。

---

## 5. 消息内容分支：从 Variant 到 MessageTemplate

### 5.1 processVariants 在各 Channel 中的调用

[send-message.base.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts#L158-L186)：

```typescript
protected async processVariants(command: SendMessageChannelCommand): Promise<MessageTemplateEntity> {
  const { messageTemplate, conditions } = await this.selectVariant.execute(
    SelectVariantCommand.create({
      step: command.step,
      job: command.job,
      filterData: command.compileContext ?? {},  // 包含 subscriber/payload/tenant 等
    })
  );

  if (conditions) {
    // 记录 Variant 被选中的执行详情
    await this.createExecutionDetails.execute({
      detail: DetailEnum.VARIANT_CHOSEN,
      raw: JSON.stringify({ conditions }),
    });
  }

  return messageTemplate;
}
```

所有 Channel 实现（Email、SMS、InApp、Chat、Push）都调用 `this.processVariants(command)`。

### 5.2 内容替换流程

以 Email 为例（[send-message-email.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L160-L170)）：

```typescript
const [template, overrideLayoutId] = await Promise.all([
  this.processVariants(command),       // ← Variant 路由
  this.getOverrideLayoutId(command, !!bridgeOutputs),
  this.sendSelectedIntegrationExecution(command.job, integration),
]);

if (template) {
  step.template = template;  // ← 用选中 Variant 的模板替换 Step 默认模板
}
```

之后所有编译（`compileEmailTemplate`）和发送操作都基于替换后的 `step.template`，包括：
- `step.template.content` → 邮件正文
- `step.template.subject` → 邮件主题
- `step.template.preheader` → 预览文本
- `step.template._layoutId` → 邮件布局

### 5.3 每个 Variant 拥有独立 MessageTemplate

Schema 中每个 Variant 通过 `_templateId` 关联一个独立的 `MessageTemplate` 文档（[notification-template.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/dal/src/repositories/notification-template/notification-template.schema.ts#L51-L54)），并通过 Mongoose Virtual 自动填充：

```typescript
notificationTemplateSchema.virtual('steps.variants.template', {
  ref: 'MessageTemplate',
  localField: 'steps.variants._templateId',
  foreignField: '_id',
  justOne: true,
});
```

---

## 6. 完整协作时序图

```
┌──────────┐    ┌──────────────────┐    ┌─────────────┐    ┌──────────────────┐
│  Trigger  │───▶│ CreateNotifJobs  │───▶│   Worker    │───▶│   SendMessage    │
└──────────┘    └──────────────────┘    └──────┬──────┘    └────────┬─────────┘
                                               │                     │
                                               │              ┌──────▼──────┐
                                               │              │ ① Step filters│
                                               │              │   条件判断    │
                                               │              │ (ConditionsFilter)│
                                               │              └──────┬──────┘
                                               │                     │ passed
                                               │              ┌──────▼──────┐
                                               │              │ ② 分发到 Channel │
                                               │              │ (Email/SMS/…) │
                                               │              └──────┬──────┘
                                               │                     │
                                               │              ┌──────▼──────┐
                                               │              │ ③ processVariants │
                                               │              │ (SelectVariant)    │
                                               │              └──────┬──────┘
                                               │                     │
                                               │         ┌───────────┼───────────┐
                                               │    ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
                                               │    │Variant 1│ │Variant 2│ │Fallback │
                                               │    │filters  │ │filters  │ │(default)│
                                               │    └────┬────┘ └────┬────┘ └────┬────┘
                                               │         │           │           │
                                               │    ┌────▼────┐ ┌────▼────┐     │
                                               │    │NormVars │ │NormVars │     │
                                               │    │CondFilter│ │CondFilter│    │
                                               │    └────┬────┘ └────┬────┘     │
                                               │     passed?     passed?         │
                                               │         │           │           │
                                               │    ┌────▼────┐      │           │
                                               │    │Load MT  │      │           │
                                               │    │by _templ│      │           │
                                               │    │ateId    │      │           │
                                               │    └────┬────┘      │           │
                                               │         │           │           │
                                               │         ▼           ▼           ▼
                                               │    ┌─────────────────────────────┐
                                               │    │ ④ 替换 step.template         │
                                               │    │ ⑤ 编译模板 + 发送消息         │
                                               │    └─────────────────────────────┘
```

---

## 7. 关键文件索引

| 文件 | 职责 |
|---|---|
| [notification-template.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/dal/src/repositories/notification-template/notification-template.schema.ts) | Variant 与 Step 的 MongoDB Schema 定义 |
| [notification-template.interface.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/packages/shared/src/entities/notification-template/notification-template.interface.ts) | IStepVariant / INotificationTemplateStep / IMessageFilter 接口 |
| [builder.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/packages/shared/src/types/builder.ts) | FilterPartTypeEnum / FieldOperatorEnum / FilterParts 等类型定义 |
| [select-variant.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts) | Variant 路由核心：遍历匹配、first-match-wins |
| [select-variant.command.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/select-variant/select-variant.command.ts) | SelectVariant 的入参定义（filterData / step / job） |
| [conditions-filter.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts) | 条件判断引擎：OR 组间 / AND-OR 组内 / 6 种 filter 类型 |
| [conditions-filter.command.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/conditions-filter/conditions-filter.command.ts) | ConditionsFilter 的入参定义 |
| [normalize-variables.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts) | 变量归一化：按需加载 subscriber / tenant |
| [filter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/utils/filter.ts) | processFilterEquality：值比较 + 运算符分派 |
| [filter-processing-details.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/utils/filter-processing-details.ts) | IFilterVariables 接口 + 条件评估详情记录 |
| [send-message.base.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts) | processVariants：连接路由结果与 Channel 发送 |
| [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts) | Step 级条件评估 + 分发到具体 Channel |
| [send-message-email.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts) | Email Channel 中 processVariants → 模板替换的完整示例 |
| [select-variant.spec.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/62-novu/libs/application-generic/src/usecases/select-variant/select-variant.spec.ts) | Variant 选择的单元测试（含完整测试数据） |
