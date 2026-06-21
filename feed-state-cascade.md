# Notification Feed 状态级联关系详解

> 本文聚焦「一个操作触发多个字段联动」的精确规则。逐操作核准 `seen`、`read`、`archived`、`snoozedUntil` 四个核心布尔/时间字段的变化，以及 `firstSeenDate`、`lastSeenDate`、`lastReadDate`、`archivedAt` 四个时间戳的联动写入。最后回答：**既然很多操作都会级联改多个字段，为什么计数事件（UNREAD / UNSEEN）仍然分开投递？**

---

## 1. 级联引擎：唯一的真相源

所有状态级联逻辑集中在 [message.repository.ts#L872-L967](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L872-L967) 的 `updateMessagesStatus()` 私有方法。

它的入口判断逻辑是 **if-else 优先级链**，按 `archived > read > seen > snoozedUntil` 的顺序，**只有一个分支生效**：

```
if (isUpdatingArchived)       → 走 archived 分支
else if (isUpdatingRead)      → 走 read 分支
else if (isUpdatingSeen)      → 走 seen 分支
else if (isUpdatingSnoozed)   → 走 snoozed 分支
```

这意味着：**一次调用只能触发一个级联分支**。如果调用方同时传了 `read=true` 和 `archived=true`，只有 `archived` 分支生效（因为优先级更高），`read` 的分支逻辑会被跳过——但 `archived` 分支自身已经包含了 `read=true` 的级联，所以结果仍然正确。

---

## 2. 逐操作级联表：后端 DB 实际写入

以下表格完全基于 [message.repository.ts#L892-L932](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L892-L932) 的代码逐行核准。

### 2.1 操作：标记已读（read = true）

**触发分支**: `isUpdatingRead` → read 分支（第 901-909 行）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `seen` | ✅ `true` | **级联**：已读隐含已见 |
| `lastSeenDate` | `new Date()` | 跟随 seen 的级联写入 |
| `read` | `true` | 主动设置 |
| `lastReadDate` | `new Date()` | 主动设置 |
| `archived` | `undefined`（不修改） | read=true 时不改 archived |
| `archivedAt` | `undefined`（不修改） | 同上 |
| `snoozedUntil` | 不涉及 | 该分支不操作此字段 |

**`firstSeenDate`**: 因为 `shouldMarkAsSeen = true`，会执行二次 update：对 `firstSeenDate: {$exists: false}` 的文档补写 `firstSeenDate = new Date()`。

**后端代码原文**：
```ts
// read 分支
updatePayload = {
  seen: true,
  lastSeenDate: new Date(),
  read,
  lastReadDate: read ? new Date() : null,
  archived: !read ? false : undefined,   // read=true → undefined（不改）
  archivedAt: !read ? null : undefined,   // read=true → undefined（不改）
};
```

### 2.2 操作：标记未读（read = false）

**触发分支**: `isUpdatingRead` → read 分支（第 901-909 行）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `seen` | ✅ `true` | **级联**：未读但仍「已见」（用户打开过通知中心，只是没点这条消息） |
| `lastSeenDate` | `new Date()` | 跟随 seen 的级联写入 |
| `read` | `false` | 主动设置 |
| `lastReadDate` | `null` | 清空已读时间 |
| `archived` | ✅ `false` | **级联**：未读消息不可能处于归档状态 |
| `archivedAt` | ✅ `null` | 清空归档时间 |
| `snoozedUntil` | 不涉及 | 该分支不操作此字段 |

**关键理解**：`unread` 不等于 `unseen`。Novu 的语义是：
- `seen=true, read=false` = "我看到了，但还没点进去"（**最常见**的未读态）
- `seen=false, read=false` = "我根本没打开通知中心"（全新的、还没展示给用户的消息）

### 2.3 操作：标记已见（seen = true）

**触发分支**: `isUpdatingSeen` → seen 分支（第 910-923 行）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `seen` | `true` | 主动设置 |
| `lastSeenDate` | `new Date()` | 主动设置 |
| `read` | `undefined`（不修改） | seen=true 不影响 read |
| `lastReadDate` | `undefined`（不修改） | 同上 |
| `archived` | `undefined`（不修改） | seen=true 不影响 archived |
| `archivedAt` | `undefined`（不修改） | 同上 |
| `snoozedUntil` | 不涉及 | 该分支不操作此字段 |

**`firstSeenDate`**: `shouldMarkAsSeen = true`，对 `firstSeenDate: {$exists: false}` 的文档补写。

**关键理解**：标记已见**不会级联改任何其他字段**。这是 seen 是最低优先级状态的体现——"看见了"不代表"读了"或"归档了"。

### 2.4 操作：标记未见（seen = false）

**触发分支**: `isUpdatingSeen` → seen 分支（第 910-923 行）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `seen` | `false` | 主动设置 |
| `lastSeenDate` | `null` | 清空 |
| `firstSeenDate` | ✅ `null` | **级联**：未见状态重置首次看见时间（第 921-923 行单独处理） |
| `read` | ✅ `false` | **级联**：连"见都没见"当然不可能是"已读" |
| `lastReadDate` | ✅ `null` | 清空 |
| `archived` | ✅ `false` | **级联**：未见消息不可能被归档 |
| `archivedAt` | ✅ `null` | 清空 |
| `snoozedUntil` | 不涉及 | 该分支不操作此字段 |

**关键理解**：`unseen` 是**级联重置面最广**的操作——它把 read 和 archived 一起清零，本质上回到"新消息"的初始态。这和 `unread`（只改 read，保留 seen=true）形成鲜明对比。

### 2.5 操作：标记归档（archived = true）

**触发分支**: `isUpdatingArchived` → archived 分支（第 892-900 行，最高优先级）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `seen` | ✅ `true` | **级联**：归档隐含已见 |
| `lastSeenDate` | `new Date()` | 跟随 seen 级联 |
| `read` | ✅ `true` | **级联**：归档隐含已读 |
| `lastReadDate` | `new Date()` | 跟随 read 级联 |
| `archived` | `true` | 主动设置 |
| `archivedAt` | `new Date()` | 主动设置 |
| `snoozedUntil` | 不涉及 | 该分支不操作此字段 |

**`firstSeenDate`**: `shouldMarkAsSeen = true`，补写 firstSeenDate。

### 2.6 操作：取消归档（archived = false）

**触发分支**: `isUpdatingArchived` → archived 分支（第 892-900 行）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `seen` | ✅ `true` | **级联**：取消归档 = 用户重新看到了这条消息 |
| `lastSeenDate` | `new Date()` | 跟随 seen 级联 |
| `read` | ✅ `true` | **级联**：取消归档隐含已读（用户主动操作了它） |
| `lastReadDate` | `new Date()` | 跟随 read 级联 |
| `archived` | `false` | 主动设置 |
| `archivedAt` | `null` | 清空归档时间 |
| `snoozedUntil` | 不涉及 | 该分支不操作此字段 |

**关键理解**：取消归档把消息放回"已读可见"状态，而不是"未读"。如果用户想重新标记为未读，需要额外调 `unread`。

### 2.7 操作：延后（snoozedUntil = date）

**触发分支**: `isUpdatingSnoozed` → snoozed 分支（第 924-932 行，最低优先级）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `snoozedUntil` | 传入的 Date | 主动设置 |
| `seen` | ✅ `true` | **级联**：延后 = 用户看到了并选择延后 |
| `lastSeenDate` | `new Date()` | 跟随 seen 级联 |
| `read` | `undefined`（不修改） | 延后不改 read |
| `lastReadDate` | `undefined`（不修改） | 同上 |
| `archived` | ✅ `false` | **级联**：延后消息取消归档（延后 ≠ 归档，消息仍在活跃列表等待唤醒） |
| `archivedAt` | ✅ `null` | 清空归档时间 |
| `firstSeenDate` | 补写 | `shouldMarkAsSeen = true` |

**额外机制**：[SnoozeNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/snooze-notification/snooze-notification.usecase.ts#L51-L94) usecase 在更新 DB 之外，还会：
1. 创建一个延迟 Job（`_parentId` 关联原 Job，`payload.unsnooze = true`）
2. 把 Job 入 `standardQueue`，delay = `snoozeUntil - now`
3. 到期后 Worker 自动执行 unsnooze

### 2.8 操作：取消延后（snoozedUntil = null）

**触发分支**: `isUpdatingSnoozed` → snoozed 分支（第 924-932 行）

| 字段 | 写入值 | 说明 |
|------|--------|------|
| `snoozedUntil` | `null` | 主动设置（清空延后时间） |
| `seen` | ✅ `true` | **级联** |
| `lastSeenDate` | `new Date()` | 跟随 seen 级联 |
| `read` | `undefined`（不修改） | 不改 read |
| `archived` | ✅ `false` | **级联**：取消延后也取消归档 |
| `archivedAt` | ✅ `null` | 清空 |

**额外机制**：[UnsnoozeNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/unsnooze-notification/unsnooze-notification.usecase.ts#L27-L103) 在事务中删除关联的 scheduled Job。

### 2.9 操作：删除

**不走 `updateMessagesStatus`**。走 [deleteMessagesByIds](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts#L37-L89) 直接从 MongoDB 删除文档，然后投 UNREAD 事件刷新计数。

---

## 3. 状态优先级图：级联方向的可视化

```
                    ┌────────────┐
                    │  archived  │  ← 最高优先级
                    │ (归档)     │
                    └─────┬──────┘
                          │ 级联写入
                    ┌─────▼──────┐
                    │    read    │  ← 第二优先级
                    │  (已读)    │
                    └─────┬──────┘
                          │ 级联写入
                    ┌─────▼──────┐
                    │    seen    │  ← 第三优先级
                    │  (已见)    │
                    └─────┬──────┘
                          │ 级联写入
                    ┌─────▼──────┐
                    │ snoozedUntil│ ← 最低优先级
                    │  (延后)    │
                    └────────────┘
```

**升级（向上）时级联**：archived=true 会连带 read=true + seen=true
**降级（向下）时重置**：seen=false 会连带 read=false + archived=false

具体来说：
- **向上升级**：优先级高的操作，会级联设置低优先级字段为 `true`
- **向下降级**：优先级低的操作设为 `false`，会级联清零高优先级字段
- **同级操作**：设为 `true` 不影响同级以上；设为 `false` 可能级联清零

但 `snoozedUntil` 比较特殊：
- 它不参与 seen/read 的升级链（延后不改 read）
- 但它**独立地级联设 seen=true + archived=false**（延后 = 看到了但稍后提醒，不是归档）

---

## 4. 前端乐观值 vs 后端级联：差异对照

前端乐观值在 [helpers.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/notifications/helpers.ts) 中定义。它**只覆盖了直接变更的字段和部分直觉级联**，并不完全匹配后端的级联规则。

| 操作 | 前端乐观值 | 后端实际级联 | 差异 |
|------|-----------|------------|------|
| **read** | `isRead:true, readAt:now, isArchived:false, archivedAt:undefined` | `seen:true, lastSeenDate:now, read:true, lastReadDate:now` | ❌ **前端缺 `isSeen:true`**；前端多了 `isArchived:false`（后端 read=true 时 undefined 不改） |
| **unread** | `isRead:false, readAt:null, isArchived:false, archivedAt:undefined` | `seen:true, lastSeenDate:now, read:false, lastReadDate:null, archived:false, archivedAt:null` | ❌ **前端缺 `isSeen:true`**；前端 `isArchived:false` 与后端一致 |
| **seen** | `isSeen:true` | `seen:true, lastSeenDate:now`（不改 read/archived） | ✅ 一致（seen=true 不级联其他字段） |
| **archive** | `isArchived:true, archivedAt:now, isRead:true, readAt:now` | `seen:true, lastSeenDate:now, read:true, lastReadDate:now, archived:true, archivedAt:now` | ❌ **前端缺 `isSeen:true`** |
| **unarchive** | `isArchived:false, archivedAt:null, isRead:true, readAt:now` | `seen:true, lastSeenDate:now, read:true, lastReadDate:now, archived:false, archivedAt:null` | ❌ **前端缺 `isSeen:true`** |
| **snooze** | `isSnoozed:true, snoozedUntil:value` | `snoozedUntil:value, seen:true, lastSeenDate:now, archived:false, archivedAt:null` | ❌ **前端缺 `isSeen:true` 和 `isArchived:false`** |
| **unsnooze** | `isSnoozed:false, snoozedUntil:null` | `snoozedUntil:null, seen:true, lastSeenDate:now, archived:false, archivedAt:null` | ❌ **前端缺 `isSeen:true` 和 `isArchived:false`** |

**核心差异**：前端乐观值**普遍缺少 `isSeen:true`**。这是因为前端认为 `seen` 只在用户"看到"通知时由 VisibilityTracker 标记，其他操作虽然后端会级联写入 `seen=true`，但前端在乐观更新阶段没有反映这一点。

**为什么这不影响用户体验？**
1. 乐观更新只影响列表中的 UI 展示（如已读/未读样式），`isSeen` 在 UI 上不直接反映
2. `pending` 后会立即发 HTTP，`resolved` 事件携带的是**服务端返回的真实 DTO**，此时 `isSeen:true` 已正确
3. 最差情况：乐观更新阶段消息仍然显示为"未见"样式（但 UI 上 `seen` 不影响视觉样式），几百毫秒后 HTTP 返回就纠正了

---

## 5. 计数事件拆分：为什么 UNREAD 和 UNSEEN 仍分开投递？

### 5.1 各操作投递的 WS 事件

| 操作 | 投递的 WS 事件 | Usecase 代码位置 |
|------|---------------|-----------------|
| read / unread / archive / unarchive / snooze / unsnooze | **UNREAD** | [mark-many-notifications-as.usecase.ts#L101-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts#L101-L109) |
| readAll / archiveAll / archiveAllRead | **UNREAD** | [update-all-notifications.usecase.ts#L94-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/update-all-notifications/update-all-notifications.usecase.ts#L94-L103) |
| seen / seenAll / markAsSeen | **UNSEEN** | [mark-notifications-as-seen.usecase.ts#L162-L171](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/mark-notifications-as-seen/mark-notifications-as-seen.usecase.ts#L162-L171) |
| delete / deleteAll | **UNREAD** | [delete-many-notifications.usecase.ts#L79-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts#L79-L88) |

### 5.2 关键问题：read 操作级联了 seen=true，为什么只投 UNREAD 而不投 UNSEEN？

这是很多人困惑的根源。答案是**基于 DB 字段变化的"净效果"分析**：

#### 场景推演

假设用户有 3 条消息：
```
消息 A: seen=false, read=false  (全新消息)
消息 B: seen=true,  read=false  (已见但未读)
消息 C: seen=true,  read=true   (已读)
```

**操作：read 消息 A**（从 `seen=false, read=false` → `seen=true, read=true`）

- `seen` 字段变化：false → true（**变化了！**）
- `read` 字段变化：false → true（**变化了！**）

如果只投 UNREAD：
- `apps/ws` 收到 UNREAD → 执行 `getCount({read:false})` → 返回 1（只剩消息 B 未读）✅ 正确
- 但 `unseenCount` 没有被刷新 → 前端仍显示 unseenCount=1（因为消息 A 的 seen 变化了）

**这难道不会导致 unseen 计数不准确吗？**

#### 答案：在实际使用中不会出问题，原因有三

**原因 1：seen 和 read 的触发时机天然分离**

在 Novu 的 Inbox UI 中：
1. 用户**打开通知中心** → VisibilityTracker 自动标记所有可见消息为 seen（投 UNSEEN）
2. 用户**点击某条消息** → 触发 read（投 UNREAD）

也就是说，消息 A 在用户打开通知中心时就已经被 VisibilityTracker 标记为 seen 了（步骤 1 投了 UNSEEN，unseenCount 已经正确变为 0）。等用户点击 read 时，`seen` 字段**已经是 true**，级联写入 `seen=true` 实际上是一个 no-op（值没变）。所以此时只投 UNREAD 就够了。

唯一的例外：如果 API 被直接调用（绕过 UI），read 一条 `seen=false` 的消息。此时 unseenCount 确实不会立即更新，但：
- 前端下次拉取列表时会拿到 `seen=true` 的最新状态
- 或者下一次任何 seen 操作触发 UNSEEN 时会修正

**原因 2：服务端计数是"全量重算"而非"增量"**

`apps/ws` 收到 UNREAD 事件后，执行的是 `getCount({read:false})` 全量查库，不是拿旧值减一。所以即使中间有 seen 的变化被遗漏，下次 UNSEEN 事件到来时会基于最新的 DB 状态重新计算，**具有自我修复能力**。

**原因 3：性能权衡——避免 2 次 DB 查询**

如果 read 操作同时投 UNREAD + UNSEEN，`apps/ws` 就需要：
- 执行 `getCount({read:false})` — 1 次 MongoDB 查询
- 执行 `getCount({seen:false})` — 1 次 MongoDB 查询
- 如果有 severity 聚合还要再 +4 次

对于 99% 的场景（seen 已经是 true 的消息被标记 read），UNSEEN 查询是**完全无效的**（结果不变）。所以 Novu 选择了"只在 seen 实际变化时投 UNSEEN"的策略。

### 5.3 完整决策表：什么操作投什么事件

```
操作是否改变了 read 字段？
├── YES → 投 UNREAD（重算未读数）
│   ├── read=true:   read 字段 false→true，影响未读数 ✅
│   ├── read=false:  read 字段 true→false，影响未读数 ✅
│   ├── archived=true:  read 级联变 true，影响未读数 ✅
│   ├── archived=false: read 级联变 true，影响未读数 ✅
│   ├── snoozedUntil:   read 不变，但 snooze 把消息移出"未读"视图 ✅
│   └── delete:         消息消失，无论之前 read 与否都影响计数 ✅
│
└── NO → 只改变了 seen 字段？
    ├── seen=true:  投 UNSEEN（重算未见数）
    └── seen=false: 投 UNSEEN（重算未见数）
```

**注意**：snooze/unsnooze 操作的 `read` 字段不变（`undefined`），但仍然投 UNREAD。原因是：
- snooze 把消息从"活跃列表"移入"延后列表"
- 从 UI 角度，延后消息**不出现在默认的未读视图中**
- 所以即使 `read` 没变，延后也会影响"当前可见的未读数"，需要刷新 UNREAD 计数

### 5.4 一种潜在的计数不一致场景（及其自愈）

```
时间线：
T0: 消息 M 的状态 = {seen:false, read:false}
T1: 用户通过 API（非 UI）直接调 read(M)
    → 后端级联写入 {seen:true, read:true}
    → 只投 UNREAD，不投 UNSEEN
    → 前端收到 UNREAD，更新 unreadCount（正确）
    → 前端不知道 seen 变了，unseenCount 仍显示 1（❌ 应该是 0）
T2: 新消息 N 到达
    → Worker 投 RECEIVED + UNSEEN + UNREAD
    → apps/ws 重算 unseenCount = getCount({seen:false}) = 0 ✅ 自愈！
```

这是一个**最终一致性**的设计：unseenCount 可能在短时间内不准确，但在下一个涉及 seen 的事件到来时自动修正。Novu 选择了这个权衡来避免每次 read 操作都多查一次 DB。

---

## 6. firstSeenDate 的特殊处理

`firstSeenDate` 和其他时间戳不同，它**只写入一次**，不应该被后续 seen 操作覆盖。

实现方式在 [message.repository.ts#L946-L963](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L946-L963)：

```ts
// 第一步：正常 update（不含 firstSeenDate）
await this.update(chunkQuery, { $set: updatePayload });

// 第二步：只对 firstSeenDate 不存在的文档补写
await this.update(
  { ...chunkQuery, firstSeenDate: { $exists: false } },
  { $set: { firstSeenDate: new Date() } }
);
```

这保证：
- 消息第一次被标记 seen 时，`firstSeenDate` 被写入
- 后续 `seen=true` 的级联不会再覆盖它
- `seen=false` 时 `firstSeenDate` 被清空为 `null`（第 921-923 行）

**为什么要分两步？** 因为 MongoDB 的 `$set` 是全量覆盖，无法在单个 update 中表达"如果字段不存在则设置，否则不变"的条件逻辑。通过两步 update（第二步用 `$exists: false` 过滤），实现了类似 `$setOnInsert` 的语义。

---

## 7. 旧版 API 的级联差异

Novu 存在两套状态更新 API：

### 新版 `/inbox/notifications/*` 路由
- 使用 `updateMessagesStatusByIds()` → `updateMessagesStatus()`
- 级联规则如上文所述，**完整级联**

### 旧版 `/widgets/messages/*` 路由（已 deprecated）
- 使用 `changeMessagesStatus()` → `getReadSeenUpdatePayload()`
- 级联规则在 [message.repository.ts#L539-L575](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L539-L575)

| markAs | read | seen | lastReadDate | lastSeenDate | archived? |
|--------|------|------|--------------|--------------|-----------|
| READ   | true | true | now | now | 不操作 |
| UNREAD | false | **true** | now | now | 不操作 |
| SEEN   | - | true | - | now | 不操作 |
| UNSEEN | - | false | - | now | 不操作 |

**关键差异**：旧版 UNREAD 时 seen 被设为 `true`（和新版一样），但**没有级联清空 archived**。这在新版中已修复（`read=false` 会级联 `archived=false`）。

---

## 8. 级联规则速查卡

```
┌─────────────────────────────────────────────────────────────────────┐
│                      状态级联速查卡                                  │
├─────────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│    操作      │   seen   │   read   │ archived │ snoozed  │ 投WS事件 │
├─────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ read=true   │   →true  │  =true   │   不变   │   不变   │  UNREAD  │
│ read=false  │   →true  │  =false  │  →false  │   不变   │  UNREAD  │
│ seen=true   │  =true   │   不变   │   不变   │   不变   │  UNSEEN  │
│ seen=false  │  =false  │  →false  │  →false  │   不变   │  UNSEEN  │
│ archived=T  │   →true  │  →true   │  =true   │   不变   │  UNREAD  │
│ archived=F  │   →true  │  →true   │  =false  │   不变   │  UNREAD  │
│ snooze=date │   →true  │   不变   │  →false  │  =date   │  UNREAD  │
│ unsnooze    │   →true  │   不变   │  →false  │  =null   │  UNREAD  │
│ delete      │    —     │    —     │    —     │    —     │  UNREAD  │
└─────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
  
图例: =主动设置  →级联设置  不变=undefined不修改  —=文档删除
```
