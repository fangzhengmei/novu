# Feed/Category 路由分组与 Inbox 过滤边界分析

---

## 1. 路由分组概览

Novu 面向 subscriber（通知中心）目前存在两套并行的 API 路由体系，加上 Feed 管理路由形成三角边界：

| 路由前缀 | 控制器职责 | 定位 | Feed 过滤能力 |
|---------|----------|------|------------|
| `/widgets` | 旧版 Notification Center | Widget | 支持 `feedIdentifier` 过滤 |
| `/inbox` | 新版 Inbox API | Inbox | 不支持 Feed 过滤，支持 tags/severity/data/archived/snoozed |
| `/feeds` | Feed CRUD 管理 | 管理端 | N/A |

---

## 2. Feed 与 Category（Notification Group）概念辨析

### 2.1 Feed 实体

Feed 是消息级的物理分组，核心字段：
- `_id`: MongoDB ObjectId，内部引用
- `identifier`: 业务标识符，对外通过 `feedIdentifier` 查询参数传入
- `name`: 展示名称
- `_environmentId` / `_organizationId`: 租户边界

### 2.2 Notification Group 实体

Notification Group（即 category 概念）是工作流级的逻辑分类：
- `_id`: MongoDB ObjectId
- `name`: 分类名称
- `_parentId`: 可选的父级分类（支持层级）
- `_environmentId` / `_organizationId`: 租户边界

### 2.3 关键区别

| 维度 | Feed | Notification Group (Category) |
|-----|------|---------------------------|
| 作用层级 | 消息 / 消息模板 (MessageTemplate) 级 | 工作流模板 (Workflow/NotificationTemplate) 级 |
| 关联字段 | `_feedId` (Message/MessageTemplate 上) | `_notificationGroupId` (工作流模板上) |
| 面向对象 | 消息的物理分组 | 工作流的逻辑分类 |
| 参与消息路由 | 是（入库时写入 `_feedId`） | 否（仅用于工作流组织，不参与查询） |
| 对外过滤方式 | feedIdentifier 查询参数 | 不参与消息过滤 |

---

## 3. 默认 Feed 查询匹配逻辑（按代码顺序）

### 3.1 唯一的 Feed 查询入口：getFilterQueryForMessage

所有旧版 `/widgets` 相关的消息查询（列表、计数、Feed Count）最终都调用同一个方法 `getFilterQueryForMessage`。该方法对 `feedId` 参数的处理是唯一的权威逻辑。

代码分支顺序如下：

```
分支 1: query.feedId === null
    → requestQuery._feedId = { $eq: null }

分支 2: query.feedId (truthy)
    → SELECT _id FROM Feed WHERE identifier IN (...)
    → requestQuery._feedId = { $in: [feed._id ...] }

分支 3: query.feedId === undefined
    → 不添加任何 _feedId 条件
```

### 3.2 MongoDB `{ $eq: null }` 的匹配语义

**这是全文档结论一致性的基础。** MongoDB 中 `{ field: { $eq: null } }` 同时匹配：

1. 字段不存在（`$exists: false`）
2. 字段值为 BSON Null（Type 10）

该语义与 MQL 简写 `{ field: null }` 完全等价。此结论由 snoozed 查询的两个不同写法印证：
- `getFilterQueryForMessage` 中用 `$or: [{ $exists: false }, { snoozedUntil: null }]`
- `paginate` 中用 `snoozedUntil: { $eq: null }`
两者在 MongoDB 层结果集完全相同。

### 3.3 三种查询参数值对应的实际语义

| feedId 参数值 | 产生的 MongoDB 查询条件 | 匹配的 `_feedId` 状态 | 含义 |
|-----------|---------------------|--------------------|------|
| `undefined`（不传） | 无 `_feedId` 条件 | 任意（有值 / null / 缺失） | 返回所有消息，不区分 Feed |
| `null` | `_feedId: { $eq: null }` | 字段缺失 **或** 值为 null | 返回所有未分配任何 Feed 的消息 |
| `string[]` | `_feedId: { $in: [feed._ids] }` | 等于某指定 Feed 的 ObjectId | 仅返回指定 Feed 的消息 |

### 3.4 调用方参数传递对照

