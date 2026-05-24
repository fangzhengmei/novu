# 模板与布局变量校验机制详解

本文档详细说明 Novu 系统中模板（Template）与布局（Layout）的变量声明、完整命名空间范围、校验期与渲染期的边界、完整校验链路以及渲染期的兜底逻辑。

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

## 二、校验期可用变量 vs 渲染期注入变量

### 2.1 核心概念澄清

这是两个**本质不同**的概念，界限必须严格区分：

| 维度 | 校验期可用变量（Validation-time Variables） | 渲染期注入变量（Render-time Injected Variables） |
|------|--------------------------------------------|------------------------------------------------|
| **本质** | 通过 `variableSchema` 构建的 **JSON Schema 元数据**，描述"允许访问的变量结构" | Worker 执行时实际传递给 Liquid 引擎的 **真实数据对象**，提供"实际值是什么" |
| **存在形式** | 内存中的 Schema 对象，仅包含字段名和类型定义 | 实际运行时数据对象，包含完整的字段值 |
| **构建时机** | 编辑/保存/构建时动态生成 | Worker 执行消息发送时动态构建 |
| **构建位置** | `build-available-variable-schema.usecase.ts`（模板）<br>`layout-variables-schema.usecase.ts`（布局） | `construct-framework-workflow.usecase.ts`<br>`send-message.usecase.ts`（旧版兼容） |
| **用途** | 编辑/保存/构建/触发时校验变量使用是否合法 | 模板渲染时替换 `{{ variable }}` 占位符的实际值 |
| **Liquid 模式** | 配合 `strictVariables: true` 严格校验 | 配合 `strictVariables: false` 宽松渲染 |

> **核心边界**：校验期 Schema 定义"**可以用什么**"，渲染期注入"**实际值是什么**"。两者范围高度相关但**不完全一致**——Schema 中有的，渲染期不一定注入；Schema 中没有的，渲染期可能实际注入。

---

### 2.2 关键差异对比（按命名空间）

两者最容易混淆的地方在于**同一命名空间下，校验期定义和渲染期实际注入的差异**：

| 命名空间 | 校验期 Schema 定义 | 渲染期实际注入 | 差异说明 |
|----------|------------------|--------------|----------|
| **`workflow`** | 仅定义 5 个字段：<br>`workflowId`, `name`, `description`, `tags`, `severity` | 完整的数据库 `NotificationTemplateEntity` 对象，包含 `_id`, `createdAt`, `updatedAt`, `steps` 等所有字段 | ✅ Schema 是**子集**，渲染期是**超集** |
| **`steps`** | 仅包含**当前步骤之前**的步骤 Schema 定义，按 `stepId` 组织 | 初始为空对象 `{}`，工作流执行过程中**逐步填充**已完成步骤的实际结果 | ✅ Schema 是**前置步骤定义**，渲染期是**动态执行结果** |
| **`subscriber`** | 定义标准字段：`firstName`, `lastName`, `email`, `subscriberId`, `data` 等 | 与 Schema 基本一致，`data` 字段包含用户自定义属性 | ⚠️ 基本一致 |
| **`payload`** | 根据 `payloadSchema` 动态定义（如用户未配置则为空） | 用户触发时传入的完整 payload 对象 | ✅ Schema 是**用户定义结构**，渲染期是**实际传入值** |
| **`context`** | 定义为 `{ [entityType]: { id, data } }` 结构 | 实际解析后的上下文对象，包含 `tenant`, `actor` 等实体 | ⚠️ 基本一致 |
| **`env`** | 仅定义环境变量的**键名列表**（类型均为 string） | 实际键值对，包含用户自定义变量 + 系统内置变量（`name`, `type`） | ✅ Schema 是**键名集合**，渲染期是**完整键值对** |
| **`step`** | ❌ **不在 Schema 中** | ✅ 注入旧版兼容变量：`{ digest, events, total_count }` | ❌ Schema 缺失，渲染期存在 |
| **`branding`** | ❌ **不在 Schema 中** | ✅ 注入品牌信息：`{ logo, color }` | ❌ Schema 缺失，渲染期存在 |
| **`tenant`** | ❌ **不在 Schema 中**（需通过 `context.tenant` 访问） | ✅ 作为**独立顶级变量**注入：`{ name, data }` | ❌ Schema 缺失，渲染期存在 |
| **`actor`** | ❌ **不在 Schema 中**（需通过 `context.actor` 访问） | ✅ 作为**独立顶级变量**注入：`{ firstName, lastName, email, ... }` | ❌ Schema 缺失，渲染期存在 |
| **`preheader`** | ❌ **不在 Schema 中** | ✅ 注入邮件预览文本（string 类型） | ❌ Schema 缺失，渲染期存在 |
| **`content`**<br>（`layout_content`） | ✅ Schema 中定义为 `string` 类型（仅布局） | ✅ 注入**渲染后的邮件正文 HTML 字符串**（仅布局） | ⚠️ 类型一致，但值是**动态渲染结果** |

> **重要结论**：
> 1. **Schema ≠ 实际注入**：不能假设 Schema 中定义的变量渲染期一定有值，也不能假设 Schema 中没有的变量渲染期一定不可用
> 2. **"系统变量"≠"被豁免"**：`branding`, `tenant`, `actor`, `preheader`, `step` 虽然是渲染期注入的系统变量，但**不在 Schema 中定义**，且**只有部分被豁免校验**（详见第四章）

---

### 2.3 覆盖范围与交集全景

#### 2.3.1 完整覆盖范围对比

| 类别 | 命名空间 | 校验期Schema包含 | 渲染期实际注入 | 备注 |
|------|----------|----------------|--------------|------|
| **交集（7个）** | `payload` | ✅ | ✅ |  |
| | `subscriber` | ✅ | ✅ |  |
| | `context` | ✅ | ✅ |  |
| | `workflow` | ✅ | ✅ | 仅模板 |
| | `steps` | ✅ | ✅ | 仅模板 |
| | `env` | ✅ | ✅ |  |
| | `content` | ✅ | ✅ | 仅布局 |
| **仅渲染期（5个）** | `step` | ❌ | ✅ | 仅模板，旧版兼容 |
| | `branding` | ❌ | ✅ |  |
| | `tenant` | ❌ | ✅ | 独立顶级变量 |
| | `actor` | ❌ | ✅ | 独立顶级变量 |
| | `preheader` | ❌ | ✅ | 仅模板 |

#### 2.3.2 集合关系可视化

