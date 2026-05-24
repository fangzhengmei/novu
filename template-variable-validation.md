# 模板与布局变量校验机制详解

本文档详细说明 Novu 系统中模板（Template）与布局（Layout）的变量声明、完整命名空间范围、校验时机、完整校验链路以及渲染期的兜底逻辑。

---

## 一、变量声明机制

### 1.1 变量数据结构 `ITemplateVariable`

**定义位置**：`packages/shared/src/types/message-templates.ts:23-28`

```typescript
export interface ITemplateVariable {
  type: TemplateVariableTypeEnum;  // String | Array | Boolean
  name: string;                    // 变量名，支持嵌套如 "user.name"
  required?: boolean;              // 是否必填，默认 false
  defaultValue?: string | boolean; // 默认值
}
```

### 1.2 变量使用场景

变量声明同时存在于模板和布局中：

| 场景 | 接口 | 字段 |
|------|------|------|
| 消息模板 | `IMessageTemplate` | `variables?: ITemplateVariable[]` |
| 布局 | `ILayoutEntity` / `LayoutDto` | `variables?: ITemplateVariable[]` |

### 1.3 Payload Schema 与 `validatePayload` 开关

工作流级别的 Payload Schema 校验是可选的，由两个字段共同控制：

| 字段 | 类型 | 说明 |
|------|------|------|
| `payloadSchema` | `JSONSchemaDto` | JSON Schema 定义，描述 payload 结构 |
| `validatePayload` | `boolean` | 是否启用 Schema 校验，默认 `false` |

**存储位置**：`INotificationTemplate` 接口（`notification-template.interface.ts:36`）
**数据库字段**：`notification-template.schema.ts:242`

---

## 二、完整命名空间范围说明

### 2.1 全部可用命名空间总览

系统支持以下 10 个命名空间，分为**模板通用**和**布局特有**两类：

| 命名空间 | 模板可用 | 布局可用 | 说明 |
|----------|---------|---------|------|
| `payload` | ✅ | ✅ | 触发时传入的自定义数据 |
| `subscriber` | ✅ | ✅ | 订阅者信息 |
| `context` | ✅ | ✅ | 上下文数据（tenant、actor 等） |
| `workflow` | ✅ | ❌ | 工作流元数据 |
| `steps` | ✅ | ❌ | 前置步骤执行结果 |
| `env` | ✅ | ✅ | 环境变量（用户自定义 + 系统内置） |
| `step` | ✅ | ❌ | 兼容旧版步骤变量（digest/events/total_count） |
| `branding` | ✅ | ✅ | 品牌信息 |
| `tenant` | ✅ | ✅ | 租户信息 |
| `actor` | ✅ | ✅ | 执行者信息 |
| `preheader` | ✅ | ❌ | 邮件预览文本 |
| `layout_content` | ❌ | ✅ | 布局内容占位符（布局特有） |

### 2.2 编辑期 vs 构建期变量范围

编辑期和构建期使用相同的 `variableSchema` 构建逻辑，但范围存在关键差异：

#### 模板编辑/构建期 Schema 构建

**核心代码**：`libs/application-generic/src/usecases/build-variable-schema/build-available-variable-schema.usecase.ts:113-128`

```typescript
return {
  type: JsonSchemaTypeEnum.OBJECT,
  properties: {
    workflow: buildWorkflowSchema(),              // 工作流元数据
    subscriber: buildSubscriberSchema(finalSubscriber), // 订阅者
    steps: buildPreviousStepsSchema({...}),       // 仅前置步骤结果
    payload: await this.resolvePayloadSchema(...),// payload 结构
    context: buildContextSchema(finalContext),    // 上下文
    env: buildEnvSchema(envVars),                 // 环境变量
  },
  additionalProperties: false,
};
```

#### 布局编辑/构建期 Schema 构建

**核心代码**：`libs/application-generic/src/usecases/layout-variables-schema/layout-variables-schema.usecase.ts:41-52`