| 上层 usecase / 接口 | feedId 来源 | 传入值 | 命中分支 |
|-------------------|-----------|--------|---------|
| `/widgets/notifications/feed` 无 feedIdentifier | 未传 → feedsQuery = undefined | undefined | 分支 3（全部） |
| `/widgets/notifications/feed` 带 feedIdentifier | query.feedIdentifier → 数组 | string[] | 分支 2（指定 Feed） |
| `/widgets/feed/count` 无 feedIdentifier | 同上 | undefined | 分支 3（全部） |
| `/widgets/feed/count` 带 feedIdentifier | 同上 | string[] | 分支 2（指定 Feed） |
| `/widgets/notifications` 旧端点 | dto.feedIdentifier? 未传 | undefined | 分支 3（全部） |
| `/inbox/*` 新端点 | paginate() **无 feedId 参数** | N/A（永远分支 3 语义） | 全部 |

关键结论：**新版 `/inbox` API 所在的 `paginate()` 方法根本没有 `feedId` 参数，也不会在查询中添加任何 `_feedId` 条件。因此 `/inbox` 接口天然等同于旧 API 中 feedId 未传的语义——即全部消息。**

---

## 4. 通知入库逻辑（写入链路）

### 4.1 消息入库的唯一 Feed 赋值点

In-App 消息发送 usecase 中，创建消息时 `_feedId` 直接复制自步骤模板：

```typescript
message = await this.messageRepository.create({
  _notificationId: command.notificationId,
  _feedId: step.template._feedId,  // ← 来自 MessageTemplate._feedId
  channel: ChannelTypeEnum.IN_APP,
  // ...其他字段
});
```

不存在任何二次计算或默认值兜底逻辑。`_feedId` 的值完全由步骤模板决定。

### 4.2 模板层 Feed 的写入链路

模板的 `_feedId` 字段写入存在三处关键路径：

**路径 A — 创建消息模板：**
```typescript
_feedId: command.feedId ? command.feedId : null,
```
- `command.feedId` 有效 → 写入指定 ObjectId
- 其他任何 falsy 值（undefined/null/空串）→ 显式写入 **`null`**

**路径 B — 设置/变更 Feed（更新消息模板）：**
```typescript
if (command.feedId) {
  updatePayload._feedId = command.feedId;  // $set 新值
}
```

**路径 C — 移除 Feed（更新消息模板）：**
```typescript
if (!command.feedId && existingTemplate._feedId) {
  unsetPayload._feedId = '';  // $unset → 删除字段
}
```

### 4.3 写入值的最终数据库状态对照表

| 操作场景 | 模板层 `MessageTemplate._feedId` | 关联消息层 `Message._feedId` |
|---------|--------------------------------|---------------------------|
| Feed 机制引入前创建 | 字段缺失 | 字段缺失 |
| 新模板不指定 Feed | 值为 `null` | 值为 `null`（随发送时复制） |
| 新模板指定 Feed | 值为 ObjectId | 值为 ObjectId（随发送时复制） |
| 更新模板：设置 Feed | `$set` → 新 ObjectId | `updateFeedByMessageTemplateId` → `$set` 新 ObjectId |
| 更新模板：移除 Feed | `$unset` → **字段被删除** | `updateFeedByMessageTemplateId` → `$set: undefined` → **值为 null** |

### 4.4 写入状态与查询分支的一致性验证

将写入状态与查询分支 1（`{ $eq: null }`）匹配情况对照：

| 写入状态 | `{ $eq: null }` 是否命中 | 说明 |
|---------|------------------------|------|
| 字段缺失（旧模板/旧消息） | ✅ 是 | 分支 1 全部覆盖 |
| 值为 `null`（新模板未指定） | ✅ 是 | 分支 1 全部覆盖 |
| 值为 ObjectId（指定了 Feed） | ❌ 否 | 必须分支 2 的 `$in` 才能命中 |
| 字段被删除（模板层移除后） | ✅ 是 | 分支 1 覆盖 |

**一致性结论：** 虽然模板层和消息层在"移除 Feed"场景下分别使用了 `$unset` 和 `$set null`，导致底层存储形态不同（字段缺失 vs 值为 null），但查询端统一使用 `{ $eq: null }`，两种形态均被正确覆盖，不会产生结果集偏差。

---

## 5. 两套查询接口差异分析

### 5.1 旧接口 `/widgets/*` → getFilterQueryForMessage → find / getCount

调用链：
- `findBySubscriberChannel()` → 列表
- `getCount()` → 计数
- 两者共用 `getFilterQueryForMessage()`

能力矩阵：

