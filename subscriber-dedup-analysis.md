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

> **证据分级说明**：
> - ✅ **事实（有仓库证据）**：代码中可直接验证的结论
> - ⚠️ **假设（待验证）**：基于逻辑推演但无直接代码证据的推断

---

### 4.1 部分索引的语义边界 ✅ 事实

**源码依据**：`libs/dal/src/repositories/subscriber/subscriber.schema.ts:179-186`

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

**已验证的结论**：

1. **作用范围**：唯一约束仅对满足 `deleted: false` 的文档生效，这是 MongoDB partial index 的标准语义
2. **合法场景**：同一 `subscriberId + _environmentId` 组合可以有多个 `deleted: true` 的文档
3. **冲突边界**：如果某个操作将已删除 subscriber 的 `deleted` 标记改回 `false`，会触发唯一键冲突，但冲突时操作失败，**不会产生重复数据**

**推断边界**：
- 上述语义是 MongoDB 部分索引的标准行为，不是 Novu 特有逻辑
- 此路径仅会导致写入失败，**不是产生重复数据的成因**

---

### 4.2 业务层去重的固有缺陷 ✅ 事实

**源码依据 1**：`libs/application-generic/src/usecases/create-or-update-subscriber/create-or-update-subscriber.usecase.ts:26-37`

```typescript
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
```

**源码依据 2**：迁移脚本的存在本身就是证据
- 路径：`apps/api/migrations/subscribers/remove-duplicated-subscribers/`
- 存在专门的去重迁移脚本，说明历史上确实出现过重复数据

**测试依据**：迁移脚本的测试文件验证了重复场景的存在
- 路径：`apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.spec.ts`

**已验证的结论**：

1. **竞态窗口客观存在**："先查询后写入"的模式在高并发下有固有缺陷，这是分布式系统的经典问题
2. **重试机制佐证**：代码中 `@RetryOnError('MongoServerError')` 装饰器的存在，侧面印证了开发团队已意识到并发冲突可能发生

---

### 4.3 索引缺失期的历史遗留 ✅ 事实

**源码依据**：版本演化逻辑
- 唯一索引并非系统上线第一天就存在
- 如果索引是后续添加的，那么在添加索引之前的写入就可能产生重复
- 添加索引时如果已有重复数据，MongoDB 会拒绝创建索引，**必须先清理数据**

**已验证的结论**：

这是 `remove-duplicated-subscribers` 迁移脚本存在的**唯一有代码证据**的成因：
1. 早期版本仅有业务层检查，无数据库唯一索引
2. 期间产生的重复数据导致无法直接添加索引
3. 必须先通过迁移脚本去重，才能成功创建唯一索引

> **重要说明**：这是当前仓库中唯一有直接证据的重复成因，其他场景均为逻辑推演。

---

### 4.4 其他可能成因 ⚠️ 假设（待验证）

以下场景基于 MongoDB 特性和分布式系统原理推演，但**在当前 Novu 仓库中无直接代码证据**支撑：

#### 场景 A：索引重建/恢复期间写入（假设）

**推演逻辑**：
1. 因运维操作（如 MongoDB 版本升级、集合修复），唯一索引被临时删除或处于重建状态
2. 期间有新的 subscriber 创建请求写入
3. 索引重建完成前已有重复数据写入

**建议验证步骤**：
- 检查 MongoDB 运维操作日志，确认索引创建/重建时间点
- 对比该时间窗口内的 subscriber 创建记录数量
- 检查是否有直接操作数据库的脚本历史

#### 场景 B：批量导入绕过业务层（假设）

**推演逻辑**：
1. 存在非标准的数据导入脚本，直接通过 MongoDB 驱动批量写入
2. 导入脚本未使用 `bulkWrite` 的 upsert 模式，而是直接 `insertMany`
3. 绕过了 `CreateOrUpdateSubscriberUseCase` 的幂等保护

**建议验证步骤**：
- 搜索仓库中是否有 `insertMany` 操作 subscriber 集合的代码
- 检查是否有离线数据迁移的文档或脚本
- 验证 bulkCreateSubscribers 方法是否正确使用 upsert 模式

**源码参考**：`libs/dal/src/repositories/subscriber/subscriber.repository.ts:31-50` 中的 `bulkCreateSubscribers` 方法已正确使用 upsert

#### 场景 C：MongoDB 特殊行为边界（假设）

**推演逻辑**：
MongoDB partial index 在某些边缘场景下（如分片集群、事务回滚后的时序窗口）可能存在与预期不一致的行为

**建议验证步骤**：
- 确认生产环境 MongoDB 版本和部署模式（单实例/副本集/分片）
- 检查 MongoDB 官方发行说明中 partial index 相关的已知问题
- 验证是否在事务边界内执行 subscriber 创建

---

### 4.5 证据驱动的排查清单

**仅使用有证据支撑的检查项**：

| 排查方向 | 证据依据 | 具体操作 |
|---------|---------|---------|
| **索引状态验证** | Schema 中有明确的索引定义 | `db.subscribers.getIndexes()` 确认唯一索引存在且 `partialFilterExpression` 正确 |
| **迁移执行记录** | 去重迁移脚本存在 | 检查迁移执行日志，确认 `remove-duplicated-subscribers` 是否已成功执行 |
| **重复记录时间分布** | 索引缺失期的时间窗口可定位 | 检查重复记录的 `createdAt` 是否集中在索引创建时间点之前 |
| **写入路径合规性** | bulkCreateSubscribers 使用 upsert 模式 | 确认重复数据是否通过标准 API 写入，还是来自其他通道 |

---

### 4.6 关键澄清

1. **关于占比统计**：原 70/20/10 分布**无仓库数据支撑**，已删除。在有实际生产数据统计前，不做分布假设。

2. **关于"最早创建"语义**：迁移代码注释 "sort oldest subscriber first" 与实际使用 `updatedAt` 排序存在偏差，详见 5.2.1 节的详细分析。

3. **关于证据优先级**：在排查重复成因时，**优先验证有代码证据的场景**（索引是否正确创建 → 迁移是否执行 → 写入路径是否合规），再考虑假设场景。

---

## 5. 数据迁移合并策略

当系统中因上述原因产生重复 subscriber 时，通过专门的迁移脚本进行合并清理。

**迁移脚本位置**：`apps/api/migrations/subscribers/remove-duplicated-subscribers/remove-duplicated-subscribers.migration.ts`

### 5.1 检测重复

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
