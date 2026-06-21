# Notification Feed 计数事件口径详解

> 本文聚焦「计数的口径到底是什么」：未读总数、严重等级分桶、延后和归档消息是否被计入、软删除如何被过滤、UNREAD 和 UNSEEN 事件为什么查不同的条件，以及哪些场景计数会暂时不一致、靠什么机制最终修正。

---

## 1. 计数来源的两条路径

客户端拿到计数有两条路径，口径**不完全相同**：

```
┌──────────────────────────────────────────────────────────────────────┐
│                         客户端计数获取方式                             │
│                                                                      │
│  ┌──────────────────────────┐   ┌───────────────────────────────────┐ │
│  │ Path A: HTTP /count      │   │ Path B: WebSocket 推送             │ │
│  │ (初始化 / 手动刷新)       │   │ (实时增量)                        │ │
│  │                          │   │                                   │ │
│  │ GET /inbox/notifications │   │ ws.on('unread_count_changed')    │ │
│  │ /count                   │   │ ws.on('unseen_count_changed')    │ │
│  │                          │   │                                   │ │
│  │ → NotificationsCount.    │   │ → ExternalServicesRoute           │ │
│  │   execute()              │   │   .sendUnreadCountChange()        │ │
│  │   (CachedQuery 缓存)     │   │   .sendUnseenCountChange()        │ │
│  └──────────────────────────┘   └───────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

两条路径最终都调用 `messageRepository.getCount()` 和 `getCountBySeverity()`，但**传入的查询参数不同**，这导致它们的数字可能对不上。

---

## 2. 核心方法：getCount 的完整口径定义

所有计数最终都汇聚到 [message.repository.ts#L437-L479](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L437-L479) 的 `getCount()`。

它通过 `getFilterQueryForMessage()` 构建 MongoDB 查询，**每一个字段的过滤规则都有明确的默认值**。完整口径定义在 [message.repository.ts#L154-L268](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L154-L268)：

| 字段 | 不传时的默认行为 | 显式传值时的行为 |
|------|-----------------|-----------------|
| `_environmentId` | 必传 | — |
| `_subscriberId` | 必传 | — |
| `channel` | 必传（Feed 中固定为 `in_app`） | — |
| `deleted` | **固定过滤**：`deleted: { $exists: false }` | 不可关闭（见第 3 节） |
| `seen` | `seen: { $in: [true, false] }`（不过滤，两种状态都算） | `seen: true/false` |
| `read` | `read: { $in: [true, false] }`（不过滤，两种状态都算） | `read: true/false` |
| `archived` | `archived: { $in: [true, false] }`（不过滤，**归档消息也算**） | `archived: true/false` |
| `snoozed` | 不加入任何条件（**延后消息也算，因为 snoozedUntil 字段可能存在或 null**） | `snoozed:true` → `snoozedUntil: { $ne: null }`<br>`snoozed:false` → `$or: [snoozedUntil: {$exists:false}, snoozedUntil: null]` |
| `severity` | 不过滤 | 按等级数组过滤，`NONE` 同时匹配 `severity:{$exists:false}` |
| `contextKeys` | 不过滤 | 精确匹配（$and + buildContextExactMatchQuery） |
| `feedId` | 不过滤 | 按 feed identifier → _feedId 过滤 |
| `tagGroups` | 不过滤 | CNF 格式 AND/OR 匹配 |
| `data` | 不过滤 | 扁平键值精确匹配 |

### 2.1 关键发现 1：archived 默认「不过滤」

**不传 `archived` 参数时，归档消息会被计入计数！**

这意味着：
- `getCount({read: false})` = 所有未读消息，**包括已归档的未读消息**
- 但正常业务中，归档操作级联 `read=true`，所以归档消息基本都是已读的，实际影响很小
- 如果用户**归档了一条未读消息**（通过绕过 UI 的 API 直接调用），它仍然会计入未读总数

### 2.2 关键发现 2：snoozed 默认「不过滤」

**不传 `snoozed` 参数时，延后消息也会被计入！**

`snoozed` 参数不参与「默认 $in」机制（没有默认值），所以当调用方不显式传 `snoozed` 时，查询中**不包含任何 snoozedUntil 条件**。

这意味着：
- `getCount({read: false})` = 未读消息 + **已延后的未读消息**
- 延后消息（哪怕是 1 年之后的）只要是 `read=false`，就会计入未读总数

### 2.3 关键发现 3：deleted 被双重过滤

软删除的 `deleted` 字段被**两层**过滤：
1. `getFilterQueryForMessage()` 显式加入 `deleted: { $exists: false }`
2. Mongoose schema 的 `pre('countDocuments')` 钩子再加入一次（[message.schema.ts#L185-L187](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.schema.ts#L185-L187)）

而且 `pre('find')` / `pre('findOne')` 钩子也会过滤，所以**所有查询**（包括列表查询）都默认只返回未删除的消息。

---

## 3. UNREAD 事件：总计数 vs Severity 分桶的口径差异

这是整个计数系统中**最容易产生困惑**的地方。在 [external-services-route.usecase.ts#L66-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/usecases/external-services-route/external-services-route.usecase.ts#L66-L85) 中：

```ts
// 未读总数（total）：只传了 read: false，没有传 snoozed/archived
const unreadCount = this.messageRepository.getCount(
  env, userId, 'in_app',
  { read: false },          // ← 注意：没有 snoozed: false！
  { limit: 101 },
  contextKeys
);