```
┌──────────────────────────────────────────────────────────┐
│                     渲染期注入变量（12个）                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │              校验期Schema包含（7个）                │  │
│  │  payload, subscriber, context, workflow, steps,    │  │
│  │  env, content(布局)                                │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  仅渲染期（5个）：step, branding, tenant, actor, preheader│
└──────────────────────────────────────────────────────────┘
```

#### 2.3.3 关键差异说明

| 差异点 | 说明 |
|--------|------|
| **Schema 是子集** | 校验期 Schema 定义的变量是渲染期注入的子集（7/12） |
| **Schema 字段不完整** | 即使在交集中，Schema 定义的字段也可能不完整。例如 `workflow` 在 Schema 中仅 5 个字段，但渲染期注入完整 DB 对象 |
| **渲染期有额外变量** | `step`, `branding`, `tenant`, `actor`, `preheader` 这 5 个变量在渲染期注入，但在校验期 Schema 中未定义 |
| **同名但含义不同** | `tenant` 和 `actor` 既可以通过 `context.tenant` / `context.actor` 访问（在 Schema 中），也可以作为独立顶级变量 `tenant` / `actor` 访问（不在 Schema 中） |

---

### 2.4 关联与转化流程

```
用户编辑模板
    ↓
[校验期] 构建 variableSchema（JSON Schema）
    ↓
[校验期] 使用 strictVariables: true 校验变量合法性
    ↓ 校验通过
工作流触发
    ↓
[触发期] 校验 payload 结构和必填变量
    ↓ 校验通过
Worker 执行
    ↓
[渲染期] 构建 FullPayloadForRender（实际数据对象）
    ↓
[渲染期] 使用 strictVariables: false 渲染模板
```

> 校验期和渲染期通过**变量命名约定**关联，但两者是完全独立的构建流程，没有直接的数据依赖。

---

## 三、完整命名空间范围说明

### 3.1 全部可用命名空间总览

系统支持以下命名空间，分为**模板特有**、**布局特有**和**通用**三类。注意**"校验期 Schema 包含"≠"渲染期一定会注入"**，反之亦然：

| 命名空间 | 模板可用 | 布局可用 | 校验期Schema包含 | 渲染期实际注入 | 触发校验豁免 | 说明 |
|----------|---------|---------|----------------|--------------|------------|------|
| `payload` | ✅ | ✅ | ✅ | ✅ | ❌ | 触发时传入的自定义数据 |
| `subscriber` | ✅ | ✅ | ✅ | ✅ | ✅ | 订阅者信息（属于 TemplateSystemVariables） |
| `context` | ✅ | ✅ | ✅ | ✅ | ❌ | 上下文数据（tenant、actor 等，通过 context.tenant 访问） |
| `workflow` | ✅ | ❌ | ✅ | ✅ | ❌ | 工作流元数据（Schema 仅5个字段，渲染期是完整DB对象） |
| `steps` | ✅ | ❌ | ✅（仅前置步骤） | ✅（逐步填充） | ❌ | 前置步骤执行结果 |
| `env` | ✅ | ✅ | ✅ | ✅ | ❌ | 环境变量（用户自定义 + 系统内置） |
| `step` | ✅ | ❌ | ❌ | ✅（旧版兼容） | ✅ | 兼容旧版步骤变量（digest/events/total_count，属于 TemplateSystemVariables） |
| `branding` | ✅ | ✅ | ❌ | ✅ | ✅ | 品牌信息（属于 TemplateSystemVariables） |
| `tenant` | ✅ | ✅ | ❌ | ✅ | ✅ | 租户信息（独立顶级变量，属于 TemplateSystemVariables） |
| `actor` | ✅ | ✅ | ❌ | ✅ | ✅ | 执行者信息（独立顶级变量，属于 TemplateSystemVariables） |
| `preheader` | ✅ | ❌ | ❌ | ✅ | ✅ | 邮件预览文本（属于 TemplateSystemVariables） |
| `content`<br>（`layout_content`） | ❌ | ✅ | ✅ | ✅ | ❌ | 布局内容占位符（布局特有，**实际值为渲染后的HTML字符串**） |

> **关键澄清**：
> 1. **模板特有**：`workflow`、`steps`、`step`、`preheader` —— 仅模板可用，布局不可用
> 2. **布局特有**：`content`（`layout_content`）—— 仅布局可用，模板不可用
> 3. **Schema 缺失但渲染期存在**：`branding`、`tenant`、`actor`、`step`、`preheader` —— 校验期 Schema 中未定义，但渲染期会实际注入
> 4. **触发校验豁免**：仅 `subscriber`、`step`、`branding`、`tenant`、`actor`、`preheader` 这 6 个 `TemplateSystemVariables` 被豁免，其余均不豁免

---

### 3.1.1 `layout_content` / `content` 深度解析

这是最容易混淆的变量，必须严格区分**代码层面的三层含义**和**用户界面的对应关系**：

#### 一、代码层面的三层含义

| 概念 | 具体内容 | 代码位置 |
|------|----------|----------|
| **常量定义** | `LAYOUT_CONTENT_VARIABLE = 'content'` | `packages/shared/src/consts/layouts.ts:1` |
| **实际变量名** | `content` | Schema 定义和渲染期注入时使用的真实 key |
| **实际值（渲染期）** | 渲染后的邮件正文 HTML 字符串 | `email-output-renderer.usecase.ts:406` |

#### 二、用户界面的显示映射

代码中**不存在** `content` → `layout_content` 的动态转换逻辑。`layout_content` 仅存在于：
1. 文档和用户教育材料中
2. 测试用例（`packages/framework/src/utils/liquid.utils.test.ts:330`）

前端代码中始终使用 `{{ content }}` 作为变量名，但在用户界面提示和文档中称为 `layout_content` 以避免歧义。

**代码验证**：

```typescript
// 常量定义（前后端共用此常量，值永远是字符串 'content'）
export const LAYOUT_CONTENT_VARIABLE = 'content';

// 布局校验期 Schema 定义
[LAYOUT_CONTENT_VARIABLE]: {
  type: JsonSchemaTypeEnum.STRING,  // 仅定义类型为 string
}

// 前端布局内容校验（component-utils.tsx:29-30）
// 将预览中的占位符 HTML 替换为实际变量表达式
const cleanedBody = previewBody.replace(
  /<table[^>]*data-content-placeholder[^>]*>[\s\S]*?<\/table>(\s*)/gi,
  `{{ ${LAYOUT_CONTENT_VARIABLE} }}`  // 生成 "{{ content }}"
);

// 渲染期实际注入（email-output-renderer.usecase.ts:406）
[LAYOUT_CONTENT_VARIABLE]: removeBrandingFromHtml(cleanedStepBodyHtml.replace(/\n/g, '')),
// ↑ 这里注入的是经过处理的 HTML 字符串，例如：
// 模板内容 "<div>Hello, {{subscriber.firstName}}</div>"
// 渲染后变成 "<div>Hello, John</div>"
```

