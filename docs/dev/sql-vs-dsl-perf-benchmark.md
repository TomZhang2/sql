# SQL vs DSL 查询性能对比验证方案

> 基于 OpenSearch 3.7.0 + SQL Plugin 3.7.0（含 UNION/UNION ALL 扩展）的白盒分析
> 数据量：100 万文档
> 所有 SQL 能力声明均已通过实测验证

---

## 第零部分：SQL 插件三种引擎概述

OpenSearch SQL 插件有三种执行引擎，是历史演进的结果。理解它们的差异是分析 SQL vs DSL 性能的前提。

### 0.1 Legacy V1 引擎（最老）

**是什么**：基于 Alibaba Druid SQL 解析器的查询翻译层。

```
SQL 字符串
  → Druid SQL Parser 解析 → Druid AST
  → OpenSearchActionFactory.create() → QueryAction 子类
  → QueryAction.explain() → SearchRequestBuilder (= DSL)
  → OpenSearch 搜索引擎执行
```

**特征**：
- **无计划抽象**：SQL 直接翻译成 OpenSearch DSL（`SearchRequestBuilder`），没有 LogicalPlan/PhysicalPlan 层
- **Druid 解析器**：使用 `com.alibaba.druid.sql.parser`，与 OpenSearch 生态无关
- **支持 JOIN**：2 表 JOIN（Hash Join / Nested Loop），但限制多
- **支持 IN 子查询**：通过 Hash Join 实现
- **支持 COALESCE、DATE_HISTOGRAM**：Druid 解析器原生理解这些函数
- **不支持**：3 表 JOIN、JOIN + GROUP BY、EXISTS 子查询、标量子查询、窗口函数
- **来源**：本项目 fork 自 `elasticsearch-sql`（NLPchina/elasticsearch-sql），那个项目用 Druid 做 SQL 翻译

**执行线程**：`sql-worker` 线程池（解析翻译）→ `search` 线程池（搜索引擎执行）

### 0.2 V2 引擎（当前主力）

**是什么**：OpenSearch 团队自研的现代化查询引擎，有完整的 AST → LogicalPlan → PhysicalPlan 抽象层。

```
SQL 字符串
  → ANTLR 4 解析 → ParseTree (CST)
  → AstBuilder → UnresolvedPlan (AST)
  → Analyzer → LogicalPlan (解析符号、类型、绑定 schema)
  → Planner → PhysicalPlan (物理执行算子树)
  → OpenSearchExecutionEngine.execute(PhysicalPlan)
      → PhysicalPlan 遍历 → 生成 SearchRequestBuilder (= DSL)
      → OpenSearch 搜索引擎执行
  → JdbcResponseFormatter → JSON 响应
```

**特征**：
- **完整计划抽象**：AST → LogicalPlan → PhysicalPlan 三层，支持优化器
- **ANTLR 4 文法**：自研 `OpenSearchSQLParser.g4`，与 OpenSearch 生态紧耦合
- **Visitor 模式**：`AbstractNodeVisitor`、`LogicalPlanNodeVisitor`、`PhysicalPlanNodeVisitor`
- **流式执行**：PhysicalPlan 实现 `Iterator<ExprValue>`，流式输出结果
- **游标分页**：通过 `PaginatedPlanCache` 序列化 PhysicalPlan 为 cursor
- **不支持**：JOIN（AstBuilder 抛异常回退 Legacy）、UNION（我们的扩展已改为走 Calcite）、CTE、EXISTS、标量子查询、COALESCE、DATE_HISTOGRAM
- **支持**：窗口函数（`RANK() OVER(...)`）、派生表（`(SELECT...) AS t`）、IN 子查询（回退 Legacy V1 Hash Join）

**为什么 V2 不支持 JOIN？**
`AstBuilder.visitJoinClause()` 主动抛出 `SyntaxCheckException`，触发回退到 Legacy V1。原因：V2 的 Analyzer 和 Planner 对 JOIN 的 schema 解析和物理执行尚未实现。

**为什么 V2 不支持 COALESCE/DATE_HISTOGRAM？**
函数注册表（`BuiltinFunctionRepository`）没有注册这些函数。DATE_HISTOGRAM 还有 INTERVAL 参数解析的 NPE bug。

### 0.3 Calcite 引擎（新引擎，PPL 默认 + SQL UNION）

**是什么**：基于 Apache Calcite 的查询引擎，利用 Calcite 的优化器和关系代数做查询规划。

```
SQL/PPL 字符串
  → ANTLR 4 解析 → ParseTree (CST)
  → AstBuilder → UnresolvedPlan (AST)
  → CalciteRelNodeVisitor.analyze(plan, context) → Calcite RelNode
  → convertToCalcitePlan() → 加 LogicalSystemLimit
  → OpenSearchExecutionEngine.execute(RelNode, CalcitePlanContext)
      → OpenSearchRelRunners.run() → JDBC PreparedStatement
      → statement.executeQuery() → ResultSet
      → (Calcite 内部将可下推的操作生成 DSL 发给 OpenSearch)
      → (不可下推的操作在内存中用 Calcite enumerable 算子计算)
  → buildResultSet() → JdbcResponseFormatter → JSON 响应
```

**特征**：
- **Calcite 优化器**：基于关系代数的 Volcano planner。注意：当前 OpenSearch 集成**未注入统计信息**（无 `RelMetadataProvider` 扩展），代价优化实际退化为规则系统。要发挥真正的 CBO 能力，需实现统计信息注入（参见第四部分 4.6 节）。
- **算子下推**：通过 Calcite 的 `Convention` trait 机制驱动（`Logical` → `Enumerable` → `OpenSearchRel`）。filter、aggregation、sort、limit 可下推到 OpenSearch 搜索引擎。下推是规则驱动的，不是代价驱动的。
- **Janino codegen**：Enumerable 算子通过 Janino 在运行时编译为 Java 字节码。首次执行每个查询形状有 ~10-50ms 编译开销，后续执行走已编译代码。50 轮预热可覆盖此成本，但生产中遇到新查询形状时仍需支付。
- **内存计算**：UNION 合并、未下推的 JOIN 等在内存中用 Enumerable 算子计算（单节点单线程，无分布式并行）
- **与 V2 共享 AST**：前端解析（ANTLR + AstBuilder）相同，从 `QueryService.shouldUseCalcite()` 开始分叉
- **PPL 默认走此路径**：`plugins.calcite.enabled=true`（3.3.0 起默认），PPL 查询走 Calcite
- **SQL 仅 UNION 走此路径**：我们的扩展让 `shouldUseCalcite` 检测到 Union 节点时路由到 Calcite（`containsUnion(plan)`）
- **LogicalSystemLimit**：Calcite 路径默认加 `LogicalSystemLimit(fetch=plugins.query.size_limit)`，V2 路径无此限制

**为什么 SQL 默认不走 Calcite？**
`QueryService.shouldUseCalcite()` 中有硬编码限制：
```java
// TODO https://github.com/opensearch-project/sql/issues/3457
// Calcite is not available for SQL query now. Maybe release in 3.1.0?
private boolean shouldUseCalcite(QueryType queryType, UnresolvedPlan plan) {
    if (!isCalciteEnabled(settings)) return false;
    if (queryType == QueryType.PPL) return true;
    return queryType == QueryType.SQL && containsUnion(plan);  // ← 我们的扩展
}
```
SQL 走 Calcite 的完整支持还在开发中（issue #3457），当前仅 UNION 通过我们的扩展走了过来。

### 0.4 三种引擎对比

| 维度 | Legacy V1 | V2 | Calcite |
|------|-----------|-----|---------|
| **解析器** | Druid SQL Parser | ANTLR 4 (自研文法) | ANTLR 4 (同 V2) |
| **计划抽象** | 无（直接翻译为 DSL） | AST → LogicalPlan → PhysicalPlan | AST → RelNode (Calcite 关系代数) |
| **优化器** | 无 | 简单规则 | Calcite 代价优化器 |
| **算子下推** | 无 | 无（直接生成 DSL） | ✅ filter/agg/sort/limit 下推 |
| **JOIN** | ✅ 2 表（Hash/Nested Loop） | ❌（回退 Legacy） | ✅ N 表（但 SQL 未路由到此） |
| **UNION** | ✅（Druid 支持） | ❌（我们的扩展已改为走 Calcite） | ✅（我们的扩展） |
| **窗口函数** | ❌ | ✅ | ✅ |
| **游标分页** | ✅（自己的 cursor 机制） | ✅（序列化 PhysicalPlan） | ⚠️（EnumerableLimit 分页，无序列化游标） |
| **COALESCE** | ✅ | ❌ | ✅（Calcite 原生） |
| **DATE_HISTOGRAM** | ✅ | ❌（NPE bug） | ✅（下推 date_histogram 聚合） |
| **回退机制** | 是 V2 的回退目标 | 是 Calcite 的回退目标 | 失败可回退到 V2 |
| **内存计算** | 无 | 无 | 有（Enumerable 算子） |
| **状态** | 维护中（不再新增功能） | 活跃开发 | 活跃开发（未来方向） |
| **引入版本** | 1.0（fork 自 elasticsearch-sql） | 2.0+ | 3.0+（PPL），3.7+（SQL UNION） |

### 0.5 为什么会有三种引擎？

这是**历史演进**的结果：

#### 阶段一：Legacy V1（1.0 ~ 2.x）

OpenSearch SQL 插件 fork 自 `elasticsearch-sql`（NLPchina），那个项目用 Druid SQL 解析器把 SQL 翻译成 Elasticsearch DSL。简单直接，但：
- 无计划抽象，无法做查询优化
- Druid 解析器与 OpenSearch 生态脱节
- 扩展新 SQL 语法需要改 Druid 的 Java 代码，不灵活
- JOIN 等复杂查询性能差（Hash Join 在内存中做）

#### 阶段二：V2 引擎（2.0+ ~ 现在）

OpenSearch 团队自研 V2 引擎，目标：
- 完整的 AST → LogicalPlan → PhysicalPlan 抽象层
- 用 ANTLR 4 自定义文法，与 OpenSearch 生态紧耦合
- 支持 Visitor 模式，方便扩展
- 流式执行（`Iterator<ExprValue>`）

但 V2 的开发**优先服务 PPL**（Piped Processing Language），SQL 的支持是次要目标。很多 SQL 特性（JOIN、UNION、CTE）在 V2 中没有实现，遇到时抛 `SyntaxCheckException` 回退到 Legacy V1。

#### 阶段三：Calcite 引擎（3.0+ ~ 现在）

引入 Apache Calcite 作为新的查询引擎，目标：
- 利用 Calcite 的成熟优化器（代价优化、算子下推）
- 统一 PPL 和 SQL 的执行路径
- 支持更复杂的查询（N-way JOIN、子查询优化）
- 与外部系统集成（Spark、CLI 工具）

Calcite 引擎**首先在 PPL 上落地**（3.0 引入，3.3 默认启用）。SQL 走 Calcite 的工作还在进行中（issue #3457），我们的 UNION 扩展是第一步。

#### 演进方向

```
Legacy V1（淘汰中）
      ↓ 回退到
V2 引擎（当前主力，维护中）
      ↓ 迁移到
Calcite 引擎（未来方向，活跃开发）
```

最终目标是 **SQL 和 PPL 都走 Calcite 引擎**，Legacy V1 退役，V2 的 PhysicalPlan 层可能被 Calcite RelNode 替代。但这个迁移工作量大，当前处于过渡期——三种引擎并存，通过 `SyntaxCheckException` 和 `shouldUseCalcite` 做路由和回退。

### 0.6 当前路由逻辑（我们的扩展后）

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

---

## 第一部分：白盒实现分析

### 1.1 DSL 查询路径（直接路径）

```
客户端
  │ POST /index/_search {"query": {"match": ...}}
  ▼
OpenSearch TransportAction
  │ RestSearchAction → TransportSearchAction
  ▼
SearchPhaseExecution
  │ QueryPhase → FetchPhase
  │ (直接在搜索引擎内核执行)
  ▼
JSON 响应
```

**关键特征：**
- 零翻译层：请求体本身就是搜索引擎的执行指令
- 零中间对象：JSON 直接反序列化为 SearchRequest
- 零额外内存：不构建 AST / Plan / RelNode
- 执行线程：OpenSearch `search` 线程池（8 核机器上默认 13 线程）

### 1.2 SQL 查询路径（翻译路径）

#### 1.2.1 V2 引擎路径（普通 SELECT）

```
客户端
  │ POST /_plugins/_sql {"query": "SELECT ..."}
  ▼
RestSqlAction → RestSQLQueryAction
  │
  ▼
SQLService.plan()
  │ ① SQLSyntaxParser.parse()        — ANTLR 词法+语法分析 → ParseTree (CST)
  │ ② AstStatementBuilder + AstBuilder — CST → UnresolvedPlan (AST)
  │ ③ QueryPlanFactory.create()       — AST → QueryPlan
  ▼
QueryService.executeWithLegacy()
  │ ④ Analyzer.analyze(plan)          — AST → LogicalPlan (解析符号/类型)
  │ ⑤ Planner.plan(logicalPlan)       — LogicalPlan → PhysicalPlan
  ▼
OpenSearchExecutionEngine.execute(PhysicalPlan)
  │ ⑥ PhysicalPlan 遍历 → 生成 SearchRequestBuilder (= DSL!)
  │ ⑦ client.schedule(() -> plan.open(); plan.hasNext(); ...)
  ▼
OpenSearch 搜索引擎执行 (与 DSL 路径相同)
  │
  ▼
结果 → JdbcResponseFormatter → JSON 响应
```

#### 1.2.2 Calcite 引擎路径（UNION ALL / UNION / PPL）

```
客户端
  │ POST /_plugins/_sql {"query": "SELECT ... UNION ALL SELECT ..."}
  ▼
RestSqlAction → RestSQLQueryAction → SQLService
  │ ①②③ 同 V2 路径
  ▼
QueryService.executeWithCalcite()    ← shouldUseCalcite 检测到 Union 节点
  │ ④ CalciteRelNodeVisitor.analyze(plan)  — AST → Calcite RelNode
  │ ⑤ convertToCalcitePlan()               — 加 LogicalSystemLimit
  ▼
OpenSearchExecutionEngine.execute(RelNode)
  │ ⑥ OpenSearchRelRunners.run() → PreparedStatement
  │ ⑦ statement.executeQuery() → ResultSet
  │    (Calcite 内部生成 DSL 下推到 OpenSearch)
  ▼
buildResultSet() → JdbcResponseFormatter → JSON 响应
```

> ⚠️ **Calcite 回退风险**：Calcite 执行失败时会静默回退到 V2（`QueryService.java:181-183`）。
> 测试时需通过日志监控是否发生回退，否则会将 V2 延迟误归因为 Calcite。

#### 1.2.3 Legacy V1 引擎路径（JOIN / IN 子查询回退）

