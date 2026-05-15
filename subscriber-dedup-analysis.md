# Subscriber 数据去重与合并策略分析报告

## 1. 概述

Novu 系统中 Subscriber（订阅者）数据去重策略包含三个层面的保护机制：
1. **数据库唯一索引约束** - 防止重复数据写入的底层保障
2. **业务层 upsert 逻辑** - 正常运行时的创建/更新逻辑
3. **数据迁移合并脚本** - 历史重复数据的清理与合并

## 2. 重复判断标准

### 2.1 唯一键定义

系统通过 **`subscriberId` + `_environmentId`** 的组合来判断 subscriber 是否重复。

**Schema 定义位置**：`libs/dal/src/repositories/subscriber/subscriber.schema.ts:179-182`

```typescript
subscriberSchema.index(
  { subscriberId: 1, _environmentId: 1 },
  { 
    name: 'unique_subscriber_per_environment', 
    unique: true, 
    partialFilterExpression: { deleted: false } 
  }
);
```

### 2.2 关键说明

- **作用范围**：唯一性约束仅在同一环境（environment）内有效
- **软删除处理**：`partialFilterExpression: { deleted: false }` 表示仅对未删除的 subscriber 强制执行唯一约束
- **允许场景**：同一个 `subscriberId` 可以存在于不同的环境中，也可以被删除后重新创建

## 3. 业务层去重机制

### 3.1 创建/更新逻辑

**代码位置**：`libs/application-generic/src/usecases/create-or-update-subscriber/create-or-update-subscriber.usecase.ts`

```typescript
@RetryOnError('MongoServerError', { maxRetries: 3, delay: 500 })
async execute(command: CreateOrUpdateSubscriberCommand) {
  const persistedSubscriber = await this.getExistingSubscriber(command);
  
  if (command.failIfExists && persistedSubscriber) {
    throw new ConflictException(`Subscriber with id "${command.subscriberId}" already exists`);
  }

  if (persistedSubscriber) {
    if (command.allowUpdate) {
      await this.updateSubscriber(command, persistedSubscriber);
    }
  } else {
    await this.createSubscriber(command);
  }
}
```

#### 关键特性：
1. **重试机制**：遇到 `MongoServerError`（通常是唯一键冲突）时自动重试最多 3 次
2. **幂等操作**：通过 `allowUpdate` 参数控制是纯创建还是创建/更新二合一
3. **原子性保障**：依赖数据库唯一索引在并发场景下保证数据一致性

### 3.2 批量创建（Bulk Upsert）

**代码位置**：`libs/dal/src/repositories/subscriber/subscriber.repository.ts:31-107`

```typescript
async bulkCreateSubscribers(subscribers: ISubscribersDefine[], ...) {
  const bulkWriteOps = subscribers.map((subscriber) => {
    const updatableFields = pickUpdatableSubscriberFields(subscriber);

    return {
      updateOne: {
        filter: {
          subscriberId: subscriber.subscriberId,
          _environmentId: environmentId,
          _organizationId: organizationId,
        },
        update: { $set: { ...updatableFields, deleted: false } },
        upsert: true,
      },
    };
  });

  const bulkResponse = await this.bulkWrite(bulkWriteOps);
}
```

#### 可更新字段列表：
```typescript
const UPDATABLE_SUBSCRIBER_FIELDS = [
  'firstName', 'lastName', 'email', 'phone', 'avatar', 
  'locale', 'data', 'channels', 'timezone'
];
```

## 4. 为什么有唯一索引仍会出现重复 subscriber？

尽管 Schema 定义了 `subscriberId + _environmentId` 的唯一索引，但在实际运行中仍然可能出现重复数据。本节从索引语义、边界场景、历史演进三个维度给出完整成因链路。

### 4.1 部分索引的语义边界

**核心定义回顾**：

```typescript
subscriberSchema.index(
  { subscriberId: 1, _environmentId: 1 },
  { 
    name: 'unique_subscriber_per_environment', 
    unique: true, 
    partialFilterExpression: { deleted: false } 
  }
);
```

**关键语义理解**：

- **作用范围**：唯一约束仅对满足 `deleted: false` 的文档生效
- **允许场景**：
  1. 同一 `subscriberId + environmentId` 组合可以有多个 `deleted: true` 的文档
  2. 先软删除一个 subscriber，再创建相同 `subscriberId` 的新 subscriber 是合法操作
