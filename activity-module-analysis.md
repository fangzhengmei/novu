# Activity 模块完整分析报告

**生成日期**: 2026-05-14  
**分析范围**: apps/api/src/app/activity, libs/application-generic/src/services/analytic-logs

---

## 目录

1. [ReportTypeEnum 准确核对](#1-reporttypeenum-准确核对)
2. [Environment 与 Organization ID 注入机制](#2-environment-与-organization-id-注入机制)
3. [查询构建完整流程](#3-查询构建完整流程)
4. [关键代码引用与证据](#4-关键代码引用与证据)

---

## 1. ReportTypeEnum 准确核对

### 1.1 完整枚举定义

**文件位置**: `apps/api/src/app/activity/dtos/shared.dto.ts:106-118`

```typescript
export enum ReportTypeEnum {
  DELIVERY_TREND = 'delivery-trend',                    // #1
  INTERACTION_TREND = 'interaction-trend',              // #2
  WORKFLOW_BY_VOLUME = 'workflow-by-volume',            // #3
  PROVIDER_BY_VOLUME = 'provider-by-volume',            // #4
  MESSAGES_DELIVERED = 'messages-delivered',            // #5
  ACTIVE_SUBSCRIBERS = 'active-subscribers',            // #6
  AVG_MESSAGES_PER_SUBSCRIBER = 'avg-messages-per-subscriber',  // #7
  WORKFLOW_RUNS_METRIC = 'workflow-runs-metric',        // #8
  TOTAL_INTERACTIONS = 'total-interactions',            // #9
  WORKFLOW_RUNS_TREND = 'workflow-runs-trend',          // #10
  ACTIVE_SUBSCRIBERS_TREND = 'active-subscribers-trend', // #11
  WORKFLOW_RUNS_COUNT = 'workflow-runs-count',          // #12
}
```

> **事实校正**: 总共 **12** 种报告类型，不是 11 或 13 种。

### 1.2 Dashboard 端配置核对

**文件位置**: `apps/dashboard/src/components/analytics/constants/analytics-page.consts.ts:27-42`

```typescript
export const CHART_CONFIG = {
  reportTypes: [
    'DELIVERY_TREND',              // 1/12
    'INTERACTION_TREND',           // 2/12
    'WORKFLOW_BY_VOLUME',          // 3/12
    'PROVIDER_BY_VOLUME',          // 4/12
    'MESSAGES_DELIVERED',          // 5/12
    'ACTIVE_SUBSCRIBERS',          // 6/12
    'AVG_MESSAGES_PER_SUBSCRIBER', // 7/12
    'WORKFLOW_RUNS_METRIC',        // 8/12
    'TOTAL_INTERACTIONS',          // 9/12
    'WORKFLOW_RUNS_TREND',         // 10/12
    'ACTIVE_SUBSCRIBERS_TREND',    // 11/12
  ] as const,
  // ...
};
```

> **注意**: Dashboard 端只配置了 **11** 种，缺少 `WORKFLOW_RUNS_COUNT`（第 12 种）。

### 1.3 后端实现覆盖情况

| ReportType | 对应 Repository 方法 | 已实现 |
|------------|---------------------|--------|
| `DELIVERY_TREND` | `buildDeliveryTrendChart` | ✅ |
| `INTERACTION_TREND` | `buildInteractionTrendChart` | ✅ |
| `WORKFLOW_BY_VOLUME` | `buildWorkflowByVolumeChart` | ✅ |
| `PROVIDER_BY_VOLUME` | `buildProviderByVolumeChart` | ✅ |
| `MESSAGES_DELIVERED` | `buildMessagesDeliveredChart` | ✅ |
| `ACTIVE_SUBSCRIBERS` | `buildActiveSubscribersChart` | ✅ |
| `AVG_MESSAGES_PER_SUBSCRIBER` | `buildAvgMessagesPerSubscriberChart` | ✅ |
| `WORKFLOW_RUNS_METRIC` | `buildWorkflowRunsMetricChart` | ✅ |
| `TOTAL_INTERACTIONS` | `buildTotalInteractionsChart` | ✅ |
| `WORKFLOW_RUNS_TREND` | `buildWorkflowRunsTrendChart` | ✅ |
| `ACTIVE_SUBSCRIBERS_TREND` | `buildActiveSubscribersTrendChart` | ✅ |
| `WORKFLOW_RUNS_COUNT` | - | ❌ 枚举定义但未实现 |

---

## 2. Environment 与 Organization ID 注入机制

### 2.1 强制注入 vs 业务过滤

| 字段 | 注入方式 | 约束来源 | 强制执行 |
|------|---------|---------|---------|
| **environment_id** | ✅ QueryBuilder 强制注入（构造函数） | `EnforcedContext` 类型 | ✅ 总是执行 |
| **organization_id** | ⚠️ 业务逻辑层手动添加 | 用例代码中的 `whereEquals()` | ❌ 非强制 |

### 2.2 EnforcedContext 类型定义

**文件位置**: `libs/application-generic/src/services/analytic-logs/log.repository.ts:54-56`

```typescript
export type EnforcedContext = {
  environmentId: string;  // 唯一强制字段
};
```

> **关键发现**: `EnforcedContext` 类型 **只包含 environmentId**，不包含 organizationId！

### 2.3 QueryBuilder 构造与强制条件注入

**文件位置**: `libs/application-generic/src/services/analytic-logs/log.repository.ts:641-644, 872-877`

```typescript
export class QueryBuilder<T> {
  private conditions: WhereCondition<T>[] = [];

  constructor(private enforced: EnforcedContext) {}  // 只接收 environmentId

  // ...

  build(): EnforcedWhere<T> {
    return {
      enforced: this.enforced,     // environmentId 在此处
      conditions: this.conditions, // 业务条件
    };
  }
}
```

### 2.4 buildEnforcedConditions 实际执行

**文件位置**: `libs/application-generic/src/services/analytic-logs/log.repository.ts:203-213`

```typescript
private buildEnforcedConditions(enforced: EnforcedContext): WhereCondition<InferClickhouseSchemaType<TSchema>>[] {
  const condition = {
    field: 'environment_id' as keyof InferClickhouseSchemaType<TSchema>,
    operator: '=' as const,
    value: enforced.environmentId,  // 只强制注入 environment_id
  };

  const conditions: WhereCondition<InferClickhouseSchemaType<TSchema>>[] = [condition];

  return conditions;  // 返回只包含 environment_id 的条件数组
}
```

### 2.5 buildWhereClause 条件合并

**文件位置**: `libs/application-generic/src/services/analytic-logs/log.repository.ts:178-201`

```typescript
protected buildWhereClause(where: Where<TEnhancedType>): {
  clause: string;
  params: Record<string, unknown>;
} {
  // ...
  if ('__unsafe' in rawWhere) {
    // 不安全模式：跳过强制注入
    allConditions = rawWhere.conditions;
  } else {
    // 安全模式：强制条件 + 业务条件
    const enforcedConditions = this.buildEnforcedConditions(rawWhere.enforced);
    allConditions = [...enforcedConditions, ...(rawWhere.conditions || [])];
    //                                    ↑
    //                          只注入 environment_id!
  }

  return this.buildWhereClauseFromConditions(allConditions);
}
```

### 2.6 organization_id 来源：业务层手动添加

**示例 1: GetWorkflowRuns use case**  
**文件位置**: `apps/api/src/app/activity/usecases/get-workflow-runs/get-workflow-runs.usecase.ts:85-189`

```typescript
// 只传了 environmentId 给 QueryBuilder！
const queryBuilder = new QueryBuilder<WorkflowRun>({
  environmentId: command.environmentId,
});

// ⚠️ organization_id 不是强制注入的，需要手动添加！
// 实际代码中并没有调用 whereEquals('organization_id', command.organizationId)
// 这是一个潜在的安全/数据隔离问题！

// ...其他条件
if (command.workflowIds?.length) {
  queryBuilder.whereIn('workflow_id', command.workflowIds);
}
```

**示例 2: WorkflowRunRepository 自定义查询**  
**文件位置**: `libs/application-generic/src/services/analytic-logs/workflow-run/workflow-run.repository.ts:470-493`

```typescript
const query = `
  SELECT workflow_name, count(*) as count
  FROM workflow_runs FINAL
  WHERE 
    environment_id = {environmentId:String}   -- 手动添加
    AND organization_id = {organizationId:String}  -- 手动添加！不是强制注入！
    AND created_at >= {startDate:DateTime64(3)}
    AND created_at <= {endDate:DateTime64(3)}
    ${workflowFilter}
  GROUP BY workflow_name
  ORDER BY count DESC
  LIMIT 5
`;
```

> **重要发现**: 在自定义 SQL 查询中，organization_id 是手动作为参数绑定的，不是通过 QueryBuilder 强制注入的。

---

## 3. 查询构建完整流程

### 3.1 标准查询流程（QueryBuilder 路径）

```
1. 构造 QueryBuilder
   └─ 传入 { environmentId: 'env_xxx' }
      └─ 类型: EnforcedContext
      ↓
2. 业务层添加条件（可选）
   ├─ queryBuilder.whereIn('workflow_id', [...])
   ├─ queryBuilder.whereEquals('status', 'completed')
   └─ ⚠️ organization_id 需要在这里手动添加！不是自动的！
      ↓
3. 调用 build()
   └─ 返回 { enforced: { environmentId }, conditions: [...] }
      ↓
4. repository.find({ where })
   └─ buildWhereClause(where)
      ├─ buildEnforcedConditions(enforced)
      │   └─ 生成: environment_id = 'env_xxx'
      └─ 合并: enforcedConditions + businessConditions
         └─ SQL: WHERE environment_id = {env} AND ...
```

### 3.2 自定义 SQL 查询流程（Repository 专用方法）

```
1. 构建参数对象
   ├─ environmentId: string (必填)
   ├─ organizationId: string (必填, 但不是强制注入的!)
   ├─ startDate: Date
   └─ endDate: Date
      ↓
2. 模板字符串中手动写入两个 ID 的过滤条件
   ├─ AND environment_id = {environmentId:String}
   └─ AND organization_id = {organizationId:String}
      ↓
3. 调用 clickhouseService.query({ query, params })
```

---

## 4. 关键代码引用与证据

### 4.1 ReportTypeEnum 定义证据

| 序号 | 枚举值 | 文件 | 行号 |
|-----|-------|------|-----|
| 1 | `DELIVERY_TREND` | shared.dto.ts | 107 |
| 2 | `INTERACTION_TREND` | shared.dto.ts | 108 |
| 3 | `WORKFLOW_BY_VOLUME` | shared.dto.ts | 109 |
| 4 | `PROVIDER_BY_VOLUME` | shared.dto.ts | 110 |
| 5 | `MESSAGES_DELIVERED` | shared.dto.ts | 111 |
| 6 | `ACTIVE_SUBSCRIBERS` | shared.dto.ts | 112 |
| 7 | `AVG_MESSAGES_PER_SUBSCRIBER` | shared.dto.ts | 113 |
| 8 | `WORKFLOW_RUNS_METRIC` | shared.dto.ts | 114 |
| 9 | `TOTAL_INTERACTIONS` | shared.dto.ts | 115 |
| 10 | `WORKFLOW_RUNS_TREND` | shared.dto.ts | 116 |
| 11 | `ACTIVE_SUBSCRIBERS_TREND` | shared.dto.ts | 117 |
| 12 | `WORKFLOW_RUNS_COUNT` | shared.dto.ts | 118 |

### 4.2 EnforcedContext 证据

| 证据点 | 文件 | 行号 | 代码 |
|-------|------|-----|------|
| EnforcedContext 定义 | log.repository.ts | 54-56 | `{ environmentId: string }` |
| QueryBuilder 构造函数 | log.repository.ts | 644 | `constructor(private enforced: EnforcedContext)` |
| build() 返回结构 | log.repository.ts | 872-877 | `{ enforced: this.enforced, conditions: ... }` |
| buildEnforcedConditions 实现 | log.repository.ts | 203-213 | 只添加 `environment_id` 条件 |

### 4.3 organization_id 非强制注入证据

| 用例文件 | 行号 | 证据 |
|---------|-----|------|
| `get-workflow-runs.usecase.ts` | 86 | `new QueryBuilder<WorkflowRun>({ environmentId: command.environmentId })` **没有 organizationId** |
| `get-requests.usecase.ts` | 16 | `new QueryBuilder<RequestLog>({ environmentId: command.environmentId })` **没有 organizationId** |
| `workflow-run.repository.ts` | 472-473 | 自定义 SQL 中手动写了 `AND organization_id = {organizationId:String}` |

### 4.4 GetCharts 命令结构

**文件位置**: `apps/api/src/app/activity/usecases/get-charts/get-charts.command.ts:1-5`

```typescript
import { EnvironmentCommand } from '@novu/application-generic';
// EnvironmentCommand 包含:
// - environmentId: string
// - organizationId: string
// - userId: string
```

> **观察**: 虽然 Command 对象继承了 EnvironmentCommand 包含 organizationId，但 organizationId 只用于数据保留期验证，不直接传给 QueryBuilder。

---

## 附录：关键发现总结

1. **ReportTypeEnum 共 12 个值**，但 Dashboard 端只配置了 11 个，`WORKFLOW_RUNS_COUNT` 未在前端使用

2. **Environment ID 强制注入**：通过 QueryBuilder 构造函数 → EnforcedContext → buildEnforcedConditions 自动添加到 WHERE 子句

3. **Organization ID 不是强制注入的**：
   - EnforcedContext 类型不包含 organizationId
   - 需要在业务层手动通过 whereEquals 添加或在自定义 SQL 中手动写条件
   - GetWorkflowRuns、GetRequests 等用例中没有添加 organization_id 过滤条件！

4. **数据隔离风险**：由于 organization_id 不是强制注入，依赖于开发者手动添加条件，存在跨租户数据泄漏风险

---

**分析完成日期**: 2026-05-14  
**分析版本**: v1.0