```
客户端
  │ POST /_plugins/_sql {"query": "SELECT ... JOIN ..."}
  ▼
RestSqlAction → RestSQLQueryAction → SQLService
  │ AstBuilder.visitJoinClause() → 抛出 SyntaxCheckException
  ▼
fallBackListener 捕获 SyntaxCheckException
  │
  ▼
Legacy V1 引擎 (SearchDao → Druid 解析器)
  │ → QueryAction → SearchRequestBuilder
  ▼
OpenSearch 搜索引擎执行 → PrettyFormatRestExecutor → 响应
```

> **关键**：JOIN 和 IN 子查询在常规 `/_plugins/_sql` 端点上会回退到 **Legacy V1 引擎**（Druid 解析器），
> 而非 V2 或 Calcite。仅当索引设置为 composite dataformat 时才走 Unified Query API（Calcite）。

### 1.3 实现差异对比

| 环节 | DSL | SQL (V2) | SQL (Calcite) | SQL (Legacy V1 回退) |
|------|-----|----------|---------------|---------------------|
| **请求解析** | JSON 反序列化 (~0.1ms) | ANTLR 解析 (~0.5-2ms) | ANTLR 解析 (~0.5-2ms) | Druid 解析 (~1-3ms) |
| **AST 构建** | 无 | AstBuilder (~0.3-1ms) | AstBuilder (~0.3-1ms) | Druid AST (~0.5-1ms) |
| **语义分析** | 无 | Analyzer (~0.5-2ms) | CalciteRelNodeVisitor (~1-5ms) | 无（Druid 内联） |
| **查询规划** | 无 | Planner (~0.3-1ms) | Calcite 优化器 (~2-8ms) | 无 |
| **DSL 生成** | 无（本身就是 DSL） | PhysicalPlan → SearchRequestBuilder (~0.2-1ms) | Calcite 下推生成 DSL (~0.5-2ms) | QueryAction → SearchRequestBuilder (~0.5-1ms) |
| **执行引擎** | OpenSearch 搜索引擎 | OpenSearch 搜索引擎 | OpenSearch 搜索引擎（下推部分）+ Calcite 内存计算（未下推部分） | OpenSearch 搜索引擎 |
| **结果格式化** | JSON 序列化 (~0.5ms) | JdbcResponseFormatter (~1-2ms) | JdbcResponseFormatter (~1-2ms) | PrettyFormatRestExecutor (~1-2ms) |
| **内存开销** | 极低（SearchRequest 对象） | 中（AST + LogicalPlan + PhysicalPlan） | 高（RelNode + CalcitePlanContext + 优化器状态） | 低（Druid AST） |
| **线程池** | search (8核→13线程) | sql-worker (8核→8线程) | sql-worker (8核→8线程) | sql-worker → search |
| **总额外开销** | ~0ms | ~2-7ms | ~5-18ms | ~2-6ms |

### 1.4 关键差异点

1. **SQL 最终都会翻译成 DSL**：无论是 V2、Calcite 还是 Legacy 路径，最终执行都是生成 `SearchRequestBuilder` 调用 OpenSearch 搜索引擎。差异只在上层翻译开销。

2. **三种 SQL 引擎路径**：
   - **V2 引擎**：普通 SELECT/WHERE/GROUP BY/ORDER BY 走此路径
   - **Calcite 引擎**：仅 UNION/UNION ALL 走此路径（我们的扩展）；PPL 默认走此路径
   - **Legacy V1 引擎**：JOIN、IN 子查询回退到此路径（AstBuilder 抛 SyntaxCheckException 触发）

3. **Calcite 的下推机制**：通过 Convention trait（`Logical` → `Enumerable` → `OpenSearchRel`）驱动，将 filter、aggregation、sort、limit 下推到 OpenSearch。下推是规则驱动的。不可下推的操作（如 UNION 合并、未下推的 JOIN）在内存中用 Enumerable 算子计算（单节点单线程，无分布式并行）。

4. **游标实现不同**：
   - DSL 用 `search_after`（无状态，需要排序字段唯一）
   - SQL V2 用序列化游标（有状态，占用服务端内存，通过 `PaginatedPlanCache` 序列化 PhysicalPlan）
   - SQL Calcite 路径不支持 V2 序列化游标，但通过 `EnumerableLimit` 原生支持分页（Union 查询的分页由 Calcite 处理）

5. **线程池大小差异**：
   - `search` 线程池：8 核机器上 13 线程（`int((cores * 3) / 2) + 1`）
   - `sql-worker` 线程池：8 核机器上 8 线程（`allocatedProcessors`）
   - 单线程测试无影响；并发测试时 DSL 有 62.5% 更多线程

6. **计划缓存（重要警告）**：
   - V2 和 Calcite 路径**均未实现按 SQL 字符串缓存编译计划**。每次查询都重新解析+规划。
   - 但 ANTLR parser 内部可能有 token 缓存，Calcite 的 `CalcitePrepareImpl` 可能缓存 PreparedStatement。
   - 基准测试中重复相同查询字符串会受益于任何隐式缓存，导致翻译开销被低估。
   - **必须增加冷启动场景**（每轮用唯一注释 `/* :run_id */` 打散缓存）和**参数化场景**（同形状不同字面量）。

7. **Calcite 内存计算风险**：
   - Enumerable 算子在协调节点单线程执行，无分布式并行。
   - 大数据量 UNION/JOIN（10k+ 行）可能导致 OOM。
   - Presto/Spark 通过 exchange/shuffle 解决分布式执行，Calcite-on-OpenSearch 无此能力。
   - 基准测试需增加大数据量内存压力场景。

8. **OpenSearch 缓存影响公平性**：
   - **request cache**：`size:0` 聚合请求被缓存。聚合场景 C1-C3 会被缓存主导。
   - **filter cache**：`bool.filter` 条件被缓存。SQL 和 DSL 均可命中，但需验证生成的 DSL 形状一致。
   - **query cache**：查询结果在 shard 级别缓存。
   - 基准测试**必须禁用或随机化**这些缓存（见 3.2 节注意事项）。

---

## 第二部分：能力对比（已实测验证）

### 2.1 查询能力矩阵

> 以下所有 SQL 能力声明均通过 `POST /_plugins/_sql` 实测验证（2026-07-18）

| 能力 | SQL | DSL | 说明 |
|------|:---:|:---:|------|
| 等值/范围查询 | ✅ | ✅ | SQL: `WHERE x = 1`; DSL: `term`/`range` |
| 多条件布尔查询 | ✅ | ✅ | SQL: `AND`/`OR`; DSL: `bool` query |
| 全文搜索 | ⚠️ 受限 | ✅ | SQL: `match()` / `multi_match()` 函数; DSL: 完整 `match` 参数 (boost, fuzziness, minimum_should_match) |
| 模糊搜索 | ⚠️ 受限 | ✅ | SQL: `LIKE`; DSL: `fuzzy` + 编辑距离 |
| 短语搜索 | ⚠️ 受限 | ✅ | SQL: 不直接支持; DSL: `match_phrase` + slop |
| 多字段搜索 | ✅ | ✅ | SQL: `MULTI_MATCH(field, query)` ✅ 实测支持; DSL: `multi_match` + tie_breaker |
| GROUP BY 聚合 | ✅ | ✅ | SQL: `GROUP BY`; DSL: `aggs` |
| 多级 GROUP BY | ✅ | ✅ | SQL: `GROUP BY a, b`（flat 聚合）; DSL: `aggs` 嵌套（树形聚合，执行路径不同） |
| 窗口函数 | ✅ | ❌ | SQL: `RANK() OVER(...)` ✅ 实测支持; DSL: 不支持 |
| 2 表 JOIN | ✅ | ❌ | SQL: `JOIN`（回退 Legacy V1）; DSL: 不支持 |
| 3 表及以上 JOIN | ❌ | ❌ | SQL: ✗ 实测报错 "currently supports only 2 tables join"; DSL: 不支持 |
| JOIN + WHERE | ✅ | ❌ | SQL: ✅ 实测支持（Legacy V1）; DSL: 不支持 |
| JOIN + ORDER BY | ✅ | ❌ | SQL: ✅ 实测支持（Legacy V1）; DSL: 不支持 |
| JOIN + LIMIT | ✅ | ❌ | SQL: ✅ 实测支持（Legacy V1）; DSL: 不支持 |
| JOIN + GROUP BY | ❌ | ❌ | SQL: ✗ 实测报错 "JOIN queries do not support aggregations"; DSL: 不支持 |
| JOIN + 聚合函数 | ❌ | ❌ | SQL: ✗ 实测报错（同上）; DSL: 不支持 |
| UNION ALL | ✅ | ❌ | SQL: ✅ 实测支持（我们的扩展，走 Calcite）; DSL: 不支持 |
| UNION DISTINCT | ✅ | ❌ | SQL: ✅ 实测支持（我们的扩展，走 Calcite）; DSL: 不支持 |
| IN 子查询 | ✅ | ❌ | SQL: ✅ 实测支持（回退 Legacy V1 Hash Join）; DSL: 不支持 |
| EXISTS 子查询 | ❌ | ❌ | SQL: ✗ 实测报错 "Unsupported subquery"; DSL: 不支持 |
| 标量子查询 (SELECT 中) | ❌ | ❌ | SQL: ✗ 实测报错; DSL: 不支持 |
| 派生表 (FROM 子查询) | ✅ | ❌ | SQL: ✅ 实测支持 `(SELECT ...) AS t`; DSL: 不支持 |
| 子查询 + 外层 GROUP BY | ❌ | ❌ | SQL: ✗ 实测报错（Druid cast 异常）; DSL: 不支持 |
| CTE (WITH) | ❌ | ❌ | SQL: ✗ 实测报错 "Query must start with SELECT"; DSL: 不支持 |
| COALESCE 函数 | ❌ | ✅ | SQL: ✗ 实测报错 "not supported in Schema: COALESCE"; DSL: 可用 script 实现 |
| DATE_HISTOGRAM 函数 | ❌ | ✅ | SQL: ✗ 实测 NPE（V2 解析失败）; DSL: `date_histogram` 聚合 |
| HISTOGRAM 函数 | ❌ | ✅ | SQL: ✗ 实测 NPE（V2 解析失败）; DSL: `histogram` 聚合 |
| 排序 | ✅ | ✅ | 都支持，DSL 更灵活（script sort, geo sort） |
| 分页 | ✅ | ✅ | SQL: `LIMIT`/`fetch_size`; DSL: `from`/`size`/`search_after` |
| 脚本字段 | ❌ | ✅ | SQL: 不支持; DSL: `script_fields` |
| 运行时字段 | ❌ | ✅ | SQL: 不支持; DSL: `runtime_mappings` |
| 高亮 | ⚠️ 受限 | ✅ | SQL: `highlight()`; DSL: 完整高亮配置 |
| 地理查询 | ⚠️ 受限 | ✅ | SQL: 受限; DSL: `geo_distance`/`geo_bounding_box` 等 |
| 建议 | ❌ | ✅ | SQL: 不支持; DSL: `suggest` |
| 折叠去重 | ❌ | ✅ | SQL: `DISTINCT`; DSL: `collapse` |

### 2.2 SQL 独有能力（DSL 不支持的，均已实测验证）

| 能力 | SQL 语法 | 实现路径 | 状态 |
|------|---------|---------|:----:|
| 2 表 JOIN | `SELECT ... FROM a JOIN b ON ...` | Legacy V1 引擎（AstBuilder 抛异常回退） | ✅ 可用 |
| JOIN + WHERE | `SELECT ... FROM a JOIN b ON ... WHERE ...` | Legacy V1 引擎 | ✅ 可用 |
| JOIN + ORDER BY | `SELECT ... FROM a JOIN b ON ... ORDER BY ...` | Legacy V1 引擎 | ✅ 可用 |
| JOIN + LIMIT | `SELECT ... FROM a JOIN b ON ... LIMIT n` | Legacy V1 引擎 | ✅ 可用 |
| UNION ALL | `SELECT ... UNION ALL SELECT ...` | Calcite 引擎（我们的扩展） | ✅ 可用 |
| UNION DISTINCT | `SELECT ... UNION SELECT ...` | Calcite 引擎（我们的扩展） | ✅ 可用 |
| IN 子查询 | `SELECT ... WHERE x IN (SELECT ...)` | Legacy V1 引擎（Hash Join） | ✅ 可用 |
| 派生表 | `SELECT ... FROM (SELECT ...) AS t` | V2 引擎 | ✅ 可用 |
| 窗口函数 | `SELECT rank() OVER (PARTITION BY ...)` | V2 引擎 | ✅ 可用 |

### 2.3 SQL 已知限制（均已实测验证）

| 限制 | 错误信息 | 实际行为 |
|------|---------|---------|
| CTE (WITH) | `Query must start with SELECT, DELETE, SHOW or DESCRIBE` | 文法无 WITH 规则，完全不支持 |
| 3 表及以上 JOIN | `currently supports only 2 tables join` | Legacy V1 限制 |
| JOIN + GROUP BY | `JOIN queries do not support aggregations on the joined result` | Legacy V1 限制 |
| JOIN + 聚合函数 | 同上 | Legacy V1 限制 |
| EXISTS 子查询 | `Unsupported subquery` | V2 不支持，未回退到 Legacy |
| 标量子查询 (SELECT 中) | `unknown field name : ...SQLSelect@...` | Druid 解析器无法处理 |
| 子查询 + 外层 GROUP BY | `class SQLSubqueryTableSource cannot be cast to SQLJoinTableSource` | Druid 解析器类型转换失败 |
| COALESCE | `The following method is not supported in Schema: COALESCE` | V2 Schema 不识别此函数 |
| DATE_HISTOGRAM | NPE: `Cannot invoke "Object.toString()" because "this.value" is null` | V2 解析器 NPE，非 Legacy 语法 |
| HISTOGRAM | 同上 NPE | V2 解析器 NPE |

### 2.4 DSL 独有能力（SQL 不支持或不完整的）

| 能力 | DSL 语法 | 说明 |
|------|---------|------|
| 复杂 bool query | `bool: { must, should, filter, must_not }` | SQL 只能 AND/OR，无法区分 filter context |
| script_fields | `script_fields: { ... }` | 运行时计算字段值 |
| runtime_mappings | `runtime_mappings: { ... }` | 运行时定义字段 |
| suggest | `suggest: { ... }` | 搜索建议 |
| collapse | `collapse: { field: ... }` | 按字段去重 |
| aggs 嵌套 | `aggs: { a: { aggs: { b: ... } } }` | 多级嵌套聚合（SQL 的 GROUP BY a, b 是 flat，执行路径不同） |
| date_histogram 聚合 | `aggs: { h: { date_histogram: { field: ..., calendar_interval: "1h" } } }` | SQL 的 DATE_HISTOGRAM 函数 NPE |
| profile API | `profile: true` | 查询性能分析 |
| search template | `_search/template` | 模板化查询 |