```typescript
return {
  type: JsonSchemaTypeEnum.OBJECT,
  properties: {
    subscriber: buildSubscriberSchema(subscriber),
    [LAYOUT_CONTENT_VARIABLE]: {                  // 布局特有 content 变量
      type: JsonSchemaTypeEnum.STRING,
    },
    context: buildContextSchema(context),
    env: buildEnvSchema(envVars),
  },
  additionalProperties: false,
};
```

#### 关键范围差异

| 差异点 | 说明 |
|--------|------|
| **steps 范围** | 仅包含**当前步骤之前**执行的步骤，不包含后续步骤。例如步骤 3 只能访问 steps.step1 和 steps.step2，不能访问 steps.step3 或 steps.step4 |
| **workflow** | 仅模板可用，布局不可用 |
| **layout_content** | 仅布局可用，模板不可用 |
| **steps.digest** | Digest 步骤的 `events` 字段有特殊处理，支持 `steps.digest-step.events[0].payload` 格式 |

---

## 三、系统内置变量完整清单

### 3.1 `TemplateSystemVariables` 列表

**定义位置**：`packages/shared/src/entities/message-template/message-template.interface.ts:54`

```typescript
export const TemplateSystemVariables = ['subscriber', 'step', 'branding', 'tenant', 'preheader', 'actor'];
```

### 3.2 完整内置变量详细说明

#### 1. `workflow` - 工作流元数据

**Schema 定义**：`libs/application-generic/src/utils/create-schema.ts:94-112`

```typescript
{
  workflowId: string;      // 工作流标识符
  name: string;            // 工作流名称
  description: string;     // 工作流描述
  tags: string[];          // 标签数组
  severity: SeverityLevelEnum; // 严重级别 (critical/warning/info)
}
```

**必填字段**：`workflowId`, `name`

#### 2. `subscriber` - 订阅者信息

**Schema 定义**：`libs/application-generic/src/utils/create-schema.ts:63-92`

```typescript
{
  firstName: string;       // 名
  lastName: string;        // 姓
  email: string;           // 邮箱
  phone: string;           // 电话（可选）
  avatar: string;          // 头像 URL（可选）
  locale: string;          // 语言区域（可选）
  timezone: string;        // 时区（可选）
  subscriberId: string;    // 订阅者唯一标识
  isOnline: boolean;       // 是否在线（可选）
  lastOnlineAt: string;    // 最后在线时间（可选）
  data: Record<string, any>; // 自定义数据
}
```

**必填字段**：`subscriberId`

#### 3. `steps` - 前置步骤结果

**Schema 定义**：`build-available-variable-schema.usecase.ts:217-281`

```typescript
// 格式：steps.{stepId}.{property}
{
  [stepId]: {
    // 根据 stepType 动态生成
    // Digest 步骤特殊结构：
    events: Array<{
      id: string;
      time: string;         // ISO 时间
      payload: any;         // 事件 payload
    }>;
    eventCount: number;     // 事件数量
  }
}
```

**Digest 步骤特殊变量**：
- `steps.digest-step.events` - 聚合的事件数组
- `steps.digest-step.events[0].payload` - 第一个事件的 payload
- `steps.digest-step.eventCount` - 事件总数

**HTTP 请求步骤**：
- 根据 `responseBodySchema` 动态生成可访问字段

#### 4. `env` - 环境变量

**Schema 定义**：`libs/application-generic/src/utils/create-schema.ts:114-128`

包含两类变量：
1. **用户自定义环境变量** - 在环境设置中配置的键值对
2. **系统内置环境变量**：
   - `env.name` - 环境名称（如 "Production", "Development"）
   - `env.type` - 环境类型

#### 5. `context` - 上下文数据

**Schema 定义**：`libs/application-generic/src/utils/create-schema.ts:130-198`

```typescript
{
  [entityType]: {
    id: string;            // 上下文实体标识
    data: Record<string, any>; // 实体数据
  }
}
```

常见实体类型：
- `context.tenant` - 租户上下文
- `context.actor` - 执行者上下文

#### 6. `step` - 兼容旧版步骤变量

**SystemVariablesWithTypes 定义**：`message-template.interface.ts:75-79`

```typescript
{
  digest: boolean;         // 是否为 digest 聚合
  events: array;           // 事件数组
  total_count: number;     // 事件总数
}
```