| 过滤维度 | 支持情况 | 备注 |
|---------|---------|------|
| feedId | ✅ | 三态分支（见第 3 节） |
| seen | ✅ | 默认 `$in: [true, false]` |
| read | ✅ | 默认 `$in: [true, false]` |
| payload | ✅ | 扁平展开后 AND 匹配 |
| tags / tagGroups | ✅ | OR 组 AND 逻辑 |
| archived | ✅ | 默认 `$in: [true, false]` |
| snoozed | ✅ | 用 `snoozedUntil` 字段判断 |
| severity | ✅ | NONE 值含字段缺失 |
| data | ✅ | 与 payload 类似但走 data 字段 |
| contextKeys | ✅（方法签名支持） | 但 usecase 层**未传递**（详见第 6 节） |
| 时间范围 (createdAt) | ✅ | `getCount` 有参数，`findBySubscriberChannel` 未暴露 |

### 5.2 新接口 `/inbox/*` → paginate()

调用链：
- `paginate()` → 游标分页
- 该方法内部**完全独立**构建查询，不经过 `getFilterQueryForMessage`

能力矩阵：

| 过滤维度 | 支持情况 | 备注 |
|---------|---------|------|
| feedId | ❌ | paginate 参数无 feedId，也未构建相关条件 |
| seen | ✅ | 默认 `$in: [true, false]` |
| read | ✅ | 默认 `$in: [true, false]` |
| payload | ❌ | 未实现 |
| tags / tagGroups | ✅ | 同旧接口 mergeTagsMongoFragment |
| archived | ✅ | 默认 `$in: [true, false]` |
| snoozed | ✅ | `{ $eq: null }` / `{ $exists: true, $ne: null }` |
| severity | ✅ | NONE 值含字段缺失 |
| data | ✅ | buildDataFilterQuery |
| contextKeys | ✅ | `buildContextExactMatchQuery`（usecase 层**已传递**） |
| 时间范围 (createdAt) | ✅ | createdGte / createdLte |

### 5.3 功能矩阵对比

| 过滤维度 | `/widgets` (旧) | `/inbox` (新) |
|---------|----------------|---------------|
| feedId | ✅ 三态分支 | ❌ 完全不支持 |
| payload 过滤 | ✅ | ❌ |
| tags | ✅ | ✅ |
| severity | ✅ | ✅ |
| archived | ✅ | ✅ |
| snoozed | ✅ | ✅ |
| data 过滤 | ✅ | ✅ |
| seen / read | ✅ | ✅ |
| contextKeys 方法支持 | ✅ | ✅ |
| contextKeys 实际传递 | ❌（见第 6 节） | ✅（见第 6 节） |
| 时间范围 | ✅（仅计数） | ✅（列表） |

**一致性校验点：** snoozed=false 处理的两种写法结果集一致。
- 旧接口：`$or: [{ snoozedUntil: { $exists: false } }, { snoozedUntil: null }]`
- 新接口：`snoozedUntil: { $eq: null }`
- 根据 MongoDB 语义，两者完全等价。

---

## 6. 权限边界与上下文隔离（按代码顺序追踪）

### 6.1 认证方式

两套接口均使用 subscriberJWT 认证。通过 `AuthGuard('subscriberJWT')` 验证 subscriber 身份。

认证通过后，请求上下文中可获得 `subscriberSession`（即 AuthGuard 注入的 request.user），其中包含：
- `_environmentId`、`_organizationId`、`subscriberId`：租户与用户边界
- `contextKeys`: 订阅者会话的上下文键数组

### 6.2 contextKeys 隔离实现：buildContextExactMatchQuery

代码分支如下：

```
分支 1: contextKeys === undefined OR contextKeys.length === 0
    → {
        $or: [
          { contextKeys: { $exists: false } },  // 兼容旧数据（无字段）
          { contextKeys: [] }                   // 新数据（显式空数组）
        ]
      }

分支 2: contextKeys 长度 > 0
    → { contextKeys: { $all: sortedKeys, $size: sortedKeys.length } }
    → 精确集合匹配：键完全相同（与顺序无关）
```

### 6.3 唯一的上下文过滤条件注入点：`if (contextKeys !== undefined)`

两套查询路径的上下文过滤都通过同一个判断控制：