---

## 第三部分：验证方案设计

### 3.1 测试环境

#### 3.1.1 硬件配置

```
OpenSearch 节点：1 个（单节点，消除分布式变量）
CPU: 8 核
内存: 16GB（JVM heap 8GB）
磁盘: SSD 200GB
网络: 本地回环（消除网络延迟）
OS: macOS / Linux
JDK: 21（启用 `-XX:+PrintCompilation` 监控 JIT 活动；启用 `-Xlog:gc*` 监控 GC）
```

> ⚠️ **shard 数量选择**：
> - **3-shard 1-replica 为主测试配置**（生产最小可用配置）
> - 1-shard 为参考下限（DSL 最佳场景，SQL 开销占比上限）
> - 多 shard 下 DSL 有 scatter-gather + merge 开销但 SQL 翻译开销不变，SQL 相对开销随 shard 数下降
> - **1-shard 结论会系统性高估 SQL 劣势 2-3 倍**，不可作为生产推断依据

#### 3.1.2 软件配置

```
OpenSearch: 3.7.0
SQL Plugin: 3.7.0.0-SNAPSHOT（含 UNION/UNION ALL 扩展）
plugins.calcite.enabled: true
plugins.calcite.pushdown.enabled: true
plugins.query.size_limit: 100000
plugins.query.memory_limit: 80%
index.max_result_window: 20000  ← 深度分页场景需要
```

#### 3.1.3 索引设计

创建一个模拟日志索引 `perf_test`，包含 100 万文档：

```json
{
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "service": { "type": "keyword" },
      "host": { "type": "keyword" },
      "message": { "type": "text" },
      "status_code": { "type": "integer" },
      "response_time_ms": { "type": "integer" },
      "bytes": { "type": "long" },
      "user_id": { "type": "keyword" },
      "region": { "type": "keyword" }
    }
  },
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.refresh_interval": "30s",
    "index.max_result_window": 20000
  }
}
```

#### 3.1.4 数据生成与导入

```python
# generate_data.py — 生成 100 万条测试数据
import random
import json

levels = ["INFO", "WARN", "ERROR", "DEBUG"]
services = ["auth-service", "payment-service", "order-service", "search-service", "notification-service"]
hosts = ["host-01", "host-02", "host-03", "host-04", "host-05"]
regions = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1", "ap-northeast-1"]
messages = [
    "Request processed successfully",
    "Connection timeout to database",
    "User authentication failed",
    "Cache miss for key",
    "Rate limit exceeded",
    "Background job completed",
    "Configuration reloaded",
    "Health check passed"
]

with open("bulk_data.json", "w") as f:
    for i in range(1, 1000001):
        doc = {
            "@timestamp": f"2026-07-{(i % 30)+1:02d}T{(i % 24):02d}:{(i % 60):02d}:{(i % 60):02d}Z",
            "level": random.choice(levels),
            "service": random.choice(services),
            "host": random.choice(hosts),
            "message": random.choice(messages),
            "status_code": random.choice([200, 200, 200, 200, 301, 404, 500, 503]),
            "response_time_ms": random.randint(1, 5000),
            "bytes": random.randint(100, 100000),
            "user_id": f"user-{i % 10000}",
            "region": random.choice(regions)
        }
        f.write(json.dumps({"index": {"_index": "perf_test"}}) + "\n")
        f.write(json.dumps(doc) + "\n")
```

```bash
# 导入数据
curl -X POST "localhost:9200/_bulk?refresh=false" -H 'Content-Type: application/x-ndjson' --data-binary @bulk_data.json

# 轮询等待索引完成（不要用固定 sleep）
while [ "$(curl -s "localhost:9200/_cat/indices/perf_test?h=docs.count")" -lt "1000000" ]; do
  echo "Waiting for indexing... ($(curl -s 'localhost:9200/_cat/indices/perf_test?h=docs.count')/1000000)"
  sleep 5
done

# 强制 merge 优化查询性能
curl -X POST "localhost:9200/perf_test/_forcemerge?max_num_segments=1"
```

### 3.2 测试场景设计

按查询复杂度分 5 个级别，每个级别设计 SQL 和 DSL 对等查询。

> **对等性原则**：DSL 等值/范围查询使用 `bool.filter`（不打分），与 SQL `WHERE` 语义对等。
> 全文搜索使用 `match`（打分），与 SQL `match()` 语义对等。

#### 场景组 A：点查（Point Lookup）

**A1. 等值查询 — 单条件**

```sql
-- SQL
SELECT * FROM perf_test WHERE status_code = 200 LIMIT 10;
```
```json
// DSL
{"query": {"term": {"status_code": 200}}, "size": 10}
```

**A2. 等值查询 — 多条件**

```sql
-- SQL
SELECT * FROM perf_test WHERE status_code = 500 AND level = 'ERROR' AND region = 'us-east-1' LIMIT 10;
```
```json
// DSL
{"query": {"bool": {"filter": [
  {"term": {"status_code": 500}},
  {"term": {"level": "ERROR"}},
  {"term": {"region": "us-east-1"}}
]}}, "size": 10}
```

**A3. 范围查询**

```sql
-- SQL
SELECT * FROM perf_test WHERE response_time_ms > 3000 AND status_code = 500 LIMIT 10;
```
```json
// DSL
{"query": {"bool": {"filter": [
  {"range": {"response_time_ms": {"gt": 3000}}},
  {"term": {"status_code": 500}}
]}}, "size": 10}
```

#### 场景组 B：全文搜索

**B1. 简单全文搜索**

```sql
-- SQL
SELECT message, service FROM perf_test WHERE match(message, 'timeout') LIMIT 10;
```
```json
// DSL
{"query": {"match": {"message": "timeout"}}, "size": 10, "_source": ["message", "service"]}
```

**B2. 多字段搜索**

```sql
-- SQL
SELECT * FROM perf_test WHERE MULTI_MATCH(message, 'request failed') AND level = 'ERROR' LIMIT 10;
```
```json
// DSL
{"query": {"bool": {"must": [
  {"match": {"message": "request failed"}}
], "filter": [
  {"term": {"level": "ERROR"}}
]}}, "size": 10}
```

> 注：SQL `WHERE` 对 keyword 字段生成 `bool.filter`（不打分），对 `match()` 生成 `bool.must`（打分）。
> 与 DSL 对照组的 filter/must 分工一致，确保公平。已通过 `_explain` 验证。

#### 场景组 C：聚合查询

> ⚠️ **request cache 警告**：OpenSearch 缓存 `size:0` 聚合请求。基准测试中重复相同聚合会命中缓存，测到的是缓存查找（~0.5ms）而非聚合计算。
> - DSL 请求必须加 `?request_cache=false` 参数
> - SQL 请求需验证插件是否转发此参数；如不可控，**每轮使用不同的 WHERE 条件**（如 `WHERE response_time_ms > <random_threshold>`）打散缓存
> - 分别报告 cache-hit 和 cache-miss 结果

**C1. 简单聚合（随机化阈值打散缓存）**

```sql
-- SQL — 每轮用不同阈值
SELECT level, COUNT(*) as cnt FROM perf_test WHERE response_time_ms > {random_threshold} GROUP BY level;
```
```json
// DSL — request_cache=false + 随机化
// POST /perf_test/_search?request_cache=false
{"size": 0, "query": {"range": {"response_time_ms": {"gt": <random_threshold>}}}, "aggs": {"by_level": {"terms": {"field": "level"}}}}
```

**C2. 多级聚合（flat GROUP BY vs composite aggregation）**

```sql
-- SQL — 每轮用不同阈值
SELECT level, service, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt
FROM perf_test
WHERE response_time_ms > {random_threshold}
GROUP BY level, service;
```
```json
// DSL — request_cache=false + 随机化
{"size": 0, "query": {"range": {"response_time_ms": {"gt": <random_threshold>}}}, "aggs": {"by_level_service": {"composite": {"sources": [
  {"level": {"terms": {"field": "level"}}},
  {"service": {"terms": {"field": "service"}}}
]}, "aggs": {"avg_rt": {"avg": {"field": "response_time_ms"}}}}}}
```

> ⚠️ **语义对等说明**：SQL `GROUP BY level, service` 是 flat 多键聚合（所有组合在一层）。
> DSL 的 nested `aggs` 是树形嵌套聚合（不同执行路径）。
> 为确保公平对比，DSL 使用 `composite` 聚合（也是 flat 多键），而非 nested `aggs`。

**C3. 范围聚合**

```sql
-- SQL — 每轮用不同阈值
SELECT service, COUNT(*) as cnt
FROM perf_test
WHERE response_time_ms > {random_threshold}
GROUP BY service
ORDER BY cnt DESC;
```
```json
// DSL — request_cache=false + 随机化
{"size": 0, "query": {"range": {"response_time_ms": {"gt": <random_threshold>}}}, "aggs": {
  "by_service": {"terms": {"field": "service", "order": {"_count": "desc"}}}
}}
```

**C4. 时间直方图聚合（仅 DSL，SQL 不支持）**

> ⚠️ SQL 的 `DATE_HISTOGRAM()` 和 `HISTOGRAM()` 函数在 V2 引擎中 NPE 崩溃（已实测验证）。
> 此场景仅测 DSL，作为 SQL 能力缺失的参考。

```json
// DSL
{"size": 0, "aggs": {
  "by_hour": {"date_histogram": {"field": "@timestamp", "calendar_interval": "1h"}}
}}
```

#### 场景组 D：排序与分页

**D1. 排序 + 小分页**

```sql
-- SQL
SELECT * FROM perf_test ORDER BY response_time_ms DESC LIMIT 10;
```
```json
// DSL
{"query": {"match_all": {}}, "sort": [{"response_time_ms": "desc"}], "size": 10}
```

**D2. 深度分页**

> 需要 `index.max_result_window: 20000`（已在索引设置中配置）

```sql
-- SQL
SELECT * FROM perf_test ORDER BY response_time_ms DESC LIMIT 10000, 10;
```
```json
// DSL
{"query": {"match_all": {}}, "sort": [{"response_time_ms": "desc"}], "from": 10000, "size": 10}
```

**D3. 大结果集（新增）**

```sql
-- SQL
SELECT * FROM perf_test WHERE status_code = 200 LIMIT 1000;
```
```json
// DSL
{"query": {"term": {"status_code": 200}}, "size": 1000}
```

> 此场景测试 `JdbcResponseFormatter` 随结果行数增长的格式化开销（SQL 独有开销）。

**D4. 游标分页（新增）**

```sql
-- SQL — 使用 fetch_size 触发游标分页
-- 请求1:
{"query": "SELECT * FROM perf_test WHERE status_code = 200", "fetch_size": 100}
-- 请求2-10: 使用返回的 cursor 继续获取
```
```json
// DSL — 使用 search_after 无状态分页
// 请求1:
{"query": {"term": {"status_code": 200}}, "sort": [{"_id": "asc"}], "size": 100}
// 请求2-10: 使用上一次结果的 sort 值作为 search_after
```

> 此场景测试游标机制差异：SQL V2 序列化游标（有状态）vs DSL search_after（无状态）。
> 需测量每页延迟和累计服务端内存。

#### 场景组 E：SQL 独有能力（无 DSL 等价对照）

**E1. UNION ALL（走 Calcite 引擎，我们的扩展）**

```sql
-- SQL（无 DSL 等价，需两次查询 + 应用层合并）
SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service
UNION ALL
SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service;
```
```python
# DSL 等价（两次查询 + 应用层合并）
# 查询1:
{"size": 0, "query": {"term": {"level": "ERROR"}}, "aggs": {"by_service": {"terms": {"field": "service"}}}}
# 查询2:
{"size": 0, "query": {"term": {"level": "WARN"}}, "aggs": {"by_service": {"terms": {"field": "service"}}}}
# 应用层合并两个结果
```

**E2. 2 表 JOIN（走 Legacy V1 引擎，注意不是 Calcite）**

> ⚠️ JOIN 在常规 `/_plugins/_sql` 端点回退到 **Legacy V1 引擎**（Druid 解析器），不是 V2 也不是 Calcite。
> 仅 Unified Query API（composite 索引）的 JOIN 才走 Calcite。
> 此场景测量的是 Legacy V1 的 JOIN 性能。

```sql
-- SQL（无 DSL 等价）
-- 需要第二个索引：perf_test_meta
SELECT a.service, a.level, b.host_name
FROM perf_test a
JOIN perf_test_meta b ON a.host = b.host
WHERE a.level = 'ERROR'
LIMIT 10;
```
```python
# DSL 等价（需多次查询 + 应用层关联）
# 1. 先查 perf_test 获取 host 列表
# 2. 再查 perf_test_meta 获取 host 详情
# 3. 应用层关联
```

**E3. IN 子查询（走 Legacy V1 引擎）**

```sql
-- SQL（无 DSL 等价，回退 Legacy V1 Hash Join）
SELECT * FROM perf_test
WHERE host IN (SELECT host FROM perf_test WHERE level = 'ERROR')
LIMIT 10;
```

**E4. 大数据量 UNION ALL（内存压力测试，新增）**

> 测试 Calcite Enumerable 算子在大量数据下的内存表现和 OOM 风险。
> Calcite 内存计算在协调节点单线程执行，无分布式并行。

```sql
-- SQL — 每分支返回大量行（不做聚合，直接 UNION ALL）
SELECT service, response_time_ms FROM perf_test WHERE status_code = 200
UNION ALL
SELECT service, response_time_ms FROM perf_test WHERE status_code = 301
UNION ALL
SELECT service, response_time_ms FROM perf_test WHERE status_code = 404;
```
```python
# DSL 等价（3 次查询 + 应用层拼接）
# 每次返回 ~12.5 万行（100 万 / 8）
# 注意：应用层拼接也需要大量内存
```

> 监控指标：峰值 heap、GC 频率、是否触发 `plugins.query.memory_limit`。

#### 场景组 F：辅助验证场景（新增）

**F1. 冷启动 / 计划缓存场景**

> 测试是否存在计划缓存，以及冷热计划延迟差异。

```sql
-- 每轮用唯一注释打散任何字符串级缓存
SELECT /* cold_run_{timestamp} */ * FROM perf_test WHERE status_code = 200 LIMIT 10;
```
- 对比"冷启动"（每轮不同注释）与"热计划"（相同查询重复）的延迟差异
- 如差异显著（>2ms），说明存在计划缓存，基准测试需区分冷热场景

**F2. Pushdown on/off 对比（仅 Calcite 路径）**

> 隔离 Calcite 规划开销与下推收益。

```bash
# 分别在 pushdown=true 和 pushdown=false 下运行 E1 (UNION ALL) 场景
curl -X PUT "localhost:9200/_cluster/settings" -d '{"transient":{"plugins.calcite.pushdown.enabled":false}}'
# 运行 E1 场景
curl -X PUT "localhost:9200/_cluster/settings" -d '{"transient":{"plugins.calcite.pushdown.enabled":true}}'
# 运行 E1 场景
# 对比两者延迟差 = 下推收益
```