> **注意**：这是旧版兼容变量，推荐使用 `steps.{stepId}` 格式访问特定步骤结果。

#### 7. `branding` - 品牌信息

**SystemVariablesWithTypes 定义**：`message-template.interface.ts:80-83`

```typescript
{
  logo: string;            // Logo URL
  color: string;           // 品牌主色
}
```

#### 8. `tenant` - 租户信息

**SystemVariablesWithTypes 定义**：`message-template.interface.ts:84-87`

```typescript
{
  name: string;            // 租户名称
  data: object;            // 租户自定义数据
}
```

#### 9. `actor` - 执行者信息

**SystemVariablesWithTypes 定义**：`message-template.interface.ts:66-74`

```typescript
{
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  avatar: string;
  locale: string;
  subscriberId: string;
}
```

#### 10. `preheader` - 邮件预览文本

**类型**：`string`

邮件收件人列表中显示的预览文本。

#### 11. `layout_content` - 布局内容占位符（布局特有）

**常量定义**：`packages/shared/src/consts/layouts.ts:1`

```typescript
export const LAYOUT_CONTENT_VARIABLE = 'content';
```

- **类型**：`string`
- **用途**：标记邮件正文在布局中的插入位置
- **校验要求**：布局内容必须包含此变量（HTML 或 Maily JSON 格式）
- **渲染时**：被替换为实际邮件模板内容

---

## 四、完整校验链路总览

变量校验分布在**编辑 → 保存 → 构建 → 触发 → 渲染**五个阶段，形成层层递进的防护链。其中**触发阶段又分为两层独立校验**。

```
用户编辑
   ↓
阶段 1：前端实时校验（Liquid 语法、命名空间、Schema 存在性）
   ↓
用户保存
   ↓
阶段 2：保存时校验（格式、必需变量、合法性）
   ↓
工作流构建
   ↓
阶段 3：构建时校验（Schema + Liquid + 自定义规则 → 生成 issues）
   ↓
API 触发 /events/trigger
   ↓
阶段 4.1：请求侧 Schema 校验（AJV + payloadSchema，可选）
   ↓
阶段 4.2：执行侧必填与默认值校验（ITemplateVariable，始终执行）
   ↓
入队 → Worker 执行
   ↓
阶段 5：渲染时兜底（undefined/null → 空字符串）
```

---

## 五、各阶段校验详细说明

### 阶段 1：前端编辑实时校验

**触发时机**：用户在 Dashboard 编辑模板/布局内容时

**核心代码**：
- `apps/dashboard/src/components/variable/hooks/use-variable-validation.ts`
- `libs/application-generic/src/utils/issues.ts:195-258` (`processControlValuesByLiquid`)

**校验内容**：
1. **命名空间校验**：变量必须以 `payload.` / `subscriber.` / `context.` / `workflow.` / `steps.` / `env.` 开头
   - 无命名空间的单段变量会提示 `invalid or missing namespace`
   - 例外：`layout_content`（布局特有，无需命名空间）
   - 建议补全：`Did you mean {{payload.xxx}}?`

2. **Schema 存在性校验**：`payload.` 开头的变量必须在 Payload Schema 中声明
   - 通过 `isPropertyAllowed()` 递归校验属性路径 `new-liquid-parser.ts:174-232`
   - 支持数组索引 `payload.items[0].name`
   - 支持 digest 事件特殊格式 `steps.digest-step.events[0].payload`

3. **过滤器有效性校验**：校验 Liquid 过滤器参数合法性
   - 如 `toSentence`、`digest`、`pluralize` 等过滤器的参数

4. **语法正确性校验**：
   - 变量不能包含空格（`contains whitespaces` 错误）
   - Liquid 语法正确性（通过 `parserEngine.parse()` 校验）
   - 布局特有：`{{ layout_content }}` 变量存在性

### 阶段 2：保存/更新时校验

**触发时机**：调用 API 创建或更新布局/模板时

**核心代码**：
- `apps/api/src/app/layouts-v2/usecases/build-layout-issues/build-layout-issues.usecase.ts`
- `apps/api/src/app/layouts-v2/usecases/upsert-layout/upsert-layout.usecase.ts:130-177`

