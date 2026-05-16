# Novu 工作流发送量图表查询机制 - 代码对账分析报告

## 目录
1. [跨租户数据隔离机制 - 逐层代码对账](#1-跨租户数据隔离机制---逐层代码对账)
2. [四条图表路由的分流对账](#2-四条图表路由的分流对账)
3. [查询路由逻辑 - 明细表 vs 聚合表](#3-查询路由逻辑---明细表-vs-聚合表)
4. [名称映射与权限校验的关系](#4-名称映射与权限校验的关系)
5. [数据迁移兼容边界 - 可复核风险清单](#5-数据迁移兼容边界---可复核风险清单)
6. [关键 SQL 查询对比](#6-关键-sql-查询对比)
7. [时间分桶逻辑差异](#7-时间分桶逻辑差异)
8. [状态枚举兼容处理](#8-状态枚举兼容处理)

---

## 1. 跨租户数据隔离机制 - 逐层代码对账

### 1.1 环境隔离自动注入 vs 组织隔离显式条件

| 隔离维度 | 生效位置 | 注入方式 | 保障机制 |
|---------|---------|---------|---------|
| **environment_id (环境隔离)** | LogRepository 基类 `buildEnforcedConditions` | ✅ **自动注入** | TypeScript 类型系统强制 `EnforcedContext`，所有查询通过基类自动追加 |
| **organization_id (组织隔离)** | 各 Repository 具体方法 | ⚠️ **显式硬编码** | 无类型系统保障，依赖各方法手动添加 WHERE 条件 |

**关键位置代码**：

```typescript
// 文件位置: libs/application-generic/src/services/analytic-logs/log.repository.ts:203-213
private buildEnforcedConditions(enforced: EnforcedContext): WhereCondition<InferClickhouseSchemaType<TSchema>>[] {
  const condition = {
    field: 'environment_id' as keyof InferClickhouseSchemaType<TSchema>,
    operator: '=' as const,
    value: enforced.environmentId,  // ✅ 环境隔离：自动强制注入
  };
  return [condition];
  // ⚠️ 注意：这里只注入 environment_id，不注入 organization_id
  // organization_id 需要每个 Repository 方法手动显式添加
}
```

### 1.2 四层强制隔离架构

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: 认证与授权 (Controller)                        │
│ - @RequireAuthentication() 强制登录                      │
│ - @RequirePermissions(PermissionsEnum.NOTIFICATION_READ)  │
│ - 从 UserSession 注入 organizationId / environmentId     │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 2: UseCase 命令对象封装                            │
│ - GetChartsCommand.create({ ...query,                    │
│     organizationId: user.organizationId,                 │
│     environmentId: user.environmentId })                 │
│ - DTO 不允许传入 organizationId / environmentId          │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 3: Repository WHERE 条件构建                       │
│ - environment_id: 基类自动注入 ✅                        │
│ - organization_id: 各方法显式硬编码 ⚠️                   │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Layer 4: ClickHouse 存储层索引优化                        │
│ - ORDER BY (organization_id, environment_id, ...)        │
│ - 主键前缀保证租户数据连续存储，查询时跳过无关分区         │
└─────────────────────────────────────────────────────────┘
```

---

### 1.3 第一层: Controller 认证注入

**文件位置**: `apps/api/src/app/activity/activity.controller.ts:121-140`

```typescript
@Get('charts')
@RequirePermissions(PermissionsEnum.NOTIFICATION_READ)
async getCharts(
  @UserSession() user: UserSessionData,      // 强制从会话获取
  @Query() query: GetChartsRequestDto        // DTO 中无 organizationId
): Promise<GetChartsResponseDto> {
  return this.getChartsUsecase.execute(
    GetChartsCommand.create({
      ...query,
      // ✅ 强制注入，不允许用户传入
      organizationId: user.organizationId,   // Line: 136
      environmentId: user.environmentId,     // Line: 137
    })
  );
}
```

**安全关键点**:
- `GetChartsRequestDto` 中**不包含** `organizationId` 和 `environmentId` 字段
- ID 只能从加密 JWT 会话中提取，无法通过 API 参数篡改

---

### 1.4 第二层: UseCase 双路径隔离

所有四条图表路由均强制传递 `environmentId` 和 `organizationId`:

| 图表路由 | environmentId | organizationId | workflowIds 支持 |
|---------|--------------|---------------|-----------------|
| Workflow By Volume | ✅ 强制 | ✅ 强制 | 明细表 ✅ / 聚合表 ❌ |
| Workflow Runs Trend | ✅ 强制 | ✅ 强制 | 明细表 ✅ / 聚合表 ❌ |
| Workflow Runs Count | ✅ 强制 | ✅ 强制 | 明细表 ✅ / 聚合表 ❌ |
| Workflow Runs Metric | ✅ 强制 | ✅ 强制 | ❌ 仅明细表 |

---

### 1.5 第三层: Repository SQL 硬编码隔离

#### 1.5.1 所有查询的统一模式

```sql
-- 每条 ClickHouse 查询都必须包含双隔离条件
WHERE
  environment_id = {environmentId:String}    -- ✅ 基类自动注入 OR 方法内硬编码
  AND organization_id = {organizationId:String}  -- ✅ 各方法显式硬编码
```

#### 1.5.2 各 Repository 的实际实现

**WorkflowRunCountRepository (聚合表)** - 双条件均显式硬编码:
```typescript
// 文件位置: workflow-run-count.repository.ts:266-267
WHERE
  environment_id = {environmentId:String}    -- Line: 266
  AND organization_id = {organizationId:String}  -- Line: 267
```

**WorkflowRunRepository (明细表)** - 双条件均显式硬编码:
```typescript
// 文件位置: workflow-run.repository.ts:472-473
WHERE
  environment_id = {environmentId:String}    -- Line: 472
  AND organization_id = {organizationId:String}  -- Line: 473
```

**⚠️ 重要发现**:
- `LogRepository.buildEnforcedConditions()` **仅自动注入 `environment_id`**
- `organization_id` 没有类型系统保障，**完全依赖各方法手动添加**
- 遗漏 `organization_id` 的查询会导致跨组织数据泄露风险

---

### 1.6 第四层: ClickHouse 存储层索引隔离

| 表名 | ORDER BY 排序键 | 设计意图 |
|------|----------------|---------|
| `workflow_runs` | `(organization_id, workflow_run_id)` | 租户级查询跳过 99% 数据块 |
| `workflow_run_count` | `(organization_id, environment_id, event_type, date, workflow_run_id)` | 四重前缀索引，环境级秒级响应 |
| `traces` (新) | `(organization_id, environment_id, entity_type, toDate(created_at), entity_id)` | 迁移优化后的新排序键 |

---

## 2. 四条图表路由的分流对账

### 2.1 分流决策树

```
传入查询请求
    ↓
Check Feature Flag: IS_WORKFLOW_RUN_COUNT_ENABLED
    │
    ├─ true → 走 workflow_run_count 聚合表 (SummingMergeTree)
    │     ├─ ✅ Workflow By Volume
    │     ├─ ✅ Workflow Runs Trend
    │     ├─ ✅ Workflow Runs Count
    │     └─ ❌ Workflow Runs Metric (未实现，仅明细表)
    │
    └─ false → 走 workflow_runs 明细表 (ReplacingMergeTree)
          ├─ ✅ Workflow By Volume
          ├─ ✅ Workflow Runs Trend
          ├─ ✅ Workflow Runs Count
          └─ ✅ Workflow Runs Metric
```

---

### 2.2 各路由详细对账

#### 2.2.1 路由 1: Workflow By Volume (工作流排名)
**文件位置**: `build-workflow-by-volume-chart.usecase.ts`

| 路径 | 实现状态 | 关键特性 |
|-----|---------|---------|
| 明细表路径 | ✅ 完整 | 直接获取 workflow_name，支持 workflowIds 过滤 |
| 聚合表路径 | ✅ 完整 | 通过 trigger_identifier 反向查询名称映射 |
| Feature Flag 控制 | ✅ 有效 | organization + environment 双维度 |

#### 2.2.2 路由 2: Workflow Runs Trend (趋势图表)
**文件位置**: `build-workflow-runs-trend-chart.usecase.ts`

| 路径 | 实现状态 | 关键特性 |
|-----|---------|---------|
| 明细表路径 | ✅ 完整 | 支持 pending/success 旧值兼容，支持 workflowIds 过滤 |
| 聚合表路径 | ✅ 完整 | 仅支持新标准状态值，不支持 workflowIds 过滤 |
| Feature Flag 控制 | ✅ 有效 | organization + environment 双维度 |

#### 2.2.3 路由 3: Workflow Runs Count (运行总数)
**文件位置**: `build-workflow-runs-count-chart.usecase.ts`

| 路径 | 实现状态 | 关键特性 |
|-----|---------|---------|
| 明细表路径 | ✅ 完整 | QueryBuilder 构建，支持 8 种过滤条件 |
| 聚合表路径 | ✅ 完整 | getTotalRunsCount，无过滤条件 |
| Feature Flag 控制 | ✅ 有效 | organization + environment 双维度 |

**明细表过滤条件支持**：
```typescript
// workflow-runs-count-chart.usecase.ts:85-127
if (workflowIds?.length) { queryBuilder.whereIn('workflow_id', workflowIds); }
if (subscriberIds?.length) { queryBuilder.whereIn('external_subscriber_id', subscriberIds); }
if (transactionIds?.length) { queryBuilder.whereIn('transaction_id', transactionIds); }
if (statuses?.length) { /* 新旧状态兼容映射 */ }
if (channels?.length) { queryBuilder.orWhere(...); }
if (topicKey) { queryBuilder.whereLike('topics', `%${topicKey}%`); }
```

#### 2.2.4 路由 4: Workflow Runs Metric (环比指标)
**文件位置**: `build-workflow-runs-metric-chart.usecase.ts`

| 路径 | 实现状态 | 关键特性 |
|-----|---------|---------|
| 明细表路径 | ✅ 完整 | 当期/上期双周期对比，支持 workflowIds 过滤 |
| 聚合表路径 | ❌ **缺失** | WorkflowRunCountRepository.getUsageReportStats 存在但未被调用 |
| Feature Flag 控制 | ❌ 无 | UseCase 未注入 FeatureFlagsService |

**⚠️ 代码缺失证据**：
```typescript
// build-workflow-runs-metric-chart.usecase.ts:1-13
// 仅注入 WorkflowRunRepository，缺少 WorkflowRunCountRepository 和 FeatureFlagsService
constructor(
  private workflowRunRepository: WorkflowRunRepository,
  private logger: PinoLogger
) {
  this.logger.setContext(BuildWorkflowRunsMetricChart.name);
}

// execute 方法直接调用明细表，无分流逻辑
const result = await this.workflowRunRepository.getWorkflowRunsMetricData(...);
```

---

### 2.3 分流完整性总结表

| 图表路由 | 明细表 | 聚合表 | Feature Flag | workflowIds 过滤支持 |
|---------|-------|-------|-------------|---------------------|
| Workflow By Volume | ✅ | ✅ | ✅ | 明细 ✅ / 聚合 ❌ |
| Workflow Runs Trend | ✅ | ✅ | ✅ | 明细 ✅ / 聚合 ❌ |
| Workflow Runs Count | ✅ | ✅ | ✅ | 明细 ✅ / 聚合 ❌ |
| **Workflow Runs Metric** | ✅ | ❌ **缺失** | ❌ 无 | 仅明细 ✅ |

---

## 3. 查询路由逻辑 - 明细表 vs 聚合表

### 3.1 双数据源路由决策

```typescript
// 文件位置: 各 UseCase execute 方法
const isWorkflowRunCountEnabled = await this.featureFlagsService.getFlag({
  key: FeatureFlagsKeysEnum.IS_WORKFLOW_RUN_COUNT_ENABLED,
  defaultValue: false,
  organization: { _id: organizationId },  // 组织级开关
  environment: { _id: environmentId },    // 环境级开关
});
```

**灰度能力**:
- 可按组织灰度开启
- 可按环境灰度开启
- 默认关闭 (保守策略)

---

### 3.2 功能矩阵对比

| 功能特性 | WorkflowRunRepository (明细表) | WorkflowRunCountRepository (聚合表) |
|---------|------------------------------|-----------------------------------|
| **查询路径** | `workflow_runs FINAL` | `workflow_run_count` |
| **存储引擎** | ReplacingMergeTree | SummingMergeTree |
| **时间字段** | `created_at` (DateTime64) | `date` (Date) |
| **时间精度** | 毫秒级 | 天级 |
| **工作流名称** | 直接 `workflow_name` 字段 | 反向查询 notification template |
| **工作流ID过滤** | ✅ 支持 | ❌ 不支持 |
| **订阅者过滤** | ✅ 支持 (Count 路由) | ❌ 不支持 |
| **事务ID过滤** | ✅ 支持 (Count 路由) | ❌ 不支持 |
| **状态过滤** | ✅ 支持 (Count 路由) | ❌ 不支持 |
| **渠道过滤** | ✅ 支持 (Count 路由) | ❌ 不支持 |
| **主题过滤** | ✅ 支持 (Count 路由) | ❌ 不支持 |
| **计数方式** | `count(*)` | `sum(count)` |
| **事件类型过滤** | 隐式 (所有状态) | 显式 `event_type = 'workflow_run_status_processing'` |

---

## 4. 名称映射与权限校验的关系

### 4.1 两条路径的权限校验差异

```
┌─────────────────────────────────────────────────────────────┐
│ 明细表路径 (workflow_runs)                                   │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ ClickHouse workflow_name 字段                            │ │
│ │ 来源: 写入时从 NotificationTemplate 预填充                │ │
│ │ 权限校验: ❌ 无 (查询时仅做租户过滤)                      │ │
│ │ 风险: 若写入时错误写入跨环境名称，查询会泄露              │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ 聚合表路径 (workflow_run_count)                              │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 1. ClickHouse workflow_run_id = trigger_identifier      │ │
│ │ 2. Mongo findByTriggerIdentifierBulk(environmentId, ...)│ │
│ │    └─ ✅ _environmentId 过滤 (Mongo 层面权限校验)        │ │
│ │    └─ ❌ 无 _organizationId 过滤 (潜在风险)              │ │
│ │ 3. 名称映射后返回给前端                                   │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

### 4.2 聚合表路径权限校验代码分析

**文件位置**: `libs/dal/src/repositories/notification-template/notification-template.repository.ts:91-111`

```typescript
async findByTriggerIdentifierBulk<K extends keyof NotificationTemplateEntity>(
  environmentId: string,    // ✅ 强制传入
  identifiers: string[],
  options: { session?: ClientSession | null; select?: K[] } = {}
): Promise<NotificationTemplateEntity[] | Pick<NotificationTemplateEntity, K>[]> {
  const requestQuery: NotificationTemplateQuery = {
    _environmentId: environmentId,  // ✅ 仅按环境过滤
    'triggers.identifier': { $in: identifiers },
    // ⚠️ 缺少 _organizationId 过滤!
  };

  const items = await this.MongooseModel.find(requestQuery, projection, { session });
  // 返回匹配的模板名称
}
```

**⚠️ 关键发现**:
- **明细表路径**: 名称来自 ClickHouse 存储的 `workflow_name` 字段，查询时**无额外权限校验**，仅依赖写入时的正确性
- **聚合表路径**: 通过 `findByTriggerIdentifierBulk` 反向查询时，**仅按 `_environmentId` 过滤**，**不按 `_organizationId` 过滤**
- **跨环境数据泄露风险**: 理论上不同环境存在相同 `trigger_identifier` 的可能

---

### 4.3 名称映射流程对比

| 阶段 | 明细表路径 | 聚合表路径 |
|-----|----------|----------|
| **数据来源** | ClickHouse `workflow_name` 字段 | `workflow_run_id` = `trigger_identifier` |
| **权限校验点** | 写入时校验，查询时无 | Mongo 查询校验 `_environmentId` |
| **组织过滤** | ClickHouse `organization_id` WHERE 条件 | ❌ Mongo 查询无 `_organizationId` 条件 |
| **映射方式** | 直接返回 | identifier → name 映射 |
| **失败处理** | 空名称直接显示 | 映射失败返回 identifier 原值 |

---

## 5. 数据迁移兼容边界 - 可复核风险清单

### 5.1 迁移架构概览

| 迁移脚本 | 核心变更 | 关键设计 |
|---------|---------|---------|
| `3_analytics_tables.sql` | 创建 `delivery_trend_counts`, `trace_rollup` | 引入 SummingMergeTree 聚合表 |
| `4_refactor_traces_schema.sql` | 创建 `traces_temp`, `workflow_run_count` | 双写过渡，新数据同步到 _temp 表 |
| `5_finalize_table_exchange.sql` | `EXCHANGE TABLES` 原子切换 | 零停机切换，可回滚 |

---

### 5.2 可复核风险清单 (Checklist)

#### ✅ 风险 R1: 历史数据 Backfill 完整性
- **风险描述**: 2026-02-03 之前的数据不会自动同步到新表，backfill 遗漏导致趋势图表断层
- **校验点**: 迁移 4 硬编码时间戳 `created_at > '2026-02-03 00:00:00'`
- **核查方法**:
  1. 对比新旧表同一天的计数差异：`SELECT count(*) FROM traces WHERE toDate(created_at) = '2026-02-01'`
  2. 验证 workflow_run_count 表最早数据日期
  3. 抽查 3 个历史日期的双表数据一致性
- **影响范围**: Workflow Runs Trend, Workflow By Volume, Workflow Runs Metric
- **风险等级**: ⚠️ 高

---

#### ✅ 风险 R2: 状态枚举兼容性缺失
- **风险描述**: 聚合表路径未处理 `pending`/`success` 旧状态值，历史数据统计不完整
- **校验点**:
  - 明细表路径兼容代码：`processing: pending + processing`, `completed: success + completed`
  - 聚合表路径仅处理：`workflow_run_status_processing`, `workflow_run_status_completed`, `workflow_run_status_error`
- **核查方法**:
  1. 查询明细表中 `status in ('pending', 'success')` 的数据占比
  2. 对比开启/关闭 Feature Flag 同一时间范围的数值差异
  3. 验证 status 字段的最后更新时间
- **影响范围**: Workflow Runs Trend
- **风险等级**: ⚠️ 中高

---

#### ✅ 风险 R3: 过滤条件功能降级
- **风险描述**: 聚合表路径不支持 8 种明细过滤条件，用户感知功能缺失
- **校验点**:
  - 明细表支持：workflowIds, subscriberIds, transactionIds, statuses, channels, topicKey
  - 聚合表支持：无任何额外过滤
- **核查方法**:
  1. 检查前端 UI 是否在 Flag 开启时隐藏过滤选项
  2. 验证传入 workflowIds 参数时聚合表路径是否忽略
  3. 对比开启/关闭 Flag 时的 API 响应结构一致性
- **影响范围**: Workflow Runs Count
- **风险等级**: ⚠️ 中

---

#### ✅ 风险 R4: 原子交换窗口数据丢失
- **风险描述**: EXCHANGE TABLES 与 DROP VIEW 之间的时间窗口，新写入数据可能丢失
- **校验点**: 迁移 5 执行顺序：EXCHANGE → DROP VIEW → CREATE VIEW
- **核查方法**:
  1. 验证迁移执行时间窗口内的消息发送量
  2. 对比交换前后 5 分钟的数据连续性
  3. 检查 MV 重建日志中的错误信息
- **影响范围**: 所有图表路由
- **风险等级**: ⚠️ 中

---

#### ✅ 风险 R5: Workflow Runs Metric 路径缺失
- **风险描述**: Metric 图表未实现聚合表路径，大租户查询性能问题无法缓解
- **校验点**:
  - UseCase 未注入 WorkflowRunCountRepository
  - 无 Feature Flag 分支逻辑
  - getUsageReportStats 方法存在但未被调用
- **核查方法**:
  1. 代码审计确认无分流逻辑
  2. 大租户 Metric 查询性能监控
  3. 验证 getUsageReportStats 方法的正确性
- **影响范围**: Workflow Runs Metric
- **风险等级**: ⚠️ 中

---

#### ✅ 风险 R6: 跨环境名称映射泄露
- **风险描述**: findByTriggerIdentifierBulk 仅按 _environmentId 过滤，不按 _organizationId 过滤
- **校验点**: Mongo 查询条件仅包含 `_environmentId`，缺少 `_organizationId`
- **核查方法**:
  1. 尝试跨环境使用相同 trigger_identifier 是否能成功
  2. 验证不同组织但同环境是否存在 identifier 冲突
  3. 审计 notification-template 表的 identifier 唯一性约束
- **影响范围**: Workflow By Volume (聚合表路径)
- **风险等级**: ⚠️ 低中

---

#### ✅ 风险 R7: 状态字段最终一致性延迟
- **风险描述**: ReplacingMergeTree 的 FINAL 查询在合并完成前返回旧状态值
- **校验点**: workflow_runs 表使用 `ReplacingMergeTree(updated_at)`，依赖后台合并
- **核查方法**:
  1. 验证 `OPTIMIZE TABLE workflow_runs FINAL` 执行频率
  2. 对比同一 workflow_run_id 的多次写入状态
  3. 监控查询中 status 不一致的比例
- **影响范围**: 所有明细表路径
- **风险等级**: ⚠️ 低

---

#### ✅ 风险 R8: 时间分桶时区偏移
- **风险描述**: 聚合表使用 Date 类型（UTC 天），明细表应用层可能产生时区偏移
- **校验点**: 明细表应用层 Gap Filling 使用本地时间还是 UTC
- **核查方法**:
  1. 验证 startDate/endDate 参数的时区处理
  2. 对比跨天时刻（如 UTC 00:00）的双表计数差异
  3. 检查 toDate(created_at) 与应用层日期格式化一致性
- **影响范围**: 所有图表
- **风险等级**: ⚠️ 低

---

### 5.3 回滚操作验证清单

```sql
-- =========================================
-- 发现问题后立即执行回滚
-- ==========================================

-- Step 1: 原子交换回旧表
EXCHANGE TABLES traces AND traces_temp;
EXCHANGE TABLES delivery_trend_counts AND delivery_trend_counts_temp;

-- Step 2: 重建临时 MV 恢复双写
CREATE MATERIALIZED VIEW IF NOT EXISTS traces_to_traces_temp_mv
TO traces_temp
AS SELECT ... FROM traces WHERE created_at > toDateTime64('2026-02-03 00:00:00', 3, 'UTC');

-- Step 3: 关闭 Feature Flag (应用层)
-- 将 IS_WORKFLOW_RUN_COUNT_ENABLED 设置为 false

-- Step 4: 验证回滚后数据正确性
-- 1. 查询最近 1 小时趋势图是否恢复正常
-- 2. 验证工作流排名 TOP 5 数值一致性
-- 3. 检查订阅者活跃度环比指标
```

---

## 6. 关键 SQL 查询对比

### 6.1 Workflow By Volume 图表查询

| 查询维度 | 明细表 SQL | 聚合表 SQL |
|---------|-----------|-----------|
| **FROM** | `workflow_runs FINAL` | `workflow_run_count` |
| **WHERE 租户** | `environment_id = ? AND organization_id = ?` | 相同 |
| **时间过滤** | `created_at >= ? AND created_at <= ?` | `date >= ? AND date <= ?` |
| **事件过滤** | 无 | `event_type = 'workflow_run_status_processing'` |
| **工作流过滤** | `AND workflow_id IN (?)` | ❌ 不支持 |
| **聚合字段** | `workflow_name` | `workflow_run_id` (需映射) |
| **计数** | `count(*)` | `sum(count)` |
| **LIMIT** | 5 | 5 |

---

### 6.2 Workflow Runs Trend 图表查询

| 查询维度 | 明细表 SQL | 聚合表 SQL |
|---------|-----------|-----------|
| **FROM** | `workflow_runs FINAL` | `workflow_run_count` |
| **时间过滤** | `created_at` DateTime64 | `date` Date |
| **分组粒度** | `toDate(created_at)` 天级 | `date` 天级 |
| **状态分组** | `status` 字段 (枚举) | `event_type` 字段 (字符串) |
| **兼容处理** | 兼容 `pending/success` 旧值 | 仅支持新标准值 |

---

### 6.3 Workflow Runs Count 图表查询

| 查询维度 | 明细表 SQL | 聚合表 SQL |
|---------|-----------|-----------|
| **FROM** | `workflow_runs FINAL` | `workflow_run_count` |
| **过滤条件** | 8 种动态条件构建 | 仅租户 + 时间范围 |
| **计数** | `count(*)` | `sum(count)` |
| **性能** | O(N) 全表扫描 | O(1) 预聚合 |

---

### 6.4 Workflow Runs Metric 图表查询 (仅明细表)

```sql
-- 当前周期
SELECT count(*) as count FROM workflow_runs FINAL
WHERE
  environment_id = {environmentId:String}
  AND organization_id = {organizationId:String}
  AND created_at >= {startDate:DateTime64(3)}
  AND created_at <= {endDate:DateTime64(3)}
  ${workflowFilter}

-- 前一周期
SELECT count(*) as count FROM workflow_runs FINAL
WHERE
  environment_id = {environmentId:String}
  AND organization_id = {organizationId:String}
  AND created_at >= {previousStartDate:DateTime64(3)}
  AND created_at <= {previousEndDate:DateTime64(3)}
  ${workflowFilter}
```

---

## 7. 时间分桶逻辑差异

### 7.1 明细表分桶逻辑

**文件位置**: `apps/api/src/app/activity/usecases/build-workflow-runs-trend-chart/build-workflow-runs-trend-chart.usecase.ts:109-125`

```typescript
// 应用层 Gap Filling - 确保每天都有数据点
const currentDate = new Date(startDate);
while (currentDate <= endDate) {
  const dateKey = currentDate.toISOString().split('T')[0];
  chartDataMap.set(dateKey, new Map([
    ['pending', 0],      // 向后兼容旧值
    ['processing', 0],
    ['success', 0],      // 向后兼容旧值
    ['completed', 0],
    ['error', 0],
  ]));
  currentDate.setDate(currentDate.getDate() + 1);
}
```

### 7.2 聚合表分桶逻辑

**文件位置**: `apps/api/src/app/activity/usecases/build-workflow-runs-trend-chart/build-workflow-runs-trend-chart.usecase.ts:55-67`

```typescript
const dataByDate = new Map<string, WorkflowRunsTrendDataPointDto>();

const currentDate = new Date(startDate);
while (currentDate <= endDate) {
  const dateKey = currentDate.toISOString().split('T')[0];
  dataByDate.set(dateKey, {
    timestamp: dateKey,
    processing: 0,  // ⚠️ 无兼容字段
    completed: 0,   // ⚠️ 无兼容字段
    error: 0,       // ⚠️ 无兼容字段
  });
  currentDate.setDate(currentDate.getDate() + 1);
}
```

**⚠️ 差异点**:
- 明细表路径保留 `pending` / `success` 向后兼容逻辑
- 聚合表路径无兼容逻辑，只支持新标准状态值
- 迁移后，历史数据的旧状态值可能统计缺失

---

## 8. 状态枚举兼容处理

### 8.1 状态枚举演进

**文件位置**: `libs/application-generic/src/services/analytic-logs/workflow-run/workflow-run.schema.ts:79-91`

```typescript
export enum WorkflowRunStatusEnum {
  /** @deprecated please use processing instead nv-6562 */
  PENDING = 'pending',
  PROCESSING = 'processing',
  /** @deprecated please use COMPLETED instead nv-6562 */
  SUCCESS = 'success',
  COMPLETED = 'completed',
  ERROR = 'error',
}
```

### 8.2 明细表路径兼容逻辑

```typescript
// build-workflow-runs-count-chart.usecase.ts:98-112
const mappedStatuses = statuses.map((status) => {
  // backward compatibility: if new statuses are used, append old status until renewed in the database, nv-6562
  if (status === WorkflowRunStatusDtoEnum.PROCESSING) {
    return [WorkflowRunStatusEnum.PENDING, WorkflowRunStatusEnum.PROCESSING];
  }
  if (status === WorkflowRunStatusDtoEnum.COMPLETED) {
    return [WorkflowRunStatusEnum.SUCCESS, WorkflowRunStatusEnum.COMPLETED];
  }
  if (status === WorkflowRunStatusDtoEnum.ERROR) {
    return [WorkflowRunStatusEnum.ERROR];
  }
  return status;
});
```

### 8.3 聚合表路径 - 无兼容

```typescript
// build-workflow-runs-trend-chart.usecase.ts:75-84
switch (workflowRun.event_type) {
  case 'workflow_run_status_processing':
    updatedDataPoint.processing += count;
    break;
  case 'workflow_run_status_completed':
    updatedDataPoint.completed += count;
    break;
  case 'workflow_run_status_error':
    updatedDataPoint.error += count;
    break;
  // ⚠️ 没有 pending / success 的映射处理
}
```

---

## 附录: 关键文件索引

| 模块 | 文件路径 |
|-----|---------|
| 图表控制器 | `apps/api/src/app/activity/activity.controller.ts` |
| 工作流容量 UseCase | `apps/api/src/app/activity/usecases/build-workflow-by-volume-chart/` |
| 工作流趋势 UseCase | `apps/api/src/app/activity/usecases/build-workflow-runs-trend-chart/` |
| 工作流计数 UseCase | `apps/api/src/app/activity/usecases/build-workflow-runs-count-chart/` |
| 工作流指标 UseCase | `apps/api/src/app/activity/usecases/build-workflow-runs-metric-chart/` |
| 明细表 Repository | `libs/application-generic/src/services/analytic-logs/workflow-run/` |
| 聚合表 Repository | `libs/application-generic/src/services/analytic-logs/workflow-run-count/` |
| 基类隔离逻辑 | `libs/application-generic/src/services/analytic-logs/log.repository.ts` |
| NotificationTemplate 名称映射 | `libs/dal/src/repositories/notification-template/notification-template.repository.ts` |
| 迁移脚本 4 | `apps/api/migrations/clickhouse-migrations/4_refactor_traces_schema.sql` |
| 迁移脚本 5 | `apps/api/migrations/clickhouse-migrations/5_finalize_table_exchange.sql` |

---

**报告版本**: v2.0 (补充核查版)
**生成时间**: 2026-05-16
**代码版本**: Novu v192 工作流图表模块