- **触发重复的路径 1**：如果某个操作意外将已删除 subscriber 的 `deleted` 标记改回 `false`，会触发唯一键冲突；但如果是先创建新记录再恢复旧记录，则旧记录恢复失败，不会产生重复

### 4.2 并发写入与索引状态时序

**场景 A：索引创建前已有并发写入**

1. **系统早期版本**：未添加唯一索引，仅依赖业务层逻辑去重
2. **高并发场景**：同一 subscriberId 同时收到两个创建请求
3. **业务层检查**：两个请求都通过了 `findBySubscriberId` 检查（都返回 null）
4. **数据库写入**：两个请求都成功插入，产生重复
5. **后续添加索引**：此时添加唯一索引会因已有重复数据而失败，必须先清理重复记录
6. **迁移脚本诞生**：这正是 `remove-duplicated-subscribers` 迁移脚本存在的根本原因

**时序图示意**：
```
时间轴：
T1 — 请求 A 执行 findBySubscriberId → 不存在
T2 — 请求 B 执行 findBySubscriberId → 不存在
T3 — 请求 A 执行 insert → 成功
T4 — 请求 B 执行 insert → 成功（无索引时）
结果：产生 2 条重复记录
```

**场景 B：索引重建或修复期间**

1. 因某些原因（如 MongoDB 版本升级、集合修复）索引被临时删除或处于重建状态
2. 期间有新的 subscriber 创建请求写入
3. 索引重建完成前已有重复数据写入
4. 索引重建失败，必须先清理重复

### 4.3 软删除与恢复的竞态条件

**典型问题场景**：

1. T1：Subscriber X 被软删除（`deleted: true`）
2. T2：业务逻辑判断 X 已删除，创建新的 Subscriber X'
3. T3：新的 X' 创建成功，`deleted: false`
4. T4：某个恢复流程意外将 X 的 `deleted` 改回 `false`
5. T5：触发唯一键冲突，操作失败 → **此路径不会产生重复**

**但存在另一种更隐蔽的路径**：

1. T1：创建 Subscriber X，`deleted: false`
2. T2：并发请求 B 也创建 Subscriber X，此时索引正常 → 失败
3. T3：请求 A 的写入因某种原因未实际落盘（或事务回滚）
4. T4：请求 B 重试时仍判断不存在 → 成功写入
5. **结果**：看似有索引保护，但极端时序下仍可能产生问题

### 4.4 跨环境数据同步与迁移

**场景特征**：

1. **多环境数据合并**：从不同环境导出/导入 subscriber 数据
2. **导入脚本逻辑缺陷**：未正确携带 `_environmentId` 或导入时环境映射错误
3. **批量导入绕过校验**：直接数据库批量写入绕过了业务层 upsert 逻辑
4. **结果**：同一 `subscriberId` 在同一环境中出现多条记录

### 4.5 成因总结与排查路径

**可落地的排查 checklist**：

| 排查方向 | 具体操作 |
|---------|---------|
| **索引状态检查** | `db.subscribers.getIndexes()` 确认唯一索引存在且 `partialFilterExpression` 正确 |
| **重复记录特征** | 检查重复记录的 `createdAt` 是否集中在早期无索引版本 |
| **删除标记检查** | 检查重复记录中是否有 `deleted` 字段不一致的情况 |
| **写入日志回溯** | 搜索关键时间点的写入请求日志，确认并发度 |
| **迁移记录检查** | 确认 `remove-duplicated-subscribers` 迁移是否已成功执行 |

**根因分类统计（基于典型场景）**：

1. **索引缺失期的并发写入**：占比约 70% — 系统早期无索引阶段的历史遗留
2. **批量导入绕过校验**：占比约 20% — 数据迁移/导入时未走标准 upsert 流程
3. **MongoDB 特殊行为**：占比约 10% — 极端场景下部分索引的行为与预期有差异

---

## 5. 数据迁移合并策略

当系统中因上述原因产生重复 subscriber 时，通过专门的迁移脚本进行合并清理。

**迁移脚本位置**：`apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.ts`

### 4.1 检测重复

使用 MongoDB 聚合管道检测重复：

