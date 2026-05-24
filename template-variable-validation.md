# 模板与布局变量校验机制详解

本文档详细说明 Novu 系统中模板（Template）与布局（Layout）的变量声明、校验时机以及渲染期的兜底逻辑。

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

**系统内置变量**（无需声明）：
`packages/shared/src/entities/message-template/message-template.interface.ts:54-90`
- `subscriber` - 订阅者信息（firstName, lastName, email, phone 等）
- `actor` - 执行者信息
- `step` - 步骤信息（digest, events, total_count）
- `branding` - 品牌信息（logo, color）
- `tenant` - 租户信息（name, data）

---

## 二、变量校验的五个阶段

变量校验并非在单一时刻完成，而是分布在五个不同阶段，形成层层递进的防护链。

### 阶段 1：前端编辑实时校验

**触发时机**：用户在 Dashboard 编辑模板/布局内容时

**核心代码**：
- `apps/dashboard/src/components/variable/hooks/use-variable-validation.ts`
- `libs/application-generic/src/utils/issues.ts:195-258` (`processControlValuesByLiquid`)

**校验内容**：
1. **命名空间校验**：变量必须以 `payload.` / `subscriber.` / `context.` 开头
   - 无命名空间的单段变量会提示 `invalid or missing namespace`
   - 建议补全：`Did you mean {{payload.xxx}}?`

2. **Schema 存在性校验**：`payload.` 开头的变量必须在 Payload Schema 中声明
   - 通过 `isPropertyAllowed()` 递归校验属性路径 `new-liquid-parser.ts:174-232`
   - 支持数组索引 `payload.items[0].name`

3. **过滤器有效性校验**：校验 Liquid 过滤器参数合法性
   - 如 `toSentence`、`digest`、`pluralize` 等过滤器的参数

4. **语法正确性校验**：
   - 变量不能包含空格（`contains whitespaces` 错误）
   - Liquid 语法正确性（通过 `parserEngine.parse()` 校验）

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

**校验结果**：以红色警告形式显示在编辑器中，但**不会阻止保存**，仅阻止触发工作流。

### 阶段 4：触发时 Payload 校验

**触发时机**：调用 `/events/trigger` API 发送消息时

**核心代码**：
- `libs/application-generic/src/usecases/verify-payload/verify-payload.usecase.ts`
- `libs/application-generic/src/services/verify-payload.service.ts`

**校验逻辑**：

```typescript
// 1. 检查必填变量是否存在且类型正确
checkRequired(variables: ITemplateVariable[], payload): string[] {
  for (const variable of variables.filter(v => v.required)) {
    const value = variable.name.split('.').reduce((a, b) => a[b], payload);
    
    switch (variable.type) {
      case 'Array':   if (!Array.isArray(value)) invalidKeys.push(...); break;
      case 'Boolean': if (value !== true && value !== false) invalidKeys.push(...); break;
      case 'String':  if (!['string','number'].includes(typeof value)) invalidKeys.push(...); break;
      default:        if (value === null || value === undefined) invalidKeys.push(...);
    }
  }
}

// 2. 填充默认值
fillDefaults(variables): Record<string, unknown> {
  for (const variable of variables.filter(v => v.defaultValue != null)) {
    this.setNestedKey(payload, variable.name.split('.'), variable.defaultValue);
  }
}
```

**校验失败处理**：抛出 `BadRequestException`，返回缺失的字段列表：
```
payload is missing required key(s) and types: userName (Value), userAge (Value)
```

### 阶段 5：渲染时校验与兜底

**触发时机**：实际渲染消息内容时（Worker 进程中）

**核心代码**：
- `packages/framework/src/utils/liquid.utils.ts:11-27` (`defaultOutputEscape`)
- `apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts:110-122`

---

## 三、渲染期兜底策略详解

渲染阶段是最后一道防线，采用"容错优先"策略，确保消息尽可能发送成功。

### 3.1 Liquid 引擎配置差异

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

### 3.2 `undefined` / `null` 变量处理

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

### 3.3 邮件渲染的特殊处理

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

### 3.4 异常兜底