// Severity 分桶计数：传了 read: false 且 snoozed: false
const severityCounts = this.messageRepository.getCountBySeverity(
  env, userId, 'in_app',
  { read: false, snoozed: false },  // ← 注意：多了 snoozed: false
  { limit: 99 },
  contextKeys
);
```

同样的差异也出现在：
- [ws.gateway.ts#L231-L249](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/ws.gateway.ts#L231-L249) `sendUnreadCountToAllConnections()`
- [session.usecase.ts#L198-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/session/session.usecase.ts#L198-L218) Session 初始化时的 count

### 3.1 这意味着什么：数字不相等

假设某用户有：
- 5 条正常未读消息（high=2, medium=2, low=1）
- 3 条**延后的未读消息**（high=1, medium=2）

则 WS 推送的 UNREAD 数据为：

```json
{
  "unreadCount": 8,           // 5 + 3（延后消息被计入 total）
  "counts": {
    "total": 8,               // 同 unreadCount
    "severity": {
      "high": 2,              // 只算正常未读，延后的 high=1 被排除
      "medium": 2,            // 延后的 medium=2 被排除
      "low": 1,
      "none": 0
    }
  },
  "hasMore": false
}
```

**总未读数 = 8，但 severity.high + medium + low + none = 5 ≠ 8**。

这不是 bug，是**刻意的设计**：
- `total` 告诉用户「你总共有多少条待处理」（包括延后但仍属于"未读状态"的消息）
- `severity` 分桶告诉用户「**当前需要立即处理**」的消息有哪些（延后消息被隐藏，因为它们的提醒时间还没到）

### 3.2 HTTP /inbox/notifications/count 的口径

[NotificationsCount](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/notifications-count/notifications-count.usecase.ts#L17-L79) usecase 接收 `filters[]` 数组（每个 filter 是独立的查询条件），把 filter **原样透传**给 `getCount()`。

也就是说：HTTP API 的口径完全由**前端传入的 filter 决定**，服务端不做任何口径修正。

例如 Session 初始化时传入的 filter 是 `[{ read: false, snoozed: false }]`，所以返回的是「当前待立即处理的未读数」（不含延后消息）。

但如果前端直接调用 `/inbox/notifications/count?filters=[{"read":false}]`，返回的数字就会包含延后消息。

### 3.3 为什么 WS 推送的 total 和 severity 口径不一致

设计意图的逆向推导：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     UI 展示中的两个位置                               │
│                                                                     │
│   🔴 小红点 (total unreadCount)                                      │
│   显示在应用图标/Bell 角标                                            │
│   → 用户期望：所有「状态上属于未读」的消息都算                           │
│   → 包括延后消息（只是提醒时间未到，但仍是未读状态）                     │
│   → 口径：{read: false}                                             │
│                                                                     │
│                                                                     │
│   📂 Inbox 中的严重等级 Badge（severity bucket）                      │
│   显示在 Filter 按钮 / 筛选标签上                                      │
│   → 用户期望：打开 Inbox 立即能看到的消息                               │
│   → 延后消息会被「延后」过滤器隐藏，不算在当前视图内                     │
│   → 口径：{read: false, snoozed: false}                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

如果让 severity 也包含延后消息，用户会看到：
- 小红点显示 8
- Severity 聚合显示：high=3, medium=4, low=1（合计 8，数字对得上）
- 但打开 Inbox 只看到 5 条消息（延后的 3 条被默认过滤）
- 用户：「为什么显示有 8 条但只看到 5 条？」

为了避免这个困惑，Novu 选择了让**severity 分桶和用户实际看到的列表数量一致**，代价是 severity 之和不等于 total。

---

## 4. UNSEEN 事件：单一口径

UNSEEN 的计算在 [external-services-route.usecase.ts#L125-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/usecases/external-services-route/external-services-route.usecase.ts#L125-L132)：

```ts
const unseenCount = this.messageRepository.getCount(
  env, userId, 'in_app',
  { seen: false },   // ← 只有 seen: false，没有其他过滤
  { limit: 101 },
  contextKeys
);
```

UNSEEN 的口径非常简单：**所有 `seen=false` 的消息**，不管 read/archived/snoozed 状态。

这带来一个有趣的结论：
- 一条 `seen=false, read=false, snoozedUntil=明年` 的延后消息，会**同时计入 unseen 和 unread total**
- 但不会计入 severity 分桶

---

## 5. 各操作对两种计数的影响矩阵

下面的表格核准了**每一种操作**之后，UNREAD 总数、UNREAD severity、UNSEEN 三个数字的变化。

「操作」列为前端 API 触发的动作，「DB 字段变化」为后端级联写入，「计数影响」为**假设操作前该条消息在计数中的状态**：

| 操作 | DB 字段变化 | 对 unread total `{read:false}` 的影响 | 对 unread severity `{read:false, snoozed:false}` 的影响 | 对 unseen `{seen:false}` 的影响 | 投递的 WS 事件 |
|------|------------|--------------------------------------|------------------------------------------------------|-------------------------------|--------------|
| **新消息到达** | `seen:F, read:F, archived:F, snoozed:null` | +1（未读） | +1（未读且非延后） | +1（未见） | RECEIVED + UNSEEN + UNREAD |
| **新消息（延后到未来）** | `seen:T, read:F, archived:F, snoozedUntil=date` | +1（未读） | +0（延后被排除） | +0（已级联 seen=T） | UNREAD |
| **read 一条正常未读** | `read:T, seen:T, lastReadDate:now` | -1（read 变为 T） | -1 | +0（之前已 seen=T） | **UNREAD 只** |
| **read 一条未见消息**（绕过 UI 直接 API） | `read:T, seen:T` | -1 | -1（非延后） | -1（seen 从 F→T） | **UNREAD 只**（⚠️ 见第 6.1 节） |
| **read 一条延后未读** | `read:T, seen:T`（snoozedUntil 保留） | -1（read 变为 T） | +0（之前就不在 severity） | +0（延后已级联 seen=T） | **UNREAD 只** |
| **unread 一条已读** | `read:F, seen:T, archived:F` | +1（read 变为 F） | +1（非延后） | +0（seen 仍为 T） | UNREAD |
| **unread 一条已读延后消息** | `read:F, seen:T, archived:F`（snoozedUntil 保留） | +1 | +0（延后被排除） | +0 | UNREAD |
| **seen（标记已见）** | `seen:T`（其他不变） | +0（read 没变） | +0 | -1（seen 从 F→T） | **UNSEEN 只** |
| **unseen（标记未见）** | `seen:F, read:F, archived:F, firstSeenDate:null` | +0（read 仍为 F，只是之前也是 F） | +0 | +1（seen 从 T→F） | **UNSEEN 只** |
| **archive 一条未读** | `read:T, seen:T, archived:T, archivedAt:now` | -1（read 变为 T） | -1（非延后） | +0（已级联 seen=T） | UNREAD |
| **archive 一条已读** | `read:T, seen:T, archived:T, archivedAt:now` | +0（read 已是 T） | +0 | +0 | UNREAD |
| **unarchive 一条归档** | `read:T, seen:T, archived:F` | +0（read 仍为 T） | +0 | +0 | UNREAD |
| **snooze 一条正常未读** | `seen:T, archived:F, snoozedUntil=date`（read 不变） | +0（read 仍为 F） | -1（从非延后变为延后） | +0（已级联 seen=T） | UNREAD |
| **snooze 一条未见消息**（API 直接调） | `seen:T, archived:F, snoozedUntil=date`（read 不变） | +0 | -1 | -1（seen 从 F→T） | **UNREAD 只**（⚠️ 见第 6.2 节） |
| **unsnooze（用户手动取消）** | `seen:T, archived:F, snoozedUntil:null` | +0 | +1（从延后变回非延后） | +0 | UNREAD |
| **snoozedUntil 自然到期**（Worker 自动） | `snoozedUntil:null, read:F, createdAt:now`（[process-unsnooze-job.usecase.ts#L56-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/worker/src/app/workflow/usecases/process-unsnooze-job/process-unsnooze-job.usecase.ts#L56-L79)） | +0 | +1（从延后变回非延后） | +0（延后时 seen=T 保持） | **RECEIVED**（UNSEEN + UNREAD 会在 RECEIVED 处理中被触发） |
| **delete 一条未读** | 文档被硬删除（`deleteMessagesByIds` 用 `this.delete()`，非软删） | -1 | -1 | -1（若之前 seen=F） | UNREAD 只 |

### 5.1 观察：snooze/unsnooze 对 unread total 没有影响

延后和取消延后，`read` 字段保持不变，所以：
- **`unread total` 不变**（因为查询条件只看 `read:false`，不关心 snoozedUntil）
- **`unread severity` 变**（因为查询条件 `{read:false, snoozed:false}` 排除了延后消息）

这解释了为什么 snooze 操作要投 UNREAD：虽然 total 没变，但 severity 分桶变了。如果不投 UNREAD，前端 Badge 上的 severity 数字会和列表中实际可见的数量不一致。

### 5.2 观察：archive 操作对 unseen 没有影响

归档消息级联 `seen:true`，但**归档操作不会投 UNSEEN**。这在绝大多数场景下没问题（归档的消息之前已经是 seen=true），但通过 API 直接归档一条 `seen=false` 的消息时，unseen 计数会有偏差。

---

## 6. 仅靠后续全量重算修正的场景（最终一致性窗口）

以下场景中，计数会在操作后**短暂不准确**，但会在下一次全量重算时自动修正。

### 6.1 场景 A：直接通过 API read 一条 `seen=false` 的消息

```
T0: 消息 M = { seen:false, read:false }
    unseenCount = 1, unreadCount = 1

