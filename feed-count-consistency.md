# Notification Feed 删除未见消息与 HTTP 计数缓存一致性

> 本文精确追踪两条容易混淆的代码路径：
> 1. **删除一条 `seen=false` 的消息** → DB 变化 → 缓存失效 → WS 事件投递 → 计数重算 → 前端收到什么
> 2. **HTTP `/count` 缓存** → TTL 多长 → key 怎么构建 → 失效如何执行 → 失败了会怎样

---

## 1. 删除操作的完整代码路径

### 1.1 入口：两种删除 API

| API 路由 | 用途 | Controller 方法 | Usecase |
|----------|------|-----------------|---------|
| `DELETE /inbox/notifications/:id/delete` | 删除单条 | [inbox.controller.ts#L353-L368](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/inbox.controller.ts#L353-L368) `deleteNotification()` | `DeleteNotification` → `DeleteManyNotifications` |
| `POST /inbox/notifications/delete` | 批量按条件删除 | `deleteAllNotifications()` | `DeleteAllNotifications` |

**单条删除走批量方法**：[DeleteNotification](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-notification/delete-notification.usecase.ts#L20-L55) 内部把单个 ID 包装成数组，委托给 `DeleteManyNotifications.execute()`。

### 1.2 DeleteManyNotifications 完整流程

代码: [delete-many-notifications.usecase.ts#L37-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts#L37-L89)

```
DeleteNotification.execute(command)
    │  单条 id 包装为 [id]
    ▼
DeleteManyNotifications.execute(command)
    │
    ├─ ① getSubscriber()  解析 subscriber
    │
    ├─ ② messageRepository.deleteMessagesByIds()  ← 硬删除！
    │     [message.repository.ts#L1147-L1174](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L1147-L1174)
    │     │
    │     ├─ 先 find(query) 取出完整文档（为了 webhook/trace）
    │     ├─ 再 this.delete(query)  ← 调用 Mongoose Model.deleteMany()
    │     └─ 返回被删除的 MessageEntity[]
    │
    ├─ ③ logTraces()  写 ClickHouse message_deleted 事件
    │     （失败只 warn，不阻塞主流程）
    │
    ├─ ④ invalidateCacheService.invalidateQuery()
    │     key = buildMessageCountKey().invalidate({
    │       subscriberId: subscriber.subscriberId,
    │       _environmentId: command.environmentId
    │     })
    │
    ├─ ⑤ processWebhooksInBatches([MESSAGE_DELETED], deletedMessages)
    │     按 100 条分片发送 Webhook
    │
    └─ ⑥ webSocketsQueueService.add({
         event: WebSocketEventEnum.UNREAD,  ← 只投 UNREAD！
         userId: subscriber._id,
         _environmentId: subscriber._environmentId,
         contextKeys: command.contextKeys ?? []
       })
```

### 1.3 DeleteAllNotifications 完整流程

代码: [delete-all-notifications.usecase.ts#L31-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-all-notifications/delete-all-notifications.usecase.ts#L31-L101)

```
DeleteAllNotifications.execute(command)
    │
    ├─ ① getSubscriber() + 解析/校验 filters
    │
    ├─ ② messageRepository.deleteMessagesWithFilters()
    │     [message.repository.ts#L1176-L1230](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L1176-L1230)
    │     │
    │     ├─ 构建 query（支持 tagGroups, data, read, archived 过滤）
    │     ├─ 先 find(query) 取出完整文档
    │     └─ 再 this.delete(query) 硬删除
    │
    ├─ ③ sendWebhookEvents()
    │
    ├─ ④ invalidateCacheService.invalidateQuery()
    │     key = buildMessageCountKey().invalidate({...})
    │
    ├─ ⑤ analyticsService.track()
    │
    └─ ⑥ webSocketsQueueService.add({
         event: WebSocketEventEnum.UNREAD  ← 同样只投 UNREAD
       })
```

**关键发现**：两个删除 usecase 都**只投 `UNREAD`**，不投 `UNSEEN`。

---

## 2. 删除 seen=false 消息的计数影响：逐步推演

### 2.1 场景设定

```
用户当前消息列表：
  消息 A: seen=false, read=false, archived=false, snoozed=null
  消息 B: seen=true,  read=false, archived=false, snoozed=null
  消息 C: seen=true,  read=true,  archived=false, snoozed=null

当前计数状态：
  unseenCount = 1  (消息 A)
  unreadCount = 2  (消息 A + B)
  unreadSeverity.high = 1, medium = 1 (假设 A=high, B=medium)
```

### 2.2 执行：删除消息 A（seen=false 的消息）

```
T0: DELETE /inbox/notifications/A/delete
    │
    ▼
T1: messageRepository.deleteMessagesByIds({ids: [A]})
    → 文档从 MongoDB 被硬删除
    → 返回 [MessageA_Entity]

T2: logTraces → ClickHouse 写入 message_deleted（异步）

T3: invalidateCacheService.invalidateQuery({
       key: buildMessageCountKey().invalidate({
         subscriberId: "sub123",
         _environmentId: "env456"
       })
     })
    → 尝试删除 Redis 中该用户所有 message_count 缓存
    → （详细分析见第 3 节）

T4: webSocketsQueueService.add({
       event: WebSocketEventEnum.UNREAD,
       userId: subscriber._id,
       _environmentId: ...,
       contextKeys: [...]
     })
    → 投递到 BullMQ/SQS 队列

T5: HTTP 响应 204 No Content 返回给客户端
```

### 2.3 apps/ws 消费 UNREAD 事件

```
T6: WebSocketWorker 消费 UNREAD 事件
    │
    ▼
T7: ExternalServicesRoute.sendUnreadCountChange()
    │
    ├─ getCount(env, userId, 'in_app', {read: false}, {limit: 101}, contextKeys)
    │   → MongoDB countDocuments:
    │     { _envId, _subId, channel:'in_app', deleted:{$exists:false},
    │       read: false, archived:{$in:[true,false]}, seen:{$in:[true,false]} }
    │   → 结果: 1 ✅（只剩消息 B）
    │
    ├─ getCountBySeverity(env, userId, 'in_app', {read:false, snoozed:false}, {limit:99}, contextKeys)
    │   → 对每个 severity 级别各查一次
    │   → 结果: {high:0, medium:1, low:0, none:0} ✅
    │
    └─ wsGateway.sendMessage(userId, UNREAD, {
         unreadCount: 1,
         counts: { total: 1, severity: { high:0, medium:1, low:0, none:0 } },
         hasMore: false
       }, contextKeys)

T8: 前端收到 unread_count_changed
    → unreadCount 从 2 变为 1 ✅
    → severity.high 从 1 变为 0 ✅
```

### 2.4 ❌ 不会触发的：unseen_count_changed

删除操作**只投了 UNREAD**，`apps/ws` **不会重算 UNSEEN**。

```
T9: 前端 unseenCount 仍显示为 1 ❌
    实际 unseenCount 应为 0（消息 A 已被删除，不存在了）
```

**为什么会这样？**

看 [delete-many-notifications.usecase.ts#L79-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts#L79-L88) 的 WS 投递代码：

```ts
this.webSocketsQueueService.add({
  name: 'sendMessage',
  data: {
    event: WebSocketEventEnum.UNREAD,  // ← 硬编码为 UNREAD
    userId: subscriber._id,
    _environmentId: subscriber._environmentId,
    contextKeys: command.contextKeys ?? [],
  },
  groupId: subscriber._organizationId,
});
```

**没有根据被删除消息的 `seen` 状态做条件判断**。无论被删消息是 seen=true 还是 seen=false，都只投 UNREAD。

### 2.5 何时自愈

```
T10: 以下任一事件触发时，unseenCount 会被修正：

  (a) 新消息到达 → RECEIVED 事件 → 同时触发 UNSEEN + UNREAD 重算
  (b) 用户打开通知中心 → VisibilityTracker 触发 seen → UNSEEN 事件重算
  (c) 前端主动调用 GET /inbox/notifications/count → 走 DB 实时查
  (d) Session 重新初始化 → 包含 count 查询

  在 T10(a/b/c/d) 时：
  getCount({seen: false}) → 0 ✅ 自愈
```

### 2.6 对比：如果删除的是 seen=true 的消息

```
删除消息 B (seen=true, read=false):
  T1-T8 同上流程
  → unreadCount 从 2 变为 1 ✅（只剩消息 A）
  → unseenCount 不变，仍为 1 ✅（消息 A 还在）
  → 不存在不一致！
```

**结论**：只有**删除 `seen=false` 的消息**才会导致 unseenCount 暂时不一致。而 `seen=true` 的消息被删除时，unseenCount 本来就不受影响，只投 UNREAD 是完全正确的。

---

## 3. HTTP /count 缓存机制

### 3.1 缓存层架构

```
┌───────────────────────────────────────────────────────────────────┐
│                 GET /inbox/notifications/count                     │
│                                                                   │
│  ① InboxController.getNotificationsCount()                        │
│      │                                                            │
│      ▼                                                            │
│  ② NotificationsCount.execute()                                   │
│      │  @CachedQuery({ builder: buildMessageCountKey() })         │
│      │  ← 装饰器拦截，先查缓存                                     │
│      │                                                            │
│      ├─ [缓存命中] → 直接返回 JSON.parse(cachedValue)              │
│      │                                                            │
│      └─ [缓存未命中] → 执行原始方法                                 │
│            │                                                      │
│            ├─ messageRepository.getCount(...)  查 MongoDB         │
│            │                                                      │
│            └─ cacheService.setQuery(cacheKey, JSON.stringify(result))
│                 │  写入 Redis（pipeline: sadd + expire + set）     │
│                 └─ 返回结果                                        │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2 CachedQuery 装饰器实现

代码: [cached-query.interceptor.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/cache/interceptors/cached-query.interceptor.ts#L1-L53)

```ts
descriptor.value = async function (...args: any[]) {
  // 缓存不可用 → 跳过缓存，直接执行原始方法
  if (!this.cacheService?.cacheEnabled()) return await originalMethod.apply(this, args);

  const cacheKey = builder(...args);  // buildMessageCountKey().cache(command)

  // 1. 读缓存
  try {
    const value = await cacheService.get(cacheKey);
    if (value) return JSON.parse(value);  // 命中 → 直接返回
  } catch (err) {
    Logger.error(err, ...);  // 读失败 → 不阻塞，继续执行原始方法
  }

  // 2. 执行原始方法
  const response = await originalMethod.apply(this, args);

  // 3. 写缓存
  try {
    await cacheService.setQuery(cacheKey, JSON.stringify(response));
  } catch (err) {
    Logger.error(err, ...);  // 写失败 → 不阻塞，返回结果
  }

  return response;
};
```

**关键设计**：缓存读/写失败都不会阻塞主流程，只是 log error 后继续。

### 3.3 缓存 Key 的两层结构

`buildMessageCountKey()` 在 [queries.ts#L35-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/cache/key-builders/queries.ts#L35-L64) 定义了两个方法：

#### cache() — 写入/读取时的 key

```
格式: {QUERY:MESSAGE_COUNT:e={envId}:s={subscriberId}}:query={JSON.stringify(command)}
示例: {QUERY:MESSAGE_COUNT:e=env456:s=sub123}:query={"environmentId":"env456","subscriberId":"sub123","filters":[{"read":false,"snoozed":false}]}
```

**特点**：key 包含完整的 command 参数（包括 filters），所以不同的 filter 组合会生成不同的 key。

#### invalidate() — 失效时的 pattern key

```
格式: {QUERY:MESSAGE_COUNT:e={envId}:s={subscriberId}}
示例: {QUERY:MESSAGE_COUNT:e=env456:s=sub123}
```

**特点**：key 只到 subscriber 粒度，不包含 query 参数。

#### setQuery() 的 Redis 两层写入

[cache.service.ts#L82-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/cache/cache.service.ts#L82-L95)

```ts
async setQuery(key: string, value: string, options?: CachingConfig) {
  const { credentials, query } = splitKey(key);
  // credentials = "{QUERY:MESSAGE_COUNT:e=env456:s=sub123}"
  // query = '{"environmentId":"env456",...}'

  const pipeline = this.client.pipeline();
  pipeline.sadd(credentials, query);  // 把 query 加入 Set（跟踪该用户所有缓存过的查询）
  pipeline.expire(credentials, ttl + getTtlInSeconds(options));  // Set 也有过期时间
  pipeline.set(key, value, 'EX', getTtlInSeconds(options));  // 实际值
  await pipeline.exec();
}
```

**两层结构**：
- **Set key** (`credentials`): 存储该用户缓存过的所有 query 参数（用于批量删除）
- **Value key** (`key`): 存储具体的查询结果

这意味着同一个用户可能有多条缓存记录（因为 filters 参数不同），它们都被记录在同一个 Set 中。

### 3.4 delQuery() — 失效时的批量删除

[cache.service.ts#L138-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/cache/cache.service.ts#L138-L155)

```ts
async delQuery(key: string) {
  // key = "{QUERY:MESSAGE_COUNT:e=env456:s=sub123}"

  const queries = await this.client.smembers(key);  // 取出 Set 中所有 query
  // queries = ['{"filters":[{"read":false}]}', '{"filters":[{"read":false,"snoozed":false}]}', ...]

  const pipeline = this.client.pipeline();
  queries.forEach((query) => {
    const fullKey = `${key}:${QUERY_PREFIX}=${query}`;
    pipeline.del(fullKey);  // 删除每条具体的缓存值
  });
  pipeline.del(key);  // 删除 Set 本身
  await pipeline.exec();
}
```

**关键理解**：调用 `invalidateQuery({key: buildMessageCountKey().invalidate({...})})` 时，会**一次性删除该用户所有 filter 组合下的 count 缓存**。不需要知道之前缓存过哪些 filter，因为 Set 里记录了全部。

### 3.5 TTL 配置

所有 Redis Provider 的默认 TTL:

[redis-provider.ts#L10](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/in-memory-provider/providers/redis-provider.ts#L10) / [elasticache-cluster-provider.ts#L10](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/in-memory-provider/providers/elasticache-cluster-provider.ts#L10) 等：

```ts
const DEFAULT_TTL_SECONDS = 60 * 60 * 2;  // 7200 秒 = 2 小时
```

可通过环境变量 `REDIS_TTL` / `REDIS_CLUSTER_TTL` 覆盖。

实际写入时的 TTL 有 ±10% 的抖动（jitter）：

```ts
private getTtlInSeconds(options?: CachingConfig): number {
  const seconds = options?.ttl || this.cacheTtl;
  const number = addJitter(seconds, this.TTL_VARIANT_PERCENTAGE);  // 0.1 = ±10%
  return number;
}
```

所以实际 TTL 范围：**6480~7920 秒（约 1.8~2.2 小时）**。Set key 的 TTL 略长（`ttl + getTtlInSeconds(options)`）。

---

## 4. 缓存失效的时序与失败分析

### 4.1 正常流程：缓存失效先于 WS 投递

回到 [DeleteManyNotifications.execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts#L37-L89)：

```
步骤 ②: deleteMessagesByIds()   ← 硬删除 MongoDB 文档
步骤 ④: invalidateCacheService.invalidateQuery()   ← 删除 Redis 缓存（await）
步骤 ⑥: webSocketsQueueService.add()   ← 投递 WS 事件到队列
```

**注意顺序**：
1. 先删 DB（步骤 ②）
2. 再删缓存（步骤 ④，**await 等待完成**）
3. 最后投 WS（步骤 ⑥，**不 await，fire-and-forget**）

这是合理的：在 WS 事件被 `apps/ws` 消费时（可能是几秒后），缓存早已被清除。如果 `apps/ws` 的处理逻辑也要读 `/count` 的 HTTP 缓存，不会命中旧值。

### 4.2 InvalidateCacheService 的容错设计

代码: [invalidate-cache.service.ts#L22-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/cache/invalidate-cache.service.ts#L22-L30)

```ts
public async invalidateQuery({ key }: { key: string }): Promise<void | unknown[]> {
  if (!this.cacheService?.cacheEnabled()) return;  // 缓存不可用 → 跳过

  try {
    return await this.cacheService.delQuery(key);   // 尝试删除
  } catch (err) {
    Logger.error(err, `An error has occurred when deleting by query "key: ${key}",`, LOG_CONTEXT);
    // ⚠️ 只 log，不 throw！
  }
}
```

**如果 Redis 宕机 / 连接超时 / delQuery 失败**：
- `catch` 只记日志，**不抛异常**
- 主流程继续执行（WS 投递不受影响）
- **缓存中保留了旧值** → 下次 HTTP /count 请求会命中旧缓存

### 4.3 失效失败的后果：时间线

```
T0: 用户有 2 条未读消息
    Redis 缓存: MESSAGE_COUNT → {count: 2}

T1: 用户 delete 一条 seen=false 的消息
    ├─ MongoDB 删除成功 ✅
    ├─ invalidateCache 失败（Redis 瞬断）❌ → 缓存仍是 {count: 2}
    └─ WS UNREAD 投递成功 ✅

T2: apps/ws 消费 UNREAD → 重算 → 推送 unreadCount=1 ✅
    → 前端 WS 通道：计数正确

T3: 另一个标签页 / 另一个设备调用 GET /inbox/notifications/count
    ├─ CachedQuery 读缓存 → 命中旧值 {count: 2} ❌
    └─ 返回 2（实际应为 1）

T4: 缓存自然过期（最坏情况：2.2 小时后）
    └─ 或者下一次写操作触发 invalidate 成功 → 自愈
```

**影响范围**：
- **WS 通道**的计数不受缓存影响（`apps/ws` 的 `getCount()` 每次直接查 MongoDB，不走 `@CachedQuery`）
- **HTTP 通道**的计数可能返回旧值，直到缓存过期或下一次成功的 invalidate

### 4.4 自愈机制排序（从快到慢）

| 触发 | 延迟 | 影响范围 | 机制 |
|------|------|---------|------|
| **WS 推送** | 毫秒级 | 仅当前 WS 连接的设备 | `apps/ws` 直接查 DB，不经过缓存 |
| **下一次写操作的 invalidate** | 秒级 | 所有后续 HTTP 请求 | 任何 read/archive/delete 操作都会重新 invalidate |
| **前端手动刷新 /count** | 不确定 | 取决于缓存是否还在 | 如果缓存未过期，仍命中旧值 ❌ |
| **Session 重新初始化** | 分钟级 | 该用户所有设备 | Session 包含 count 查询（走 @CachedQuery），如果缓存已过期则查 DB |
| **缓存 TTL 自然过期** | 最坏 2.2 小时 | 所有后续 HTTP 请求 | Redis key 自然过期后，下次 /count 必然查 DB |

### 4.5 CachedQuery 读取失败的容错

[cached-query.interceptor.ts#L27-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/application-generic/src/services/cache/interceptors/cached-query.interceptor.ts#L27-L37)

```ts
try {
  const value = await cacheService.get(cacheKey);
  if (value) return JSON.parse(value);  // 命中 → 返回
} catch (err) {
  Logger.error(err, ...);  // 读取异常 → 不阻塞，fallback 到 DB
}

const response = await originalMethod.apply(this, args);  // 查 DB
```

**如果 Redis 读取超时 / 连接断开**：
- 不会返回错误给用户
- 会 fallback 到直接查 MongoDB
- 查询结果仍然正确，只是更慢

### 4.6 CachedQuery 写入失败的容错

```ts
try {
  await cacheService.setQuery(cacheKey, JSON.stringify(response));
} catch (err) {
  Logger.error(err, ...);  // 写入异常 → 不阻塞，继续返回
}
```

**如果 Redis 写入失败**：
- 用户仍然拿到正确结果
- 但下次 /count 请求不会命中缓存，需要再次查 DB
- 实际上**更安全**（因为没有缓存 = 没有旧值）

---

## 5. 删除场景的完整一致性矩阵

### 5.1 单条删除：按消息状态细分

| 被删消息状态 | 对 unread total 的影响 | 对 unread severity 的影响 | 对 unseen 的影响 | WS 事件 | unseen 是否暂时不一致 |
|-------------|----------------------|-------------------------|----------------|---------|---------------------|
| `seen=F, read=F` | -1 | -1 | -1（但不推送） | UNREAD | **是** |
| `seen=T, read=F` | -1 | -1 | 0 | UNREAD | 否 |
| `seen=T, read=T` | 0 | 0 | 0 | UNREAD | 否 |
| `seen=F, read=F, snoozedUntil=date` | -1 | 0（之前不在 severity） | -1（但不推送） | UNREAD | **是** |
| `seen=T, read=F, snoozedUntil=date` | -1 | 0 | 0 | UNREAD | 否 |

### 5.2 批量删除（DeleteAllNotifications）

按 `filters` 参数的不同，删除的消息范围不同：

| filters 参数 | 被删消息特征 | 对 unseen 的影响 | unseen 是否暂时不一致 |
|-------------|------------|----------------|---------------------|
| `{read: false}` | 所有未读（含未见） | 可能 -N | **是**（如果其中包含 seen=F 的消息） |
| `{read: true}` | 所有已读 | 0 | 否 |
| `{archived: true}` | 所有归档 | 0（归档级联 seen=T） | 否 |
| `{tags: [...]}` | 按标签 | 取决于标签下的消息 | **可能** |
| 无 filters | **全部消息** | -unseenCount | **是** |

### 5.3 deleteMessagesWithFilters 的查询条件

[message.repository.ts#L1192-L1221](file:///d:/fz/0601-2/solo-dogfeeding/code/63-novu/libs/dal/src/repositories/message/message.repository.ts#L1192-L1221)

注意 `archived` 和 `read` 过滤条件的互斥逻辑：

```ts
if (isArchivedFiltered) {
  // archived 过滤优先
  if (!filters.archived) {
    query.$or = [{ archived: { $exists: false } }, { archived: false }];
  } else {
    query.archived = true;
  }
} else if (isReadFiltered) {
  // read 过滤次之
  if (!filters.read) {
    query.$or = [{ read: { $exists: false } }, { read: false }];
  } else {
    query.read = true;
  }
}
```

**注意**：`archived` 和 `read` 不会同时出现在 query 中。如果同时传了 `archived=true` 和 `read=false`，只有 `archived=true` 生效（因为 `isArchivedFiltered` 优先判断）。

---

## 6. 缓存一致性时序图

### 6.1 正常场景：所有步骤成功

```
时间 →
Client          API (apps/api)              Redis              MongoDB          WS (apps/ws)
  │                 │                         │                   │                │
  │──DELETE /A──→   │                         │                   │                │
  │                 │──deleteMessagesByIds──→  │                   │                │
  │                 │                         │     ──delete()──→ │                │
  │                 │←── [MessageA] ──────────────────────────   │                │
  │                 │                         │                   │                │
  │                 │──invalidateQuery──────→ │                   │                │
  │                 │   (delQuery)            │                   │                │
  │                 │                         │── SMEMBERS ──→   │                │
  │                 │                         │←── queries[] ──  │                │
  │                 │                         │── DEL key1,key2 ─→│                │
  │                 │                         │── DEL set ──────→│                │
  │                 │←── OK ─────────────────  │                   │                │
  │                 │                         │                   │                │
  │                 │──WS.add(UNREAD)─────────────────────────────────────────────→│
  │                 │   (不 await)             │                   │                │
  │←── 204 ─────── │                         │                   │                │
  │                 │                         │                   │                │
  │                 │                         │                   │    ┌───────────┤
  │                 │                         │                   │    │ 消费 UNREAD│
  │                 │                         │     ──getCount()─→│    │           │
  │                 │                         │                   │←───│ count=1   │
  │←── ws: unread_count_changed(1) ───────────────────────────────────────────────│
  │   ✅ unreadCount=1                       │                   │                │
  │   ❌ unseenCount 仍为 1（没有 UNSEEN 推送）│                   │                │
```

### 6.2 异常场景：Redis 瞬断导致 invalidate 失败

```
Client          API (apps/api)              Redis              MongoDB          WS (apps/ws)
  │                 │                         │                   │                │
  │──DELETE /A──→   │                         │                   │                │
  │                 │──deleteMessagesByIds──→  │                   │                │
  │                 │                         │     ──delete()──→ │                │
  │                 │←── [MessageA] ──────────────────────────   │                │
  │                 │                         │                   │                │
  │                 │──invalidateQuery──────→ │                   │                │
  │                 │   (delQuery)            │                   │                │
  │                 │                         │── SMEMBERS ──→ ❌ 连接超时          │
  │                 │←── catch(error) ─────── │                   │                │
  │                 │   (只 log，不 throw)     │                   │                │
  │                 │                         │                   │                │
  │                 │──WS.add(UNREAD)─────────────────────────────────────────────→│
  │←── 204 ─────── │                         │                   │                │
  │                 │                         │                   │                │
  │                 │                         │                   │    ┌───────────┤
  │←── ws: unread_count_changed(1) ───────────────────────────────────────────────│
  │   ✅ WS 通道正确                          │                   │                │
  │                 │                         │                   │                │
  │──GET /count──→ │                         │                   │                │
  │                 │──cacheService.get()───→ │                   │                │
  │                 │                         │←── {count:2} ──  │ ← 旧缓存！      │
  │                 │←── hit ──────────────── │                   │                │
  │←── {count:2} ──│                         │                   │                │
  │   ❌ HTTP 通道返回旧值                     │                   │                │
  │                 │                         │                   │                │
  │   ...最多 2.2 小时后缓存过期...             │                   │                │
  │   或下一次写操作 invalidate 成功 → 自愈     │                   │                │
```

---

## 7. 为什么删除不投 UNSEEN？以及该不该改？

### 7.1 现状分析

删除操作只投 UNREAD，不投 UNSEEN。当被删消息的 `seen=false` 时，unseenCount 会暂时不一致。

**影响评估**：
- `seen=false` 的消息被删除的场景**极少**：在正常 UI 流程中，用户要看到删除按钮就必须先看到这条消息 → VisibilityTracker 已经标记 seen=true
- 只有绕过 UI 直接调 API 才会出现删除 seen=false 消息的情况
- 即使出现，unseenCount 也会在下一个 UNSEEN 事件时自愈

### 7.2 如果要修复：条件投递

理论上可以在 `DeleteManyNotifications` 中检查被删消息是否包含 `seen=false` 的消息：

```ts
const hasUnseenMessages = deletedMessages.some(m => !m.seen);

this.webSocketsQueueService.add({ event: WebSocketEventEnum.UNREAD, ... });
if (hasUnseenMessages) {
  this.webSocketsQueueService.add({ event: WebSocketEventEnum.UNSEEN, ... });
}
```

但 Novu 没有这样做，原因可能是：
1. **极端场景概率极低**（需要绕过 UI 删除未见消息）
2. **额外 WS 事件 = 额外 DB 查询**（UNSEEN 需要 `getCount({seen:false})`），对 99% 的删除场景是无用开销
3. **最终一致性**在通知系统中可以接受（unseenCount 只是"小红点"，不影响业务正确性）

### 7.3 DeleteAllNotifications 更容易触发

批量删除 `DELETE /inbox/notifications/delete`（特别是 `filters={read:false}` 或无 filters）更容易删除到 seen=false 的消息。但 Novu 仍然只投 UNREAD，说明团队有意选择了「简洁性 > 绝对一致性」的设计。

---

## 8. 速查卡

```
┌──────────────────────────────────────────────────────────────────────┐
│                  删除 seen=false 消息的计数一致性                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  删除操作                                                             │
│  ├─ 硬删除（this.delete()，非软删除）                                  │
│  ├─ invalidateCache（await，但失败只 log 不 throw）                    │
│  ├─ 投 UNREAD（fire-and-forget 到 BullMQ/SQS）                       │
│  └─ 不投 UNSEEN ❌                                                    │
│                                                                      │
│  计数影响（删除 seen=F 消息时）                                        │
│  ├─ unreadCount ✅ 下一帧通过 WS 推送修正                              │
│  ├─ unreadSeverity ✅ 同上                                            │
│  └─ unseenCount ❌ 暂时不一致，直到下一个 UNSEEN 事件自愈               │
│                                                                      │
│  HTTP /count 缓存                                                     │
│  ├─ TTL: 7200 秒（2 小时）+ ±10% jitter                              │
│  ├─ Key: {QUERY:MESSAGE_COUNT:e={env}:s={sub}}:query={JSON}          │
│  ├─ 失效: delQuery() → SMEMBERS + 批量 DEL（一次性清该用户所有 filter）│
│  ├─ 失效失败: 只 log 不 throw → 缓存保留旧值                          │
│  └─ 自愈: 下一次写操作 invalidate / TTL 过期 / WS 推送覆盖             │
│                                                                      │
│  两层保障                                                             │
│  ├─ WS 通道: 每次直接查 DB，不经过缓存 → 近实时正确                    │
│  └─ HTTP 通道: @CachedQuery 装饰器 → 最终一致（最坏 2.2 小时）        │
│                                                                      │
│  容错                                                                 │
│  ├─ 缓存读失败 → fallback 到 DB → 返回正确值                          │
│  ├─ 缓存写失败 → 不缓存 → 下次直接查 DB → 返回正确值                  │
│  ├─ 缓存失效失败 → 旧值滞留 → 下次写操作 / TTL 到期自愈               │
│  └─ WS 消息丢失 → at-most-once → 下次任何计数事件自愈                 │
└──────────────────────────────────────────────────────────────────────┘
```
