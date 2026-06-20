# Notification Feed 持久化与读/未读状态分析

## 1. 核心数据模型

Notification Feed 涉及三个核心实体，职责明确分离：

### 1.1 Feed 实体（Feed 配置）
Feed 本身只是一个"分类标签"配置，不存储消息内容。

- 实体定义: [feed.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/feed/feed.entity.ts#L5-L15)
- 数据库 Schema: [feed.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/feed/feed.schema.ts#L8-L27)

核心字段：
| 字段 | 说明 |
|------|------|
| `_id` | Feed ID |
| `name` | Feed 名称（展示用） |
| `identifier` | Feed 标识符（API 查询用） |
| `_environmentId` | 环境 ID |
| `_organizationId` | 组织 ID |

### 1.2 Notification 实体（通知顶层容器）
Notification 是一次通知触发的顶层容器，关联 Workflow 模板和订阅者。

- 实体定义: [notification.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/notification/notification.entity.ts#L26-L63)
- 数据库 Schema: [notification.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/notification/notification.schema.ts#L6-L78)

核心字段：
| 字段 | 说明 |
|------|------|
| `_templateId` | 关联的 NotificationTemplate（工作流） |
| `_subscriberId` | 接收的订阅者 |
| `transactionId` | 事务 ID，用于关联整条链路 |
| `channels` | 本次通知涉及的渠道列表 |
| `payload` | 通知 payload |

### 1.3 Message 实体（Feed 展示的消息 + 状态存储）
**Message 是 Notification Feed 的核心持久化实体**，真正存储了 In-App 渠道的消息内容以及所有读/未读状态字段。

- 实体定义: [message.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.entity.ts#L23-L140)
- 数据库 Schema: [message.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.schema.ts#L7-L159)

**状态相关字段**（核心）：
| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `seen` | boolean | `false` | 用户是否"见过"这条消息（通常 = 打开了通知中心） |
| `read` | boolean | `false` | 用户是否"阅读"了这条消息（通常 = 点击了消息） |
| `archived` | boolean | `false` | 是否已归档 |
| `snoozedUntil` | Date | - | 延后提醒到的时间 |
| `firstSeenDate` | Date | - | 首次被标记为 seen 的时间 |
| `lastSeenDate` | Date | - | 最后一次 seen 时间 |
| `lastReadDate` | Date | - | 最后一次 read 时间 |
| `archivedAt` | Date | - | 归档时间 |

**关联字段**：
| 字段 | 说明 |
|------|------|
| `_feedId` | 关联的 Feed（可空，表示不在特定 feed 中） |
| `_notificationId` | 关联的 Notification |
| `_subscriberId` | 订阅者 |
| `_templateId` / `_messageTemplateId` | 工作流模板 / 步骤消息模板 |
| `channel` | 渠道（Feed 中固定为 `in_app`） |
| `transactionId` | 事务 ID |
| `contextKeys` | 上下文隔离键（用于多租户/多实例 Inbox） |

---

## 2. 消息入库（持久化）完整路径

### 2.1 整体流程图

```
Trigger 事件
    │
    ▼
CreateNotificationJobs (libs/application-generic)
    │  └─► notificationRepository.create()  ← 创建 Notification
    │  └─► 创建 WorkflowRun（审计）
    │  └─► 创建多个 Job（每个渠道一个）
    ▼
Job 进入队列（BullMQ / SQS）
    ▼
Worker 消费 Job (apps/worker)
    ▼
SendMessageInApp.execute()  ← In-App 渠道专属处理器
    │
    ├─ 编译模板（Handlebars / Bridge）
    │
    ├─ 检查是否已存在相同 message（幂等去重）
    │   └─ 按 _notificationId + _templateId + _messageTemplateId + transactionId 查询
    │
    ├─ [不存在] messageRepository.create()   ← 插入新 Message，seen/read 默认 false
    │
    ├─ [已存在] messageRepository.findOneAndUpdate()  ← 重置 seen=false，更新内容
    │
    ├─ InvalidateCache：清除消息计数缓存
    │
    ├─ WebSocketsQueueService.add()：投递 RECEIVED 事件到 WS 队列
    │
    └─ 发送 Webhook：MESSAGE_SENT / MESSAGE_DELIVERED
```

### 2.2 关键代码位置

**步骤 1：创建 Notification**
- Usecase: [create-notification-jobs.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/usecases/create-notification-jobs/create-notification-jobs.usecase.ts#L90-L111)
  - `createNotification()` 方法调用 `notificationRepository.create()`，写入 `notifications` 集合
  - 同时创建 WorkflowRun（可观测/审计用）

**步骤 2：Worker 中创建 Message**
- Usecase: [send-message-in-app.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-in-app.usecase.ts#L67-L336)

核心逻辑：
1. **幂等去重查询**（第 175-189 行）：通过 `_notificationId`, `_templateId`, `_messageTemplateId`, `transactionId`, `_feedId` 等字段查询是否已有相同 Message。**注意**：有状态工作流才会有 `_templateId`，无状态工作流依赖其他字段组合去重。
2. **新建 Message**（第 217-240 行）：`messageRepository.create()`，此时 `seen` 和 `read` 使用 Mongoose schema 默认值 `false`，同时设置 `deliveredAt: [new Date()]`。
3. **更新已有 Message**（第 243-258 行）：`findOneAndUpdate()` 重置 `seen: false`，刷新 `createdAt`。
4. **失效缓存**（第 193-198 行）：`invalidateCache.invalidateQuery(buildMessageCountKey())`。
5. **投递 WebSocket 事件**（第 276-292 行）：`WebSocketsQueueService.add()` 投递 `WebSocketEventEnum.RECEIVED`。
6. **发送 Webhook**（第 307-331 行）：`MESSAGE_SENT` 和 `MESSAGE_DELIVERED`。

---

## 3. 读/未读（seen/read）标记路径

### 3.1 API 入口（HTTP）

所有 Widget/Inbox 相关的接口都在 [widgets.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/widgets/widgets.controller.ts#L80-L549)，使用 `subscriberJwt` 鉴权。

| HTTP 方法 & 路径 | 用途 | 对应 Usecase |
|-------------------|------|--------------|
| `POST /widgets/messages/markAs` | **旧版（deprecated）**单条/批量标记，body: `{messageId, mark: {seen?, read?}}` | `MarkMessageAs` |
| `POST /widgets/messages/mark-as` | **新版**单条/批量标记，body: `{messageId, markAs: 'read'\|'seen'\|'unread'\|'unseen'}` | `MarkMessageAsByMark` |
| `POST /widgets/messages/read` | 标记所有未读 → 已读（可按 feedId 过滤） | `MarkAllMessagesAs` |
| `POST /widgets/messages/seen` | 标记所有未见 → 已见（可按 feedId 过滤） | `MarkAllMessagesAs` |
| `POST /widgets/messages/:messageId/actions/:type` | 用户点击消息 CTA 按钮（自动标记为已读） | `UpdateMessageActions` |

状态枚举定义见 [messages.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/shared/src/types/messages.ts#L1-L6)：
```ts
enum MessagesStatusEnum {
  READ = 'read',
  SEEN = 'seen',
  UNREAD = 'unread',
  UNSEEN = 'unseen',
}
```

### 3.2 三种 Usecase 的职责

#### （1）MarkMessageAs（旧版，已 deprecated）
- 代码: [mark-message-as.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts#L25-L213)
- 调用仓库方法: `messageRepository.changeStatus()`（deprecated）
- 支持 body: `{ seen?: boolean; read?: boolean }` 组合

流程：
1. 失效 messageCount 缓存
2. 查询 subscriber
3. `changeStatus()` 更新 DB
4. 根据 seen/read 分别发送 Webhook（MESSAGE_SEEN / MESSAGE_READ / MESSAGE_UNREAD）
5. 投递 WS 计数事件（UNREAD / UNSEEN）
6. 写入交互 Trace（ClickHouse）

#### （2）MarkMessageAsByMark（新版，推荐）
- 代码: [mark-message-as-by-mark.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/widgets/usecases/mark-message-as-by-mark/mark-message-as-by-mark.usecase.ts#L26-L187)
- 调用仓库方法: `messageRepository.changeMessagesStatus()`
- 使用 `MessagesStatusEnum` 统一枚举

流程与旧版基本一致，但仓库层调用了更完善的 `changeMessagesStatus()`。

#### （3）MarkAllMessagesAs（批量"全部标记"）
- 代码: [mark-all-messages-as.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/widgets/usecases/mark-all-messages-as/mark-all-messages-as.usecase.ts#L17-L113)
- 调用仓库方法: `messageRepository.markAllMessagesAs()`
- 支持按 `feedIdentifiers`（feed identifier 数组）过滤
- 固定 channel = `ChannelTypeEnum.IN_APP`

### 3.3 仓库层：核心状态更新逻辑

所有状态更新最终汇聚到 [message.repository.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L100-L1250) 的几个方法。

#### 方法 1：`markAllMessagesAs()`（第 577-648 行）
用于"全部标记为已读/已见"场景。

关键逻辑：
1. 将 `feedIdentifiers`（字符串）转换为 `_feedId`（ObjectId）查询条件
2. `getReadSeenUpdateQuery()` 生成**筛选条件**（哪些记录需要更新）
3. `getReadSeenUpdatePayload()` 生成**更新内容**
4. 先查出将被更新的文档 ID（为了性能，只取 `_id`）
5. 按 BATCH_SIZE=100 分批次执行 `update({_id: {$in: chunk}}, {$set: updatePayload})`
6. 返回更新后的完整文档

#### 方法 2：`changeMessagesStatus()`（第 685-719 行）
用于按指定 messageId 列表批量更新。逻辑与 markAll 类似，但直接用传入的 `messageIds`。

#### 方法 3：`updateMessagesStatus()`（第 872-967 行，私有核心）
这是**最完整、最新的状态更新实现**，其他方法最终会演进到调用它。被以下公开方法调用：
- `updateMessagesStatusByIds()` — 按 ID 列表更新
- `updateMessagesFromToStatus()` — 按 from→to 条件更新

**状态联动规则**（核心！按优先级从高到低）：

| 触发动作 | `seen` | `read` | `archived` | `snoozedUntil` | `firstSeenDate` | 其他 |
|---------|--------|--------|------------|----------------|-----------------|------|
| `archived: true` | ✅ true | ✅ true | ✅ true | - | 若不存在则设为 now | `archivedAt=now` |
| `archived: false` | ✅ true | ✅ true | ✅ false | - | 若不存在则设为 now | `archivedAt=null` |
| `read: true` | ✅ true | ✅ true | - | - | 若不存在则设为 now | `lastReadDate=now`, `lastSeenDate=now` |
| `read: false` | ✅ true | ✅ false | ✅ false | - | 若不存在则设为 now | `lastReadDate=null`, `archivedAt=null` |
| `seen: true` | ✅ true | - | - | - | **若不存在则设为 now** | `lastSeenDate=now` |
| `seen: false` | ✅ false | ✅ false | ✅ false | - | 设为 null | `lastSeenDate=null`, `lastReadDate=null`, `archivedAt=null` |
| `snoozedUntil` 设置 | ✅ true | - | ✅ false | ✅ 传入值 | 若不存在则设为 now | `archivedAt=null` |

代码实现见 [message.repository.ts#L892-L932](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L892-L932)。

**关键设计点**：
- **`firstSeenDate` 的处理**（第 946-960 行）：不是简单 `$set`，而是先做普通 update，再针对 `firstSeenDate: {$exists: false}` 的文档做一次补充 update。这保证只有"首次 seen"才写入 `firstSeenDate`。
- **分批更新**（第 949 行）：所有批量操作都按 100 条分片，避免 MongoDB 单次更新过大。

#### 方法 4：`getReadSeenUpdatePayload()`（第 539-575 行）
旧版枚举式状态 payload 映射（供 `markAllMessagesAs` 和 `changeMessagesStatus` 使用）：

| markAs | read | seen | lastReadDate | lastSeenDate |
|--------|------|------|--------------|--------------|
| READ   | true | true | now | now |
| UNREAD | false | true | now | now |
| SEEN   | - | true | - | now |
| UNSEEN | - | false | - | now |

> ⚠️ 注意：这里 UNREAD 时 seen 会被强制设为 true（"已见但未读"），与新的 `updateMessagesStatus()` 中 `seen:false` 会级联把 read 也置为 false 的规则不同，新旧方法存在语义差异。

---

## 4. 状态同步机制（WebSocket 实时推送）

### 4.1 整体架构

```
┌────────────── API / Worker ──────────────┐
│                                          │
│  WebSocketsQueueService.add(job)         │
│     │                                    │
│     ├─ [Enterprise 可选]                  │
│     │   SocketWorkerService.sendMessage() │
│     │   ──► 直接调用 Durable Object       │
│     │                                    │
│     └─ BullMQ / SQS 队列（WEB_SOCKETS）    │
│                                          │
└──────────────────────────────────────────┘
                      │
                      ▼
┌──────────────── apps/ws ─────────────────┐
│                                          │
│  WebSocketWorker（消费队列）               │
│     │                                    │
│     ▼                                    │
│  ExternalServicesRoute.execute()         │
│     │                                    │
│     ├─ RECEIVED ──► 推送消息 + 重算计数    │
│     ├─ UNREAD   ──► 重算并推送 unread 数  │
│     └─ UNSEEN   ──► 重算并推送 unseen 数  │
│                                          │
│     ▼                                    │
│  WSGateway.sendMessage()                 │
│     └─ 按 contextKeys 精确匹配后 socket.emit │
│                                          │
└──────────────────────────────────────────┘
                      │
                      ▼
              前端 Socket.io 客户端
```

### 4.2 事件类型

定义在 [ws.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/packages/shared/src/types/ws.ts#L1-L5)：

```ts
enum WebSocketEventEnum {
  RECEIVED = 'notification_received',  // 新消息到达
  UNREAD   = 'unread_count_changed',   // 未读数变化
  UNSEEN   = 'unseen_count_changed',   // 未见数变化
}
```

状态标记与 WS 事件的映射：[mapMarkMessageToWebSocketEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/shared/helpers/utils/mapMarkMessageToWebSocketEvent.ts#L1-L13)
- READ / UNREAD → `UNREAD` 事件
- SEEN / UNSEEN → `UNSEEN` 事件

### 4.3 WebSocketsQueueService（投递端）
代码: [web-sockets-queue.service.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/queues/web-sockets-queue.service.ts#L16-L110)

**双通道设计**：
1. **Enterprise Socket Worker**（优先）：如果 `socketWorkerService.isEnabled()` 返回 true，会直接调用 `socketWorkerService.sendMessage()`（Cloudflare Durable Objects 通道）。如果 `isLegacyWsDisabled()` 也为 true，则不再投递到 BullMQ。
2. **BullMQ / SQS 队列**（兼容/社区版）：投递到 `JobTopicNameEnum.WEB_SOCKETS` 队列，由 `apps/ws` 消费。

### 4.4 WebSocketWorker（消费端）
代码: [web-socket.worker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/services/web-socket.worker.ts#L23-L108)

关键点：
- WS 任务是 **at-most-once**：失败不重试（`setSqsFailedHandler` 返回 false），因为实时事件是点-in-time 状态，重放反而可能造成 UI 回滚。
- 调用 `ExternalServicesRoute.execute()` 做事件分发。

### 4.5 ExternalServicesRoute（事件分发）
代码: [external-services-route.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/usecases/external-services-route/external-services-route.usecase.ts#L11-L157)

三种事件处理：

#### `RECEIVED`（新消息）
1. 如果 payload 带 `messageId`（Worker 默认只传 ID），再查 DB 取出完整 Message 对象
2. `wsGateway.sendMessage()` 把完整消息推给前端
3. **额外触发**：重新计算并推送 `unseen_count_changed` 和 `unread_count_changed`

#### `UNREAD`（未读数变化）
1. `messageRepository.getCount({read: false})` 获取未读总数（最多查 101 条用于 hasMore 判断）
2. `messageRepository.getCountBySeverity()` 按严重等级聚合未读数
3. 推送 `{unreadCount, counts: {total, severity:{high,medium,low,none}}, hasMore}`

#### `UNSEEN`（未见数变化）
1. `messageRepository.getCount({seen: false})` 获取未见总数
2. 推送 `{unseenCount, hasMore}`

### 4.6 WSGateway（实际 Socket 推送 + Context 隔离）
代码: [ws.gateway.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/ws/src/socket/ws.gateway.ts#L16-L379)

**连接建立**：
- 客户端通过 JWT（audience=`widget_user`）鉴权
- 连接加入 `subscriber._id` 对应的 Socket.io Room
- 解析 JWT 中的 `contextKeys` 存到 `socket.data.contextKeys`

**推送匹配规则 `isExactMatch()`**（第 203-214 行）：
- 消息的 contextKeys 与 Socket 的 inboxContextKeys 必须完全一致（长度相同、元素相同，不考虑顺序）
- 如果消息没有 contextKeys（`[]`），只推送给同样没有 contextKeys 的 Inbox
- 这是**多 Inbox 实例/多租户隔离**的核心机制

**独立计数推送方法**：
- `sendUnreadCountToAllConnections()` 和 `sendUnseenCountToAllConnections()`：遍历该用户的所有活跃 socket，按每个 socket 自己的 `contextKeys` 分别计算和推送计数。

---

## 5. Feed 查询路径

### 5.1 Feed 列表查询
- Controller: [widgets.controller.ts#L118-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/widgets/widgets.controller.ts#L118-L146) → `GET /widgets/notifications/feed`
- Usecase: [get-notifications-feed.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/widgets/usecases/get-notifications-feed/get-notifications-feed.usecase.ts#L16-L125)
- Repository 方法: `messageRepository.findBySubscriberChannel()`

查询条件支持：
- `feedIdentifier`（转为 `_feedId`）
- `seen` / `read`（布尔过滤）
- `payload`（Base64 编码后的 JSON 精确匹配）
- 分页（page + limit）

### 5.2 计数查询
- `GET /widgets/notifications/unseen` → `getCount({seen: false})`
- `GET /widgets/notifications/unread` → `getCount({read: false})`
- `GET /widgets/notifications/count` → 灵活组合

核心方法: `messageRepository.getCount()` 和 `messageRepository.getCountBySeverity()`。

---

## 6. 数据库索引优化

Message 集合上针对 Feed 和状态查询创建了多个复合索引（见 [message.schema.ts#L257-L389](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.schema.ts#L257-L389)），其中最重要的是：

```
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

该索引同时覆盖：
- `findBySubscriberChannel()` — feed 列表查询
- `markAllMessagesAs()` — 批量标记查询
- `getCount()` / `getTotalCount()` — 计数查询
- `getMessages` usecase — API 消息列表

---

## 7. 总结：关键设计要点

| 维度 | 设计 |
|------|------|
| **状态存储位置** | `Message` 集合的 `seen`/`read`/`archived`/`snoozedUntil` 字段 + 三个时间戳 |
| **Feed 与 Message 关系** | Feed 是标签（`_feedId` 外键），Message 才是持久化实体；一个 Message 只能属于一个 Feed |
| **Notification vs Message** | 1 个 Notification 可以产生多个 Message（按渠道/步骤）；Feed 中只取 `channel=in_app` 的 Message |
| **seen vs read 语义** | seen = 用户打开通知中心被动看到；read = 用户主动点击/交互；read=true 隐含 seen=true |
| **状态级联** | archived > read > seen；升级时自动连带低级状态，降级（unseen）会连带清空 read/archived |
| **实时同步通道** | WebSocketsQueueService → BullMQ/SQS → WebSocketWorker → ExternalServicesRoute → WSGateway；Enterprise 版额外支持 Socket Worker 直推 |
| **多 Inbox 隔离** | `contextKeys` 字段 + Socket 连接时绑定的 contextKeys，消息推送和计数都按精确匹配做隔离 |
| **幂等性** | Message 创建时用 `_notificationId + _templateId + _messageTemplateId + transactionId` 去重，重复触发时重置 seen=false |
| **缓存失效** | 任何写操作（创建/标记/删除）都会 `invalidateCache` 清除 `buildMessageCountKey()` 对应的计数缓存 |