T1: 调用 POST /inbox/notifications/:id/read
    → 后端级联写入 {seen:true, read:true}
    → 只投 UNREAD
    → apps/ws 重算 unreadCount = 0 ✅
    → apps/ws 不重算 unseen（因为只投了 UNREAD）
    → 前端收到 unread_count_changed，但看不到 unseen_count_changed
    → 前端 unseenCount 仍显示为 1 ❌（实际应为 0）

T2: 新消息到达 / 用户手动刷新 / 下一次 seen 操作
    → RECEIVED 触发 UNSEEN 重算
    → unseenCount = 0 ✅ 自愈
```

**触发概率**：很低。正常 UI 流程中 VisibilityTracker 在用户看到消息时就已经把 seen 置为 true，到用户点击 read 时 seen 已经是 true。只有绕过 UI 直接调 API 才会触发。

**修正机制**：
- 下次任何 UNSEEN 事件到达（seenAll / markAsSeen / 新消息 RECEIVED）
- 或前端手动调用 `/inbox/notifications/count` 刷新

### 6.2 场景 B：直接通过 API snooze 一条 `seen=false` 的消息

```
T0: 消息 M = { seen:false, read:false, snoozedUntil:null }
    unseenCount = 1, unreadSeverity = 1

T1: 调用 POST /inbox/notifications/:id/snooze
    → 后端级联写入 {seen:true, archived:false, snoozedUntil:date}
    → 只投 UNREAD
    → apps/ws 重算 unreadSeverity = 0 ✅
    → apps/ws 不重算 unseen
    → 前端 unseenCount 仍显示为 1 ❌

