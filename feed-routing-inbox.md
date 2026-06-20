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
| contextKeys | ✅ | `buildContextExactMatchQuery`（`contextKeys !== undefined` 时启用） |
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
| contextKeys | ✅ | `buildContextExactMatchQuery`（`contextKeys !== undefined` 时启用） |
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
| contextKeys | ✅（但 usecase 层未传递） | ✅（usecase 层已传递） |
| 时间范围 | ✅（仅计数） | ✅（列表） |

**一致性校验点：** snoozed=false 处理的两种写法结果集一致。
- 旧接口：`$or: [{ snoozedUntil: { $exists: false } }, { snoozedUntil: null }]`
- 新接口：`snoozedUntil: { $eq: null }`
- 根据 MongoDB 语义，两者完全等价。

---

## 6. 权限边界与上下文隔离

### 6.1 认证方式

两套接口均使用 subscriberJWT 认证。通过 `AuthGuard('subscriberJWT')` 验证 subscriber 身份。

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

### 6.3 权限边界对比

| 权限维度 | `/widgets` | `/inbox` |
|---------|----------|---------|
| subscriberJWT 认证 | ✅ | ✅ |
| environmentId 强制过滤 | ✅ | ✅ |
| subscriberId 强制过滤 | ✅ | ✅ |
| contextKeys 隔离（底层能力） | ✅ 方法支持 | ✅ 方法支持 |
| contextKeys 隔离（usecase 传递） | ❌ 未传递 | ✅ 已传递 |

**实际效果：** 旧 `/widgets` 接口虽然 `getFilterQueryForMessage` 方法支持 contextKeys 参数，但调用 usecase 未传递，所以执行时始终落入分支 1（`undefined` → 匹配 `$exists:false` + `[]`），即无法按上下文隔离。新 `/inbox` 接口从 session usecase 获取 contextKeys 并透传，能正确执行分支 2。

### 6.4 contextKeys 与 feedId 在查询中的组合逻辑

由于旧接口两条分支互不在对方的判断逻辑中（feedId 在方法开头，contextKeys 在方法中段、通过 `$and` 追加），两者用 `$and` 组合时独立生效，不会互相干扰。

例如：`feedId=null` + `contextKeys=undefined` →
```
AND (
  _feedId: { $eq: null },
  $or: [{ contextKeys: { $exists: false } }, { contextKeys: [] }]
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
- 查询层：`contextKeys === undefined` 分支通过 `$or` 同时匹配"字段不存在"和"空数组"
- 结论：新旧数据混合不会产生查询偏差

### 7.3 `severity` 字段兼容

- NONE 分支：`$or: [{ severity: { $exists: false } }, { severity: { $in: [...] } }]`
- 与 `_feedId` 和 `contextKeys` 模式一致：字段缺失按"最低等级 None"处理

### 7.4 `snoozedUntil` 字段兼容

- snoozed=false 分支：`$or: [{ $exists: false }, { snoozedUntil: null }]`（旧接口）或 `{ $eq: null }`（新接口）
- 语义等价，新旧消息一致

**迁移兼容模式一致性结论：** 四个字段（`_feedId`, `contextKeys`, `severity`, `snoozedUntil`）的迁移兼容策略完全对齐——即字段不存在与显式空值在查询中等价。这是代码库中明确的统一设计模式。

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

### 8.4 清空后的查询行为（与第 6 节的衔接）

当 `contextKeys` 被设为 `undefined` 后，在 `buildContextExactMatchQuery` 中落入分支 1：

```
$or: [{ contextKeys: { $exists: false } }, { contextKeys: [] }]
```

效果：旧客户端仅能看到所有"无上下文"的消息和偏好数据，无法看到任何带上下文隔离的数据。这与旧客户端对上下文概念无知的状态一致。

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
| contextKeys 未传递 | `/widgets` 旧接口底层方法支持 contextKeys，但 usecase 层未传递，在多上下文场景下可能泄漏跨上下文数据 |
| payload vs data | 旧接口用 payload 过滤，新接口用 data 过滤，二者字段位置不同（payload vs data），迁移时需注意 |

### 9.3 移除 Feed 时存储形态不一致

虽然查询结果一致，但模板层用 `$unset` 删除字段、消息层用 `$set: undefined` 写 null 的不一致可能引发：
- 数据一致性审计时误判
- 未来引入严格索引或类型校验时产生差异

### 9.4 建议澄清

1. 明确 Feed vs Notification Group 的官方定位文档：Feed 是**消息路由分组**，Category 是**工作流分类标签**
2. 统一 `/widgets` 和 `/inbox` 能力：补齐 `/inbox` 的 feedId 过滤，补齐 `/widgets` 的 contextKeys 传递
3. 统一移除 Feed 操作：模板层也改为 `$set: null` 而非 `$unset`，保持存储形态一致
4. 在 API 文档中明确"默认 Feed"三种参数状态的语义：不传（全量）/ null（无 Feed）/ 指定（指定 Feed）
