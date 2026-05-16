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

---

#### 🔍 验证步骤 1: 检查代码中 organizationId 过滤缺失

| 标准项 | 详细定义 |
|-------|---------|
| **验证目的** | 静态代码检查确认 MongoDB 查询层面是否缺少组织隔离条件 |
| **执行路径** | 代码审计，无需运行服务 |
| **✅ 通过阈值** | 同时满足两个条件：<br>1. `_environmentId: environmentId` 条件明确存在<br>2. `_organizationId: organizationId` 条件明确存在 |
| **❌ 失败判定** | 任一条件成立即失败：<br>1. `_organizationId` 查询条件缺失<br>2. `_organizationId` 参数未从方法签名传入<br>3. `_organizationId` 值硬编码为固定值 |
| **⚠️ 异常分支处理** | <br>• 方法被多处调用 → 追踪所有调用点，确认每个调用链都正确传参<br>• 方法被其他模块复用 → 检查复用场景的权限边界<br>• 代码使用 Query Builder 动态构建 → 检查所有构建分支<br>• 代码存在条件分支构建查询 → 检查每个分支都包含组织过滤 |
| **🔄 回滚准则** | <br>• 发现缺少组织过滤 → 立即暂停 Feature Flag 灰度<br>• 无法快速修复 → 回滚到明细表查询路径（安全但性能下降）<br>• 修复后必须通过所有后续验证步骤才可重新开启灰度 |

**执行命令**:

```bash
# 1. 定位到目标文件
cd libs/dal/src/repositories/notification-template

# 2. 查看 findByTriggerIdentifierBulk 方法实现
cat notification-template.repository.ts | grep -A 30 "findByTriggerIdentifierBulk"

# 3. 查找所有调用该方法的地方
grep -r "findByTriggerIdentifierBulk" --include="*.ts" .
```

**验证输出检查清单**:
```typescript
// ✅ PASS: 双隔离条件都存在
const requestQuery: NotificationTemplateQuery = {
  _environmentId: environmentId,    // ✅ 环境隔离 - 存在
  _organizationId: organizationId,  // ✅ 组织隔离 - 存在
  'triggers.identifier': { $in: identifiers },
};

// ❌ FAIL: 缺少组织隔离
const requestQuery: NotificationTemplateQuery = {
  _environmentId: environmentId,    // ✅ 环境隔离 - 存在
  // ⚠️ 组织隔离 - 缺失（失败）
  'triggers.identifier': { $in: identifiers },
};
```

---

#### 🔍 验证步骤 2: 数据库中 identifier 冲突检测

| 标准项 | 详细定义 |
|-------|---------|
| **验证目的** | 检测生产环境中是否已经存在跨组织的 trigger_identifier 冲突 |
| **执行路径** | MongoDB 聚合查询，只读操作 |
| **✅ 通过阈值** | 两个层级的通过标准：<br>1. 聚合查询返回空结果 → 无冲突，完全安全<br>2. 查询返回冲突但冲突组织 ID 完全相同 → 组织内不同工作流重名，无越权风险 |
| **❌ 失败判定** | 任一条件成立即失败：<br>1. 查询结果中 `organization_ids` 数组长度 ≥ 2<br>2. 同一 `trigger_identifier` 对应不同组织的工作流<br>3. 冲突数量 ≥ 3 条 → 系统性命名冲突问题 |
| **⚠️ 异常分支处理** | <br>• 发现少量冲突（1-2条） → 记录冲突组织，沟通重命名后继续灰度<br>• 发现大量冲突（≥5条） → 暂停灰度，先统一清理命名<br>• 冲突涉及重要客户 → 优先处理该客户的命名调整<br>• 空字符串或 null identifier → 过滤掉不纳入统计 |
| **🔄 回滚准则** | <br>• 冲突数量 ≥ 3 条 → 暂停 Feature Flag 开启计划<br>• 存在可验证的实际越权场景 → 立即关闭已开启的 Flag<br>• 冲突清理完成后重新执行本步骤验证 |

**Mongo Shell 执行脚本**（含统计输出）:

```javascript
use novu;

// Step 1: 检测跨组织冲突
const conflictResults = db.notification_templates.aggregate([
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
    org_count: { $size: "$organization_ids" },
    organization_ids: 1,
    template_names: 1,
    _id: 0
  }}
]).toArray();

// Step 2: 输出结果判定
print("=== 跨组织 Identifier 冲突检测报告 ===");
print("总冲突数:", conflictResults.length);

if (conflictResults.length === 0) {
  print("✅ PASS: 无跨组织 identifier 冲突");
} else {
  print("⚠️ FAIL: 发现", conflictResults.length, "处跨组织冲突");
  conflictResults.forEach((c, i) => {
    print(`\n冲突 #${i+1}:`);
    print(`  Trigger: ${c.trigger_identifier}`);
    print(`  环境: ${c.environment_id}`);
    print(`  涉及组织数: ${c.org_count}`);
    print(`  组织 IDs: ${c.organization_ids.join(', ')}`);
    print(`  工作流名称: ${c.template_names.join(', ')}`);
  });
}

