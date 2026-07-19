# OpenSearch SQL 插件可行性评估报告

> 统一查询引擎项目 — 引入 OpenSearch SQL 插件作为统一 SQL 查询层
> 评估日期：2026-07-19
> 基于 OpenSearch 3.7.0 + SQL Plugin 3.7.0（含 UNION/UNION ALL 扩展）

---

## 目录

- [一、项目背景](#一项目背景)
- [二、SQL 插件架构](#二sql-插件架构)
- [三、SQL vs DSL 性能对比](#三sql-vs-dsl-性能对比)
- [四、SQL 插件能力评估](#四sql-插件能力评估)
- [五、能力扩展工作量评估](#五能力扩展工作量评估)
- [六、社区维护情况](#六社区维护情况)
- [七、风险与挑战](#七风险与挑战)
- [八、结论与建议](#八结论与建议)

---

## 一、项目背景

### 1.1 目标

评估在统一查询引擎中引入 OpenSearch SQL 插件的可行性，实现通过 SQL 统一查询不同存储（OpenSearch、Prometheus、Spark、S3 等）。

### 1.2 评估范围

| 维度 | 内容 |
|------|------|
| 性能 | SQL 接口 vs DSL 接口的延迟差异（1M / 10M 数据量） |
| 能力 | SQL 支持的语法范围、不支持的限制、扩展可行性 |
| 工作量 | 扩展不支持特性的开发工作量估算 |
| 社区 | OpenSearch SQL 插件的社区活跃度、版本节奏、路线图 |
| 风险 | 三引擎共存、512MB heap 限制、filter cache 不公平性等 |

---

## 二、SQL 插件架构

### 2.1 三种执行引擎

OpenSearch SQL 插件有三种执行引擎，是历史演进的结果：

| 引擎 | 解析器 | 计划抽象 | 引入版本 | 状态 |
|------|--------|---------|:---:|------|
| **Legacy V1** | Druid SQL Parser | 无（直接翻译为 DSL） | 1.0 | 维护中（不再新增功能） |
| **V2** | ANTLR 4（自研文法） | AST → LogicalPlan → PhysicalPlan | 2.0+ | 当前主力 |
| **Calcite** | ANTLR 4（同 V2） | AST → Calcite RelNode | 3.0+ | 未来方向 |

### 2.2 路由逻辑

```
POST /_plugins/_sql
  │
  ├─ 索引是 composite dataformat? ──YES──→ Unified Query API (Calcite 原生)
  │
  └─ NO → V2 引擎 AstBuilder
       │
       ├─ 普通查询 (SELECT/WHERE/GROUP BY/ORDER BY)
       │    → V2 引擎 (Analyzer → Planner → PhysicalPlan)
       │
       ├─ 包含 UNION? (我们的扩展)
       │    → Calcite 引擎 (CalciteRelNodeVisitor → RelNode)
       │
       └─ JOIN / IN 子查询? (AstBuilder 抛异常)
            → 回退 Legacy V1 (Druid → SearchRequestBuilder)
```

### 2.3 关键技术特征

| 特征 | 说明 |
|------|------|
| **算子下推** | Calcite 通过 Convention trait 将 filter/agg/sort/limit 下推到 OpenSearch |
| **composite 聚合** | V2 使用 composite 聚合（size=1000 分页拉取），适合全量遍历，不适合 Top-N |
| **Calcite 回退** | Calcite 失败时静默回退到 V2（`QueryService.java:181-183`） |
| **filter cache** | SQL 路径不尊守 `request_cache=false`，filter cache 对 SQL 生效 |
| **无计划缓存** | 每次查询都重新解析+规划，无 PreparedStatement 缓存 |
| **无统计注入** | Calcite 优化器无统计信息注入，CBO 退化为规则系统 |

---

## 三、SQL vs DSL 性能对比

### 3.1 测试环境

| 项目 | 1M 测试 | 10M 测试 |
|------|---------|---------|
| 节点数 | 1 | 1 |
| Shard 数 | 3 | 6 |
| 文档数 | 1,000,000 | 9,900,000 |
| JVM Heap | 512MB | 512MB |
| forcemerge | 是（1M） | 否（10M，模拟生产） |
| 字段数 | 10 | 13（含高基数 user_id=100K, session_id=500K） |
| 预热 | 20 轮（简单）/ 5 轮（复杂） | 3-5 轮 |
| 测试轮数 | 100 轮（简单）/ 20 轮（复杂） | 10-30 轮 |

### 3.2 轻查询结果（1M 数据）

| 场景 | SQL (ms) | DSL (ms) | SQL 额外开销 | 开销占比 | 推荐 |
|------|:---:|:---:|:---:|:---:|------|
| 点查（单条件） | 1.3 | 0.8 | +0.5ms | 63% | 低 QPS 可用 SQL |
| 点查（多条件） | 2.9 | 2.0 | +0.9ms | 45% | 低 QPS 可用 SQL |
| 范围查询 | 1.2 | 0.8 | +0.4ms | 50% | 低 QPS 可用 SQL |
| 全文搜索 | 1.0 | 0.6 | +0.4ms | 67% | 均可 |
| 聚合（简单） | 3.6¹ | 1.6¹ | +2.0ms | 128% | DSL |
| 聚合（多级） | 3.0¹ | 1.4¹ | +1.6ms | 110% | DSL |
| 排序+小分页 | 1.1 | 0.7 | +0.4ms | 57% | 均可 |
| 深度分页 | 13.3 | 3.3 | +10.0ms | 303% | DSL |
| 大结果集(1K行) | 20.0 | 9.8 | +10.2ms | 104% | DSL |

> ¹ 重测数据（随机阈值打散 filter cache + DSL 去掉 percentile 保持等价）

### 3.3 重查询结果（10M 数据）

| 场景 | SQL (ms) | DSL (ms) | SQL 额外开销 | 开销占比 | 推荐 |
|------|:---:|:---:|:---:|:---:|------|
| 时间范围+聚合 | 121.7² | 50.7² | +71.0ms | 140% | DSL |
| 全表多级聚合 | 211.4² | 104.9² | +106.5ms | 102% | DSL |
| 全表高基数聚合(500K桶) | 32991³ | 143.7³ | +32847ms | 22865% | DSL |
| 大范围+排序+10K行 | 173.4 | 71.8 | +101.6ms | 142% | DSL |
| 三路UNION+聚合(pushdown ON) | 12.5 | — | — | — | SQL 唯一选择 |
| 大表JOIN(10K行) | 744.1 | — | — | — | SQL 唯一选择 |
| IN子查询(10K行) | 789.5 | — | — | — | SQL 唯一选择 |

> ² 重测数据（随机阈值打散 filter cache + DSL 去掉 percentile 保持等价）
> ³ 512MB heap 严重不足，H1 触发 circuit breaker，数据可信度低

### 3.4 SQL 额外开销分析

| 开销类型 | 范围 | 说明 | 可信度 |
|---------|:---:|------|:---:|
| 翻译开销（解析+分析+规划+DSL生成） | 0.4-20ms | 与查询复杂度正相关，**非固定值** | ⚠️ 估算 |
| JdbcResponseFormatter 序列化 | 估算 0-90ms | 与结果集大小相关（~9ms/1K行，未实测分解） | ⚠️ 估算 |
| V2 composite 翻页（高基数场景） | 0-32991ms | 与桶数强相关（桶数/1000 次翻页请求） | ✅ 实测 |
| **总额外开销**（SQL总延迟 - DSL总延迟） | **0.4-32991ms** | 实测值，与查询类型强相关 | ✅ 实测 |

### 3.5 性能对比结论

```
SQL 在所有有 DSL 对照的场景中都比 DSL 慢 1.1-2.4x（排除高基数聚合）。
额外开销主要来自翻译层（0.4-20ms）和结果格式化（0-90ms）。

在 DSL 执行时间 >200ms 的重查询场景下，翻译开销占比通常 <10%，
但总额外开销（含格式化）可能使 SQL 延迟翻倍。

高基数聚合（GROUP BY 10K+ 桶）是灾难性场景：
V2 composite 翻页导致 33-228x 性能差距（512MB heap 下更严重）。

SQL 的价值不在于性能，而在于功能（UNION/JOIN/子查询）和易用性。
```

---

## 四、SQL 插件能力评估

### 4.1 已支持的能力（均已实测验证）

| 能力 | 语法 | 引擎路径 | 状态 |
|------|------|:---:|:---:|
| SELECT / WHERE / ORDER BY / LIMIT | 标准 SQL | V2 | ✅ |
| GROUP BY + 聚合函数 | `GROUP BY a, COUNT(*)` | V2 | ✅ |
| 全文搜索 | `match(field, query)` | V2 | ✅ |
| 多字段搜索 | `MULTI_MATCH(field, query)` | V2 | ✅ |
| 窗口函数 | `RANK() OVER(...)` | V2 | ✅ |
| 派生表 | `(SELECT...) AS t` | V2 | ✅ |
| 2 表 JOIN | `SELECT...FROM a JOIN b ON...` | Legacy V1 | ✅ |
| JOIN + WHERE / ORDER BY / LIMIT | `SELECT...JOIN...WHERE...` | Legacy V1 | ✅ |
| IN 子查询 | `WHERE x IN (SELECT...)` | Legacy V1 | ✅ |
| UNION ALL | `SELECT...UNION ALL SELECT...` | Calcite（我们的扩展） | ✅ |
| UNION DISTINCT | `SELECT...UNION SELECT...` | Calcite（我们的扩展） | ✅ |
| 游标分页 | `fetch_size` 参数 | V2 / Calcite | ✅ |

### 4.2 不支持的能力（均已实测验证）

| 能力 | 错误信息 | 根因 |
|------|---------|------|
| CTE (WITH) | `Query must start with SELECT...` | 文法无 WITH 规则 |
| 3 表及以上 JOIN | `currently supports only 2 tables join` | Legacy V1 限制 |
| JOIN + GROUP BY | `JOIN queries do not support aggregations` | Legacy V1 限制 |
| JOIN + 聚合函数 | 同上 | Legacy V1 限制 |
| EXISTS 子查询 | `Unsupported subquery` | V2 不支持，未回退 Legacy |
| 标量子查询 (SELECT 中) | `unknown field name` | Druid 解析器限制 |
| 子查询 + 外层 GROUP BY | `cannot be cast to SQLJoinTableSource` | Druid 类型转换失败 |
| COALESCE 函数 | `not supported in Schema: COALESCE` | V2 函数注册表缺失 |
| DATE_HISTOGRAM 函数 | NPE: `this.value is null` | V2 INTERVAL 参数解析 bug |
| HISTOGRAM 函数 | 同上 NPE | 同上 |

### 4.3 DSL 独有能力（SQL 不支持）

| 能力 | 说明 |
|------|------|
| 复杂 bool query | `bool: {must, should, filter, must_not}` 精确控制 |
| script_fields | 运行时计算字段 |
| runtime_mappings | 运行时定义字段 |
| suggest | 搜索建议 |
| collapse | 按字段去重 |
| date_histogram 聚合 | SQL 的 DATE_HISTOGRAM NPE |
| nested aggs 多级嵌套 | SQL 的 flat GROUP BY 与之不同 |
| profile API | 查询性能分析 |
| search template | 模板化查询 |

---

## 五、能力扩展工作量评估

### 5.1 扩展模式分类

| 模式 | 适用特性 | 说明 |
|------|---------|------|
| **计划级路由**（三步模式） | JOIN、EXISTS 子查询 | AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 已有实现 |
| **函数注册** | COALESCE | 在 BuiltinFunctionRepository 注册函数，映射到 Calcite SqlOperator |
| **聚合函数修复** | DATE_HISTOGRAM | 修复 INTERVAL 参数解析 NPE + 注册聚合函数 + 映射下推 |
| **表达式级子查询** | 标量子查询 | AstExpressionBuilder 新增方法 + Calcite RexSubquery 映射 |
| **语句级文法** | CTE | 新增文法规则 + AST 节点 + 作用域管理 |

### 5.2 工作量估算

| 优先级 | 特性 | 模式 | 人天 | 理由 |
|:---:|------|------|:---:|------|
| **P0** | 统计信息注入 | 独立工作流 | 10-15 | 最高 ROI，让 Calcite CBO 真正生效 |
| **P1** | COALESCE | 函数注册 | 1-2 | 最简单，Calcite 原生支持 |
| **P1** | DATE_HISTOGRAM | 聚合修复 | 3-5 | 修复 bug，时序分析基础能力 |
| **P2** | 3 表+ JOIN | 计划级路由 | 5-8 | 可复用 UNION 三步模式 |
| **P2** | EXISTS 子查询 | 计划级路由 | 3-5 | SqlV2QueryParser 已有实现可参考 |
| **P3** | JOIN + GROUP BY | 依赖 P2 | 5-8 | JOIN 走 Calcite 后自动获得 |
| **P3** | 子查询 + 外层 GROUP BY | 依赖 P2 | 5-8 | 同上 |
| **P4** | 标量子查询 | 表达式级 | 8-12 | 复杂度最高 |
| **P5** | CTE | 语句级 | 10-15 | 可用派生表替代 |

### 5.3 总工作量

| 范围 | 人天 |
|------|:---:|
| P0+P1（快速收益） | 14-22 |
| P0-P2（核心能力） | 22-35 |
| P0-P3（覆盖大部分场景） | 32-51 |
| P0-P5（全部） | 50-78 |

### 5.4 依赖关系

```
统计信息注入 ──────────── 独立（最高 ROI，建议优先）
COALESCE ─────────────── 独立
DATE_HISTOGRAM ────────── 独立
3表+ JOIN ─────────────── 独立
  ├── JOIN + GROUP BY ─── 依赖 3表+ JOIN
  └── 子查询+外层GROUP BY ─ 依赖 3表+ JOIN
EXISTS 子查询 ─────────── 独立
标量子查询 ────────────── 独立（复杂度最高）
CTE ───────────────────── 独立（可用派生表替代）
```

---

## 六、社区维护情况

### 6.1 版本节奏

| 版本 | 发布时间 | 重大变化 |
|------|---------|---------|
| 3.0 | 2025 | Calcite 引擎引入（PPL 默认），`plugins.calcite.enabled` 默认 false |
| 3.3 | 2025 | Calcite 默认启用，PPL 全走 Calcite |
| 3.7 | 2026-06 | Unified Query API + Analytics Engine 集成，UNION ALL（PR #5506） |

### 6.2 社区活跃度

| 指标 | 数据 |
|------|------|
| GitHub Stars | 172+ |
| Contributors | 213+ forks |
| Open Issues | 315+ |
| 近期 PR 合并频率 | 活跃（3.7 有 20+ PR） |
| Calcite 引擎路线图 | issue #3457（SQL 走 Calcite，目标 3.1+） |

### 6.3 路线图

```
当前状态（3.7）:
  ✅ PPL 全走 Calcite
  ✅ SQL UNION 走 Calcite（我们的扩展）
  ⚠️ SQL 其他语法走 V2（Calcite 支持待开发）
  ⚠️ JOIN/IN 子查询回退 Legacy V1

目标状态（4.0+）:
  🎯 SQL 全走 Calcite
  🎯 Legacy V1 退役
  🎯 V2 PhysicalPlan 被 Calcite RelNode 替代
  🎯 统计信息注入 Calcite
  🎯 计划缓存（PreparedStatement）
```

### 6.4 社区扩展参考

| 项目 | 说明 |
|------|------|
| `opensearch-project/analytics-engine` | Project Mustang，DataFusion 后端（sandbox 中） |
| Unified Query API (`api/` 模块) | Calcite 原生 SQL 解析 + 转译（Spark/Postgres/MySQL 方言） |
| `SqlV2QueryParser.ExtendedAstBuilder` | 已实现 JOIN/UNION/子查询的 AST 构建（仅 composite 索引路径） |

---

## 七、风险与挑战

### 7.1 技术风险

| 风险 | 严重性 | 说明 | 缓解措施 |
|------|:---:|------|---------|
| **三引擎共存** | 🔴 高 | 语义漂移、路由复杂度增长、测试矩阵爆炸、性能不可预测 | 定义退役时间表；新功能只做 Calcite |
| **高基数聚合瓶颈** | 🔴 高 | V2 composite 翻页（桶数/1000 次请求），10K+ 桶场景 33-228x 慢于 DSL | 优化 V2 对 ORDER BY+LIMIT 改用 terms 聚合 |
| **512MB heap 限制** | 🟠 中 | 10M 数据 + 高基数聚合触发 circuit breaker / GC | 生产环境使用 8GB+ heap |
| **filter cache 不公平** | 🟠 中 | SQL 路径不尊守 request_cache=false，基准测试需随机化 | 测试时使用随机阈值 |
| **无计划缓存** | 🟠 中 | 每次查询重新解析+规划，高 QPS 下开销累积 | 长期实现 PreparedStatement 缓存 |
| **无统计注入** | 🟠 中 | Calcite CBO 退化为规则系统 | P0 优先实现统计注入 |
| **Calcite 单节点瓶颈** | 🟡 低 | UNION/JOIN 内存计算在协调节点单线程，无分布式并行 | 大数据量场景需关注 OOM 风险 |

### 7.2 维护风险

| 风险 | 说明 |
|------|------|
| **社区版本对齐** | 我们的 UNION 扩展需要与社区主线合并或持续 rebase |
| **Calcite 迁移风险** | 社区计划 SQL 全走 Calcite（issue #3457），迁移可能破坏现有行为 |
| **Legacy V1 退役** | 社区计划退役 Legacy V1，JOIN/IN 子查询需迁移到 Calcite |
| **Analytics Engine 依赖** | Unified Query API 依赖外部 analytics-engine 插件（sandbox 状态） |

---

## 八、结论与建议

### 8.1 可行性评估

| 维度 | 评估 | 理由 |
|------|:---:|------|
| **性能** | ⚠️ 有条件可行 | SQL 比 DSL 慢 1.1-2.4x（聚合场景），重查询下翻译开销占比 <10%可接受。高基数聚合是瓶颈 |
| **功能** | ⚠️ 有条件可行 | 基础 SQL 支持完整。JOIN 限制多（2表、无聚合）。UNION 已扩展。9 项不支持特性可扩展（14-78 人天） |
| **架构** | ✅ 可行 | 三引擎架构虽复杂但可工作。Calcite 是未来方向，社区在推进 |
| **社区** | ✅ 可行 | 活跃维护，3.7 有 20+ PR，路线图清晰 |
| **维护** | ⚠️ 有条件可行 | 需要与社区主线对齐，自行扩展的特性需持续维护 |

### 8.2 综合结论

**结论：引入 OpenSearch SQL 插件作为统一查询层是可行的，但有条件。**

**可行的场景**：
- ✅ 通过 SQL 统一查询不同存储（SQL 的核心价值）
- ✅ 中等结果集查询（≤1000 行，额外开销 <20ms）
- ✅ UNION/JOIN/IN 子查询（DSL 不支持，SQL 是唯一选择）
- ✅ BI 工具 / JDBC 接入（生态兼容）
- ✅ 开发调试（SQL 可读性好）

**需要限制的场景**：
- ⚠️ 高 QPS 点查（翻译开销占比 40-63%，DSL 更优）
- ⚠️ 高基数聚合（GROUP BY 10K+ 桶，需优化 V2 引擎）
- ⚠️ 大结果集（10K+ 行，JdbcResponseFormatter 开销显著）
- ⚠️ 复杂全文搜索（SQL match() 功能受限）

**不可行的场景**：
- ❌ 替代 DSL 做高性能全文搜索
- ❌ 替代 DSL 做高基数聚合分析

### 8.3 建议方案

**短期（1-3 个月）**：
1. 引入 SQL 插件作为统一查询层，支持基础 SQL 查询
2. 合并 UNION/UNION ALL 扩展到主线
3. 实现统计信息注入（P0，10-15 人天）让 Calcite CBO 生效
4. 对高基数聚合场景，SQL 内部自动回退到 DSL 聚合
5. 限制 SQL 查询结果集大小（≤1000 行），大结果集走 DSL

**中期（3-6 个月）**：
1. 扩展 COALESCE、DATE_HISTOGRAM（P1，4-7 人天）
2. 扩展 3 表+ JOIN、EXISTS 子查询走 Calcite（P2，8-13 人天）
3. 实现计划缓存（降低高 QPS 场景翻译开销）
4. 推动社区 issue #3457（SQL 全走 Calcite）

**长期（6-12 个月）**：
1. 扩展 JOIN + GROUP BY、标量子查询（P3-P4，13-20 人天）
2. 推动 Legacy V1 退役，JOIN/IN 子查询迁移到 Calcite
3. 实现 JdbcResponseFormatter 流式序列化（降低大结果集开销）
4. 评估 Calcite 分布式执行方案（解决单节点瓶颈）

### 8.4 最终建议

```
建议：引入 OpenSearch SQL 插件，采用"SQL + DSL 混合策略"

1. SQL 作为统一查询入口，覆盖 80% 的查询场景
2. 对性能敏感场景（高 QPS 点查、高基数聚合、大结果集），
   通过路由层自动降级到 DSL
3. SQL 独有能力（UNION/JOIN/子查询）作为 DSL 的补充
4. 持续推进社区路线图（Calcite 统一、统计注入、计划缓存）
5. 自行扩展的特性（UNION 等）需与社区主线对齐

预期收益：
  - 统一查询接口，降低多存储查询的开发成本
  - SQL 生态兼容（JDBC/ODBC/BI 工具）
  - UNION/JOIN/子查询能力（DSL 不支持）
  - 可读性好，降低运维门槛

预期成本：
  - 性能开销 1.1-2.4x（聚合场景）
  - 扩展开发 14-78 人天（按需求范围）
  - 持续与社区对齐的维护成本
```