T2: 自愈（同场景 A）
```

**触发概率**：更低。正常 UI 中用户能看到 snooze 按钮意味着消息已在视口中，VisibilityTracker 已经标记了 seen。

### 6.3 场景 C：archive 一条 `seen=false` 的消息

```
T0: 消息 M = { seen:false, read:false, archived:false }
    unseenCount = 1, unreadCount = 1

T1: 调用 POST /inbox/notifications/:id/archive
    → 后端级联写入 {seen:true, read:true, archived:true}
    → 只投 UNREAD
    → apps/ws 重算 unreadCount = 0 ✅
    → 前端 unseenCount 仍为 1 ❌

T2: 自愈
```

**触发概率**：低。归档在 UI 中通常是用户在消息列表中操作，此时消息已可见 → seen=true。

### 6.4 场景 D：UNREAD total 和 severity 分桶不一致（不是 bug）

如第 3 节所述，WS 推送的 `counts.total` 和 `counts.severity.*` 之和可能不相等：

```json
{
  "unreadCount": 8,
  "counts": {
    "total": 8,
    "severity": { "high": 2, "medium": 2, "low": 1, "none": 0 }  // 合计 5
  },
  "hasMore": false
}
```

**这不是临时不一致，而是长期有意的口径差异**。如果前端显示：
- 应用角标（小红点）：用 `unreadCount`（含延后）
- Inbox 内的 severity Badge：用 `counts.severity.*`（不含延后）

则完全没问题。如果前端把 severity 之和和 total 做等式校验，就会报错。

### 6.5 场景 E：自然到期 unsnooze 时 seen/read 被重置

[ProcessUnsnoozeJob](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/worker/src/app/workflow/usecases/process-unsnooze-job/process-unsnooze-job.usecase.ts#L56-L79) 在延后到期自动唤醒时，执行的不是「级联取消延后」，而是**显式写入 `read: false`**：

```ts
$set: {
  snoozedUntil: null,
  createdAt: nowDate,
  read: false,                // ← 强制置为未读
  lastReadDate: null,
  deliveredAt: { $concatArrays: [...] }
}
```

这意味着：
- 如果用户在延后之前**已经读过**这条消息（`read=true`），到期后会被**强制变回未读**
- 这是有意的设计：延后的目的是「稍后提醒我」，提醒到达时应该像新消息一样显示为未读
- 同时投的是 RECEIVED 事件（会触发 UNSEEN + UNREAD 重算），计数会正确

### 6.6 场景 F：WebSocket 消息不重试丢失

WebSocketWorker 是 **at-most-once** 语义：队列消息失败不重试。

```
T0: unreadCount = 5
T1: 用户 read 一条消息 → 后端 DB unreadCount 变 4
T2: WebSocketsQueueService 投递 UNREAD 到队列 ✅
T3: apps/ws 消费时网络异常 / 进程重启 / DB 瞬断 → 消费失败
T4: 队列配置 removeOnFail: true → 消息被丢弃
T5: 前端 unreadCount 仍显示 5 ❌（实际应为 4）