渲染过程中的异常处理策略：

| 场景 | 兜底行为 | 代码位置 |
|------|----------|----------|
| 组织设置获取失败（品牌移除逻辑） | 返回原始 HTML，不中断邮件发送 | `email-output-renderer.usecase.ts:896-898` |
| 翻译后 Maily JSON 解析失败 | 抛出 `InternalServerErrorException` | `email-output-renderer.usecase.ts:566-568` |
| Liquid 渲染后 JSON 解析失败 | 抛出 `InternalServerErrorException` | `email-output-renderer.usecase.ts:625-627` |
| 可迭代变量不是数组 | 抛出 Error，中断渲染 | `email-output-renderer.usecase.ts:767-769` |

---

## 四、校验阶段对比总结

| 阶段 | 触发时机 | 严格程度 | 失败后果 | 主要校验内容 |
|------|----------|----------|----------|--------------|
| 1. 编辑时 | 输入内容时 | ⭐⭐⭐⭐⭐ | 红色下划线提示 | 命名空间、Schema、语法 |
| 2. 保存时 | 点击保存时 | ⭐⭐⭐⭐ | 阻止保存 | 格式、必需变量、合法性 |
| 3. 构建时 | 编辑工作流时 | ⭐⭐⭐ | 显示警告，阻止触发 | Schema、Liquid、自定义规则 |
| 4. 触发时 | 调用 Trigger API 时 | ⭐⭐⭐⭐ | 返回 400 错误 | 必填变量存在性、类型匹配 |
| 5. 渲染时 | 实际发送前 | ⭐ | 尽可能渲染 | undefined/null 兜底为空 |

---

## 五、常见疑惑解答

### Q1: 为什么编辑时提示变量不存在，但触发时还能正常发送？

因为**阶段 1/3 的校验使用严格模式**（`strictVariables: true`），而**阶段 5 渲染使用宽松模式**。编辑时的警告是预防性的，实际渲染时缺失变量仅显示为空。

### Q2: `defaultValue` 在哪个阶段生效？

`defaultValue` 仅在**阶段 4（触发时）**通过 `fillDefaults()` 填充到 payload 中。编辑时/保存时的校验不会自动填充默认值。

### Q3: 布局变量和模板变量有什么区别？

**声明方式相同**（都用 `ITemplateVariable[]`），但**校验时机不同**：
- 布局变量在布局保存时校验（阶段 2）
- 模板变量在触发时校验（阶段 4）

### Q4: 为什么 `{{ subscriber.firstName }}` 不需要声明也能通过校验？

因为 `subscriber`、`step`、`branding`、`tenant`、`actor` 是**系统内置变量**，在 `variableSchema` 中自动包含，无需在 `variables` 数组中声明。

### Q5: 渲染时变量是 `undefined`，为什么没有报错？

这是**设计预期**。渲染阶段使用 `strictVariables: false`，`undefined` 变量会被 `outputEscape` 转换为空字符串，确保邮件不会因为单个变量缺失而发送失败。

---

## 六、关键代码索引

| 功能模块 | 文件路径 |
|----------|----------|
| 变量接口定义 | `packages/shared/src/types/message-templates.ts:23-28` |
| 前端校验 Hook | `apps/dashboard/src/components/variable/hooks/use-variable-validation.ts` |
| Liquid 变量解析 | `libs/application-generic/src/utils/template-parser/new-liquid-parser.ts` |
| Schema + Liquid 校验 | `libs/application-generic/src/utils/issues.ts` |
| 构建步骤问题 | `libs/application-generic/src/usecases/build-step-issues/build-step-issues.usecase.ts` |
| 构建布局问题 | `apps/api/src/app/layouts-v2/usecases/build-layout-issues/build-layout-issues.usecase.ts` |
| Payload 校验服务 | `libs/application-generic/src/services/verify-payload.service.ts` |
| Liquid 引擎工厂 | `packages/framework/src/utils/liquid.utils.ts` |
| 邮件渲染器 | `apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts` |
| 系统变量列表 | `packages/shared/src/entities/message-template/message-template.interface.ts:54-90` |