// Step 3: 判定结果
const isPass = conflictResults.length === 0;
print("\n=== 最终判定: " + (isPass ? "✅ PASS" : "❌ FAIL") + " ===");
```

---

#### 🔍 验证步骤 3: 构造冲突场景功能测试

| 标准项 | 详细定义 |
|-------|---------|
| **验证目的** | 人工构造越权访问场景，验证安全边界是否生效 |
| **执行路径** | 功能测试，需要测试环境账号 |
| **✅ 通过阈值** | 三个层级全部通过：<br>1. OrgA API 响应仅包含 OrgA 的工作流名称<br>2. OrgB API 响应仅包含 OrgB 的工作流名称<br>3. 即使 identifier 完全相同，名称也按组织正确隔离 |
| **❌ 失败判定** | 任一条件成立即失败：<br>1. OrgA 响应中出现 OrgB 的工作流名称<br>2. OrgB 响应中出现 OrgA 的工作流名称<br>3. 响应中出现 null / undefined / 空名称 |
| **⚠️ 异常分支处理** | <br>• 创建工作流失败 → 排查 API Key 和环境权限<br>• 相同 identifier 也能成功创建 → 说明无唯一性校验，确认风险存在<br>• 测试环境无数据 → 先发送测试消息生成统计数据<br>• 数量太少 TOP 5 不显示 → 多发消息让工作流进入排名 |
| **🔄 回滚准则** | <br>• 测试失败 → 立即停止所有灰度并关闭已开启的 Flag<br>• 测试通过但生产有冲突数据 → 先清理数据再开启灰度<br>• 修复后必须重新执行本步骤验证 |

**标准测试流程**:

```bash
# =========================================
# 阶段 1: 测试环境准备
# =========================================

# 1.1 创建 OrgA 工作流 (Trigger ID: security-test-trigger)
curl -X POST "https://api.novu.co/v1/workflows" \
  -H "Authorization: ApiKey <ORG_A_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"name":"OrgA-Test-Workflow","triggerIdentifier":"security-test-trigger",...}'

# 1.2 创建 OrgB 同名 trigger 工作流
curl -X POST "https://api.novu.co/v1/workflows" \
  -H "Authorization: ApiKey <ORG_B_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"name":"OrgB-Test-Workflow","triggerIdentifier":"security-test-trigger",...}'

# 1.3 两边分别发送测试消息（让数据进入统计）
for i in {1..10}; do
  curl -X POST "https://api.novu.co/v1/events/trigger" \
    -H "Authorization: ApiKey <ORG_A_API_KEY>" \
    -d '{"name":"security-test-trigger","to":{"subscriberId":"test-user"}}'
done

# =========================================
# 阶段 2: 执行越权测试
# =========================================

# 2.1 使用 OrgA 凭证查询图表
curl -s "https://api.novu.co/v1/activity/charts?reportType[]=workflow-by-volume" \
  -H "Authorization: ApiKey <ORG_A_API_KEY>" | jq '.data' > orgA-response.json

# 2.2 验证 OrgA 响应中不应出现 OrgB 的工作流名称
if grep -q "OrgB-Test-Workflow" orgA-response.json; then
  echo "❌ FAIL: OrgA 响应中发现 OrgB 的工作流名称 - 越权泄露"
  exit 1
else
  echo "✅ PASS: OrgA 响应中仅包含 OrgA 工作流"
fi

# 2.3 反向测试：OrgB 凭证查询
curl -s "https://api.novu.co/v1/activity/charts?reportType[]=workflow-by-volume" \
  -H "Authorization: ApiKey <ORG_B_API_KEY>" | jq '.data' > orgB-response.json

if grep -q "OrgA-Test-Workflow" orgB-response.json; then
  echo "❌ FAIL: OrgB 响应中发现 OrgA 的工作流名称 - 越权泄露"
  exit 1
else
  echo "✅ PASS: OrgB 响应中仅包含 OrgB 工作流"