```typescript
if (contextKeys !== undefined) {
  const contextQuery = this.buildContextExactMatchQuery(contextKeys);
  // 旧接口 (getFilterQueryForMessage):
  requestQuery.$and = [...(requestQuery.$and ?? []), contextQuery];
  // 新接口 (paginate):
  query.$and = [...(query.$and ?? []), contextQuery];
}
```

**关键语义：** 当 `contextKeys === undefined` 时，不进入 if 分支，**不会向查询追加任何 `$and` 条件，即完全不做上下文过滤。** 这与"显式调用 buildContextExactMatchQuery(undefined) 落入分支 1 匹配无上下文消息"是两种完全不同的语义。

### 6.4 旧 `/widgets` 接口：contextKeys 的完整传递链路

按代码执行顺序追踪：

**Step 1：Controller 层创建 Command**

在 widgets.controller.ts 中创建 `GetNotificationsFeedCommand` / `GetFeedCountCommand` 时，传入的字段：
- organizationId、subscriberId、environmentId、feedId、seen、read、limit、payload 等
- **`contextKeys` 字段未被显式传入**

**Step 2：Command 类定义**

- `GetNotificationsFeedCommand` 继承 `EnvironmentWithSubscriber`
- `GetFeedCountCommand` 继承 `EnvironmentWithSubscriber`
- `EnvironmentWithSubscriber` 基类中 `contextKeys?: string[]` 是可选字段，默认 undefined
- Command.create() 不会额外注入 contextKeys
- 结果：`command.contextKeys === undefined`

**Step 3：Usecase 层调用 messageRepository**

`GetNotificationsFeed.execute()` 调用：
```typescript
this.messageRepository.findBySubscriberChannel(
  command.environmentId,
  subscriber._id,
  ChannelTypeEnum.IN_APP,
  { feedId: command.feedId, seen: ..., read: ..., payload },
  { limit: ..., skip: ... }
  // ← contextKeys 参数未传递
);
```

`GetFeedCount.execute()` 调用：
```typescript
this.messageRepository.getCount(
  command.environmentId,
  subscriber._id,
  ChannelTypeEnum.IN_APP,
  { feedId: command.feedId, seen: ..., read: ... },
  { limit: command.limit }
  // ← contextKeys 参数未传递
);
```

两个 usecase 均未传递 contextKeys 参数。

**Step 4：Repository 方法参数默认值**

- `findBySubscriberChannel` 方法签名不包含 contextKeys 参数，直接调用 `getFilterQueryForMessage` 时也不传递
- `getCount` 方法签名有 `contextKeys?: string[]`，但调用方未传，因此参数值为 `undefined`

**Step 5：getFilterQueryForMessage 中的判断**

```typescript
if (contextKeys !== undefined) {  // false，因为 contextKeys === undefined
  // ← 不进入
}
```

**最终结果：旧 `/widgets` 接口的所有读操作（列表、计数、feed count）都不会向 MongoDB 查询追加任何上下文过滤条件，即返回该 subscriber 在当前 environmentId 下的**全部** in-app 消息，不论其 contextKeys 值如何。**

### 6.5 新 `/inbox` 接口：contextKeys 的完整传递链路

按代码执行顺序追踪：

**Step 1：Controller 层创建 Command**

在 inbox.controller.ts 中创建各 Command 时：
```typescript
contextKeys: subscriberSession.contextKeys,
// ← 从 JWT 会话中显式取出并传递
```

**Step 2：Command 类定义**

`GetNotificationsCommand`、`NotificationsCountCommand` 等均继承 `EnvironmentWithSubscriber`，`contextKeys` 字段由 controller 显式赋值。

**Step 3：Usecase 层调用 messageRepository**

`GetNotifications.execute()`：
```typescript
this.messageRepository.paginate(
  {
    ...
    contextKeys: command.contextKeys,  // ← 透传
    ...
  },
  ...
);
```

`NotificationsCount.execute()`：
```typescript
this.messageRepository.getCount(
  ...
  command.contextKeys  // ← 透传（getCount 的第 7 个参数）
);
```

所有 inbox usecase（包括写操作 mark-as、snooze 等）均透传 `command.contextKeys`。

**Step 4：Repository 层判断**

- `paginate()`：`if (contextKeys !== undefined)` → 取决于 subscriberSession 中是否有值
- `getCount()`：同上

**最终结果：新 `/inbox` 接口的读操作会根据 subscriberSession.contextKeys 的值，有条件地追加上下文过滤条件。**

### 6.6 权限边界对比（修正后）

