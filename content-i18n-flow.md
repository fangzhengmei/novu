# Content Engine 模板渲染与 i18n 本地化衔接流程

## 概览

Novu 的通知内容渲染分为**两大阶段**，涉及**两套模板引擎**和**一套 i18n 翻译机制**：

1. **Framework 阶段（Node.js SDK 侧）**：用 **Liquid** 引擎渲染用户在代码中定义的 `{{payload.xxx}}`、`{{subscriber.firstName}}` 等变量，同时保护 `{{t.key}}` 翻译标记不被 Liquid 吃掉。
2. **Worker 阶段（平台侧）**：用 **Handlebars** 引擎做最终渲染，此时注册了 `i18n` helper，将 `{{t.key}}` 翻译为对应 locale 的文本。

```
用户代码 (Framework/Liquid)
  ↓  预处理: {{t.key}} → [T:key]
  ↓  Liquid 渲染: {{payload.name}} → "Alice"
  ↓  后处理: [T:key] → {{t.key}}
  ↓
Worker 接收 controls
  ↓  Handlebars 渲染: {{t.key}} → "你好" (根据 subscriber.locale)
  ↓  输出最终通知内容
```

---

## 第一阶段：Framework 侧 — Liquid 渲染 + 翻译标记保护

### 核心文件

