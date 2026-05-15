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

## 4. 数据迁移合并策略

当系统中因历史原因或索引缺失导致存在重复 subscriber 时，通过专门的迁移脚本进行合并清理。

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

- 保留 **最早创建**（按 `_id` 或 `updatedAt` 排序）的 subscriber 作为主记录
- 删除其他重复的 subscriber 记录

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

1. **排序**：将重复组按时间排序（最早的在前）
2. **合并数据**：调用 `mergeSubscribers()` 合并所有重复 subscriber 的数据
3. **更新主记录**：将合并后的数据更新到保留的主 subscriber
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
| **保留最早创建** | 主记录始终是最早创建的那个 |

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
| **合并时保留最早记录** | 保持 _id 的引用稳定性，不影响外键关联 | 可能丢失较新记录的创建时间信息 |
| **channels 按 integrationId 聚合，tokens 去重** | 最大限度保留推送令牌，提高送达率 | 可能导致 channels 数组逐渐膨胀 |

## 7. 相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `libs/dal/src/repositories/subscriber/subscriber.schema.ts` | Subscriber Schema 与唯一索引定义 |
| `libs/dal/src/repositories/subscriber/subscriber.repository.ts` | Subscriber Repository，包含 bulkCreateSubscribers |
| `libs/application-generic/src/usecases/create-or-update-subscriber/create-or-update-subscriber.usecase.ts` | 创建/更新 subscriber 主逻辑 |
| `apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.ts` | 重复数据合并迁移脚本 |
| `apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.spec.ts` | 迁移脚本测试 |
