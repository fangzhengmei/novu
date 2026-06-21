# Content Engine 模板渲染与 i18n 本地化衔接流程

## 核心架构总览

Novu 的内容渲染和国际化系统涉及 **两个工作流版本（V1/V2）**、**两条渲染路径（Preview/Worker）**、**两套模板引擎（Liquid/Handlebars）** 和 **两种翻译语法（`{{t.key}}` / `{{i18n "key"}}`）**。

最关键的纠正：**`{{t.key}}` 和 `{{i18n "key"}}` 是两种不同的语法，用于不同的系统，它们之间不会互相转换。**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Novu 内容渲染架构概览                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  V1 工作流（模板式 / Legacy）          V2 工作流（Bridge / 新式）          │
│  ──────────────────────────          ─────────────────────────          │
│  模板语法: Handlebars                  模板语法: Liquid                   │
│  翻译语法: {{i18n "key"}}              翻译语法: {{t.key}}                │
│  模板引擎: Handlebars                  模板引擎: Liquid                   │
│  变量替换 + 翻译: 同一 Handlebars pass   变量替换 + 翻译: 两个独立步骤        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 两种翻译语法的区别

### `{{t.key}}` — V2 Bridge 工作流语法

**使用场景**：V2 Bridge 工作流的 step controls 中

**处理方式**：由 `@novu/ee-translation` Enterprise 模块通过正则表达式直接替换为翻译文本

**代码位置**：
- 定义：[translation.constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/translation/translation.constants.ts)
- 正则：`TRANSLATION_KEY_SINGLE_REGEX = /\{\{\s*t\.([^}]+?)\s*\}\}/i`

**示例**：
```handlebars
Hello {{subscriber.firstName}}, {{t.greeting}}!
```

---

### `{{i18n "key"}}` — V1 模板工作流语法

**使用场景**：V1 模板工作流的 Handlebars 模板中

**处理方式**：由 Handlebars 的 `i18n` helper 调用 `i18next.t(key)` 进行翻译

