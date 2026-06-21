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

## 关键执行顺序：翻译先于变量替换

**在 API Preview 路径（Output Renderers）中，执行顺序是：**

1. **翻译替换** — `{{t.key}}` → 翻译后的文本（可能包含 `{{payload.xxx}}` 变量）
2. **变量替换** — 翻译后的文本中的 `{{payload.xxx}}` → 实际变量值

**代码证据**：[email-output-renderer.usecase.ts#L572-L614](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts#L572-L614)

```typescript
// processTextTranslations() 方法
private async processTextTranslations({...}): Promise<string> {
  // Step 1: 翻译替换
  const translatedText = translationContext
    ? await this.processStringWithContext({...})  // 调用 @novu/ee-translation
    : await this.processStringTranslations({...});

  // Step 2: 变量替换（Liquid）
  const unescapedTranslatedText = this.unescapeJsonString(translatedText);
  return await this.liquidEngine.parseAndRender(unescapedTranslatedText, variables);
}
```

**E2E 测试证据**：[translation-replacement.e2e-ee.ts#L299-L329](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/translations/e2e/v2/translation-replacement.e2e-ee.ts#L299-L329)

```typescript
// 翻译内容中包含 Liquid 变量
content: {
  personalized: 'Hello {{payload.name}}!',
}

// 模板中使用
body: '<p>{{t.personalized}}</p>'

// 结果: Hello John!
// 流程: {{t.personalized}} → Hello {{payload.name}}! → Hello John!
```

---

## API Preview 路径：Output Renderers

### 核心文件

| Channel | Renderer Usecase |
|---------|------------------|
| Email | [email-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/email-output-renderer.usecase.ts) |
| In-App | [in-app-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/in-app-output-renderer.usecase.ts) |
| SMS | [sms-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/sms-output-renderer.usecase.ts) |
| Push | [push-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/push-output-renderer.usecase.ts) |
| Chat | [chat-output-renderer.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/chat-output-renderer.usecase.ts) |

### 基类：BaseTranslationRendererUsecase

所有 Output Renderer 都继承自 [BaseTranslationRendererUsecase](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/api/src/app/environments-v1/usecases/output-renderers/base-translation-renderer.usecase.ts)，提供统一的翻译处理逻辑。

### 调用链

```
PreviewUsecase.execute()
  ↓
PreviewStep.execute()
  ↓
ExecuteBridgeRequest.execute() → 调用 Novu Bridge
  ↓
[ API 返回 bridge outputs ]
  ↓
Output Renderer (e.g. EmailOutputRendererUsecase)
  ↓
processTranslations() → @novu/ee-translation.execute()
  ↓
  ├─ createTranslationContext() → 从 @novu/ee-translation 创建上下文
  ├─ processStringTranslations() → 翻译 {{t.key}}
  └─ liquidEngine.parseAndRender() → 变量替换 {{payload.xxx}}
```

### 关键方法

| 方法 | 作用 |
|------|------|
| `processTranslations()` | 处理整个 controls 对象的翻译 |
| `processStringTranslations()` | 处理单个字符串的翻译 |
| `createTranslationContext()` | 创建翻译上下文（缓存 i18n 实例） |
| `processStringWithContext()` | 使用缓存上下文处理翻译（性能优化） |

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