#### 三、对应关系总结

```
用户界面显示：{{ layout_content }}
           ↓（概念映射，非代码转换）
代码常量名：content
           ↓（代码运行）
渲染期值：<div>Hello, John</div>（实际 HTML 字符串）
```

> **常见误区警示**：
> - ❌ 错误：`layout_content` 的值是 "content"
> - ✅ 正确：`layout_content` 是用户友好名称，**实际代码中的变量名是 `content`**，**渲染期实际值是邮件正文 HTML 字符串**
> - ❌ 错误：代码中存在 `content` 到 `layout_content` 的动态转换
> - ✅ 正确：代码中始终使用 `content` 作为变量名，`layout_content` 仅是文档和 UI 中的友好称呼
> - ❌ 错误：`layout_content` 是系统变量，触发校验时会被豁免
> - ✅ 正确：`content` 不在 `TemplateSystemVariables` 中，**不会被豁免**，如果在布局 variables 中声明为 `required: true` 会被校验

---

### 3.2 校验期 Schema 构建逻辑

#### 模板编辑/构建期 Schema 构建

**核心代码**：`libs/application-generic/src/usecases/build-variable-schema/build-available-variable-schema.usecase.ts:113-128`

```typescript
return {
  type: JsonSchemaTypeEnum.OBJECT,
  properties: {
    workflow: buildWorkflowSchema(),              // { workflowId, name, description, tags, severity }
    subscriber: buildSubscriberSchema(finalSubscriber), // { firstName, lastName, email, ..., data }
    steps: buildPreviousStepsSchema({...}),       // 仅当前步骤之前的步骤结果
    payload: await this.resolvePayloadSchema(...),// payload 结构
    context: buildContextSchema(finalContext),    // { id, data }
    env: buildEnvSchema(envVars),                 // 环境变量键列表
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
    [LAYOUT_CONTENT_VARIABLE]: {                  // 实际变量名为 'content'
      type: JsonSchemaTypeEnum.STRING,
    },
    context: buildContextSchema(context),
    env: buildEnvSchema(envVars),
  },
  additionalProperties: false,
};
```

#### 关键范围限制

| 限制项 | 说明 |
|--------|------|
| **steps 范围** | 仅包含**当前步骤之前**执行的步骤结果。例如步骤 3 只能访问 `steps.step1` 和 `steps.step2`，不能访问 `steps.step3` 或 `steps.step4` |
| **workflow 范围** | 仅模板可用，布局不可用 |
| **content 范围** | 仅布局可用，模板不可用。实际变量名为 `content`，模板中显示为 `layout_content` |
| **steps.digest** | Digest 步骤的 `events` 字段有特殊处理，支持 `steps.digest-step.events[0].payload` 格式 |

---

### 3.3 渲染期实际注入变量

#### FullPayloadForRender 接口定义

**核心代码**：`apps/api/src/app/environments-v1/usecases/output-renderers/render-command.ts:11-24`

```typescript
export class FullPayloadForRender {
  workflow?: Record<string, unknown>;  // 完整 DB workflow 对象，不止 Schema 字段
  subscriber: Record<string, unknown>;
  payload: Record<string, unknown>;
  context?: ContextResolved;
  steps: Record<string, unknown>;      // stepId -> 步骤结果，初始为空 {}，逐步填充
  env?: Record<string, unknown>;       // 用户自定义 env + 系统变量（name, type）
  [LAYOUT_CONTENT_VARIABLE]?: string;  // 仅布局渲染时注入：'content' = 渲染后的 HTML
}
```

#### 布局渲染时的特殊注入

**核心代码**：`email-output-renderer.usecase.ts:402-413`

```typescript
return this.processBodyContent({
  body: layoutBody,
  payload: {
    ...payload,
    [LAYOUT_CONTENT_VARIABLE]: removeBrandingFromHtml(cleanedStepBodyHtml.replace(/\n/g, '')),
  },
  // ...
});
```

> **`layout_content` 真相**：
> - 常量定义：`LAYOUT_CONTENT_VARIABLE = 'content'`（`packages/shared/src/consts/layouts.ts:1`）
> - 模板中显示：`{{ layout_content }}`（用户友好名）
> - 实际变量名：`content`（注入时使用的 key）
> - 实际值：**渲染后的邮件正文 HTML 字符串**，不是常量 "content"

#### Worker 中额外注入的旧版兼容变量

**核心代码**：`apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts:453-465`

```typescript
return {
  subscriber,
  payload: command.payload,
  step: {                                         // 旧版兼容变量
    digest: !!command.events?.length,
    events: command.events,
    total_count: command.events?.length,
  },
  ...(tenant && { tenant }),                      // 租户信息
  ...(actor && { actor }),                        // 执行者信息
  ...(context && { context }),                    // 上下文
  env,                                            // 环境变量
};
```

---

## 四、系统内置变量完整清单

### 4.1 "系统变量"的三层含义澄清

"系统变量"是一个容易混淆的术语，必须严格区分以下三层含义：

| 概念 | 说明 | 范围 |
|------|------|------|
| **渲染期系统注入变量** | Worker 渲染时自动注入的变量，无需用户传入 | `subscriber`, `step`, `branding`, `tenant`, `actor`, `preheader`, `workflow`, `steps`, `env`, `context`, `content` |
| **`TemplateSystemVariables` 列表** | 代码中明确定义的常量列表，共 6 个 | `['subscriber', 'step', 'branding', 'tenant', 'preheader', 'actor']` |
| **触发校验豁免变量** | 在阶段 4.2 中不进行必填校验和默认值填充的变量 | **等同于 `TemplateSystemVariables` 列表**，仅 6 个 |

> **关键结论**：
> - ❌ 不是所有"系统注入变量"都在 `TemplateSystemVariables` 中
> - ❌ 不是所有"系统注入变量"都能获得触发校验豁免
> - ✅ **只有 `TemplateSystemVariables` 列表中的 6 个变量能获得豁免**

---

### 4.2 `TemplateSystemVariables` 列表（仅 6 个）

**定义位置**：`packages/shared/src/entities/message-template/message-template.interface.ts:54`

```typescript
export const TemplateSystemVariables = ['subscriber', 'step', 'branding', 'tenant', 'preheader', 'actor'];
```