**F3. 结果集等价验证（所有场景前置步骤）**

> TPC 标准要求：验证 SQL 和 DSL 返回相同结果集。

```python
def verify_results_match(sql_result, dsl_result):
    """验证 SQL 和 DSL 结果集等价（行数、值、忽略顺序差异）"""
    sql_rows = sorted([tuple(row) for row in sql_result['datarows']])
    dsl_rows = sorted([tuple(hit['_source'].values()) for hit in dsl_result['hits']['hits']])
    assert len(sql_rows) == len(dsl_rows), f"行数不匹配: SQL={len(sql_rows)}, DSL={len(dsl_rows)}"
    for i, (s, d) in enumerate(zip(sql_rows, dsl_rows)):
        assert s == d, f"第 {i} 行不匹配: SQL={s}, DSL={d}"
    print("结果集验证通过")
```

### 3.2b 重查询场景设计

> 前述 3.2 场景为轻查询（返回 ≤10 行，DSL 执行 <10ms），SQL 翻译开销占比 40-800%+。
> 重查询场景通过高命中率 + 大结果集 + 高基数聚合 + 深度翻页，让 DSL 执行时间达到 100-400ms，
> 使 SQL 翻译开销占比降至 <5%，验证"重查询下 SQL 与 DSL 基本持平"的预期。

#### 设计原则

```
轻查询 → 重查询的转换公式:

命中行数:  10 → 5000-10000+    (fetch 阶段耗时)
聚合桶数:  4  → 1000+          (聚合计算耗时)
翻页深度:  0  → 19990          (排序耗时)
返回字段:  少量 → 全字段         (序列化耗时)
排序复杂度: 无 → 多字段排序       (排序耗时)
聚合指标:  1  → 4+             (多遍计算耗时)
UNION分支: 2  → 3              (合并开销)
JOIN行数:  10 → 10000+         (Hash Join 开销)
```

#### A 组（点查 → 重扫描 + 大结果集）

**A1-H. 高命中 + 大结果集**

```sql
-- SQL — 命中 ~50% 数据(500K行)，返回 10000 行
SELECT * FROM perf_test WHERE status_code = 200 ORDER BY response_time_ms DESC LIMIT 10000;
```
```json
// DSL
{"query":{"term":{"status_code":200}},"sort":[{"response_time_ms":"desc"}],"size":10000}
```
> 预期 DSL: 80-150ms（query 50ms + fetch 10K docs 50-100ms）

**A2-H. 多条件高命中 + 大结果集**

```sql
-- SQL — 多条件 OR 命中大部分数据，返回 5000 行
SELECT * FROM perf_test WHERE status_code IN (200, 301, 404) AND level IN ('INFO','WARN','ERROR') ORDER BY bytes DESC LIMIT 5000;
```
```json
// DSL
{"query":{"bool":{"filter":[{"terms":{"status_code":[200,301,404]}},{"terms":{"level":["INFO","WARN","ERROR"]}}]}},"sort":[{"bytes":"desc"}],"size":5000}
```
> 预期 DSL: 60-120ms

**A3-H. 范围扫描高命中 + 大结果集**

```sql
-- SQL — 命中 ~60% 数据，返回 10000 行
SELECT * FROM perf_test WHERE response_time_ms > 2000 ORDER BY `@timestamp` DESC LIMIT 10000;
```
```json
// DSL
{"query":{"range":{"response_time_ms":{"gt":2000}}},"sort":[{"@timestamp":"desc"}],"size":10000}
```
> 预期 DSL: 80-150ms

#### B 组（全文搜索 → 高命中 + 大结果集）

**B1-H. 高命中打分 + 大结果集**

```sql
-- SQL — match 命中大部分文档(~250K)，返回 5000 行
SELECT message, service, level, response_time_ms FROM perf_test WHERE match(message, 'request') ORDER BY response_time_ms DESC LIMIT 5000;
```
```json
// DSL
{"query":{"match":{"message":"request"}},"sort":[{"response_time_ms":"desc"}],"size":5000,"_source":["message","service","level","response_time_ms"]}
```
> 预期 DSL: 60-120ms（打分 250K docs + fetch 5000）

**B2-H. 多词打分 + filter + 大结果集**

```sql
-- SQL — 多词 match + filter，返回 10000 行
SELECT * FROM perf_test WHERE match(message, 'request failed timeout') AND status_code >= 400 ORDER BY bytes DESC LIMIT 10000;
```
```json
// DSL
{"query":{"bool":{"must":[{"match":{"message":"request failed timeout"}}],"filter":[{"range":{"status_code":{"gte":400}}}]}},"sort":[{"bytes":"desc"}],"size":10000}
```
> 预期 DSL: 80-150ms

#### C 组（聚合 → 高基数 + 多指标）

**C1-H. 高基数聚合 + 多指标**

```sql
-- SQL — user_id 10000 个桶 + 4 个指标
SELECT user_id, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, MAX(bytes) as max_bytes, MIN(response_time_ms) as min_rt
FROM perf_test
WHERE response_time_ms > 100
GROUP BY user_id
ORDER BY cnt DESC
LIMIT 1000;
```
```json
// DSL
{"size":0,"query":{"range":{"response_time_ms":{"gt":100}}},"aggs":{"by_user":{"terms":{"field":"user_id","size":1000,"order":{"cnt":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"max_bytes":{"max":{"field":"bytes"}},"min_rt":{"min":{"field":"response_time_ms"}}}}}}
```
> 预期 DSL: 100-300ms（10K 桶 + 4 指标聚合）

**C2-H. 三级聚合 + 多指标 + percentile**

```sql
-- SQL — level×service×region = 100 组合 + 4 指标 + percentile
SELECT level, service, region, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, SUM(bytes) as total_bytes
FROM perf_test
WHERE response_time_ms > 500
GROUP BY level, service, region
ORDER BY level, cnt DESC;
```
```json
// DSL — 使用 nested aggs（三级嵌套）
{"size":0,"query":{"range":{"response_time_ms":{"gt":500}}},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_region":{"terms":{"field":"region"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"total_bytes":{"sum":{"field":"bytes"}},"p95":{"percentiles":{"field":"response_time_ms","percents":[95]}}}}}}}}}}
```
> 预期 DSL: 100-250ms（三级嵌套 + percentile 计算）
> ⚠️ DSL 使用 nested aggs（树形），SQL 使用 flat GROUP BY（composite），执行路径不同，结果包含相同数据但结构不同

**C3-H. 时间直方图 + 二级聚合（仅 DSL，SQL 不支持 DATE_HISTOGRAM）**

```json
// DSL — SQL 的 DATE_HISTOGRAM NPE，此场景仅测 DSL
{"size":0,"aggs":{"by_hour":{"date_histogram":{"field":"@timestamp","calendar_interval":"1h"},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"p95":{"percentiles":{"field":"response_time_ms","percents":[95,99]}}}}}}}}
```
> 预期 DSL: 150-400ms（720 小时桶 × 5 service × percentile）

#### D 组（分页 → 深度 + 大结果集）

**D1-H. 极深度翻页**

```sql
-- SQL — 排序全部数据取最后 10 条
SELECT * FROM perf_test ORDER BY response_time_ms ASC LIMIT 19990, 10;
```
```json
// DSL
{"query":{"match_all":{}},"sort":[{"response_time_ms":"asc"}],"from":19990,"size":10}
```
> 预期 DSL: 100-200ms（排序 20000 docs）

**D2-H. 大结果集（10000 行）**

```sql
-- SQL — fetch 10000 行
SELECT * FROM perf_test WHERE status_code = 200 ORDER BY response_time_ms DESC LIMIT 10000;
```
```json
// DSL
{"query":{"term":{"status_code":200}},"sort":[{"response_time_ms":"desc"}],"size":10000}
```
> 预期 DSL: 80-150ms

**D3-H. 复合条件 + 大结果集**

```sql
-- SQL — 多条件 + 排序 + fetch 10000 行
SELECT service, level, response_time_ms, bytes, `@timestamp` FROM perf_test WHERE response_time_ms > 1000 AND status_code IN (200, 500) ORDER BY response_time_ms DESC LIMIT 10000;
```
```json
// DSL
{"query":{"bool":{"filter":[{"range":{"response_time_ms":{"gt":1000}}},{"terms":{"status_code":[200,500]}}]}},"sort":[{"response_time_ms":"desc"}],"size":10000,"_source":["service","level","response_time_ms","bytes","@timestamp"]}
```
> 预期 DSL: 80-150ms

#### E 组（SQL 独有 → 大数据量 UNION/JOIN）

**E1-H. 三路 UNION ALL + 聚合**

```sql
-- SQL — 三路聚合 UNION ALL
SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service
UNION ALL
SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service
UNION ALL
SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'DEBUG' GROUP BY service;
```
> 预期 SQL (Calcite pushdown ON): 20-50ms（3 次下推聚合 + 合并 15 行）

**E1b-H. UNION ALL 大结果集（不做聚合）**

```sql
-- SQL — ~250K 行内存合并
SELECT service, response_time_ms FROM perf_test WHERE status_code = 500
UNION ALL
SELECT service, response_time_ms FROM perf_test WHERE status_code = 503;
```
> 预期 SQL: 100-300ms（内存合并 250K 行）
> ⚠️ 监控 OOM 风险

**E2-H. JOIN 大结果集**

```sql
-- SQL — ~250K 行 Hash Join
SELECT a.service, a.level, a.response_time_ms, b.dept_name
FROM perf_test a
JOIN perf_test_meta b ON a.host = b.host
WHERE a.level = 'ERROR'
LIMIT 10000;
```
> 预期 SQL (Legacy V1): 200-500ms

**E3-H. IN 子查询大结果集**

```sql
-- SQL — 大表 IN 子查询
SELECT service, level, response_time_ms FROM perf_test
WHERE host IN (SELECT host FROM perf_test WHERE level = 'ERROR')
LIMIT 10000;
```
> 预期 SQL (Legacy V1): 200-500ms

#### 重查询预期结果矩阵

| 场景 | 预期 DSL (ms) | 预期 SQL 翻译开销 (ms) | 预期开销占比 | 设计要点 |
|------|:---:|:---:|:---:|------|
| A1-H | 80-150 | 1-3 | 1-3% | 500K 命中 + fetch 10K |
| A2-H | 60-120 | 2-5 | 2-5% | 多条件 + fetch 5K |
| A3-H | 80-150 | 1-3 | 1-3% | 600K 命中 + fetch 10K |
| B1-H | 60-120 | 1-3 | 1-4% | 250K 打分 + fetch 5K |
| B2-H | 80-150 | 2-5 | 2-5% | 多词打分 + fetch 10K |
| C1-H | 100-300 | 5-15 | 2-7% | 10K 桶 + 4 指标 |
| C2-H | 100-250 | 5-15 | 3-10% | 三级嵌套 + percentile |
| C3-H | 150-400 | — | — | DSL only |
| D1-H | 100-200 | 1-3 | 1-2% | from:19990 |
| D2-H | 80-150 | 1-3 | 1-3% | fetch 10K |
| D3-H | 80-150 | 2-5 | 2-5% | 多条件 + fetch 10K |
| E1-H | — | 20-50 | — | 3路聚合UNION |
| E1b-H | — | 100-300 | — | 250K行UNION合并 |
| E2-H | — | 200-500 | — | 大表JOIN |
| E3-H | — | 200-500 | — | 大表IN子查询 |

### 3.3 测试方法

#### 3.3.1 压测工具

```python
# benchmark.py
import requests
import time
import json
import statistics
import random

BASE_URL = "http://localhost:9200"
WARMUP_RUNS = 50      # 充分预热 JIT + Calcite codegen
TEST_RUNS = 200       # 200 轮获得稳定 p99

# 使用 Session 复用 TCP 连接
session = requests.Session()

def bench_sql(query, fetch_size=None):
    body = {"query": query}
    if fetch_size:
        body["fetch_size"] = fetch_size
    resp = session.post(f"{BASE_URL}/_plugins/_sql", json=body)
    return resp

def bench_dsl(index, dsl, request_cache=False):
    url = f"{BASE_URL}/{index}/_search"
    if not request_cache:
        url += "?request_cache=false"
    resp = session.post(url, json=dsl)
    return resp

def run_benchmark(name, func, *args, warmup=WARMUP_RUNS, runs=TEST_RUNS):
    # Warmup — 用滑动窗口 CV 判断预热是否完成
    for i in range(warmup):
        func(*args)

    # Test — 记录每次延迟用于异常值检测
    latencies = []
    for i in range(runs):
        start = time.perf_counter()
        resp = func(*args)
        elapsed_ms = (time.perf_counter() - start) * 1000
        if resp.status_code == 200:
            latencies.append(elapsed_ms)
        else:
            print(f"  ERROR (run {i}): {resp.status_code} {resp.text[:100]}")

    if latencies:
        latencies.sort()
        p50 = statistics.median(latencies)
        p99 = latencies[int(len(latencies) * 0.99)]
        p999 = latencies[int(len(latencies) * 0.999)] if len(latencies) > 100 else max(latencies)
        print(f"{name}:")
        print(f"  avg={statistics.mean(latencies):.1f}ms  p50={p50:.1f}ms  p99={p99:.1f}ms  p99.9={p999:.1f}ms  min={min(latencies):.1f}ms  max={max(latencies):.1f}ms")
        # 输出 per-run 数据用于 JIT/GC 异常值分析
        return {"latencies": latencies, "p50": p50, "p99": p99, "p999": p999}
    return {}

def run_concurrent_benchmark(name, func, args_list, concurrency=8, duration_sec=60):
    """并发吞吐量测试"""
    import concurrent.futures
    latencies = []
    stop_time = time.time() + duration_sec

    def worker():
        local_latencies = []
        while time.time() < stop_time:
            start = time.perf_counter()
            func(*random.choice(args_list))
            local_latencies.append((time.perf_counter() - start) * 1000)
        return local_latencies

    with concurrent.futures.ThreadPoolExecutor(max_workers=concurrency) as pool:
        futures = [pool.submit(worker) for _ in range(concurrency)]
        for f in concurrent.futures.as_completed(futures):
            latencies.extend(f.result())

    latencies.sort()
    qps = len(latencies) / duration_sec
    print(f"{name} (concurrency={concurrency}):")
    print(f"  QPS={qps:.1f}  p50={statistics.median(latencies):.1f}ms  p99={latencies[int(len(latencies)*0.99)]:.1f}ms")
    return {"qps": qps, "latencies": latencies}
```

#### 3.3.2 执行步骤