**校验内容**：
1. **内容格式校验**：
   - HTML 模式：必须包含完整的 `<html>...</html><body>...</body>` 结构
   - Block 模式：必须是合法的 Maily JSON 格式

2. **必需变量校验**：布局内容必须包含 `{{ layout_content }}` 变量
   - 用于标记邮件正文插入位置
   - 校验同时支持 Maily JSON 和 HTML 两种格式

3. **Schema 校验**：基于 `layoutControlSchema` 校验控件值类型
4. **Liquid 变量校验**：同阶段 1 的变量合法性校验

**校验失败处理**：抛出 `BadRequestException`，阻止保存。

### 阶段 3：工作流构建时校验

**触发时机**：在工作流编辑界面，每次修改步骤内容时

**核心代码**：
- `libs/application-generic/src/usecases/build-step-issues/build-step-issues.usecase.ts`

**三重校验并行**：

```typescript
// 1. Schema 校验（控件值类型）
const schemaIssues = processControlValuesBySchema({
  controlSchema, controlValues, stepType,
});

// 2. Liquid 变量校验（变量合法性）
processControlValuesByLiquid({
  variableSchema, currentValue, currentPath, issues,
});

// 3. 自定义规则校验（如 tier 限制、skip 逻辑等）
const customIssues = await this.processControlValuesByCustomeRules(...);
const skipLogicIssues = this.validateSkipField(...);

// 合并所有问题
return merge(schemaIssues, liquidIssues, customIssues, skipLogicIssues);
```

**校验结果存储**：结果存入 `step.issues` 字段（`StepIssues` 类型）
- `controls?: Record<string, RuntimeIssue[]>` - 控件相关问题
- `integration?: Record<string, RuntimeIssue[]>` - 集成相关问题

**校验结果影响**：
- 以红色警告形式显示在编辑器中
- **不阻止保存**，仅影响前端显示的工作流状态
- 工作流状态计算：`compute-workflow-status.ts:4-15`

```typescript
export function computeWorkflowStatus(workflowActive: boolean, steps: NotificationStep[]) {
  if (!workflowActive) return WorkflowStatusEnum.INACTIVE;
  
  const hasIssues = steps.some((step) => hasControlIssues(step.issues));
  if (!hasIssues) return WorkflowStatusEnum.ACTIVE;
  
  return WorkflowStatusEnum.ERROR;
}
```

> **重要**：`issues` 仅影响前端显示状态（ERROR），**不阻止 API 触发**。只有 `active=false` 会真正阻止触发。

### 阶段 4：触发时校验（核心重点 - 两层校验）

**触发时机**：调用 `/events/trigger` API 发送消息时

**执行位置**：API 进程中**同步执行**，在入队之前完成，而非 Worker 执行期。

---

#### 阶段 4.1：请求侧 Schema 校验

**触发条件**：`template.validatePayload === true && template.payloadSchema !== null`

**核心代码**：
- `apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts:138-157`

**校验逻辑**：
```typescript
if (template.validatePayload && template.payloadSchema) {
  try {
    const validatedPayload = this.validateAndApplyPayloadDefaults(
      command.payload, 
      template.payloadSchema
    );
    command.payload = validatedPayload;
  } catch (error) {
    // 抛出 PayloadValidationException
    throw error;
  }
}
```

**实现细节**：
- 使用 **AJV**（Another JSON Schema Validator）进行校验
- 配置：`allErrors: true, useDefaults: true, strict: false`
- 自动应用 Schema 中定义的 `default` 值
- 校验失败抛出 `PayloadValidationException`，包含详细的 AJV 错误信息

**校验失败处理**：
- HTTP 400 错误
- 响应体包含 `validationErrors` 字段，列出所有不匹配项
- 请求 trace 记录 `request_payload_validation_failed` 事件

---

#### 阶段 4.2：执行侧必填与默认值校验

**触发条件**：始终执行，不受 `validatePayload` 开关控制

**核心代码**：
- `libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts:73-82`
- `libs/application-generic/src/services/verify-payload.service.ts`

**校验逻辑**：