> **⚠️ 绝对注意**：此列表**仅包含 6 个变量**。以下变量虽然也是系统注入的，但**不在此列表中**，也**不会获得校验豁免**：
> - ❌ `workflow` - 工作流元数据
> - ❌ `steps` - 前置步骤结果
> - ❌ `env` - 环境变量
> - ❌ `context` - 上下文数据
> - ❌ `content` - 布局内容占位符

---

### 4.3 系统变量豁免规则（核心）

#### 4.3.1 适用范围

**仅适用于阶段 4.2：触发时必填变量校验和默认值填充**

- **checkRequired()** 方法：过滤掉 `isSystemVariable()` 返回 true 的变量，不进行必填校验
- **fillDefaults()** 方法：过滤掉 `isSystemVariable()` 返回 true 的变量，不填充默认值

#### 4.3.2 核心代码

**定义位置**：`libs/application-generic/src/services/verify-payload.service.ts:71-73`

```typescript
isSystemVariable(variableName: string): boolean {
  // 只取第一段命名空间前缀进行匹配
  const prefix = variableName.includes('.') ? variableName.split('.')[0] : variableName;
  return TemplateSystemVariables.includes(prefix);
}
```

**调用位置 1 - 必填校验过滤**（第 8 行）：
```typescript
for (const variable of variables.filter((vari) => vari.required && !this.isSystemVariable(vari.name))) {
  // 仅对非系统变量进行必填校验
}
```

**调用位置 2 - 默认值填充过滤**（第 46-48 行）：
```typescript
for (const variable of variables.filter(
  (elem) => elem.defaultValue !== undefined && elem.defaultValue !== null && !this.isSystemVariable(elem.name)
)) {
  // 仅对非系统变量填充默认值
}
```

#### 4.3.3 豁免影响全景

| 变量 | 属于 TemplateSystemVariables | 触发校验豁免 | 备注 |
|------|----------------------------|------------|------|
| `subscriber.firstName` | ✅ | ✅ 豁免 | 不校验 required，不填充 defaultValue |
| `step.digest` | ✅ | ✅ 豁免 | 不校验 required，不填充 defaultValue |
| `branding.logo` | ✅ | ✅ 豁免 | 不校验 required，不填充 defaultValue |
| `tenant.name` | ✅ | ✅ 豁免 | 不校验 required，不填充 defaultValue |
| `actor.email` | ✅ | ✅ 豁免 | 不校验 required，不填充 defaultValue |
| `preheader` | ✅ | ✅ 豁免 | 不校验 required，不填充 defaultValue |
| `workflow.name` | ❌ | ❌ **不豁免** | 声明为 required 会被校验 |
| `steps.step1.result` | ❌ | ❌ **不豁免** | 声明为 required 会被校验 |
| `env.API_KEY` | ❌ | ❌ **不豁免** | 声明为 required 会被校验 |
| `context.tenant.id` | ❌ | ❌ **不豁免** | 声明为 required 会被校验 |
| `content`（layout_content） | ❌ | ❌ **不豁免** | 声明为 required 会被校验 |

#### 4.3.4 重要风险提示

如果用户在 `variables` 数组中声明以下变量为 `required: true`，**触发时一定会报错**，因为这些变量不在 payload 中：

```typescript
// ⚠️ 错误示例：这些变量声明为 required 会导致触发失败
const variables: ITemplateVariable[] = [
  { name: 'workflow.name', type: TemplateVariableTypeEnum.STRING, required: true },      // ❌ 不豁免
  { name: 'steps.step1.result', type: TemplateVariableTypeEnum.STRING, required: true }, // ❌ 不豁免
  { name: 'env.API_KEY', type: TemplateVariableTypeEnum.STRING, required: true },        // ❌ 不豁免
  { name: 'context.tenant.id', type: TemplateVariableTypeEnum.STRING, required: true },  // ❌ 不豁免
  { name: 'content', type: TemplateVariableTypeEnum.STRING, required: true },            // ❌ 不豁免
];
```

> **最佳实践**：**永远不要**将系统注入变量声明为 `required: true` 或设置 `defaultValue`。这些变量的值由系统在渲染期动态注入，不受触发时校验控制。

---

### 4.4 完整内置变量详细说明

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
**系统变量豁免**：❌ 不在 TemplateSystemVariables 中

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
**系统变量豁免**：✅ 在 TemplateSystemVariables 中

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

**系统变量豁免**：❌ 不在 TemplateSystemVariables 中

#### 4. `env` - 环境变量

**Schema 定义**：`libs/application-generic/src/utils/create-schema.ts:114-128`

包含两类变量：
1. **用户自定义环境变量** - 在环境设置中配置的键值对
2. **系统内置环境变量**：
   - `env.name` - 环境名称（如 "Production", "Development"）
   - `env.type` - 环境类型

**系统变量豁免**：❌ 不在 TemplateSystemVariables 中

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

**系统变量豁免**：❌ 不在 TemplateSystemVariables 中

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
> **系统变量豁免**：✅ 在 TemplateSystemVariables 中

#### 7. `branding` - 品牌信息

**SystemVariablesWithTypes 定义**：`message-template.interface.ts:80-83`

```typescript
{
  logo: string;            // Logo URL
  color: string;           // 品牌主色
}
```

**系统变量豁免**：✅ 在 TemplateSystemVariables 中

#### 8. `tenant` - 租户信息

**SystemVariablesWithTypes 定义**：`message-template.interface.ts:84-87`

```typescript
{
  name: string;            // 租户名称
  data: object;            // 租户自定义数据
}
```

**系统变量豁免**：✅ 在 TemplateSystemVariables 中

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

**系统变量豁免**：✅ 在 TemplateSystemVariables 中

#### 10. `preheader` - 邮件预览文本

**类型**：`string`

邮件收件人列表中显示的预览文本。
**系统变量豁免**：✅ 在 TemplateSystemVariables 中

#### 11. `content` / `layout_content` - 布局内容占位符（布局特有）

**常量定义**：`packages/shared/src/consts/layouts.ts:1`

```typescript
export const LAYOUT_CONTENT_VARIABLE = 'content';
```

- **模板中使用**：`{{ layout_content }}`（用户友好显示名）
- **实际变量名**：`content`（注入时使用的 key）
- **类型**：`string`
- **实际值**：渲染后的邮件正文 HTML 字符串
- **用途**：标记邮件正文在布局中的插入位置
- **校验要求**：布局内容必须包含此变量（HTML 或 Maily JSON 格式）
- **系统变量豁免**：❌ 不在 TemplateSystemVariables 中

---

## 五、完整校验链路总览

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

## 六、各阶段校验详细说明