fi
```

---

#### 🔍 验证步骤 4: 对比明细/聚合双路径查询结果

| 标准项 | 详细定义 |
|-------|---------|
| **验证目的** | 验证双查询路径的数据一致性，确保聚合表路径没有越权数据 |
| **执行路径** | A/B 对比测试，需要 Feature Flag 控制权限 |
| **✅ 通过阈值** | 数据完全一致：<br>1. 工作流名称集合完全相等（子集相等 → 实际相等）<br>2. 每个工作流的数量误差 ≤ ±1（允许 Replication 延迟）<br>3. TOP 5 排名顺序一致（或差异可解释） |
| **❌ 失败判定** | 任一条件成立即失败：<br>1. 聚合表路径包含明细表路径没有的工作流名称<br>2. 名称完全不匹配（映射逻辑出错）<br>3. 数量误差 > 5 条（排除时间窗口差异） |
| **⚠️ 异常分支处理** | <br>• 聚合表数据比明细表少 → 正常：聚合表只统计最近数据<br>• 数量有轻微差异（2-5条） → 归因于物化视图延迟，等待 5 分钟重试<br>• 名称有微小差异（大小写、空格） → 规范化后比较<br>• 工作流在 Flag 切换期间刚创建 → 等待聚合表同步 |
| **🔄 回滚准则** | <br>• 发现额外工作流名称 → 立即关闭 Flag<br>• 名称完全不匹配 → 检查 mapping 逻辑，暂停灰度<br>• 差异无法解释 → 回滚到明细表路径，深入调查 |

**标准化验证脚本**:

```typescript
// =========================================
// 双路径对比测试脚本
// =========================================

interface WorkflowVolume {
  workflowName: string;
  count: number;
}

function normalizeName(name: string): string {
  return name.toLowerCase().replace(/\s+/g, ' ').trim();
}

function compareResults(detailResults: WorkflowVolume[], aggregatedResults: WorkflowVolume[]): {
  pass: boolean;
  issues: string[];
} {
  const issues: string[] = [];
  
  // 1. 名称集合对比
  const detailNames = new Set(detailResults.map(r => normalizeName(r.workflowName)));
  const aggregatedNames = new Set(aggregatedResults.map(r => normalizeName(r.workflowName)));
  
  // 检查聚合表是否有额外名称（越权泄露）
  for (const name of aggregatedNames) {
    if (!detailNames.has(name)) {
      issues.push(`越权风险: 聚合表包含明细表没有的工作流名称 '${name}'`);
    }
  }
  
  // 2. 数量对比（允许误差 ±1）
  const detailMap = new Map(detailResults.map(r => [normalizeName(r.workflowName), r.count]));
  for (const aggregated of aggregatedResults) {
    const normName = normalizeName(aggregated.workflowName);
    const detailCount = detailMap.get(normName);
    
    if (detailCount !== undefined && Math.abs(aggregated.count - detailCount) > 1) {
      issues.push(`数量差异: '${aggregated.workflowName}' 明细=${detailCount}, 聚合=${aggregated.count}`);
    }
  }
  
  return {
    pass: issues.length === 0,
    issues
  };
}