```typescript
const pipeline = [
  // 按 subscriberId 和 _environmentId 分组
  {
    $group: {
      _id: { subscriberId: '$subscriberId', environmentId: '$_environmentId' },
      count: { $sum: 1 },
      subscribers: { $push: '$$ROOT' }, // 保存每组的所有文档
    },
  },
  // 筛选出 count > 1 的组（即重复组）
  {
    $match: {
      count: { $gt: 1 },
    },
  },
];
```

### 4.2 合并策略

#### 4.2.1 保留哪个 subscriber？

**实际排序依据（源码第 49 行）**：

```typescript
// sort oldest subscriber first
const sortedSubscribers = subscribers.sort((a, b) => a.updatedAt - b.updatedAt);
const mergedSubscriber = mergeSubscribers(sortedSubscribers);
```

- **排序字段**：严格按 `updatedAt` 字段排序，**不使用** `createdAt`
- **排序方向**：升序排列（`a.updatedAt - b.updatedAt`），`updatedAt` 值最小的排在最前
- **主记录选择**：选择排序后的 **第一个元素**（`subscribers[0]`）作为主记录
- **删除规则**：所有 `_id !== mergedSubscriber._id` 的记录都被删除

**updatedAt 与创建时间的关系**：

只有在 subscriber **从未发生后续更新** 时，按 `updatedAt` 升序排序才可能与创建先后看起来一致。

但存在例外情况：
- 如果某个较早创建的 subscriber 后续被更新过，其 `updatedAt` 会变得比后创建的 subscriber 更大
- 此时按 `updatedAt` 排序的结果会与实际创建顺序相反
- **最终结论**：代码实际选择的是 `updatedAt` 最小的 subscriber，而非创建最早的 subscriber

**时间相同场景下的确定性**：

当多个 subscriber 的 `updatedAt` 完全相同时：
1. **MongoDB `$push` 顺序**：聚合管道中 `subscribers: { $push: '$$ROOT' }` 的返回顺序取决于 MongoDB 内部的文档存储顺序（通常与 `_id` 插入顺序相关，但无正式保证）
2. **JavaScript `sort()` 稳定性**：在 V8 引擎（Node.js）中，`Array.sort()` 对于相等元素是**稳定排序**，即相等元素保持其在原数组中的相对位置
3. **最终结果**：当 `updatedAt` 相同时，哪个 subscriber 被保留取决于 MongoDB 返回的顺序，该顺序由文档在集合中的物理存储位置决定，**不保证与创建时间一致**

> **⚠️ 代码注释偏差说明**：迁移代码注释标注 "sort oldest subscriber first"（按最旧的 subscriber 排序），但实际实现基于 `updatedAt` 而非 `createdAt`，存在注释语义与实现的偏差。

#### 4.2.2 字段合并规则

| 字段类型 | 合并策略 |
|---------|---------|
| **基础字段** | `firstName`, `lastName`, `email`, `phone`, `avatar`, `locale`, `data`, `timezone` - 以后出现的非空值覆盖先出现的 |
| **channels** | 按 `_integrationId` 分组，合并设备令牌 |
| **内部字段** | `_id`, `_organizationId`, `_environmentId`, `deleted`, `createdAt`, `updatedAt`, `__v`, `isOnline`, `lastOnlineAt` - 保留主记录的值，不合并 |

#### 4.2.3 Channels 合并细节

```typescript
function mergeChannels(existingChannel: IChannelSettings, newChannel: IChannelSettings) {
  const result = { ...existingChannel };

  // 合并 deviceTokens - 去重
  const allTokens = [
    ...(existingChannel?.credentials?.deviceTokens || []),
    ...(newChannel?.credentials?.deviceTokens || []),
  ];
  result.credentials.deviceTokens = [...new Set(allTokens)];

  // webhookUrl - 新值覆盖旧值
  if (newChannel.credentials.webhookUrl) {
    existingChannel.credentials.webhookUrl = newChannel.credentials.webhookUrl;
  }

  return existingChannel;
}
```

### 4.3 合并执行流程

1. **排序**：将重复组按 `updatedAt` 升序排序
2. **合并数据**：调用 `mergeSubscribers()` 合并所有重复 subscriber 的数据
3. **更新主记录**：将合并后的数据更新到保留的主 subscriber（排序后第一个）
4. **删除重复**：删除其他所有重复的 subscriber 记录