### 阶段 1：前端编辑实时校验

**触发时机**：用户在 Dashboard 编辑模板/布局内容时

**核心代码**：
- `apps/dashboard/src/components/variable/hooks/use-variable-validation.ts`
- `libs/application-generic/src/utils/issues.ts:195-258` (`processControlValuesByLiquid`)

**校验内容**：
1. **命名空间校验**：变量必须以 `payload.` / `subscriber.` / `context.` / `workflow.` / `steps.` / `env.` 开头
   - 无命名空间的单段变量会提示 `invalid or missing namespace`
   - 例外：`content`（布局特有，无需命名空间，模板中显示为 `layout_content`）
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

**⚠️ 触发条件修正**：**并非始终执行**，仅在**有状态工作流（Stateful Workflow）**路径下执行，且不受 `validatePayload` 开关控制。

**核心代码**：
- `libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts:73-82`
- `libs/application-generic/src/services/verify-payload.service.ts`

---

##### 4.2.1 不同触发路径下的成立条件

系统存在两种工作流触发路径，必填校验仅在其中一种路径下执行：

| 工作流类型 | 触发路径 | verifyPayload 是否执行 | 说明 |
|-----------|----------|----------------------|------|
| **有状态工作流**<br>（Stateful） | 工作流存储在数据库中，通过 `triggerIdentifier` 查找 | ✅ **执行** | 常规 API 触发路径，经过 `TriggerEvent` 用例 |
| **无状态工作流**<br>（Stateless / Bridge） | 工作流定义在外部 `bridgeUrl`，通过 DISCOVER 接口获取 | ❌ **不执行** | 直接调用 `dispatchEventToWorkflowQueue`，不经过 `TriggerEvent` 用例 |

**代码验证 1 - 执行前提判断**（`trigger-event.usecase.ts:63-82`）：

```typescript
// 只有非 bridge workflow 才会查询并设置 storedWorkflow
if (!command.bridgeWorkflow) {
  storedWorkflow = await this.getAndUpdateWorkflowById({
    environmentId: command.environmentId,
    triggerIdentifier: command.identifier,
    payload: command.payload,
    organizationId: command.organizationId,
    userId: command.userId,
  });
}

// ⚠️ 只有 storedWorkflow 存在时才执行 verifyPayload
if (storedWorkflow) {
  const defaultPayload = this.verifyPayload.execute(
    VerifyPayloadCommand.create({
      payload: command.payload,
      template: storedWorkflow,
    })
  );

  command.payload = toMerged(defaultPayload, command.payload);
}
```

**代码验证 2 - 无状态工作流跳过路径**（`parse-event-request.usecase.ts:96-117`）：

```typescript
const statelessWorkflowAllowed = this.isStatelessWorkflowAllowed(command.bridgeUrl);

if (statelessWorkflowAllowed) {
  const discoveredWorkflow = await this.queryDiscoverWorkflow(command);

  if (!discoveredWorkflow) {
    throw new UnprocessableEntityException('workflow_not_found');
  }

  // ⚠️ 直接入队，不经过 TriggerEvent，因此不执行 verifyPayload
  return await this.dispatchEventToWorkflowQueue({
    requestId,
    command,
    transactionId,
    discoveredWorkflow,
  });
}
```

---

##### 4.2.2 校验逻辑