```
1. 启动 OpenSearch 集群（3-shard, 1-replica, JIT/GC 日志开启）
2. 创建索引（含 max_result_window: 20000）+ 导入 100 万数据
3. 轮询等待索引完成（_cat/indices 确认 docs.count=1000000）
4. 强制 merge 到 1 个 segment per shard
5. 结果集等价验证（F3 场景）— 确认 SQL 和 DSL 返回相同结果
6. 预热查询（warmup 50 轮，让 JIT C2 编译 + Calcite Janino codegen + OS cache 填充）
   - 用滑动窗口 CV 判断预热是否完成（连续 3 个窗口 CV <5% 则完成）
7. 正式测试（每个场景 200 轮，记录 per-run 延迟）
   - 聚合场景用随机化阈值打散 request cache
   - 同时运行冷启动场景 F1 对比
8. 并发吞吐量测试（8/16/32 并发客户端，持续 60 秒）
9. Pushdown on/off 对比测试（F2 场景，仅 Calcite 路径）
10. 运行完整测试套件两遍（第一遍丢弃，验证 JIT 稳定性）
11. 收集指标 + 关联 JIT/GC 日志分析异常值
12. 清理
```

#### 3.3.3 收集的指标

| 指标 | 收集方式 | 说明 |
|------|---------|------|
| **端到端延迟** | 客户端计时 | 从发送请求到收到响应的总时间 |
| **p50 / p99 / p99.9 延迟** | 客户端统计（200 轮） | 中位数和尾部延迟 |
| **per-run 延迟** | 客户端记录每次 | 用于关联 JIT 编译事件和 GC 暂停 |
| **吞吐量 (QPS)** | 并发测试统计 | 8/16/32 并发客户端持续 60 秒 |
| **服务端执行耗时** | OpenSearch slow log（`threshold.query.info: 0ms`） | **不使用** `_explain` 提取的 DSL（与实际执行 DSL 不一致） |
| **翻译开销** | `SQL端到端延迟 - 服务端执行耗时(slowlog)` | 通过 slowlog 获取实际执行时间，而非 explain |
| **SQL explain 计划** | `_plugins/_sql/_explain` | 确认走 V2 / Calcite / Legacy V1 哪条路径 + 下推情况 |
| **Calcite 回退监控** | 服务端日志 | 检查 "Fallback to V2 query engine" 日志 |
| **JIT 编译事件** | `-XX:+PrintCompilation` 输出 | 关联延迟尖峰与 JIT 重编译 |
| **JVM heap 使用** | `_nodes/stats/jvm` | 每场景前后采集 + 运行中持续采样 |
| **JVM off-heap 内存** | `_nodes/stats/jvm`（`pools.direct`） | Calcite Enumerable 可能使用 direct buffer |
| **GC 频率/耗时** | GC 日志 (`-Xlog:gc*`) | 检测 SQL 路径是否触发更多 GC |
| **分配率** | GC 日志或 async-profiler | MB/s 分配速率，比 heap 快照更敏感 |
| **线程池状态** | `_nodes/stats/thread_pool` | sql-worker vs search 线程池利用率（并发测试关键） |
| **CPU 使用率** | 系统监控 | 进程级 CPU |
| **结果集等价** | F3 验证函数 | 确认 SQL 和 DSL 返回相同结果（TPC 标准） |
| **计划稳定性** | 1000 轮 diff explain 输出 | 检测 Calcite 规则应用是否确定性 |
| **CPU 使用率** | 系统监控 | 进程级 CPU |

### 3.4 预期结果与分析框架

#### 3.4.1 预期延迟对比

| 场景 | DSL 预期 | SQL V2 预期 | SQL Calcite 预期 | SQL Legacy V1 预期 | SQL 额外开销 |
|------|---------|------------|------------------|-------------------|:---:|
| A1 点查（单条件） | 1-5ms | 3-10ms | — | — | 2-7ms |
| A2 点查（多条件） | 1-5ms | 3-10ms | — | — | 2-7ms |
| A3 范围查询 | 2-8ms | 5-13ms | — | — | 3-7ms |
| B1 简单全文搜索 | 3-10ms | 5-15ms | — | — | 2-7ms |
| B2 多字段搜索 | 3-10ms | 5-15ms | — | — | 2-7ms |
| C1 简单聚合 | 5-20ms | 8-25ms | — | — | 3-7ms |
| C2 多级聚合 | 10-40ms | 15-45ms | — | — | 5-10ms |
| C3 范围聚合 | 10-30ms | 15-35ms | — | — | 3-7ms |
| D1 排序+小分页 | 3-10ms | 5-15ms | — | — | 2-7ms |
| D2 深度分页 | 50-200ms | 55-210ms | — | — | 5-15ms |
| D3 大结果集(1000行) | 10-30ms | 15-40ms | — | — | 5-15ms |
| E1 UNION ALL | — | — | 20-50ms | — | — |
| E2 2表JOIN | — | — | — | 20-100ms | — |

#### 3.4.2 分析维度

每个场景需分析：

1. **绝对延迟差异**：`SQL延迟 - DSL延迟`
2. **相对开销比例**：`(SQL延迟 - DSL延迟) / DSL延迟 × 100%`
3. **开销占比**：`翻译规划开销 / 总延迟 × 100%`（随查询变重而下降）
4. **翻译开销分解**：`SQL端到端延迟 - 服务端slowlog执行耗时 = 翻译开销`
   > ⚠️ **不使用** `_explain` 提取的 DSL 来测执行时间——explain 输出的 DSL 与实际执行的 DSL 可能不同（Calcite 下推在运行时才生成最终 DSL）。使用 OpenSearch slow log（`index.search.slowlog.threshold.query.info: 0ms`）获取实际服务端执行时间。
5. **p99/p99.9 稳定性**：关联 per-run 延迟尖峰与 JIT 编译事件（`-XX:+PrintCompilation`）和 GC 暂停（`-Xlog:gc*`）
6. **内存影响**：`heap使用(SQL) - heap使用(DSL)` + off-heap direct buffer 对比，定义"显著"为 >5% heap 增长
7. **冷热计划对比**：F1 场景的冷启动 vs 热计划延迟差异，判断计划缓存影响
8. **下推收益**：F2 场景 pushdown on/off 延迟差，量化下推的实际价值
9. **并发性能**：不同并发度下的 QPS 和 p99，量化线程池差异影响
10. **结果集等价**：F3 验证通过/失败，记录任何语义差异

#### 3.4.3 预期结论框架

```
轻查询（搜索引擎 <10ms）:
  → SQL 解析开销占比 30-50%，DSL 更优
  → 适合用 DSL：高 QPS 点查、实时搜索

中查询（搜索引擎 10-50ms）:
  → SQL 解析开销占比 10-30%，DSL 有优势但可接受
  → 适合用 SQL：复杂分析查询、开发便利性

重查询（搜索引擎 >100ms）:
  → SQL 解析开销占比 <5%，两者基本持平
  → SQL 优势：UNION/JOIN 能力、优化器、可读性

SQL 独有能力场景:
  → UNION ALL/UNION（Calcite 引擎）：SQL 是唯一选择
  → 2表JOIN（Legacy V1 引擎）：SQL 是唯一选择，但需注意 V1 性能特征
  → IN 子查询（Legacy V1 引擎）：SQL 是唯一选择
  → 无 DSL 对照，需与"两次 DSL 查询 + 应用层合并"对比
```

### 3.5 注意事项

1. **预热必须充分**：50 轮预热让 ANTLR/Calcite classloader 初始化、JIT C2 编译完成。前 5 轮会有冷启动尖峰（classloader 加载 ~100-500ms），必须丢弃。运行完整测试两遍验证 JIT 稳定性。

2. **单线程串行测试**：先做单线程对比消除并发干扰，再补充并发压测。

3. **JVM GC 影响**：SQL 路径生成更多临时对象（AST/Plan），可能触发更频繁的 GC。通过 per-run 延迟和 GC 日志监控。

4. **Segment merge**：测试前必须 `forcemerge?max_num_segments=1`，否则 segment 数量影响查询性能。

5. **OS cache**：确保操作系统 page cache 已 warm（预热查询会覆盖）。

6. **DSL 使用 filter context**：等值查询用 `bool.filter`（不打分），与 SQL 的 `WHERE` 语义对等。已通过 `_explain` 验证 SQL V2 对 keyword 字段生成 `bool.filter`。

7. **SQL 路径确认**：通过 `_plugins/_sql/_explain` 确认每个 SQL 查询走的是 V2、Calcite 还是 Legacy V1 路径：
   - V2 输出格式: `{"root": {"name": "ProjectOperator", ...}}`
   - Calcite 输出格式: `{"calcite": {"logical": "LogicalProject(...)", ...}}`
   - Legacy V1 输出格式: `{"Physical Plan": {...}, "Logical Plan": {...}}`

8. **Calcite 回退监控**：检查服务端日志中 "Fallback to V2 query engine since got exception" 消息。如发生回退，该轮结果需标记或丢弃。

9. **连接复用**：使用 `requests.Session()` 消除 TCP 连接建立开销（~0.5-1ms/次）。

10. **数据分布**：确保测试数据基数合理（如 `level` 有 4 个值，`service` 有 5 个值），避免数据倾斜。

11. **`@timestamp` 标识符**：以 `@` 开头的字段名在 SQL 中可能需要反引号引用（如 `` `@timestamp` ``）。

### 3.6 完整测试执行脚本框架

```bash
#!/bin/bash
# run_benchmark.sh

set -e

ES="http://localhost:9200"

echo "=== 1. Setup ==="
# 创建索引（含 max_result_window）
curl -s -X PUT "$ES/perf_test" -H 'Content-Type: application/json' -d '@index_mapping.json'
# 导入数据（假设已生成 bulk_data.json）
echo "Loading 1M documents..."
curl -s -X POST "$ES/_bulk?refresh=false" -H 'Content-Type: application/x-ndjson' --data-binary @bulk_data.json > /dev/null

# 轮询等待索引完成
echo "Waiting for indexing..."
while [ "$(curl -s "$ES/_cat/indices/perf_test?h=docs.count" | tr -d ' ')" -lt "1000000" ]; do
  echo "  $(curl -s "$ES/_cat/indices/perf_test?h=docs.count" | tr -d ' ')/1000000"
  sleep 5
done

echo "Forcemerging..."
curl -s -X POST "$ES/perf_test/_forcemerge?max_num_segments=1" > /dev/null
echo "Data ready: $(curl -s "$ES/perf_test/_count" | python3 -c 'import json,sys; print(json.load(sys.stdin)["count"])') docs"

echo "=== 2. Enable Calcite ==="
curl -s -X PUT "$ES/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": {"plugins.calcite.enabled": "true", "plugins.calcite.pushdown.enabled": "true"}
}' > /dev/null

echo "=== 3. Run Benchmarks (first pass - discard) ==="
python3 benchmark.py --tag pass1

echo "=== 4. Run Benchmarks (second pass - keep) ==="
python3 benchmark.py --tag pass2

echo "=== 5. Result Equivalence Audit ==="
python3 verify_results.py

echo "=== 6. Cleanup ==="
# curl -X DELETE "$ES/perf_test"
```

### 3.7 架构风险与战略建议

> 基于 Oracle 大数据查询引擎专家审视，以下风险和建议补充到基准测试的分析框架中。

#### 3.7.1 架构风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| **三引擎共存** | 语义漂移、路由复杂度增长、测试矩阵爆炸、性能不可预测 | 定义目标状态和截止日期（如"4.0 全走 Calcite，Legacy 移除"）；新功能只做 Calcite |
| **Calcite 单节点内存瓶颈** | UNION/JOIN 的 Enumerable 算子在协调节点单线程执行，无分布式并行。100M 数据上会 OOM | E4 场景测量内存上限；文档化扩展边界；长期需 exchange/shuffle 机制 |
| **无计划缓存** | 每次查询都重新解析+规划，高 QPS 下翻译开销累积 | 长期实现 PreparedStatement 缓存（Presto/Spark 有 30-50% 延迟优化） |
| **无统计信息注入** | Calcite CBO 退化为规则系统，JOIN 计划选择随机 | 优先实现统计注入（10-15 人天），ROI 高于任何单一功能扩展 |
| **游标不兼容** | V2 序列化游标 vs Calcite 无序列化游标，迁移破坏现有客户端 | 游标版本化 + 回退机制 |

#### 3.7.2 SQL vs DSL 决策矩阵

> 测试完成后，应产出以下决策矩阵供团队参考：

| 查询形态 | QPS 层级 | 延迟预算 | 推荐 | 理由 |
|---------|---------|---------|------|------|
| 点查 (SELECT+WHERE) | 高 (>1000) | <10ms | DSL | 翻译开销占比高 |
| 点查 (SELECT+WHERE) | 低 (<100) | <100ms | 均可 | 翻译开销可忽略 |
| 聚合 (GROUP BY) | 中 | <200ms | 均可 | 翻译开销占比低 |
| 全文搜索 | 高 | <10ms | DSL | SQL match() 功能受限 |
| 全文搜索 | 低 | <100ms | DSL | 表达力更完整 |
| JOIN | 任意 | 任意 | SQL | DSL 不支持 |
| UNION | 任意 | 任意 | SQL | DSL 不支持 |
| 复杂嵌套聚合 | 任意 | 任意 | DSL | SQL 表达力不足 |
| BI 工具接入 | 任意 | 任意 | SQL | JDBC/ODBC 兼容 |
| 跨数据源查询 | 任意 | 任意 | SQL | DSL 只能查本地索引 |

#### 3.7.3 与行业最佳实践对比

| 维度 | 本方案 | TPC-H/TPC-DS | JMH | Presto/Spark |
|------|--------|-------------|-----|-------------|
| 数据生成 | 自定义随机数据 | dbgen 规范化生成 | N/A | TPC 数据 |
| 查询审计 | F3 结果等价验证 ✅ | 强制要求 | N/A | 内置 |
| 规模扩展 | 1M（可扩展到 100M） | Scale Factor (SF) | N/A | 多 SF |
| 功率测试 | 单线程 p50/p99 ✅ | Power test | Single shot | ✅ |
| 吞吐测试 | 并发 QPS ✅ | Throughput test | N/A | ✅ |
| JVM 隔离 | 两遍运行 | N/A | Fork per benchmark | N/A |
| JIT 控制 | PrintCompilation ✅ | N/A | `-XX:-TieredCompilation` | N/A |
| 分配 profiling | GC 日志 ✅ | N/A | `-prof gc` | N/A |

> **差距**：缺少 TPC 标准的规范化数据生成和 Scale Factor 扩展。JMH 级别的 JVM 隔离（fork per benchmark）未实现，但两遍运行 + JIT 日志可部分替代。

---

## 附录：SQL Explain 路径验证

每个 SQL 查询在测试前需通过 explain 确认执行路径：