T6: 用户收到新消息 / 用户刷新页面 → 自愈
```

**修正机制**：
- 下一个任何 UNREAD 事件到达时重算
- 或前端下次调用 `/inbox/notifications/count`（初始化时 / 用户手动刷新）
- 或 Session 重新初始化（页面刷新 → count 重新拉取）

### 6.7 场景 G：HTTP 计数缓存失效不及时

`NotificationsCount` usecase 使用 `@CachedQuery` 注解，通过 `buildMessageCountKey()` 做缓存。任何写操作都会调用 `invalidateCache.invalidateQuery(buildMessageCountKey().invalidate(...))`。

如果缓存服务（Redis）瞬断导致 invalidate 失败：
```
T0: 缓存：unreadCount=5
T1: 用户 read → DB unreadCount=4
T2: invalidateCache 调用因 Redis 连接异常失败
T3: 下一次 HTTP GET /count 命中缓存 → 返回 5 ❌

T4: 缓存 TTL 到期（默认 5 分钟）或下一次 invalidate 成功 → 自愈
```

**修正机制**：
- WebSocket 推送不受 HTTP 缓存影响（WS 推送每次直接查库，不经过 `@CachedQuery`）
- 缓存 TTL 到期（通常 5-15 分钟）

### 6.8 场景 H：多 contextKeys 的 Inbox 之间的计数漂移

用户有两个 Inbox 实例，contextKeys 分别为 `["tenantA"]` 和 `["tenantB"]`：

```
T0: tenantA 有 3 条未读，tenantB 有 2 条未读
T1: 用户在 tenantA 的 Inbox 中 readAll
T2: 后端投 UNREAD（contextKeys=["tenantA"]）
T3: apps/ws → ExternalServicesRoute
T4: WSGateway.sendMessage(userId, UNREAD, {...}, ["tenantA"])
    → isExactMatch() 只匹配 tenantA 的 socket
    → tenantA 的计数重算为 0 ✅
    → tenantB 的计数保持 2 ✅

