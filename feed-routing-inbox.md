# Feed/Category 路由分组与 Inbox 过滤边界分析

## 1. 路由分组概览

Novu 存在两个面向 subscriber（通知中心）目前存在两套并行的 API 路由体系，加上 Feed 管理路由形成三角边界：

| 路由前缀 | 控制器 | 职责定位 | Feed 过滤能力 |
|---------|------|---------|------------|
| `/widgets` | [widgets.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/apps/api/src/app/widgets/widgets.controller.ts) | 旧版 Notification Center | 中心 | 支持 `feedIdentifier` 过滤 |
| `/inbox` | [inbox.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/apps/api/src/app/inbox/inbox.controller.ts) | 新版 Inbox API | **不支持** Feed 过滤，支持 tags/severity/data/archived/snoozed |
| `/feeds` | [feeds.controller.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/apps/api/src/app/feeds/feeds.controller.ts) | Feed CRUD 管理 | N/A |

---

## 2. Feed 与 Category（Notification Group）概念辨析

### 2.1 Feed 实体定义

Feed 实体定义在 [feed.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/dal/src/repositories/feed/feed.entity.ts)：

```typescript
export class FeedEntity {
  _id: string;
  name: string;
  identifier: string;  // 业务标识符，与 name 相同
  _environmentId: EnvironmentId;
  _organizationId: OrganizationId;
}
```

Feed 通过 `identifier` 字段对外暴露，用于查询时通过 `feedIdentifier 查询参数传入。

### 2.2 Notification Group 实体定义

Notification Group（即 category 概念）定义在 [notification-group.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/dal/src/repositories/notification-group/notification-group/notification-group.entity.ts)：

```typescript
export class NotificationGroupEntity {
  _id: string;
  name: string;
  _environmentId: EnvironmentId;
  _organizationId: OrganizationId;
  _parentId?: string;
}
```

### 2.3 关键区别：

| 维度 | Feed | Notification Group (Category)
|-----|------|---------------------------|
| 作用层级 | 消息/消息模板 (MessageTemplate) 级 | 工作流模板 (NotificationTemplate/Workflow) 级
| 关联字段 | `_feedId`（Message/MessageTemplate 级 | `_notificationGroupId`（工作流模板级）
| 面向对象 | 消息的物理分组 | 工作流的逻辑分类
| 过滤方式 | feedIdentifier 查询参数 | 内部管理不参与消息过滤
| 是否参与消息路由 | ✅ 是（消息入库时写入 `_feedId` | ❌ 否（仅用于工作流组织）

---

## 3. 默认 Feed 机制

### 3.1 没有显式的 "默认 Feed"常量

系统中 **不存在** `DEFAULT_FEED` 常量。默认行为通过查询参数的缺失/null 分支实现。

### 3.2 过滤逻辑在 [message.repository.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/dal/src/repositories/message/message.repository.ts#L182-L199) 的 `getFilterQueryForMessage` 方法：

```typescript
// feedId === null 时：
if (query.feedId === null) {
  requestQuery._feedId = { $eq: null };  // 查询 _feedId 为空的消息
}

