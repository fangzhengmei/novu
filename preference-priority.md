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

按此顺序依次覆盖 `channels` 的布尔值，后者的值覆盖前者。这仅用于 Dashboard 展示"覆盖来源"信息，不影响 Worker 侧的发送决策。

---

## 七、关键代码索引

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
| [get-subscriber-template-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts) | 旧版 overridePreferences 展示逻辑 |
| [get-subscriber-schedule.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-schedule/get-subscriber-schedule.usecase.ts) | 独立获取 Schedule |