```typescript
// 在 TriggerEvent.execute() 中调用
const defaultPayload = this.verifyPayload.execute(
  VerifyPayloadCommand.create({
    payload: command.payload,
    template: storedWorkflow,
  })
);

// 合并默认值（用户传入的 payload 优先级更高）
command.payload = toMerged(defaultPayload, command.payload);
```

**具体实现**（`verify-payload.service.ts`）：

```typescript
verifyPayload(variables: ITemplateVariable[], payload: Record<string, unknown>) {
  const invalidKeys: string[] = [];
  
  // 1. 检查必填变量（排除系统变量）
  for (const variable of variables.filter(v => v.required && !isSystemVariable(v.name))) {
    const value = variable.name.split('.').reduce((a, b) => a[b], payload);
    
    // 按类型校验
    switch (variable.type) {
      case 'Array':   if (!Array.isArray(value)) invalidKeys.push(...); break;
      case 'Boolean': if (value !== true && value !== false) invalidKeys.push(...); break;
      case 'String':  if (!['string','number'].includes(typeof value)) invalidKeys.push(...); break;
      default:        if (value === null || value === undefined) invalidKeys.push(...);
    }
  }
  
  if (invalidKeys.length) {
    throw new BadRequestException(
      `payload is missing required key(s) and type(s): ${invalidKeys.join(', ')}`
    );
  }
  
  // 2. 填充默认值（排除系统变量）
  return this.fillDefaults(variables.filter(
    v => v.defaultValue != null && !isSystemVariable(v.name)
  ));
}

// 系统变量判断逻辑
isSystemVariable(variableName: string) {
  const prefix = variableName.includes('.') ? variableName.split('.')[0] : variableName;
  return TemplateSystemVariables.includes(prefix);
}
```

**系统变量豁免清单**（`TemplateSystemVariables`）：
`['subscriber', 'step', 'branding', 'tenant', 'preheader', 'actor']`

**关键特性**：
- **系统变量豁免**：`subscriber.*`、`step.*` 等内置变量跳过校验
- **嵌套支持**：通过 `reduce` 递归访问嵌套属性 `user.name`
- **默认值合并**：使用 `toMerged(defaultPayload, command.payload)`，用户传入值优先级更高
- **两层校验独立**：Schema 校验和变量校验是两个独立步骤，结果互不影响

---

#### 触发阶段的其他检查

在 `ParseEventRequest` 中还会执行：

1. **工作流存在性检查**：找不到工作流返回 `workflow_not_found`
2. **工作流激活状态检查**：`active=false` 返回 `NOT_ACTIVE` 状态
3. **租户存在性检查**：租户不存在返回 `TENANT_MISSING` 状态
4. **接收者校验**：校验 `to` 字段格式，过滤无效接收者
5. **TransactionId 唯一性检查**：防止重复触发

**触发状态枚举**（`TriggerEventStatusEnum`）：
- `PROCESSED` - 正常处理，已入队
- `NOT_ACTIVE` - 工作流未激活
- `TENANT_MISSING` - 租户不存在
- `INVALID_RECIPIENTS` - 接收者全部无效

### 阶段 5：渲染时校验与兜底

**触发时机**：实际渲染消息内容时（Worker 进程中）

**核心代码**：
- `packages/framework/src/utils/liquid.utils.ts:11-27` (`defaultOutputEscape`)
- `apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts:110-122`

---

## 六、渲染期兜底策略详解

渲染阶段是最后一道防线，采用"容错优先"策略，确保消息尽可能发送成功。

### 6.1 Liquid 引擎配置差异

系统中有两种不同配置的 Liquid 引擎实例：

| 用途 | `strictVariables` | `catchAllErrors` | 场景 |
|------|-------------------|------------------|------|
| **校验解析** | `true` | `true` | 编辑时/构建时校验，严格模式 |
| **实际渲染** | `false`(默认) | `false`(默认) | 发送时渲染，宽松模式 |

**校验引擎配置**：`libs/application-generic/src/utils/template-parser/liquid-engine.ts:3-10`
```typescript
createLiquidEngine({
  strictVariables: true,    // 缺失变量报错
  strictFilters: true,      // 未知过滤器报错
  greedy: false,
  catchAllErrors: true,     // 捕获所有错误继续执行
});
```