T5: 但如果 WS 推送的 contextKeys 在服务端处理时丢失（例如某个中间件没有正确传递）
    → 被当作 []，则可能推送给 contextKeys=[] 的 Inbox
    → tenantA 和 tenantB 的计数都不变 ❌
T6: 用户手动刷新 → 自愈
```

---

## 7. 计数修正触发源汇总

| 触发源 | 触发时机 | 修正的计数 | 备注 |
|--------|---------|-----------|------|
| **RECEIVED** | 新消息到达 / 延后到期唤醒 | UNSEEN + UNREAD | 唯一同时触发两种计数重算的事件 |
| **UNREAD 事件** | read / unread / archive / unarchive / snooze / unsnooze / delete / readAll / archiveAll / archiveAllRead / deleteAll | UNREAD 总计数 + severity 分桶 | 不含 UNSEEN |
| **UNSEEN 事件** | seen / seenAll / markAsSeen | UNSEEN 总计数 | 不含 UNREAD |
| **Session 初始化** | 页面刷新 / Inbox 重新挂载 | 所有计数（通过 `/inbox/session` 返回） | 走 HTTP + CachedQuery |
| **HTTP /count** | 前端主动刷新（`count()`） | 所有计数 | 走 HTTP + CachedQuery |
| **缓存 TTL 到期** | 5-15 分钟自然到期 | HTTP 缓存的计数 | 自然修正 |
| **浏览器标签页 BroadcastChannel** | 同浏览器其他标签页收到 WS 推送 | 所有计数 | 多标签同步 |

---

## 8. 计数查询的 MongoDB 索引

Message Schema 上为计数查询优化的复合索引（见 [message.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.schema.ts)）：

```js
{
  _subscriberId: 1,
  _environmentId: 1,
  channel: 1,
  seen: 1,
  read: 1,
  archived: 1,
  snoozedUntil: 1,
  severity: 1,
  createdAt: -1
}
```

这个索引覆盖了：
- `getCount({read:false})` — UNREAD total
- `getCount({read:false, snoozed:false})` — UNREAD severity
- `getCount({seen:false})` — UNSEEN
- `getCountBySeverity()` 的每个 severity 分支查询

---

## 9. 速查卡

```
┌─────────────────────────────────────────────────────────────────────┐
│                    计数口径速查卡                                     │
├──────────────────────────┬───────────────────────────────────────────┤
│ 未读总数（小红点）        │ {read: false}                             │
│ (unreadCount / total)    │ 不含 deleted，含 archived，含 snoozed     │
├──────────────────────────┼───────────────────────────────────────────┤
│ 严重等级分桶（Badge）     │ {read: false, snoozed: false}            │
│ (counts.severity.*)      │ 不含 deleted，含 archived，不含 snoozed   │
├──────────────────────────┼───────────────────────────────────────────┤
│ 未见总数                  │ {seen: false}                             │
│ (unseenCount)            │ 不含 deleted，含 archived，含 snoozed     │
├──────────────────────────┼───────────────────────────────────────────┤
│ 软删除过滤                │ deleted: { $exists: false }               │
│                          │ （getFilterQueryForMessage + pre hook 双重）│
├──────────────────────────┼───────────────────────────────────────────┤
│ 多 Inbox 隔离             │ contextKeys 精确匹配 ($and)               │
└──────────────────────────┴───────────────────────────────────────────┘

不一致场景 → 自愈触发：
  API 直接 read/snooze/archive seen=false 消息 → 下一次 UNSEEN / RECEIVED
  WS 消息消费失败（at-most-once）              → 下一次任何计数事件 / 刷新
  HTTP 缓存失效失败                            → 下次 WS 推送 / TTL 到期
  severity 之和 ≠ total                        → 有意的设计（不是不一致）
```