```typescript
// 仅在有状态工作流路径下的 TriggerEvent.execute() 中调用
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
  
  // 1. 检查必填变量（排除 TemplateSystemVariables）
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
  
  // 2. 填充默认值（排除 TemplateSystemVariables）
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

---

##### 4.2.3 关键特性

| 特性 | 说明 |
|------|------|
| **路径依赖** | 仅在有状态工作流路径下执行，无状态工作流不执行 |
| **不受 validatePayload 控制** | 在有状态工作流路径下，无论 `validatePayload` 开关是否开启，此校验都会执行 |
| **仅豁免 TemplateSystemVariables** | `workflow.*`、`steps.*`、`env.*`、`context.*`、`content` 不豁免 |
| **嵌套支持** | 通过 `reduce` 递归访问嵌套属性 `user.name` |
| **默认值合并** | 使用 `toMerged(defaultPayload, command.payload)`，用户传入值优先级更高 |
| **两层校验独立** | Schema 校验和变量校验是两个独立步骤，结果互不影响 |

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

## 七、渲染期兜底策略详解

渲染阶段是最后一道防线，采用"容错优先"策略，确保消息尽可能发送成功。

### 7.1 Liquid 引擎配置差异

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

### 7.2 `undefined` / `null` 变量处理

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

### 7.3 邮件渲染的特殊处理

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

### 7.4 异常兜底

渲染过程中的异常处理策略：

| 场景 | 兜底行为 | 代码位置 |
|------|----------|----------|
| 组织设置获取失败（品牌移除逻辑） | 返回原始 HTML，不中断邮件发送 | `email-output-renderer.usecase.ts:896-898` |
| 翻译后 Maily JSON 解析失败 | 抛出 `InternalServerErrorException` | `email-output-renderer.usecase.ts:566-568` |
| Liquid 渲染后 JSON 解析失败 | 抛出 `InternalServerErrorException` | `email-output-renderer.usecase.ts:625-627` |
| 可迭代变量不是数组 | 抛出 Error，中断渲染 | `email-output-renderer.usecase.ts:767-769` |

---

## 八、触发入口校验与执行期校验的边界澄清

### 8.1 两层校验的对比

| 维度 | 阶段 4.1：请求侧 Schema 校验 | 阶段 4.2：执行侧变量校验 |
|------|----------------------------|------------------------|
| **开关控制** | 受 `validatePayload` 控制，可选 | 不受 `validatePayload` 控制，但**仅在有状态工作流路径下执行** |
| **校验依据** | `payloadSchema`（JSON Schema） | `variables: ITemplateVariable[]` |
| **校验内容** | 完整结构校验（类型、格式、枚举、默认值） | 仅必填变量存在性和类型 |
| **默认值来源** | Schema 中 `default` 字段 | `ITemplateVariable.defaultValue` |
| **默认值优先级** | 较低，被用户值覆盖 | 较低，被用户值覆盖 |
| **校验工具** | AJV | 自定义 reduce 遍历 |
| **执行位置** | `ParseEventRequest`（API 入口） | `TriggerEvent`（API 入口） |
| **执行时机** | 入队前同步 | 入队前同步（仅有状态工作流） |
| **系统变量** | 不区分，全部校验 | 仅豁免 TemplateSystemVariables（6 个） |
| **路径依赖** | 两种工作流路径下都可能执行 | **仅在有状态工作流路径下执行**，无状态工作流不执行 |

### 8.2 关键边界澄清

> **误解 1**：Schema 校验在 Worker 执行期执行
> 
> **事实**：两层校验都在 **API 进程中同步执行**，在任务入队之前完成。Worker 进程不再做变量校验，直接进入渲染阶段。

> **误解 2**：有 issues 的工作流无法触发
> 
> **事实**：`issues` 仅影响前端显示状态（ERROR），**不阻止 API 触发**。只有 `active=false` 会在 `ParseEventRequest` 中返回 `NOT_ACTIVE` 状态，真正阻止触发。

> **误解 3**：`validatePayload=false` 时完全不校验
> 
> **事实**：`validatePayload` 只控制 Schema 校验。即使关闭，**在有状态工作流路径下，阶段 4.2 的必填变量校验仍然执行**。但在无状态工作流（Bridge）路径下，阶段 4.2 校验根本不执行。

> **误解 4**：Worker 执行期会再次校验变量
> 
> **事实**：Worker 中没有变量校验逻辑，直接进入渲染。变量缺失会被渲染阶段的宽松模式兜底为空字符串。

> **误解 5**：`steps` 命名空间可以访问所有步骤
> 
> **事实**：`steps` 仅包含**当前步骤之前**执行的步骤结果，不能访问当前步骤和后续步骤。

> **误解 6**：`layout_content` 是一个普通变量，值为 "content"
> 
> **事实**：`layout_content` 是布局特有的必需变量，常量名为 `content`，**渲染时实际值为邮件正文 HTML 字符串**，不是常量 "content"。

> **误解 7**：所有系统内置变量在触发校验时都会被豁免
> 
> **事实**：只有 `TemplateSystemVariables` 列表中的 6 个变量（`subscriber`, `step`, `branding`, `tenant`, `preheader`, `actor`）会被豁免。`workflow`, `steps`, `env`, `context`, `content` **不被豁免**，如果声明为 required 会触发校验错误。

> **误解 8**：校验期 Schema 中定义的变量，渲染期一定会注入
> 
> **事实**：`branding`, `tenant`, `actor` 在校验期 Schema 中未定义，但渲染期会实际注入。反之，校验期 Schema 中定义的 `workflow` 等字段是子集，渲染期注入的是完整 DB 对象。

> **误解 9**："系统变量"就是 `TemplateSystemVariables` 列表
> 
> **事实**："系统变量"有三层含义：
> 1. 渲染期系统注入变量（11 个）
> 2. `TemplateSystemVariables` 列表（仅 6 个）
> 3. 触发校验豁免变量（等同于列表，仅 6 个）
> 
> 不要混淆这三层含义。`workflow`, `steps`, `env`, `context`, `content` 是系统注入变量，但不在列表中，也不被豁免。

> **误解 10**：`branding`, `tenant`, `actor` 在 Schema 中定义了，所以可以直接用
> 
> **事实**：`branding`, `tenant`, `actor` **不在校验期 Schema 中定义**。它们能正常使用是因为：
> 1. 渲染期会实际注入这些变量
> 2. Liquid 校验时的宽松匹配或特殊处理
> 
> 但从严格的 Schema 定义角度，它们是缺失的。

> **误解 11**：`layout_content` 是系统变量，会被豁免校验
> 
> **事实**：`layout_content`（实际变量名 `content`）**不在 `TemplateSystemVariables` 列表中**，不会被豁免。如果在布局 variables 中声明为 `required: true`，触发时会校验失败。

> **误解 12**：所有 TemplateSystemVariables 都在 Schema 中定义了
> 
> **事实**：TemplateSystemVariables 的 6 个变量中，只有 `subscriber` 在校验期 Schema 中明确定义。`step`, `branding`, `tenant`, `actor`, `preheader` 都不在 Schema 中定义，但渲染期会注入。

> **误解 13**：触发阶段必填校验始终执行
> 
> **事实**：阶段 4.2 的必填变量校验**仅在有状态工作流路径下执行**。对于无状态工作流（Bridge Workflow，通过 `bridgeUrl` 触发），系统直接调用 `dispatchEventToWorkflowQueue` 入队，不经过 `TriggerEvent` 用例，因此 `verifyPayload` 校验**完全不执行**。
> 
> 执行条件判断逻辑：
> ```typescript
> if (!command.bridgeWorkflow) {
>   storedWorkflow = await this.getAndUpdateWorkflowById({...});
> }
> 
> // 只有 storedWorkflow 存在（即非 bridge 工作流）时才执行校验
> if (storedWorkflow) {
>   const defaultPayload = this.verifyPayload.execute(...);
>   command.payload = toMerged(defaultPayload, command.payload);
> }
> ```

---

## 九、校验阶段对比总结

| 阶段 | 触发时机 | 严格程度 | 失败后果 | 主要校验内容 |
|------|----------|----------|----------|--------------|
| 1. 编辑时 | 输入内容时 | ⭐⭐⭐⭐⭐ | 红色下划线提示 | 命名空间、Schema、语法 |
| 2. 保存时 | 点击保存时 | ⭐⭐⭐⭐ | 阻止保存 | 格式、必需变量、合法性 |
| 3. 构建时 | 编辑工作流时 | ⭐⭐⭐ | 显示警告，前端状态变 ERROR | Schema、Liquid、自定义规则 |
| **4.1 触发时-Schema** | 调用 Trigger API，`validatePayload=true` | ⭐⭐⭐⭐⭐ | 返回 400 错误 | 完整 JSON Schema 校验 |
| **4.2 触发时-必填** | 调用 Trigger API，始终执行 | ⭐⭐⭐⭐ | 返回 400 错误 | 必填变量存在性、类型匹配（仅豁免 6 个系统变量） |
| 5. 渲染时 | Worker 实际发送前 | ⭐ | 尽可能渲染 | undefined/null 兜底为空 |

---

## 十、常见疑惑解答

### Q1: 为什么编辑时提示变量不存在，但触发时还能正常发送？

因为**阶段 1/3 的校验使用严格模式**（`strictVariables: true`），而**阶段 5 渲染使用宽松模式**。编辑时的警告是预防性的，实际渲染时缺失变量仅显示为空。

### Q2: `defaultValue` 在哪个阶段生效？

有两处独立的默认值逻辑，但**都仅在有状态工作流路径下生效**：

1. **Schema 默认值**：在**阶段 4.1** 由 AJV 根据 `payloadSchema` 中的 `default` 字段填充（有状态/无状态工作流路径下都可能执行，取决于 `validatePayload` 开关）
2. **变量默认值**：在**阶段 4.2** 通过 `fillDefaults()` 根据 `ITemplateVariable.defaultValue` 填充（**仅在有状态工作流路径下执行**）

两者都在触发入口处执行，且都被用户传入的 payload 覆盖（用户值优先级更高）。注意：
- **系统变量（TemplateSystemVariables）的默认值会被忽略**，不会填充
- **无状态工作流（Bridge）路径下**，阶段 4.2 校验完全不执行，因此 `ITemplateVariable.defaultValue` 完全不生效

### Q3: 布局变量和模板变量有什么区别？

**声明方式相同**（都用 `ITemplateVariable[]`），但**校验时机和可用范围不同**：
- 布局变量在布局保存时校验（阶段 2）
- 模板变量在触发时校验（阶段 4.2，**仅在有状态工作流路径下执行**）
- 布局特有 `content`（`layout_content`）变量，模板特有 `workflow`、`steps`、`step`、`preheader` 变量

### Q4: 为什么 `{{ subscriber.firstName }}` 不需要声明也能通过校验？

因为 `subscriber`、`step`、`branding`、`tenant`、`actor`、`preheader` 是 **TemplateSystemVariables**，但它们的处理方式分为两类：

#### 第一类：在校验期 Schema 中定义
- `subscriber`：在 `variableSchema` 中明确定义，阶段 1/3 校验通过

#### 第二类：在校验期 Schema 中**未定义**，但渲染期会注入
- `step`、`branding`、`tenant`、`actor`、`preheader`：**不在校验期 Schema 中**，但由于 Liquid 校验时的特殊处理（或 Schema 的宽松匹配），编辑时不会报错

#### 共性：触发校验豁免
- 所有 6 个变量在阶段 4.2 中都通过 `isSystemVariable()` 跳过必填检查
- 实际值都在渲染时由 Worker 动态注入

> **⚠️ 重要澄清**：
> - 不要误以为所有 TemplateSystemVariables 都在 Schema 中定义
> - `step`、`branding`、`tenant`、`actor`、`preheader` 在校验期 Schema 中是缺失的
> - `workflow`、`steps`、`env`、`context` 不在 TemplateSystemVariables 中，虽然校验期 Schema 包含它们，但**不会**豁免必填校验

### Q5: 渲染时变量是 `undefined`，为什么没有报错？

这是**设计预期**。渲染阶段使用 `strictVariables: false`，`undefined` 变量会被 `outputEscape` 转换为空字符串，确保邮件不会因为单个变量缺失而发送失败。

### Q6: 工作流显示 ERROR 状态，为什么还能触发成功？

因为 `computeWorkflowStatus()` 计算的 ERROR 状态仅用于前端显示，**触发入口只检查 `active` 字段**。只要 `active=true`，即使有 issues 也能正常触发。

### Q7: 关闭 `validatePayload` 后，还会检查必填变量吗？

**取决于触发路径**：

- **有状态工作流路径**（工作流存储在数据库中）：**会检查**。`validatePayload` 只控制阶段 4.1 的 Schema 校验，阶段 4.2 的必填变量检查（基于 `ITemplateVariable.required`）不受此开关控制，仍然执行。
- **无状态工作流路径**（通过 `bridgeUrl` 触发）：**不会检查**。无状态工作流直接入队，不经过 `TriggerEvent` 用例，因此阶段 4.2 校验完全不执行。

### Q8: `steps` 命名空间可以访问哪些步骤？

`steps` 仅包含**当前步骤之前**执行的步骤。例如：
- 工作流顺序：step1 → step2 → step3 → step4
- 在 step3 中，`steps` 包含 step1 和 step2 的结果
- 在 step3 中，**不能**访问 steps.step3（当前步骤）或 steps.step4（后续步骤）

### Q9: `layout_content` 变量有什么特殊要求？

需要区分**用户界面显示**和**代码实际运行**的差异：

- **用户界面要求**：布局内容**必须**包含 `{{ layout_content }}` 变量（这是用户友好的显示名）
- **代码实际使用**：代码中始终使用变量名 `content`（通过 `LAYOUT_CONTENT_VARIABLE` 常量）
- **不需要命名空间**：直接使用 `{{ content }}`（代码中）或 `{{ layout_content }}`（UI 显示）
- **实际常量名**：`content`（`LAYOUT_CONTENT_VARIABLE = 'content'`）
- **渲染时实际值**：**渲染后的邮件正文 HTML 字符串**，不是常量 "content"
- **可用范围**：仅在布局编辑/渲染时可用，模板中不可用
- **校验豁免**：不在 TemplateSystemVariables 中，如果在布局 variables 中声明为 required，**在有状态工作流路径下会被校验**（无状态工作流路径下不校验）

> **代码中不存在 `content` → `layout_content` 的动态转换**，`layout_content` 仅作为文档和 UI 中的友好名称使用。

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

### Q11: 哪些系统变量在触发校验时会被豁免？

首先必须澄清：**"系统变量"≠"会被豁免"**。"系统变量"有三层含义，只有中间那层才与豁免相关：

| 概念 | 范围 | 是否被豁免 |
|------|------|----------|
| 渲染期系统注入变量 | 11 个：`subscriber`, `step`, `branding`, `tenant`, `actor`, `preheader`, `workflow`, `steps`, `env`, `context`, `content` | 仅 6 个被豁免 |
| **`TemplateSystemVariables` 列表** | **仅 6 个**：`['subscriber', 'step', 'branding', 'tenant', 'preheader', 'actor']` | ✅ 全部被豁免 |
| 触发校验豁免变量 | 等同于 `TemplateSystemVariables` 列表 | ✅ |

#### ✅ 会被豁免的变量（仅 6 个）：
- `subscriber.*`
- `step.*`（旧版兼容变量）
- `branding.*`
- `tenant.*`
- `preheader`
- `actor.*`

#### ❌ 不会被豁免的变量（即使是系统注入的）：
- `workflow.*` —— 系统注入，但不在 TemplateSystemVariables 中
- `steps.*` —— 系统注入，但不在 TemplateSystemVariables 中
- `env.*` —— 系统注入，但不在 TemplateSystemVariables 中
- `context.*` —— 系统注入，但不在 TemplateSystemVariables 中
- `content`（`layout_content`）—— 系统注入，但不在 TemplateSystemVariables 中

> **💡 记忆技巧**：豁免清单 = `TemplateSystemVariables` = 6 个变量，一个不多一个不少。

### Q12: 校验期可用变量和渲染期注入变量有什么区别？

这是两个**本质不同**的概念，界限必须严格区分：

| 维度 | 校验期可用变量 | 渲染期注入变量 |
|------|--------------|--------------|
| **本质** | JSON Schema 元数据，定义"**可以用什么**" | 真实数据对象，提供"**实际值是什么**" |
| **存在形式** | 内存中的 Schema 对象，仅含字段名和类型 | 运行时数据对象，包含完整字段值 |
| **构建时机** | 编辑/保存/构建时动态生成 | Worker 发送消息时动态构建 |
| **Liquid 模式** | 配合 `strictVariables: true` 严格校验 | 配合 `strictVariables: false` 宽松渲染 |

#### 按命名空间的关键差异：

| 命名空间 | 校验期 Schema 定义 | 渲染期实际注入 |
|----------|------------------|--------------|
| **`workflow`** | 仅 5 个字段：`workflowId`, `name`, `description`, `tags`, `severity` | 完整数据库对象，含 `_id`, `createdAt`, `steps` 等 |
| **`steps`** | 仅前置步骤的 Schema 定义 | 初始 `{}`，执行后逐步填充实际结果 |
| **`env`** | 仅键名列表（类型均为 string） | 完整键值对，含用户自定义 + 系统内置变量 |
| **`step`** | ❌ 不在 Schema 中 | ✅ 注入 `{ digest, events, total_count }` |
| **`branding`** | ❌ 不在 Schema 中 | ✅ 注入 `{ logo, color }` |
| **`tenant`** | ❌ 不在 Schema 中 | ✅ 作为独立顶级变量注入 |
| **`actor`** | ❌ 不在 Schema 中 | ✅ 作为独立顶级变量注入 |
| **`preheader`** | ❌ 不在 Schema 中 | ✅ 注入邮件预览文本 |
| **`content`** | ✅ 定义为 string 类型 | ✅ 注入渲染后的 HTML 字符串 |

> **核心结论**：Schema ≠ 实际注入。不能假设 Schema 中有的渲染期一定有值，也不能假设 Schema 中没有的渲染期一定不可用。

### Q13: 不同触发路径下的校验行为有什么差异？

系统支持两种工作流触发路径，校验行为差异显著：

| 对比项 | 有状态工作流（Stateful） | 无状态工作流（Stateless / Bridge） |
|--------|------------------------|----------------------------------|
| **工作流存储位置** | 数据库中，通过 `triggerIdentifier` 查找 | 外部 `bridgeUrl`，通过 DISCOVER 接口获取 |
| **阶段 4.1 Schema 校验** | 受 `validatePayload` 开关控制 | 受 `validatePayload` 开关控制（如果有 payloadSchema） |
| **阶段 4.2 必填校验** | ✅ **执行**，经过 `TriggerEvent` 用例 | ❌ **不执行**，直接入队 |
| **verifyPayload 调用** | ✅ 调用 | ❌ 不调用 |
| **storedWorkflow 查询** | ✅ 查询数据库 | ❌ 不查询 |
| **ITemplateVariable.defaultValue** | ✅ 生效（非系统变量） | ❌ 不生效 |
| **必填变量检查** | ✅ 执行（非系统变量） | ❌ 不执行 |

**关键代码判断逻辑**：
```typescript
// 只有非 bridge workflow 才会查询并设置 storedWorkflow
if (!command.bridgeWorkflow) {
  storedWorkflow = await this.getAndUpdateWorkflowById({...});
}

