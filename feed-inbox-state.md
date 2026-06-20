# Notification Feed - Inbox 入口与客户端状态刷新全链路分析

> 本文是对 feed-read-state.md 的**纵深补充**：从「用户打开页面到看到小红点」的时序视角，拆解 SDK → API → 状态写回 → 计数广播 → 多端同步 的完整路径。

---

## 1. 整体架构：客户端四层联动

Novu Inbox 客户端状态刷新由 **4 层机制叠加** 实现，任意一层缺失都会导致状态不一致：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户浏览器（SDK 端）                            │
│                                                                     │
│  ┌─────────────────┐   ┌─────────────────┐   ┌───────────────────┐ │
│  │ Layer 4         │   │ Layer 3         │   │ Layer 2           │ │
│  │ 浏览器标签页     │──▶│ 事件总线 Emitter │──▶│ 乐观更新缓存      │ │
│  │ BroadcastChannel│   │ (mitt)          │   │ NotificationsCache│ │
│  │ (跨标签同步)    │◀──│                 │◀──│                   │ │
│  └─────────────────┘   └────────┬────────┘   └────────┬──────────┘ │
│                                 │                      │            │
│                                 ▼                      ▼            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Layer 1           HTTP + WebSocket (网络层)                    │  │
│  │ InboxService(HTTP)  +  Socket(Socket.io)                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
           │                                    │
           ▼ HTTP                               ▼ Socket.io
┌─────────────────────────────────────────────────────────────────────┐
│                     Novu 服务端（apps/api + apps/ws）                 │
│                                                                     │
│  ┌──────────────┐  ┌───────────────────┐  ┌──────────────────────┐  │
│  │ InboxController│  │ MarkMany / UpdateAll│  │ WebSocketsQueueService│ │
│  │ (路由)       │  │ (业务+持久化)      │  │ → BullMQ/SQS 队列    │  │
│  └──────┬───────┘  └──────┬────────────┘  └────────┬─────────────┘  │
│         │                 │                        │                │
│         ▼                 ▼                        ▼                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  MessageRepository (MongoDB)  +  缓存失效(buildMessageCountKey) │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│         │                                                           │
│         ▼                                                           │
│  ┌───────────────────┐   →    WS Gateway 推送（Socket.io emit）     │
│  │ apps/ws           │                                              │
│  └───────────────────┘                                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 阶段一：初始化（Session + 拉取初始状态）

### 2.1 客户端入口调用顺序

当前端代码调用 `novu.inbox.initialize()` 或 `<Inbox />` 组件挂载时，执行以下同步步骤：

