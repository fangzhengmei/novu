# Novu 订阅偏好与全局静默规则优先级合并分析

## 一、偏好来源分层（4 层 + 1 专项）

Novu 的偏好体系由 5 种类型（`PreferencesTypeEnum`）组成。**注意：文档注释中的"具体度排序"与代码中实际的决策顺序不一致。**

### 1.1 名义优先级（来自 `PreferencesTypeEnum` 注释）

[workflow-channel-preferences.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/types/workflow-channel-preferences.ts#L3-L21) 的 enum 注释声称优先级为 1 到 5，数字越小优先级越高：

| 名义优先级 | 枚举值 | 含义 |
|-----------|--------|------|
| 1（最高） | `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` | 订阅范围内的工作流偏好 |
| 2 | `SUBSCRIBER_WORKFLOW` | 订阅者对某工作流的偏好 |
| 3 | `SUBSCRIBER_GLOBAL` | 订阅者全局偏好（含勿扰日程） |
| 4 | `USER_WORKFLOW` | Dashboard 用户对工作流的偏好 |
| 5（最低） | `WORKFLOW_RESOURCE` | Framework 代码定义的工作流默认偏好 |

### 1.2 实际决策顺序（对照代码的真实路径）

实际代码中，**`SUBSCRIPTION_SUBSCRIBER_WORKFLOW` 不参与 `MergePreferences` 的主合并**，而是在 trigger 阶段做独立的前置过滤。完整的决策顺序按时间线排列如下：

```
Trigger 进入 → ① 订阅级前置过滤 → ② MergePreferences 4 层深度合并 → ③ overridePreferences 三路扁平覆盖 → ④ Schedule 勿扰检查
```

每层的具体定位：

| 阶段 | 实际顺序 | 枚举值 | 决策点 |
|------|---------|--------|--------|
| ① 前置过滤 | 第 1 位 | `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` | 在 [SubscriberJobBound](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/subscriber-job-bound/subscriber-job-bound.usecase.ts#L430-L493) 中逐 subscription 判断，粒度为 `all.enabled` + `all.condition` |
| ② 主合并 | 第 2~5 位 | `WORKFLOW_RESOURCE` → `USER_WORKFLOW` → `SUBSCRIBER_GLOBAL` → `SUBSCRIBER_WORKFLOW` | 在 [MergePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.usecase.ts#L58-L98) 中深度合并 |
| ③ 二次叠加 | 第 6 位 | 含 `WORKFLOW_OVERRIDE`（独立表） | 在 [overridePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts#L288-L310) 中扁平覆盖 |
| ④ 勿扰检查 | 第 7 位 | 来自 `SUBSCRIBER_GLOBAL` 的 `schedule` 字段 | 在 [RunJob](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L176-L256) 中独立检查 |

**修正后的完整分层表**：

| 实际决策顺序 | 枚举值 | 含义 | 存储位置 | 设定者 |
|-------------|--------|------|----------|--------|
| 1（最早判定） | `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` | 订阅范围内的工作流偏好 | preferences 集合 | 订阅 API / Inbox |
| 2 | `WORKFLOW_RESOURCE` | Framework 代码定义的工作流默认偏好 | preferences 集合 | 开发者在代码中定义 |
| 3 | `USER_WORKFLOW` | Dashboard 用户对工作流的偏好 | preferences 集合 | 管理员在 Dashboard |
| 4 | `SUBSCRIBER_GLOBAL` | 订阅者全局偏好（含勿扰日程） | preferences 集合 | 终端用户在偏好面板 |
| 5 | `SUBSCRIBER_WORKFLOW` | 订阅者对某工作流的偏好 | preferences 集合 | 终端用户在偏好面板 |
| 6（独立表） | *`WorkflowOverride`* | 租户维度渠道覆盖 | workflow-overrides 集合 | 租户管理员 |
| 7（独立检查） | *`Schedule`* | 勿扰窗口 | preferences 集合（SUBSCRIBER_GLOBAL 上） | 终端用户 |

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

在 [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L176-L256) 中，Schedule 检查**独立于偏好合并**进行，且采用"先尝试延期，不能延期再取消"的两段式策略：

```
isOutsideSubscriberSchedule = schedule.isEnabled && !isWithinSchedule(schedule, now, timezone)

if isOutsideSubscriberSchedule:
  ├─ 第一步：尝试延期（仅 delay/digest 且非 critical）
  │    shouldExtendToSubscriberSchedule?
  │      ├─ 是 → extendJobToNextAvailableSchedule
  │      │      ├─ 延期成功 → job 状态设为 DELAYED，return（等下一轮触发）
  │      │      └─ 延期失败 → 继续向下（不 return）
  │      └─ 否 → 继续向下
  │
  └─ 第二步：判断是否取消
       !shouldSkipScheduleCheck?
         ├─ 是（不跳过检查）→ job 标记为 CANCELED，return
         └─ 否（跳过检查）→ 继续向下 → 实际放行
```

### 4.4 `isWithinSchedule` 算法与跨夜判定

[isWithinSchedule](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/schedule-validator.ts#L15-L67) 的核心是**"两天窗口 + 时段比较"**：

**步骤 1：时区转换**
- 若有 timezone，用 `utcToZonedTime` 把 UTC 现在时刻转到订阅者时区
- 没有 timezone 就用 UTC 时间

**步骤 2：确定需要检查哪几天**
- 初始只检查"今天"
- 再看"昨天"是否有跨夜时段（`end < start`，即结束时间早于开始时间，说明跨午夜）
  - 如果有，把"昨天"也加入检查列表
- 这是为了处理类似 "11:00 PM - 02:00 AM" 这种跨午夜的可用时段：当现在是凌晨 1 点时，它属于昨天晚上开始的那个时段

**步骤 3：逐天检查**
- 跳过 `isEnabled = false` 的日子
- 跳过没有配置 hours 的日子
- 对每个时段调用 `isTimeInRange`

**步骤 4：`isTimeInRange` 的跨夜处理**
```typescript
if (endInMinutes < startInMinutes) {
  // 跨午夜时段：当前时间 >= 开始 或 当前时间 <= 结束 → 在范围内
  return timeInMinutes >= startInMinutes || timeInMinutes <= endInMinutes;
}
// 普通时段：开始 <= 当前时间 <= 结束 → 在范围内
return timeInMinutes >= startInMinutes && timeInMinutes <= endInMinutes;
```

**举例**：时段 "11:00 PM - 02:00 AM"（start=23:00, end=2:00）
- 现在 00:30 → 0:30 < 2:00 → 满足 `time <= end` → 在范围内 ✓
- 现在 23:30 → 23:30 >= 23:00 → 满足 `time >= start` → 在范围内 ✓
- 现在 15:00 → 两个都不满足 → 不在范围内 ✗

### 4.5 延期机制与计数挂点

[extendJobToNextAvailableSchedule](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts#L911-L983) 是 delay/digest 类型 job 的"软处理"路径。

**计数字段**：`job.scheduleExtensionsCount`
- 挂在 MongoDB 的 `jobs` 集合文档上
- 初始为 `undefined`，读取时当作 `0`
- 每次成功延期时 `$set` 为 `currentExtensions + 1`

**最大延期次数**：`MAX_EXTENSIONS = 3`

**延期成功的条件**（全部满足）：
1. `currentExtensions < MAX_EXTENSIONS`（还没到 3 次）
2. `calculateNextAvailableTime` 返回的时间与当前时间差 > 0（确实存在未来的可用时间）
3. job 更新成功

**延期失败的两种情况**：
| 原因 | 返回值 | 后续行为 |
|------|--------|----------|
| 已达 3 次上限 | `false` | 走到第二步判断（取消 or 放行） |
| 下一可用时间就是现在（delayMs = 0） | `false` | 走到第二步判断（取消 or 放行） |

### 4.6 第 4 次为什么"跳过"即放行

第 4 次触发时，`scheduleExtensionsCount = 3`，达到 `MAX_EXTENSIONS`，`extendJobToNextAvailableSchedule` 返回 `false`。

然后代码继续执行到第二步判断：

```typescript
if (isOutsideSubscriberSchedule && !this.shouldSkipScheduleCheck(job, notification.critical)) {
  // → 标记为 CANCELED
}
```

关键在于 `shouldSkipScheduleCheck` 对 **delay 和 digest 类型返回 `true`**（表示"跳过 Schedule 检查"）。所以：

```
!shouldSkipScheduleCheck = !true = false
```

整个 `if` 条件为 `false`，**不会进入 CANCELED 分支**，代码继续向下执行 → job 正常运行 → 消息实际被发送。

> 这就是"第 4 次走名为跳过实际放行的分支"的含义：`shouldSkipScheduleCheck` 函数名意为"是否跳过 Schedule 检查"，返回 `true` 表示跳过检查，跳过检查的结果就是消息直接放行。

**对不同类型 job 的完整命运矩阵**（在勿扰时段内）：

| job 类型 | critical? | 第一步：尝试延期? | 延期失败后 | 最终命运 |
|----------|-----------|------------------|-----------|----------|
| email/sms/push/chat | 否 | 否（shouldExtend=false） | 不跳过检查 → CANCELED | 取消 |
| email/sms/push/chat | 是 | 否（shouldExtend=false） | 跳过检查 → 放行 | 立即发送 |
| in-app | 任意 | 否（shouldExtend=false） | 跳过检查 → 放行 | 立即发送 |
| delay/digest | 否 | 是，最多延 3 次 | 跳过检查 → 放行 | 前 3 次延期，第 4 次强发 |
| delay/digest | 是 | 否（shouldExtend=false） | 跳过检查 → 放行 | 立即发送 |
| trigger | 任意 | 否 | 跳过检查 → 放行 | 立即执行 |

### 4.7 下一可用时间预扫算法

[calculateNextAvailableTime](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/schedule-validator.ts#L140-L229) 用于找到"下一个开始时间"。算法是**以天为外层循环，以时段为内层循环的线性扫描**：

**扫描窗口**：`dayOffset` 从 `-1` 到 `7`，共 9 天
- dayOffset = -1：昨天（用于检测跨午夜且当前正处于跨夜时段中）
- dayOffset = 0：今天
- dayOffset = 1 ~ 7：未来 7 天（保证覆盖完整一周）

**对每一天**：
1. 取那天的星期几（monday/tuesday/...）
2. 如果那天没启用或没有 hours，跳过
3. 对那天的每个时段：
   a. 解析 start/end 为时分
   b. 构造当天的 startZoned 和 endZoned 时刻
   c. 如果 end 在 start 之前（跨午夜），把 endZoned 加 1 天
   d. **如果是昨天或今天，且当前时间正好在时段内** → 返回当前时间（现在就在可用时段里）
   e. **如果是未来的某天，或 start 在当前时间之后** → 返回 start 时刻（下一个开始点）

**返回的都是 UTC 时间**（如果输入有时区，用 `zonedTimeToUtc` 转回 UTC）。

**边界情况**：如果 9 天都扫完了还没找到可用时段（理论上不应该发生，因为一周内至少有一天配置了时段），兜底返回 `nowUtc`。

### 4.8 Schedule 与 Critical 的交互

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

当一条通知触发后，决策路径如下（**注意：Step 2 实际包含两次独立的偏好叠加**）：

```
┌──────────────────────────────────────────────────────────────────────┐
│ Step 1: MergePreferences 4 层深度合并                                  │
│                                                                      │
│  WORKFLOW_RESOURCE ─┐                                                │
│  USER_WORKFLOW     ─┤── 深度合并 ──→ workflowResult                   │
│                      │                                                │
│  SUBSCRIBER_GLOBAL ──┤                                                │
│  SUBSCRIBER_WORKFLOW─┘── 深度合并 ──→ subscriberResult                │
│                                                                      │
│  如果 readOnly=true 或 excludeSubscriber=true:                         │
│    finalPreferences = workflowResult                                   │
│  否则:                                                                │
│    finalPreferences = merge(workflowResult, subscriberResult)         │
│         ↓                                                             │
│  产出：WorkflowPreferences 结构化对象（all + channels）                │
└──────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Step 2: evaluateChannelPreference 渠道偏好检查（含二次叠加）            │
│                                                                      │
│  子步骤 2a：从 MergePreferences 拿 channels 映射                       │
│            WorkflowPreferences.channels → { email: true, sms: false } │
│                                                                      │
│  子步骤 2b：overridePreferences 三路扁平覆盖                           │
│                                                                      │
│            initialChannels = { email:true, sms:true, push:true, ... } │
│              ← TEMPLATE (旧版 preferenceSettings) 覆盖                │
│              ← WORKFLOW_OVERRIDE (租户维度) 覆盖                       │
│              ← SUBSCRIBER (来自子步骤 2a 的渠道映射) 覆盖              │
│                                                                      │
│            产出：最终 channels + overrides 来源追踪                    │
│                                                                      │
│  子步骤 2c：stepPreferred 判定                                         │
│            result = all.enabled && channels[currentChannel]            │
│            若 result = false → SKIPPED (SUBSCRIBER_PREFERENCE)        │
└──────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Step 3: Schedule 勿扰窗口检查（独立于 Step 1-2）                         │
│                                                                      │
│  从 SUBSCRIBER_GLOBAL 偏好获取 schedule                                │
│  如果 schedule.isEnabled && !isWithinSchedule():                       │
│    对 delay/digest 非 critical → 最多延期 3 次，第 4 次强发             │
│    对 email/sms/push/chat 非 critical → CANCELED                      │
│    in-app / critical / trigger → 始终放行                             │
└──────────────────────────────────────────────────────────────────────┘
```

> 重要提示：Step 2b 的 `WORKFLOW_OVERRIDE`（租户维度覆盖）**仅在发送链（Worker）路径中生效**，Dashboard 的偏好展示路径会传空对象 `{}` 跳过这一层。

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

## 七、写全局偏好时顺手清除工作流级同名渠道

### 7.1 触发时机

在 [upsertSubscriberGlobalPreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts#L66-L79) 方法中，**写入全局偏好之前**，会先调用 `deleteSubscriberWorkflowChannelPreferences`：

```typescript
public async upsertSubscriberGlobalPreferences(command: UpsertSubscriberGlobalPreferencesCommand) {
  await this.deleteSubscriberWorkflowChannelPreferences(command);  // ← 先清
  return this.upsert({ ... });                                    // ← 再写
}
```

### 7.2 清除逻辑

[deleteSubscriberWorkflowChannelPreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts#L81-L108) 的行为：

1. 提取本次更新涉及的所有渠道（`command.preferences.channels` 的 keys）
2. 如果没有渠道字段被更新，直接返回（不做任何清除）
3. 构造 MongoDB `$unset` payload：
   ```
   {
     "preferences.channels.email": "",
     "preferences.channels.sms": "",
     ...
   }
   ```
4. 查找所有 `type = SUBSCRIBER_WORKFLOW` 的偏好记录，要求至少有一个目标渠道字段存在（`$exists: true`）
5. 对匹配的记录执行 `$unset`，删除这些渠道字段

**作用范围**：只清除 `SUBSCRIBER_WORKFLOW`（订阅者工作流级）偏好中的对应渠道，不碰 `SUBSCRIPTION_SUBSCRIBER_WORKFLOW`（订阅级）和工作流层偏好。

### 7.3 设计意图

这是一个**"回退"机制**：当用户在全局层面重新设定某个渠道的偏好时，之前在各个工作流上单独设置的渠道覆盖就变得没有意义了——因为全局值已经改变，工作流级的旧值是相对于旧全局值的覆盖。系统选择直接清除工作流级的渠道字段，让合并时自然回退到新的全局值。

**举例**：
- 初始状态：全局 `email=true`，工作流 A `email=false`（用户在工作流 A 上关了 email）
- 用户修改全局偏好：`email=false`（全局关掉 email）
- 系统自动清除所有 SUBSCRIBER_WORKFLOW 记录中的 `preferences.channels.email` 字段
- 结果：工作流 A 不再有独立的 email 覆盖，合并时继承全局的 `email=false`

### 7.4 副作用

这种"顺带清除"的设计有一个值得注意的副作用：**用户之前在各工作流上精心配置的个性化偏好，会因为一次全局偏好修改而全部丢失**，且无法恢复。

例如用户在 5 个工作流上分别设置了不同的 email 开关，某天他修改了全局的 sms 偏好，结果所有工作流上的 sms 个性化设置都被清掉了。

### 7.5 反向不成立

注意这个清除是**单向的**：
- ✅ 写全局偏好 → 清除工作流级同名渠道
- ❌ 写工作流级偏好 → 不会影响全局偏好

这符合"更具体的层可以覆盖更通用的层，但通用层的修改会重置具体层"的设计哲学。

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

## 十二、读路径与写路径使用两套不同的深合并工具

### 12.1 读路径：`es-toolkit/toMerged`

[MergePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.usecase.ts#L5) 在读路径中使用 `toMerged`（来自 `es-toolkit`）：

```typescript
import { toMerged } from 'es-toolkit';

const merged = preferencesList.reduce(
  (acc, preference) => toMerged(acc, preference),
  {}
);
```

`toMerged` 的行为特点：
- 递归深合并，对对象做 merge，对非对象做**直接替换**
- **数组行为**：源数组**完全替换**目标数组（不逐元素合并）
- 不提供自定义合并策略的入口

这在偏好场景下是合适的：`preferences.channels.email` 是一个对象 `{ enabled: true }`，应该被递归合并；而 `preferences.all.condition` 是一个 JSON Logic 对象，替换语义也是正确的。

### 12.2 写路径：自研 `deepMerge`

[UpsertPreferences.updatePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts#L214-L217) 在写路径中使用自研的 [deepMerge](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/utils/deepmerge.ts#L186-L192)：

```typescript
import { deepMerge } from '../../utils';

const mergedPreferences = deepMerge([
  foundPreference.preferences,       // 数据库中的现有值
  command.preferences as WorkflowPreferencesPartial,  // 用户提交的部分更新
]);
```

`deepMerge` 的行为特点（源自 [deepmerge](https://github.com/TehShrike/deepmerge) 库）：
- 递归深合并，行为与 `toMerged` 类似
- **数组行为**：默认 `arrayMerge` 策略是**concat + clone**（拼接而非替换）
- 支持 `customMerge` 自定义特定 key 的合并策略
- 支持 `isMergeableObject` 控制哪些值被视为可合并对象

### 12.3 两套工具的关键差异

| 维度 | `toMerged`（读路径） | `deepMerge`（写路径） |
|------|---------------------|---------------------|
| 来源 | `es-toolkit` 第三方库 | 自研（fork 自 deepmerge） |
| 数组合并策略 | **替换**（source 覆盖 target） | **拼接**（concat） |
| 自定义合并 | 不支持 | 支持 `customMerge` |
| 使用场景 | 多层偏好合并（跨文档合并） | 单层偏好局部更新（同文档合并） |
| 输入 | 两个对象 | 对象数组 |

**为什么不用同一套**：读路径需要"后者覆盖前者"的语义（更具体的层覆盖更通用的层），数组替换正好符合；写路径需要"局部 patch"的语义——用户可能只提交 `{ channels: { email: { enabled: false } } }`，需要和数据库中的完整记录合并，而不是替换。

### 12.4 写路径中 condition 字段的显式再覆盖

`deepMerge` 合并后，[updatePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts#L226-L235) 又对 `condition` 字段做了一次显式覆盖：

```typescript
$set: {
  preferences: {
    ...mergedPreferences,
    ...(mergedPreferences.all && {
      all: {
        ...mergedPreferences.all,
        ...(command.preferences.all?.condition !== undefined && {
          condition: command.preferences.all?.condition,
        }),
      },
    }),
  },
  schedule: command.schedule,
  _userId: command.userId,
}
```

**原因**：`condition` 是一个 JSON Logic 对象，`deepMerge` 会把它当作可合并对象做递归合并。但 condition 的语义是**完整替换**，不是逐字段合并——用户提交了新的 condition，就应该完全覆盖旧的，不能把新旧两个 condition 的字段混在一起。

**机制**：`command.preferences.all?.condition !== undefined` 判断用户是否提交了 condition 字段：
- 提交了（包括 `null`）→ 用提交的值覆盖合并结果
- 没提交（`undefined`）→ 保持合并结果中的旧 condition 不变

这是一个**"合后再覆盖"的兜底模式**——先让通用工具做默认合并，再对特殊语义的字段做显式修正。

---

## 十三、偏好面板点开关路由到三种不同层

### 13.1 `PreferenceLevelEnum` 与路由逻辑

[PreferenceLevelEnum](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/entities/subscriber-preference/subscriber-preference.interface.ts#L43-L46) 定义了两个级别：

```typescript
export enum PreferenceLevelEnum {
  GLOBAL = 'global',
  TEMPLATE = 'template',
}
```

在 [UpdatePreferences.storePreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts#L306-L378) 中，同一个偏好面板的开关点击，根据是否携带工作流和订阅信息，被路由到三种不同的写入层：

```
storePreferences(item):
  if item.workflowId && item.subscriptionId:
    → upsertTopicSubscriptionPreferences     // SUBSCRIPTION_SUBSCRIBER_WORKFLOW

  else if item.workflowId:
    → upsertSubscriberWorkflowPreferences    // SUBSCRIBER_WORKFLOW

  else:
    → upsertSubscriberGlobalPreferences      // SUBSCRIBER_GLOBAL
```

### 13.2 三种路由场景

| 请求参数 | level | 写入层 | 含义 |
|---------|-------|--------|------|
| 无 workflowId，无 subscriptionId | `global` | `SUBSCRIBER_GLOBAL` | 用户改全局偏好 |
| 有 workflowId，无 subscriptionId | `template` | `SUBSCRIBER_WORKFLOW` | 用户改某个工作流的偏好 |
| 有 workflowId + subscriptionId | `template` | `SUBSCRIPTION_SUBSCRIBER_WORKFLOW` | 用户改某个订阅上下文的工作流偏好 |

### 13.3 写入后的读取路由

[UpdatePreferences.findPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts#L206-L304) 同样按三级路由读取，但读路径的差异更大：

| 场景 | 读取方式 | 特点 |
|------|---------|------|
| template + subscriptionIdentifier + workflowId | 直接查 preferences 集合 | **绕过 MergePreferences**，直接读 SUBSCRIPTION_SUBSCRIBER_WORKFLOW 记录 |
| template + workflowId | 调用 `GetSubscriberTemplatePreference` | 走完整合并路径（MergePreferences + overridePreferences） |
| 无 workflowId | 调用 `GetSubscriberGlobalPreference` | 只读全局层 + 填充默认值 |

**关键发现**：订阅级偏好的读取路径**不经过 MergePreferences**，直接从数据库读取单条记录。这意味着用户在偏好面板上看到的订阅级偏好值，和发送决策中经过多层合并后的值可能不同——因为发送决策走的是完整合并路径，而面板只展示该层自己的值。

### 13.4 Critical 工作流的保护

[UpdatePreferences.getWorkflow](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts#L93-L113) 在写入前检查工作流是否 critical：

```typescript
if (workflow.critical) {
  throw new BadRequestException(
    `Critical workflow with id: ${command.workflowIdOrIdentifier} can not be updated`
  );
}
```

这是 API 层的保护，阻止用户通过 Inbox API 修改 critical 工作流的偏好。但注意这个检查只发生在 `level === TEMPLATE` 且有 `workflowIdOrIdentifier` 时——`level === GLOBAL` 的修改不受此限制。

---

## 十四、查全局偏好的三个独立入口

### 14.1 入口一：`GetSubscriberGlobalPreference`（API 层）

[get-subscriber-global-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscribers/usecases/get-subscriber-global-preference/get-subscriber-global-preference.usecase.ts) — 用于 Dashboard 和 Inbox 的全局偏好展示：

- 调用 `GetPreferences.getSubscriberGlobalPreference` 获取合并后的全局偏好
- 额外计算活跃渠道列表（通过遍历所有工作流的 steps），过滤掉不活跃的渠道
- 填充默认值（未设置的渠道默认 `true`）
- 返回 `{ enabled, channels, schedule }` 格式

**用途**：Inbox API 和 Dashboard 的偏好面板展示。

### 14.2 入口二：`GetSubscriberSchedule`（Worker 层）

[get-subscriber-schedule.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-schedule/get-subscriber-schedule.usecase.ts) — 用于 Worker 的 Schedule 检查：

- 直接查 `preferencesRepository.findOne`，取 `SUBSCRIBER_GLOBAL` 类型记录
- **完全绕过 `GetPreferences`**，不经过任何合并逻辑
- 只返回 `subscriberGlobalPreference?.schedule`
- 不关心 `preferences` 字段

**为什么绕过主路径**：Schedule 检查在 Worker 的 RunJob 热路径上，每次执行 job 都会触发。走完整的 `GetPreferences` → `MergePreferences` 路径太重——Schedule 只需要 `schedule` 字段，不需要 4 层合并后的渠道偏好。直接查一条记录比合并 4 条记录快得多。

### 14.3 入口三：`GetPreferences.getSubscriberGlobalPreference`（内部层）

[get-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-preferences/get-preferences.usecase.ts) 中的 `getSubscriberGlobalPreference` 方法 — 用于内部服务调用：

- 只取 `SUBSCRIBER_GLOBAL` 一层偏好
- 用 `buildWorkflowPreferences` 填充默认值
- 返回结构化的 `WorkflowPreferences` 对象
- **不合并其他层**——只看全局层自己的值

**用途**：被 `GetSubscriberGlobalPreference`（入口一）和其他需要全局偏好原始值的场景调用。

### 14.4 三个入口的对比

| 入口 | 调用者 | 经过合并? | 返回内容 | 绕过主路径的原因 |
|------|--------|----------|---------|----------------|
| `GetSubscriberGlobalPreference` | Inbox/Dashboard API | 仅全局层 | `{ enabled, channels, schedule }` + 活跃渠道过滤 | 需要展示友好的全局偏好视图 |
| `GetSubscriberSchedule` | Worker RunJob | 完全不合并 | 仅 `schedule` | 热路径性能优化，不需要渠道偏好 |
| `GetPreferences.getSubscriberGlobalPreference` | 内部服务 | 仅全局层 | 结构化 `WorkflowPreferences` | 只需原始全局值，不需多层合并 |

---

## 十五、Schedule 是文档顶层字段而非合并对象属性

### 15.1 Schema 定义

在 [preferences.schema.ts#L76](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/preferences/preferences.schema.ts#L76) 中，`schedule` 是 `preferencesSchema` 的**顶层字段**，与 `preferences` 对象平级：

```typescript
const preferencesSchema = new Schema<PreferencesDBModel>({
  _environmentId: ...,
  _subscriberId: ...,
  type: ...,
  preferences: {         // ← 偏好合并对象
    all: { ... },
    channels: { ... },
  },
  schedule: Schema.Types.Mixed,  // ← 顶层独立字段，不在 preferences 内
  contextKeys: ...,
  contextKeysHash: ...,
});
```

### 15.2 对合并逻辑的影响

`MergePreferences` 使用 `toMerged` 做深度合并时，输入的每个偏好实体被展平为 `preference.preferences`（不包含 `schedule`）。`schedule` 的透传在合并完成后单独处理：

```typescript
// merge-preferences.usecase.ts
return {
  preferences: mergedPreferences.preferences,  // 来自 toMerged 的结果
  schedule: mergedPreferences.schedule,         // 直接从最后一条有 schedule 的记录透传
  type: mergedPreferences.type,
  source,
};
```

这意味着 `schedule` **永远不参与深度合并**——如果某条记录上有 `schedule`，它就原样透传；如果没有，就是 `undefined`。

### 15.3 写入时的特殊处理

在 `updatePreferences` 中，`schedule` 也被单独处理，不参与 `deepMerge`：

```typescript
$set: {
  preferences: { ...mergedPreferences, ... },  // deepMerge 的结果
  schedule: command.schedule,                   // 直接覆盖，不合并
  _userId: command.userId,
}
```

如果用户提交了 `schedule: undefined`，MongoDB 的 `$set` 会把 `schedule` 设为 `undefined`（等同于删除）。如果用户没有提交 `schedule` 字段，`command.schedule` 为 `undefined`，也会被 `$set` 覆盖。

**注意**：这意味着任何一次偏好更新（即使只改渠道开关）如果不带 `schedule` 字段，都可能意外清除已有的 Schedule 设定。但由于只有 `upsertSubscriberGlobalPreferences` 会传 `schedule` 参数，且前端在更新全局偏好时通常会把当前 Schedule 一起传回来，所以实际上不太容易触发这个问题。

---

## 十六、条件表达式访问触发数据的固定前缀 `payload`

### 16.1 jsonLogic 调用的数据绑定

[evaluatePreferenceCondition](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/subscriber-job-bound/subscriber-job-bound.usecase.ts#L503) 使用 jsonLogic 求值时的数据绑定：

```typescript
const result = jsonLogic.apply(condition as RulesLogic, { payload });
```

第二个参数 `{ payload }` 就是传给 jsonLogic 的数据上下文。这意味着在 condition 表达式中，**必须通过 `payload` 前缀访问触发数据**。

### 16.2 正确 vs 错误的 condition 写法

假设触发时传入了 `{ "event_type": "order_created", "amount": 100 }`，则：

**正确写法**：
```json
{ "==": [{ "var": "payload.event_type" }, "order_created"] }
{ ">": [{ "var": "payload.amount" }, 50] }
```

**错误写法**（不会报错，但永远匹配不到）：
```json
{ "==": [{ "var": "event_type" }, "order_created"] }
{ ">": [{ "var": "amount" }, 50] }
```

因为 jsonLogic 的 `{ "var": "event_type" }` 会在数据上下文的顶层查找 `event_type` 键，而数据上下文的顶层只有 `payload` 键——实际的触发数据被嵌套了一层。

### 16.3 与工作流 step filter 的对比

工作流 step 的 filter（ConditionsFilter）使用不同的数据绑定结构：

```typescript
// conditions-filter.usecase.ts
const variables = {
  payload: command.payload,
  subscriber: command.subscriber,
  ...
};
```

这里 subscriber、payload 等都在顶层，所以 step filter 中可以用 `{ "var": "payload.event_type" }` 来访问——格式恰好与偏好 condition 一致。

但偏好 condition 的数据上下文**只有 `payload`**，不支持访问 `subscriber` 等其他上下文变量。这是一个功能限制：condition 表达式不能基于订阅者属性做条件判断。

### 16.4 设计原因

将触发数据包裹在 `payload` 前缀下，目的是**命名空间隔离**——防止 condition 表达式意外访问到 jsonLogic 内部状态或其他非预期数据。如果直接把触发数据展开到顶层，当触发数据中包含与 jsonLogic 保留字冲突的键名时，可能导致不可预期的行为。

---

## 十七、工作流偏好写入后进程内缓存不主动失效

### 17.1 缓存的位置和范围

在 [get-preferences.usecase.ts#L304-L329](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-preferences/get-preferences.usecase.ts#L304-L329) 中，`WORKFLOW_RESOURCE` 和 `USER_WORKFLOW` 这两层工作流偏好使用 `InMemoryLRUCacheService` 做进程内缓存：

```typescript
const workflowPreferences = await this.inMemoryLRUCacheService.get(
  InMemoryLRUCacheStore.WORKFLOW_PREFERENCES,
  `${command.environmentId}:${command.templateId}`,  // key = environmentId:templateId
  async (): Promise<[PreferencesEntity | null, PreferencesEntity | null]> => {
    // 从数据库查询 WORKFLOW_RESOURCE + USER_WORKFLOW
  },
  cacheOptions
);
```

**缓存特性**（来自 [in-memory-lru-cache.store.ts#L84-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.store.ts#L84-L88)）：
- 缓存 key：`environmentId:templateId`
- 存储内容：`[workflowResourcePref, workflowUserPref]` 二元组
- TTL：**1 分钟**
- 容量：最多 1000 条
- Feature Flag：`IS_LRU_CACHE_ENABLED`

### 17.2 问题：写入后没有主动失效

关键发现：**`UpsertPreferences` 的所有 upsert 方法（包括 `upsertUserWorkflowPreferences` 和 `upsertWorkflowPreferences`）都没有调用 `inMemoryLRUCacheService.invalidate(WORKFLOW_PREFERENCES, key)`。**

```typescript
// upsert-preferences.usecase.ts
public async upsertUserWorkflowPreferences(command: UpsertUserWorkflowPreferencesCommand) {
  const result = await this.upsert({ ... });  // ← 只写 DB，不失效缓存
  return result as WorkflowPreferencesFull;
}
```

这意味着：

1. 管理员在 Dashboard 上修改了工作流偏好（`USER_WORKFLOW`）
2. 写入数据库成功
3. **但所有进程内的缓存仍然是旧值**
4. 最多需要等 **1 分钟**（TTL）之后，新的偏好才会体现在发送决策中

**影响场景**：
- 多进程部署时，只有执行写入操作的那个进程可能立即感知到变化（但代码也没有主动失效它自己的缓存），其他进程全靠 TTL 自然过期
- 管理员刚改完偏好就立刻触发通知，可能仍然使用旧的偏好设置
- 对于 critical 状态切换尤其敏感：如果管理员把一个通知设为 critical，可能 1 分钟内发出的消息仍然遵循旧的非 critical 路径

### 17.3 订阅者层偏好没有缓存

注意缓存仅覆盖 **WORKFLOW_RESOURCE 和 USER_WORKFLOW** 这两层工作流偏好。`SUBSCRIBER_GLOBAL` 和 `SUBSCRIBER_WORKFLOW` 两层**不缓存**，每次都查数据库。所以用户修改个人偏好不会有缓存不一致问题。

### 17.4 现有的失效机制

`InMemoryLRUCacheService` 提供了 `invalidate()` 方法（[in-memory-lru-cache.service.ts#L82-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.service.ts#L82-L93)），支持精确 key 匹配和前缀匹配（`key:v:*`），但在整个 `upsert-preferences.usecase.ts` 中**没有被调用过**。

---

## 十八、数据落库唯一索引依据 — `contextKeysHash` 而非 `contextKeys` 数组本身

### 18.1 为什么不能直接用数组做唯一索引

MongoDB 的唯一索引对数组字段的行为是**多键索引**——如果字段是数组 `["a", "b"]`，索引会为每个数组元素单独创建一条索引条目，这意味着：
- 记录 A `{ contextKeys: ["a", "b"] }` 和记录 B `{ contextKeys: ["a"] }` 会冲突（都有 `"a"`）
- 这不是我们想要的行为

### 18.2 解决方案：`contextKeysHash` 哈希字段

[preferences.schema.ts#L106-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/preferences/preferences.schema.ts#L106-L132) 在 `pre('save')` 和 `pre('insertMany')` 钩子中自动计算并写入 `contextKeysHash` 字段：

```typescript
function generateContextKeysHash(contextKeys: string[] | undefined): string {
  if (!contextKeys || contextKeys.length === 0) {
    return 'DEFAULT_CONTEXT';
  }
  const sorted = [...contextKeys].sort();
  return createHash('sha256').update(JSON.stringify(sorted)).digest('hex').substring(0, 16);
}

preferencesSchema.pre('save', function (next) {
  if (shouldApplyContextKeysHash(this.type)) {
    this.contextKeysHash = generateContextKeysHash(this.contextKeys);
  }
  next();
});
```

**关键点**：
1. 排序后哈希 → `["a", "b"]` 和 `["b", "a"]` 生成相同的 hash
2. 空数组或 undefined → 统一哈希为 `"DEFAULT_CONTEXT"`
3. 仅对三类 context 敏感的类型计算 hash：`SUBSCRIBER_GLOBAL`、`SUBSCRIBER_WORKFLOW`、`SUBSCRIPTION_SUBSCRIBER_WORKFLOW`

### 18.3 四张唯一索引（按类型隔离）

[preferences.schema.ts#L134-L214](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/preferences/preferences.schema.ts#L134-L214) 使用 `partialFilterExpression` 按类型分四张独立的唯一索引，这就是**类型隔离**机制：

| 索引类型 | 唯一键字段 | partialFilterExpression |
|---------|-----------|------------------------|
| SUBSCRIBER_GLOBAL | `{ _environmentId, _subscriberId, type, contextKeysHash }` | `{ type: SUBSCRIBER_GLOBAL, contextKeysHash: { $exists: true } }` |
| SUBSCRIBER_WORKFLOW | `{ _environmentId, _subscriberId, _templateId, type, contextKeysHash }` | `{ type: SUBSCRIBER_WORKFLOW, contextKeysHash: { $exists: true } }` |
| SUBSCRIPTION_SUBSCRIBER_WORKFLOW | `{ _environmentId, _subscriberId, _topicSubscriptionId, _templateId, type, contextKeysHash }` | `{ type: SUBSCRIPTION_SUBSCRIBER_WORKFLOW, contextKeysHash: { $exists: true } }` |
| WORKFLOW 层（USER_WORKFLOW + WORKFLOW_RESOURCE） | `{ _environmentId, _templateId, type }` | `{ type: { $in: [USER_WORKFLOW, WORKFLOW_RESOURCE] } }` |

### 18.4 类型隔离如何避免索引冲突

每个唯一索引只对特定 `type` 的文档生效。这意味着：

- 一条 `SUBSCRIBER_GLOBAL` 记录和一条 `SUBSCRIBER_WORKFLOW` 记录可以有完全相同的 `(_environmentId, _subscriberId)` 值，不会冲突——因为它们命中的是不同的索引
- 一条 `USER_WORKFLOW` 和一条 `WORKFLOW_RESOURCE` 可以有相同的 `(_environmentId, _templateId)`，但因为 `type` 不同，也不会冲突

这就是"**如何通过类型隔离来避免唯一索引冲突**"——表面上是同一张表，实际上通过 `partialFilterExpression` 分成了 4 张逻辑上独立的唯一约束空间。

### 18.5 `partialFilterExpression` 的意义

没有 `partialFilterExpression` 的话，索引会对所有文档生效，那么：
- 一条 `SUBSCRIBER_GLOBAL` 记录（没有 `_templateId`）和一条 `SUBSCRIBER_WORKFLOW` 记录（有 `_templateId`）会因为 `_templateId = null` 和 `_templateId = ObjectId("xxx")` 而不冲突，但如果两者都没有 `_templateId` 就可能冲突
- `partialFilterExpression` 确保索引只对特定类型的文档生效，从根本上隔离了不同类型的键空间

---

## 十九、条件表达式错误或返回值不对的备用链

### 19.1 三层备用链结构

[evaluateSubscriptionPreferences](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/subscriber-job-bound/subscriber-job-bound.usecase.ts#L430-L535) 的设计中有三层备用链（fail-safe），目标是在异常情况下**偏向放行**（默认通过），而不是默默关闭订阅。

```
evaluateSubscriptionPreferences
  ├─ 外层 try-catch（L437）
  │    └─ catch → return { result: true }
  │
  └─ if (subscriptionPreference)
        └─ evaluatePreferenceCondition
              ├─ 条件求值 try-catch（L502）
              │    └─ catch → return false （注意：这里是 false！）
              │
              ├─ 返回值类型检查（L505）
              │    └─ typeof !== boolean → return false
              │
              └─ no condition → enabled 判断 → undefined/null → return true
```

### 19.2 内层：`evaluatePreferenceCondition` 的备用链

```typescript
// subscriber-job-bound.usecase.ts L495-L535
private async evaluatePreferenceCondition(
  preferences: WorkflowPreferencesPartial,
  payload: Record<string, unknown>
): Promise<boolean> {
  const condition = preferences.all?.condition;

  // 第 1 层：有 condition 表达式
  if (condition !== undefined && condition !== null) {
    try {
      const result = jsonLogic.apply(condition as RulesLogic, { payload });

      // 返回值不是 boolean → 警告 + 返回 false
      if (typeof result !== 'boolean') {
        this.logger.warn(...);
        return false;  // ← 可能导致订阅被默默关闭
      }

      return result;
    } catch (error) {
      // 表达式求值异常（语法错误、引用不存在字段等） → 错误 + 返回 false
      this.logger.error(...);
      return false;  // ← 可能导致订阅被默默关闭
    }
  }

  // 第 2 层：没有 condition 时看 enabled 字段
  const enabled = preferences.all?.enabled;

  if (enabled === undefined || enabled === null) {
    return true;  // ← 没有明确设定时，默认放行
  }

  return enabled;
}
```

### 19.3 外层：`evaluateSubscriptionPreferences` 的备用链

```typescript
// subscriber-job-bound.usecase.ts L430-L493
try {
  // ... 查询 subscriptionPreference ...

  if (subscriptionPreference) {
    const passes = await this.evaluatePreferenceCondition(...);
    if (!passes) {
      return { result: false, ... };  // ← 被内层返回 false 时，走到这里
    }
    return { result: true, ... };
  }

  // 没有 subscriptionPreference 记录 → 默认放行
  return { result: true, subscriptionIdentifier };
} catch (error) {
  // 最外层兜底：任何未被捕获的异常（如数据库查询失败等）→ 放行
  this.logger.error(...);
  return { result: true, subscriptionIdentifier };
}
```

### 19.4 可能导致订阅被默默关闭的场景

**注意：只有内层两处路径会返回 `false`（关闭订阅），外层全部是放行导向。**

| 场景 | 行为 | 结果 |
|------|------|------|
| condition 语法错误（如无效 JSON Logic） | `jsonLogic.apply` 抛异常 → catch → `return false` | 订阅被关闭 |
| condition 返回非 boolean（如返回字符串/数字/对象） | `typeof result !== 'boolean'` → 警告 → `return false` | 订阅被关闭 |
| condition 求值正常，返回 `false` | 正常业务逻辑 → `return false` | 订阅被关闭 |
| condition 求值正常，返回 `true` | → `return true` | 订阅放行 |
| 没有 condition，但 `all.enabled = false` | → `return false` | 订阅被关闭 |
| 没有 condition，且 `all.enabled` 未设定 | → `return true` | 订阅放行 |
| 数据库查询 subscriptionPreference 抛异常 | 外层 catch → `return true` | 订阅放行 |
| 没有找到 subscriptionPreference 记录 | → `return true` | 订阅放行 |

### 19.5 设计考量：平衡安全性与可用性

这个三层备用链的设计体现了一种权衡：
- 对**表达式错误/返回值错误**采取保守策略（`false`），避免错误配置的条件意外放行通知
- 对**系统级错误**（如数据库不可用）采取宽容策略（`true`），避免系统故障导致所有订阅失效

代价就是：条件表达式写错（语法错、返回类型错）会导致订阅被"默默关闭"——用户在 UI 上可能看到订阅是正常的，但实际上所有通知都被过滤掉了，而且只有日志中有 warn/error 记录，没有其他告警渠道。

---

## 二十、类型隔离避免唯一索引冲突的进一步说明

### 20.1 同一张表，多个唯一索引

`preferences` 集合只有一个物理表，但定义了 **4 个独立的唯一索引**，每个索引通过 `partialFilterExpression` 只作用于特定 `type` 的文档：

```
物理表 preferences
  ├─ 唯一索引 1（仅 SUBSCRIBER_GLOBAL）：键 = { env, subscriber, type, contextKeysHash }
  ├─ 唯一索引 2（仅 SUBSCRIBER_WORKFLOW）：键 = { env, subscriber, template, type, contextKeysHash }
  ├─ 唯一索引 3（仅 SUBSCRIPTION_...）：键 = { env, subscriber, topicSubscription, template, type, contextKeysHash }
  └─ 唯一索引 4（仅 WORKFLOW_RESOURCE + USER_WORKFLOW）：键 = { env, template, type }
```

### 20.2 为什么需要 `type` 在键中

即使有 `partialFilterExpression` 过滤，键中仍然包含 `type` 字段。这是因为：
- 对于第 4 个索引（工作流层），同一个 `(env, template)` 组合下需要区分 `USER_WORKFLOW` 和 `WORKFLOW_RESOURCE` 两条独立记录
- 没有 `type` 在键中，这两条记录会冲突
- 其他三个索引的 `partialFilterExpression` 已经限定了单个 type，键中的 `type` 更多是形式上的一致性

### 20.3 各类型可共存的键空间示例

以下 5 条记录可以**同时存在于同一张表中**，不会触发任何唯一索引冲突：

| 记录 | type | _subscriberId | _templateId | _topicSubscriptionId | contextKeysHash | 命中的索引 |
|------|------|---------------|-------------|---------------------|----------------|-----------|
| ① | SUBSCRIBER_GLOBAL | S1 | (无) | (无) | DEFAULT_CONTEXT | 索引 1 |
| ② | SUBSCRIBER_WORKFLOW | S1 | T1 | (无) | DEFAULT_CONTEXT | 索引 2 |
| ③ | SUBSCRIPTION_SUBSCRIBER_WORKFLOW | S1 | T1 | TS1 | DEFAULT_CONTEXT | 索引 3 |
| ④ | USER_WORKFLOW | (无) | T1 | (无) | (不存在) | 索引 4 |
| ⑤ | WORKFLOW_RESOURCE | (无) | T1 | (无) | (不存在) | 索引 4 |

记录 ④ 和 ⑤ 命中同一个索引（索引 4），但因为键中的 `type` 不同（`USER_WORKFLOW` vs `WORKFLOW_RESOURCE`），所以不冲突。

### 20.4 为什么不分成 5 张物理表

当前设计的权衡：
- ✅ 单一集合便于查询（`GetPreferences` 只需查一个集合就能拿到所有层）
- ✅ 通过 `partialFilterExpression` 实现逻辑隔离，运维简单
- ❌ 索引结构复杂，需要理解 `partialFilterExpression` 的行为
- ❌ 不同类型的记录共享表空间，数据增长时可能需要考虑分片策略

---

## 二十一、关键代码索引

| 文件 | 作用 |
|------|------|
| [merge-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.usecase.ts) | 核心合并逻辑 |
| [merge-preferences.command.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.command.ts) | 合并命令定义 |
| [merge-preferences.spec.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/merge-preferences/merge-preferences.spec.ts) | 合并测试用例（覆盖所有组合） |
| [get-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-preferences/get-preferences.usecase.ts) | 从数据库收集 4 层偏好（含工作流层 LRU 缓存） |
| [buildWorkflowPreferences.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/utils/buildWorkflowPreferences.ts) | 部分偏好 → 完整偏好填充 |
| [preferences.const.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/consts/preferences/preferences.const.ts) | 默认偏好常量 |
| [workflow-channel-preferences.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/packages/shared/src/types/workflow-channel-preferences.ts) | 类型定义（PreferencesTypeEnum, Schedule 等） |
| [schedule-validator.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/schedule-validator.ts) | 勿扰时段判断 + 下次可用时间计算 |
| [run-job.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/run-job/run-job.usecase.ts) | Worker 侧 Schedule 检查入口 |
| [send-message.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/send-message/send-message.usecase.ts) | 渠道偏好评估（stepPreferred） |
| [upsert-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/upsert-preferences/upsert-preferences.usecase.ts) | 偏好写入（含全局偏好 channel 联动清理，**缺少缓存失效**） |
| [get-subscriber-template-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-template-preference/get-subscriber-template-preference.usecase.ts) | overridePreferences 叠加（含 WorkflowOverride） |
| [get-subscriber-schedule.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/usecases/get-subscriber-schedule/get-subscriber-schedule.usecase.ts) | 独立获取 Schedule |
| [base-repository.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/base-repository.ts#L74-L111) | buildContextExactMatchQuery 定义 |
| [subscriber-job-bound.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/worker/src/app/workflow/usecases/subscriber-job-bound/subscriber-job-bound.usecase.ts#L430-L535) | SUBSCRIPTION 级前置过滤 + condition 备用链 |
| [workflow-override.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/workflow-override/workflow-override.entity.ts) | 租户维度偏好覆盖实体 |
| [create-subscription-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscriptions/usecases/create-subscription-preferences/create-subscription-preferences.usecase.ts) | 订阅偏好初始化（excludeSubscriberPreferences） |
| [update-preferences.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts) | Inbox 偏好更新（含 subscription 级偏好路由） |
| [get-subscriber-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscribers/usecases/get-subscriber-preference/get-subscriber-preference.usecase.ts) | Dashboard 偏好列表（不含 WorkflowOverride） |
| [preferences.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/dal/src/repositories/preferences/preferences.schema.ts) | 数据落库 Schema + 4 张唯一索引（partialFilterExpression 类型隔离） |
| [in-memory-lru-cache.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.service.ts) | LRU 缓存服务（get / invalidate 方法） |
| [in-memory-lru-cache.store.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/services/in-memory-lru-cache/in-memory-lru-cache.store.ts) | LRU 缓存配置（WORKFLOW_PREFERENCES TTL = 1 分钟） |
| [deepmerge.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/libs/application-generic/src/utils/deepmerge.ts) | 写路径深合并工具（自研 fork 自 deepmerge） |
| [get-subscriber-global-preference.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/6-novu/apps/api/src/app/subscribers/usecases/get-subscriber-global-preference/get-subscriber-global-preference.usecase.ts) | 全局偏好查询入口之一（API 层，含活跃渠道过滤） |