```bash
# 确认走 V2、Calcite 还是 Legacy V1
curl -X POST "$ES/_plugins/_sql/_explain" -d '{"query": "SELECT ..."}'

# V2 输出: {"root": {"name": "ProjectOperator", ...}}
# Calcite 输出: {"calcite": {"logical": "LogicalProject(...)", ...}}
# Legacy V1 输出: {"Physical Plan": {...}, "Logical Plan": {...}}
```

确认下推情况（Calcite 路径）：

```
CalciteEnumerableIndexScan → 已下推到 OpenSearch（最优）
EnumerableCalc → 内存中计算（有额外开销）
EnumerableAggregate → 内存中聚合（有额外开销）
EnumerableUnion → 内存中 UNION 合并
```

### 服务端耗时分解方法

```bash
# 1. 获取 SQL explain，提取生成的 DSL
SQL_EXPLAIN=$(curl -s -X POST "$ES/_plugins/_sql/_explain" -d '{"query": "SELECT ..."}')

# 2. 从 explain 输出中提取 sourceBuilder 内容（即生成的 DSL）

# 3. 用 profile:true 执行提取的 DSL
curl -s -X POST "$ES/perf_test/_search" -d '{
  "profile": true,
  "query": {"term": {"status_code": 200}},
  "size": 10
}'

# 4. 从 profile 结果提取服务端执行时间
# 5. SQL 翻译开销 = SQL 端到端延迟 - 服务端 DSL 执行时间
```

---

## 第四部分：SQL 插件能力扩展评估

> 基于扩展 UNION/UNION ALL 支持的实战经验，评估当前 SQL 插件不支持的场景各自需要多大的工作量。
> 所有"现状"均通过 `POST /_plugins/_sql` 实测验证（2026-07-18）。

### 4.1 评估框架

每个特性的工作量按以下维度评分：

| 维度 | 分值 | 说明 |
|------|:----:|------|
| **文法改动** | S/M/L | ANTLR `.g4` 文件修改量 |
| **AST 节点** | S/M/L | 是否需要新增 AST 节点类 |
| **AstBuilder** | S/M/L | 解析器 visitor 改动量 |
| **Analyzer** | S/M/L | 语义分析改动量（V2 路径） |
| **Calcite 集成** | S/M/L | CalciteRelNodeVisitor 改动量（如走 Calcite） |
| **Legacy 兼容** | S/M/L | 是否影响 Legacy 回退路径 |
| **测试** | S/M/L | 单元测试 + 集成测试量 |

**S** = 小（改几行）｜ **M** = 中（改几十行，新增 1-2 个类）｜ **L** = 大（改上百行，新增多个类，涉及架构调整）

### 4.2 总览

| 特性 | 文法 | AST | AstBuilder | Analyzer | Calcite | Legacy | 测试 | 总工作量 | 人天 |
|------|:----:|:---:|:----------:|:--------:|:-------:|:------:|:----:|:--------:|:----:|
| **COALESCE** | S | S | S | M | — | — | S | S | 1-2 |
| **DATE_HISTOGRAM** | M | M | M | M | M | — | M | M | 3-5 |
| **EXISTS 子查询** | M | S | M | L | L | M | M | M | 3-5 |
| **3 表 JOIN** | S | S | S | — | S | L | M | L | 5-8 |
| **JOIN + GROUP BY** | S | S | — | L | M | L | M | L | 5-8 |
| **JOIN + 聚合函数** | — | — | — | L | M | L | M | L | 5-8 |
| **子查询+外层 GROUP BY** | M | S | M | L | M | L | M | L | 5-8 |
| **标量子查询 (SELECT 中)** | M | M | L | L | L | L | L | L | 8-12 |
| **CTE (WITH)** | L | M | L | L | L | L | L | L | 10-15 |

### 4.3 逐项详细分析

#### 4.3.1 COALESCE 函数 — ⭐ 最简单

**现状**：报错 `The following method is not supported in Schema: COALESCE`

**根因**：V2 的函数注册表（`BuiltinFunctionRepository`）没有注册 COALESCE 函数。`COALESCE` 是 SQL 标准函数，底层可映射到 Calcite 的 `COALESCE` SqlOperator。

**改动点**：
- `core/.../expression/function/BuiltinFunctionName.java` — 新增 `COALESCE` 枚举
- `core/.../expression/function/ImplementationResolved` — 注册 COALESCE 实现
- Calcite 已有 `SqlLibraryOperators.COALESCE`，直接复用
- 如果走 V2 路径，需实现 `FunctionImplementation` 做空值合并

**工作量：1-2 人天**

```
文法:       S (COALESCE 关键字已在词法器中)
AST:        S (复用 Function 节点)
AstBuilder: S (复用 visitFunction)
Analyzer:   M (注册函数 + 类型推导)
Calcite:    — (Calcite 原生支持)
Legacy:     — (Legacy 已支持 COALESCE)
测试:       S (单元测试 + IT)
```

---

#### 4.3.2 DATE_HISTOGRAM / HISTOGRAM 函数 — 中等

**现状**：NPE `Cannot invoke "Object.toString()" because "this.value" is null`

**根因**：V2 的 `HISTOGRAM` 函数解析有 bug——`INTERVAL` 参数的值在 AST 构建时为 null。词法器有 `DATE_HISTOGRAM` 和 `HISTOGRAM` token，但 V2 parser 没有对应的函数规则，走了通用函数解析路径导致 NPE。

**改动点**：
- 文法：确认 `HISTOGRAM(field, INTERVAL n UNIT)` 语法在 V2 parser 中的支持情况
- `AstBuilder` — 修复 INTERVAL 参数解析，确保 `IntervalValue` 正确构建
- `Analyzer` — 注册 HISTOGRAM 为聚合函数
- `CalciteRelNodeVisitor` — 映射到 Calcite 的 `LogicalAggregate` + date histogram
- 或映射到 OpenSearch 的 `date_histogram` 聚合（下推）

**工作量：3-5 人天**

```
文法:       M (确认/修复 HISTOGRAM + INTERVAL 语法规则)
AST:        M (确保 IntervalValue 正确构建)
AstBuilder: M (修复 INTERVAL 参数解析 NPE)
Analyzer:   M (注册 HISTOGRAM 聚合函数)
Calcite:    M (映射到 date_histogram 聚合下推)
Legacy:     — (Legacy 已支持 DATE_HISTOGRAM)
测试:       M (各种 INTERVAL: MINUTE/HOUR/DAY/MONTH IT)
```

---

#### 4.3.3 EXISTS 子查询 — 中等

**现状**：报错 `Unsupported subquery`

**根因**：V2 `AstBuilder` 没有 `visitExistsSubquery` 方法。`SqlV2QueryParser`（Unified Query API）的 `ExtendedAstExpressionBuilder` 已实现此方法，但仅对 composite 索引生效。

**改动点**：
- 文法：`OpenSearchSQLParser.g4` 已有 `existsSubqueryExpression` 规则（词法器有 EXISTS token）
- `AstBuilder` — 需要像 `SqlV2QueryParser.ExtendedAstExpressionBuilder` 那样实现 `visitExistsSubqueryExpression`，构建 `ExistsSubquery` AST 节点
- `Analyzer` — 需要实现 `visitExistsSubquery` 语义分析，将子查询转换为 SemiJoin
- `CalciteRelNodeVisitor` — Calcite 原生支持 SemiJoin，需映射 AST 到 RelNode
- 或选择回退到 Legacy V1（Legacy 已支持 EXISTS）

**两条路径**：

| 路径 | 改动 | 风险 |
|------|------|------|
| **A. 回退 Legacy V1**（最快） | 在 `AstBuilder` 中抛 `SyntaxCheckException` 触发回退 | 低，但 Legacy 的 EXISTS 实现可能不完整 |
| **B. 走 Calcite**（完整） | 实现 AstBuilder + Analyzer + CalciteRelNodeVisitor | 中，需要处理 SemiJoin 下推 |

**工作量：3-5 人天（路径 B）**

```
文法:       M (已有规则，可能需微调)
AST:        S (复用 ExistsSubquery 节点，已存在于 core/ast/expression/)
AstBuilder: M (实现 visitExistsSubqueryExpression)
Analyzer:   L (新增 visitExistsSubquery，SemiJoin 语义分析)
Calcite:    L (SemiJoin RelNode 映射 + 下推优化)
Legacy:     M (如走回退路径，确保 Legacy 能处理)
测试:       M (单元测试 + IT)
```

---

#### 4.3.4 3 表及以上的 JOIN — 大

**现状**：报错 `currently supports only 2 tables join`

**根因**：Legacy V1 的 `QueryAction` 只支持 2 表 JOIN。V2 的 `AstBuilder.visitJoinClause()` 直接抛异常回退到 Legacy。`SqlV2QueryParser` 支持但仅限 composite 索引。

**改动点**：

| 路径 | 改动 |
|------|------|
| **A. 走 Calcite**（推荐） | 在 `AstBuilder.visitJoinClause()` 中构建 `Join` AST 节点（而非抛异常），在 `QueryService.shouldUseCalcite()` 中检测 JOIN 节点路由到 Calcite，Calcite 原生支持 N-way JOIN |
| **B. 扩展 Legacy V1** | 需要修改 Druid 解析器的 `OpenSearchActionFactory`，改动量大且 Legacy 是要淘汰的引擎 |

**路径 A 详细**：
- `AstBuilder.visitJoinClause()` — 改为构建 `Join` 节点（参考 `SqlV2QueryParser.ExtendedAstBuilder` 的实现）
- `QueryService.shouldUseCalcite()` — 新增 `containsJoin(plan)` 检测，类似 `containsUnion`
- `CalciteRelNodeVisitor.visitJoin()` — 已有实现（PPL JOIN 走这里），确认支持 N-way
- `CanPaginateVisitor` — 新增 `visitJoin` 返回 false（或 true，取决于是否可分页）

**工作量：5-8 人天**

```
文法:       S (已有 JOIN 规则，支持链式 JOIN)
AST:        S (复用 Join 节点)
AstBuilder: S (改为构建 Join 节点，不抛异常)
Analyzer:   — (Analyzer.visitJoin 已有，但可能需调整)
Calcite:    S (CalciteRelNodeVisitor.visitJoin 已实现)
Legacy:     L (如不回退 Legacy，不影响；如回退，需扩展)
CanPaginate: S (新增 visitJoin)
shouldUseCalcite: S (新增 containsJoin)
测试:       M (2表/3表/4表 JOIN IT)
```

> ⚠️ 与 UNION 扩展模式高度相似——都是"AstBuilder 不抛异常 + shouldUseCalcite 路由 + Calcite 已有实现"。可复用 UNION 的扩展模式。

---

#### 4.3.5 JOIN + GROUP BY / 聚合函数 — 大

**现状**：报错 `JOIN queries do not support aggregations on the joined result`

**根因**：Legacy V1 的 `JoinQueryBuilder` 不支持在 JOIN 结果上做聚合。如果走 Calcite 路径（路径 A），Calcite 原生支持 JOIN + GROUP BY。

**前提**：必须先完成"3 表 JOIN"扩展（即让 JOIN 走 Calcite 而非 Legacy）。

**改动点**：
- 如果 JOIN 已走 Calcite（上一项扩展完成），则 JOIN + GROUP BY 自动支持——Calcite 的 `LogicalAggregate` 可以建立在 `LogicalJoin` 之上
- 需要验证 `CalciteRelNodeVisitor` 的 `visitAggregation` 是否能正确处理 JOIN 后的 schema
- 可能需要处理 JOIN 后的字段名冲突（`a.name` vs `b.name`）

**工作量：5-8 人天**（前提：JOIN 扩展已完成）

```
文法:       S (GROUP BY 语法已有)
AST:        S (Aggregation 节点已有)
AstBuilder: — (无额外改动)
Analyzer:   L (需要处理 JOIN 后的 Aggregation 语义，字段解析)
Calcite:    M (验证 LogicalAggregate on LogicalJoin，处理字段名)
Legacy:     L (如不走 Legacy，不影响)
测试:       M (JOIN + GROUP BY / SUM / COUNT / AVG IT)
```

---

#### 4.3.6 子查询 + 外层 GROUP BY — 大

**现状**：报错 `SQLSubqueryTableSource cannot be cast to SQLJoinTableSource`（Druid 类型转换异常）

**根因**：Legacy V1 的 Druid 解析器无法处理 `(SELECT ... JOIN ...) AS t GROUP BY ...` 这种嵌套结构。

**改动点**：
- 如果 JOIN 已走 Calcite，子查询作为派生表 `(SELECT ...) AS t` 在 V2 中已支持
- 需要确保派生表内含 JOIN 时，AstBuilder 能正确构建嵌套 AST
- `Analyzer` 需要处理派生表的 schema 传播到外层 GROUP BY

**工作量：5-8 人天**（前提：JOIN 扩展已完成）

```
文法:       M (确认 subqueryAsRelation 规则支持嵌套 JOIN)
AST:        S (复用 RelationSubquery 节点)
AstBuilder: M (处理派生表内的 JOIN)
Analyzer:   L (派生表 schema 传播 + 外层 GROUP BY 语义)
Calcite:    M (Calcite 原生支持，验证 RelNode 构建)
Legacy:     L (Druid 不支持，必须走 Calcite)
测试:       M (各种子查询 + GROUP BY 组合 IT)
```

---

#### 4.3.7 标量子查询（SELECT 中的子查询）— 大

**现状**：报错 `unknown field name : ...SQLSelect@...`（Druid 无法解析 SELECT 中的子查询）

**根因**：V2 的 `AstExpressionBuilder` 没有 `visitScalarSubquery` 方法。Calcite 原生支持标量子查询（`RexSubquery`）。

**改动点**：
- 文法：确认 `OpenSearchSQLParser.g4` 是否有 `scalarSubquery` 规则
- `AstExpressionBuilder` — 新增 `visitScalarSubquery`，构建子查询 AST
- `Analyzer` / `ExpressionAnalyzer` — 新增标量子查询的语义分析和类型推导
- `CalciteRelNodeVisitor` — 将标量子查询映射到 Calcite 的 `RexSubquery.SCALAR`
- 这是最复杂的扩展，因为涉及表达式层面的子查询（不是语句层面）

**工作量：8-12 人天**

```
文法:       M (可能需要新增 scalarSubquery 表达式规则)
AST:        M (可能需要新增 ScalarSubquery 表达式节点)
AstBuilder: L (表达式层面的子查询构建，比语句层面复杂)
Analyzer:   L (标量子查询类型推导 + 相关性检查)
Calcite:    L (RexSubquery 映射 + 解关联子查询)
Legacy:     L (Druid 不支持，必须走 Calcite)
测试:       L (标量子查询的各种场景：相关/非相关/嵌套)
```

---

#### 4.3.8 CTE (WITH) — 最大

**现状**：报错 `Query must start with SELECT, DELETE, SHOW or DESCRIBE`