// 执行测试流程:
// 1. 关闭 Flag → 调用 API → 保存 detailResults
// 2. 开启 Flag → 调用同一 API → 保存 aggregatedResults
// 3. compareResults(detailResults, aggregatedResults)
// 4. pass=true → 继续下一验证步骤
//    pass=false → 记录 issues，执行回滚
```

---

#### 🔍 验证步骤 5: 代码修复验证

| 标准项 | 详细定义 |
|-------|---------|
| **验证目的** | 确认修复方案正确解决了安全问题，无回归风险 |
| **执行路径** | 修复后全量回归测试 |
| **✅ 通过阈值** | 修复后所有前置步骤全部通过：<br>1. 步骤 1 通过：代码中双隔离条件均存在<br>2. 步骤 2 通过：（如有）冲突已清理或不影响<br>3. 步骤 3 通过：构造场景测试无越权泄露<br>4. 步骤 4 通过：双路径数据一致<br>5. 所有调用点均已更新，无遗漏 |
| **❌ 失败判定** | 任一条件成立即失败：<br>1. 任何前置步骤失败<br>2. 存在遗漏的调用点未修复<br>3. TypeScript 类型错误未解决<br>4. 编译 / 单元测试失败 |
| **⚠️ 异常分支处理** | <br>• 第三方模块也调用了该方法 → 检查第三方是否需要传组织参数<br>• 调用点分散在多个仓库 → 逐个仓库检查，确保同步发布<br>• 修复引入 Breaking Change → 保持向后兼容：organizationId 可选参数，内部校验必须传入<br>• 有测试覆盖该方法 → 更新测试用例包含组织参数断言 |
| **🔄 回滚准则** | <br>• 修复验证失败 → 回滚修复代码，评估影响范围<br>• 修复引入其他 Bug → 回滚到未修改版本，重新设计修复方案<br>• 无法在发布窗口内完成 → 推迟灰度计划 |

**修复验证清单**:

```markdown
✅ 修复代码验证清单
-----------------
[ ] 1. Repository 方法签名已新增 organizationId 参数
[ ] 2. MongoDB 查询条件包含 _organizationId 过滤
[ ] 3. 所有调用点均传入了正确的 organizationId（从 Session 取值）
[ ] 4. TypeScript 类型检查通过：tsc --noEmit
[ ] 5. 相关单元测试通过
[ ] 6. 步骤 2 数据库冲突检测重新执行通过
[ ] 7. 步骤 3 构造场景功能测试重新执行通过
[ ] 8. 步骤 4 双路径对比重新执行通过
[ ] 9. 接口文档同步更新（如有）
[ ] 10. CHANGELOG 记录安全修复说明
```

---

#### 🔍 验证步骤 6: ClickHouse 聚合表组织隔离复核

| 标准项 | 详细定义 |
|-------|---------|
| **验证目的** | 确认 ClickHouse 层查询严格按组织隔离，防止跨组织数据泄露 |
| **执行路径** | 代码审计 + 实际 SQL 日志核查 |
| **✅ 通过阈值** | 三层防护均满足：<br>1. SQL 字符串中明确包含 `organization_id = {organizationId:String}` 条件<br>2. 参数绑定使用预编译语句，非字符串拼接<br>3. organizationId 值来源为 User Session，非用户可控制输入 |
| **❌ 失败判定** | 任一条件成立即失败：<br>1. WHERE 子句缺少 organization_id 条件<br>2. organization_id 条件使用 OR 连接（可能被绕过）<br>3. 参数值可被用户通过 API 参数控制<br>4. 使用字符串拼接构造 SQL（注入风险） |
| **⚠️ 异常分支处理** | <br>• 使用 Query Builder 动态构建 → 检查所有分支都包含 AND 条件<br>• 有多个 environment_id / organization_id 条件 → 确认逻辑正确<br>• SQL 注释中提到 organization_id → 确认实际执行也包含<br>• 多表 JOIN 查询 → 确认每个表都有正确的组织过滤条件 |
| **🔄 回滚准则** | <br>• ClickHouse 层缺少组织过滤 → 极高风险，立即停止所有灰度<br>• 无法快速修复 → 回滚到明细表查询路径<br>• 修复后执行安全渗透测试确认 |

**SQL 审计检查清单**:

```sql
-- ✅ PASS: 正确的双隔离查询
SELECT workflow_run_id, sum(count) as count
FROM workflow_run_count
WHERE
  environment_id = {environmentId:String}   -- ✅ 预编译参数绑定
  AND organization_id = {organizationId:String}  -- ✅ AND 连接，不可绕过
  AND date >= {startDate:Date}
  AND date <= {endDate:Date}
GROUP BY workflow_run_id
ORDER BY count DESC
LIMIT 5;

-- ⚠️ FAIL: 缺少组织过滤
SELECT workflow_run_id, sum(count) as count
FROM workflow_run_count
WHERE
  environment_id = {environmentId:String}
  -- ❌ organization_id 条件缺失
  AND date >= {startDate:Date}
GROUP BY workflow_run_id;

-- ⚠️ FAIL: OR 连接可能被绕过
WHERE
  environment_id = {environmentId:String}
  OR organization_id = {organizationId:String}  -- ❌ 使用 OR，可被绕过
```

---

### 5.6.1 验证结果判定矩阵

| 步骤 | 失败 → 风险等级 | 行动 |
|-----|---------------|-----|
| Step 1 (代码检查) | 🔴 高 | 立即暂停灰度计划，修复后重新开始 |
| Step 2 (数据库冲突) | 🟡 中 | 少量冲突 → 沟通清理后继续；大量冲突 → 暂停 |
| Step 3 (功能测试) | 🔴 高 | 立即关闭所有已开启的 Flag，修复后验证 |
| Step 4 (双路径对比) | 🟠 中高 | 发现额外名称 → 关闭 Flag；数量差异 → 调查原因 |
| Step 5 (修复验证) | 🟡 中 | 修复验证失败 → 回滚修复代码 |
| Step 6 (ClickHouse 审计) | 🔴 高 | 立即停止所有灰度，安全团队介入调查 |

---

### 5.6.2 全量验证通过标准

**只有同时满足以下所有条件，才可判定验证通过，允许继续 Feature Flag 灰度**:

```markdown
✅ 全部验证通过标准
------------------
[ ] Step 1: 代码静态检查 - 双隔离条件均存在
[ ] Step 2: 数据库冲突检测 - 无跨组织冲突 或 冲突已清理
[ ] Step 3: 构造场景功能测试 - 无越权泄露
[ ] Step 4: 双路径数据对比 - 结果完全一致
[ ] Step 5: 修复验证 - 所有回归测试通过
[ ] Step 6: ClickHouse 审计 - 组织隔离正确

满足所有条件 → ✅ 验证通过，可继续灰度
任一条件不满足 → ❌ 验证不通过，执行对应回滚措施
```

---

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