**渲染引擎配置**：无 `strictVariables`，即默认 `false`
```typescript
// 缺失变量不会抛出异常，仅渲染为空字符串
createLiquidEngine({
  outputEscape: customOutputEscape,  // 自定义输出转义
});
```

### 6.2 `undefined` / `null` 变量处理

**核心兜底逻辑**在 `defaultOutputEscape` 函数中：

```typescript
export function defaultOutputEscape(output: unknown): string {
  // 对象/数组：序列化为单引号 JSON
  if (Array.isArray(output) || (typeof output === 'object' && output !== null)) {
    return stringifyDataStructureWithSingleQuotes(output);
  }
  // 字符串：转义特殊字符
  else if (typeof output === 'string') {
    return output.replace(/\\/g, '\\\\').replace(/"/g, '\\"')...;
  }
  // undefined / null：渲染为空字符串
  else {
    return output === undefined || output === null ? '' : String(output);
  }
}
```

**关键行为**：
- `{{ payload.undefinedVar }}` → 渲染为 `""`（空字符串）
- `{{ payload.nullVar }}` → 渲染为 `""`（空字符串）
- `{{ payload.objVar }}` → 渲染为 `{'key':'value'}`（单引号 JSON）
- `{{ payload.arrVar }}` → 渲染为 `[1,2,3]`（单引号 JSON）

### 6.3 邮件渲染的特殊处理

邮件渲染使用自定义 `outputEscape`，避免破坏 HTML 结构：

```typescript
// email-output-renderer.usecase.ts:110-122
this.liquidEngine = createLiquidEngine({
  outputEscape: (output: unknown): string => {
    if (Array.isArray(output) || (typeof output === 'object' && output !== null)) {
      // 对象/数组仍序列化为 JSON（用于循环）
      return JSON.stringify(output).replace(/"/g, "'").replace(/\n/g, '\\n');
    }
    // 字符串不转义引号！否则 style="color:red" 会变成 style=\"color:red\"
    return output === undefined || output === null ? '' : String(output);
  },
});
```

### 6.4 异常兜底

渲染过程中的异常处理策略：

| 场景 | 兜底行为 | 代码位置 |
|------|----------|----------|
| 组织设置获取失败（品牌移除逻辑） | 返回原始 HTML，不中断邮件发送 | `email-output-renderer.usecase.ts:896-898` |
| 翻译后 Maily JSON 解析失败 | 抛出 `InternalServerErrorException` | `email-output-renderer.usecase.ts:566-568` |
| Liquid 渲染后 JSON 解析失败 | 抛出 `InternalServerErrorException` | `email-output-renderer.usecase.ts:625-627` |
| 可迭代变量不是数组 | 抛出 Error，中断渲染 | `email-output-renderer.usecase.ts:767-769` |

---

## 七、触发入口校验与执行期校验的边界澄清

### 7.1 两层校验的对比

| 维度 | 阶段 4.1：请求侧 Schema 校验 | 阶段 4.2：执行侧变量校验 |
|------|----------------------------|------------------------|
| **开关控制** | 受 `validatePayload` 控制，可选 | 始终执行，不可关闭 |
| **校验依据** | `payloadSchema`（JSON Schema） | `variables: ITemplateVariable[]` |
| **校验内容** | 完整结构校验（类型、格式、枚举、默认值） | 仅必填变量存在性和类型 |
| **默认值来源** | Schema 中 `default` 字段 | `ITemplateVariable.defaultValue` |
| **默认值优先级** | 较低，被用户值覆盖 | 较低，被用户值覆盖 |
| **校验工具** | AJV | 自定义 reduce 遍历 |
| **执行位置** | `ParseEventRequest`（API 入口） | `TriggerEvent`（API 入口） |
| **执行时机** | 入队前同步 | 入队前同步 |
| **系统变量** | 不区分，全部校验 | 跳过系统变量 |

### 7.2 关键边界澄清

> **误解 1**：Schema 校验在 Worker 执行期执行
> 
> **事实**：两层校验都在 **API 进程中同步执行**，在任务入队之前完成。Worker 进程不再做变量校验，直接进入渲染阶段。