**根因**：`OpenSearchSQLParser.g4` 的 `dmlStatement` 规则没有 WITH 选项。ANTLR 词法器有 WITH token 但解析器不消费它。

**改动点**：
- 文法：新增 `withClause` 规则，修改 `dmlStatement` 允许 `WITH ... SELECT ...`
- AST：新增 `WithClause` / `CommonTableExpression` 节点
- `AstBuilder` — 新增 `visitWithClause`，将 CTE 注册为命名派生表
- `Analyzer` — 处理 CTE 的 schema 注册和引用解析
- `CalciteRelNodeVisitor` — Calcite 原生支持 CTE（`RelBuilder.withRenamedInputs`）
- 或简单实现：将 CTE 展开为内联派生表（语法糖展开）

**两条路径**：

| 路径 | 改动 | 复杂度 |
|------|------|--------|
| **A. CTE 展开为派生表**（简单实现） | 文法 + AstBuilder 中将 CTE 引用替换为内联子查询 | 中，不需要 Calcite 特殊支持 |
| **B. Calcite 原生 CTE**（完整实现） | 文法 + AST + Analyzer + Calcite RelBuilder | 高，但 Calcite 有 CTE 优化 |

**工作量：10-15 人天**

```
文法:       L (新增 withClause + CTE 引用规则)
AST:        M (新增 WithClause / CommonTableExpression 节点)
AstBuilder: L (visitWithClause + CTE 引用解析)
Analyzer:   L (CTE schema 注册 + 作用域管理)
Calcite:    L (路径 B: RelBuilder CTE 支持; 路径 A: 展开为派生表)
Legacy:     L (Druid 有 CTE 语法但实现不完整)
测试:       L (单 CTE / 多 CTE / 嵌套 CTE / 递归 CTE)
```

### 4.4 扩展优先级建议

按**性价比**（价值/工作量比）排序：

| 优先级 | 特性 | 人天 | 理由 |
|:------:|------|:----:|------|
| **P0** | COALESCE | 1-2 | 最简单，Calcite 原生支持，只需注册函数 |
| **P1** | DATE_HISTOGRAM | 3-5 | 修复 bug 而非新功能，时序分析基础能力 |
| **P2** | 3表+ JOIN（走 Calcite） | 5-8 | 与 UNION 扩展模式高度相似，可复用模式 |
| **P2** | EXISTS 子查询 | 3-5 | 中等复杂度，SqlV2QueryParser 已有实现可参考 |
| **P3** | JOIN + GROUP BY | 5-8 | 依赖 P2 JOIN 扩展完成后自动获得 |
| **P3** | 子查询 + 外层 GROUP BY | 5-8 | 依赖 P2 JOIN 扩展完成后自动获得 |
| **P4** | 标量子查询 | 8-12 | 表达式层面子查询，复杂度高 |
| **P5** | CTE | 10-15 | 工作量最大，可用派生表替代 |

### 4.5 总工作量估算

| 范围 | 人天 |
|------|:----:|
| 全部完成 | 42-63 |
| P0+P1（快速收益） | 4-7 |
| P0-P2（核心能力） | 9-15 |
| P0-P3（覆盖大部分场景） | 19-31 |
| P0-P5（全部） | 42-63 |

### 4.6 扩展模式分类

UNION 扩展的"三步模式"（AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 实现）仅适用于**计划级路由**的扩展。并非所有特性都适用此模式：

| 扩展模式 | 适用特性 | 说明 |
|---------|---------|------|
| **计划级路由（三步模式）** | JOIN、EXISTS 子查询、3表+ JOIN | AstBuilder 构建节点 → shouldUseCalcite 检测 → CalciteRelNodeVisitor 已有实现 |
| **函数注册** | COALESCE | 在 `BuiltinFunctionRepository` 注册函数，映射到 Calcite `SqlOperator` |
| **聚合函数修复** | DATE_HISTOGRAM | 修复 INTERVAL 参数解析 NPE + 注册聚合函数 + 映射下推 |
| **表达式级子查询** | 标量子查询 | 需 `AstExpressionBuilder` 新增方法 + Calcite `RexSubquery` 映射 + 子查询去关联 |
| **语句级文法** | CTE | 新增文法规则 + AST 节点 + 作用域管理（或展开为派生表） |

### 4.7 统计信息注入（最高杠杆独立工作流）

> Oracle 专家审视发现：Calcite 优化器在无统计信息时退化为规则系统，JOIN 计划选择实际是随机的。

**现状**：OpenSearch 有丰富的 per-shard 统计（doc_count、字段基数、min/max），但未注入 Calcite 的 `RelMetadataProvider`。

**改动点**：
- 在 `CalcitePlanContext.create()` 中注册自定义 `RelMetadataProvider`
- 实现 `TableStats` / `RowCount` / `Selectivity` 元数据提供者
- 从 OpenSearch `_stats` API 获取统计信息注入

**工作量：10-15 人天**

**ROI**：高于任何单一功能扩展。无统计时 Calcite CBO 实际是规则系统；有统计后 JOIN 计划选择有 2-10x 性能差异。

### 4.8 依赖关系

```
COALESCE ──────────────────────────────── 独立（函数注册模式）
DATE_HISTOGRAM ────────────────────────── 独立（聚合函数修复模式）
统计信息注入 ──────────────────────────── 独立（最高 ROI，建议优先）
EXISTS 子查询 ─────────────────────────── 独立（计划级路由模式）
3表+ JOIN ─────────────────────────────── 独立（计划级路由模式）
  ├── JOIN + GROUP BY ─────────────────── 依赖 3表+ JOIN
  ├── JOIN + 聚合函数 ─────────────────── 依赖 3表+ JOIN
  └── 子查询 + 外层 GROUP BY ──────────── 依赖 3表+ JOIN
标量子查询 ────────────────────────────── 独立（表达式级，复杂度最高）
CTE ───────────────────────────────────── 独立（语句级，可用派生表替代）
```

### 4.9 修订后的优先级

| 优先级 | 特性 | 人天 | 模式 | 理由 |
|:------:|------|:----:|------|------|
| **P0** | 统计信息注入 | 10-15 | 独立工作流 | 最高 ROI，让 Calcite CBO 真正生效 |
| **P1** | COALESCE | 1-2 | 函数注册 | 最简单，Calcite 原生支持 |
| **P1** | DATE_HISTOGRAM | 3-5 | 聚合修复 | 修复 bug，时序分析基础能力 |
| **P2** | 3表+ JOIN | 5-8 | 计划级路由 | 可复用 UNION 三步模式 |
| **P2** | EXISTS 子查询 | 3-5 | 计划级路由 | SqlV2QueryParser 已有实现可参考 |
| **P3** | JOIN + GROUP BY | 5-8 | 依赖 P2 | JOIN 走 Calcite 后自动获得 |
| **P3** | 子查询 + 外层 GROUP BY | 5-8 | 依赖 P2 | 同上 |
| **P4** | 标量子查询 | 8-12 | 表达式级 | 复杂度最高 |
| **P5** | CTE | 10-15 | 语句级 | 可用派生表替代 |

---

## 第五部分：10M 多节点生产级验证方案

> 前述测试在单节点 512MB heap + 1M 数据 + forcemerge 环境下进行，DSL 异常快（6-60ms），
> 导致 SQL 开销占比即使重查询也偏高（100-260%）。
> 本部分设计 10M 数据 + 多节点集群的验证方案，目标是验证：
> **"在大数据量重查询场景下，SQL 接口与 DSL 接口性能差异不大，SQL 额外开销在固定范围内（<10%）。"**

### 5.1 测试环境

#### 5.1.1 集群拓扑

```
3 节点集群（模拟生产最小可用配置）:
  node-1: cluster_manager + data (8C 16G)
  node-2: data (8C 16G)
  node-3: data (8C 16G)

每节点 JVM Heap: 8GB
每节点磁盘: SSD 200GB
网络: 万兆局域网（模拟生产网络延迟 ~0.1-0.5ms）
```

#### 5.1.2 索引设计

```json
{
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "service": { "type": "keyword" },
      "host": { "type": "keyword" },
      "message": { "type": "text" },
      "status_code": { "type": "integer" },
      "response_time_ms": { "type": "integer" },
      "bytes": { "type": "long" },
      "user_id": { "type": "keyword" },
      "region": { "type": "keyword" },
      "session_id": { "type": "keyword" },
      "request_path": { "type": "keyword" },
      "client_ip": { "type": "ip" }
    }
  },
  "settings": {
    "number_of_shards": 6,
    "number_of_replicas": 1,
    "index.refresh_interval": "30s",
    "index.max_result_window": 20000
  }
}
```

| 配置项 | 值 | 理由 |
|--------|-----|------|
| `number_of_shards` | 6 | 每节点 2 shard，3 节点并行查询 |
| `number_of_replicas` | 1 | 生产标准配置，高可用 |
| `max_result_window` | 20000 | 深度分页需要 |
| 字段数 | 13 | 模拟生产日志索引宽表 |

#### 5.1.3 数据量

| 指标 | 值 |
|------|-----|
| 文档总数 | 10,000,000 |
| 每文档大小 | ~500 bytes (JSON) |
| 总数据量 | ~5GB |
| 每 shard 数据量 | ~830MB |
| 高基数字段 | `user_id`(100K)、`session_id`(500K)、`client_ip`(100K) |
| 低基数字段 | `level`(4)、`service`(5)、`host`(5)、`region`(5) |

#### 5.1.4 数据生成

```python
# generate_10m.py
import json, random, time

levels = ["INFO","WARN","ERROR","DEBUG"]
services = ["auth-service","payment-service","order-service","search-service","notification-service"]
hosts = [f"host-{i:02d}" for i in range(1, 21)]  # 20 hosts
regions = ["us-east-1","us-west-2","eu-west-1","ap-southeast-1","ap-northeast-1"]
messages = [
    "Request processed successfully","Connection timeout to database",
    "User authentication failed","Cache miss for key","Rate limit exceeded",
    "Background job completed","Configuration reloaded","Health check passed",
    "SSL certificate renewal required","Disk usage above threshold",
    "Memory pressure detected","Thread pool exhausted","Circuit breaker opened",
    "Latency spike detected","Downstream service unavailable"
]
request_paths = ["/api/v1/auth","/api/v1/orders","/api/v2/search","/api/v1/payment","/api/v1/users",
                 "/api/v1/health","/api/v1/metrics","/api/v2/reports","/api/v1/config","/api/v1/logout"]
status_codes = [200,200,200,200,200,200,301,302,400,401,403,404,404,500,500,502,503]

batch_size = 100000
total = 10000000
batches = total // batch_size

for batch_num in range(batches):
    lines = []
    for i in range(batch_num * batch_size, (batch_num + 1) * batch_size):
        ts = int(time.time()) - random.randint(0, 30*24*3600)  # 最近30天
        doc = {
            "@timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime(ts)),
            "level": random.choice(levels),
            "service": random.choice(services),
            "host": random.choice(hosts),
            "message": random.choice(messages),
            "status_code": random.choice(status_codes),
            "response_time_ms": random.randint(1, 10000),
            "bytes": random.randint(100, 500000),
            "user_id": f"user-{random.randint(1, 100000)}",
            "region": random.choice(regions),
            "session_id": f"sess-{random.randint(1, 500000)}",
            "request_path": random.choice(request_paths),
            "client_ip": f"10.{random.randint(0,255)}.{random.randint(0,255)}.{random.randint(1,254)}"
        }
        lines.append(json.dumps({"index": {"_index": "perf_test_10m"}}))
        lines.append(json.dumps(doc))
    with open(f"batch_{batch_num:03d}.json", "w") as f:
        f.write("\n".join(lines) + "\n")
    if batch_num % 10 == 0:
        print(f"Generated {batch_num * batch_size} / {total}")
print(f"Done: {total} docs in {batches} batches")
```

#### 5.1.5 软件配置

```
OpenSearch: 3.7.0 (3 节点集群)
SQL Plugin: 3.7.0.0-SNAPSHOT (含 UNION/UNION ALL 扩展)
plugins.calcite.enabled: true
plugins.calcite.pushdown.enabled: true
plugins.query.size_limit: 100000
plugins.query.memory_limit: 80%
plugins.sql.slowlog: 0  # 记录所有查询的服务端执行时间

# 索引级
index.max_result_window: 20000
index.requests.cache.enable: false  # 禁用 request cache 确保公平
```

#### 5.1.6 预期 DSL 延迟范围

| 场景 | 单节点 1M (实测) | 多节点 10M (预期) | 变化因素 |
|------|:---:|:---:|------|
| 点查 10 行 | 0.8ms | 5-15ms | 数据量 10x + 网络 |
| 大结果集 10K 行 | 59ms | 200-500ms | 数据量 10x + fetch 跨节点 |
| 高基数聚合 | 18ms | 200-600ms | 100K 桶 + 跨 shard 聚合 |
| 三级聚合 | 39ms | 300-800ms | 跨 shard 三级归并 |
| 深度翻页 | 6ms | 300-600ms | 跨 shard 排序 20000 行 |
| UNION+聚合 | 14ms | 50-150ms | 3 次跨 shard 聚合 |

> **关键**：多节点 10M 下 DSL 执行时间预期 50-800ms，SQL 翻译开销固定在 5-80ms，
> 开销占比预期降至 1-15%，可验证"差异不大"的结论。

### 5.2 测试场景

> 从 3.2b 重查询场景中选取有 DSL 对照的场景，确保每场景 DSL 预期 >100ms。
> 额外增加生产典型场景（时间范围过滤 + 聚合、多字段排序）。

#### 场景组 G：生产典型重查询

**G1. 时间范围 + 聚合（日志分析最常见查询）**

```sql
-- SQL — 最近 7 天数据（~2.3M docs）按 service 聚合
SELECT service, level, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, PERCENTILE(response_time_ms, 95) as p95
FROM perf_test_10m
WHERE @timestamp > NOW() - INTERVAL 7 DAY
GROUP BY service, level
ORDER BY service, cnt DESC;
```
```json
// DSL
{"size":0,"query":{"range":{"@timestamp":{"gte":"now-7d"}}},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"p95":{"percentiles":{"field":"response_time_ms","percents":[95]}}}}}}}}
```
> 预期 DSL: 300-600ms（2.3M docs 扫描 + 聚合 + percentile）
> 预期 SQL 开销: 5-15ms
> 预期开销占比: 1-5%

**G2. 多字段排序 + 大结果集**

```sql
-- SQL — ERROR 日志按多字段排序取 10000
SELECT * FROM perf_test_10m
WHERE level = 'ERROR' AND @timestamp > NOW() - INTERVAL 1 DAY
ORDER BY response_time_ms DESC, bytes DESC
LIMIT 10000;
```
```json
// DSL
{"query":{"bool":{"filter":[{"term":{"level":"ERROR"}},{"range":{"@timestamp":{"gte":"now-1d"}}}]}},"sort":[{"response_time_ms":"desc"},{"bytes":"desc"}],"size":10000}
```
> 预期 DSL: 200-500ms（~330K docs 排序 + fetch 10K）
> 预期 SQL 开销: 5-15ms
> 预期开销占比: 1-7%