**（1）创建 Novu 实例**
- 核心类: [novu.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/novu.ts)
- 内部实例化 5 个模块：
  - `InboxService`（HTTP 客户端，[inbox-service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/api/inbox-service.ts#L120-L685)）
  - `NovuEventEmitter`（基于 mitt 的事件总线，[novu-event-emitter.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/event-emitter/novu-event-emitter.ts#L4-L26)）
  - `Notifications`（通知操作 + 乐观更新，[notifications.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/notifications/notifications.ts#L48-L395)）
  - `NotificationsCache`（内存缓存 + 事件同步，[notifications-cache.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/cache/notifications-cache.ts#L129-L378)）
  - `Socket`（Socket.io 客户端，[socket.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/ws/socket.ts#L111-L232)）

**（2）Session 初始化（POST /inbox/session）**
```
InboxService.initializeSession()
    │  POST /inbox/session
    │  body: { applicationIdentifier, subscriberHash, subscriber, context }
    ▼
后端: InboxController.sessionInitialize() [inbox.controller.ts#L142-L156]
    └─► Session.usecase.execute()
         ├─ 查找 Environment 并验证 In-App Integration
         ├─ CreateOrUpdateSubscriber（有 HMAC 时限制 allowUpdate）
         ├─ 生成 subscriberJwt（audience=widget_user，包含 contextKeys）
         └─ 返回 { token, profile, applicationIdentifier }
    │
    ▼
客户端接收到 token:
    InboxService.#httpClient.setAuthorizationToken(token)
    Socket.#token = token（延迟到 connect() 时使用）
```

### 2.2 拉取初始数据：并行 HTTP 请求

Session 成功后，SDK 会按需发起 3 类 HTTP 请求（由 UI Hook `useNotifications` / `useCount` 触发）：

| 目的 | 方法 | HTTP 路由 | 后端 Usecase | 服务端缓存 |
|------|------|-----------|-------------|-----------|
| 拉消息列表 | `InboxService.fetchNotifications()` | `GET /inbox/notifications?limit=10&...` | `GetNotifications` | ❌ 无缓存（实时） |
| 拉计数 | `InboxService.count()` | `GET /inbox/notifications/count` | `NotificationsCount` | ❌ 无缓存（但 DB 走索引） |
| 拉偏好 | `InboxService.fetchPreferences()` | `GET /inbox/preferences` | `GetInboxPreferences` | - |

**列表查询后端实现**：[get-notifications.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/get-notifications/get-notifications.usecase.ts#L22-L100)
- 调用 `messageRepository.paginate()`
- 必带条件: `channel = IN_APP` + `_subscriberId` + `_environmentId` + `contextKeys`（精确匹配）
- 可选过滤: `read`、`seen`、`archived`、`snoozed`、`tags/tagGroups`、`data`、`severity`、时间范围
- 返回 `{data: [...], hasMore, filter}` — 注意 `filter` 被客户端原样回传用于缓存 key 匹配

**计数查询后端实现**：[notifications-count.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/notifications-count/notifications-count.usecase.ts)
- 支持传入 `filters` 数组（多个过滤条件批量查询），减少前端往返

### 2.3 建立 WebSocket 连接

当 `notifications.list()` 或 `socket.connect()` 被调用时：

```
Socket.connect()
    │
    ├─ 无 token 时自动 callWithSession → initializeSession()
    │
    ▼
Socket.#initializeSocket()
    ├─ io(wsUrl, { transports: ['websocket'], query: { token } })
    ├─ 绑定 3 个 Socket.io 事件:
    │   ├─ WebSocketEvent.RECEIVED  →  #notificationReceived()
    │   ├─ WebSocketEvent.UNSEEN    →  #unseenCountChanged()
    │   └─ WebSocketEvent.UNREAD    →  #unreadCountChanged()
    │
    └─ 事件转发到 Emitter:
         ├─ 'notifications.notification_received'  // 新消息
         ├─ 'notifications.unseen_count_changed'    // 未见数
         └─ 'notifications.unread_count_changed'    // 未读数（含 severity 聚合）
```

服务端建立连接鉴权: [ws.gateway.ts#L103-L144](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/ws.gateway.ts#L103-L144)
- 校验 JWT，解析出 `subscriber._id`
- 提取 JWT 中的 `contextKeys`，绑定到 `socket.data.contextKeys`
- Socket 加入 `subscriber._id` 对应的 Room

---

## 3. 阶段二：状态变更 — 单条通知操作

客户端点击「读 / 未读 / 归档 / 取消归档 / 延后」等操作时，触发 `read/unread/archive/unarchive/snooze/unsnooze` 共 6 条路径，它们**架构模式完全相同**，只是状态字段不同。

### 3.1 完整调用链（以 `read` 为例）

```
用户点击 "标记已读"
    │
    ▼
Notification.read() 或 Notifications.read({notificationId})
    │  [notifications.ts#L139-L149]
    ▼
helpers.read()   [helpers.ts#L20-L60]
    │
    ├─ ① 乐观值构造（getNotificationDetails）
    │    - 若调用方传入 notification 实例 → 直接用它 + 状态覆盖 {isRead: true, readAt: now, isArchived: false}
    │    - 若只传 notificationId → 无乐观值（UI 需等待服务端响应）
    │
    ├─ ② Emitter.emit('notification.read.pending', { data: optimisticValue })
    │
    ├─ ③ HTTP PATCH /inbox/notifications/:id/read
    │    │  [inbox-service.ts#L252-L254]
    │    │
    │    ▼
    │  InboxController.markNotificationAsRead()  [inbox.controller.ts#L245-L260]
    │      │  AuthGuard('subscriberJwt') 从 token 解析 subscriberSession
    │      ▼
    │  MarkNotificationAs.execute()  [mark-notification-as.usecase.ts#L22-L75]
    │      │
    │      ├─ findOneForInbox() 验证归属（防止越权读他人消息）
    │      ├─ 委托 MarkManyNotificationsAs（即使单条也走批量方法）
    │      │   │  [mark-many-notifications-as.usecase.ts#L38-L111]
    │      │   │
    │      │   ├─ messageRepository.updateMessagesStatusByIds({
    │      │   │    ids: [id], read: true  →  最终调用 updateMessagesStatus()
    │      │   │  })
    │      │   │  （联动规则: read=true 会连带 seen=true, 写 lastReadDate/firstSeenDate）
    │      │   │
    │      │   ├─ invalidateCache: buildMessageCountKey()
    │      │   ├─ 写 ClickHouse Trace: messageInteractionService.trace()
    │      │   ├─ 发送 Webhook: MESSAGE_READ
    │      │   │
    │      │   └─ 核心！投递 WS 队列:
    │      │      WebSocketsQueueService.add({
    │      │        event: WebSocketEventEnum.UNREAD,  // 注意这里只投 UNREAD
    │      │        userId, contextKeys
    │      │      })
    │      │
    │      └─ 重新查库返回更新后的 DTO
    │
    ├─ ④ HTTP 成功: Emitter.emit('notification.read.resolved', { data: updatedNotification })
    │   失败:   Emit error，前端可回滚
    │
    ▼
↓ 客户端本地缓存联动（自动，无需调用方关心）↓
NotificationsCache（构造时订阅了 updateEvents）
    │  监听 'notification.read.pending' / '.resolved'
    │  [notifications-cache.ts#L88-L99 + L140-L142]
    │
    ├─ 遍历所有缓存 key（即所有过滤条件的 list 结果）
    ├─ 对每条包含该 notification 的缓存:
    │   - updateEvents (read/unread/complete/revert/read_all): 替换为新 Notification
    │   - removeEvents (archive/snooze/delete/archive_all):   从列表移除
    │
    ├─ 对有变化的过滤条件聚合后:
    │   Emitter.emit('notifications.list.updated', { data: aggregatedList })
    │
    └─ UI Hook useNotifications() 监听 list.updated，
       自动 mutate 无限滚动状态
       [useNotifications.ts#L28-L40]
```

### 3.2 6 种操作 → 乐观值对照

| 操作 | 乐观值 `isRead` | 乐观值 `isSeen` | 乐观值 `isArchived` | 乐观值 `isSnoozed` | 后端仓库方法 | 投 WS 事件 |
|------|----------------|----------------|---------------------|--------------------|------------|-----------|
| **read** | ✅ true | - | ❌ false | - | `updateMessagesStatusByIds({read:true})` | UNREAD |
| **unread** | ❌ false | -（后端级联: seen=true） | ❌ false | - | `updateMessagesStatusByIds({read:false})` | UNREAD |
| **archive** | ✅ true | -（后端级联） | ✅ true | - | `updateMessagesStatusByIds({archived:true})` | UNREAD |
| **unarchive** | ✅ true | - | ❌ false | - | `updateMessagesStatusByIds({archived:false})` | UNREAD |
| **snooze** | - | -（后端级联: seen=true） | ❌ false | ✅ true | `updateMessagesStatusByIds({snoozedUntil})` | UNREAD |
| **unsnooze** | - | - | - | ❌ false | `updateMessagesStatusByIds({snoozedUntil:null})` | UNREAD |

**重要观察**：
1. 所有 6 种操作都投 **UNREAD** 事件（read/archive/snooze 都会影响未读数），但**没有**投 UNSEEN（因为这些操作不会改变 seen 状态）。
2. `seen` 的变更走的是另一条专用路径（第 4 章）。
3. 后端的「状态级联」在 [message.repository.ts#L892-L932](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L892-L932) 实现：如 `read=true` 自动让 `seen=true`，但**前端乐观值没有级联**（只依赖服务端返回的 DTO 做最终同步）。

---

## 4. 阶段三：已见（seen）的特殊路径 — 自动曝光追踪

`seen` 和 `read/archive` 完全不同：**99% 的 seen 标记不是用户主动点击，而是消息在视口中停留足够时间自动触发。**

### 4.1 IntersectionObserver 自动追踪

核心实现: [visibility-tracker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/notifications/visibility-tracker.ts#L21-L274)

UI 层通过 Hook 接入: [useNotificationVisibility.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/ui/helpers/useNotificationVisibility.ts#L5-L36)

```
Notification 列表项渲染
    │
    ▼
observeNotification(element, notificationId)
    │
    ├─ IntersectionObserver 进入 callback
    │   threshold=0.5（50% 面积可见才算）
    │
    ├─ pendingNotifications.set(id, Date.now())  // 记录进入视口的时间
    │
    ▼
1 秒定时器（checkAllElementsVisibility）轮询 + 回调双重保险
    │
    ├─ now - startTime >= visibilityDuration (默认 1000ms)
    │   → 加入 seenNotifications Set（本会话去重，关闭 Inbox 才清空）
    │   → 从 pendingNotifications 移除
    │
    ▼
加入 pendingBatch 集合，等待 batchDelay (默认 500ms)
    │
    ▼
processBatch() 按 maxBatchSize=20 分片
    │
    ▼
POST /inbox/notifications/seen
   body: { notificationIds: chunk[] }
```

### 4.2 后端 seen 路径

Controller: [inbox.controller.ts#L518-L536](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/inbox.controller.ts#L518-L536)

Usecase: [mark-notifications-as-seen.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/mark-notifications-as-seen/mark-notifications-as-seen.usecase.ts#L46-L172)

```
MarkNotificationsAsSeen.execute()
    │
    ├─ 分支 A: 传了 notificationIds[]（VisibilityTracker 路径）
    │   ├─ 50 条一个 chunk 调用 updateMessagesStatusByIds({seen:true})
    │   └─ analytics: method='by_ids'
    │
    ├─ 分支 B: 传了 tags/data 过滤器（用户主动"全部标记已见"）
    │   ├─ 调用 updateMessagesFromToStatus(from:{tagGroups/data}, to:{seen:true})
    │   └─ analytics: method='by_filters'
    │
    ├─ logTraces: ClickHouse 写入 message_seen 事件
    ├─ 发送 Webhook MESSAGE_SEEN
    │
    ├─ invalidateCache: buildMessageCountKey()
    │
    └─ 投递 WebSocketQueue: WebSocketEventEnum.UNSEEN ← 注意只投 UNSEEN
```

### 4.3 Seen 操作不触发 UNREAD 的设计意义

因为 `seen=true` 不改变 `read` 字段，所以不影响未读数。通过**拆分两个独立的 WS 事件**（UNSEEN / UNREAD），服务端只需要重算真正变化的维度，减少数据库查询压力。

---

## 5. 阶段四：批量操作路径

客户端调用 `readAll()` / `archiveAll()` / `archiveAllRead()` / `deleteAll()`。

### 5.1 客户端乐观更新 + 后端执行

以 `readAll()` 为例，完整路径见 [helpers.ts#L433-L474](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/notifications/helpers.ts#L433-L474):

```
Notifications.readAll({tags?, data?})
    │
    ├─ ① 从缓存拿已有的 UniqueNotifications（按 tags/data 匹配）
    │    构造乐观值: {isRead:true, readAt:now, isArchived:false}
    │
    ├─ ② emit('notifications.read_all.pending', {data: optimisticNotifications})
    │
    ├─ ③ POST /inbox/notifications/read
    │    body: { tags, data }
    │
    │    后端: UpdateAllNotifications.execute()
    │      [update-all-notifications.usecase.ts#L31-L104]
    │      │
    │      ├─ tags → tagGroups，data JSON.parse()
    │      ├─ updateMessagesFromToStatus(
    │      │    from: { tagGroups, data, archived: false, snoozed: false },
    │      │    to:   { read: true }
    │      │  )
    │      │  — 内部先查符合条件的 _id 列表，再分批 updateMessagesStatusByIds
    │      │
    │      ├─ sendWebhookEvents (MESSAGE_READ，100 条一批)
    │      ├─ invalidateCache
    │      └─ WebSocketQueue.add(UNREAD)
    │
    └─ ④ emit('notifications.read_all.resolved')
```

**后端 `updateMessagesFromToStatus` 是通用的 from→to 转换机**：
- readAll: `from:{现有条件}` → `to:{read:true}`
- archiveAll: `from:{现有条件}` → `to:{archived:true, read:true}`（隐含）
- archiveAllRead: `from:{read:true, 现有条件}` → `to:{archived:true}`
- deleteAll: `from:{现有条件}` → 硬删除 / 软删除（根据实现）

### 5.2 四类批量接口对照

| 方法 | HTTP 路由 | 后端 Usecase | from 条件 | to 状态 | WS 事件 |
|------|-----------|-------------|----------|---------|---------|
| `readAll()` | `POST /inbox/notifications/read` | `UpdateAllNotifications` | tags, data | `{read:true}` | UNREAD |
| `archiveAll()` | `POST /inbox/notifications/archive` | `UpdateAllNotifications` | tags, data | `{archived:true}` | UNREAD |
| `archiveAllRead()` | `POST /inbox/notifications/read-archive` | `UpdateAllNotifications` | tags, data, `read:true` | `{archived:true}` | UNREAD |
| `seenAll()` / `markAsSeen()` | `POST /inbox/notifications/seen` | `MarkNotificationsAsSeen` | notificationIds 或 tags, data | `{seen:true}` | UNSEEN |
| `deleteAll()` | `POST /inbox/notifications/delete` | `DeleteAllNotifications` | tags, data | （删除） | UNREAD + UNSEEN |

---

## 6. 阶段五：计数刷新 & WebSocket 广播链路

### 6.1 计数的三种获取方式

客户端计数并非单一来源，而是 **3 路叠加 + 优先降级**：

```
                    ┌──────────────────────────────┐
                    │ 计数刷新优先级（从高到低）      │
                    └──────────────────────────────┘
                                    │
    ┌───────────────┬───────────────┼───────────────┐
    ▼               ▼               ▼               ▼
  WS 推送        乐观更新        HTTP 兜底         首次拉取
(实时)          (本地瞬时)      (延迟一致性)      (初始值)
    │               │               │               │
    │               │               │               └─ Notifications.count()
    │               │               │                  GET /inbox/notifications/count
    │               │               │                  → NotificationsCount.usecase
    │               │               │                  → messageRepository.getCount()
    │               │               │
    │               │               └─ 以下任一条件触发重新请求:
    │               │                  - useCount 组件 mount
    │               │                  - useCount 的 filter 变化
    │               │                  - WebSocket 断连重连后
    │               │
    │               └─ 操作 helpers 中已包含在「乐观值构造」内:
    │                  e.g. readAll 时 pending 阶段本地 +N/-N
    │                  （但 count Hook 实际不依赖 Emitter 直接更新，
    │                   而是依赖 WS 推送的权威值）
    │
    └─ 权威来源!（下一节详细展开）
```

### 6.2 WebSocket 计数事件：服务端重算 → 精确推送

当任何写操作调用 `WebSocketsQueueService.add()` 后，经过 BullMQ/SQS 队列由 `apps/ws` 消费：

```
WebSocketsQueueService.add()   [web-sockets-queue.service.ts#L16-L110]
    │
    ├─ [Enterprise 优先] socketWorkerService.sendMessage()
    │   直接发 Cloudflare Durable Object → 立刻推送（不走队列）
    │
    ├─ [通用/社区版] 投递到 BullMQ WEB_SOCKETS 队列
    │
    ▼
WebSocketWorker (apps/ws) 消费 at-most-once（失败不重试）
    │  [web-socket.worker.ts#L23-L108]
    ▼
ExternalServicesRoute.execute()  [external-services-route.usecase.ts#L11-L157]
    │
    ├─ 分支 A: event = RECEIVED（新消息）
    │   ├─ 如果 payload 只带 messageId → 再查 DB 拿完整 Message
    │   ├─ wsGateway.sendMessage(message) → 推完整消息给前端
    │   └─ 额外追加: 重算 unseen + unread 并推送
    │
    ├─ 分支 B: event = UNSEEN（标记已见/新消息引起）
    │   ├─ messageRepository.getCount({seen:false}, limit=101)
    │   │   → 返回 unseenCount（最多 100，>100 就 hasMore=true）
    │   └─ wsGateway.sendUnseenCountToAllConnections()
    │      ▼
    │      对该用户的每个活跃 socket:
    │        - 按 socket 自己的 contextKeys 单独重算
    │        - socket.emit('unseen_count_changed', {unseenCount, hasMore})
    │
    └─ 分支 C: event = UNREAD（标记读/归档/延后）
        ├─ 普通计数: getCount({read:false})
        ├─ 严重等级聚合: getCountBySeverity({read:false})
        │  → counts = { total: N, severity: { high, medium, low, none } }
        └─ wsGateway.sendUnreadCountToAllConnections()
           ▼
           对每个 socket:
             按自己的 contextKeys 单独重算
             socket.emit('unread_count_changed', {counts, hasMore})
```

**contextKeys 隔离关键代码**: [ws.gateway.ts#L203-L214](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/ws.gateway.ts#L203-L214) `isExactMatch()`
- 用户 A 有两个 Inbox 实例：实例 X 的 `contextKeys=["tenant1"]`，实例 Y 的 `contextKeys=["tenant2"]`
- 当 WebSocket 推送计数时，会为 X、Y **各自独立查一次库**，保证 tenant1 和 tenant2 的未读数不会串

### 6.3 前端接收计数并广播到所有标签页

Socket 收到事件后，通过 Hook `useWebSocketEvent` 分发到 UI，并通过 **BroadcastChannel** 同步到同浏览器的其他标签页:

Hook: [useWebSocketEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/js/src/ui/helpers/useWebSocketEvent.ts#L6-L45)

```
useWebSocketEvent({
  event: 'notifications.unread_count_changed',
  eventHandler: (data) => updateUI(data.counts.total)
})
    │
    ├─ 为避免同一用户在同一浏览器开 N 个标签页时，N 个 WebSocket 都重连一次造成风暴，
    │   使用了 requestLock()（基于 Web Locks API）：
    │   只有拿到锁的标签页才真正连接 WebSocket + 订阅 Emitter 事件
    │
    ├─ 建立 BroadcastChannel:
    │   channelName = `nv_ws_connection:a=应用ID:s=订阅者:c=contextKey:e=事件名`
    │
    ├─ 事件转发流程:
    │   WebSocket 消息 → Emitter → 拿到锁的标签页
    │                           │
    │                           ├─ ① 在本页执行 eventHandler（更新本页 UI）
    │                           └─ ② BroadcastChannel.postMessage(data)
    │                                    │
    │                                    ▼
    │                              其他所有标签页
    │                              监听 onmessage
    │                              也执行 eventHandler
    │                              （但它们没有 WebSocket 连接）
    │
    └─ 这样：10 个标签页共享 1 个 WebSocket 连接，
       计数通过 BroadcastChannel 分发给全部标签页
```

### 6.4 新消息到达时的 UI 刷新流程

```
服务端收到新 Message → WebSocketsQueueService.add(RECEIVED)
    │
    ▼
apps/ws RECEIVED 分支处理（见 6.2）
    │
    ├─ ① 发送完整 Message 给前端 Socket
    │
    ▼
前端 Socket.#notificationReceived()
    │  [socket.ts#L142-L146]
    ▼
将 WS payload 转成 InboxNotification（mapToNotification）
    │
    ▼
Emitter.emit('notifications.notification_received', {
  result: new Notification(...)
})
    │
    ├─ useWebSocketEvent 同样用锁 + BroadcastChannel 分发
    │
    └─ UI 组件（如 InboxTabs / NotificationList）监听该事件
       │
       └─ NotificationsCache.unshift(filter, notification)
          │  插入到内存缓存的最前端
          └─ emit('notifications.list.updated')
             → useNotifications 的无限滚动状态自动 mutate
```

---

## 7. 事件汇总表：所有 Emitter 事件 & 触发时机

前端 SDK 内部使用了大量事件（基于 mitt），以下是完整索引：

| 事件名 | 触发时机 | payload.data | 谁在监听 |
|--------|---------|-------------|---------|
| **socket.connect.pending** | WebSocket 开始连接前 | args | - |
| **socket.connect.resolved** | WebSocket 连接成功/失败 | args + error? | - |
| **notifications.list.pending** | list() 发请求前 | （可能带缓存数据） | - |
| **notifications.list.resolved** | list() 成功/失败 | ListNotificationsResponse \| error | - |
| **notifications.list.updated** | 缓存因任何操作被变更后 | 聚合后的 ListNotificationsResponse | `useNotifications` hook 自动 mutate |
| **notifications.count.pending** | count() 发请求前 | args | - |
| **notifications.count.resolved** | count() 成功/失败 | count data \| error | - |
| **notification.X.pending** (X=read/unread/seen/archive/snooze/unsnooze/delete/complete_action/revert_action) | 单个操作发 HTTP 前 | 乐观值 Notification? | `NotificationsCache`（更新/移除缓存） |
| **notification.X.resolved** | 单个操作 HTTP 返回 | 最终 Notification \| error | `NotificationsCache` |
| **notifications.X_all.pending** (X=read/seen/archive/archive_read/delete) | 批量操作发 HTTP 前 | optimisticNotifications[] | `NotificationsCache` |
| **notifications.X_all.resolved** | 批量操作 HTTP 返回 | optimisticNotifications[] \| error | `NotificationsCache` |
| **notifications.notification_received** | WS 推送新消息 | Notification（包装对象） | UI Hook + Cache.unshift |
| **notifications.unseen_count_changed** | WS 推送未见数变化 | unseenCount (number) | 计数 UI + 标签页广播 |
| **notifications.unread_count_changed** | WS 推送未读数变化 | `{total, severity:{...}}` | 计数 UI（小红点） + 标签页广播 |

---

## 8. 关键设计洞察

### 8.1 客户端「3 级一致性」为什么这样设计？
1. **乐观更新**（pending 级）：保证 UI 0 延迟，用户体验的下限
2. **本地缓存广播**（list.updated 级）：避免重新拉 HTTP，无限滚动状态不丢失
3. **WebSocket 权威计数**（unread/unseen_count_changed 级）：解决批量更新、跨设备、跨标签页「离线变更」等乐观值无法覆盖的场景

三者互补：乐观更新负责「瞬时手感」，WS 推送负责「长期正确」，HTTP 轮询兜底。

### 8.2 为什么 read/unread/archive 走 UNREAD，seen 单独走 UNSEEN？
- read/archive/snooze 都会引起 `read` 字段变化，**不改变** `seen`
- seen 反之，只改 `seen`，不改 `read`
- 服务端每个计数都要查一次 MongoDB（getCount + getCountBySeverity），拆成两个事件可以**只算真的变化的那一半**，节省 50% DB 压力

### 8.3 WebSocket + BroadcastChannel + Web Locks = 连接风暴防护
Novu 是第三方组件，一个应用里可能出现多个 Inbox 实例 + 多个标签页。如果不做防护：
- 同浏览器 5 个标签页 × 2 个 Inbox = 10 个 WebSocket 连接
- 这 10 个连接收到同样的 UNREAD 事件后各自重算 UI 造成重复渲染

引入「Web Locks 单连接 + BroadcastChannel 扇出」后，实际只需 **1 个 WebSocket 连接**，UI 刷新由内存广播完成。

### 8.4 Seen 自动追踪为什么不直接用 IntersectionObserver 的 duration？
原生 IntersectionObserver v2 支持 `trackVisibility: true` + `delay`，但兼容性差（2024 年前 Safari 不支持）。Novu 采用了「IO 只算进/出视口 + 1s setInterval 自计时」方案，兼容所有主流浏览器。

同时用了 `pendingNotifications`（计时中）+ `pendingBatch`（凑批量）+ `seenNotifications`（会话级去重）三重集合，保证：
- 滚出视口的消息不会被误标记
- 不会一次发 100 个 HTTP（batch 合并）
- 会话内不会重复给同一条消息发 HTTP（避免后端空更新浪费）

### 8.5 Context Compatibility：新旧 Inbox 的兼容桥
InboxController 在部分路由上使用了 `@UseInterceptors(ContextCompatibilityInterceptor)` [context-compatibility.interceptor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/interceptors/context-compatibility.interceptor.ts)，它的作用是把旧版 widget（没有 contextKeys）的请求，自动补齐默认 context，让老前端 + 新后端可以无缝共存。

### 8.6 为什么 UpdateAllNotifications 要先查 _id 再 updateByIds（而不是直接 updateMany）？
因为需要 **Webhook 每条消息单独发** + **ClickHouse 每条消息单独 trace**。直接 `updateMany` 只返回修改数，拿不到被修改的文档内容。Novu 用「先 find _id → 分片 updateByIds（返回完整文档） → 分片发 Webhook/Trace」的模式，在批量场景下保证了 observability 不丢。