| 权限维度 | `/widgets` (旧) | `/inbox` (新) |
|---------|----------------|---------------|
| subscriberJWT 认证 | ✅ | ✅ |
| environmentId 强制过滤 | ✅ | ✅ |
| subscriberId 强制过滤 | ✅ | ✅ |
| contextKeys 过滤（方法层能力） | ✅ 方法支持 | ✅ 方法支持 |
| contextKeys 过滤（Controller 传递） | ❌ Command 创建时未注入 | ✅ 从 subscriberSession 显式赋值 |
| contextKeys 过滤（Usecase 传递） | ❌ 调用 messageRepository 时未传 | ✅ 全部透传 |
| contextKeys 过滤（最终效果） | ❌ **完全不做上下文过滤** | ✅ 按 session 上下文过滤 |

**修正后的真实风险：** 旧 `/widgets` 接口的权限边界只到 subscriberId 级别，**不做任何 contextKeys 隔离**。这意味着在多上下文场景下，使用旧版 Widget Notification Center 的 subscriber 可以看到自己所有上下文的全部消息，可能造成跨上下文数据泄漏。

### 6.7 contextKeys 与 feedId 在查询中的组合逻辑

由于两条过滤分支互不在对方的判断逻辑中（feedId 在方法开头，contextKeys 在方法中段、通过 `$and` 追加），两者用 `$and` 组合时独立生效，不会互相干扰。

旧 `/widgets` 接口的实际查询条件示例（`feedId=null` 但 `contextKeys=undefined` 未传）：
```
AND (
  _environmentId: "...",
  _subscriberId: "...",
  channel: "in_app",
  deleted: { $exists: false },
  _feedId: { $eq: null },
  seen: { $in: [true, false] },
  read: { $in: [true, false] },
  archived: { $in: [true, false] }
  // ← 无任何 contextKeys 条件
)
```

---

## 7. 迁移兼容逻辑汇总

### 7.1 `_feedId` 字段兼容

- Schema 层：`_feedId` 为可选属性，默认 undefined（不写入）
- 创建层：新模板未指定 Feed 时显式写入 `null`
- 查询层：`{ $eq: null }` 同时匹配"字段缺失"和"值为 null"
- 结论：新旧数据混合不会产生查询偏差

### 7.2 `contextKeys` 字段兼容

- Schema 层：`contextKeys` 默认 undefined（不写入）
- 查询层判断条件：`if (contextKeys !== undefined)`
  - 当 contextKeys 被实际传递且值为 `undefined` 或 `[]` → 调用 `buildContextExactMatchQuery` → 分支 1 → `$or: [$exists:false, []]`
  - 当 contextKeys 未被传递（旧 widgets 接口） → 判断不通过 → 不追加任何条件 → 无上下文过滤
- 结论：对新旧消息数据本身兼容（字段不存在和空数组等价），但旧 widgets 接口因未传递参数，**完全跳过上下文过滤**，这不是迁移兼容设计，而是参数遗漏（详见第 6 节）。

### 7.3 `severity` 字段兼容

- NONE 分支：`$or: [{ severity: { $exists: false } }, { severity: { $in: [...] } }]`
- 与 `_feedId` 和 `contextKeys`（被传递时）模式一致：字段缺失按"最低等级 None"处理

### 7.4 `snoozedUntil` 字段兼容

- snoozed=false 分支：`$or: [{ $exists: false }, { snoozedUntil: null }]`（旧接口）或 `{ $eq: null }`（新接口）
- 语义等价，新旧消息一致

**迁移兼容模式一致性结论：** 四个字段（`_feedId`, `contextKeys`, `severity`, `snoozedUntil`）在**被传递且命中时**的迁移兼容策略完全对齐——即字段不存在与显式空值在查询中等价。这是代码库中明确的统一设计模式。`contextKeys` 的特殊之处在于旧接口根本没有传递该参数，导致完全跳过过滤。

---

## 8. 旧 Inbox 客户端清空 ContextKeys 的兼容分支

### 8.1 背景

从 `@novu/js` v3.13.0 起，Inbox SDK 在创建订阅标识符时自动携带 `:ctx_` 前缀以支持上下文隔离。旧版本客户端不生成此前缀，如果服务端仍然按 JWT 中的 contextKeys 创建订阅标识符，新旧客户端的标识符不一致，导致偏好查找失败。

### 8.2 拦截器核心逻辑