**G3. 高基数聚合 + 过滤（用户行为分析）**

```sql
-- SQL — 按 user_id 聚合 Top 1000 活跃用户
SELECT user_id, COUNT(*) as req_count, AVG(response_time_ms) as avg_rt, MAX(bytes) as max_bytes, SUM(bytes) as total_bytes
FROM perf_test_10m
WHERE @timestamp > NOW() - INTERVAL 1 DAY AND status_code = 200
GROUP BY user_id
ORDER BY req_count DESC
LIMIT 1000;
```
```json
// DSL
{"size":0,"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"now-1d"}}},{"term":{"status_code":200}}]}},"aggs":{"by_user":{"terms":{"field":"user_id","size":1000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"max_bytes":{"max":{"field":"bytes"}},"total_bytes":{"sum":{"field":"bytes"}}}}}}
```
> 预期 DSL: 300-600ms（330K docs + 100K 桶 + 4 指标）
> 预期 SQL 开销: 5-15ms
> 预期开销占比: 1-5%

**G4. 复合查询 + 排序 + 中等结果集**

```sql
-- SQL — 多条件 + 排序 + 5000 行
SELECT service, level, response_time_ms, bytes, @timestamp, request_path, client_ip
FROM perf_test_10m
WHERE status_code >= 400 AND response_time_ms > 2000 AND @timestamp > NOW() - INTERVAL 3 DAY
ORDER BY response_time_ms DESC
LIMIT 5000;
```
```json
// DSL
{"query":{"bool":{"filter":[{"range":{"status_code":{"gte":400}}},{"range":{"response_time_ms":{"gt":2000}}},{"range":{"@timestamp":{"gte":"now-3d"}}}]}},"sort":[{"response_time_ms":"desc"}],"size":5000,"_source":["service","level","response_time_ms","bytes","@timestamp","request_path","client_ip"]}
```
> 预期 DSL: 150-400ms（过滤 + 排序 + fetch 5K）
> 预期 SQL 开销: 5-15ms
> 预期开销占比: 2-7%

#### 场景组 H：极重查询（验证开销占比收敛）

**H1. 全表扫描 + 高基数聚合**

```sql
-- SQL — 全表 10M docs 按 session_id 聚合（500K 桶）
SELECT session_id, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt
FROM perf_test_10m
WHERE response_time_ms > 100
GROUP BY session_id
ORDER BY cnt DESC
LIMIT 5000;
```
```json
// DSL
{"size":0,"query":{"range":{"response_time_ms":{"gt":100}}},"aggs":{"by_session":{"terms":{"field":"session_id","size":5000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}
```
> 预期 DSL: 800-2000ms（10M docs + 500K 桶聚合）
> 预期 SQL 开销: 10-30ms
> 预期开销占比: 0.5-3% ✅ 核心验证场景

**H2. 全表多级聚合 + percentile**

```sql
-- SQL — 全表 按 service×level×region 聚合（500 桶）+ percentile
SELECT service, level, region, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, SUM(bytes) as total_bytes, PERCENTILE(response_time_ms, 99) as p99
FROM perf_test_10m
GROUP BY service, level, region
ORDER BY service, cnt DESC;
```
```json
// DSL
{"size":0,"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"by_region":{"terms":{"field":"region"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"total_bytes":{"sum":{"field":"bytes"}},"p99":{"percentiles":{"field":"response_time_ms","percents":[99]}}}}}}}}}}
```
> 预期 DSL: 600-1500ms（10M docs + 三级聚合 + percentile）
> 预期 SQL 开销: 5-15ms（composite 聚合下推，可能比 DSL 更快）
> 预期开销占比: 0.3-2% ✅ 核心验证场景

**H3. 大范围扫描 + 排序 + 大结果集**

```sql
-- SQL — 7 天数据（~2.3M docs）排序取 10000
SELECT * FROM perf_test_10m
WHERE @timestamp > NOW() - INTERVAL 7 DAY
ORDER BY response_time_ms DESC
LIMIT 10000;
```
```json
// DSL
{"query":{"range":{"@timestamp":{"gte":"now-7d"}}},"sort":[{"response_time_ms":"desc"}],"size":10000}
```
> 预期 DSL: 400-800ms（2.3M docs 排序 + fetch 10K 跨节点）
> 预期 SQL 开销: 5-15ms
> 预期开销占比: 0.6-3% ✅ 核心验证场景

#### 场景组 I：SQL 独有（大数据量验证）

**I1. 三路 UNION ALL + 聚合（pushdown ON）**

```sql
SELECT service, COUNT(*) as cnt FROM perf_test_10m WHERE level = 'ERROR' GROUP BY service
UNION ALL
SELECT service, COUNT(*) as cnt FROM perf_test_10m WHERE level = 'WARN' GROUP BY service
UNION ALL
SELECT service, COUNT(*) as cnt FROM perf_test_10m WHERE level = 'DEBUG' GROUP BY service;
```
> 预期 SQL: 50-200ms（3 次跨 shard 下推聚合 + 合并 15 行）

**I2. 大表 JOIN**

```sql
SELECT a.service, a.level, a.response_time_ms, b.dept_name
FROM perf_test_10m a
JOIN perf_test_meta b ON a.host = b.host
WHERE a.level = 'ERROR' AND a.@timestamp > NOW() - INTERVAL 1 DAY
LIMIT 10000;
```
> 预期 SQL: 300-800ms（~330K 行 Hash Join）

**I3. IN 子查询**

```sql
SELECT service, level, response_time_ms FROM perf_test_10m
WHERE host IN (SELECT host FROM perf_test_meta)
AND @timestamp > NOW() - INTERVAL 1 DAY
LIMIT 10000;
```
> 预期 SQL: 300-800ms（大表 IN 子查询）

### 5.3 预期结果矩阵

| 场景 | 预期 DSL (ms) | 预期 SQL (ms) | 预期翻译开销 (ms) | 预期开销占比 | 验证目标 |
|------|:---:|:---:|:---:|:---:|------|
| G1 时间范围+聚合 | 300-600 | 310-620 | 5-15 | 1-5% | 生产典型场景开销可忽略 |
| G2 多字段排序+10K | 200-500 | 210-520 | 5-15 | 1-7% | 大结果集开销可忽略 |
| G3 高基数聚合 | 300-600 | 310-620 | 5-15 | 1-5% | 100K 桶开销可忽略 |
| G4 复合+5K | 150-400 | 160-420 | 5-15 | 2-7% | 中等重查询 |
| **H1 全表高基数聚合** | **800-2000** | **820-2030** | **10-30** | **0.5-3%** | **核心：极重查询验证** |
| **H2 全表多级聚合** | **600-1500** | **610-1520** | **5-15** | **0.3-2%** | **核心：极重查询验证** |
| **H3 大范围+10K** | **400-800** | **410-820** | **5-15** | **0.6-3%** | **核心：极重查询验证** |
| I1 三路UNION | — | 50-200 | — | — | SQL 独有 |
| I2 大表JOIN | — | 300-800 | — | — | SQL 独有 |
| I3 IN子查询 | — | 300-800 | — | — | SQL 独有 |

### 5.4 验证方法论

#### 5.4.1 翻译开销分解（核心方法）

> 通过 OpenSearch slowlog 获取实际服务端执行时间，精确分解 SQL 翻译开销。

**步骤**：

1. **开启 slowlog**：`index.search.slowlog.threshold.query.info: 0ms`（记录所有查询）
2. **运行 SQL 查询**：记录端到端延迟 `T_sql`
3. **从 slowlog 提取**：服务端执行时间 `T_exec`（搜索引擎实际耗时）
4. **计算翻译开销**：`T_translate = T_sql - T_exec`
5. **运行等价 DSL**：记录端到端延迟 `T_dsl`
6. **验证**：`T_exec ≈ T_dsl`（SQL 生成的 DSL 执行时间应接近直接 DSL）
7. **开销占比**：`T_translate / T_dsl × 100%`

**关键指标**：

| 指标 | 含义 | 预期 |
|------|------|------|
| `T_translate` | SQL 翻译开销（解析+分析+规划+DSL生成） | 5-30ms，与查询复杂度弱相关 |
| `T_translate / T_dsl` | 翻译开销占 DSL 执行时间的比例 | <10%（重查询） |
| `T_translate` 标准差 | 翻译开销的稳定性 | <2ms（JIT 充分预热后） |

#### 5.4.2 预热与测试轮次

```
预热: 30 轮（充分触发 JIT C2 + Calcite Janino codegen + OS page cache + filter cache）
测试: 100 轮（多节点环境下每轮更慢，100 轮足够统计）
两遍运行: 第一遍丢弃（验证 JIT 稳定性）
```

#### 5.4.3 缓存控制

| 缓存 | 处理方式 | 理由 |
|------|---------|------|
| request cache | `?request_cache=false` | 聚合场景必须禁用 |
| filter cache | 不禁用 | 生产环境会命中，保留此优势 |
| query cache | 不禁用 | 同上 |
| OS page cache | 预热后保留 | 生产环境会命中 |

#### 5.4.4 结果验证

每个场景需验证：

1. **结果集等价**：SQL 和 DSL 返回相同数据（行数+值，忽略顺序）
2. **执行路径确认**：通过 `_explain` 确认 SQL 走 V2/Calcite/Legacy 哪条路径
3. **下推确认**（Calcite 路径）：explain 中是否有 `PushDownContext`
4. **回退监控**：检查服务端日志 "Fallback to V2" 消息
5. **slowlog 一致性**：SQL 生成的 DSL 与手写 DSL 的服务端执行时间是否接近

### 5.5 预期结论框架

```
目标结论: 大数据量重查询场景下，SQL 与 DSL 性能差异不大。

验证条件:
  H1/H2/H3 场景（DSL >800ms）: SQL 开销占比 <5%  → "差异不大" ✅
  G1-G4 场景（DSL 150-600ms）: SQL 开销占比 <10% → "差异可接受" ✅
  T_translate 稳定在 5-30ms    → "固定范围内" ✅
  T_exec ≈ T_dsl               → "SQL 生成的 DSL 执行效率与手写 DSL 一致" ✅

如验证通过，结论:
  "在 10M 数据量多节点集群的重查询场景下（DSL 执行时间 >200ms），
   SQL 接口的额外翻译开销稳定在 5-30ms 范围内，
   占 DSL 执行时间的 1-10%，性能差异在可接受范围内。
   SQL 生成的 DSL 执行效率与手写 DSL 基本一致。"

如验证不通过（开销占比 >10%）:
  分析根因 — 可能是 JdbcResponseFormatter 序列化开销（大结果集）
  或 V2 聚合路径翻译开销（高基数聚合）导致。
```

### 5.6 与单节点 1M 测试的对比维度

| 维度 | 单节点 1M（已完成） | 多节点 10M（本方案） | 预期变化 |
|------|-----|-----|------|
| DSL 绝对延迟 | 6-60ms | 150-2000ms | 10-30x 增大 |
| SQL 翻译开销 | 0.5-80ms | 5-30ms | 范围收窄（JIT 充分预热） |
| SQL 开销占比 | 18-807% | 0.3-10% | 大幅下降 |
| 网络 | 本地回环 | 万兆局域网 | DSL 有 scatter-gather 开销 |
| 数据量 | 1M | 10M | DSL 执行时间线性增长 |
| 聚合桶数 | 4-100 | 500-500K | DSL 聚合计算显著变重 |
| 结果集 | 10-10K 行 | 5K-10K 行 | 相似（受 max_result_window 限制） |
| forcemerge | 是 | 否 | DSL 需多 segment 合并 |
| 可推广性 | 仅单节点 | 接近生产 | ✅ 可推广到生产环境 |

### 5.7 执行步骤

```bash
#!/bin/bash
# run_10m_benchmark.sh

ES="http://node-1:9200"

echo "=== 1. Create Index ==="
curl -s -X PUT "$ES/perf_test_10m" -H 'Content-Type: application/json' -d '@index_10m_mapping.json'

echo "=== 2. Load 10M Docs ==="
# 分批导入，每批 100K
for i in $(seq -w 000 099); do
  curl -s -X POST "$ES/_bulk?refresh=false" -H 'Content-Type: application/x-ndjson' --data-binary @batch_$i.json > /dev/null
  echo "Loaded batch $i: $(curl -s "$ES/_cat/indices/perf_test_10m?h=docs.count" | tr -d ' ') docs"
done

echo "=== 3. Wait for Indexing ==="
while [ "$(curl -s "$ES/_cat/indices/perf_test_10m?h=docs.count" | tr -d ' ')" -lt "10000000" ]; do
  echo "  $(curl -s "$ES/_cat/indices/perf_test_10m?h=docs.count" | tr -d ' ')/10000000"
  sleep 10
done

echo "=== 4. DO NOT forcemerge (模拟生产) ==="
echo "Skipping forcemerge to simulate production segment distribution"

echo "=== 5. Enable Slowlog ==="
curl -s -X PUT "$ES/perf_test_10m/_settings" -H 'Content-Type: application/json' -d '{
  "index.search.slowlog.threshold.query.info": "0ms",
  "index.search.slowlog.threshold.fetch.info": "0ms"
}'

echo "=== 6. Enable Calcite ==="
curl -s -X PUT "$ES/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": {"plugins.calcite.enabled": "true", "plugins.calcite.pushdown.enabled": "true"}
}'

echo "=== 7. Result Equivalence Audit ==="
python3 verify_results.py

echo "=== 8. Run Benchmarks (pass 1 - discard) ==="
python3 benchmark_10m.py --tag pass1

echo "=== 9. Run Benchmarks (pass 2 - keep) ==="
python3 benchmark_10m.py --tag pass2

echo "=== 10. Analyze ==="
python3 analyze_results.py
```

### 5.8 分析报告模板

```markdown
## 10M 多节点验证结果

### 翻译开销分解

| 场景 | T_sql (ms) | T_exec (ms, slowlog) | T_translate (ms) | T_dsl (ms) | T_translate/T_dsl |
|------|:---:|:---:|:---:|:---:|:---:|
| H1 | ___ | ___ | ___ | ___ | ___% |
| H2 | ___ | ___ | ___ | ___ | ___% |
| H3 | ___ | ___ | ___ | ___ | ___% |

### 结论验证

- [ ] H1/H2/H3 开销占比 <5% → "差异不大" ✅/❌
- [ ] G1-G4 开销占比 <10% → "差异可接受" ✅/❌
- [ ] T_translate 稳定在 5-30ms → "固定范围" ✅/❌
- [ ] T_exec ≈ T_dsl → "执行效率一致" ✅/❌

### 最终结论
___
```