**代码位置**：
- Helper 注册：[compile-template.usecase.ts#L13-L41](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/usecases/compile-template/compile-template.usecase.ts#L13-L41)
- Helper 名称：`HandlebarHelpersEnum.I18N = 'i18n'` — [handlebarHelpers.ts#L12](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/handlebar-helpers/handlebarHelpers.ts#L12)

**示例**：
```handlebars
Hello {{subscriber.firstName}}, {{i18n "greeting"}}!
```

---

## API Preview 路径：各渠道 Output Renderers 详解

Preview 路径共涉及 **5 种渲染器**，每种对翻译和变量替换的处理方式完全不同，不能混成一条流程。

### 核心文件与总体架构

| Channel | Renderer Usecase | 处理策略 |
|---------|------------------|----------|
| Email | [email-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts) | 分主体类型多流程：Maily JSON / 纯文本 body / subject / layout |
| In-App | [in-app-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/in-app-output-renderer.usecase.ts) | 整体 controls JSON → `@novu/ee-translation.execute()` |
| SMS | [sms-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/sms-output-renderer.usecase.ts) | 整体 controls JSON → `@novu/ee-translation.execute()` |
| Push | [push-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/push-output-renderer.usecase.ts) | 整体 controls JSON → `@novu/ee-translation.execute()` |
| Chat | [chat-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/chat-output-renderer.usecase.ts) | 整体 controls JSON → `@novu/ee-translation.execute()` |

所有 Renderer 都继承自 [BaseTranslationRendererUsecase](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/base-translation-renderer.usecase.ts)。

---

### 路径 A：非 Email 渠道（SMS / In-App / Push / Chat）— 整体对象翻译

这四个渠道的实现完全一致：将整个 `controls` 对象序列化为 JSON 字符串，直接调用 `@novu/ee-translation` 一次性完成 **翻译 + 变量替换**。

代码（以 SMS 为例）：[sms-output-renderer.usecase.ts#L28-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/sms-output-renderer.usecase.ts#L28-L38)

```typescript
const translatedControls = await this.processTranslations({
  controls: outputControls,
  variables: renderCommand.fullPayloadForRender,
  // ...
});
```

`processTranslations()` → `executeTranslation()`：[base-translation-renderer.usecase.ts#L22-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/base-translation-renderer.usecase.ts#L22-L58)

```typescript
private async executeTranslation({...}): Promise<string | Record<string, unknown>> {
  const translate = this.getTranslationModule();
  const contentString = typeof content === 'string' ? content : JSON.stringify(content);
  const liquidEngine = createLiquidEngine();

  const translatedContent = await translate.execute({
    resourceId, resourceType, organizationId, environmentId,
    userId: 'system', locale,
    content: contentString,
    payload: variables,
    liquidEngine,
    resourceEntity, organization,
  });

  return typeof content === 'string' ? translatedContent : JSON.parse(translatedContent);
}
```

**完整流程**：

```
controls 对象
  ↓ JSON.stringify()
JSON 字符串（包含所有字段的原始值，如 body/subject 等）
  ↓ @novu/ee-translation.execute(content, payload, liquidEngine)
  ├─ 翻译替换: "{{t.greeting}} {{payload.name}}" → "Hello {{payload.name}}"
  └─ 变量替换: "Hello {{payload.name}}" → "Hello John"
  ↓ JSON.parse()
最终 controls 对象
```

**关键特点**：
- 翻译和变量替换在 **同一次 `@novu/ee-translation.execute()` 调用中完成**（内部先翻译后 Liquid）
- 一次性处理所有字段：`body`、`subject`、`title`、`data` 等所有 controls 字段同时被处理
- 结果直接反序列化为 controls 对象返回

---

### 路径 B：Email Subject — 翻译 + JSON 反转义 + decodeHTML

**代码**：[email-output-renderer.usecase.ts#L492-L522](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L492-L522)

```typescript
private async processSubjectTranslations(subject, variables, ...): Promise<string> {
  const unescapedVariables = this.deepUnescapeTranslationStrings(variables);

  // 调用翻译服务
  const translatedSubject = translationContext
    ? await this.processStringWithContext({ context: translationContext, content: subject, variables: unescapedVariables })
    : await this.processStringTranslations({ content: subject, variables: unescapedVariables, ... });

  return decodeHTML(this.unescapeJsonString(translatedSubject));
}
```

**完整流程**：

```
subject: '{{t.email_subject}} - {{payload.orderId}}'
  ↓ deepUnescapeTranslationStrings(variables)  — 变量中的 \" 转义符取消（因为后续不进入 JSON 解析）
  ↓ @novu/ee-translation
  ├─ 翻译: '{{t.email_subject}}' → 'Order Update'
  └─ 变量替换: '{{payload.orderId}}' → '12345'
translatedSubject: 'Order Update - 12345'
  ↓ unescapeJsonString()  — 取消 JSON 序列化导致的 \" → "
  ↓ decodeHTML()  — &amp; → & 等 HTML 实体解码
最终: 'Order Update - 12345'
```

**与纯文本 body 的区别**：
- Subject **不调用** `liquidEngine.parseAndRender()`（因为翻译服务内部已做 Liquid）
- Subject 额外执行 `unescapeJsonString()` 和 `decodeHTML()`，因为 subject 来自 JSON 序列化的原始字符串
- Subject 没有 Maily JSON / 纯文本分流判断

---

### 路径 C：Email 纯文本 Body — 翻译服务内 Liquid + Renderer 本地 Liquid（双重 Liquid Pass）

**代码**：[email-output-renderer.usecase.ts#L572-L614](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L572-L614)

```typescript
private async processTextTranslations({ text, variables, ... }): Promise<string> {
  const unescapedVariables = this.deepUnescapeTranslationStrings(variables);

  // Step 1: 翻译 + 第一遍 Liquid（@novu/ee-translation 内部，默认配置）
  const translatedText = translationContext
    ? await this.processStringWithContext({ context: translationContext, content: text, variables: unescapedVariables })
    : await this.processStringTranslations({ content: text, variables: unescapedVariables, ... });

  // Step 2: 第二遍 Liquid（Renderer 本地 this.liquidEngine，自定义 outputEscape）
  const unescapedTranslatedText = this.unescapeJsonString(translatedText);
  return await this.liquidEngine.parseAndRender(unescapedTranslatedText, unescapedVariables);
}
```

**完整流程（两遍 Liquid Pass）**：

```
body: '<p>{{t.greeting}} {{subscriber.firstName}}! {{t.personalized}}</p>'

翻译:
  t.greeting     → 'Hello'
  t.personalized → 'Your order {{payload.orderId}} is on the way'

  ↓ deepUnescapeTranslationStrings(variables)  — 变量字符串中的 \" 取消转义
  ↓ Step 1: @novu/ee-translation 翻译替换 + 翻译服务内部 Liquid
  ├─ {{t.greeting}}     → 'Hello'
  ├─ {{subscriber.firstName}} → 'Smith'（翻译服务内 Liquid 可能已解析）
  └─ {{t.personalized}} → 'Your order {{payload.orderId}} is on the way'
translatedText: '<p>Hello Smith! Your order {{payload.orderId}} is on the way</p>'
  ↓ unescapeJsonString()  — \" → "
  ↓ Step 2: this.liquidEngine.parseAndRender() 第二遍 Liquid (自定义 outputEscape)
  └─ {{payload.orderId}} → '12345'
最终: '<p>Hello Smith! Your order 12345 is on the way</p>'
```

**关键特点**：
- **翻译先、Renderer 本地 Liquid 后**，两个阶段分开
- 翻译文本中可以包含 Liquid 变量，在 Step 2 被解析
- **变量替换实际跑了两遍**：翻译服务内部一遍（默认 Liquid 配置），Renderer 本地一遍（自定义 `outputEscape`）
- Renderer 自有的 `this.liquidEngine` 的 `outputEscape` 对对象/数组做 `JSON.stringify` + 双引号转单引号 + 换行转义，保证结果能嵌入更大结构；字符串不做 HTML 转义

**为什么要两遍 Liquid**：
- 翻译服务用 `createLiquidEngine()` 默认配置，outputEscape 对数组/对象不做特殊处理
- Email 场景中 payload 可能含有数组/对象，Step 1 翻译服务的默认 Liquid 输出可能不符合 JSON 兼容性要求
- Step 2 用 Renderer 自有 liquidEngine（自定义 outputEscape）做最终保障，确保对象/数组输出的字符串能被正确 `JSON.parse`

---

### 路径 D：Email Maily JSON Body — 五阶段流水线

Maily 是 Novu Email 编辑器使用的 JSON 格式（类似 ProseMirror / Tiptap），body 字段不是 HTML 字符串，而是形如 `{ "type": "doc", "content": [...] }` 的 JSON 对象。

**代码入口**：[email-output-renderer.usecase.ts#L456-L490](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L456-L490)

```typescript
private async processBodyContent({ body, ... }): Promise<string> {
  if (typeof body === 'object' || (typeof body === 'string' && isJsonString(body))) {
    // ===== Maily JSON 路径 =====
    const unescapedPayload = this.deepUnescapeTranslationStrings(payload);
    const escapedPayloadForJson = this.deepEscapePayloadStrings(unescapedPayload);

    const liquifiedMaily = wrapMailyInLiquid(this.enhanceContentVariable(body));   // 阶段 1
    const transformedMaily = await this.transformMailyContent(liquifiedMaily, escapedPayloadForJson);  // 阶段 2
    const translatedMaily = await this.processMailyTranslations({ mailyContent: transformedMaily, ... });  // 阶段 3
    const parsedMaily = await this.parseMailyContentByLiquid(translatedMaily, escapedPayloadForJson);   // 阶段 4
    const renderedMaily = await mailyRender(parsedMaily, { noHtmlWrappingTags });  // 阶段 5
    return decodeHTML(renderedMaily);
  } else {
    // ===== 纯文本路径 C =====
    return await this.processTextTranslations(...);
  }
}
```

#### 阶段 1：`wrapMailyInLiquid()` — Maily 节点属性按需转 Liquid 表达式

**代码**：[maily-utils.ts#L455-L482](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/utils/maily-utils.ts#L455-L482)

遍历 Maily JSON 所有节点，**按 `variableAttributeConfig(type)` 返回的属性白名单**将属性值转为 `{{ variable | default: 'fallback' }}` 形式的 Liquid 表达式。

**属性白名单映射**（由 `variableAttributeConfig()` 在 [maily-utils.ts#L96-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/utils/maily-utils.ts#L96-L146) 定义）：

| 节点 type | 可包裹 Liquid 的 attr (flag) |
|-----------|------------------------------|
| `variable` / `repeat` / `for` / 所有未显式列出的 type | `id(id)`、`showIfKey(showIfKey)`、`each(each)` — 即 commonConfig |
| `button` | `text(isTextVariable)`、`url(isUrlVariable)` + commonConfig |
| `image` / `inlineImage` | `src(isSrcVariable)`、`externalLink(isExternalLinkVariable)` + commonConfig |
| `link` | `href(isUrlVariable)` + commonConfig |

**包裹触发条件**（`processVariableNodeAttributes`）：
```typescript
const flagValue = attrs[flag];   // 例如 flag='isTextVariable'，attrs.isTextVariable 必须为真
const attrValue = attrs[attr];   // 例如 attr='text'
if (!flagValue || !attrValue || typeof attrValue !== 'string') return;  // flag 为假则不包裹
```

- 只有 `flag` 对应的属性**为真**时，才会把 `attr` 的值包裹为 Liquid
- 对 `variable` 节点：flag 和 attr 都是 `id`，只要 `id` 本身非空字符串（**总是 truthy**），就一定包裹

**示例**：

```
输入 variable 节点: { "type": "variable", "attrs": { "id": "payload.user.name", "fallback": "Guest" } }
输出:              { "type": "variable", "attrs": { "id": "{{ payload.user.name | default: 'Guest' }}", ... } }

输入 button 节点:  { "type": "button", "attrs": { "text": "payload.link", "isTextVariable": true } }
输出:              { "type": "button", "attrs": { "text": "{{ payload.link }}", "isTextVariable": true, ... } }
```

##### 翻译 key 豁免规则（仅 button text）

**代码**：[maily-utils.ts#L460-L468](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/utils/maily-utils.ts#L460-L468)

```typescript
shouldProcessAttr: ({ attrValue, attrKey, attrs }) => {
  // 仅当 attrKey === 'text'（button text）且 isTextVariable === true 且值匹配 {{t.key}} 时才豁免
  if (
    attrKey === MailyAttrsEnum.TEXT &&
    attrs.isTextVariable === true &&
    TRANSLATION_KEY_SINGLE_REGEX.test(attrValue)
  ) {
    return false;  // 不包裹 Liquid，保留原始 {{t.key}}
  }
  return true;
}
```

**豁免仅适用于 button 的 text 属性**。`variable` 节点的 `id`、button 的 `url`、image 的 `src`、link 的 `href` — 这些属性即便是翻译 key 仍然会被包裹 Liquid。

| 节点 + attr | 值为翻译 key 时的行为 |
|------------|----------------------|
| `variable` 节点的 `id`（值如 `'t.greeting'`） | ✅ 被包裹为 `{{ t.greeting }}`（flag=`id` 总是 truthy，且豁免规则不命中 `id` attr） |
| `button.attrs.text`（`isTextVariable=true`，值如 `'{{t.order_status}}'`） | ❌ **豁免**，保留 `{{t.order_status}}` 原样 |
| `button.attrs.url`（`isUrlVariable=true`，值如 `'t.track_url'`） | ✅ 被包裹为 `{{ t.track_url }}`（豁免仅针对 `text` attr） |
| `image.attrs.src`（`isSrcVariable=true`） | ✅ 被包裹 |
| `link.attrs.href`（`isUrlVariable=true`） | ✅ 被包裹 |

---

#### 阶段 2：`transformMailyContent()` — 结构化节点预处理（show/each/variable）

**代码**：[email-output-renderer.usecase.ts#L631-L665](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L631-L665)

BFS 遍历 Maily 节点树，对特殊节点做预处理：

| 节点类型 | 处理逻辑 |
|---------|----------|
| `show` 条件节点 | `handleShowNode()`：用 Liquid 执行 `showIf` 表达式，返回 false 则从 parent.content 中移除该节点 |
| `each` 循环节点 | `multiplyForEachNode()`：展开循环，为每个迭代复制子节点并重写变量路径为带数组下标的形式 |
| `variable` 变量节点 | `processVariableNodeTypes()`：将 `{type:"variable", attrs:{id:"{{ ... }}"}}` 改为 `{type:"text", text:"{{ ... }}"}`，把 Liquid 表达式移到 `text` 字段以便后续在 JSON 字符串中被 Liquid 识别 |

**关键细节**：`processVariableNodeTypes()` 不做任何变量解析，只是**改变节点结构**：
```typescript
private processVariableNodeTypes(node: MailyJSONContent) {
  node.type = 'text';                // variable → text
  node.text = node.attrs?.id || '';  // attrs.id → text（id 里已经是 {{ ... }} 形式）
}
```

`multiplyForEachNode()` 中的 `addIndexToLiquidExpression`：
- 原始：`{{ payload.comments.author }}`
- 迭代 0：`{{ payload.comments[0].author }}`
- 迭代 1：`{{ payload.comments[1].author }}`

---

#### 阶段 3：`processMailyTranslations()` — 翻译服务替换

**代码**：[email-output-renderer.usecase.ts#L524-L570](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L524-L570)

```typescript
private async processMailyTranslations({ mailyContent, variables, ... }): Promise<MailyJSONContent> {
  const contentString = JSON.stringify(mailyContent);
  const translatedContent = translationContext
    ? await this.processStringWithContext({ context: translationContext, content: contentString, variables })
    : await this.processStringTranslations({ content: contentString, variables, ... });

  return JSON.parse(translatedContent);
}
```

**流程**：
1. 将整个 Maily JSON 序列化为字符串
2. 调用 `@novu/ee-translation`（`Translate.execute()` 或 `executeWithContext()`）对 JSON 字符串做翻译
3. 反序列化为 Maily JSON 对象

##### `@novu/ee-translation` 的职责边界

从其调用签名和参数推断（[base-translation-renderer.usecase.ts#L229-L241](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/base-translation-renderer.usecase.ts#L229-L241)）：

```typescript
const translatedContent = await translate.execute({
  resourceId, resourceType, organizationId, environmentId,
  userId: 'system', locale,
  content: contentString,     // 原始内容字符串
  payload: variables,          // 完整变量对象
  liquidEngine,                // Liquid 引擎实例
  resourceEntity, organization,
});
```

**翻译服务内部**做两件事：
1. **翻译替换**：通过 `TRANSLATION_KEY_SINGLE_REGEX` 正则匹配 `{{t.key}}` / `{{ t.key }}`，替换为对应 locale 的翻译文本
2. **Liquid 变量替换**：对翻译后的文本调用传入的 `liquidEngine.parseAndRender()` 解析 `{{payload.xxx}}` 等变量（**至少对非 Email 路径是如此**）

但在 **Email Maily 路径**中，`processMailyTranslations()` 之后**还有**阶段 4 的 `parseMailyContentByLiquid()`，因此翻译服务对 Maily JSON 字符串内部 Liquid 变量的解析结果会再次被阶段 4 覆盖或补充。对 Email 来说，阶段 3 的核心职责是**翻译替换**，阶段 4 的核心职责是**Liquid 变量替换（包括自定义 outputEscape 的处理）**。

翻译服务不负责：
- Maily 节点结构处理（show/each/variable 展开在阶段 2）
- Maily JSON 到 HTML 的转换（在阶段 5）
- 自定义 outputEscape 转义（阶段 4 用 Renderer 自有 liquidEngine）

---

#### 阶段 4：`parseMailyContentByLiquid()` — Liquid 变量替换（Renderer 自有引擎）

**代码**：[email-output-renderer.usecase.ts#L616-L629](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L616-L629)

```typescript
private async parseMailyContentByLiquid(mailyContent, variables): Promise<MailyJSONContent> {
  const parsedString = await this.liquidEngine.parseAndRender(JSON.stringify(mailyContent), variables);
  return JSON.parse(parsedString);
}
```

- 将翻译后的 Maily JSON 再次序列化为字符串
- 用 **Renderer 自有的 `this.liquidEngine`** 解析所有 `{{payload.xxx}}`、`{{subscriber.xxx}}` 等 Liquid 表达式
- 反序列化为 Maily JSON 对象

**自定义 `outputEscape` 的作用**（[email-output-renderer.usecase.ts#L110-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L110-L122)）：

```typescript
outputEscape: (output: unknown): string => {
  if (Array.isArray(output) || (typeof output === 'object' && output !== null)) {
    // 对象/数组：JSON.stringify → 双引号转单引号 → 转义换行
    const valueStringified = JSON.stringify(output);
    const valueSingleQuotes = valueStringified.replace(/"/g, "'");
    const valueEscapedNewLines = valueSingleQuotes.replace(/\n/g, '\\n');
    return valueEscapedNewLines;
  }
  // 字符串/数字/布尔：不做 HTML 转义，直接输出
  return output === undefined || output === null ? '' : String(output as unknown);
}
```

这确保 Maily JSON 中的 Liquid 变量（如 `{{payload.items}}` 对应一个数组）输出后仍能被 `JSON.parse` 正确解析，同时字符串中的引号、HTML 属性不被破坏。

---

#### 阶段 5：`mailyRender()` + `decodeHTML()` — Maily JSON 转 HTML

- `@novu/maily-render` 的 `render()` 函数将处理完的 Maily JSON 转为 HTML 字符串
- `decodeHTML()` 对 HTML 实体做解码（如 `&amp;` → `&`）

---

### Maily JSON 完整五阶段示例（精确版）

```
原始 Maily JSON (混合 variable 节点 + button 翻译 key):
{
  type: 'doc',
  content: [
    { type: 'paragraph', content: [
      // variable 节点，id 是翻译 key（无 {{ }} 包裹）
      { type: 'variable', attrs: { id: 't.greeting' } },
      { type: 'text', text: ' ' },
      // variable 节点，id 是普通变量
      { type: 'variable', attrs: { id: 'subscriber.firstName' } },
      { type: 'text', text: '! ' },
      // button 节点，text 是翻译 key（带 {{ }} 包裹，isTextVariable=true）
      { type: 'button', attrs: {
          text: '{{t.order_status}}', isTextVariable: true,
          url: 't.track_url', isUrlVariable: true   // url 也是翻译 key（但无 {{ }}，且豁免仅针对 text）
      }}
    ]}
  ]
}

翻译数据:
  t.greeting     → 'Hello'
  t.order_status → 'Your order {{payload.orderId}} is ready'
  t.track_url    → 'https://example.com/track/{{payload.orderId}}'

payload: { orderId: '12345' }
subscriber: { firstName: 'Smith' }
```

**阶段 1: `wrapMailyInLiquid()`**

```
{
  type: 'doc', content: [{ type: 'paragraph', content: [
    // variable.id='t.greeting'：flag=id, attrValue 为 truthy → 包裹为 {{ t.greeting }}
    { type: 'variable', attrs: { id: '{{ t.greeting }}' } },
    { type: 'text', text: ' ' },
    // variable.id='subscriber.firstName' → 包裹为 {{ subscriber.firstName }}
    { type: 'variable', attrs: { id: '{{ subscriber.firstName }}' } },
    { type: 'text', text: '! ' },
    // button.text='{{t.order_status}}': attrKey=TEXT, isTextVariable=true, 正则匹配 → 豁免！保留原样
    // button.url='t.track_url': attrKey=URL（非 TEXT） → 不触发豁免，包裹为 {{ t.track_url }}
    { type: 'button', attrs: {
        text: '{{t.order_status}}', isTextVariable: true,
        url: '{{ t.track_url }}', isUrlVariable: true
    }}
  ]}]
}
```

**阶段 2: `transformMailyContent()` (variable→text 转换)**

```
{
  type: 'doc', content: [{ type: 'paragraph', content: [
    // variable → text, attrs.id → text
    { type: 'text', text: '{{ t.greeting }}' },
    { type: 'text', text: ' ' },
    { type: 'text', text: '{{ subscriber.firstName }}' },
    { type: 'text', text: '! ' },
    // button 不变（不是 variable 节点）
    { type: 'button', attrs: {
        text: '{{t.order_status}}', isTextVariable: true,
        url: '{{ t.track_url }}', isUrlVariable: true
    }}
  ]}]
}
```

**阶段 3: `processMailyTranslations()` (翻译服务 — JSON 字符串替换)**

Maily JSON 被 stringify 后，翻译服务在 JSON 字符串中匹配 `{{ t.greeting }}`、`{{t.order_status}}`、`{{ t.track_url }}`：

```
"{{ t.greeting }}"        → "Hello"
"{{t.order_status}}"      → "Your order {{payload.orderId}} is ready"
"{{ t.track_url }}"       → "https://example.com/track/{{payload.orderId}}"
```

结果（翻译服务可能已解析部分 Liquid，但阶段 4 会再跑一次 Liquid 确保完整性）：

```
{
  type: 'doc', content: [{ type: 'paragraph', content: [
    { type: 'text', text: 'Hello' },
    { type: 'text', text: ' ' },
    { type: 'text', text: '{{ subscriber.firstName }}' },
    { type: 'text', text: '! ' },
    { type: 'button', attrs: {
        text: 'Your order {{payload.orderId}} is ready', isTextVariable: true,
        url: 'https://example.com/track/{{payload.orderId}}', isUrlVariable: true
    }}
  ]}]
}
```

**阶段 4: `parseMailyContentByLiquid()` (Renderer Liquid 引擎 — 变量替换)**

```
{{ subscriber.firstName }} → 'Smith'
{{payload.orderId}}        → '12345'
```

```
{
  type: 'doc', content: [{ type: 'paragraph', content: [
    { type: 'text', text: 'Hello' },
    { type: 'text', text: ' ' },
    { type: 'text', text: 'Smith' },
    { type: 'text', text: '! ' },
    { type: 'button', attrs: {
        text: 'Your order 12345 is ready', isTextVariable: true,
        url: 'https://example.com/track/12345', isUrlVariable: true
    }}
  ]}]
}
```

**阶段 5: `mailyRender()` → HTML**

```html
<p>Hello Smith! <a href="https://example.com/track/12345">Your order 12345 is ready</a></p>
```

---

### `@novu/ee-translation` 与本地 Liquid 的职责边界总表

| 职责 | `@novu/ee-translation` (阶段 3) | Renderer 本地 Liquid (阶段 4) |
|------|---------------------------------|-------------------------------|
| 翻译 `{{t.key}}` → 文本 | ✅ 是（核心职责） | ❌ 否（Liquid 不识别 t. 前缀） |
| 解析 `{{payload.xxx}}` 变量 | ⚠️ 内部有 Liquid 引擎，可能会解析 | ✅ 是（最终保障，用自定义 outputEscape） |
| 解析 `{{subscriber.xxx}}` 变量 | ⚠️ 同上 | ✅ 是 |
| 自定义 outputEscape (对象/数组转义) | ❌ 否（翻译服务用 `createLiquidEngine()` 默认配置） | ✅ 是（Renderer 自有 liquidEngine） |
| Maily show/each 节点处理 | ❌ 否 | ❌ 否（阶段 2 已完成） |
| Maily JSON → HTML 渲染 | ❌ 否 | ❌ 否（阶段 5 mailyRender） |

**设计意图**：
- 对 **SMS/In-App/Push/Chat**：翻译服务内部的 Liquid 引擎**足够用**，不需要再跑本地 Liquid
- 对 **Email (Maily)**：翻译服务的 Liquid 引擎配置不满足 Maily JSON 对对象/数组转义的特殊需求，因此额外在阶段 4 用 Renderer 自有 liquidEngine 再跑一次，确保 JSON 完整性

---

### 路径 E：Email Layout — body 进入 layout 的二次渲染

当 step 指定了 layout 时，在 step body 渲染完成后，将其作为 `{{ layout_content }}` 变量的值注入 layout body，再走一次 `processBodyContent()`。

**代码**：[email-output-renderer.usecase.ts#L388-L414](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L388-L414)

```typescript
// Layout 不经过 Framework，需手动将 't.key' filter 参数转为 '{{t.key}}'
const layoutBody = (layoutControlValues.email?.body ?? '').replace(/'t\.([\p{L}\p{N}_.-]+)'/gu, "'{{t.$1}}'");

return this.processBodyContent({
  body: layoutBody,
  payload: {
    ...payload,
    [LAYOUT_CONTENT_VARIABLE]: removeBrandingFromHtml(cleanedStepBodyHtml.replace(/\n/g, '')),
  },
  resourceId: overriddenStepLayoutId,
  resourceType: LocalizationResourceEnum.LAYOUT,
  locale,
});
```

**特殊点**：
- Layout body 直接从 DB 读取，**不经过 Framework**，所以没有 `preprocessFilterTranslationArgs` → `postprocessTranslationMarkers` 的保护
- 用一条正则捷径：`'t.key'` → `'{{t.key}}'`，直接把 filter 字符串参数变成翻译标记（Liquid 在引号内不解析 `{{ }}`，所以透传）
- `LAYOUT_CONTENT_VARIABLE`（即 `layout_content`）作为 Liquid 变量注入 layout 的 body，值为 step body 渲染好的 HTML

**layout_content 特殊处理**：[enhanceContentVariable()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L416-L431) 将 Maily JSON 中 `id === 'layout_content'` 的 variable 节点加上 `shouldDangerouslySetInnerHTML: true`，让渲染时直接注入 HTML 而不做转义。

---

### 各路径翻译 + 变量替换顺序对比表

| 路径 | 翻译替换位置 | Liquid 变量替换次数 & 位置 | 翻译与变量是否分开 | 翻译服务内部是否跑 Liquid |
|------|-------------|---------------------------|-------------------|--------------------------|
| SMS/In-App/Push/Chat | `@novu/ee-translation.execute()` 内部 | **1 次** — 翻译服务内部 | ❌ 同一步 | ✅ 是，一次搞定 |
| Email Subject | `processSubjectTranslations` → `processStringTranslations` | **1 次** — 翻译服务内部 | ❌ 同一步 | ✅ 是，后续只做 unescapeJsonString + decodeHTML |
| Email 纯文本 Body | `processTextTranslations` 前半段（翻译服务） | **2 次** — 翻译服务内 1 次 + `this.liquidEngine.parseAndRender()` 1 次 | ✅ 翻译阶段与 Renderer 本地 Liquid 阶段分开 | ✅ 是，但 Step 2 用自定义 outputEscape 再跑一遍做最终保障 |
| Email Maily JSON Body | `processMailyTranslations`（阶段 3, JSON 字符串） | **2 次** — 翻译服务内 1 次（阶段 3）+ `this.liquidEngine.parseAndRender()` 1 次（阶段 4） | ✅ 阶段 3 与阶段 4 分开 | ✅ 是，但阶段 4 用自定义 outputEscape 保障 JSON 可解析性 |
| Email Layout Body | 走 processBodyContent（同纯文本或 Maily） | 同上 | ✅ 两步（同上） | ✅ 同上 |

---

### 各路径涉及的关键方法一览

| 方法 | 所在类 | 仅哪个路径用 |
|------|--------|-------------|
| `processTranslations()` | BaseTranslationRenderer | SMS/In-App/Push/Chat |
| `processStringTranslations()` | BaseTranslationRenderer | Email Subject, Email 纯文本 Body, Maily 阶段 3 |
| `processStringWithContext()` | BaseTranslationRenderer | 同上（有缓存上下文时） |
| `processSubjectTranslations()` | EmailOutputRenderer | Email Subject 专属 |
| `processTextTranslations()` | EmailOutputRenderer | Email 纯文本 Body 专属 |
| `processMailyTranslations()` | EmailOutputRenderer | Email Maily JSON Body 阶段 3 |
| `transformMailyContent()` | EmailOutputRenderer | Email Maily JSON Body 阶段 2 |
| `parseMailyContentByLiquid()` | EmailOutputRenderer | Email Maily JSON Body 阶段 4 |
| `wrapMailyInLiquid()` | application-generic/maily-utils | Email Maily JSON Body 阶段 1 |
| `executeTranslation()` | BaseTranslationRenderer | 所有路径底层共用 |
| `createTranslationContext()` | BaseTranslationRenderer | Email（有缓存上下文时） |

---

## Worker 路径：实际发送流程

### V1 工作流（模板式）

**代码位置**：各 `send-message-*.usecase.ts` 中的 `!command.bridgeData` 分支

```
SendMessage*.execute()
  ↓
initiateTranslations() → 创建 i18next 实例
  ↓
CompileEmailTemplate / CompileTemplate.execute(template, data, i18nInstance)
  ↓
Handlebars.compile(template)(data)
  ↓
  ├─ 变量替换: {{subscriber.firstName}} → "Alice"
  └─ i18n helper: {{i18n "greeting"}} → "你好"
```

**关键代码**：
- i18n helper 注册：[compile-template.usecase.ts#L18-L41](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/usecases/compile-template/compile-template.usecase.ts#L18-L41)
- i18nInstance 创建：[send-message.base.ts#L189-L228](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts#L189-L228)
- V1 分支判断：[send-message-email.usecase.ts#L235-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L235-L263)

**V1 特点**：
- 变量替换和翻译在 **同一个 Handlebars 编译过程** 中完成
- 翻译语法：`{{i18n "key"}}`
- i18n helper 内部调用 `i18next.t(key, options)`
- 支持 hash 参数：`{{i18n "greeting" name=subscriber.firstName}}`

---

### V2 工作流（Bridge 式）

**代码位置**：各 `send-message-*.usecase.ts` 中的 `command.bridgeData` 分支

```
SendMessage*.execute()
  ↓
initiateTranslations() → 创建 i18next 实例（可能未使用）
  ↓
[ if command.bridgeData exists ]
  ↓
content = command.bridgeData?.outputs?.body  // 直接使用 bridge 输出
  ↓
[ 跳过 CompileTemplate ]
```

**关键代码**：
- V2 分支判断：[send-message-chat.usecase.ts#L164-L176](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts#L164-L176)
- V2 Email：[send-message-email.usecase.ts#L235-L263](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L235-L263) 和 [L287](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L287)

**V2 特点**：
- Worker **不做任何编译**，直接使用 bridge 返回的预渲染内容
- 翻译在 **Bridge 或 API 层** 完成
- 翻译语法：`{{t.key}}`

---

## Framework 阶段：Liquid 渲染 + 翻译标记保护

### 核心文件

- [client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/client.ts#L733-L806) — `compileControls()` 方法
- [liquid.utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/utils/liquid.utils.ts) — Liquid 引擎创建

### 流程详解

`compileControls()` 将 step 的 controls 序列化为 JSON 字符串，然后通过三步处理：

1. **预处理 — `preprocessTranslationPatterns()`**
   - `{{t.key}}` → `[T:key]` 占位符
   - 原因：`{{t.key}}` 语法与 Liquid 的 `{{ }}` 冲突
   - 正则：`/\{\{\s*t\.([\p{L}\p{N}_.-]+)\s*\}\}/gu` → `[T:$1]`

2. **预处理 — `preprocessFilterTranslationArgs()`**
   - 处理翻译 key 作为 filter 参数的情况
   - `'t.key'` → `'[T:key]'`
   - 例如：`pluralize: 't.apple', 't.apples'` → `pluralize: '[T:apple]', '[T:apples]'`
   - 正则：`/'t\.([\p{L}\p{N}_.-]+)'/gu` → `'[T:$1]'`

3. **Liquid 渲染**
   - 解析和渲染模板变量：`{{payload.name}}`, `{{subscriber.firstName}}`
   - 可用变量：`payload`, `subscriber`, `context`, `steps`, `workflow`, `env`

4. **后处理 — `postprocessTranslationMarkers()`**
   - `[T:key]` → `{{t.key}}`
   - 正则：`/\[T:([\p{L}\p{N}_.-]+)\]/gu` → `{{t.$1}}`
   - 翻译标记以 `{{t.key}}` 形式保留，交给下一阶段处理

---

## 预览路径 vs Worker 路径对比

| 维度 | API Preview 路径 | Worker 路径 (V1) | Worker 路径 (V2) |
|------|-----------------|------------------|------------------|
| 入口 | `PreviewUsecase` → `OutputRenderer` | `SendMessage*` → `CompileTemplate` | `SendMessage*` → bridge output |
| 翻译模块 | `@novu/ee-translation` | `i18next` + Handlebars i18n helper | Bridge 预渲染 |
| 翻译语法 | `{{t.key}}` | `{{i18n "key"}}` | `{{t.key}}` |
| 模板引擎 | Liquid | Handlebars | Liquid (in Bridge) |
| 执行顺序 | 翻译先，后变量替换 | 翻译 + 变量替换同时 | Bridge 完成 |
| 主要用途 | 编辑器预览 | 实际发送 V1 模板 | 实际发送 V2 Bridge |

---

## 完整端到端流程示例

### 示例 1：V2 工作流 + Email + Preview 路径

```
模板:
  subject: '{{t.email.subject}}'
  body: '<p>{{t.greeting}} {{subscriber.firstName}}! {{t.order_status}}: {{payload.orderId}}</p>'

翻译 (en_US):
  email.subject: 'Order Update'
  greeting: 'Hello'
  order_status: 'Your order status is'

1. Framework compileControls():
   body: '<p>[T:greeting] {{subscriber.firstName}}! [T:order_status]: {{payload.orderId}}</p>'
   ↓ Liquid render (subscriber={firstName: 'Alice'}, payload={orderId: '12345'})
   body: '<p>[T:greeting] Alice! [T:order_status]: 12345</p>'
   ↓ postprocess
   body: '<p>{{t.greeting}} Alice! {{t.order_status}}: 12345</p>'

2. API Preview - EmailOutputRenderer:
   ↓ processStringTranslations() → @novu/ee-translation
   body: '<p>Hello Alice! Your order status is: 12345</p>'
   ↓ Liquid parseAndRender (无剩余变量)
   body: '<p>Hello Alice! Your order status is: 12345</p>'

3. 最终输出:
   subject: 'Order Update'
   body: '<p>Hello Alice! Your order status is: 12345</p>'
```

### 示例 2：翻译内容包含变量

```
翻译:
  personalized: 'Hello {{payload.name}}!'

模板:
  body: '<p>{{t.personalized}}</p>'

1. @novu/ee-translation 翻译:
   body: '<p>Hello {{payload.name}}!</p>'

2. Liquid 变量替换 (payload={name: 'John'}):
   body: '<p>Hello John!</p>'
```

### 示例 3：V1 工作流 + Handlebars + i18n helper

```
模板 (Handlebars):
  subject: '{{i18n "email.subject"}}'
  body: '<p>{{i18n "greeting" name=subscriber.firstName}}! {{i18n "order_status"}}: {{payload.orderId}}</p>'

翻译 (en_US):
  email.subject: 'Order Update'
  greeting: 'Hello {{name}}!'
  order_status: 'Your order status is'

1. Handlebars 编译:
   subject: 'Order Update'
   body: '<p>Hello Alice! Your order status is: 12345</p>'
```

---

## 特殊情况：翻译 key 作为 filter 参数

在 V2 Liquid 模板中，翻译 key 可以作为 filter 的字符串参数：

```handlebars
You have {{payload.appleCount | pluralize: 't.appleSingular', 't.applePlural', 'false'}} and {{payload.itemCount | pluralize: 't.itemSingular', 't.itemPlural', 'false' | append: 't.suffix'}}
```

这里的 `'t.appleSingular'` 是一个 **Liquid filter 的字符串参数**，不是模板变量。翻译 key 的完整生命周期如下：

### 完整四阶段流转

```
原始 control:    {{payload.appleCount | pluralize: 't.appleSingular', 't.applePlural', 'false'}}
                 ↓
阶段 1 预处理:   {{payload.appleCount | pluralize: '[T:appleSingular]', '[T:applePlural]', 'false'}}
                 ↓
阶段 2 Liquid:   pluralize(1, '[T:appleSingular]', '[T:applePlural]', 'false') → '[T:appleSingular]'
                 完整输出: "You have [T:appleSingular]"
                 ↓
阶段 3 后处理:   "You have {{t.appleSingular}}"
                 ↓
阶段 4 翻译:     @novu/ee-translation → "You have apple"
```

### 各阶段详解

**阶段 1 — `preprocessFilterTranslationArgs()`**

代码：[client.ts#L796-L798](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/client.ts#L796-L798)

```typescript
private preprocessFilterTranslationArgs(template: string): string {
  return template.replace(/'t\.([\p{L}\p{N}_.-]+)'/gu, "'[T:$1]'");
}
```

- 匹配单引号包裹的 `'t.key'` 字符串（注意：只匹配 filter 参数中的翻译 key，不匹配独立的 `{{t.key}}`）
- `'t.appleSingular'` → `'[T:appleSingular]'`
- 转换后仍然是合法的 Liquid 字符串参数

**阶段 2 — Liquid 渲染**

`pluralize` filter（[pluralize.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/filters/pluralize.ts)）接收字符串参数并原样返回选中的那个：

```
pluralize(1, '[T:appleSingular]', '[T:applePlural]', 'false')
  → count=1, 选 singular 参数 → 返回字符串 '[T:appleSingular]'
```

关键点：`[T:appleSingular]` 不是 Liquid 语法，Liquid 引擎对它不做任何处理，它作为普通字符串透传。

`append` filter 同理：
```
{{payload.itemCount | pluralize: '[T:itemSingular]', '[T:itemPlural]', 'false' | append: '[T:suffix]'}}
  → pluralize(5, ...) → '[T:itemPlural]'
  → append('[T:suffix]') → '[T:itemPlural][T:suffix]'
```

**阶段 3 — `postprocessTranslationMarkers()`**

代码：[client.ts#L804-L806](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/client.ts#L804-L806)

```typescript
private postprocessTranslationMarkers(content: string): string {
  return content.replace(/\[T:([\p{L}\p{N}_.-]+)\]/gu, '{{t.$1}}');
}
```

- 将所有 `[T:key]` 占位符还原为 `{{t.key}}` 翻译标记
- 无论是独立的 `{{t.key}}` 还是 filter 参数中的 `'t.key'`，最终都统一变成 `{{t.key}}` 形式
- `postprocess` 是**统一**处理 `preprocessTranslationPatterns` 和 `preprocessFilterTranslationArgs` 两步产出的

**阶段 4 — `@novu/ee-translation` 翻译**

`{{t.key}}` 标记由 `@novu/ee-translation` 模块通过正则匹配解析并替换为翻译文本。

### 单元测试证据

[client.test.ts#L1449-L1462](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/client.test.ts#L1449-L1462)：

```typescript
// 输入 controls
controls: {
  body: "You have {{ payload.count | pluralize: 't.apple', 't.apples' }}",
  subject: "{{ payload.count | pluralize: 't.itemSingular', 't.itemPlural' }} in your cart",
}

// Framework 输出 (count=5)
// 注意：输出中仍是 {{t.key}} 标记，不是 [T:key] 占位符
expect(emailExecutionResult.outputs).toEqual({
  body: 'You have 5 {{t.apples}}',
  subject: '5 {{t.itemPlural}} in your cart',
});
```

这证明了：
1. `[T:key]` 只是 Liquid 渲染期间的**临时中间态**，Framework 输出时已经还原为 `{{t.key}}`
2. `{{t.key}}` 才是交给下一阶段（`@novu/ee-translation`）的**真正翻译标记**

### E2E 测试证据

[translation-replacement.e2e-ee.ts#L400-L437](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/translations/e2e/v2/translation-replacement.e2e-ee.ts#L400-L437)：

```typescript
// 翻译内容
content: {
  appleSingular: 'apple',
  applePlural: 'apples',
  itemSingular: 'item',
  itemPlural: 'items',
  suffix: ' in cart',
}

// 输入 controls (包含 pluralize + append 两种 filter)
body: "You have {{payload.appleCount | pluralize: 't.appleSingular', 't.applePlural', 'false'}} and {{payload.itemCount | pluralize: 't.itemSingular', 't.itemPlural', 'false' | append: 't.suffix'}}"

// payload: { appleCount: 1, itemCount: 5 }

// 最终结果
expect(preview.body).to.include('You have apple and items in cart');
```

完整流转：
```
'You have {{payload.appleCount | pluralize: 't.appleSingular', 't.applePlural', 'false'}} ...'
  ↓ preprocessFilterTranslationArgs
'You have {{payload.appleCount | pluralize: '[T:appleSingular]', '[T:applePlural]', 'false'}} ...'
  ↓ Liquid render (appleCount=1, itemCount=5)
'You have [T:appleSingular] and [T:itemPlural][T:suffix]'
  ↓ postprocessTranslationMarkers
'You have {{t.appleSingular}} and {{t.itemPlural}}{{t.suffix}}'
  ↓ @novu/ee-translation
'You have apple and items in cart'
```

---

### Layout 中的 filter 参数翻译 key — 跳过 Framework 的捷径

Layout 内容直接从数据库获取，**不经过 Framework 的 `compileControls()` 流程**，因此缺少 `preprocessFilterTranslationArgs` 和 `postprocessTranslationMarkers` 两个步骤。

代码：[email-output-renderer.usecase.ts#L390-L400](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L390-L400)

```typescript
const layoutBody = (layoutControlValues.email?.body ?? '').replace(/'t\.([\p{L}\p{N}_.-]+)'/gu, "'{{t.$1}}'");
```

这条替换将 `'t.key'` **直接**转为 `'{{t.key}}'`，跳过了 `[T:key]` 中间态。

为什么可以跳过？因为在 Layout 场景下，layout body 是纯 HTML/Liquid 文本，`'{{t.key}}'` 作为 Liquid 的字符串参数会原样透传（Liquid 不会解析引号内的 `{{ }}`），效果等同于 Framework 中 `[T:key]` 透传 Liquid 后再 `postprocess` 还原为 `{{t.key}}`。

```
Framework 路径 (4步):
  't.key' → '[T:key]' → Liquid透传 → '{{t.key}}' → 翻译

Layout 捷径 (2步):
  't.key' → '{{t.key}}' → Liquid透传 → 翻译
```

两种路径的最终效果一致：`{{t.key}}` 标记到达 `@novu/ee-translation` 进行翻译。

---

### 总结：`'t.key'` 的三种形态和它们的意义

| 形态 | 出现阶段 | 作用 |
|------|----------|------|
| `'t.key'` | 用户在模板中书写 | 原始形式，Liquid 会将其视为普通字符串（无法被翻译） |
| `[T:key]` | Framework 预处理后、Liquid 渲染期间 | **临时逃逸占位符**，保护翻译 key 不被 Liquid 破坏的同时透传到输出 |
| `{{t.key}}` | Framework 后处理后、进入翻译服务 | **最终翻译标记**，`@novu/ee-translation` 识别并替换为翻译文本 |

---

## Enterprise 特性门控

### API Preview 路径

[BaseTranslationRendererUsecase](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/base-translation-renderer.usecase.ts#L43-L45):

```typescript
if (process.env.NOVU_ENTERPRISE !== 'true' && process.env.CI_EE_TEST !== 'true') {
  return controls;  // 不做翻译，直接返回
}
```

### Worker 路径

[send-message.base.ts#L191-L193](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts#L191-L193):

```typescript
if (process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true') {
  // 创建 i18nInstance
}
```

### 翻译模块加载

[BaseTranslationRendererUsecase.getTranslationModule()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/base-translation-renderer.usecase.ts#L261-L277):

```typescript
const translationModule = require('@novu/ee-translation')?.Translate;
```

`@novu/ee-translation` 是 Enterprise 专属模块，Community 版本不存在。

---

## 数据模型

### 翻译实体

- [TranslationEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/translations/translation.entity.ts) — 按语言存储翻译内容
- [TranslationGroupEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/translation-group/translation-group.entity.ts) — 翻译分组
- [LocalizationEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/localization/localization.entity.ts) — 本地化内容

### Locale 支持

- [locale-registry.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/utils/locales/locale-registry.ts) — 支持的 locale 列表
- [locale-validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/utils/locales/locale-validator.ts) — locale 格式校验
- 默认 locale: `en_US` — [translation.constants.ts#L4](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/translation/translation.constants.ts#L4)

### Subscriber locale

- Subscriber 实体的 `locale` 字段
- 在 `initiateTranslations()` 中作为 i18next 的 `lng` 参数
- 若未设置，回退到 `'en'`

---

## 关键常量

[translation.constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/translation/translation.constants.ts):

| 常量 | 值 | 用途 |
|------|----|------|
| `DEFAULT_LOCALE` | `'en_US'` | 默认回退语言 |
| `TRANSLATION_NAMESPACE_SEPARATOR` | `'t.'` | 翻译 key 前缀 |
| `TRANSLATION_KEY_SINGLE_REGEX` | `/\{\{\s*t\.([^}]+?)\s*\}\}/i` | 匹配 `{{t.key}}` |
| `TRANSLATION_TRIGGER_CHARACTER` | `'{{t.'` | 翻译标记起始字符 |
| `MISSING_TRANSLATION_TEMPLATE` | `[Translation missing: ${key}]` | 翻译缺失占位符 |

---

## 常见误区澄清

### ❌ 错误：`{{t.key}}` 会被转换为 `{{i18n "key"}}`

**✅ 正确**：它们是两种独立的语法，用于不同的系统：
- `{{t.key}}` → V2 Bridge 工作流 + Liquid + `@novu/ee-translation`
- `{{i18n "key"}}` → V1 模板工作流 + Handlebars + i18n helper

### ❌ 错误：变量替换先于翻译

**✅ 正确**：在 API Preview 路径中，**翻译先于变量替换**。翻译内容中可以包含 Liquid 变量，翻译完成后再进行变量替换。

### ❌ 错误：Worker 路径总是使用 Handlebars 编译

**✅ 正确**：V2 工作流的 Worker 路径直接使用 bridge 输出，**不做任何编译**。只有 V1 工作流才使用 `CompileTemplate` 进行 Handlebars 编译。

### ❌ 错误：`{{t.key}}` 是 Handlebars 语法

**✅ 正确**：`{{t.key}}` 不是 Handlebars 语法，而是 `@novu/ee-translation` 模块通过正则识别的自定义标记。

### ❌ 错误：`[T:key]` 占位符是最终翻译目标

**✅ 正确**：`[T:key]` 只是 Liquid 渲染期间的**临时逃逸占位符**，用于让翻译 key 安全穿越 Liquid 引擎。Framework 输出时已经通过 `postprocessTranslationMarkers()` 将其还原为 `{{t.key}}`。`{{t.key}}` 才是交给 `@novu/ee-translation` 翻译服务的最终标记。

### ❌ 错误：`'t.key'` filter 参数直接被翻译

**✅ 正确**：`'t.key'` 作为 Liquid filter 的字符串参数，Liquid 引擎将其视为普通字符串不会触发翻译。完整路径是：`'t.key'` → `preprocessFilterTranslationArgs()` 转为 `'[T:key]'` → Liquid 透传 → `postprocessTranslationMarkers()` 还原为 `{{t.key}}` → `@novu/ee-translation` 翻译为最终文本。

### ❌ 错误：Maily `wrapMailyInLiquid` 会把所有属性都包裹成 Liquid

**✅ 正确**：`wrapMailyInLiquid` 只按 `variableAttributeConfig(type)` 返回的白名单包裹属性，且还需 `flag` 对应的属性值为 truthy。例如 `button.text` 仅当 `isTextVariable=true` 时才会被包裹，`image.src` 仅当 `isSrcVariable=true` 时才会被包裹。

### ❌ 错误：所有节点上的翻译 key 都能豁免 Liquid 包裹

**✅ 正确**：**翻译 key 豁免规则仅适用于 `button.attrs.text`（且需 `isTextVariable=true`）**。其他所有属性 — 包括 `variable` 节点的 `id`、`button.url`、`image.src`、`link.href` — 即便值是翻译 key，仍然会被包裹成 `{{ t.xxx }}`。例如 `{type:"variable", attrs:{id:"t.greeting"}}` 会被包裹为 `{type:"variable", attrs:{id:"{{ t.greeting }}"}}`。

### ❌ 错误：Maily `variable` 节点的 id 会被翻译服务直接识别

**✅ 正确**：`variable` 节点的 `id` 不是文本节点，翻译服务在 JSON 字符串中搜索 `{{t.key}}` / `{{ t.key }}`。`variable` 节点先在阶段 1 被 `wrapMailyInLiquid` 包裹为 `{{ t.xxx }}`，再在阶段 2 被 `processVariableNodeTypes` 从 `attrs.id` 搬到 `text` 字段，此时 JSON 字符串中才出现 `"text":"{{ t.xxx }}"`，翻译服务才能在阶段 3 匹配并替换。

### ❌ 错误：所有路径只执行一次 Liquid 变量替换

**✅ 正确**：不同路径 Liquid 执行次数不同：
- SMS/In-App/Push/Chat：**1 次**（翻译服务内部）
- Email Subject：**1 次**（翻译服务内部）
- Email 纯文本 Body：**2 次**（翻译服务内部 + Renderer 自有 liquidEngine）
- Email Maily JSON：**2 次**（翻译服务内部阶段 3 + Renderer 自有 liquidEngine 阶段 4）

Renderer 自有 `this.liquidEngine` 额外执行的原因是其 `outputEscape` 对对象/数组做了特殊处理（`JSON.stringify` + 双引号转单引号 + 换行转义），保证 Maily JSON 和 Email 场景中嵌入的复杂变量能被正确 `JSON.parse`。
