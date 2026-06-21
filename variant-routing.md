# Variant 个性化路由匹配机制

本文档梳理 Novu 中 Variant 个性化路由匹配的完整运转流程，重点覆盖条件判断、受众选择与消息内容分支三者的协作方式，以及否定条件（`isNegated`）和引用型过滤（previousStep、webhook、isOnline 等）的实际实现。

---

## 1. 数据模型：Variant 在 Schema 中的位置

### 1.1 Step 与 Variant 共享同一结构

在 [notification-template.schema.ts](libs/dal/src/repositories/notification-template/notification-template.schema.ts#L9-L103) 中，`variantSchemePart` 定义了 Step 和 Variant 共用的字段结构：

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

在 [notification-template.interface.ts](packages/shared/src/entities/notification-template/notification-template.interface.ts#L57-L92) 中：

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

每个 `filters` 元素是一个 `IMessageFilter`（[notification-template.interface.ts](packages/shared/src/entities/notification-template/notification-template.interface.ts#L94-L99)）：

```typescript
interface IMessageFilter {
  isNegated?: boolean;          // 条件组否定（见第 7 节详细分析）
  type?: BuilderFieldType;      // 如 'GROUP'
  value: BuilderGroupValues;    // 'AND' | 'OR'，表示组内逻辑
  children: FilterParts[];      // 具体条件列表
}
```

`FilterParts` 是联合类型（[builder.ts](packages/shared/src/types/builder.ts#L121-L127)），包含：

| 类型 | `on` 字段值 | 用途 |
|---|---|---|
| `IFieldFilterPart` | `subscriber` / `payload` | 订阅者属性或触发载荷字段匹配 |
| `IWebhookFilterPart` | `webhook` | 外部 Webhook 返回结果判定（引用型） |
| `ITenantFilterPart` | `tenant` | 租户属性匹配 |
| `IRealtimeOnlineFilterPart` | `isOnline` | 用户是否在线（引用型） |
| `IOnlineInLastFilterPart` | `isOnlineInLast` | 用户最近在线时间（引用型） |
| `IPreviousStepFilterPart` | `previousStep` | 前置步骤的 read/seen 状态（引用型） |

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

**Step 级条件**（Step 的 `filters`）：决定该步骤是否对当前订阅者执行。在 [send-message.usecase.ts](apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts#L215-L263) 中评估：

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

**Variant 级条件**（Variant 的 `filters`）：在 Step 级条件通过后，决定使用哪个 Variant 的消息模板。在 [select-variant.usecase.ts](libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts) 中评估。

两者使用同一个 `ConditionsFilter`，但作用域不同：Step filters 控制「是否执行」，Variant filters 控制「执行哪个内容变体」。

---

## 3. 核心路由逻辑：SelectVariant

### 3.1 入口与短路返回

[select-variant.usecase.ts](libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts#L20-L30)：

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

[select-variant.usecase.ts](libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts#L32-L89)：

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

[normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts) 在条件评估前准备所有必要数据：

```typescript
async execute(command) {
  // 合并 step 及其所有 variants 的 filters，统一判断需要哪些变量
  const combinedFilters = [command.step, ...(command.step?.variants || [])]
    .flatMap(variant => variant?.filters ?? []);

  // 按需从 DB 加载 subscriber（仅当存在 subscriber / isOnline / isOnlineInLast / previousStep 类型 filter）
  filterVariables.subscriber = await this.fetchSubscriberIfMissing(command, combinedFilters);

  // 按需从 DB 加载 tenant（仅当存在 tenant 类型 filter）
  filterVariables.tenant = await this.fetchTenantIfMissing(command, combinedFilters);

  // payload 直接取自 job 或 command
  filterVariables.payload = command.variables?.payload ?? command.job?.payload;
}
```

归一化产生的 `IFilterVariables` 结构（[filter-processing-details.ts](libs/application-generic/src/utils/filter-processing-details.ts#L5-L17)）：

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

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L63-L112)：

`filters` 数组中的每个元素是一个**条件组**（`IMessageFilter`），组与组之间是 **OR** 关系：

```typescript
const foundFilter = await this.findAsync(filters, async (filter) => {
  // 任意一个 filter 组通过即可
});
return { passed: !!foundFilter };
```

### 4.2 组内逻辑：AND / OR

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L378-L393)：

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

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L395-L423)：

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

OR 组中采用同样的拆分策略：先评非 Webhook 条件，命中则立即返回；未命中再评 Webhook 条件。

### 4.4 各类 Filter 的处理

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L343-L377) 的 `processFilter` 方法按 `on` 字段分派：

| `on` 值 | 处理方法 | 说明 |
|---|---|---|
| `payload` / `subscriber` / `tenant` | `processFilterEquality` | 从 `IFilterVariables` 中用 lodash `_.get(variables, 'on.field')` 取实际值，与 `value` 比较 |
| `webhook` | `getWebhookResponse` → `processFilterEquality` | 先 POST 请求外部 URL，用返回结果进行比较 |
| `isOnline` | `processIsOnline` | 检查 `subscriber.isOnline` 布尔值 |
| `isOnlineInLast` | `processIsOnline` | 计算 `lastOnlineAt` 与当前时间的差值 |
| `previousStep` | `processPreviousStep` | 查询前置步骤 Job 对应 Message 的 seen/read 状态 |

### 4.5 值比较：processFilterEquality

[filter.ts](libs/application-generic/src/utils/filter.ts#L7-L68) 中的核心比较方法：

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

[send-message.base.ts](apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts#L158-L186)：

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

以 Email 为例（[send-message-email.usecase.ts](apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L160-L170)）：

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

Schema 中每个 Variant 通过 `_templateId` 关联一个独立的 `MessageTemplate` 文档（[notification-template.schema.ts](libs/dal/src/repositories/notification-template/notification-template.schema.ts#L51-L54)），并通过 Mongoose Virtual 自动填充：

```typescript
notificationTemplateSchema.virtual('steps.variants.template', {
  ref: 'MessageTemplate',
  localField: 'steps.variants._templateId',
  foreignField: '_id',
  justOne: true,
});
```

---

## 6. 引用型过滤（Reference-based Filtering）详解

引用型过滤指那些不直接比较当前上下文中的变量，而是需要额外查询（DB / HTTP / 其他 Step）来获取引用数据后再判定的过滤类型。包括：`previousStep`、`webhook`、`isOnline`、`isOnlineInLast`。

### 6.1 previousStep：引用前置步骤的消息状态

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L135-L185)

```typescript
private async processPreviousStep(filter, command, details) {
  // 1. 通过 step.uuid 定位同一 workflow 中对应前置步骤的 Job
  const job = await this.jobRepository.findOne({
    transactionId: command.job.transactionId,  // 同一触发事务
    _subscriberId: command.job._subscriberId,   // 同一订阅者
    _environmentId: command.environmentId,
    _organizationId: command.organizationId,
    'step.uuid': filter.step,                   // 被引用 Step 的 uuid
  });

  if (!job) return true;   // 前置 Job 不存在 → 默认通过

  // 2. 根据 Job 查找对应的 Message（记录了 seen/read 状态）
  const message = await this.messageRepository.findOne({
    _jobId: job._id,
    _environmentId: command.environmentId,
    _subscriberId: command.job._subscriberId,
    transactionId: command.job.transactionId,
  });

  if (!message) return true;  // Message 不存在 → 默认通过

  // 3. 根据 stepType 判断需要检查的字段
  const value = [PreviousStepTypeEnum.SEEN, PreviousStepTypeEnum.UNSEEN]
    .includes(filter.stepType) ? message.seen : message.read;

  // 4. UNREAD / UNSEEN 语义取反：期望的是「未读/未见」，所以 value === false 才算通过
  const passed = [PreviousStepTypeEnum.UNREAD, PreviousStepTypeEnum.UNSEEN]
    .includes(filter.stepType) ? value === false : value;

  return passed;
}
```

关键引用字段：
- `filter.step`：被引用 Step 的 `uuid`（即 [builder.ts](packages/shared/src/types/builder.ts#L92-L98) 中 `IPreviousStepFilterPart.step`）
- `filter.stepType`：枚举 `read / unread / seen / unseen`（[builder.ts](packages/shared/src/types/builder.ts#L70-L75) `PreviousStepTypeEnum`）

注意：当被引用 Step 的 Job 或 Message 不存在时，`processPreviousStep` 返回 `true`（默认通过），这是一种容错策略。

### 6.2 isOnline / isOnlineInLast：引用订阅者在线状态

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L187-L244)

```typescript
private async processIsOnline(filter, command, details) {
  // 1. 通过 subscriberId 查询 subscriber（带缓存 @CachedResponse）
  const subscriber = await this.getSubscriberBySubscriberId({
    subscriberId: command.job.subscriberId,
    _environmentId: command.environmentId,
  });

  // 2. 老订阅者（isOnline 和 lastOnlineAt 都未设置）→ 直接不通过
  const hasNoOnlineFieldsSet =
    typeof subscriber?.isOnline === 'undefined' && typeof subscriber?.lastOnlineAt === 'undefined';
  if (hasNoOnlineFieldsSet) return false;

  // 3. isOnline：精确匹配布尔值
  if (filter.on === FilterPartTypeEnum.IS_ONLINE) {
    return subscriber?.isOnline === filter.value;
  }

  // 4. isOnlineInLast：当前时间 - lastOnlineAt <= N 分钟/小时/天
  const diff = differenceIn(currentDate, parseISO(subscriber.lastOnlineAt), filter.timeOperator);
  // 若当前在线（isOnline=true）也视为通过
  const result = subscriber?.isOnline || (!subscriber?.isOnline && diff >= 0 && diff <= filter.value);
  return result;
}
```

引用字段：
- `subscriber.isOnline`：布尔值，由 Novu 实时服务（WebSocket）维护
- `subscriber.lastOnlineAt`：ISO 字符串，记录最后一次在线时间
- `filter.timeOperator`：`minutes` / `hours` / `days`（[builder.ts](packages/shared/src/types/builder.ts#L54-L58) `TimeOperatorEnum`）
- `filter.value`：数值阈值

### 6.3 webhook：引用外部服务返回结果

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L246-L341)

```typescript
private async getWebhookResponse(child, variables, command) {
  // 1. SSRF 防护：校验 URL 安全性
  assertSafeOutboundUrl(child.webhookUrl);

  // 2. 构造请求载荷：包含 subscriber / payload / identifier / channel / providerId
  const payload = await this.buildPayload(variables, command);

  // 3. 构造 HMAC 签名（用 environment 的 apiKey 加密 environmentId），
  //    作为 nv-hmac-256 请求头，供接收方校验来源合法性
  const hmac = await this.buildHmac(command);

  // 4. POST 请求外部 URL
  const response = await safeOutboundJsonRequest({
    url: child.webhookUrl,
    method: 'POST',
    headers: hmac ? { 'nv-hmac-256': hmac } : {},
    body: payload,
  });

  return response.body;
}
```

外部 Webhook 的返回值随后被放入 `variables.webhook`，由 `processFilterEquality` 按 `field` 取值（如 `webhook.allowed`）与 `value` 比较。

引用字段：
- `child.webhookUrl`：外部服务地址
- HMAC 算法：`createHash(decryptApiKey(environment.apiKeys[0].key), environmentId)`（[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L290-L307)）

### 6.4 引用型过滤在 NormalizeVariables 中的按需加载

[normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts#L45-L93)

```typescript
// 仅当 filters 中存在 subscriber / isOnline / isOnlineInLast / previousStep 等
// 依赖 subscriber 的 filter 类型时，才从 DB 加载 subscriber
const subscriberFilterExist = filters?.find(filter => {
  return filter?.children?.find(item =>
    item?.on === FilterPartTypeEnum.SUBSCRIBER
    || item?.on === FilterPartTypeEnum.IS_ONLINE
    || item?.on === FilterPartTypeEnum.IS_ONLINE_IN_LAST
    || item?.on === FilterPartTypeEnum.PREVIOUS_STEP
  );
});
```

注意：**previousStep 虽然不直接使用 subscriber 对象做属性比较，但为了确保 subscriberId → subscriber._id 的映射链完整，它也被归入了 subscriber 相关 filter 的触发条件**。实际 previousStep 的比较是通过 `jobRepository` + `messageRepository` 的单独查询完成的。

---

## 7. 否定条件（`isNegated`）分析：已定义但未实现

### 7.1 `isNegated` 在数据层中的定义

`isNegated` 字段出现在以下位置（均为 schema / interface 层的定义）：

| 位置 | 文件 | 行号 |
|---|---|---|
| MongoDB Schema | [notification-template.schema.ts](libs/dal/src/repositories/notification-template/notification-template.schema.ts) | L32 |
| DAL Entity | [libs/dal/.../notification-template.entity.ts](libs/dal/src/repositories/notification-template/notification-template.entity.ts) | L177 |
| Shared Interface | [notification-template.interface.ts](packages/shared/src/entities/notification-template/notification-template.interface.ts) | L95 |
| Application VO | [message.filter.ts](libs/application-generic/src/value-objects/message.filter.ts) | L5 |
| API DTO | [step-filter-dto.ts](libs/application-generic/src/dtos/step-filter-dto.ts) | L118 |
| Deprecated Workflow DTO | [workflow-deprecated.dto.ts](packages/shared/src/dto/workflows/workflow-deprecated.dto.ts) | L16 |

### 7.2 `isNegated` 在运行时代码中的实际消费情况

对以下核心执行路径的代码进行了全量 grep：

- `libs/application-generic/src/usecases/conditions-filter/` — 无任何 `isNegated` 读取
- `libs/application-generic/src/utils/filter.ts` — 无任何 `isNegated` 读取
- `libs/application-generic/src/usecases/select-variant/` — 测试数据中写入 `isNegated: false`，但未被代码消费
- `apps/worker/src/app/workflow/` — 测试数据中写入 `isNegated: false`，但未被代码消费
- `apps/dashboard/` — 条件编辑器基于 `react-querybuilder`，完全未引用 `isNegated`
- `apps/api/` — E2E 测试中写入 `isNegated: false`，仅作为 fixture 数据

结论：**`isNegated` 在所有评估路径中均未被读取，是一个定义了但从未实现的字段**。

### 7.3 替代方案：通过 `NOT_EQUAL` / `NOT_IN` 运算符实现否定

尽管 `isNegated` 未实现，用户仍可通过以下方式表达否定语义：

```typescript
// 否定方式 1：使用 NOT_EQUAL 运算符
{ field: 'plan', value: 'premium', operator: FieldOperatorEnum.NOT_EQUAL, on: 'tenant' }

// 否定方式 2：使用 NOT_IN 运算符
{ field: 'tags', value: 'blocked', operator: FieldOperatorEnum.NOT_IN, on: 'subscriber' }

// 否定方式 3：previousStep 的 UNREAD / UNSEEN 语义
{ step: '<uuid>', stepType: PreviousStepTypeEnum.UNREAD, on: 'previousStep' }
```

### 7.4 若要实现 `isNegated` 的预期位置

如果将来要实现该功能，应在 `ConditionsFilter.filter()` 的 `findAsync` 回调中，组内逻辑返回后对 `filter.isNegated` 取反：

```typescript
// conditions-filter.usecase.ts filter() 方法中的 findAsync 回调
const result = /* handleGroupFilters(...) 或 processFilter(...) 的返回值 */;
// 此处应加上：
return filter.isNegated ? !result : result;
```

同时需要在 Dashboard 的 `ConditionsEditor`（[conditions-editor.tsx](apps/dashboard/src/components/conditions-editor/conditions-editor.tsx)）中增加对否定开关的 UI 支持。

---

## 8. 数据加载关系总览

Variant 路由中 6 种 filter 类型的数据获取方式各不相同。核心问题在于：**谁负责加载数据，数据从哪来，各加载路径之间是否共享缓存？**

### 8.1 六种 filter 类型的数据获取路径

| filter.on | 触发 NormalizeVariables 预加载? | 实际数据获取方式 | 共享缓存? |
|---|---|---|---|
| `subscriber` | **是**（`fetchSubscriberIfMissing` 仅检查 `on === SUBSCRIBER`） | NormalizeVariables → `subscriberRepository.findOne` → 写入 `variables.subscriber` | — |
| `tenant` | **是**（`fetchTenantIfMissing` 仅检查 `on === TENANT`） | NormalizeVariables → `tenantRepository.findOne` → 写入 `variables.tenant` | — |
| `payload` | **否**（直接从 job 取） | `command.variables.payload ?? command.job.payload` → 写入 `variables.payload` | — |
| `isOnline` / `isOnlineInLast` | **否** | ConditionsFilter 自身的 `getSubscriberBySubscriberId`（L468-486）独立查询 | 与 NormalizeVariables 共享相同 Redis key（`buildSubscriberKey`），二次查询命中缓存 |
| `previousStep` | **否** | ConditionsFilter → `jobRepository.findOne` + `messageRepository.findOne`，**完全不依赖 subscriber 对象** | 无 |
| `webhook` | **否**（自身不查 subscriber） | ConditionsFilter → `safeOutboundJsonRequest` POST 外部 URL；`buildPayload` 中**间接复用** `variables.subscriber` | `buildPayload` 优先使用已预加载的 `variables.subscriber`，不存在则单独查询 |

### 8.2 关键差异：NormalizeVariables 不感知 isOnline / previousStep

[normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts#L53-L55) 的 `fetchSubscriberIfMissing` 仅检查 `item?.on === FilterPartTypeEnum.SUBSCRIBER`：

```typescript
const subscriberFilterExist = filters?.find((filter) => {
  return filter?.children?.find((item) => item?.on === FilterPartTypeEnum.SUBSCRIBER);
  // 注意：这里不检查 IS_ONLINE / IS_ONLINE_IN_LAST / PREVIOUS_STEP
});
```

这意味着：
- 如果一个 Step 的 filters 中**只有** `isOnline` 条件而没有 `subscriber` 条件，NormalizeVariables **不会**预加载 subscriber
- `processIsOnline` 必须自己查询 subscriber
- `processPreviousStep` 完全不使用 subscriber，直接查 Job + Message

### 8.3 两处 getSubscriberBySubscriberId 的关系

代码中存在两个同名方法，分别属于不同类，但共享 Redis 缓存：

| 位置 | 类 | 用途 |
|---|---|---|
| [normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts#L95-L113) | `NormalizeVariables` | 预加载 subscriber 到 `variables.subscriber` |
| [conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L468-L486) | `ConditionsFilter` | `processIsOnline` 内部独立查询 subscriber |

两者都使用 `@CachedResponse({ builder: buildSubscriberKey })` 装饰器，缓存 key 格式均为 `{entity:subscriber:e=<environmentId>:s=<subscriberId>}`。因此如果 NormalizeVariables 先执行了查询，ConditionsFilter 的第二次查询会命中 Redis 缓存而非 MongoDB。

### 8.4 webhook 对预加载结果的间接依赖

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L309-L341) 的 `buildPayload` 方法：

```typescript
private async buildPayload(variables, command) {
  if (variables.subscriber) {
    payload.subscriber = variables.subscriber;      // ← 复用预加载结果
  } else {
    payload.subscriber = await this.subscriberRepository.findBySubscriberId(
      command.environmentId,
      command.job.subscriberId
    );                                              // ← 单独查询，无缓存
  }
}
```

注意：这里的 fallback 查询调用的是 `subscriberRepository.findBySubscriberId`（无 `@CachedResponse`），与前面两处 `getSubscriberBySubscriberId`（有缓存）**不同**。如果 NormalizeVariables 未预加载 subscriber 且 webhook filter 存在，buildPayload 会直接查 MongoDB，绕过 Redis 缓存。

### 8.5 数据流向图

```
NormalizeVariables.execute()
  ├─ fetchSubscriberIfMissing()     ← 仅 on=SUBSCRIBER 触发
  │    └─ getSubscriberBySubscriberId() → @CachedResponse → subscriberRepository.findOne()
  │         └─ variables.subscriber = SubscriberEntity
  │
  ├─ fetchTenantIfMissing()         ← 仅 on=TENANT 触发
  │    └─ tenantRepository.findOne()
  │         └─ variables.tenant = TenantEntity
  │
  ├─ variables.payload = command.variables.payload ?? command.job.payload
  ├─ variables.step / actor / context = command.variables.*
  └─ 返回 IFilterVariables

ConditionsFilter.filter(variables)
  ├─ processFilter(on=SUBSCRIBER/TENANT/PAYLOAD)
  │    └─ processFilterEquality(variables, child)     ← 消费 variables.*
  │
  ├─ processFilter(on=WEBHOOK)
  │    ├─ buildPayload(variables)                     ← 优先用 variables.subscriber
  │    │    └─ fallback: subscriberRepository.findBySubscriberId()  ← 无缓存！
  │    └─ safeOutboundJsonRequest() → processFilterEquality({ webhook: res })
  │
  ├─ processFilter(on=IS_ONLINE / IS_ONLINE_IN_LAST)
  │    └─ processIsOnline()
  │         └─ this.getSubscriberBySubscriberId()     ← ConditionsFilter 自身方法，@CachedResponse
  │              └─ subscriber.isOnline / lastOnlineAt
  │
  └─ processFilter(on=PREVIOUS_STEP)
       └─ processPreviousStep()
            ├─ jobRepository.findOne({ transactionId, 'step.uuid' })
            └─ messageRepository.findOne({ _jobId })
                 └─ message.seen / read
```

---

## 9. 订阅者属性预加载（Subscriber Preload）数据来源

### 9.1 触发条件：按需加载

[normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts#L45-L67)

```typescript
private async fetchSubscriberIfMissing(command, filters) {
  // 优先使用已传入的 subscriber（上层调用方已预加载）
  if (command.variables?.subscriber) {
    return command.variables.subscriber;
  }

  // 仅当 filters 中存在 on=SUBSCRIBER 类型条件时才加载
  const subscriberFilterExist = filters?.find((filter) =>
    filter?.children?.find((item) => item?.on === FilterPartTypeEnum.SUBSCRIBER)
  );

  if (subscriberFilterExist && command.job) {
    return await this.getSubscriberBySubscriberId({
      subscriberId: command.job.subscriberId,
      _environmentId: command.environmentId,
    });
  }
  return undefined;
}
```

**注意**：`isOnline` / `isOnlineInLast` 虽然需要读取 subscriber 的 `isOnline` / `lastOnlineAt` 字段，但它们**不经过此处的预加载判断**——`processIsOnline` 会在 ConditionsFilter 内部独立查询 subscriber（见第 10.4 节）。`previousStep` 完全不依赖 subscriber 对象（见第 11 章）。

### 9.2 查询方法与缓存

[normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts#L95-L113)

```typescript
@CachedResponse({
  builder: (command) => buildSubscriberKey({
    _environmentId: command._environmentId,
    subscriberId: command.subscriberId,
  }),
})
public async getSubscriberBySubscriberId({ subscriberId, _environmentId }) {
  return await this.subscriberRepository.findOne({
    _environmentId,
    subscriberId,
  });
}
```

缓存 key 格式（[entities.ts](libs/application-generic/src/services/cache/key-builders/entities.ts#L12-L25)）：

```
{entity:subscriber:e=<environmentId>:s=<subscriberId>}
```

由 `@CachedResponse` 装饰器（基于 Redis）提供缓存，避免重复 DB 查询。

### 9.3 底层存储：Subscriber Collection

[subscriber.schema.ts](libs/dal/src/repositories/subscriber/subscriber.schema.ts#L8-L36) 中的关键字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `subscriberId` | String | **外部业务 ID**，来自 Trigger 事件，查询条件之一 |
| `_environmentId` | ObjectId | 环境 ID，查询条件之一 |
| `_organizationId` | ObjectId | 组织 ID |
| `firstName` / `lastName` | String | 基本属性，可用于 filter 比较 |
| `email` / `phone` | String | 渠道地址 |
| `locale` / `timezone` | String | 本地化信息 |
| `data` | Mixed | 自定义属性对象，filter 中通过 `subscriber.data.xxx` 访问 |
| `channels` | Mixed[] | 渠道凭证（token/endpoint 等） |
| `isOnline` | Boolean | 在线状态（见第 10 节） |
| `lastOnlineAt` | Date | 最后在线时间（见第 10 节） |

**唯一索引**（[subscriber.schema.ts](libs/dal/src/repositories/subscriber/subscriber.schema.ts#L179-L182)）：

```javascript
{ subscriberId: 1, _environmentId: 1 }
// unique: true, partialFilterExpression: { deleted: false }
```

这保证同一环境下 subscriberId 唯一，`findOne({ _environmentId, subscriberId })` 能命中索引高效查询。

### 9.4 可用于 Filter 的字段层级

`processFilterEquality` 使用 `_.get(variables, `${on}.${field}`)` 取值（[filter.ts](libs/application-generic/src/utils/filter.ts#L7-L18)），因此 subscriber 相关 filter 的取值路径为：

| filter.on | filter.field 示例 | lodash 取值路径 | 对应 Subscriber Entity 字段 |
|---|---|---|---|
| `subscriber` | `firstName` | `variables.subscriber.firstName` | `subscriberEntity.firstName` |
| `subscriber` | `email` | `variables.subscriber.email` | `subscriberEntity.email` |
| `subscriber` | `data.vipLevel` | `variables.subscriber.data.vipLevel` | `subscriberEntity.data.vipLevel`（自定义数据） |
| `subscriber` | `locale` | `variables.subscriber.locale` | `subscriberEntity.locale` |

---

## 10. 在线状态判断（isOnline / isOnlineInLast）数据来源

### 10.1 数据存储位置

在线状态同样存储在 **Subscriber Collection** 中（[subscriber.schema.ts](libs/dal/src/repositories/subscriber/subscriber.schema.ts#L26-L31)）：

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `isOnline` | Boolean | `false` | 当前是否在线 |
| `lastOnlineAt` | Date | （无默认） | 最后一次在线的 ISO 时间戳 |

### 10.2 写入路径一：WebSocket 实时连接

这是在线状态的主要写入源，由 `apps/ws` 服务负责。

#### 连接建立

[ws.gateway.ts](apps/ws/src/socket/ws.gateway.ts#L134-L168)

```typescript
private async processConnectionRequest(connection: Socket) {
  const token = this.extractToken(connection);
  const subscriber = await this.getSubscriber(token); // JWT 校验，aud === 'widget_user'

  await connection.join(subscriber._id);   // 加入以 subscriber._id 命名的 Socket.IO 房间
  await this.subscriberOnlineService.handleConnection(subscriber);
}
```

[subscriber-online.service.ts](apps/ws/src/shared/subscriber-online/subscriber-online.service.ts#L14-L18)

```typescript
async handleConnection(subscriber: ISubscriberJwt) {
  await this.subscriberRepository.update(
    { _id: subscriber._id, _environmentId: subscriber.environmentId },
    { $set: { isOnline: true } }
  );
}
```

#### 连接断开

[ws.gateway.ts](apps/ws/src/socket/ws.gateway.ts#L99-L126)

```typescript
private async handlerSubscriberDisconnection(connection: Socket) {
  const subscriber = await this.getSubscriber(token);
  // 检查该 subscriber 当前还有多少个活跃 Socket.IO 连接
  const activeConnections = await this.getActiveConnections(connection, subscriber._id);

  await this.subscriberOnlineService.handleDisconnection(subscriber, activeConnections);
}
```

[subscriber-online.service.ts](apps/ws/src/shared/subscriber-online/subscriber-online.service.ts#L20-L38)

```typescript
async handleDisconnection(subscriber, activeConnections) {
  let isOnline = false;
  const lastOnlineAt = new Date().toISOString();

  // 多设备/多标签页场景：仍有活跃连接则保持在线
  if (activeConnections > 1) {
    isOnline = true;
  }

  await this.subscriberRepository.update(
    { _id: subscriber._id, _environmentId: subscriber.environmentId },
    { $set: { isOnline, lastOnlineAt } }
  );
}
```

关键细节：断开时并非直接置为离线，而是先通过 `server.in(subscriber._id).fetchSockets()` 统计该 subscriber 仍有多少活跃连接——多于 1 个则保持 `isOnline = true`。

### 10.3 写入路径二：REST API（手动标记）

- **公共 API**：[update-subscriber-online-flag.usecase.ts](apps/api/src/app/subscribers/usecases/update-subscriber-online-flag/update-subscriber-online-flag.usecase.ts#L15-L37)

  ```typescript
  private getUpdatedFields(isOnline: boolean) {
    return {
      isOnline,
      ...(!isOnline && { lastOnlineAt: new Date().toISOString() }),
    };
  }
  ```

  仅在 `isOnline = false` 时更新 `lastOnlineAt`。

- **内部 API**：[update-subscriber-online-state.usecase.ts](apps/api/src/app/internal/usecases/update-subscriber-online-state/update-subscriber-online-state.usecase.ts#L15-L43)

  无论 `isOnline` 是 true 还是 false，都会更新 `lastOnlineAt`。

### 10.4 读取路径：processIsOnline

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L187-L244)

```typescript
private async processIsOnline(filter, command, details) {
  // ConditionsFilter 自身的 getSubscriberBySubscriberId（L468-486）
  // 不使用 variables.subscriber，而是独立查询
  const subscriber = await this.getSubscriberBySubscriberId({
    subscriberId: command.job.subscriberId,
    _environmentId: command.environmentId,
  });

  // 老订阅者（在线功能上线前创建）：两个字段都为 undefined → 判定为不通过
  if (typeof subscriber?.isOnline === 'undefined' && typeof subscriber?.lastOnlineAt === 'undefined') {
    return false;
  }

  // IS_ONLINE：精确匹配布尔值
  if (filter.on === FilterPartTypeEnum.IS_ONLINE) {
    return subscriber?.isOnline === filter.value;
  }

  // IS_ONLINE_IN_LAST：当前在线 或 lastOnlineAt 距今 <= N 时间单位
  const diff = differenceIn(new Date(), parseISO(subscriber.lastOnlineAt), filter.timeOperator);
  return subscriber?.isOnline || (!subscriber?.isOnline && diff >= 0 && diff <= filter.value);
}
```

**关键细节**：`processIsOnline` 调用的是 ConditionsFilter **自身的** `getSubscriberBySubscriberId`（L468-486），而非 NormalizeVariables 中的同名方法。两者是完全独立的方法，但都使用 `@CachedResponse({ builder: buildSubscriberKey })` 装饰器，共享同一个 Redis 缓存 key（`{entity:subscriber:e=<env>:s=<subscriberId>}`）。因此：

- 如果同一次评估中**同时存在** `subscriber` filter 和 `isOnline` filter，subscriber 会被查询两次：第一次由 NormalizeVariables 预加载（写 Redis），第二次由 ConditionsFilter 独立查询（命中 Redis 缓存）
- 如果**只有** `isOnline` filter 没有 `subscriber` filter，NormalizeVariables 不会预加载，ConditionsFilter 直接查 MongoDB（但结果会写入 Redis 缓存）

### 10.5 数据一致性说明

- **潜在延迟**：WS 服务的在线状态更新依赖 Socket.IO 连接事件。当 WS 实例异常宕机时，`isShutdown` 标志为 true，断开事件会被跳过（[ws.gateway.ts](apps/ws/src/socket/ws.gateway.ts#L99-L105)），可能导致 `isOnline` 状态残留为 `true`。
- **无主动心跳检测**：目前代码中未发现定期扫描长时间未更新 `lastOnlineAt` 的订阅者并强制置为离线的任务。

---

## 11. 前置步骤状态查询（previousStep）数据来源

`previousStep` filter 通过查询同一次 Trigger 事务中前置 Step 的消息状态（seen/read）来做条件判断。涉及两张 MongoDB Collection：**Job** 和 **Message**。

### 11.1 数据源一：Job Collection

#### 存储内容

[job.entity.ts](libs/dal/src/repositories/job/job.entity.ts#L24-L61)

| 字段 | 作用（对 previousStep 查询而言） |
|---|---|
| `_id` | Job 的 ObjectId，用于关联 Message |
| `transactionId` | 同一次 Trigger 事务的唯一标识，**查询条件之一** |
| `_subscriberId` | subscriber 的内部 ObjectId，**查询条件之一** |
| `_environmentId` / `_organizationId` | 多租户隔离，**查询条件之一** |
| `step` | 完整的 `NotificationStepEntity`，内嵌 `step.uuid` 字段 — **这就是被引用的 Step 标识** |
| `status` | Job 状态 |
| `type` | Step 类型（EMAIL / SMS / IN_APP 等） |

#### 查询方式

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L140-L150)

```typescript
const job = await this.jobRepository.findOne({
  transactionId: command.job.transactionId,  // 同一事务
  _subscriberId: command.job._subscriberId,   // 同一订阅者
  _environmentId: command.environmentId,
  _organizationId: command.organizationId,
  'step.uuid': filter.step,                   // filter.step = 被引用 Step 的 uuid
});
```

这是一个「点号查询」（dot notation），在 MongoDB 中匹配嵌套文档字段 `step.uuid`。

**容错策略**：如果 job 不存在，直接返回 `true`（视为条件通过）。这意味着如果前置步骤尚未执行或根本不存在，不会阻断当前 Variant 的匹配。

### 11.2 数据源二：Message Collection

#### 存储内容

[message.entity.ts](libs/dal/src/repositories/message/message.entity.ts#L23-L99)

| 字段 | 作用 |
|---|---|
| `_id` | Message 的 ObjectId |
| `_jobId` | 关联的 Job ObjectId，**查询条件之一** |
| `_subscriberId` | subscriber 内部 ID，**查询条件之一** |
| `transactionId` | 同事务标识，**查询条件之一** |
| `_environmentId` / `_organizationId` | 多租户隔离 |
| `seen` | Boolean — 用户是否「看到」消息（In-App Feed 中出现在视口） |
| `read` | Boolean — 用户是否「阅读」消息（主动点击/打开） |
| `channel` | 渠道类型（EMAIL / IN_APP / SMS 等） |

#### 查询方式

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L152-L161)

```typescript
const message = await this.messageRepository.findOne({
  _jobId: job._id,
  _environmentId: command.environmentId,
  _subscriberId: command.job._subscriberId,
  transactionId: command.job.transactionId,
});
```

同样的容错策略：Message 不存在也返回 `true`。

#### seen/read 的写入路径

主要由 Widget / Inbox API 的「标记已读/已见」接口写入：

[mark-message-as.usecase.ts](apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts#L39-L117)

```typescript
async execute(command) {
  // 1. 失效计数缓存
  await this.invalidateCache.invalidateQuery({ key: buildMessageCountKey().invalidate(...) });

  // 2. 更新 Message 文档的 seen / read 字段（含 lastSeenDate / lastReadDate）
  await this.messageRepository.changeStatus(
    command.environmentId,
    subscriber._id,
    command.messageIds,
    command.mark   // { seen?: boolean, read?: boolean }
  );

  // 3. 通过 WebSocket 推送新的未读/未见计数到前端
  this.webSocketsQueueService.add({
    name: 'sendMessage',
    data: { event: WebSocketEventEnum.UNREAD, userId: subscriber._id, ... },
  });

  // 4. 触发 MESSAGE_SEEN / MESSAGE_READ Webhook
  await this.sendWebhookMessage.execute({ eventType: WebhookEventEnum.MESSAGE_SEEN, ... });
}
```

底层写入（[message.repository.ts](libs/dal/src/repositories/message/message.repository.ts#L724-L758)）：

```typescript
async changeStatus(environmentId, subscriberId, messageIds, mark) {
  const requestQuery = {};
  if (mark.seen != null)  { requestQuery.seen = mark.seen;  requestQuery.lastSeenDate = new Date(); }
  if (mark.read != null)  { requestQuery.read = mark.read;  requestQuery.lastReadDate = new Date(); }

  for (const chunk of this.chunkArray(messageIds)) {
    await this.update(
      { _environmentId, _subscriberId, _id: { $in: chunk.map(id => ObjectId(id)) } },
      { $set: requestQuery }
    );
  }
}
```

### 11.3 状态判定逻辑

[conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts#L163-L184)

```typescript
// stepType 为 SEEN / UNSEEN → 检查 message.seen
// stepType 为 READ / UNREAD → 检查 message.read
const value = [PreviousStepTypeEnum.SEEN, PreviousStepTypeEnum.UNSEEN]
  .includes(filter.stepType) ? message.seen : message.read;

// UNREAD / UNSEEN 语义自动取反：期望「未读」等价于 value === false
const passed = [PreviousStepTypeEnum.UNREAD, PreviousStepTypeEnum.UNSEEN]
  .includes(filter.stepType) ? value === false : value;
```

即：

| filter.stepType | 比较逻辑 |
|---|---|
| `read` | `message.read === true` |
| `unread` | `message.read === false` |
| `seen` | `message.seen === true` |
| `unseen` | `message.seen === false` |

### 11.4 完整数据流图

```
用户触发 Widget In-App「标记已读」
        │
        ▼
apps/api: MarkMessageAs.execute()
        │
        ├──► messageRepository.changeStatus()
        │       └──► MongoDB Message Collection
        │              └─► _id, seen, read, lastSeenDate, lastReadDate
        │
        ├──► invalidateCache(buildMessageCountKey)  ← Redis 计数缓存失效
        │
        └──► WebSocketsQueueService.add()
                └──► apps/ws: WSGateway.sendMessage()
                        └──► 前端实时刷新未读数
                                    │
                                    ▼
                    ┌────────────────────────────────────┐
Variant 路由时       │  Worker: processPreviousStep()     │
(previousStep filter)│    1. jobRepository.findOne(       │
                     │         transactionId,             │
                     │         _subscriberId,             │
                     │         'step.uuid': <uuid>)       │
                     │    2. messageRepository.findOne(   │
                     │         _jobId, _subscriberId,     │
                     │         transactionId)             │
                     │    3. 比较 seen / read             │
                     └────────────────────────────────────┘
```

---

## 12. 完整协作时序图

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

## 13. 关键文件索引

| 文件 | 职责 |
|---|---|
| [notification-template.schema.ts](libs/dal/src/repositories/notification-template/notification-template.schema.ts) | Variant 与 Step 的 MongoDB Schema 定义（含 `isNegated` 字段） |
| [notification-template.entity.ts](libs/dal/src/repositories/notification-template/notification-template.entity.ts) | DAL 层 StepFilter Entity |
| [notification-template.interface.ts](packages/shared/src/entities/notification-template/notification-template.interface.ts) | IStepVariant / INotificationTemplateStep / IMessageFilter 接口 |
| [builder.ts](packages/shared/src/types/builder.ts) | FilterPartTypeEnum / FieldOperatorEnum / FilterParts / PreviousStepTypeEnum / TimeOperatorEnum 等类型定义 |
| [select-variant.usecase.ts](libs/application-generic/src/usecases/select-variant/select-variant.usecase.ts) | Variant 路由核心：遍历匹配、first-match-wins |
| [select-variant.command.ts](libs/application-generic/src/usecases/select-variant/select-variant.command.ts) | SelectVariant 的入参定义（filterData / step / job） |
| [conditions-filter.usecase.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.usecase.ts) | 条件判断引擎：OR 组间 / AND-OR 组内 / 6 种 filter 类型（含 previousStep / isOnline / webhook 引用型过滤） |
| [conditions-filter.command.ts](libs/application-generic/src/usecases/conditions-filter/conditions-filter.command.ts) | ConditionsFilter 的入参定义 |
| [normalize-variables.usecase.ts](libs/application-generic/src/usecases/normalize-variables/normalize-variables.usecase.ts) | 变量归一化：按需加载 subscriber / tenant |
| [filter.ts](libs/application-generic/src/utils/filter.ts) | processFilterEquality：值比较 + 运算符分派 |
| [filter-processing-details.ts](libs/application-generic/src/utils/filter-processing-details.ts) | IFilterVariables 接口 + 条件评估详情记录 |
| [message.filter.ts](libs/application-generic/src/value-objects/message.filter.ts) | 应用层 MessageFilter VO（含 `isNegated` 字段声明） |
| [step-filter-dto.ts](libs/application-generic/src/dtos/step-filter-dto.ts) | HTTP API 层 StepFilterDto（含 `isNegated` 字段声明） |
| [send-message.base.ts](apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts) | processVariants：连接路由结果与 Channel 发送 |
| [send-message.usecase.ts](apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts) | Step 级条件评估 + 分发到具体 Channel |
| [send-message-email.usecase.ts](apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts) | Email Channel 中 processVariants → 模板替换的完整示例 |
| [subscriber.schema.ts](libs/dal/src/repositories/subscriber/subscriber.schema.ts) | Subscriber MongoDB Schema（含 `isOnline` / `lastOnlineAt` 字段 + 唯一索引定义） |
| [subscriber.repository.ts](libs/dal/src/repositories/subscriber/subscriber.repository.ts) | Subscriber DAL：findBySubscriberId / findByEmail / findByPhone / bulkCreateSubscribers |
| [job.entity.ts](libs/dal/src/repositories/job/job.entity.ts) | Job Entity：`transactionId` / `_subscriberId` / `step.uuid` 等 previousStep 查询所需字段 |
| [job.repository.ts](libs/dal/src/repositories/job/job.repository.ts) | Job DAL：storeJobs / updateStatus / findJobsToDigest |
| [message.entity.ts](libs/dal/src/repositories/message/message.entity.ts) | Message Entity：`_jobId` / `seen` / `read` / `transactionId` 等 previousStep 查询所需字段 |
| [message.repository.ts](libs/dal/src/repositories/message/message.repository.ts) | Message DAL：changeStatus（seen/read 更新） / updateMessagesStatusByIds |
| [ws.gateway.ts](apps/ws/src/socket/ws.gateway.ts) | WebSocket Gateway：连接建立/断开 → 调用 SubscriberOnlineService |
| [subscriber-online.service.ts](apps/ws/src/shared/subscriber-online/subscriber-online.service.ts) | WS 在线状态写入：handleConnection / handleDisconnection（多设备感知） |
| [update-subscriber-online-flag.usecase.ts](apps/api/src/app/subscribers/usecases/update-subscriber-online-flag/update-subscriber-online-flag.usecase.ts) | 公共 REST API：手动标记订阅者在线状态 |
| [update-subscriber-online-state.usecase.ts](apps/api/src/app/internal/usecases/update-subscriber-online-state/update-subscriber-online-state.usecase.ts) | 内部 REST API：标记订阅者在线状态（每次都写 lastOnlineAt） |
| [mark-message-as.usecase.ts](apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts) | Widget In-App：标记消息已读/已见（seen/read 写入 + WebSocket 推送 + Webhook） |
| [entities.ts](libs/application-generic/src/services/cache/key-builders/entities.ts) | 缓存 key 构建：buildSubscriberKey / buildDedupSubscriberKey |
| [select-variant.spec.ts](libs/application-generic/src/usecases/select-variant/select-variant.spec.ts) | Variant 选择的单元测试（含完整测试数据，`isNegated` 仅在 fixture 中出现） |
| [conditions-filter.usecase.spec.ts](apps/worker/src/app/workflow/specs/conditions-filter.usecase.spec.ts) | ConditionsFilter 的集成测试（`isNegated` 仅在 fixture 中出现） |
| [conditions-editor.tsx](apps/dashboard/src/components/conditions-editor/conditions-editor.tsx) | Dashboard 条件编辑器（基于 react-querybuilder，未处理 `isNegated`） |
| [edit-step-conditions-form.tsx](apps/dashboard/src/components/workflow-editor/steps/conditions/edit-step-conditions-form.tsx) | Step 条件配置表单（JsonLogic ↔ react-querybuilder RuleGroupType 互转） |
| [conditions.ts](apps/dashboard/src/utils/conditions.ts) | Dashboard 条件辅助：计数、命名空间提取、JsonLogic 自定义操作解析 |