- [client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/client.ts#L733-L806) — `compileControls()` 方法
- [liquid.utils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/framework/src/utils/liquid.utils.ts) — Liquid 引擎创建

### 流程详解

1. **`compileControls()`** 将 step 的 controls 序列化为 JSON 字符串，然后通过三步处理：

2. **预处理 — `preprocessTranslationPatterns()`**
   - 将 `{{t.key}}` 转换为 `[T:key]` 占位符
   - 原因：`{{t.key}}` 语法与 Liquid 的 `{{ }}` 冲突，Liquid 会尝试解析它但找不到变量导致报错
   - 正则：`/\{\{\s*t\.([\p{L}\p{N}_.-]+)\s*\}\}/gu` → `[T:$1]`

3. **预处理 — `preprocessFilterTranslationArgs()`**
   - 处理翻译 key 作为 helper 参数的情况，如 `pluralize: 't.apple', 't.apples'`
   - 将 `'t.key'` 转换为 `'[T:key]'`
   - 正则：`/'t\.([\p{L}\p{N}_.-]+)'/gu` → `'[T:$1]'`

4. **Liquid 渲染**
   - 使用 `Liquid` 引擎解析和渲染模板
   - 可用变量：`payload`、`subscriber`、`context`、`steps`、`workflow`、`env`
   - Liquid 的自定义 filter：`json`、`digest`、`toSentence`、`pluralize`

5. **后处理 — `postprocessTranslationMarkers()`**
   - 将 `[T:key]` 还原为 `{{t.key}}`
   - 正则：`/\[T:([\p{L}\p{N}_.-]+)\]/gu` → `{{t.$1}}`
   - 这样翻译标记以 Handlebars 语法的形式保留下来，交给 Worker 阶段处理

### 关键设计意图

`{{t.key}}` 不是普通的变量替换，而是一个**翻译标记**。它在 Framework 阶段不需要被解析（因为此时还不知道 subscriber 的 locale），只需要被**保护**不被 Liquid 破坏，然后原样传递给 Worker。

---

## 第二阶段：Worker 侧 — Handlebars 渲染 + i18n 翻译

### 核心文件

- [send-message.base.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.base.ts#L189-L228) — `initiateTranslations()` 创建 i18next 实例
- [compile-template.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/usecases/compile-template/compile-template.usecase.ts#L13-L150) — `createHandlebarsInstance()` 注册 i18n helper
- [compile-step-template.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/usecases/compile-step-template/compile-step-template.usecase.ts) — Step 模板编译
- [compile-email-template.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/usecases/compile-email-template/compile-email-template.usecase.ts) — Email 模板编译
- [compile-in-app-template.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/application-generic/src/usecases/compile-in-app-template/compile-in-app-template.usecase.ts) — In-App 模板编译

### 流程详解

#### 1. 创建 i18next 实例 — `initiateTranslations()`

```
位置: SendMessageBase.initiateTranslations()
触发: 每个 SendMessage* 子类在发送前调用
```

- 仅在 **Enterprise 版本** (`NOVU_ENTERPRISE=true`) 下启用
- 通过 `TRANSLATIONS_SERVICE`（一个 NestJS 动态注入的 Enterprise 服务）获取翻译数据
- 调用 `service.getTranslationsList(environmentId, organizationId)` 获取：
  - `namespaces` — 翻译命名空间列表
  - `resources` — 各语言翻译资源
  - `defaultLocale` — 默认语言
- 创建 `i18next.createInstance()`：
  - `lng` 来自 `subscriber.locale`，默认 `'en'`
  - `fallbackLng` 来自 `defaultLocale`，默认 `'en'`
  - 支持日期格式化插值

#### 2. Handlebars i18n Helper 注册

```
位置: CompileTemplate → createHandlebarsInstance()
```

当 `i18nInstance` 存在时，注册 Handlebars `i18n` helper：

```handlebars
{{i18n "greeting" name=user.firstName}}
```

helper 内部逻辑：
1. 从 `data.root.i18next` 获取 i18next 配置
2. 合并 hash 参数作为翻译选项
3. 将当前 Handlebars 上下文 (`this`) 中的变量作为 `replace` 插值参数传给 i18next
4. 调用 `i18next.t(key, options)` 获取翻译文本
5. 用 `SafeString` 包裹返回，避免 HTML 转义

#### 3. 模板编译流程

各 channel 的编译 use case 都遵循同一模式：

```
SendMessageBase.initiateTranslations(envId, orgId, subscriber.locale)
  → i18nInstance
    → CompileStepTemplate.execute(command, i18nInstance)
      → CompileTemplate.execute({ template, data }, i18nInstance)
        → createHandlebarsInstance(i18nInstance)
          → handlebars.compile(template)(data)
```

以 Email 为例的完整调用链：

1. [SendMessageEmail.execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts#L229-L244) 调用 `initiateTranslations()` 获取 `i18nInstance`
2. 调用 `CompileEmailTemplate.execute(command, i18nInstance)`
3. Email 内部对 `subject`、`preheader`、`senderName`、每个 block 的 `content` 和 `url` 分别调用 `CompileTemplate.execute({ template, data }, i18nInstance)`
4. 最终将 body 放入 layout 做第二次编译

---

## 两套模板引擎对比

| 维度 | Framework (Liquid) | Worker (Handlebars) |
|------|-------------------|---------------------|
| 运行位置 | 用户 Node.js 进程 | Novu Worker 服务 |
| 模板引擎 | LiquidJS | Handlebars |
| 变量来源 | `payload`、`subscriber`、`steps` 等 | trigger payload + 系统变量 |
| 翻译处理 | **保护** `{{t.key}}` 不被解析 | **解析** `{{t.key}}` / `{{i18n "key"}}` |
| 输出 | 序列化的 JSON controls | 最终通知内容（HTML/文本） |

---

## 翻译标记语法

### `{{t.key}}` 语法（隐式翻译）

这是最常用的翻译标记形式，直接在模板文本中嵌入：

```handlebars
Hello {{subscriber.firstName}}, {{t.welcome_message}}
```

- 通过 `TRANSLATION_KEY_SINGLE_REGEX` 正则识别：`/\{\{\s*t\.([^}]+?)\s*\}\}/i`
- 在 Framework 阶段被保护为 `[T:key]`，Liquid 渲染后还原
- 在 Worker 阶段被 Handlebars 识别并翻译

### `{{i18n "key" ...}}` 语法（显式翻译）

显式调用 i18n helper，支持传参：

```handlebars
{{i18n "greeting" name=subscriber.firstName count=items.length}}
```

- 仅在 Worker 阶段的 Handlebars 中可用
- 支持 hash 参数作为翻译插值变量
- 支持 block 形式提供 `defaultValue`

### `{{t.key}}` 作为 helper 参数

翻译 key 可以作为其他 helper 的参数：

```handlebars
{{pluralize count (i18n "apple") (i18n "apples")}}
```

在 Framework 阶段，`'t.apple'` 形式的参数会被预处理为 `'[T:apple]'`。

---

## 数据模型

### 翻译数据存储

- [TranslationEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/translations/translation.entity.ts) — 翻译条目，按语言 (`isoLanguage`) 存储 JSON 翻译内容
- [TranslationGroupEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/translation-group/translation-group.entity.ts) — 翻译分组，按环境和组织隔离
- [LocalizationEntity](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/localization/localization.entity.ts) — 本地化内容存储

### Locale 支持

- [locale-registry.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/utils/locales/locale-registry.ts) — `SUPPORTED_LOCALES` 全量 locale 列表及查询方法
- [locale-validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/utils/locales/locale-validator.ts) — locale 格式校验（`en-US` → `en_US` 归一化）
- 默认 locale: `en_US`（定义在 [translation.constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/translation/translation.constants.ts#L4)）

### Subscriber locale 来源

- [subscriber.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/libs/dal/src/repositories/subscriber/subscriber.entity.ts) 的 `locale` 字段
- 在 `initiateTranslations()` 中作为 `i18next` 的 `lng` 参数
- 若 subscriber 未设置 locale，回退到 `'en'`

---

## 变量替换与模板变量提取

### Handlebars 变量提取

- [content.engine.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/stateless/src/lib/content/content.engine.ts#L42-L53) — `getHandlebarsVariables()` 通过解析 Handlebars AST 提取变量名
- [getTemplateVariables.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/handlebar-helpers/getTemplateVariables.ts) — 更完整的变量提取，处理：
  - `MustacheStatement` — 简单变量 `{{name}}` 和 helper 调用 `{{i18n "key"}}`
  - `BlockStatement` — `{{#each}}` / `{{#with}}` / `{{#if}}` / `{{#unless}}`
  - `HashPair` — helper 的 hash 参数中的变量
  - **i18n helper 特殊处理**：当 `body.path.original === 'i18n'` 时，递归提取 hash pairs 中的变量

### 注册的 Handlebars Helpers

定义在 [handlebarHelpers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/handlebar-helpers/handlebarHelpers.ts)：

| Helper | 用途 |
|--------|------|
| `i18n` | 翻译 |
| `equals` | 相等判断 |
| `titlecase` | 首字母大写 |
| `uppercase` / `lowercase` | 大小写转换 |
| `pluralize` | 单复数 |
| `dateFormat` | 日期格式化 |
| `numberFormat` | 数字格式化 |
| `unique` / `groupBy` / `sortBy` | 数组操作 |
| `gt` / `gte` / `lt` / `lte` / `eq` / `ne` | 比较运算 |

---

## 完整端到端流程示例

以一个带翻译的 Email 通知为例：

```
1. 用户代码定义 (Framework):
   subject: "Order Update"
   body: "Hi {{subscriber.firstName}}, {{t.order_status_update}}: {{payload.orderId}}"

2. compileControls() 预处理:
   subject: "Order Update"
   body: "Hi {{subscriber.firstName}}, [T:order_status_update]: {{payload.orderId}}"

3. Liquid 渲染 (payload = {orderId: "12345"}, subscriber = {firstName: "Alice"}):
   subject: "Order Update"
   body: "Hi Alice, [T:order_status_update]: 12345"

4. postprocessTranslationMarkers():
   subject: "Order Update"
   body: "Hi Alice, {{t.order_status_update}}: 12345"

5. Worker 接收 controls，subscriber.locale = "zh_CN"

6. initiateTranslations() 创建 i18next 实例 (lng = "zh_CN")

7. CompileEmailTemplate 调用 CompileTemplate:
   - Handlebars 注册 i18n helper
   - 渲染 body: "Hi Alice, 订单状态更新: 12345"
   - 渲染 subject: "Order Update" (无翻译 key，原样输出)

8. 最终发送 HTML 邮件
```

---

## Enterprise 特性门控

翻译功能是 **Enterprise 专属**：

- `initiateTranslations()` 中检查 `process.env.NOVU_ENTERPRISE === 'true'`
- `TRANSLATIONS_SERVICE` 通过 NestJS 动态模块注入，仅在 Enterprise 版本中注册
- Community 版本中 `i18nInstance` 为 `undefined`，`{{t.key}}` 不会被翻译，保留为原始 Handlebars 变量（输出为空字符串）

---

## 关键常量

定义在 [translation.constants.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/61-novu/packages/shared/src/consts/translation/translation.constants.ts)：

| 常量 | 值 | 用途 |
|------|----|------|
| `DEFAULT_LOCALE` | `'en_US'` | 默认回退语言 |
| `TRANSLATION_NAMESPACE_SEPARATOR` | `'t.'` | 翻译 key 前缀 |
| `TRANSLATION_KEY_SINGLE_REGEX` | `/\{\{\s*t\.([^}]+?)\s*\}\}/i` | 匹配单个 `{{t.key}}` |
| `TRANSLATION_TRIGGER_CHARACTER` | `'{{t.'` | 翻译标记起始字符 |
| `MISSING_TRANSLATION_TEMPLATE` | `[Translation missing: ${key}]` | 翻译缺失时的占位文本 |
