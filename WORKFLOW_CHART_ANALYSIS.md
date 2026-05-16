# Novu 工作流发送量图表查询机制 - 代码对账分析报告

## 目录
1. [跨租户数据隔离机制 - 逐层代码对账](#1-跨租户数据隔离机制---逐层代码对账)
2. [四条图表路由的分流对账](#2-四条图表路由的分流对账)
3. [查询路由逻辑 - 明细表 vs 聚合表](#3-查询路由逻辑---明细表-vs-聚合表)
4. [名称映射与权限校验的关系](#4-名称映射与权限校验的关系)
5. [🔒 跨环境名称映射泄露风险 - 细化对账](#5--跨环境名称映射泄露风险---细化对账)
6. [数据迁移兼容边界 - 可复核风险清单](#6-数据迁移兼容边界---可复核风险清单)
7. [关键 SQL 查询对比](#7-关键-sql-查询对比)
8. [时间分桶逻辑差异](#8-时间分桶逻辑差异)
9. [状态枚举兼容处理](#9-状态枚举兼容处理)

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
│ │    └─ ❌ 无 _organizationId 过滤 (核心安全风险)          │ │
│ │ 3. 名称映射后返回给前端                                   │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. 🔒 跨环境名称映射泄露风险 - 细化对账

### 5.1 风险全景总览

| 风险项 | 详情 |
|-------|-----|
| **风险编号** | R6 - 跨环境名称映射泄露 |
| **CWE 分类** | CWE-284: Improper Access Control |
| **影响组件** | Workflow By Volume 图表 - 聚合表路径 |
| **风险等级** | ⚠️ 中 (理论存在，实际需特定触发条件) |
| **发现位置** | `libs/dal/src/repositories/notification-template/notification-template.repository.ts:91-111` |

---

### 5.2 风险触发条件拆解

#### ✅ 触发前提 1: 聚合表 Feature Flag 开启
```
触发条件: IS_WORKFLOW_RUN_COUNT_ENABLED = true
验证方法:
  1. 查询 LaunchDarkly / Flagsmith 配置
  2. 检查组织维度 Flag 是否开启
  3. 确认环境维度 Flag 是否开启
影响范围: 仅开启 Flag 的组织受此风险影响
```

#### ✅ 触发前提 2: 跨组织存在相同 trigger_identifier
```
触发条件: OrgA 和 OrgB 在同一 Environment 下定义了相同的 trigger_identifier
验证方法:
  1. 在 NotificationTemplate 集合执行聚合查询:
     db.notification_templates.aggregate([
       { $group: { 
         _id: { trigger: "$triggers.identifier", env: "$_environmentId" },
         count: { $sum: 1 },
         orgs: { $addToSet: "$_organizationId" }
       }},
       { $match: { count: { $gt: 1 }}}
     ])
  2. 检查查询结果是否存在 orgId 数量 > 1 的记录
影响范围: 仅 identifier 冲突的工作流名称
```

#### ✅ 触发前提 3: User 可访问冲突 Environment 但不可访问冲突 Organization
```
触发条件:
  - User 有 EnvironmentA 的访问权限 (Member/Admin)
  - User 无 OrganizationB 的访问权限
  - OrganizationB 在 EnvironmentA 下有冲突 identifier
验证方法:
  1. 创建跨组织成员测试账号
  2. 验证该账号组织权限边界
  3. 访问冲突 identifier 工作流图表观察响应
影响范围: 多组织共享环境的部署场景 (Enterprise Plan)
```

---

### 5.3 现有代码防护机制分析

| 防护层级 | 防护措施 | 有效性 | 说明 |
|---------|---------|--------|-----|
| **L1: Controller 层** | `organizationId` 从 JWT Session 注入 | ✅ 有效 | 防止用户通过 API 参数篡改组织 ID |
| **L2: UseCase 层** | 传入 `environmentId` 给 Mongo 查询 | ✅ 有效 | 限制仅查询用户所在环境的模板 |
| **L3: Repository 层** | `_environmentId` 查询条件 | ✅ 有效 | Mongo 层面确保环境隔离 |
| **L4: Repository 层** | `_organizationId` 查询条件 | ❌ 缺失 | 未按组织过滤，跨组织 identifier 冲突时名称泄露 |

**防护代码证据**：

```typescript
// 文件位置: notification-template.repository.ts:91-111
async findByTriggerIdentifierBulk<K extends keyof NotificationTemplateEntity>(
  environmentId: string,    // ✅ 强制传入环境
  identifiers: string[],
  options: { session?: ClientSession | null; select?: K[] } = {}
): Promise<NotificationTemplateEntity[] | Pick<NotificationTemplateEntity, K>[]> {
  const requestQuery: NotificationTemplateQuery = {
    _environmentId: environmentId,  // ✅ L3: 仅按环境过滤
    'triggers.identifier': { $in: identifiers },
    // ⚠️ L4 缺失: 缺少 _organizationId 过滤条件
  };

  const items = await this.MongooseModel.find(requestQuery, projection, { session });
  // 返回匹配的模板名称
}
```

---

### 5.4 实际影响范围矩阵

| 部署模式 | 受影响可能性 | 风险说明 |
|---------|-------------|---------|
| **单组织私有部署** | ❌ 无风险 | 仅一个组织，identifier 冲突不跨组织 |
| **多组织共享环境 (SaaS)** | ⚠️ 中风险 | 多组织共用环境，存在恶意构造 identifier 可能性 |
| **企业多租户隔离部署** | ✅ 低风险 | 每个组织独立部署，环境物理隔离 |

| 用户角色 | 受影响可能性 | 风险说明 |
|---------|-------------|---------|
| **组织 Owner/Admin** | ❌ 无风险 | 可访问组织内所有工作流 |
| **组织 Member** | ❌ 无风险 | 已在组织边界内 |
| **跨组织用户 (SaaS)** | ⚠️ 中风险 | 理论上可能看到其他组织同名工作流名称 |

---

### 5.5 剩余风险量化评估

| 剩余风险项 | 概率 | 影响 | 风险值 | 缓解措施 |
|-----------|------|-----|-------|---------|
| 跨组织 identifier 名称泄露 | 低 (需刻意冲突) | 中 (仅名称泄露) | 中 | 添加 `_organizationId` 查询条件 |
| 写入时跨环境名称污染 | 极低 | 中 | 低 | 写入时校验名称来源组织 |
| 聚合表数据越权查询 | 极低 | 高 | 中 | 完整实现 ClickHouse 双条件隔离 |

---

### 5.6 可直接执行的验证步骤清单

#### 🔍 验证步骤 1: 检查代码中 organizationId 过滤缺失

**执行路径**:

```bash
# 1. 定位到目标文件
cd libs/dal/src/repositories/notification-template

# 2. 查看 findByTriggerIdentifierBulk 方法实现
cat notification-template.repository.ts | grep -A 25 "findByTriggerIdentifierBulk"

# 预期观察点:
# ✅ 存在: _environmentId: environmentId
# ❌ 缺失: _organizationId 过滤条件
```

**验证命令输出示例**:
```typescript
const requestQuery: NotificationTemplateQuery = {
  _environmentId: environmentId,  // ✅ 存在
  'triggers.identifier': { $in: identifiers },
  // ❌ 此处缺少 _organizationId 条件
};
```

---

#### 🔍 验证步骤 2: 数据库中 identifier 冲突检测

**Mongo Shell 执行**:

```javascript
// 连接到 Novu 主数据库
use novu;

// 查询: 同一 Environment 下跨组织的 identifier 冲突
db.notification_templates.aggregate([
  { $unwind: "$triggers" },
  { $group: {
    _id: {
      trigger_identifier: "$triggers.identifier",
      environment_id: "$_environmentId"
    },
    count: { $sum: 1 },
    organization_ids: { $addToSet: "$_organizationId" },
    template_names: { $addToSet: "$name" }
  }},
  { $match: {
    count: { $gt: 1 },
    $expr: { $gt: [{ $size: "$organization_ids" }, 1] }
  }},
  { $project: {
    trigger_identifier: "$_id.trigger_identifier",
    environment_id: "$_id.environment_id",
    conflict_count: "$count",
    organization_ids: 1,
    template_names: 1,
    _id: 0
  }}
]).pretty();
```

**结果解读**:
- 空结果: ✅ 无冲突
- 有结果: ⚠️ 发现跨组织 identifier 冲突，记录冲突详情

---

#### 🔍 验证步骤 3: 构造冲突场景功能测试

**测试准备**:
1. 两个组织账号: OrgA, OrgB
2. 创建同一 Environment (env-123)
3. OrgA 创建工作流 trigger: `test-conflict-trigger`
4. OrgB 创建工作流 trigger: `test-conflict-trigger`

**测试执行**:
```typescript
// 1. 使用 OrgA 的用户 Token 调用图表 API
// GET /v1/activity/charts?reportType[]=workflow-by-volume

// 2. 验证响应中的工作流名称列表
// ✅ 预期: 仅包含 OrgA 的工作流名称
// ❌ 异常: 出现 OrgB 的工作流名称

// 3. 使用 curl 实际验证
curl -X GET "https://api.novu.co/v1/activity/charts?reportType[]=workflow-by-volume" \
  -H "Authorization: ApiKey <ORG_A_API_KEY>" \
  -H "Content-Type: application/json"

// 检查响应 data 中的工作流名称
```

---

#### 🔍 验证步骤 4: 对比明细/聚合双路径查询结果

**验证点**: 同环境同用户，开启/关闭 Flag 的名称列表一致性

```typescript
// 1. 关闭 Feature Flag (IS_WORKFLOW_RUN_COUNT_ENABLED = false)
// 查询明细表路径，获取工作流名称列表 → 基准结果

// 2. 开启 Feature Flag (IS_WORKFLOW_RUN_COUNT_ENABLED = true)
// 查询聚合表路径，获取工作流名称列表 → 验证结果

// 3. 对比两次结果的差异
const expected = new Set(['Workflow-A', 'Workflow-B']);  // 明细表结果
const actual = new Set(['Workflow-A', 'Workflow-B', 'Workflow-From-OrgB']);  // 聚合表结果

// ✅ 一致: 无越权泄露
// ❌ 差异: actual 包含 expected 以外的项目
```

---

#### 🔍 验证步骤 5: 代码修复验证

**修复代码示例**：

```typescript
// 文件: notification-template.repository.ts
// 修改前:
async findByTriggerIdentifierBulk(
  environmentId: string,
  identifiers: string[],
  options: { session?: ClientSession | null; select?: K[] } = {}
) {
  const requestQuery = {
    _environmentId: environmentId,
    'triggers.identifier': { $in: identifiers },
    // ⚠️ 缺少组织过滤
  };
  // ...
}

// 修改后:
async findByTriggerIdentifierBulk(
  environmentId: string,
  organizationId: string,  // ✅ 新增组织参数
  identifiers: string[],
  options: { session?: ClientSession | null; select?: K[] } = {}
) {
  const requestQuery = {
    _environmentId: environmentId,
    _organizationId: organizationId,  // ✅ 新增组织过滤
    'triggers.identifier': { $in: identifiers },
  };
  // ...
}
```

**回归验证**:
1. 修复后执行步骤 2 的数据库冲突检测
2. 即使存在 identifier 冲突，查询结果也应正确过滤
3. 执行步骤 3-4 的功能测试，确认名称列表仅包含当前组织工作流

---

#### 🔍 验证步骤 6: ClickHouse 聚合表组织隔离复核

```sql
-- 验证 WorkflowRunCountRepository 中的 organizationId 过滤
-- 实际执行的 SQL 检查:

SELECT
  workflow_run_id,
  sum(count) as count
FROM workflow_run_count
WHERE
  environment_id = {environmentId:String}
  AND organization_id = {organizationId:String}  -- ✅ 确认存在
  AND date >= {startDate:Date}
  AND date <= {endDate:Date}
  AND event_type = 'workflow_run_status_processing'
GROUP BY workflow_run_id
ORDER BY count DESC
LIMIT 5;
```

**验证点**:
- ✅ `organization_id` 条件存在且正确绑定参数
- ✅ 参数值来自 User Session，非用户可控输入
- ✅ ClickHouse 层面的组织隔离已生效

---

### 5.7 修复优先级与建议

| 建议修复项 | 优先级 | 修复成本 | 收益 |
|----------|--------|---------|-----|
| 1. `findByTriggerIdentifierBulk` 增加 `_organizationId` 过滤 | 🔴 高 | 低 (仅修改 query 条件) | 完全消除越权风险 |
| 2. 向上游方法传递 `organizationId` 参数 | 🟡 中 | 中 (多处调用链) | 完整修复安全漏洞 |
| 3. 数据库唯一约束: `(_organizationId, _environmentId, triggers.identifier)` | 🟡 中 | 中 (DDL + 数据清理) | 从源头防止冲突 |
| 4. 创建工作流时校验 trigger 唯一性 | 🟢 低 | 中 | 预防冲突产生 |

**最小修复代码路径**:

```typescript
// 1. 修改 Repository 方法签名和查询条件
// libs/dal/src/repositories/notification-template/notification-template.repository.ts

// 2. 修改 UseCase 调用处传入 organizationId
// apps/api/src/app/activity/usecases/build-workflow-by-volume-chart/build-workflow-by-volume-chart.usecase.ts:63
const templates = await this.notificationTemplateRepository.findByTriggerIdentifierBulk(
  environmentId,
  organizationId,  // ✅ 新增
  triggerIdentifiers,
  { select: ['name', 'triggers'] }
);
```

---

## 6. 数据迁移兼容边界 - 可复核风险清单

### 6.1 迁移架构概览

| 迁移脚本 | 核心变更 | 关键设计 |
|---------|---------|---------|
| `3_analytics_tables.sql` | 创建 `delivery_trend_counts`, `trace_rollup` | 引入 SummingMergeTree 聚合表 |
| `4_refactor_traces_schema.sql` | 创建 `traces_temp`, `workflow_run_count` | 双写过渡，新数据同步到 _temp 表 |
| `5_finalize_table_exchange.sql` | `EXCHANGE TABLES` 原子切换 | 零停机切换，可回滚 |

---

### 6.2 可复核风险清单 (Checklist)

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

#### ✅ 风险 R6: 跨环境名称映射泄露 (详细见第 5 节)
- **风险描述**: findByTriggerIdentifierBulk 仅按 _environmentId 过滤，不按 _organizationId 过滤
- **校验点**: Mongo 查询条件仅包含 `_environmentId`，缺少 `_organizationId`
- **核查方法**: 见第 5.6 节详细验证步骤
- **影响范围**: Workflow By Volume (聚合表路径)
- **风险等级**: ⚠️ 中

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

### 6.3 回滚操作验证清单

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
| **NotificationTemplate 名称映射** | **`libs/dal/src/repositories/notification-template/notification-template.repository.ts`** |
| 迁移脚本 4 | `apps/api/migrations/clickhouse-migrations/4_refactor_traces_schema.sql` |
| 迁移脚本 5 | `apps/api/migrations/clickhouse-migrations/5_finalize_table_exchange.sql` |

---

**报告版本**: v3.0 (安全风险细化版)
**生成时间**: 2026-05-16
**代码版本**: Novu v192 工作流图表模块
**核心更新**: 新增第 5 节 - 跨环境名称映射泄露风险的完整细化对账，包含可执行验证步骤
