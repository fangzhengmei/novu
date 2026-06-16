# Novu 订阅偏好与全局静默规则优先级合并分析

## 一、偏好来源分层（4 层 + 1 专项）

Novu 的偏好体系由 4 种类型（`PreferencesTypeEnum`）组成，按**具体度从高到低**排列：

| 优先级 | 枚举值 | 含义 | 存储位置 | 设定者 |
|--------|--------|------|----------|--------|
| 1（最高） | `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` | 订阅范围内的工作流偏好 | preferences 集合 | 订阅 API |
| 2 | `SUBSCRIBER_WORKFLOW` | 订阅者对某工作流的偏好 | preferences 集合 | 终端用户在偏好面板 |
| 3 | `SUBSCRIBER_GLOBAL` | 订阅者全局偏好（含勿扰日程） | preferences 集合 | 终端用户在偏好面板 |
| 4 | `USER_WORKFLOW` | Dashboard 用户对工作流的偏好 | preferences 集合 | 管理员在 Dashboard |
| 5（最低） | `WORKFLOW_RESOURCE` | Framework 代码定义的工作流默认偏好 | preferences 集合 | 开发者在代码中定义 |

> 枚举定义见 [workflow-channel-preferences.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/types/workflow-channel-preferences.ts#L15-L21)

### 每一层的结构

```typescript
// WorkflowPreferences — 每层偏好都遵循此结构
{
  all: {
    enabled: boolean,   // 工作流级别开关，默认 true
    readOnly: boolean,  // 是否只读（即 "critical"），默认 false
    condition?: any,    // JSON Logic 条件表达式
  },
  channels: {
    in_app: { enabled: boolean },
    email:   { enabled: boolean },
    sms:     { enabled: boolean },
    push:    { enabled: boolean },
    chat:    { enabled: boolean },
  }
}
```

> 类型定义见 [WorkflowPreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/types/workflow-channel-preferences.ts#L70-L83)

### 默认值

```typescript
// preferences.const.ts
PREFERENCE_DEFAULT_VALUE = true    // all.enabled 默认值
PREFERENCE_DEFAULT_READ_ONLY = false // all.readOnly 默认值
WORKFLOW_PREFERENCE_DEFAULT = { enabled: true, readOnly: false }
CHANNEL_PREFERENCE_DEFAULT = { enabled: true }

DEFAULT_WORKFLOW_PREFERENCES = {
  all: { enabled: true, readOnly: false },
  channels: {
    in_app: { enabled: true },
    sms:    { enabled: true },
    email:  { enabled: true },
    push:   { enabled: true },
    chat:   { enabled: true },
  }
}
```

> 常量定义见 [preferences.const.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/consts/preferences/preferences.const.ts#L1-L24)

---

## 二、合并核心逻辑 — `MergePreferences`

核心合并发生在 [merge-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.usecase.ts#L58-L98)。

### 2.1 输入

`MergePreferencesCommand` 接受 4 个可选偏好实体 + 1 个排除标志：

```typescript
// merge-preferences.command.ts
workflowResourcePreference?       // WORKFLOW_RESOURCE
workflowUserPreference?           // USER_WORKFLOW
subscriberGlobalPreference?      // SUBSCRIBER_GLOBAL
subscriberWorkflowPreference?    // SUBSCRIBER_WORKFLOW
excludeSubscriberPreferences?     // 是否排除订阅者偏好（默认 false）
```

> 命令定义见 [merge-preferences.command.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.command.ts#L4-L15)

### 2.2 合并算法

```
步骤 1: 分组
  workflowPreferences  = [workflowResourcePreference, workflowUserPreference] （过滤掉 undefined）
  subscriberPreferences = [subscriberGlobalPreference, subscriberWorkflowPreference] （过滤掉 undefined）

步骤 2: 判断是否排除订阅者偏好
  isWorkflowPreferenceReadonly = workflowPreferences 中任一 all.readOnly === true
  shouldExcludeSubscriberPreferences = excludeSubscriberPreferences || isWorkflowPreferenceReadonly

步骤 3: 构建合并列表
  preferencesList = [...workflowPreferences, ...(shouldExcludeSubscriberPreferences ? [] : subscriberPreferences)]

步骤 4: 归一化 — 确保每个偏好的 all.enabled 默认为 true
  如果 preference.preferences.all 存在但 all.enabled 为 undefined，则补全为 true

步骤 5: 深度合并（从左到右，后者覆盖前者）
  mergedPreferences = preferencesList.reduce((acc, pref) => toMerged(acc, pref), {})

步骤 6: 产出结果
  return { preferences, schedule, type, source }
  — source 保留各层原始偏好，供 UI 展示覆盖来源
```

### 2.3 关键合并规则总结

#### 规则 A：工作流偏好 → 订阅者偏好，后定义优先

合并列表的顺序为：
```
WORKFLOW_RESOURCE → USER_WORKFLOW → SUBSCRIBER_GLOBAL → SUBSCRIBER_WORKFLOW
```
右侧（更具体）覆盖左侧（更通用），使用 `es-toolkit/toMerged` 做深度合并。

**示例**：
- `WORKFLOW_RESOURCE` 设置 `email.enabled = false`
- `SUBSCRIBER_WORKFLOW` 设置 `email.enabled = true`
- 合并结果：`email.enabled = true`（订阅者偏好胜出）

#### 规则 B：`readOnly = true`（Critical）完全屏蔽订阅者偏好

当任一工作流层偏好（WORKFLOW_RESOURCE 或 USER_WORKFLOW）设置了 `all.readOnly = true`：
- **所有订阅者偏好被排除在合并之外**
- 最终结果 = 仅合并工作流层偏好
- 这就是 "critical" 通知的含义：管理员声明该通知不可被用户关闭

**测试验证**（来自 [merge-preferences.spec.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.spec.ts#L110-L222)）：

| 场景 | readOnly | 结果 |
|------|----------|------|
| WORKFLOW_RESOURCE + SUBSCRIBER_WORKFLOW | false | SUBSCRIBER_WORKFLOW 胜出 |
| WORKFLOW_RESOURCE + SUBSCRIBER_WORKFLOW | true  | WORKFLOW_RESOURCE 胜出（订阅者被排除） |
| WORKFLOW_RESOURCE + USER_WORKFLOW + SUBSCRIBER_GLOBAL + SUBSCRIBER_WORKFLOW | true  | USER_WORKFLOW 胜出（所有订阅者被排除） |

#### 规则 C：`excludeSubscriberPreferences` 标志

当 `excludeSubscriberPreferences = true` 时，同样排除订阅者偏好。此标志用于"提取订阅偏好"场景，此时只需了解工作流层定义了什么，而不考虑用户选择。

#### 规则 D：`all.enabled` 的默认值为 `true`

[ensureDefaultAllEnabled](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.usecase.ts#L37-L56) 确保如果某层偏好的 `all` 对象存在但 `enabled` 为 `undefined`，会补全为 `true`。避免合并时因 `undefined` 被误解析为 `false`。

---

## 三、`buildWorkflowPreferences` — 偏好结构的内部填充

[buildWorkflowPreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/utils/buildWorkflowPreferences.ts#L11-L46) 在将部分偏好（`WorkflowPreferencesPartial`）转为完整偏好时，遵循：

1. **渠道级偏好优先**：如果渠道上直接设定了 `enabled`，使用渠道值
2. **工作流级偏好向下继承**：如果渠道上未设定 `enabled`，但 `all.enabled` 有值，则渠道继承 `all.enabled`
3. **最终回退默认**：都没有时，使用 `DEFAULT_WORKFLOW_PREFERENCES`

```
渠道 enabled ← 渠道自身设定 || all.enabled 设定 || 默认 true
```

---

## 四、勿扰窗口（Schedule / Do-Not-Disturb）

### 4.1 数据结构

`Schedule` 定义在 [workflow-channel-preferences.ts#L117-L120](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/types/workflow-channel-preferences.ts#L117-L120)：

```typescript
type Schedule = {
  isEnabled: boolean;
  weeklySchedule?: {
    monday?:    DaySchedule;
    tuesday?:   DaySchedule;
    wednesday?: DaySchedule;
    thursday?:  DaySchedule;
    friday?:    DaySchedule;
    saturday?:  DaySchedule;
    sunday?:    DaySchedule;
  };
};

type DaySchedule = {
  isEnabled: boolean;
  hours?: Array<{ start: string; end: string }>; // e.g. "09:00 AM" - "05:00 PM"
};
```

**Schedule 仅存储在 `SUBSCRIBER_GLOBAL` 偏好实体上**，是用户全局级的设定。

### 4.2 合并时 Schedule 的传递

在 `MergePreferences.execute` 中，`schedule` 从合并后的实体上透传：

```typescript
return {
  preferences: mergedPreferences.preferences,
  schedule: mergedPreferences.schedule,  // ← 由最后一个有 schedule 的实体决定
  type: mergedPreferences.type,
  source,
};
```

由于 `SUBSCRIBER_GLOBAL` 是唯一携带 `schedule` 的层，且它在合并列表中排在 `SUBSCRIBER_WORKFLOW` 之前，所以如果 `SUBSCRIBER_WORKFLOW` 不携带 schedule，则 schedule 来自 `SUBSCRIBER_GLOBAL`。

### 4.3 运行时 Schedule 检查（Worker 侧）

在 [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L176-L218) 中，Schedule 检查**独立于偏好合并**进行：

```
1. 获取 subscriber 的 Schedule（通过 GetSubscriberSchedule 从 SUBSCRIBER_GLOBAL 偏好读取）
2. 获取 subscriber 的 timezone
3. 判断 isOutsideSubscriberSchedule = schedule.isEnabled && !isWithinSchedule(schedule, now, timezone)
4. 如果在勿扰时段内：
   a. 先尝试 extendToSubscriberSchedule（仅对 delay/digest 类型且非 critical 的 job）
      — 将 job 延迟到下一个可用时间（最多 3 次延期）
   b. 如果不能延期，且非 in-app/critical 类型 → 取消 job
   c. in-app 消息和 critical 通知始终跳过 Schedule 检查
```

### 4.4 `isWithinSchedule` 算法

[schedule-validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/schedule-validator.ts#L15-L67)：

1. 如果没有 schedule 或 `schedule.isEnabled = false` → 允许所有消息
2. 将当前时间转换到 subscriber 的时区
3. 查询当天对应的 `DaySchedule`
4. 还要检查前一天是否有跨夜时段（如 11:00 PM - 02:00 AM）
5. 判断当前时间是否落在任何一个配置的时间范围内

### 4.5 Schedule 与 Critical 的交互

```typescript
// run-job.usecase.ts
private shouldSkipScheduleCheck(job: JobEntity, critical: boolean | undefined): boolean {
  return (
    job.type === StepTypeEnum.TRIGGER ||
    job.type === StepTypeEnum.IN_APP ||    // in-app 始终不受 Schedule 限制
    job.type === StepTypeEnum.DELAY ||
    job.type === StepTypeEnum.DIGEST ||
    job.type === StepTypeEnum.HTTP_REQUEST ||
    critical                             // critical 通知跳过 Schedule 检查
  );
}
```

**结论**：Critical 通知（`readOnly = true`）不仅在偏好合并中屏蔽用户选择，还在 Schedule 检查中直接跳过勿扰窗口。

---

## 五、完整优先级决策流程

当一条通知触发后，决策路径如下：

```
┌─────────────────────────────────────────────────────┐
│ Step 1: MergePreferences 合并偏好                     │
│                                                      │
│  WORKFLOW_RESOURCE ─┐                               │
│  USER_WORKFLOW     ─┤── 深度合并 ──→ workflowResult   │
│                      │                               │
│  SUBSCRIBER_GLOBAL ──┤                               │
│  SUBSCRIBER_WORKFLOW─┘── 深度合并 ──→ subscriberResult │
│                                                      │
│  如果 readOnly=true 或 excludeSubscriber=true:        │
│    finalPreferences = workflowResult                  │
│  否则:                                               │
│    finalPreferences = merge(workflowResult, subscriberResult) │
└─────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│ Step 2: evaluateChannelPreference 渠道偏好检查         │
│                                                      │
│  将合并后的 WorkflowPreferences 转为 IPreferenceChannels │
│  调用 stepPreferred():                               │
│    result = all.enabled && channels[currentChannel]   │
│  如果 result = false → SKIPPED (SUBSCRIBER_PREFERENCE)│
└─────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│ Step 3: Schedule 勿扰窗口检查（独立于 Step 1-2）        │
│                                                      │
│  从 SUBSCRIBER_GLOBAL 偏好获取 schedule               │
│  如果 schedule.isEnabled && !isWithinSchedule():      │
│    对 delay/digest 非 critical → 延迟到下一可用时间     │
│    对 email/sms/push/chat 非 critical → CANCELED      │
│    in-app / critical → 始终放行                       │
└─────────────────────────────────────────────────────┘
```

---

## 六、旧版 `overridePreferences` 逻辑（Dashboard API）

[get-subscriber-template-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts#L288-L310) 使用独立的 `overridePreferences` 函数处理旧版 API 的偏好覆盖展示：

```typescript
const PRIORITY_ORDER = [
  PreferenceOverrideSourceEnum.TEMPLATE,          // 模板级 → 最低
  PreferenceOverrideSourceEnum.WORKFLOW_OVERRIDE,  // 工作流覆盖 → 中
  PreferenceOverrideSourceEnum.SUBSCRIBER,         // 订阅者 → 最高
];
```

按此顺序依次覆盖 `channels` 的布尔值，后者的值覆盖前者。此函数既用于 Dashboard 展示"覆盖来源"信息，**也直接参与 Worker 发送决策**（详见第九章、第十一章）。

---

## 八、订阅上下文分裂（contextKeys）

### 8.1 问题的本质

同一订阅者的偏好记录在数据库中并不是唯一的——`SUBSCRIBER_GLOBAL`、`SUBSCRIBER_WORKFLOW`、`SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 三类偏好都额外携带 `contextKeys: string[]` 字段。不同 `contextKeys` 的记录在 MongoDB 中是**不同的文档**，但属于同一个订阅者。

这意味着同一个订阅者可以拥有**多份**同类型的偏好，按上下文分裂：

```
subscriber A, type=SUBSCRIBER_GLOBAL, contextKeys=[]         → 文档 1
subscriber A, type=SUBSCRIBER_GLOBAL, contextKeys=["org:1"]  → 文档 2
subscriber A, type=SUBSCRIBER_GLOBAL, contextKeys=["org:2"]  → 文档 3
```

### 8.2 查询时的精确匹配

[buildContextExactMatchQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/base-repository.ts#L74-L111) 实现了严格的多文档匹配逻辑：

```
contextKeys === undefined || []  →  匹配 { $or: [字段不存在, 字段为 []] }
contextKeys = ["org:1"]         →  匹配 { contextKeys: { $all: ["org:1"], $size: 1 } }
```

关键点：**精确匹配**——输入 `["org:1"]` 只匹配恰好包含 `"org:1"` 一项的记录，不会匹配 `["org:1", "team:a"]`。

### 8.3 分裂对合并的影响

在 [get-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-preferences/get-preferences.usecase.ts#L193-L264) 中，查询 4 层偏好时，`SUBSCRIBER_WORKFLOW` 和 `SUBSCRIBER_GLOBAL` 的查询都追加了 `contextQuery`：

```typescript
// SUBSCRIBER_WORKFLOW 查询
{ _subscriberId, _templateId, type: SUBSCRIBER_WORKFLOW, ...contextQuery }

// SUBSCRIBER_GLOBAL 查询
{ _subscriberId, type: SUBSCRIBER_GLOBAL, ...contextQuery }
```

**效果**：同一次合并只会取到对应 contextKeys 的那一份偏好文档。如果 trigger 请求带了 `contextKeys: ["org:1"]`，那么合并使用的就是该上下文下的偏好，而不是默认空上下文的偏好。不同上下文之间的偏好**互不干扰，互不合并**。

### 8.4 Feature Flag 守护

整个 context 分裂机制由 `IS_CONTEXT_PREFERENCES_ENABLED` Feature Flag 控制。当 Flag 关闭时：
- `buildContextExactMatchQuery` 返回空对象 `{}`，不追加任何 context 过滤
- 所有上下文的偏好记录都可能被返回（取决于 MongoDB 的 findOne 行为，通常返回第一条匹配）
- 写入时，`contextKeys` 设为 `undefined`（不存储字段）

> 参见 [upsert-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts#L176-L193)

### 8.5 分裂的实际效果

| 维度 | 无 contextKeys | 有 contextKeys=["org:1"] |
|------|----------------|--------------------------|
| SUBSCRIBER_GLOBAL | 全局默认偏好 | org:1 上下文下的全局偏好 |
| SUBSCRIBER_WORKFLOW | 工作流级偏好 | org:1 上下文下的工作流级偏好 |
| Schedule（勿扰） | 来自默认全局偏好 | 来自 org:1 的全局偏好 |
| 合并逻辑 | 不变 | 不变，只是输入的偏好记录不同 |

---

## 九、租户维度覆盖 — WorkflowOverride

### 9.1 WorkflowOverride 是什么

`WorkflowOverride` 是一张**完全独立于 preferences 集合**的表，键为 `(workflowId, tenantId)`，值是 `IPreferenceChannels`（扁平的渠道布尔映射）：

```typescript
// workflow-override.entity.ts
class WorkflowOverrideEntity {
  _workflowId: string;
  _tenantId: string;
  active: boolean;
  preferenceSettings: IPreferenceChannels;  // { email: true, sms: false, ... }
}
```

> 实体定义见 [workflow-override.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/workflow-override/workflow-override.entity.ts#L9-L37)

它**不进 `MergePreferences`**，不在 4 层合并的任何位置。它仅通过 `overridePreferences` 函数在**展示层**叠加一次。

### 9.2 进入点：GetSubscriberTemplatePreference

[get-subscriber-template-preference.usecase.ts#L44-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts#L44-L79) 的执行流程：

```
1. 获取 initialChannels（工作流活跃步骤对应的渠道）
2. 获取 workflowOverride（仅当 command.tenant.identifier 存在时）
3. 获取 templateChannelPreference = template.preferenceSettings（旧版字段）
4. 获取 subscriberWorkflowPreference（通过 GetPreferences → MergePreferences 合并后的结果）
5. 调用 overridePreferences({
     template: templateChannelPreference,
     subscriber: subscriberWorkflowPreference.channels,
     workflowOverride: workflowOverride?.preferenceSettings,
   }, initialChannels)
```

### 9.3 overridePreferences 的三路叠加

```typescript
const PRIORITY_ORDER = [
  PreferenceOverrideSourceEnum.TEMPLATE,           // 1. 模板 preferenceSettings（最低）
  PreferenceOverrideSourceEnum.WORKFLOW_OVERRIDE,   // 2. 租户维度覆盖（中间）
  PreferenceOverrideSourceEnum.SUBSCRIBER,          // 3. 订阅者合并后偏好（最高）
];
```

[overridePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts#L288-L310) 按此顺序依次用每个 source 的渠道布尔值覆盖 `initialChannels`，后写覆盖前写，最终产出 `channels` + `overrides`（记录每个渠道被谁覆盖）。

### 9.4 WorkflowOverride 在发送决策中的真实权重

关键发现：**`overridePreferences` 的结果直接影响 Worker 的发送决策**。

在 [send-message.usecase.ts#L362-L379](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts#L362-L379) 中，`evaluateChannelPreference` 调用 `GetSubscriberTemplatePreference`，后者内部执行了 `overridePreferences`，返回的 `preference.channels` 直接被 `stepPreferred` 使用：

```typescript
// stepPreferred 判定逻辑
const workflowPreferred = preference.enabled;
const channelPreferred = Object.keys(preference.channels || {})
  .some(key => key === job.type && preference.channels?.[job.type]);
return workflowPreferred && channelPreferred;
```

这意味着 **WorkflowOverride 虽然不在 MergePreferences 中，但它可以通过 overridePreferences 二次叠加来关闭某个渠道**，且此关闭优先于订阅者偏好中该渠道的开启。

### 9.5 两条路径的差异

| 路径 | 是否含 WorkflowOverride | 何时走 |
|------|------------------------|--------|
| Worker SendMessage → evaluateChannelPreference → GetSubscriberTemplatePreference | **含**（前提：有 tenant） | 每个渠道步骤发送前 |
| Dashboard GetSubscriberPreference → calculateChannelsAndOverrides | **不含**（传空 `{}`） | 展示偏好列表时 |

Dashboard 路径在 [get-subscriber-preference.usecase.ts#L216-L225](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscribers/usecases/get-subscriber-preference/get-subscriber-preference.usecase.ts#L216-L225) 中传入 `workflowOverride: {}`，刻意忽略租户覆盖。这导致**用户在 Dashboard 看到的偏好状态可能与实际发送决策不一致**——Dashboard 不体现租户覆盖。

### 9.6 template.preferenceSettings 与 WORKFLOW_RESOURCE 的关系

`template.preferenceSettings` 是 NotificationTemplate 实体上的旧版字段（标记为 `@deprecated`），格式为 `IPreferenceChannels`（扁平布尔），不是 `WorkflowPreferences`（结构化对象）。

在新架构中，`WORKFLOW_RESOURCE` 偏好替代了 `preferenceSettings` 的角色。但 `overridePreferences` 仍然把 `preferenceSettings` 作为 TEMPLATE 层参与叠加，这是因为旧版工作流没有 `WORKFLOW_RESOURCE` 偏好记录，需要兼容。

对于新版 Framework 创建的工作流，`preferenceSettings` 通常为 `undefined`，TEMPLATE 层不参与覆盖。

---

## 十、SUBSCRIPTION_SUBSCRIBER_WORKFLOW — 名义最高优先级但不进主合并

### 10.1 设计意图

`SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 在 `PreferencesTypeEnum` 注释中被标记为优先级 1（最高），但它**不参与 `MergePreferences` 的 4 层深度合并**。

`MergePreferencesCommand` 只有 4 个偏好输入槽位，不含 subscription 维度：

```typescript
workflowResourcePreference?
workflowUserPreference?
subscriberGlobalPreference?
subscriberWorkflowPreference?
// 注意：没有 subscriptionSubscriberWorkflowPreference
```

### 10.2 实际执行路径：前置过滤，而非合并叠加

`SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 的生效方式是在 [subscriber-job-bound.usecase.ts#L430-L493](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/subscriber-job-bound/subscriber-job-bound.usecase.ts#L430-L493) 中的 `evaluateSubscriptionPreferences` 做前置判断：

```
对于 trigger 中的每个 topic subscription：
  1. 从 preferences 集合中查找 type=SUBSCRIPTION_SUBSCRIBER_WORKFLOW 的记录
     （需匹配 _subscriberId + _templateId + _topicSubscriptionId + contextKeys）
  2. 如果找到记录：
     a. 检查 all.condition — 如果有 JSON Logic 条件，用 payload 求值
     b. 如果没有 condition，使用 all.enabled 的值
  3. 如果求值结果为 false → 该 topic subscription 被过滤掉（从 topics 列表中移除）
  4. 如果未找到记录 → 默认放行（result: true）
```

### 10.3 与主合并的关系

```
Trigger 请求进入
  │
  ├─ 有 topics? → evaluateSubscriptionPreferences (SUBSCRIPTION_SUBSCRIBER_WORKFLOW)
  │   │
  │   ├─ 某个 subscription 被过滤 → 整个 trigger 对该 subscription 不会产生 notification
  │   └─ 某个 subscription 通过 → 继续执行
  │
  ├─ 获取 critical 标志（通过 GetPreferences → MergePreferences，不含 subscription 层）
  │
  └─ 创建 notification jobs → Worker 执行
       │
       ├─ RunJob: Schedule 检查
       └─ SendMessage: evaluateChannelPreference（GetSubscriberTemplatePreference → MergePreferences + overridePreferences）
```

**关键区别**：
- `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 的决策粒度是 **"这个 topic subscription 是否允许通知"**，是全量开关（all.enabled / all.condition）
- 主合并（4 层 MergePreferences + overridePreferences）的决策粒度是 **"某个渠道是否允许发送"**，可以精确到渠道级别

### 10.4 创建时机

[create-subscription-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscriptions/usecases/create-subscription-preferences/create-subscription-preferences.usecase.ts) 在订阅创建时，调用 `GetPreferences` 获取**仅工作流层的偏好**（`excludeSubscriberPreferences: true`），将其快照为 `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 记录：

```typescript
// 获取工作流层偏好时排除了订阅者偏好
const getPreferencesResult = await this.getPreferences.safeExecute(
  GetPreferencesCommand.create({
    ...
    excludeSubscriberPreferences: true,  // ← 关键：只用工作流层
  })
);
enabled = getPreferencesResult?.preferences.all?.enabled;
```

这意味着 subscription 偏好的初始值来自工作流定义，之后用户可以通过 Inbox API 的 `UpdatePreferences` 单独修改。

### 10.5 订阅级偏好的 condition 字段

`SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 支持 `all.condition`（JSON Logic 表达式），在 `evaluateSubscriptionPreferences` 中用 `{ payload }` 求值。其他 4 层偏好的 `condition` 字段目前仅在创建时存储，**不在发送链中求值**（仅 subscription 层做了 condition 求值）。

---

## 十一、overridePreferences 在发送链上的二次叠加

### 11.1 两条合并路径的叠加

发送决策实际上经历了**两次独立的偏好合并**：

**第一次：MergePreferences（深度合并，结构化）**

```
WORKFLOW_RESOURCE + USER_WORKFLOW + SUBSCRIBER_GLOBAL + SUBSCRIBER_WORKFLOW
→ 深度合并 → WorkflowPreferences（含 all + channels 结构）
```

**第二次：overridePreferences（扁平覆盖，渠道级）**

```
initialChannels（全部 true）
← TEMPLATE（旧版 preferenceSettings）覆盖
← WORKFLOW_OVERRIDE（租户维度）覆盖
← SUBSCRIBER（MergePreferences 合并后的渠道映射）覆盖
→ 最终 channels + overrides 来源追踪
```

### 11.2 二次叠加带来的复杂性

由于 `overridePreferences` 的 SUBSCRIBER 层输入来自 `MergePreferences` 的合并结果，而 TEMPLATE 和 WORKFLOW_OVERRIDE 来自独立数据源，可能出现：

**场景**：管理员在 Dashboard 设置 WORKFLOW_RESOURCE `email.enabled = true`，租户 A 设置 WorkflowOverride `email = false`，用户设置 SUBSCRIBER_WORKFLOW `email.enabled = true`

- MergePreferences 结果：`email.enabled = true`（用户偏好覆盖工作流默认）
- overridePreferences：
  1. TEMPLATE 层：无值（新版工作流无 preferenceSettings）→ 不覆盖
  2. WORKFLOW_OVERRIDE 层：`email = false` → 覆盖为 false
  3. SUBSCRIBER 层：`email = true`（来自 MergePreferences）→ 覆盖回 true
- 最终结果：`email = true`

虽然在这个例子中最终结果符合预期（用户偏好最高），但覆盖路径是"先被租户关闭，再被用户打开"，而非"用户直接覆盖工作流"。这导致 `overrides` 追踪数组中会同时出现 WORKFLOW_OVERRIDE 和 SUBSCRIBER 两个来源。

### 11.3 Dashboard 展示与实际发送的脱节

由于 Dashboard 路径（`GetSubscriberPreference`）传入 `workflowOverride: {}`，而 Worker 路径（`GetSubscriberTemplatePreference`）传入实际的 WorkflowOverride，两个路径的最终渠道结果可能不同。

具体来说，如果存在 WorkflowOverride 关闭了某个渠道：
- Dashboard 展示：不体现租户覆盖，渠道显示为开启
- 实际发送：租户覆盖生效，渠道可能被关闭

这是一个已知的展示/行为不一致。

---

## 十二、关键代码索引

| 文件 | 作用 |
|------|------|
| [merge-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.usecase.ts) | 核心合并逻辑 |
| [merge-preferences.command.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.command.ts) | 合并命令定义 |
| [merge-preferences.spec.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.spec.ts) | 合并测试用例（覆盖所有组合） |
| [get-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-preferences/get-preferences.usecase.ts) | 从数据库收集 4 层偏好 |
| [buildWorkflowPreferences.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/utils/buildWorkflowPreferences.ts) | 部分偏好 → 完整偏好填充 |
| [preferences.const.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/consts/preferences/preferences.const.ts) | 默认偏好常量 |
| [workflow-channel-preferences.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/types/workflow-channel-preferences.ts) | 类型定义（PreferencesTypeEnum, Schedule 等） |
| [schedule-validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/schedule-validator.ts) | 勿扰时段判断 + 下次可用时间计算 |
| [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts) | Worker 侧 Schedule 检查入口 |
| [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts) | 渠道偏好评估（stepPreferred） |
| [upsert-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts) | 偏好写入（含全局偏好 channel 联动清理） |
| [get-subscriber-template-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts) | overridePreferences 叠加（含 WorkflowOverride） |
| [get-subscriber-schedule.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-schedule/get-subscriber-schedule.usecase.ts) | 独立获取 Schedule |
| [base-repository.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/base-repository.ts#L74-L111) | buildContextExactMatchQuery 定义 |
| [subscriber-job-bound.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/subscriber-job-bound/subscriber-job-bound.usecase.ts#L430-L493) | SUBSCRIPTION_SUBSCRIBER_WORKFLOW 前置过滤 |
| [workflow-override.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/workflow-override/workflow-override.entity.ts) | 租户维度偏好覆盖实体 |
| [create-subscription-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscriptions/usecases/create-subscription-preferences/create-subscription-preferences.usecase.ts) | 订阅偏好初始化（excludeSubscriberPreferences） |
| [update-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts) | Inbox 偏好更新（含 subscription 级偏好路由） |
| [get-subscriber-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscribers/usecases/get-subscriber-preference/get-subscriber-preference.usecase.ts) | Dashboard 偏好列表（不含 WorkflowOverride） |
