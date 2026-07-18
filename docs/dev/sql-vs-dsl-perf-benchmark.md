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
- **Calcite 优化器**：基于关系代数，支持算子下推、代价优化
- **算子下推**：filter、aggregation、sort、limit 可下推到 OpenSearch 搜索引擎
- **内存计算**：UNION 合并、未下推的 JOIN 等在内存中用 Enumerable 算子计算
- **与 V2 共享 AST**：前端解析（ANTLR + AstBuilder）相同，从 `QueryService.shouldUseCalcite()` 开始分叉
- **PPL 默认走此路径**：`plugins.calcite.enabled=true`（3.3.0 起默认），PPL 查询走 Calcite
- **SQL 仅 UNION 走此路径**：我们的扩展让 `shouldUseCalcite` 检测到 Union 节点时路由到 Calcite（`containsUnion(plan)`）

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

3. **Calcite 的下推机制**：Calcite 会尽可能将操作下推到 OpenSearch（filter、aggregation、sort、limit），但某些操作必须在内存中执行（如 UNION 的合并、未下推的 JOIN）。

4. **游标实现不同**：
   - DSL 用 `search_after`（无状态，需要排序字段唯一）
   - SQL V2 用序列化游标（有状态，占用服务端内存，通过 `PaginatedPlanCache` 序列化 PhysicalPlan）
   - SQL Calcite 路径不支持 V2 序列化游标，但通过 `EnumerableLimit` 原生支持分页（Union 查询的分页由 Calcite 处理）

