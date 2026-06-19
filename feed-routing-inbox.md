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