// feedId 存在时：
if (query.feedId) {
  const feeds = await this.feedRepository.find(
    { _environmentId: environmentId, identifier: { $in: query.feedId }, '_id');
  requestQuery._feedId = { $in: feeds.map((feed) => feed._id) };
}

// feedId 为 undefined 时：
// 不添加任何 _feedId 过滤（查询所有 Feed
```

### 3.3 三种查询语义边界

| feedId 参数值 | MongoDB 查询条件 | 含义
|-----------|--------------|------
| `undefined` (不传) | 无 `_feedId` 条件 | 返回所有 Feed，包括有 feed 的消息
| `null` | `_feedId: { $eq: null }` 仅查询未分配任何 Feed 的消息
| `string[]` | `_feedId: { $in: [...] }` 仅查询指定 Feed 的消息

---

## 4. 通知入库逻辑

### 4.1 消息创建链路

消息入库的核心链路在 [send-message-in-app.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/apps/worker/src/app/workflow/usecases/send-message/send-message-in-app.usecase.ts#L218-L241)：

```typescript
// 创建消息时，_feedId 直接从 step.template._feedId 传递：
message = await this.messageRepository.create({
  _notificationId: command.notificationId,
  _feedId: step.template._feedId,  // ← 来自消息模板
  channel: ChannelTypeEnum.IN_APP,
  // ...其他字段
});
```

### 4.2 Feed 来源追溯

`_feedId` 的来源层级链路：

```
Feed (DB: 工作流步骤模板 (Workflow Step MessageTemplate._feedId
  └── MessageTemplate._feedId (消息模板
      └── Message._feedId (最终存储
```

### 4.3 Feed 变更同步

当更新消息模板的 `_feedId` 变更时，会同步更新历史消息，在 [update-message-template.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/application-generic/src/usecases/message-template/update-message-template/update-message-template.usecase.ts#L134-L140)：

```typescript
if (command.feedId || (!command.feedId && existingTemplate._feedId) {
  await this.messageRepository.updateFeedByMessageTemplateId(
    command.environmentId,
    command.templateId,
    command.feedId
  );
}
```

---

## 5. 两套查询接口差异分析

### 5.1 `/widgets/notifications/feed 旧接口

[get-notifications-feed.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/apps/api/src/app/widgets/usecases/get-notifications-feed/get-notifications-feed.usecase.ts#L52-L61)

调用链：
- `messageRepository.findBySubscriberChannel()
- **支持** `feedId` 过滤
- 支持 `seen`/`read` 状态过滤
- 支持 `payload` 过滤
- **不支持** tags/severity/archived/snoozed/data 过滤

### 5.2 `/inbox/notifications 新接口

[get-notifications.usecase.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/apps/api/src/app/inbox/usecases/get-notifications/get-notifications.usecase.ts#L57-L78)

调用链：
- `messageRepository.paginate()`
- **不支持** `feedId` 过滤
- 支持 `tags`/`severity`/`archived`/`snoozed`/`data` 过滤
- 支持 `createdGte`/`createdLte` 时间范围
- 支持 contextKeys 上下文隔离

### 5.3 功能矩阵对比

| 过滤维度 | `/widgets` (旧) | `/inbox` (新)
|---------|------------------|---------------|
| feedId | ✅ 支持 | ❌ 不支持
| tags | ❌ 不支持 | ✅ 支持
| severity | ❌ 不支持 | ✅ 支持
| archived | ❌ 不支持 | ✅ 支持
| snoozed | ❌ 不支持 | ✅ 支持
| data | ❌ 不支持 | ✅ 支持
| seen/read | ✅ 支持 | ✅ 支持
| payload | ✅ 支持 | ❌ 不支持（用 data 替代)
| contextKeys | ❌ 不支持 | ✅ 支持
| 时间范围 | ❌ 支持 | ✅ 支持

---

## 6. 权限边界

### 6.1 认证方式认证

两套接口都使用 `AuthGuard('subscriberJWT 认证
- 两者都基于 subscriberJWT 认证

### 6.2 上下文隔离 (contextKeys)

上下文隔离机制在 [base-repository.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/dal/src/repositories/base-repository.ts#L74-L111) 实现 `buildContextExactMatchQuery()`：

```typescript
// contextKeys === undefined 或 [] 时：
return {
  $or: [
    { contextKeys: { $exists: false } },  // 兼容旧数据（无 contextKeys 字段
    { contextKeys: [] }                    // 新数据（空数组）
  ]
};

// contextKeys 有值时：
return {
  contextKeys: { $all: sortedKeys, $size: sortedKeys.length }
};
```

### 6.3 权限边界对比

| 权限维度 | `/widgets` | `/inbox`
|---------|----------|---------
| subscriberJWT 认证 | ✅ | ✅
| contextKeys 上下文隔离 | ❌ 未使用 | ✅ 全面使用
| environmentId 强制过滤 | ✅ | ✅
| subscriberId 强制过滤 | ✅ | ✅

关键问题：`/widgets` 接口未传递 contextKeys，可能导致跨上下文数据泄露。

---

## 7. 迁移兼容逻辑

### 7.1 Feed 字段兼容

`message.entity.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/dal/src/repositories/message/message.entity.ts#L111)：

```typescript
_feedId?: string;  // 可选字段，兼容历史消息可能为 undefined
```

### 7.2 contextKeys 兼容

`message.schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/58-novu/libs/dal/src/repositories/message/message.schema.ts#L153-L156)：

```typescript
contextKeys: {
  type: [Schema.Types.String],
  default: undefined,  // 默认 undefined，兼容旧数据无此字段
}
```

### 7.3 数据迁移策略

1. **Feed 迁移策略：
- Feed 引入前创建的消息 `_feedId` 为 `null/undefined
- 查询时 feedId===null 专门查询这批历史未分配 Feed 的消息

2. **contextKeys 迁移策略：
- contextKeys 引入前创建的消息 `contextKeys` 字段不存在
- 查询时 contextKeys===undefined 时使用 `$or: [{ contextKeys: { $exists: false } }, { contextKeys: [] }]` 同时匹配两种形态

---

## 8. 边界模糊点汇总

### 8.1 Feed/Category 边界混淆问题：
1. **Feed vs Notification Group (Category) 是两个独立概念但命名上容易混淆
2. `/widgets` 支持 Feed 过滤但不支持 contextKeys，反之亦然
3. 新老接口功能不对等导致功能重叠但功能矩阵

### 8.2 风险点：
- 两套接口功能不统一
- `/widgets` 缺少 contextKeys 可能导致跨上下文风险
- `/inbox` 缺少 Feed 过滤导致无法按 Feed 分组查询
- 默认 Feed 语义不明确（undefined vs null 语义差异

### 8.3 建议澄清：
1. 统一 `/widgets` 和 `/inbox` 功能对齐
2. 明确默认 Feed 的官方语义定义
3. 统一 Feed 过滤和 Category 概念边界
4. 补齐 `/inbox` 的 Feed 过滤能力
5. 补齐 `/widgets` 的 contextKeys 隔离

---

## 9. 消息模板创建时默认 Feed 的写入值

### 9.1 创建路径：显式 `null`

消息模板创建逻辑在 `CreateMessageTemplate.execute()` 中：

```typescript
_feedId: command.feedId ? command.feedId : null,
```

关键行为：当 `command.feedId` 为 `undefined`、空字符串、`null` 或其他 falsy 值时，一律写入 **`null`**。不存在"不写入"的中间态。

### 9.2 写入值的三态分析

| `command.feedId` 值 | 写入 `_feedId` | MongoDB 实际存储 |
|---------------------|---------------|-----------------|
| `"660a1b..."` (有效 ObjectId) | `"660a1b..."` | ObjectId 引用 |
| `undefined` (不传) | `null` | `null` |
| `null` | `null` | `null` |
| `""` (空字符串) | `null` | `null` |

这意味着：**新创建的 In-App 消息模板，如果不指定 feed，其 `_feedId` 一定是 `null`，而非字段缺失。**

### 9.3 与历史数据的语义差异

| 数据来源 | `_feedId` 存储值 | 查询 `feedId === null` 是否命中 |
|---------|-----------------|-------------------------------|
| Feed 引入前创建的旧消息 | 字段不存在（`$exists: false`） | ❌ 不命中 |
| 新创建（未指定 Feed） | `null` | ✅ 命中 |
| 新创建后移除 Feed（见第 10 节） | `null` | ✅ 命中 |

`getFilterQueryForMessage` 中 `query.feedId === null` 分支使用 `{ $eq: null }`，此条件同时匹配字段缺失和值为 `null` 的文档，因此实际上新旧数据均被覆盖。

### 9.4 widgets 控制器中的 `feedIdentifier` 传递

在 `widgets.controller.ts` 的 `getNotificationsFeed` 方法中：

```typescript
let feedsQuery: string[] | undefined;
if (query.feedIdentifier) {
  feedsQuery = Array.isArray(query.feedIdentifier) ? query.feedIdentifier : [query.feedIdentifier];
}
```

当客户端不传 `feedIdentifier` 时，`feedsQuery` 为 `undefined`，直接传入 `getNotificationsFeedCommand.feedId = undefined`。此值最终到达 `getFilterQueryForMessage` 的 `query.feedId` 参数，由于既非 `null` 也非 truthy，不触发任何 `_feedId` 条件，即 **返回所有 Feed 的消息**。

---

## 10. 移除 Feed 后模板与历史消息同步的不同点

### 10.1 更新模板的两条路径

`UpdateMessageTemplate.execute()` 对 Feed 变更的处理分两种场景：

**场景 A — 设置 Feed（`command.feedId` 有值）：**

```typescript
if (command.feedId) {
  updatePayload._feedId = command.feedId;  // $set 操作
}
```

MongoDB 操作：`{ $set: { _feedId: "660a1b..." } }` — 将字段设为新 ObjectId。

**场景 B — 移除 Feed（`command.feedId` 为 falsy，但模板原 `_feedId` 存在）：**

```typescript
if (!command.feedId && existingTemplate._feedId) {
  unsetPayload._feedId = '';  // $unset 操作
}
```

MongoDB 操作：`{ $unset: { _feedId: '' } }` — **删除字段**，而非设为 `null`。

### 10.2 关键差异：模板层 vs 消息层的 Feed 同步

无论场景 A 还是 B，同步到历史消息时调用的是同一个方法：

```typescript
if (command.feedId || (!command.feedId && existingTemplate._feedId)) {
  await this.messageRepository.updateFeedByMessageTemplateId(
    command.environmentId,
    command.templateId,
    command.feedId   // 场景 B 时为 undefined
  );
}
```

`updateFeedByMessageTemplateId` 的实现：

```typescript
async updateFeedByMessageTemplateId(environmentId: string, messageId: string, feedId?: string | null) {
  return this.update(
    { _environmentId: environmentId, _messageTemplateId: messageId },
    { $set: { _feedId: feedId } }   // 始终用 $set
  );
}
```

### 10.3 不同点汇总

| 操作 | 模板层 (MessageTemplate) | 消息层 (Message) |
|------|------------------------|-----------------|
| 设置 Feed | `$set: { _feedId: ObjectId }` | `$set: { _feedId: ObjectId }` |
| 移除 Feed | `$unset: { _feedId: '' }` (字段被删除) | `$set: { _feedId: undefined }` (字段设为 null) |
| 移除后的存储状态 | `_feedId` 字段不存在 | `_feedId` 值为 `null` |

这种不一致会导致：

- 模板文档中 `_feedId` 字段不存在（`$exists: false`）
- 关联消息文档中 `_feedId` 值为 `null`
- 两者在语义上等价（均表示"未分配 Feed"），但在 MongoDB 查询中行为不同：`{ _feedId: null }` 同时匹配字段缺失和值为 null，而 `{ _feedId: { $eq: null } }` 也同时匹配两者
- **实际查询不会出错**，因为 `getFilterQueryForMessage` 使用 `$eq: null`，但底层存储形态不一致，未来若引入严格等值匹配（如索引查询）可能产生差异

---

## 11. 旧 Inbox 客户端清空 ContextKeys 的兼容分支

### 11.1 问题背景

从 `@novu/js` v3.13.0 起，Inbox SDK 在创建订阅标识符时会自动携带 `:ctx_` 前缀以支持上下文隔离。旧版本客户端不会生成此前缀，但 JWT 中可能已包含 `contextKeys`。如果服务端仍然按 contextKeys 创建订阅，则新旧客户端生成的标识符不一致，导致偏好查找失败。

### 11.2 兼容拦截器实现

`ContextCompatibilityInterceptor` 定义在 `inbox/interceptors/context-compatibility.interceptor.ts`：

```typescript
function shouldDisableContextForOldClient(clientVersion?: string): boolean {
  const version = parseClientVersion(clientVersion);
  if (!version) {
    return true;  // 无版本头 = 旧客户端，禁用 context
  }
  return !isContextAwareVersion(version);  // < 3.13.0 也禁用
}

intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
  const subscriberSession = request.user;
  if (!subscriberSession?.contextKeys || subscriberSession.contextKeys.length === 0) {
    return next.handle();  // 本身无 contextKeys，无需处理
  }
  if (shouldDisableContextForOldClient(clientVersion)) {
    subscriberSession.contextKeys = undefined;  // 清空 contextKeys
  }
  return next.handle();
}
```

### 11.3 拦截范围

| 控制器 | 拦截方式 | 拦截粒度 |
|-------|---------|---------|
| `InboxController` | 方法级 `@UseInterceptors(ContextCompatibilityInterceptor)` | 仅 `PATCH /inbox/subscriptions/:subscriptionIdentifier/preferences/:workflowIdOrIdentifier` |
| `InboxTopicController` | 类级 `@UseInterceptors(ContextCompatibilityInterceptor)` | 该控制器下所有端点 |

`InboxController` 中仅有一个端点（订阅偏好更新）应用了此拦截器，其余端点（通知列表、计数、标记等）未拦截，因为这些端点本身需要 contextKeys 来正确过滤数据。

### 11.4 兼容分支的三种情况

| 客户端状态 | `Novu-Client-Version` 头 | `contextKeys` 处理 | 行为 |
|-----------|--------------------------|-------------------|------|
| 新客户端 (≥ 3.13.0) | `@novu/js@3.13.0` | 保持原值 | 正常上下文隔离 |
| 旧客户端 (< 3.13.0) | `@novu/js@3.0.0` | 设为 `undefined` | 退回无上下文模式 |
| 无版本头 | 不传 | 设为 `undefined` | 退回无上下文模式 |

### 11.5 清空 contextKeys 后的查询行为

当 `contextKeys` 被设为 `undefined` 后，在 `buildContextExactMatchQuery` 中：

```typescript
if (contextKeys === undefined || contextKeys.length === 0) {
  return {
    $or: [{ contextKeys: { $exists: false } }, { contextKeys: [] }],
  };
}
```

这意味着旧客户端会看到所有"无上下文"的消息和偏好数据，不会看到带上下文隔离的数据。这对旧客户端是安全的，因为旧客户端本身不理解上下文概念。

### 11.6 潜在风险

1. **写操作遗漏**：拦截器仅覆盖订阅偏好的写操作端点。如果旧客户端通过其他写路径（如 `PATCH /inbox/preferences`）修改偏好，这些路径未拦截，可能导致在新上下文下写入旧格式的偏好数据。

2. **读操作不拦截**：`GET /inbox/notifications` 等读操作不应用此拦截器。如果 JWT 中包含 contextKeys，旧客户端仍然只能看到该上下文的数据——但这与旧客户端预期一致（旧客户端获取的 JWT 本身就不应包含 contextKeys，因为 session 创建时旧客户端不会传递 context 参数）。

3. **Session 创建的防线**：在 `session.usecase.ts` 中，`resolveContexts` 方法仅在客户端请求体包含 `context` 字段时才解析上下文。旧客户端不会发送此字段，因此其 JWT 中 `contextKeys` 为空数组，拦截器不会触发清空逻辑——这构成了第一道防线。