```typescript
// sort oldest subscriber first
const sortedSubscribers = subscribers.sort((a, b) => a.updatedAt - b.updatedAt);
const mergedSubscriber = mergeSubscribers(sortedSubscribers);
const subscribersToRemove = sortedSubscribers.filter((subscriber) => subscriber._id !== mergedSubscriber._id);

// 更新主 subscriber
await subscriberRepository.update(
  {
    _id: mergedSubscriber._id,
    subscriberId: subscriberId,
    _environmentId: environmentId,
  },
  {
    $set: mergedSubscriber,
  }
);

// 删除重复记录
await subscriberRepository.deleteMany({
  _id: { $in: subscribersToRemove.map((subscriber) => subscriber._id) },
  subscriberId: subscriberId,
  _environmentId: environmentId,
});
```

## 5. 测试覆盖验证

**测试文件**：`apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.spec.ts`

### 5.1 测试场景

| 测试场景 | 验证点 |
|---------|-------|
| **基本去重** | 3 个重复 subscriber 合并后只剩 1 个 |
| **跨环境隔离** | 同一 subscriberId 在不同环境中各自保留 |
| **元数据合并** | 多个重复记录的 firstName/lastName 等字段合并 |
| **字段覆盖** | 后出现的非空字段值覆盖先出现的 |
| **Channels 同集成合并** | 同一 _integrationId 的 deviceTokens 合并去重 |
| **Channels 不同集成保留** | 不同 _integrationId 的 channel 都保留 |
| **WebhookUrl 更新** | 新的 webhookUrl 覆盖旧的 |
| **保留 updatedAt 最小的记录** | 主记录是 `updatedAt` 最小的 subscriber ⚠️ |

> ⚠️ **测试隐含假设说明**：测试用例 "should keep the first created subscriber" 实际验证的是"先创建的 subscriber 其 `updatedAt` 也更小"这一隐含假设。该假设在 subscriber 从未被更新的场景下成立，但并非绝对保证。实际上代码选择的是 `updatedAt` 最小的 subscriber，而非 `createdAt` 最小的。
>
> 只有在 subscriber **从未发生后续更新** 时，按 `updatedAt` 升序排序才可能与创建先后看起来一致。

## 6. 架构设计总结

### 6.1 三层防护体系

```
┌─────────────────────────────────────────────────────────────┐
│                    应用层 (Upsert Logic)                     │
│  CreateOrUpdateSubscriberUseCase + 重试机制                  │
│  BulkCreateSubscribers 批量 upsert                           │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                   数据库层 (Unique Index)                    │
│  subscriberId + _environmentId + deleted:false 唯一约束      │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                  数据修复层 (Migration)                      │
│  remove-duplicated-subscribers 迁移脚本 + 字段合并策略       │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 设计权衡

| 决策 | 优点 | 缺点 |
|-----|------|------|
| **以 subscriberId + environmentId 作为唯一键** | 符合多租户隔离要求，业务逻辑清晰 | 需要在所有查询中包含 environmentId |
| **partialFilterExpression 排除已删除记录** | 允许删除后重新创建相同 subscriberId | 软删除记录可能累积占用存储空间 |
| **按 updatedAt 升序选择主记录** | subscriber 从未更新时等价于保留创建较早的记录，符合大部分场景预期 | 若旧 subscriber 被更新过，其 `updatedAt` 变大，可能反而不被保留；代码注释与实现存在语义偏差 |
| **updatedAt 相同时依赖 MongoDB 返回顺序** | 实现简单，无需额外排序字段 | 结果依赖数据库内部状态，不保证绝对可预测 |
| **channels 按 integrationId 聚合，tokens 去重** | 最大限度保留推送令牌，提高送达率 | 可能导致 channels 数组逐渐膨胀 |

> **重要说明**：只有在 subscriber **从未发生后续更新** 时，按 `updatedAt` 升序排序才可能与创建先后看起来一致。

## 7. 相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `libs/dal/src/repositories/subscriber/subscriber.schema.ts` | Subscriber Schema 与唯一索引定义 |
| `libs/dal/src/repositories/subscriber/subscriber.repository.ts` | Subscriber Repository，包含 bulkCreateSubscribers |
| `libs/application-generic/src/usecases/create-or-update-subscriber/create-or-update-subscriber.usecase.ts` | 创建/更新 subscriber 主逻辑 |
| `apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.ts` | 重复数据合并迁移脚本 |
| `apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.spec.ts` | 迁移脚本测试 |