```
shouldDisableContextForOldClient(clientVersion):
  1. 解析 "Novu-Client-Version" 头（格式 @novu/js@x.y.z）
  2. 无版本头 → 旧客户端 → 禁用（返回 true）
  3. < 3.13.0 → 旧客户端 → 禁用（返回 true）
  4. ≥ 3.13.0 → 新客户端 → 保留（返回 false）

intercept():
  1. 若 contextKeys 本就为空 → 直接放行
  2. 若是旧客户端 → subscriberSession.contextKeys = undefined
  3. 否则 → 保留原值
```

### 8.3 拦截范围

| 控制器 | 拦截方式 | 应用粒度 |
|-------|---------|---------|
| InboxController | 方法级装饰 | 仅 `PATCH /inbox/subscriptions/:subscriptionIdentifier/preferences/:workflowIdOrIdentifier`（订阅偏好更新写操作） |
| InboxTopicController | 类级装饰 | 该控制器下所有端点 |

### 8.4 清空后的行为（与第 6 节的衔接）

当拦截器将 `subscriberSession.contextKeys` 设为 `undefined` 后，后续 Controller 创建 Command 时 `contextKeys: subscriberSession.contextKeys` 即传入 `undefined`。该值一路透传到 Repository 层后，命中 `if (contextKeys !== undefined)` 的反条件——**不追加任何上下文过滤条件**。

效果：旧客户端仅能在偏好更新的写操作上禁用上下文隔离，但在消息列表、计数等读操作上（这些端点未被拦截），如果 JWT 中本就有 contextKeys，仍然会按上下文过滤。读操作和写操作的上下文隔离行为在旧客户端场景下不完全一致。

### 8.5 防漏双层保障

1. **Session 创建层（第一道防线）**：`session.usecase` 的 `resolveContexts` 仅在请求体有 `context` 字段时才解析。旧客户端不会发送此字段 → JWT 中 `contextKeys` 为空数组 → 拦截器不触发。
2. **拦截器（第二道防线）**：即使某些路径下旧客户端的 JWT 包含了 contextKeys，拦截器也会在偏好更新端点将其清空，避免在旧格式的订阅标识符下写入带上下文的偏好数据。

---

## 9. 边界模糊点与建议

### 9.1 Feed / Category 概念边界

| 风险点 | 说明 |
|-------|------|
| 命名歧义 | Feed 和 Notification Group（Category）都含"分组/分类"含义，容易在 API 文档和业务沟通中混淆 |
| 对外过滤差异 | Feed 参与消息查询过滤，Category 不参与，但在管理端界面中二者呈现方式相似 |

### 9.2 两套接口功能不对等

| 风险点 | 说明 |
|-------|------|
| Feed 过滤缺失 | `/inbox` 新接口不支持 feed 过滤，历史上依赖 Feed 分组的客户端迁移到新接口时能力下降 |
| contextKeys 完全未传递 | `/widgets` 旧接口从 Controller → Command → Usecase → Repository 全链路均未传递 contextKeys，在多上下文场景下会返回所有上下文的消息，存在跨上下文数据泄漏风险 |
| payload vs data | 旧接口用 payload 过滤，新接口用 data 过滤，二者字段位置不同（payload vs data），迁移时需注意 |

### 9.3 移除 Feed 时存储形态不一致

虽然查询结果一致，但模板层用 `$unset` 删除字段、消息层用 `$set: undefined` 写 null 的不一致可能引发：
- 数据一致性审计时误判
- 未来引入严格索引或类型校验时产生差异

### 9.4 建议澄清

1. 明确 Feed vs Notification Group 的官方定位文档：Feed 是**消息路由分组**，Category 是**工作流分类标签**
2. 统一 `/widgets` 和 `/inbox` 能力：
   - 补齐 `/inbox` 的 feedId 过滤能力
   - 补齐 `/widgets` 的 contextKeys 传递链路（Controller 创建 Command 时从 subscriberSession 注入 contextKeys，Usecase 调用 messageRepository 时透传）
3. 统一移除 Feed 操作：模板层也改为 `$set: null` 而非 `$unset`，保持存储形态一致
4. 在 API 文档中明确"默认 Feed"三种参数状态的语义：不传（全量）/ null（无 Feed）/ 指定（指定 Feed）
5. 在 API 文档中明确 contextKeys 隔离的生效范围：旧 `/widgets` 接口**不提供**上下文隔离，新 `/inbox` 接口**提供**上下文隔离