5. **线程池大小差异**：
   - `search` 线程池：8 核机器上 13 线程（`int((cores * 3) / 2) + 1`）
   - `sql-worker` 线程池：8 核机器上 8 线程（`allocatedProcessors`）
   - 单线程测试无影响；并发测试时 DSL 有 62.5% 更多线程

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
JDK: 21
```

> ⚠️ **单 shard 局限性**：单 shard 消除了分布式协调开销，是 DSL 的最佳场景。
> 生产环境通常 3-10 个 shard，多 shard 下 DSL 有 scatter-gather 开销但 SQL 翻译开销不变，
> SQL 的相对开销比例会随 shard 数增加而下降。
> 建议追加 3-shard 变体验证相对开销的稳定性。

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
    "number_of_shards": 1,
    "number_of_replicas": 0,
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

**C1. 简单聚合**

```sql
-- SQL
SELECT level, COUNT(*) as cnt FROM perf_test GROUP BY level;
```
```json
// DSL
{"size": 0, "aggs": {"by_level": {"terms": {"field": "level"}}}}
```

**C2. 多级聚合（flat GROUP BY vs composite aggregation）**

```sql
-- SQL
SELECT level, service, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt
FROM perf_test
GROUP BY level, service;
```
```json
// DSL — 使用 composite 聚合与 SQL flat GROUP BY 语义对等
{"size": 0, "aggs": {"by_level_service": {"composite": {"sources": [
  {"level": {"terms": {"field": "level"}}},
  {"service": {"terms": {"field": "service"}}}
]}, "aggs": {"avg_rt": {"avg": {"field": "response_time_ms"}}}}}}
```

> ⚠️ **语义对等说明**：SQL `GROUP BY level, service` 是 flat 多键聚合（所有组合在一层）。
> DSL 的 nested `aggs` 是树形嵌套聚合（不同执行路径）。
> 为确保公平对比，DSL 使用 `composite` 聚合（也是 flat 多键），而非 nested `aggs`。

**C3. 范围聚合**

```sql
-- SQL
SELECT service, COUNT(*) as cnt
FROM perf_test
WHERE response_time_ms > 1000
GROUP BY service
ORDER BY cnt DESC;
```
```json
// DSL
{"size": 0, "query": {"range": {"response_time_ms": {"gt": 1000}}}, "aggs": {
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

### 3.3 测试方法

#### 3.3.1 压测工具

```python
# benchmark.py
import requests
import time
import json
import statistics

BASE_URL = "http://localhost:9200"
WARMUP_RUNS = 50      # ← 充分预热 JIT（原 5 轮不足）
TEST_RUNS = 200       # ← 200 轮获得稳定 p99

# 使用 Session 复用 TCP 连接（消除连接建立开销噪声）
session = requests.Session()

def bench_sql(query, fetch_size=None):
    body = {"query": query}
    if fetch_size:
        body["fetch_size"] = fetch_size
    resp = session.post(f"{BASE_URL}/_plugins/_sql", json=body)
    return resp

def bench_dsl(index, dsl):
    resp = session.post(f"{BASE_URL}/{index}/_search", json=dsl)
    return resp

def run_benchmark(name, func, *args, warmup=WARMUP_RUNS, runs=TEST_RUNS):
    # Warmup
    for _ in range(warmup):
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
        print(f"{name}:")
        print(f"  avg={statistics.mean(latencies):.1f}ms  p50={p50:.1f}ms  p99={p99:.1f}ms  min={min(latencies):.1f}ms  max={max(latencies):.1f}ms")
        # 输出 per-run 数据用于异常值分析
        return latencies
    return []
```

#### 3.3.2 执行步骤

```
1. 启动 OpenSearch 集群
2. 创建索引（含 max_result_window: 20000）+ 导入 100 万数据
3. 轮询等待索引完成（_cat/indices 确认 docs.count=1000000）
4. 强制 merge 到 1 个 segment
5. 预热查询（warmup 50 轮，让 JIT C2 编译 + ANTLR/Calcite classloader 初始化 + OS cache 填充）
6. 正式测试（每个场景 200 轮，记录 per-run 延迟）
7. 运行完整测试套件两遍（第一遍丢弃，验证 JIT 稳定性）
8. 收集指标
9. 清理
```

#### 3.3.3 收集的指标

| 指标 | 收集方式 | 说明 |
|------|---------|------|
| **端到端延迟** | 客户端计时 | 从发送请求到收到响应的总时间 |
| **p50 / p99 延迟** | 客户端统计（200 轮） | 中位数和 99 分位延迟 |
| **per-run 延迟** | 客户端记录每次 | 用于检测 GC/JIT 异常值 |
| **吞吐量 (QPS)** | 客户端统计 | 每秒完成请求数 |
| **服务端查询耗时** | DSL: `profile:true`; SQL: 从 `_explain` 提取生成的 DSL 再 `profile:true` | 分离翻译开销和执行开销 |
| **SQL explain 计划** | `_plugins/_sql/_explain` | 确认走 V2 / Calcite / Legacy V1 哪条路径 |
| **Calcite 回退监控** | 服务端日志 | 检查 "Fallback to V2 query engine" 日志 |
| **JVM heap 使用** | `_nodes/stats/jvm` | 量化内存差异（每个场景前后各采集一次） |
| **GC 频率/耗时** | GC 日志 (`-Xlog:gc*`) | 检测 SQL 路径是否触发更多 GC |
| **线程池状态** | `_nodes/stats/thread_pool` | sql-worker vs search 线程池利用率 |
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
4. **翻译开销分解**：`SQL总延迟 - profile提取的DSL执行时间 = 翻译开销`
5. **p99 稳定性**：SQL 的解析层是否有 GC 抖动
6. **内存影响**：`heap使用(SQL) - heap使用(DSL)`，定义"显著"为 >5% heap 增长

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

echo "=== 5. Cleanup ==="
# curl -X DELETE "$ES/perf_test"
```

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

### 4.6 关键洞察

UNION 扩展的"三步模式"可以复用到 JOIN 和 EXISTS 子查询：

```
步骤 1: AstBuilder 中不抛异常，改为构建对应 AST 节点
步骤 2: QueryService.shouldUseCalcite() 中新增检测条件，路由到 Calcite
步骤 3: CalciteRelNodeVisitor 中已有实现（为 PPL 服务），确认可用
```

这两个特性（JOIN、EXISTS）的 Calcite 底层实现已经存在（为 PPL 服务），只需要打通 SQL → Calcite 的路由。

**依赖关系**：

```
COALESCE ──────────────────────────────── 独立
DATE_HISTOGRAM ────────────────────────── 独立
EXISTS 子查询 ─────────────────────────── 独立
3表+ JOIN ─────────────────────────────── 独立
  ├── JOIN + GROUP BY ─────────────────── 依赖 3表+ JOIN
  ├── JOIN + 聚合函数 ─────────────────── 依赖 3表+ JOIN
  └── 子查询 + 外层 GROUP BY ──────────── 依赖 3表+ JOIN
标量子查询 ────────────────────────────── 独立（但复杂度最高）
CTE ───────────────────────────────────── 独立（可用派生表替代）
```