> **误解 2**：有 issues 的工作流无法触发
> 
> **事实**：`issues` 仅影响前端显示状态（ERROR），**不阻止 API 触发**。只有 `active=false` 会在 `ParseEventRequest` 中返回 `NOT_ACTIVE` 状态，真正阻止触发。

> **误解 3**：`validatePayload=false` 时完全不校验
> 
> **事实**：`validatePayload` 只控制 Schema 校验。即使关闭，**阶段 4.2 的必填变量校验仍然执行**。

> **误解 4**：Worker 执行期会再次校验变量
> 
> **事实**：Worker 中没有变量校验逻辑，直接进入渲染。变量缺失会被渲染阶段的宽松模式兜底为空字符串。

> **误解 5**：`steps` 命名空间可以访问所有步骤
> 
> **事实**：`steps` 仅包含**当前步骤之前**执行的步骤结果，不能访问当前步骤和后续步骤。

> **误解 6**：`layout_content` 是一个普通变量
> 
> **事实**：`layout_content` 是布局特有的必需变量，实际值为 `LAYOUT_CONTENT_VARIABLE = 'content'`，无需命名空间，且仅在布局编辑/渲染时可用。

---

## 八、校验阶段对比总结

| 阶段 | 触发时机 | 严格程度 | 失败后果 | 主要校验内容 |
|------|----------|----------|----------|--------------|
| 1. 编辑时 | 输入内容时 | ⭐⭐⭐⭐⭐ | 红色下划线提示 | 命名空间、Schema、语法 |
| 2. 保存时 | 点击保存时 | ⭐⭐⭐⭐ | 阻止保存 | 格式、必需变量、合法性 |
| 3. 构建时 | 编辑工作流时 | ⭐⭐⭐ | 显示警告，前端状态变 ERROR | Schema、Liquid、自定义规则 |
| **4.1 触发时-Schema** | 调用 Trigger API，`validatePayload=true` | ⭐⭐⭐⭐⭐ | 返回 400 错误 | 完整 JSON Schema 校验 |
| **4.2 触发时-必填** | 调用 Trigger API，始终执行 | ⭐⭐⭐⭐ | 返回 400 错误 | 必填变量存在性、类型匹配 |
| 5. 渲染时 | Worker 实际发送前 | ⭐ | 尽可能渲染 | undefined/null 兜底为空 |

---

## 九、常见疑惑解答

### Q1: 为什么编辑时提示变量不存在，但触发时还能正常发送？

因为**阶段 1/3 的校验使用严格模式**（`strictVariables: true`），而**阶段 5 渲染使用宽松模式**。编辑时的警告是预防性的，实际渲染时缺失变量仅显示为空。

### Q2: `defaultValue` 在哪个阶段生效？

有两处独立的默认值逻辑：
1. **Schema 默认值**：在**阶段 4.1** 由 AJV 根据 `payloadSchema` 中的 `default` 字段填充
2. **变量默认值**：在**阶段 4.2** 通过 `fillDefaults()` 根据 `ITemplateVariable.defaultValue` 填充

两者都在触发入口处执行，且都被用户传入的 payload 覆盖（用户值优先级更高）。

### Q3: 布局变量和模板变量有什么区别？

**声明方式相同**（都用 `ITemplateVariable[]`），但**校验时机和可用范围不同**：
- 布局变量在布局保存时校验（阶段 2）
- 模板变量在触发时校验（阶段 4.2）
- 布局特有 `layout_content` 变量，模板特有 `workflow`、`steps` 变量

### Q4: 为什么 `{{ subscriber.firstName }}` 不需要声明也能通过校验？

因为 `subscriber`、`step`、`branding`、`tenant`、`actor`、`preheader`、`workflow`、`steps`、`env`、`context` 是**系统内置变量**：
- 在 `variableSchema` 中自动包含，阶段 1/3 校验通过
- 在阶段 4.2 中通过 `isSystemVariable()` 跳过必填检查
- 实际值在渲染时由 Worker 动态注入

### Q5: 渲染时变量是 `undefined`，为什么没有报错？