// ⚠️ 只有 storedWorkflow 存在时才执行 verifyPayload
if (storedWorkflow) {
  const defaultPayload = this.verifyPayload.execute(...);
  command.payload = toMerged(defaultPayload, command.payload);
}
```

> **重要提示**：如果使用无状态工作流（Bridge），请确保在调用方自行验证 payload 的完整性，因为系统不会执行阶段 4.2 的必填变量校验和默认值填充。

---

## 十一、关键代码索引

| 功能模块 | 文件路径 |
|----------|----------|
| 变量接口定义 | `packages/shared/src/types/message-templates.ts:23-28` |
| TemplateSystemVariables 列表 | `packages/shared/src/entities/message-template/message-template.interface.ts:54` |
| LAYOUT_CONTENT_VARIABLE 常量 | `packages/shared/src/consts/layouts.ts:1` |
| 工作流 `validatePayload` 字段 | `libs/dal/src/repositories/notification-template/notification-template.entity.ts:92` |
| **命名空间 Schema 构建** | `libs/application-generic/src/utils/create-schema.ts` |
| **模板变量 Schema 构建** | `libs/application-generic/src/usecases/build-variable-schema/build-available-variable-schema.usecase.ts` |
| **布局变量 Schema 构建** | `libs/application-generic/src/usecases/layout-variables-schema/layout-variables-schema.usecase.ts` |
| **FullPayloadForRender 接口** | `apps/api/src/app/environments-v1/usecases/output-renderers/render-command.ts:11-24` |
| **渲染期变量注入** | `apps/api/src/app/environments-v1/usecases/construct-framework-workflow/construct-framework-workflow.usecase.ts:144-163` |
| **layout_content 实际值注入** | `apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts:402-413` |
| **Worker 旧版变量构建** | `apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts:453-465` |
| 前端校验 Hook | `apps/dashboard/src/components/variable/hooks/use-variable-validation.ts` |
| Liquid 变量解析 | `libs/application-generic/src/utils/template-parser/new-liquid-parser.ts` |
| Schema + Liquid 校验 | `libs/application-generic/src/utils/issues.ts` |
| 构建步骤问题 | `libs/application-generic/src/usecases/build-step-issues/build-step-issues.usecase.ts` |
| 构建布局问题 | `apps/api/src/app/layouts-v2/usecases/build-layout-issues/build-layout-issues.usecase.ts` |
| 工作流状态计算 | `libs/application-generic/src/utils/compute-workflow-status.ts` |
| **触发时 Schema 校验** | `apps/api/src/app/events/usecases/parse-event-request/parse-event-request.usecase.ts:138-157` |
| **触发时必填变量校验** | `libs/application-generic/src/services/verify-payload.service.ts` |
| **系统变量豁免逻辑** | `libs/application-generic/src/services/verify-payload.service.ts:71-73` |
| Payload 校验异常 | `apps/api/src/app/events/exceptions/payload-validation-exception.ts` |
| 触发事件入口 | `libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts` |
| Liquid 引擎工厂 | `packages/framework/src/utils/liquid.utils.ts` |
| 邮件渲染器 | `apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts` |
| 触发状态枚举 | `packages/shared/src/types/trigger-event-status.enum.ts` |