这是**设计预期**。渲染阶段使用 `strictVariables: false`，`undefined` 变量会被 `outputEscape` 转换为空字符串，确保邮件不会因为单个变量缺失而发送失败。

### Q6: 工作流显示 ERROR 状态，为什么还能触发成功？

因为 `computeWorkflowStatus()` 计算的 ERROR 状态仅用于前端显示，**触发入口只检查 `active` 字段**。只要 `active=true`，即使有 issues 也能正常触发。

### Q7: 关闭 `validatePayload` 后，还会检查必填变量吗？

会。`validatePayload` 只控制阶段 4.1 的 Schema 校验，阶段 4.2 的必填变量检查（基于 `ITemplateVariable.required`）始终执行，不可关闭。

### Q8: `steps` 命名空间可以访问哪些步骤？

`steps` 仅包含**当前步骤之前**执行的步骤。例如：
- 工作流顺序：step1 → step2 → step3 → step4
- 在 step3 中，`steps` 包含 step1 和 step2 的结果
- 在 step3 中，**不能**访问 steps.step3（当前步骤）或 steps.step4（后续步骤）

### Q9: `layout_content` 变量有什么特殊要求？

- 布局内容**必须**包含 `{{ layout_content }}` 变量
- 该变量**不需要命名空间**，直接使用 `{{ layout_content }}`
- 实际常量值为 `content`（`LAYOUT_CONTENT_VARIABLE`）
- 渲染时会被替换为实际邮件模板内容
- 仅在布局编辑/渲染时可用，模板中不可用

### Q10: 如何在模板中访问 Digest 聚合的事件？

使用 `steps.{digestStepId}.events` 变量：
```liquid
{% for event in steps.digest-step.events %}
  #{{ forloop.index }}: {{ event.payload.title }}
{% endfor %}

共 {{ steps.digest-step.eventCount }} 条更新
```

支持的格式：
- `steps.digest-step.events` - 完整事件数组
- `steps.digest-step.events[0].payload` - 第一个事件的 payload
- `steps.digest-step.eventCount` - 事件总数

---

## 十、关键代码索引

| 功能模块 | 文件路径 |
|----------|----------|
| 变量接口定义 | `packages/shared/src/types/message-templates.ts:23-28` |
| 系统变量列表 | `packages/shared/src/entities/message-template/message-template.interface.ts:54-90` |
| LAYOUT_CONTENT_VARIABLE 常量 | `packages/shared/src/consts/layouts.ts:1` |
| 工作流 `validatePayload` 字段 | `libs/dal/src/repositories/notification-template/notification-template.entity.ts:92` |
| **命名空间 Schema 构建** | `libs/application-generic/src/utils/create-schema.ts` |
| **模板变量 Schema 构建** | `libs/application-generic/src/usecases/build-variable-schema/build-available-variable-schema.usecase.ts` |
| **布局变量 Schema 构建** | `libs/application-generic/src/usecases/layout-variables-schema/layout-variables-schema.usecase.ts` |
| 前端校验 Hook | `apps/dashboard/src/components/variable/hooks/use-variable-validation.ts` |
| Liquid 变量解析 | `libs/application-generic/src/utils/template-parser/new-liquid-parser.ts` |
| Schema + Liquid 校验 | `libs/application-generic/src/utils/issues.ts` |
| 构建步骤问题 | `libs/application-generic/src/usecases/build-step-issues/build-step-issues.usecase.ts` |
| 构建布局问题 | `apps/api/src/app/layouts-v2/usecases/build-layout-issues/build-layout-issues.usecase.ts` |
| 工作流状态计算 | `libs/application-generic/src/utils/compute-workflow-status.ts` |
| **触发时 Schema 校验** | `apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts:138-157` |
| **触发时必填变量校验** | `libs/application-generic/src/services/verify-payload.service.ts` |
| Payload 校验异常 | `apps/api/src/app/events/exceptions/payload-validation-exception.ts` |
| 触发事件入口 | `libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts` |
| Liquid 引擎工厂 | `packages/framework/src/utils/liquid.utils.ts` |
| 邮件渲染器 | `apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts` |
| 触发状态枚举 | `packages/shared/src/types/trigger-event-status.enum.ts` |
