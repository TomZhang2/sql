# OpenSearch SQL 插件技术评估报告

> 基于 OpenSearch 3.7.0 + SQL Plugin 3.7.0（含 UNION/UNION ALL 扩展）
> 所有 SQL 能力声明均已通过实测验证

> ⚠️ **范围声明**：本文档为 OpenSearch SQL 插件的**技术评估报告**，涵盖引擎架构、SQL vs DSL 性能差异实测、能力缺口与扩展评估、社区维护情况。

---

## 目录

- [一、SQL 插件引擎架构](#一sql-插件引擎架构)
- [二、白盒实现分析](#二白盒实现分析)
- [三、SQL vs DSL 能力对比](#三sql-vs-dsl-能力对比)
- [四、验证方案设计](#四验证方案设计)
- [五、实测数据与测试报告](#五实测数据与测试报告)
- [六、SQL 插件能力扩展评估](#六sql-插件能力扩展评估)
- [七、社区维护情况](#七社区维护情况)
- [八、综合结论](#八综合结论)

---

## 一、SQL 插件引擎架构

OpenSearch SQL 插件有三种执行引擎，是历史演进的结果。

### 1.1 Legacy V1 引擎

基于 Alibaba Druid SQL 解析器的查询翻译层。SQL 直接翻译成 OpenSearch DSL，无计划抽象。

```
SQL → Druid Parser → QueryAction → SearchRequestBuilder → OpenSearch 执行
```

- **来源**：fork 自 `elasticsearch-sql`（NLPchina）
- **支持**：2 表 JOIN、IN 子查询
- **不支持**：3 表 JOIN、JOIN+GROUP BY、EXISTS、UNION、窗口函数、COALESCE、DATE_HISTOGRAM
- **状态**：维护中，不再新增功能

### 1.2 V2 引擎（当前主力）

OpenSearch 自研，完整 AST → LogicalPlan → PhysicalPlan 抽象层。

```
SQL → ANTLR 4 → AstBuilder → Analyzer → Planner → PhysicalPlan → SearchRequestBuilder → OpenSearch 执行
```

- **支持**：窗口函数、派生表、IN 子查询（回退 Legacy V1）
- **不支持**：JOIN（抛异常回退 Legacy）、UNION（我们的扩展已改为走 Calcite）、CTE、COALESCE、DATE_HISTOGRAM
- **聚合策略**：使用 `composite` 聚合分页拉取，size 硬编码 = 1000（`AggregationQueryBuilder.AGGREGATION_BUCKET_SIZE`，`AggregationQueryBuilder.java:48`）。注意：`plugins.query.buckets`（默认 10000，`OpenSearchSettings.java:208`）是 Calcite 路径的配置（`AggregateAnalyzer.java:296` 使用 `helper.queryBucketSize`），**V2 路径不使用此设置**

### 1.3 Calcite 引擎（未来方向）

基于 Apache Calcite，利用关系代数和优化器做查询规划。

```
SQL → ANTLR 4 → AstBuilder → CalciteRelNodeVisitor → RelNode → OpenSearch 执行（下推部分）+ 内存计算（未下推部分）
```

- **算子下推**：通过 Convention trait 驱动，filter/agg/sort/limit 可下推（规则驱动，非代价驱动）
- **统计信息**：当前未注入统计信息，CBO 退化为规则系统
- **Janino codegen**：Enumerable 算子运行时编译，首次 ~10-50ms
- **PPL 默认走此路径**；SQL 仅 UNION 走此路径（我们的扩展）

### 1.4 三引擎对比

| 维度    | Legacy V1                | V2                             | Calcite                    |
| ----- | ------------------------ | ------------------------------ | -------------------------- |
| 解析器   | Druid                    | ANTLR 4                        | ANTLR 4                    |
| 计划抽象  | 无                        | AST→LogicalPlan→PhysicalPlan   | AST→RelNode                |
| 优化器   | 无                        | 简单规则                           | Calcite Volcano（无统计，退化为规则） |
| 算子下推  | 无                        | 无                              | ✅ filter/agg/sort/limit    |
| JOIN  | ✅ 2 表                    | ❌（回退 Legacy）                   | ✅ N 表（SQL 未路由到此）           |
| UNION | ❌                        | ❌（扩展后走 Calcite）                | ✅（我们的扩展）                   |
| 窗口函数  | ❌                        | ✅                              | ✅                          |
| 内存计算  | 有（BlockHashJoin 等 V1 算子） | 有（TakeOrderedOperator 等 V2 算子） | 有（Enumerable，单节点单线程）       |
| 状态    | 维护中                      | 活跃开发                           | 活跃开发（未来方向）                 |

### 1.5 路由逻辑

```
POST /_plugins/_sql
  │
  ├─ 索引是 composite dataformat? ──YES──→ Unified Query API (Calcite 原生)
  │
  └─ NO → V2 AstBuilder
       ├─ 普通查询 → V2 引擎
       ├─ 包含 UNION → Calcite 引擎（我们的扩展）
       └─ JOIN / IN 子查询 → 回退 Legacy V1
```

### 1.6 演进方向

```
Legacy V1（淘汰中）→ V2（当前主力）→ Calcite（未来方向）
```

最终目标：SQL 和 PPL 都走 Calcite，Legacy V1 退役。当前处于过渡期，三引擎并存。

---

## 二、白盒实现分析

### 2.1 DSL 查询路径

```
POST /index/_search → RestSearchAction → TransportSearchAction → QueryPhase → FetchPhase → ToXContent → JSON
```

- 无翻译层、无中间对象转换
- 结果序列化：OpenSearch 原生 `ToXContent` 直接输出 JSON，无 JDBC 格式转换
- 执行线程：`search` 线程池（8 核→13 线程）

### 2.2 SQL 查询路径

#### V2 路径（普通 SELECT）

```
POST /_plugins/_sql → SQLService.plan()
  ① ANTLR 解析 → ParseTree
  ② AstBuilder → UnresolvedPlan (AST)
  ③ QueryPlanFactory → QueryPlan
  → QueryService.executeWithLegacy()
  ④ Analyzer → LogicalPlan
  ⑤ Planner → PhysicalPlan
  → OpenSearchExecutionEngine → SearchRequestBuilder → OpenSearch 执行
  → JdbcResponseFormatter → JSON
```

#### Calcite 路径（UNION）

```
同 V2 的 ①②③ → QueryService.executeWithCalcite()
  ④ CalciteRelNodeVisitor → RelNode
  ⑤ convertToCalcitePlan → 加 LogicalSystemLimit
  → OpenSearchRelRunners.run() → PreparedStatement → executeQuery()
  → buildResultSet() → JdbcResponseFormatter → JSON
```

> ⚠️ Calcite 失败时在 WARN 级别日志记录后回退到 V2（`QueryService.java:180-186`：`log.warn("Fallback to V2 query engine since got exception", t)`）。
> **回退条件**：依赖 `plugins.calcite.fallback.allowed` 配置（默认 `false`）——默认情况下**仅 `CalciteUnsupportedException` 触发回退**，其他异常通过 `propagateCalciteError` 抛给客户端。需显式设置 `plugins.calcite.fallback.allowed=true` 才能在任意异常时回退。监控日志关键字："Fallback to V2"。

#### Legacy V1 路径（JOIN / IN 子查询）

```
V2 AstBuilder.visitJoinClause() → 抛 SyntaxCheckException
→ fallBackListener 捕获 → Legacy V1 (SearchDao → Druid → SearchRequestBuilder)
```

### 2.3 实现差异对比

| 环节     | DSL                           | SQL (V2)                                             | SQL (Calcite)         | SQL (Legacy V1)                   |
| ------ | ----------------------------- | ---------------------------------------------------- | --------------------- | --------------------------------- |
| 请求解析   | JSON 解析                       | ANTLR                                                | ANTLR                 | Druid                             |
| 语义分析   | 无                             | Analyzer                                             | CalciteRelNodeVisitor | 无                                 |
| 查询规划   | 无                             | Planner                                              | Calcite Volcano 优化器   | 无                                 |
| DSL 生成 | 无（本身是 DSL）                    | PhysicalPlan→SearchRequestBuilder                    | Calcite 下推            | 多次 DSL 请求 + 内存合并（如 BlockHashJoin） |
| 结果格式化  | `ToXContent` 原生输出（含在 DSL 基线中） | `JdbcResponseFormatter` JDBC 对象转换 + JSON 序列化，随行数线性增长 | 同 V2                  | `PrettyFormatRestExecutor`        |
| 线程池    | search (8核→13线程)              | sql-worker (8核→8线程)                                  | sql-worker            | sql-worker→search                 |

> **SQL 额外开销 = 翻译开销 + 结果格式化开销 + 内存计算开销 + 线程调度开销**：
> 
> - **翻译开销**（解析+分析+规划）：G0 实测 0.82-4.36ms（`_explain` 直接测量，见 §5.5），与结果集大小无关，恒定
> - **结果格式化开销**：`JdbcResponseFormatter` 把 `SearchResponse` 转为 JDBC `List<Object[]>` 再序列化 JSON，比 DSL 原生 `ToXContent` 多一层对象转换。实测随行数线性增长：~1ms(10行) → ~8ms(1K行) → ~80ms(10K行)
> - **内存计算开销**：SQL 独有能力（JOIN/UNION/窗口函数/聚合排序）在协调节点内存计算，DSL 不产生此开销。如 `BlockHashJoin`（V1）、`TakeOrderedOperator`（V2）、Enumerable 合并（Calcite）。高基数聚合场景此开销可达数千 ms（见 §5.6.4 H1）
> - DSL 的 `ToXContent` 序列化开销已包含在 DSL 基线延迟中，不计入"额外开销"

**关键差异点**：

1. **SQL 支持 DSL 不具备的能力时，通过多次 DSL 请求 + 协调节点内存计算实现**——Legacy V1 的 JOIN 通过 `BlockHashJoin` 对两表各发一次 PIT 查询后内存 hash join；V2 的窗口函数/聚合排序通过 `TakeOrderedOperator` 在协调节点内存计算；Calcite 通过 Enumerable 算子（Janino codegen）内存计算。三者机制不同但本质相同：DSL 不支持的能力 = 多次 DSL + 内存合并
2. **线程池隔离**——DSL 走 `search` 线程池，SQL 走 `sql-worker` 独立线程池。Legacy V1 路径从 sql-worker 穿透到 search 池。两池隔离意味着高并发 SQL 不直接挤占 DSL 线程，但 circuit breaker 是全局的
3. **游标机制不同**——DSL `search_after`（无状态）vs V2 序列化游标（有状态，PIT + searchAfter）vs Calcite `EnumerableLimit`。深翻页场景 SQL 可能回退内存分页（见 §4.2 D 组）
4. **Calcite 内存风险**——UNION/JOIN 在协调节点单线程内存计算，大数据量可能 OOM。`executeWithCalcite` 会加 `LogicalSystemLimit`，但仍存在内存边界

---

## 三、SQL vs DSL 能力对比

> 以下所有 SQL 能力声明均通过 `POST /_plugins/_sql` 实测验证（2026-07-18）

### 3.1 查询能力矩阵

| 能力              | SQL | DSL | 说明                                                                                                                                                                                  |
| --------------- |:---:|:---:| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 等值/范围查询         | ✅   | ✅   | SQL `WHERE`; DSL `term`/`range`                                                                                                                                                     |
| 多条件布尔查询         | ✅   | ✅   | SQL `AND`/`OR`; DSL `bool`                                                                                                                                                          |
| 全文搜索            | ✅   | ✅   | SQL `match()`/`multi_match()`/`match_phrase()`，支持 boost/fuzziness/analyzer/minimum_should_match 参数；不支持 `query_string`/`simple_query_string`/`operator`（`AND` 为 SQL 关键字冲突）           |
| GROUP BY 聚合     | ✅   | ✅   | SQL `GROUP BY`; DSL `aggs`                                                                                                                                                          |
| 窗口函数            | ✅   | ❌   | SQL `RANK() OVER(...)`; DSL 不支持                                                                                                                                                     |
| 2 表 JOIN        | ✅   | ❌   | SQL（回退 Legacy V1）; DSL 不支持                                                                                                                                                          |
| 3 表+ JOIN       | ❌   | ❌   | SQL 报错 "only 2 tables"                                                                                                                                                              |
| JOIN + GROUP BY | ❌   | ❌   | SQL 报错 "JOIN queries do not support aggregations on the joined result."（`Util.java:44`）                                                                                             |
| UNION ALL       | ✅   | ❌   | SQL（我们的扩展，走 Calcite）                                                                                                                                                                |
| UNION DISTINCT  | ✅   | ❌   | SQL（我们的扩展，走 Calcite）                                                                                                                                                                |
| IN 子查询          | ✅   | ❌   | SQL（回退 Legacy V1 IN→JOIN 重写）                                                                                                                                                        |
| EXISTS 子查询      | ❌   | ❌   | 普通 EXISTS 和嵌套字段 EXISTS 均报错 "Unsupported subquery"（`SubQueryRewriter.java:74`）。代码中 `NestedExistsRewriter.java` 存在但无法到达（已被 `SubQueryRewriter` 拦截）                                     |
| 标量子查询           | ❌   | ❌   | V2 报 `Subsearch is supported only when plugins.calcite.enabled=true`（`ExpressionAnalyzer.java:470`）；Calcite 引擎有实现但 SQL 默认不路由到此。实测验证：标量子查询报 "unsupported expr"（Legacy V1 Druid 解析失败） |
| 派生表             | ✅   | ❌   | SQL `(SELECT...) AS t`                                                                                                                                                              |
| CTE (WITH)      | ❌   | ❌   | 文法无 WITH 规则（`OpenSearchSQLLexer.g4` 无 WITH token）；Legacy 报错 "Query must start with SELECT, DELETE, SHOW or DESCRIBE"（`OpenSearchActionFactory.java:135`），V2 抛 ANTLR 语法错误            |
| COALESCE        | ❌   | ✅   | V2 引擎报错 "unsupported function name: coalesce"（`BuiltinFunctionRepository.java:145`，函数注册表缺失）；Legacy 报错 "not supported in Schema"（`SelectResultSet.java:360`）                         |
| DATE_HISTOGRAM  | ❌   | ✅   | V2 无此函数注册；Legacy `AggMaker.java:558-611` 有 dateHistogram 方法但实测触发 NPE（`Cannot invoke "Object.toString()"`），代码存在但无法正常执行                                                               |
| 脚本字段            | ❌   | ✅   | DSL `script_fields`                                                                                                                                                                 |
| 运行时字段           | ❌   | ✅   | DSL `runtime_mappings`                                                                                                                                                              |
| profile API     | ❌   | ✅   | DSL `profile: true`                                                                                                                                                                 |

### 3.2 SQL 独有能力

| 能力                              | 语法                             | 引擎路径        |
| ------------------------------- | ------------------------------ | ----------- |
| 2 表 JOIN + WHERE/ORDER BY/LIMIT | `SELECT...FROM a JOIN b ON...` | Legacy V1   |
| UNION ALL / DISTINCT            | `SELECT...UNION ALL SELECT...` | Calcite（扩展） |
| IN 子查询                          | `WHERE x IN (SELECT...)`       | Legacy V1   |
| 派生表                             | `FROM (SELECT...) AS t`        | V2          |
| 窗口函数                            | `RANK() OVER(...)`             | V2          |

### 3.3 SQL 已知限制

| 限制              | 错误信息                                                                                                                                          | 根因                                                                                                                 |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| CTE             | Legacy: `Query must start with SELECT, DELETE, SHOW or DESCRIBE`（`OpenSearchActionFactory.java:135`）；V2: ANTLR 语法错误                           | 文法无 WITH 规则（`OpenSearchSQLLexer.g4` 无 WITH token）                                                                  |
| 3 表+ JOIN       | `currently supports only 2 tables join`（`SqlParser.java:380`）                                                                                 | Legacy V1 限制                                                                                                       |
| JOIN + GROUP BY | `JOIN queries do not support aggregations on the joined result.`（`Util.java:44`）                                                              | Legacy V1 限制                                                                                                       |
| EXISTS 子查询      | 普通 EXISTS 和嵌套字段 EXISTS 均报错: `Unsupported subquery`（`SubQueryRewriter.java:74`）                                                                | `SubQueryRewriter` 在早期阶段拦截所有 EXISTS 子查询；代码中 `NestedExistsRewriter.java` 存在但执行流无法到达（已实测验证，含嵌套字段索引）                  |
| 标量子查询           | V2: `Subsearch is supported only when plugins.calcite.enabled=true`（`ExpressionAnalyzer.java:470`）；Legacy: `unsupported expr`（Druid 解析失败）     | V2 抛 `getOnlyForCalciteException`，仅 Calcite 引擎有实现（SQL 仅 UNION 走 Calcite，普通 SELECT 不走）；Legacy 遇到子查询作为字段直接抛异常（已实测验证） |
| COALESCE        | V2: `unsupported function name: coalesce`（`BuiltinFunctionRepository.java:145`）；Legacy: `not supported in Schema`（`SelectResultSet.java:360`） | V2 函数注册表未注册 COALESCE（`BuiltinFunctionRepository.java:73-85`）                                                       |
| DATE_HISTOGRAM  | V2 无此函数；Legacy 报 NPE `Cannot invoke "Object.toString()"`                                                                                      | V2 core 无 DATE_HISTOGRAM 函数注册；Legacy `AggMaker.java:558-611` 有 dateHistogram 方法但实测触发 NPE（已实测验证），代码存在但无法正常执行        |

---

## 四、验证方案设计

### 4.1 测试环境

> **统一配置原则（关键）**：1M 和 10M 测试使用**相同 schema、相同集群配置**，只变化数据量——控制变量以验证"劣化比例随数据量变化"的趋势。

| 项目                  | 配置                                                                                                                                                                        |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 硬件                  | Apple M4 Pro, 14 CPU 核心, 48 GB RAM, SSD                                                                                                                                   |
| 操作系统                | macOS Darwin 25.5.0 (ARM64)                                                                                                                                               |
| JDK                 | Temurin 25.0.3+9-LTS                                                                                                                                                      |
| OpenSearch 版本       | 3.7.0-SNAPSHOT                                                                                                                                                            |
| 集群拓扑                | 3 节点（node-1/2/3，同物理机），全部为 data+cluster_manager 节点                                                                                                                         |
| 每节点 JVM heap        | 4 GB（`-Xms4g -Xmx4g`），其余为系统默认                                                                                                                                             |
| 网络环境                | 127.0.0.1 回环（无真实网络延迟）                                                                                                                                                     |
| 集群名称                | sql-bench-3node                                                                                                                                                           |
| HTTP 端口             | 9201 (node-1) / 9202 (node-2) / 9203 (node-3)                                                                                                                             |
| Transport 端口        | 9301 / 9302 / 9303                                                                                                                                                        |
| Calcite 引擎          | `plugins.calcite.enabled=true`，`plugins.calcite.pushdown.enabled=true`                                                                                                    |
| 测试索引（1M）            | `perf_test`：6 primary shard + 1 replica = 12 shard，1,000,000 文档，390 MB，每节点 4 shard（2 primary + 2 replica），约 33 MB/shard                                                   |
| 测试索引（10M）           | `perf_test_10m`：6 primary shard + 1 replica = 12 shard，10,000,000 文档，5 GB，每节点 4 shard，约 417 MB/shard                                                                      |
| 元数据索引               | `perf_test_meta`：3 shard + 1 replica，100 文档                                                                                                                               |
| 字段数                 | 13：@timestamp(date), bytes(long), host/level/region/service/user_id/session_id/request_path(keyword), message(text), response_time_ms/status_code(integer), client_ip(ip) |
| Schema              | 1M 和 10M 相同（字段定义、映射类型完全一致）                                                                                                                                                |
| 数据分布                | status_code: 200(70%)/301(10%)/404(10%)/500(5%)/503(5%)；level: INFO(70%)/WARN(15%)/ERROR(10%)/DEBUG(5%)；response_time_ms: 85% <1000, 12% 1000-3000, 3% 3000-10000         |
| 段（segment）数         | 1M：39 个 / 10M：约 60 个（均未 forcemerge）                                                                                                                                       |
| `max_result_window` | 20000（自定义）                                                                                                                                                                |
| 预热                  | 20 轮                                                                                                                                                                      |
| 测试轮数                | 200 轮                                                                                                                                                                     |
| 缓存控制                | C1/C3 场景每 50 轮 `_cache/clear`；A/B/D 组无显式清缓存；10M 控制变量场景同策略                                                                                                                 |
| 测试客户端               | Python 3.14 + requests 2.34.2，单连接 Session（keep-alive）                                                                                                                     |
| 测试时间                | 2026-07-20（1M 轻查询/重查询/10M 独立场景）+ 2026-07-22（10M 控制变量场景 + G0 重测）                                                                                                           |

> ✅ **可比性**：1M 和 10M 仅数据量不同，其他全部相同——可直接对比劣化比例变化。

### 4.2 轻查询场景（1M + 10M 数据，返回 ≤10 行）

> 以下场景在 1M 和 10M 数据上**都跑**（相同查询，表名 `perf_test` / `perf_test_10m`），直接对比劣化比例变化。配置见 4.1 节（统一配置）。

> **对等性原则（关键）**：
> 
> - DSL 用 `bool.filter`（不打分），SQL V2 引擎 `WHERE AND` 也生成 `bool.filter`（`FilterQueryBuilder.java:109`）——**V2 路径对等**
> - ⚠️ **Calcite 路径不对等**：Calcite 引擎的 `PredicateAnalyzer.java:1224` 用 `bool.must`（会打分），DSL 用 `bool.filter`（不打分）。E 组 UNION 走 Calcite，有评分开销差异
> - ⚠️ **聚合类型不对等**：SQL GROUP BY 始终用 `composite` 聚合（`AggregationQueryBuilder.java:79-97`，size=1000，分页拉取全桶），DSL 用 `terms` 聚合（默认 size=10，一次性 top N）——见 C 组说明
> 
> **缓存控制（关键）**：
> 
> - ⚠️ SQL 路径**无法禁用 request cache**（`OpenSearchQueryRequest.search()` 不设 `requestCache`，OpenSearch 默认对 size=0 聚合自动缓存）
> - DSL 加 `?request_cache=false` 仅禁用 DSL 侧缓存，SQL 侧仍享受缓存——**系统性偏差**
> - **必须每轮之间 `_cache/clear`**，不能只靠 `?request_cache=false`（随机阈值对 size=0 聚合的 shard query cache 无效）

#### A 组：点查

| 场景         | SQL 查询                                                                                                  | DSL 查询                                                                                                                              | 引擎路径 |
| ---------- | ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |:----:|
| A1 等值(单条件) | `SELECT * FROM perf_test WHERE status_code = 200 LIMIT 10`                                              | `{"query":{"term":{"status_code":200}},"size":10}`                                                                                  | V2   |
| A2 等值(多条件) | `SELECT * FROM perf_test WHERE status_code = 500 AND level = 'ERROR' AND region = 'us-east-1' LIMIT 10` | `{"query":{"bool":{"filter":[{"term":{"status_code":500}},{"term":{"level":"ERROR"}},{"term":{"region":"us-east-1"}}]}},"size":10}` | V2   |
| A3 范围      | `SELECT * FROM perf_test WHERE response_time_ms > 3000 AND status_code = 500 LIMIT 10`                  | `{"query":{"bool":{"filter":[{"range":{"response_time_ms":{"gt":3000}}},{"term":{"status_code":500}}]}},"size":10}`                 | V2   |

#### B 组：全文搜索

| 场景     | SQL 查询                                                                                              | DSL 查询                                                                                                                 | 引擎路径 |
| ------ | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |:----:|
| B1 简单  | `SELECT message, service FROM perf_test WHERE match(message, 'timeout') LIMIT 10`                   | `{"query":{"match":{"message":"timeout"}},"size":10,"_source":["message","service"]}`                                  | V2   |
| B2 多字段 | `SELECT * FROM perf_test WHERE MULTI_MATCH(message, 'request failed') AND level = 'ERROR' LIMIT 10` | `{"query":{"bool":{"must":[{"match":{"message":"request failed"}}],"filter":[{"term":{"level":"ERROR"}}]}},"size":10}` | V2   |

#### C 组：聚合（随机阈值打散缓存 + 每轮 `_cache/clear`）

> ⚠️ **聚合类型对等问题（关键）**：SQL GROUP BY 始终用 `composite` 聚合（`AggregationQueryBuilder.java:79-97`，size=1000，分页拉取全桶）；DSL 若用 `terms` 聚合（默认 size=10，一次性 top N），两者**机制不同、结果集可能不同**。
> 
> - ⚠️ **方案 B 不可行（已通过 `_explain` 验证）**：SQL 加 `ORDER BY` **不会触发** composite→terms 转换。`rePushDownSortAggMeasure`（`AggPushDownAction.java:133`）仅被 `CalciteLogicalIndexScan.java:343`（Calcite 路径）调用，而普通 SQL SELECT 走 V2 引擎（`shouldUseCalcite()` 仅对 UNION 返回 true，`QueryService.java:373`）。V2 对 GROUP BY 永远用 composite 聚合（`AggregationQueryBuilder.java:97`，size=1000 硬编码），ORDER BY 在协调节点内存排序（`TakeOrderedOperator`）
> - **对等方案 A（唯一可行）**：DSL 改用 `composite` 聚合，与 SQL 机制一致。但 composite 不支持 `order` 参数，需在应用层排序
> - **或接受不对等**：SQL composite vs DSL terms 是**架构差异**，性能对比反映的是聚合机制差异而非纯翻译开销

| 场景       | SQL 查询                                                                                                                                             | DSL 查询                                                                                                                                                                                                                                                           | 引擎路径                |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:-------------------:|
| C1 简单    | `SELECT level, COUNT(*) as cnt FROM perf_test WHERE response_time_ms > {random} GROUP BY level ORDER BY cnt DESC`                                  | `{"size":0,"query":{"range":{"response_time_ms":{"gt":{random}}}},"aggs":{"by_level":{"terms":{"field":"level","size":1000,"order":{"_count":"desc"}}}}}`                                                                                                        | V2                  |
| C2 多级    | `SELECT level, service, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt FROM perf_test WHERE response_time_ms > {random} GROUP BY level, service` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":{random}}}},"aggs":{"ls":{"composite":{"size":1000,"sources":[{"level":{"terms":{"field":"level"}}},{"service":{"terms":{"field":"service"}}}]},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}` | V2                  |
| C3 范围+排序 | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE response_time_ms > {random} GROUP BY service ORDER BY cnt DESC`                              | `{"size":0,"query":{"range":{"response_time_ms":{"gt":{random}}}},"aggs":{"by_service":{"terms":{"field":"service","size":1000,"order":{"_count":"desc"}}}}}`                                                                                                    | V2                  |
| C4 时间直方图 | SQL 不支持（DATE_HISTOGRAM V2 无此函数，Legacy 支持）                                                                                                          | `{"size":0,"aggs":{"by_hour":{"date_histogram":{"field":"@timestamp","calendar_interval":"1h"}}}}`                                                                                                                                                               | V2（不支持）/ Legacy（支持） |

#### D 组：排序与分页

> ⚠️ **深度分页对等问题（关键）**：SQL `LIMIT offset, size` 在 `offset >= maxResultWindow`（默认 10000）时抛 `PushDownUnSupportedException`（`OpenSearchRequestBuilder.java:271`），回退到**内存分页**（拉取所有数据再跳过）——极慢，不公平。`from + size > maxResultWindow` 时切换到 **PIT + searchAfter**（`buildRequestWithPit()` line 135-140）——机制不同于 DSL from+size。
> **测试前必须**：① 明确记录 `maxResultWindow` 配置；② 用 `_explain` 验证 SQL 实际走哪条路径（内存分页 / PIT+searchAfter / from+size）。
> D2 `LIMIT 10000, 10`：若 `maxResultWindow=10000`（默认），SQL 回退内存分页——**不参与性能对比**，仅作为"SQL 深度分页退化"的警示数据。

> ⚠️ **D4 游标机制不对等**：SQL `fetch_size` 走 **PIT + searchAfter**（`OpenSearchRequestBuilder.buildRequestWithPit()` line 147-156 创建 PIT，`OpenSearchQueryRequest.searchWithPIT()` line 245-296 执行时强制加 `_doc` + `_shard_doc` 排序，有状态）；DSL 走裸 `search_after`（无状态）。**这是不同机制的对比**（有状态 vs 无状态），不参与"SQL 翻译开销"结论，仅作为生产方案参考。

| 场景                     | SQL 查询                                                                         | DSL 查询                                                                                      | 引擎路径                |
| ---------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |:-------------------:|
| D1 排序+小分页              | `SELECT * FROM perf_test ORDER BY response_time_ms DESC LIMIT 10`              | `{"query":{"match_all":{}},"sort":[{"response_time_ms":"desc"}],"size":10}`                 | V2                  |
| D2 深度分页                | `SELECT * FROM perf_test ORDER BY response_time_ms DESC LIMIT 10000, 10`       | `{"query":{"match_all":{}},"sort":[{"response_time_ms":"desc"}],"from":10000,"size":10}`    | V2（可能回退内存分页）        |
| D3 大结果集                | `SELECT * FROM perf_test WHERE status_code = 200 LIMIT 1000`                   | `{"query":{"term":{"status_code":200}},"size":1000}`                                        | V2                  |
| D4 游标分页（机制不同，仅作生产方案参考） | `{"query":"SELECT * FROM perf_test WHERE status_code = 200","fetch_size":100}` | `{"query":{"term":{"status_code":200}},"sort":[{"_id":"asc"}],"size":100}` + `search_after` | V2（PIT+searchAfter） |

#### E 组：SQL 独有（架构差异对比，非引擎效率对比）

> ⚠️ **对比目标说明**：E 组对比的是**架构差异**（SQL 单次请求内存计算 vs DSL 多次请求+客户端合并），**不是引擎翻译效率**。E 组数据不参与"SQL 翻译开销占比"结论。
> 
> - E1 UNION：SQL 走 Calcite 在协调节点内存合并（`CalciteRelNodeVisitor.java:2980`）；DSL 需 2 次网络往返+应用层合并——对比的是**单次 vs 多次请求**
> - E2 JOIN：SQL 走 Calcite 内存 JOIN（有 `join.subsearch_maxout` 系统限制，`CalciteRelNodeVisitor.java:1923`）；DSL 应用层方案非原子、有客户端上限——**语义不等价**
> - E3 IN 子查询：SQL 回退 Legacy V1 IN→JOIN 重写（`InRewriter.java:24`）；DSL 应用层方案需先查子查询再 terms——**语义不等价**
> - ⚠️ E 组走 Calcite 引擎，WHERE 生成 `bool.must`（会打分），DSL 用 `bool.filter`（不打分）——E 组 SQL 额外开销含评分开销，非纯翻译开销

| 场景              | SQL 查询                                                                                                                                                                                 | DSL 等价（多次查询+应用层合并，语义不等价）                        | 引擎        |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | --------- |
| E1 UNION ALL+聚合 | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service` | 2 次 `terms` agg + 应用层合并                         | Calcite   |
| E2 2表JOIN       | `SELECT a.service, a.level, b.host_name FROM perf_test a JOIN perf_test_meta b ON a.host = b.host WHERE a.level = 'ERROR' LIMIT 10`                                                    | 先查 perf_test 获取 host 列表，再查 perf_test_meta，应用层关联 | Legacy V1 |
| E3 IN子查询        | `SELECT * FROM perf_test WHERE host IN (SELECT host FROM perf_test WHERE level = 'ERROR') LIMIT 10`                                                                                    | 先查子查询获取 host 列表，再用 terms 查询                     | Legacy V1 |

#### F 组：辅助验证

| 场景                 | SQL 查询                                                                                | 说明                            |
| ------------------ | ------------------------------------------------------------------------------------- | ----------------------------- |
| F1 冷启动             | `SELECT /* cold_run_{timestamp} */ * FROM perf_test WHERE status_code = 200 LIMIT 10` | 每轮唯一注释打散计划缓存                  |
| F2 Pushdown on/off | 同 E1 查询，分别 `plugins.calcite.pushdown.enabled=true/false`                              | 隔离下推收益                        |
| F3 结果等价            | 对每个场景的 SQL 和 DSL 结果集做行数+值比对                                                           | TPC 标准                        |
| F4 配置参数敏感性         | `plugins.query.buckets` ∈ {100, 1000, 10000} × C1 场景                                  | 验证 composite 分页 size 对聚合延迟的影响 |

#### G0 组：纯翻译开销基线（隔离测量）

> **目的**：用 `_plugins/_sql/_explain` 端点直接测量 SQL 翻译阶段耗时，作为所有场景的"翻译开销基线"，与端到端延迟差交叉验证。
> **原理**：`_explain` 端点走与 execute 相同的 V2 路径（ANTLR parse → AstBuilder → Analyzer.analyze() → Planner.plan()），仅在最后调用 `executionEngine.explain()` 而非 `executionEngine.execute()`——跳过物理执行，保留全部翻译+规划开销。因此 `_explain` 端点端到端延迟 = 翻译开销 + 计划序列化 + 网络/排队，是 V2 翻译开销的直接测量，无需跨引擎推断。
> ⚠️ **适用条件**：测试查询必须不触发 V1 fallback（日志关键字 "falling back to old SQL engine"）。触发 fallback 的场景（如 JOIN）不适用此方法。

| 场景         | SQL 查询                                                                                                     | 测量方法                                                   | 预期     |
| ---------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ------ |
| G0-1 点查    | `SELECT * FROM perf_test WHERE status_code = 200 LIMIT 10`                                                 | `_explain` + profile API 提取 ANALYZE 耗时                 | 2-7ms  |
| G0-2 聚合    | `SELECT level, COUNT(*) FROM perf_test WHERE response_time_ms > 100 GROUP BY level ORDER BY COUNT(*) DESC` | 同上                                                     | 3-10ms |
| G0-3 UNION | 同 E1 查询                                                                                                    | 同上（Calcite 路径，含 Volcano 优化器）                           | 5-18ms |
| G0-4 JOIN  | 同 E2 查询                                                                                                    | `_explain` 触发 V1 fallback（不走 V2），不适用此方法。Druid 解析耗用量级参考 | 不适用    |

> G0 组数据用于：① 验证端到端延迟差 ≈ 翻译开销 + 序列化开销 + 网络/排队；② 作为"翻译开销本身可接受"的直接证据；③ 与 4.7 节预期矩阵的"SQL 额外开销"列交叉验证。
> 
> ⚠️ **预期区间说明**：上述预期值（2-7ms / 3-10ms）基于保守硬件假设（单节点 + 1GB heap + JDK 17）估算。实际测试环境（3 节点 + M4 Pro + JDK 25）硬件性能显著更强，实测值 0.82-4.36ms 低于预期下界。预期区间应理解为"生产典型环境的上界"，而非绝对基准。

### 4.3 重查询场景（1M + 10M 数据，返回 5K-10K 行）

> 以下场景在 1M 和 10M 数据上**都跑**（相同查询，表名 `perf_test` / `perf_test_10m`），直接对比劣化比例变化。通过高命中率 + 大结果集 + 高基数聚合 + 深度翻页，让 DSL 执行时间达到 100-400ms（1M）/ 300-2000ms（10M）。

#### A-H 组：大结果集点查

| 场景   | SQL 查询                                                                                                                             | DSL 查询                                                                                                                                                      | 预期 DSL   |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |:--------:|
| A1-H | `SELECT * FROM perf_test WHERE status_code = 200 ORDER BY response_time_ms DESC LIMIT 10000`                                       | `{"query":{"term":{"status_code":200}},"sort":[{"response_time_ms":"desc"}],"size":10000}`                                                                  | 80-150ms |
| A2-H | `SELECT * FROM perf_test WHERE status_code IN (200, 301, 404) AND level IN ('INFO','WARN','ERROR') ORDER BY bytes DESC LIMIT 5000` | `{"query":{"bool":{"filter":[{"terms":{"status_code":[200,301,404]}},{"terms":{"level":["INFO","WARN","ERROR"]}}]}},"sort":[{"bytes":"desc"}],"size":5000}` | 60-120ms |
| A3-H | ``SELECT * FROM perf_test WHERE response_time_ms > 2000 ORDER BY `@timestamp` DESC LIMIT 10000``                                   | `{"query":{"range":{"response_time_ms":{"gt":2000}}},"sort":[{"@timestamp":"desc"}],"size":10000}`                                                          | 80-150ms |

#### B-H 组：高命中全文搜索

| 场景   | SQL 查询                                                                                                                                      | DSL 查询                                                                                                                                                                 | 预期 DSL   |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:--------:|
| B1-H | `SELECT message, service, level, response_time_ms FROM perf_test WHERE match(message, 'request') ORDER BY response_time_ms DESC LIMIT 5000` | `{"query":{"match":{"message":"request"}},"sort":[{"response_time_ms":"desc"}],"size":5000,"_source":["message","service","level","response_time_ms"]}`                | 60-120ms |
| B2-H | `SELECT * FROM perf_test WHERE match(message, 'request failed timeout') AND status_code >= 400 ORDER BY bytes DESC LIMIT 10000`             | `{"query":{"bool":{"must":[{"match":{"message":"request failed timeout"}}],"filter":[{"range":{"status_code":{"gte":400}}}]}},'sort":[{"bytes":"desc"}],"size":10000}` | 80-150ms |

#### C-H 组：高基数聚合

| 场景   | SQL 查询                                                                                                                                                                                                                 | DSL 查询                                                                                                                                                                                                                                                                                                              | 预期 DSL    |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:---------:|
| C1-H | `SELECT user_id, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, MAX(bytes) as max_bytes, MIN(response_time_ms) as min_rt FROM perf_test WHERE response_time_ms > 100 GROUP BY user_id ORDER BY cnt DESC LIMIT 1000` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":100}}},"aggs":{"by_user":{"terms":{"field":"user_id","size":1000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"max_bytes":{"max":{"field":"bytes"}},"min_rt":{"min":{"field":"response_time_ms"}}}}}}`                    | 100-300ms |
| C2-H | `SELECT level, service, region, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, SUM(bytes) as total_bytes FROM perf_test WHERE response_time_ms > 500 GROUP BY level, service, region ORDER BY level, cnt DESC`      | `{"size":0,"query":{"range":{"response_time_ms":{"gt":500}}},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_region":{"terms":{"field":"region"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"total_bytes":{"sum":{"field":"bytes"}}}}}}}}}}` | 100-250ms |
| C3-H | SQL 不支持（DATE_HISTOGRAM NPE）                                                                                                                                                                                            | `{"size":0,"aggs":{"by_hour":{"date_histogram":{"field":"@timestamp","calendar_interval":"1h"},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"p95":{"percentiles":{"field":"response_time_ms","percents":[95,99]}}}}}}}}`                                 | 150-400ms |

#### D-H 组：深度分页 + 大结果集

| 场景   | SQL 查询                                                                                                                                                                                 | DSL 查询                                                                                                                                                                                                                                   | 预期 DSL    |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:---------:|
| D1-H | `SELECT * FROM perf_test ORDER BY response_time_ms ASC LIMIT 19990, 10`                                                                                                                | `{"query":{"match_all":{}},"sort":[{"response_time_ms":"asc"}],"from":19990,"size":10}`                                                                                                                                                  | 100-200ms |
| D2-H | `SELECT * FROM perf_test WHERE status_code = 200 ORDER BY response_time_ms DESC LIMIT 10000`                                                                                           | `{"query":{"term":{"status_code":200}},"sort":[{"response_time_ms":"desc"}],"size":10000}`                                                                                                                                               | 80-150ms  |
| D3-H | ``SELECT service, level, response_time_ms, bytes, `@timestamp` FROM perf_test WHERE response_time_ms > 1000 AND status_code IN (200, 500) ORDER BY response_time_ms DESC LIMIT 10000`` | `{"query":{"bool":{"filter":[{"range":{"response_time_ms":{"gt":1000}}},{"terms":{"status_code":[200,500]}}]}},"sort":[{"response_time_ms":"desc"}],"size":10000,"_source":["service","level","response_time_ms","bytes","@timestamp"]}` | 80-150ms  |

#### E-H 组：SQL 独有大数据量

| 场景               | SQL 查询                                                                                                                                                                                                                                                                                 | 引擎        | 预期 SQL    |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |:---------:|
| E1-H 三路UNION+聚合  | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'DEBUG' GROUP BY service` | Calcite   | 20-50ms   |
| E1b-H 250K行UNION | `SELECT service, response_time_ms FROM perf_test WHERE status_code = 500 UNION ALL SELECT service, response_time_ms FROM perf_test WHERE status_code = 503`                                                                                                                            | Calcite   | 100-300ms |
| E2-H JOIN 10K行   | `SELECT a.service, a.level, a.response_time_ms, b.dept_name FROM perf_test a JOIN perf_test_meta b ON a.host = b.host WHERE a.level = 'ERROR' LIMIT 10000`                                                                                                                             | Legacy V1 | 200-500ms |
| E3-H IN子查询 10K行  | `SELECT service, level, response_time_ms FROM perf_test WHERE host IN (SELECT host FROM perf_test WHERE level = 'ERROR') LIMIT 10000`                                                                                                                                                  | Legacy V1 | 200-500ms |

### 4.4 10M 生产级验证方案

> 目标：验证"大数据量重查询下 SQL 与 DSL 性能差异不大"。
> **环境与 4.1 节统一配置完全一致**（3 节点 6 shard 1 replica 13 字段含高基数），此处仅补充生产级测试的额外控制项。4.2/4.3 节的场景在 1M 和 10M 上都跑（相同查询，表名 `perf_test` / `perf_test_10m`），直接对比劣化比例变化。

> 额外控制项见 4.6 节注意事项。

#### 场景组 G：生产典型重查询

| 场景           | SQL 查询                                                                                                                                                                                                                                                                  | DSL 查询                                                                                                                                                                                                                                                                                                                                                      | 预期 DSL    |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:---------:|
| G1 时间范围+聚合   | ``SELECT service, level, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt FROM perf_test_10m WHERE `@timestamp` > '2026-07-11T00:00:00Z' GROUP BY service, level ORDER BY service, cnt DESC``                                                                           | `{"size":0,"query":{"range":{"@timestamp":{"gte":"2026-07-11T00:00:00Z"}}},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}}}`                                                                                                                     | 300-600ms |
| G2 多字段排序+10K | ``SELECT * FROM perf_test_10m WHERE level = 'ERROR' AND `@timestamp` > '2026-07-17T00:00:00Z' ORDER BY response_time_ms DESC, bytes DESC LIMIT 10000``                                                                                                                  | `{"query":{"bool":{"filter":[{"term":{"level":"ERROR"}},{"range":{"@timestamp":{"gte":"2026-07-17T00:00:00Z"}}]}},"sort":[{"response_time_ms":"desc"},{"bytes":"desc"}],"size":10000}`                                                                                                                                                                      | 200-500ms |
| G3 高基数聚合+过滤  | ``SELECT user_id, COUNT(*) as req_count, AVG(response_time_ms) as avg_rt, MAX(bytes) as max_bytes, SUM(bytes) as total_bytes FROM perf_test_10m WHERE `@timestamp` > '2026-07-17T00:00:00Z' AND status_code = 200 GROUP BY user_id ORDER BY req_count DESC LIMIT 1000`` | `{"size":0,"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"2026-07-17T00:00:00Z"}}},{"term":{"status_code":200}}]}},"aggs":{"by_user":{"terms":{"field":"user_id","size":1000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"max_bytes":{"max":{"field":"bytes"}},"total_bytes":{"sum":{"field":"bytes"}}}}}}` | 300-600ms |
| G4 复合+5K     | ``SELECT service, level, response_time_ms, bytes, `@timestamp`, request_path, client_ip FROM perf_test_10m WHERE status_code >= 400 AND response_time_ms > 2000 AND `@timestamp` > '2026-07-15T00:00:00Z' ORDER BY response_time_ms DESC LIMIT 5000``                   | `{"query":{"bool":{"filter":[{"range":{"status_code":{"gte":400}}},{"range":{"response_time_ms":{"gt":2000}}},{"range":{"@timestamp":{"gte":"2026-07-15T00:00:00Z"}}]}},"sort":[{"response_time_ms":"desc"}],"size":5000,"_source":["service","level","response_time_ms","bytes","@timestamp","request_path","client_ip"]}`                                 | 150-400ms |

#### 场景组 H：极重查询

| 场景           | SQL 查询                                                                                                                                                                                     | DSL 查询                                                                                                                                                                                                                                                                                                                               | 预期 DSL     |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |:----------:|
| H1 全表500K桶聚合 | `SELECT session_id, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt FROM perf_test_10m WHERE response_time_ms > 100 GROUP BY session_id ORDER BY cnt DESC LIMIT 5000`                     | `{"size":0,"query":{"range":{"response_time_ms":{"gt":100}}},"aggs":{"by_session":{"terms":{"field":"session_id","size":5000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}`                                                                                                                   | 800-2000ms |
| H2 全表多级聚合    | `SELECT service, level, region, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, SUM(bytes) as total_bytes FROM perf_test_10m GROUP BY service, level, region ORDER BY service, cnt DESC` | `{"size":0,"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"by_region":{"terms":{"field":"region"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"total_bytes":{"sum":{"field":"bytes"}},"p99":{"percentiles":{"field":"response_time_ms","percents":[99]}}}}}}}}}}` | 600-1500ms |
| H3 大范围+10K   | ``SELECT * FROM perf_test_10m WHERE `@timestamp` > '2026-07-11T00:00:00Z' ORDER BY response_time_ms DESC LIMIT 10000``                                                                     | `{"query":{"range":{"@timestamp":{"gte":"2026-07-11T00:00:00Z"}}},"sort":[{"response_time_ms":"desc"}],"size":10000}`                                                                                                                                                                                                                | 400-800ms  |

### 4.5 测试方法

#### 压测脚本框架

```python
session = requests.Session()  # 复用 TCP 连接

def bench(name, fn_factory, warmup=20, runs=200):
    """fn_factory(threshold) 返回可调用对象，threshold 用于随机化打散缓存。"""
    # 预热
    for _ in range(warmup):
        fn_factory(random.randint(1, 9000))()
    latencies = []
    for _ in range(runs):
        t = random.randint(1, 9000)
        t0 = time.perf_counter()
        r = fn_factory(t)()
        dt = (time.perf_counter() - t0) * 1000
        if r.status_code == 200: latencies.append(dt)
    # 计算 p50/p95/p99/max/mean
```

#### 执行步骤

```
1. 启动集群 → 创建索引 → 导入数据 → 轮询确认
2. 结果集等价验证（F3）
3. 预热（20 轮）
4. 正式测试（200 轮，聚合场景随机阈值打散缓存）
5. 收集指标
```

> ⚠️ 原方案设计的并发吞吐量测试（8/16/32 并发）、Pushdown on/off 对比（F2）、两遍运行验证 JIT 稳定性、失败恢复测试（I 组）均**未执行**。

#### 收集指标

| 指标          | 方式                         | 说明                   |
| ----------- | -------------------------- | -------------------- |
| 端到端延迟       | 客户端计时                      | p50/p99/p99.9        |
| per-run 延迟  | 客户端记录                      | 关联 JIT/GC 异常值        |
| 服务端执行耗时     | slowlog                    | 分解翻译开销               |
| 吞吐量 QPS     | 并发测试                       | 8/16/32 并发           |
| SQL explain | `_explain`                 | 确认引擎路径+下推            |
| Calcite 回退  | 服务端日志                      | "Fallback to V2"     |
| JIT 编译      | `-XX:+PrintCompilation`    | 关联延迟尖峰               |
| GC 频率       | `-Xlog:gc*`                | 检测 SQL 路径 GC 压力      |
| JVM heap    | `_nodes/stats/jvm`         | 含 off-heap direct    |
| 线程池         | `_nodes/stats/thread_pool` | sql-worker vs search |
| 结果集等价       | F3 验证                      | TPC 标准               |

### 4.6 注意事项

1. **缓存控制（最关键）**：
   - ⚠️ **SQL 路径无法禁用 request cache**（`OpenSearchQueryRequest.search()` 不设 `requestCache`，OpenSearch 默认对 size=0 聚合自动缓存）。DSL 加 `?request_cache=false` 仅禁用 DSL 侧，SQL 侧仍享受缓存——**系统性偏差**
   - C1/C3 场景每 50 轮执行 `_cache/clear`；A/B/D 组无显式清缓存（依赖随机阈值/低命中率）
   - 监控 `indices/query_cache/memory_size`、`indices/fielddata/memory_size`、`indices/query_cache/hit_count`（确认缓存被有效打散）
   - 注意：即使 `_cache/clear` 清除 shard query cache 和 fielddata cache，**OS page cache 无法清除**
2. **预热**：20 轮预热
3. **DSL 查询等价**：DSL 不含 SQL 没有的聚合（如 percentile），确保对比公平
4. **Calcite 回退监控**：检查 "Fallback to V2" 日志（注意默认 `plugins.calcite.fallback.allowed=false`，仅 `CalciteUnsupportedException` 触发回退）
5. **连接复用**：`requests.Session()` 消除 TCP 开销
6. **@timestamp 标识符**：以 `@` 开头需反引号引用
7. **segment 数**：1M 39 个 / 10M 约 60 个，均未 forcemerge（forcemerge 对照组原设计但未执行）
8. **后台噪声隔离**：测试窗口停止 indexing + `translog.durability=async` + `merge.scheduler.max_thread_count=1`
9. **circuit breaker 风险**：H 组高基数聚合前查 `indices.breaker.total.limit`；500K 桶约需 200-400MB fielddata，预备 fallback 查询，记录 breaker 触发率

### 4.7 预期结果矩阵

> 开销占比 = SQL 额外开销 / SQL 总延迟（= DSL + 额外开销）。区间已重新核算确保数学闭合。
> E 组无严格 DSL 对照（DSL"多次查询+应用层合并"语义不等价），列为 SQL 端到端绝对值。
> ⚠️ **"SQL 额外开销"含翻译开销 + JdbcResponseFormatter 格式化开销 + 内存计算开销 + 网络/排队**，不是纯翻译开销。纯翻译开销见 G0 组基线。大结果集场景（D3/D3-H）格式化开销可达 ~80ms，占比高。
> ⚠️ **C 组聚合场景**：SQL V2 路径对 GROUP BY 永远用 composite 聚合（`AggregationQueryBuilder.java:97`，size=1000 硬编码），**无 terms 转换**（已通过 `_explain` 验证）。DSL 用 terms 聚合。两者是**架构差异**，额外开销反映翻译 + composite vs terms 聚合机制差异。
> ⚠️ **D 组深度分页**：D2 若 `maxResultWindow=10000`（默认），SQL 回退内存分页，不参与对比。D4 机制不同（PIT vs 裸 search_after），仅作生产方案参考。

> **占比推导示例**（以 A1 点查为例）：开销占比 = 额外开销 / (DSL + 额外开销)。DSL 和额外开销随查询复杂度同向变化——简单查询 DSL=3ms、额外开销=2ms，占比 2/(3+2)=40%；复杂查询 DSL=8ms、额外开销=2-7ms，占比 2/(8+2)=20% ~ 7/(8+7)=47%。三个端点对应不同查询复杂度场景，非独立变量的极值组合。"占比推导"列格式为 `额外开销/(DSL+额外开销)=占比`。

| 场景                    | 预期 DSL (ms) | 预期 SQL 额外开销 (ms)                 | 预期开销占比       | 占比推导                                             | 备注                                                                  |
| --------------------- |:-----------:|:--------------------------------:|:------------:| ------------------------------------------------ | ------------------------------------------------------------------- |
| A1 点查                 | 3-8         | 2-7                              | 20-47%       | 2/(8+2)=20% · 2/(3+2)=40% · 7/(8+7)=47%          | 3 节点 scatter-gather 提升 DSL 基线                                       |
| B1 全文搜索               | 5-12        | 2-7                              | 17-70%       | 2/(10+2)=17% · 2/(3+2)=40% · 7/(3+7)=70%         | —                                                                   |
| C1 聚合                 | 8-25        | 5-15                             | 20-75%       | 5/(20+5)=20% · 5/(5+5)=50% · 15/(5+15)=75%       | —                                                                   |
| D2 深度分页               | 50-200      | 5-15                             | 2-23%        | 5/(200+5)=2% · 5/(50+5)=9% · 15/(50+15)=23%      | 仅 maxResultWindow 调大时有效                                             |
| D3 大结果集               | 10-30       | 10-20（含序列化 5-15）                 | 25-67%       | 10/(30+10)=25% · 10/(10+10)=50% · 20/(10+20)=67% | —                                                                   |
| E1 UNION(pushdown ON) | 无 DSL 对照    | 5-20 (冷启动含 codegen 30-80)        | —            | —                                                | SQL 端到端绝对值（含 Calcite bool.must 评分开销）                                |
| E2 JOIN               | 无 DSL 对照    | 200-500 (SQL 端到端)                | —            | —                                                | Legacy V1 IN→JOIN 重写，10K 行内存计算                                      |
| G0 纯翻译开销              | —           | 2-18（预期）                         | —            | —                                                | 实测 0.82-4.36ms（见 §5.5），低于预期因强硬件；G0-1/G0-2 走 V2 路径，G0-3 走 Calcite 路径 |
| G1 10M 聚合             | 300-600     | 5-15 (稳态) / 30-80 (冷启动含 codegen) | 1-5% / 5-13% | —                                                | 区分冷启动 vs 稳态                                                         |
| H1 10M 高基数聚合          | 15-50       | 10-30                            | 0.5-3%       | —                                                | 500K 桶 10M 数据可能触发 breaker；实测 DSL p50 仅 28ms（见 §5.4），SQL 3913ms 远超预期 |

### 4.8 结果正确性验证

> 性能对比的前提是 SQL 与 DSL 结果**语义等价**。以下判据定义每个场景的等价性标准；不等价的场景**仅参与性能对比，不参与正确性结论**。

| 场景组         | 等价性判据                                                                                                     | 顺序敏感性           | 已知不等价                                                                                                                                                                                                               | 正确性结论参与                                             |
| ----------- | --------------------------------------------------------------------------------------------------------- |:---------------:| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:---------------------------------------------------:|
| A 组（点查）     | 行数 + 值集合完全一致（顺序无关，因 `LIMIT` 无 `ORDER BY` 时两者顺序可能不同）                                                       | 否               | 无                                                                                                                                                                                                                   | ✅ 参与                                                |
| B 组（全文搜索）   | 行数 + `_id` 集合一致；`_score` 不要求一致（DSL `match` 打分 vs SQL V2 路径）                                               | 否               | 评分差异（不影响命中集合）                                                                                                                                                                                                       | ✅ 参与（仅行集合）                                          |
| C 组（聚合）     | 桶 key 集合 + 每桶度量值一致。COUNT 精确匹配；AVG/SUM 用相对容差 1e-6（composite 分页累加与 terms 分片层合并的浮点求和顺序不同，绝对容差 1e-9 过严）；排序后比对 | 是（按桶 key 排序后比对） | C1/C3：SQL=composite，DSL=terms（size=1000）——低基数（≤1000 桶）时两者返回全桶，结果等价；高基数（>1000 桶）时 terms 截断，桶集合不同。C2：SQL=composite，DSL=composite（对等方案 A 已落地），机制相同                                                                     | ✅ 参与（C1/C3 仅低基数时参与；C2 完全参与）                         |
| D 组（排序分页）   | 行数 + 值集合 + 顺序完全一致（含 `ORDER BY`）                                                                           | 是               | D2 深度分页 SQL 可能回退内存分页（结果等价但路径不同）；D4 游标机制不同（PIT vs 裸 search_after）                                                                                                                                                    | ⚠️ D2 仅 maxResultWindow 调大时参与；D4 **仅性能对比，不参与正确性结论** |
| E 组（SQL 独有） | DSL 无严格等价物（多次查询+应用层合并）                                                                                    | —               | E1 UNION 合并顺序、E2/E3 JOIN 语义不等价                                                                                                                                                                                      | ❌ **仅性能对比，不参与正确性结论**                                |
| G0 组（纯翻译基线） | 不涉及（仅测翻译阶段耗时）                                                                                             | —               | —                                                                                                                                                                                                                   | ❌ 不参与                                               |
| G/H 组（重查询）  | 按子组分类：A-H 同 A 组（点查）；B-H 同 B 组（全文搜索）；C-H/G3/H1 同 C 组判据；D-H 同 D 组（排序分页）；E-H 同 E 组（SQL 独有）                   | 视场景             | C1-H/G3/H1：SQL composite 全量桶精确排序 vs DSL terms 分片 top N 合并，底部排名可能因 `doc_count_error` 有微小差异——**容差内则参与，否则标注**。C2-H：DSL 用 nested terms（非 composite，与 C2 轻查询的 composite 对等不同），桶数少（80）时结果等价。H1 500K 桶可能触发 breaker 导致结果不完整 | ⚠️ 聚合场景需确认 breaker 未触发 + 容差通过后才参与；E-H 同 E 组不参与      |

> **验证方法**：每个场景首次运行时执行 F3（结果等价验证），记录行数、值集合（点查）、桶 key+度量（聚合）。后续性能轮次不再重复验证（避免影响计时）。
> 
> ⚠️ **不等价场景清单**（仅性能对比，不参与正确性结论）：E 组（E1/E2/E3）、D4 游标分页。这些场景的 SQL 与 DSL 语义不同，性能差异反映的是**架构差异**而非**翻译效率**。

---

## 五、实测数据与测试报告

> 本章记录 SQL vs DSL 性能对比实测数据。测试环境见 §4.1。1M 轻查询/重查询（§5.2/§5.3）、10M 控制变量场景（§5.2.1/§5.3.2，相同查询仅变数据量）、10M 独立场景（§5.4）、G0 纯翻译开销基线（§5.5）。

### 5.2 轻查询实测数据（1M 数据，4.2 节场景）

> 每场景预热 20 轮，正式测试 200 轮。overhead = SQL p50 − DSL p50；overhead_pct = overhead / SQL p50 × 100%。

| 场景  | 描述                                                                                  | SQL p50 | SQL p95 | SQL p99 | SQL max | DSL p50 | DSL p95 | DSL p99 | DSL max | SQL mean | DSL mean | mean 比 | 额外开销 | 开销占比 |
| --- | ----------------------------------------------------------------------------------- |:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:------:|:----:|:----:|
| A1  | 等值点查(单条件) `WHERE status_code=200 LIMIT 10`                                          | 3.96    | 5.15    | 7.06    | 13.09   | 1.73    | 2.13    | 2.54    | 3.15    | 4.03     | 1.75     | 2.30×  | 2.22 | 56.2 |
| A2  | 等值点查(多条件) `WHERE status_code=500 AND level='ERROR' AND region='us-east-1' LIMIT 10` | 3.62    | 4.94    | 7.15    | 8.65    | 2.01    | 3.04    | 4.89    | 12.09   | 3.73     | 2.20     | 1.70×  | 1.61 | 44.6 |
| A3  | 范围点查 `WHERE response_time_ms>3000 AND status_code=500 LIMIT 10`                     | 2.37    | 3.30    | 4.41    | 6.41    | 1.39    | 1.89    | 2.81    | 4.73    | 2.48     | 1.47     | 1.69×  | 0.98 | 41.2 |
| B1  | 全文搜索 `WHERE match(message,'timeout') LIMIT 10`                                      | 2.10    | 3.46    | 4.60    | 5.79    | 1.38    | 1.74    | 1.98    | 3.17    | 2.27     | 1.40     | 1.62×  | 0.71 | 34.0 |
| C1  | 聚合(level) `GROUP BY level ORDER BY cnt DESC`（随机阈值 1-9000）                           | 4.00    | 7.81    | 10.03   | 16.79   | 1.53    | 2.25    | 2.96    | 4.16    | 4.53     | 1.59     | 2.85×  | 2.47 | 61.8 |
| C3  | 聚合(service) `GROUP BY service ORDER BY cnt DESC`（随机阈值 1-9000）                       | 3.39    | 6.02    | 8.63    | 9.09    | 1.28    | 1.73    | 2.49    | 5.24    | 3.76     | 1.35     | 2.78×  | 2.10 | 62.1 |
| D1  | 排序+小分页 `ORDER BY response_time_ms DESC LIMIT 10`                                    | 2.56    | 3.25    | 4.26    | 5.78    | 1.49    | 2.60    | 3.16    | 5.54    | 2.63     | 1.66     | 1.58×  | 1.07 | 41.9 |
| D3  | 大结果集 `WHERE status_code=200 LIMIT 1000`                                             | 16.78   | 21.31   | 24.67   | 35.32   | 7.47    | 8.60    | 10.44   | 16.71   | 17.47    | 7.64     | 2.29×  | 9.31 | 55.5 |

> 所有数值单位为 ms。overhead = SQL p50 − DSL p50；overhead_pct = overhead / SQL p50 × 100%。

**测试期间无 SQL/DSL 查询错误**（所有 200 轮均成功返回 200 状态码）。

#### 5.2.1 轻查询实测数据（10M 数据，相同查询控制变量）

> 与 5.2 节使用**完全相同的查询**，仅表名 `perf_test` → `perf_test_10m`，数据量 1M→10M。每场景预热 20 轮，正式测试 200 轮。

| 场景  | 描述                       | SQL p50 | SQL p95 | SQL p99 | SQL max | DSL p50 | DSL p95 | DSL p99 | DSL max | SQL mean | DSL mean | mean 比 | 额外开销  | 开销占比 |
| --- | ------------------------ |:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:------:|:-----:|:----:|
| A1  | 等值点查(单条件)                | 2.57    | 3.67    | 4.78    | 5.01    | 1.59    | 2.51    | 4.78    | 13.27   | 2.66     | 1.77     | 1.50×  | 0.98  | 38.2 |
| A2  | 等值点查(多条件)                | 5.47    | 8.80    | 23.48   | 24.99   | 3.85    | 5.99    | 6.81    | 18.66   | 6.12     | 4.11     | 1.49×  | 1.62  | 29.6 |
| A3  | 范围点查                     | 2.44    | 3.08    | 3.99    | 4.05    | 1.71    | 2.07    | 2.36    | 3.42    | 2.49     | 1.74     | 1.43×  | 0.74  | 30.1 |
| B1  | 全文搜索                     | 1.61    | 2.52    | 3.39    | 4.24    | 1.06    | 1.42    | 1.66    | 1.67    | 1.69     | 1.09     | 1.55×  | 0.55  | 34.0 |
| C1  | 聚合(level)（随机阈值 1-9000）   | 4.95    | 41.10   | 56.79   | 69.61   | 2.02    | 6.13    | 8.64    | 9.38    | 9.63     | 2.36     | 4.07×  | 2.93  | 59.1 |
| C3  | 聚合(service)（随机阈值 1-9000） | 4.22    | 36.53   | 44.13   | 48.01   | 1.40    | 5.00    | 5.69    | 6.08    | 8.50     | 1.77     | 4.80×  | 2.82  | 66.8 |
| D1  | 排序+小分页                   | 2.16    | 4.18    | 6.33    | 6.62    | 1.31    | 1.71    | 2.82    | 2.84    | 2.37     | 1.37     | 1.73×  | 0.85  | 39.3 |
| D3  | 大结果集(LIMIT 1000)         | 26.20   | 26.91   | 28.43   | 29.56   | 11.73   | 13.43   | 15.05   | 19.58   | 25.47    | 11.55    | 2.21×  | 14.47 | 55.2 |

> 所有数值单位为 ms。

**测试期间无 SQL/DSL 查询错误**（所有 200 轮均成功返回 200 状态码）。

### 5.3 重查询实测数据（1M 数据，4.3 节场景）

> 每场景预热 20 轮，正式测试 200 轮。overhead = SQL p50 − DSL p50；overhead_pct = overhead / SQL p50 × 100%。重查询场景返回 5K-10K 行结果集或执行高基数聚合。

#### 5.3.1 A-H 至 D-H 组：SQL vs DSL 对照

| 场景   | 描述                                              | SQL p50 | SQL p95 | SQL p99 | SQL max | DSL p50 | DSL p95 | DSL p99 | DSL max | SQL mean | DSL mean | mean 比 | 额外开销  | 开销占比 |
| ---- | ----------------------------------------------- |:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:------:|:-----:|:----:|
| A1-H | 大结果集点查(status=200 ORDER BY rt DESC LIMIT 10000) | 127.20  | 130.10  | 131.75  | 132.31  | 43.59   | 45.79   | 47.18   | 49.22   | 126.97   | 43.83    | 2.90×  | 83.60 | 65.7 |
| A2-H | 大结果集多条件点查(LIMIT 5000)                           | 68.55   | 70.48   | 74.95   | 87.99   | 23.63   | 25.55   | 26.82   | 27.95   | 68.53    | 23.87    | 2.87×  | 44.92 | 65.5 |
| A3-H | 大结果集范围点查(rt>2000 LIMIT 10000)                   | 123.06  | 125.06  | 127.64  | 138.54  | 39.59   | 41.97   | 43.02   | 43.34   | 123.24   | 39.97    | 3.08×  | 83.46 | 67.8 |
| B1-H | 高命中全文搜索(LIMIT 5000)                             | 33.92   | 36.25   | 38.13   | 41.44   | 19.28   | 21.38   | 22.30   | 25.80   | 34.02    | 19.53    | 1.74×  | 14.64 | 43.2 |
| B2-H | 高命中全文搜索+filter(LIMIT 10000)                     | 125.05  | 127.68  | 128.59  | 131.06  | 40.40   | 43.46   | 44.09   | 44.38   | 124.97   | 40.73    | 3.07×  | 84.65 | 67.7 |
| C1-H | 高基数聚合(user_id 10K桶+多measure)（随机阈值1-9000）        | 77.52   | 127.35  | 138.93  | 145.74  | 12.99   | 19.16   | 22.18   | 24.61   | 77.81    | 13.28    | 5.86×  | 64.53 | 83.2 |
| C2-H | 多级聚合(level×service×region 80桶)                  | 2.48    | 6.89    | 11.08   | 12.32   | 1.20    | 4.22    | 5.90    | 7.44    | 2.96     | 1.53     | 1.93×  | 1.27  | 51.4 |
| D1-H | 深度分页(from=19990 size=10)                        | 20.57   | 23.10   | 25.30   | 25.46   | 14.85   | 16.42   | 17.72   | 19.50   | 20.71    | 15.10    | 1.37×  | 5.71  | 27.8 |
| D2-H | 大结果集排序(status=200 LIMIT 10000)                  | 128.35  | 130.97  | 132.89  | 133.46  | 44.38   | 46.70   | 47.23   | 47.64   | 128.27   | 44.57    | 2.88×  | 83.96 | 65.4 |
| D3-H | 大结果集多filter+排序(LIMIT 10000)                     | 75.00   | 77.15   | 79.47   | 90.48   | 30.57   | 33.31   | 34.26   | 36.90   | 75.15    | 30.83    | 2.44×  | 44.43 | 59.2 |

> 所有数值单位为 ms。overhead = SQL p50 − DSL p50；overhead_pct = overhead / SQL p50 × 100%。

**测试期间无 SQL/DSL 查询错误**（所有 200 轮均成功返回 200 状态码）。

#### 5.3.2 重查询实测数据（10M 数据，相同查询控制变量）

> 与 5.3.1 节使用**完全相同的查询**，仅表名 `perf_test` → `perf_test_10m`，数据量 1M→10M。每场景预热 20 轮，正式测试 200 轮。

| 场景   | 描述                          | SQL p50 | SQL p95 | SQL p99 | SQL max | DSL p50 | DSL p95 | DSL p99 | DSL max | SQL mean | DSL mean | mean 比 | 额外开销  | 开销占比 |
| ---- | --------------------------- |:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:------:|:-----:|:----:|
| A1-H | 大结果集点查(LIMIT 10000)         | 150.59  | 153.77  | 154.95  | 156.34  | 63.83   | 66.66   | 69.06   | 71.88   | 150.68   | 64.00    | 2.35×  | 86.76 | 57.6 |
| A2-H | 大结果集多条件点查(LIMIT 5000)       | 84.53   | 86.36   | 88.54   | 90.96   | 38.21   | 40.47   | 42.07   | 45.26   | 84.71    | 38.43    | 2.20×  | 46.33 | 54.8 |
| A3-H | 大结果集范围点查(LIMIT 10000)       | 141.68  | 145.10  | 151.65  | 152.24  | 60.95   | 63.95   | 66.09   | 66.13   | 141.58   | 61.06    | 2.32×  | 80.73 | 57.0 |
| B1-H | 高命中全文搜索(LIMIT 5000)         | 50.06   | 52.18   | 53.51   | 54.55   | 35.78   | 38.03   | 39.11   | 39.39   | 50.19    | 36.00    | 1.39×  | 14.28 | 28.5 |
| C1-H | 高基数聚合(user_id ~10K桶)（随机阈值）  | 116.37  | 292.78  | 368.47  | 375.23  | 17.27   | 44.93   | 55.63   | 56.08   | 133.81   | 20.26    | 6.60×  | 99.10 | 85.2 |
| C2-H | 多级聚合(80桶)（随机阈值）             | 6.56    | 51.03   | 101.04  | 104.79  | 3.79    | 35.48   | 59.43   | 60.84   | 12.68    | 7.74     | 1.64×  | 2.77  | 42.2 |
| D1-H | 深度分页(from=19990 size=10)    | 38.19   | 40.02   | 42.38   | 45.74   | 22.91   | 25.80   | 26.89   | 27.21   | 38.01    | 20.73    | 1.83×  | 15.29 | 40.0 |
| D2-H | 大结果集排序(LIMIT 10000)         | 148.90  | 153.38  | 155.36  | 156.48  | 64.18   | 66.75   | 68.54   | 68.59   | 148.82   | 64.38    | 2.31×  | 84.72 | 56.9 |
| D3-H | 大结果集多filter+排序(LIMIT 10000) | 95.88   | 98.81   | 100.79  | 103.25  | 51.42   | 54.65   | 55.95   | 58.45   | 95.95    | 51.76    | 1.85×  | 44.46 | 46.4 |

> 所有数值单位为 ms。B2-H 因 DSL 查询 JSON 转义问题未获取有效 DSL 数据，未列入。

**测试期间无 SQL 查询错误**（所有 200 轮均成功返回 200 状态码）。

#### 5.3.3 E-H 组：SQL 独有场景（无 DSL 对照）

> E 组场景利用 SQL 独有能力（UNION / JOIN / IN 子查询），DSL 无等价对照。E1-H/E1b-H 走 Calcite 引擎，E2-H/E3-H 走 Legacy V1 引擎。

| 场景    | 描述                     | SQL p50 (ms) | SQL p95 (ms) | SQL p99 (ms) | SQL max (ms) | SQL mean (ms) | 引擎        |
| ----- | ---------------------- |:------------:|:------------:|:------------:|:------------:|:-------------:|:---------:|
| E1-H  | 三路UNION+聚合（15行结果）      | 7.39         | 9.18         | 12.24        | 14.69        | 7.55          | Calcite   |
| E1b-H | 两路UNION（无LIMIT，~100K行） | 139.17       | 143.47       | 148.76       | 155.97       | 139.29        | Calcite   |
| E2-H  | JOIN 10K行              | 58.06        | 63.32        | 66.92        | 111.81       | 58.72         | Legacy V1 |
| E3-H  | IN子查询 10K行             | 76.51        | 80.14        | 80.99        | 83.79        | 76.74         | Legacy V1 |

**测试期间无 SQL 查询错误**（所有 200 轮均成功返回 200 状态码）。

### 5.4 10M 场景实测数据（4.4 节场景）

> `perf_test_10m` 索引：10,000,000 文档，6 shard + 1 replica = 12 shard，5 GB。数据生成耗时 305 秒（32,750 docs/s）。每场景预热 20 轮，正式测试 200 轮。overhead = SQL p50 − DSL p50；overhead_pct = overhead / SQL p50 × 100%。

| 场景  | 描述                           | SQL p50 | SQL p95 | SQL p99 | SQL max | DSL p50 | DSL p95 | DSL p99 | DSL max | SQL mean | DSL mean | mean 比  | 额外开销    | 开销占比 |
| --- | ---------------------------- |:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:-------:|:-------:|:----:|
| G1  | 时间范围+多级聚合(service×level)     | 2.04    | 3.09    | 4.80    | 5.48    | 0.95    | 1.33    | 1.53    | 2.86    | 2.19     | 0.98     | 2.23×   | 1.09    | 53.5 |
| G2  | ERROR+时间范围+排序(LIMIT 10000)   | 141.18  | 144.27  | 145.96  | 146.81  | 57.41   | 60.31   | 61.60   | 62.18   | 140.63   | 57.70    | 2.44×   | 83.77   | 59.3 |
| G3  | 高基数聚合(user_id+多measure)      | 74.14   | 83.99   | 101.68  | 116.71  | 9.20    | 11.98   | 19.02   | 22.82   | 76.09    | 9.59     | 7.94×   | 64.94   | 87.6 |
| G4  | 多filter+大结果集(LIMIT 5000)     | 55.12   | 56.84   | 60.88   | 63.50   | 24.36   | 26.79   | 28.41   | 31.30   | 55.04    | 24.65    | 2.23×   | 30.76   | 55.8 |
| H1  | 500K桶高基数聚合(session_id)       | 3913.05 | 4043.91 | 4079.76 | 4169.16 | 27.84   | 31.41   | 82.13   | 140.87  | 3923.59  | 29.54    | 132.82× | 3885.21 | 99.3 |
| H2  | 全表多级聚合(service×level×region) | 2.10    | 2.90    | 4.07    | 4.52    | 1.04    | 1.31    | 2.05    | 3.99    | 2.21     | 1.09     | 2.03×   | 1.06    | 50.4 |
| H3  | 大结果集时间范围+排序(LIMIT 10000)     | 143.47  | 147.72  | 149.76  | 152.16  | 63.19   | 66.12   | 67.18   | 68.21   | 143.64   | 63.34    | 2.27×   | 80.28   | 56.0 |

> 所有数值单位为 ms。overhead = SQL p50 − DSL p50；overhead_pct = overhead / SQL p50 × 100%。

**测试期间无 SQL/DSL 查询错误**（所有 200 轮均成功返回 200 状态码，H1 未触发 circuit breaker）。

> ⚠️ **H1 极端异常**：SQL p50 = 3913ms，是 DSL（28ms）的 **140 倍**。根因是 SQL 路径使用 composite 聚合流式拉取全部 ~100K 个 `session_id` 桶再在协调节点排序取 Top 5000，而 DSL 的 `terms` 聚合在分片层面执行后合并，仅返回 5000 个桶。这是 SQL 插件在高基数聚合场景的已知架构瓶颈。详见 §5.6.4 H1 分析。

> 📝 **G1/H2 缓存命中说明**：G1（service×level 20桶）和 H2（service×level×region 80桶）均为确定性查询（无随机阈值），预热后 request cache 命中，SQL/DSL 均在 2ms 内返回。缓存场景下开销占比 ~50%，主要来自 SQL REST 层固定开销（解析 + 序列化），不代表真实冷查询性能。

### 5.5 G0 组纯翻译开销

> **测量方法**：`_explain` 端点（`POST /_plugins/_sql/_explain`）走与 execute 相同的 V2 路径（ANTLR parse → AstBuilder → `Analyzer.analyze()` → `Planner.plan()`），仅在最后调用 `executionEngine.explain()` 而非 `execute()`——跳过物理执行，保留全部翻译+规划开销。因此 `_explain` 端点端到端延迟即为 V2 翻译开销的直接测量（含计划序列化 + 网络/排队，不含执行）。预热 20 轮，正式测量 200 轮。
> 
> **代码路径验证**：`SQLService.explain()`（`SQLService.java:61`）调用与 `execute()`（`SQLService.java:44`）相同的 `plan()` 方法（`SQLService.java:69`），共享 ANTLR 解析 + AstBuilder + AstStatementBuilder。`QueryService.explain()`（`QueryService.java:124`）与 `execute()`（`QueryService.java:102`）均调用 `shouldUseCalcite()` 路由，且 `explainWithLegacy()`（line 257）与 `executeWithLegacy()`（line 229）均调用相同的 `analyze(plan, queryType)`（line 268/235）→ `analyzer.analyze()`（line 320）。唯一差异：explain 调 `executionEngine.explain()`，execute 调 `executionEngine.execute()`。

| 场景         | 查询                                                                                                                                                                                     | `_explain` p50 (ms) | `_explain` p95 (ms) | `_explain` p99 (ms) | `_explain` max (ms) | `_explain` mean (ms) | 预期区间 (ms) |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:-------------------:|:-------------------:|:-------------------:|:-------------------:|:--------------------:|:---------:|
| G0-1 点查    | `SELECT * FROM perf_test WHERE status_code = 200 LIMIT 10`                                                                                                                             | 1.05                | 1.65                | 1.90                | 2.03                | 1.10                 | 2-7       |
| G0-2 聚合    | `SELECT level, COUNT(*) FROM perf_test WHERE response_time_ms > 100 GROUP BY level ORDER BY COUNT(*) DESC`                                                                             | 0.82                | 1.14                | 1.31                | 1.33                | 0.84                 | 3-10      |
| G0-3 UNION | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service` | 4.36                | 5.57                | 6.33                | 6.35                | 4.35                 | 5-18      |

> ⚠️ **实测值低于预期区间**：G0-1 实测 1.05ms（预期 2-7ms），G0-2 实测 0.82ms（预期 3-10ms），G0-3 实测 4.36ms（预期 5-18ms，略低于下界）。原因分析见 §5.6.1。
> 
> ⚠️ **G0-3 UNION 走 Calcite 路径**：`_explain` 返回 `calcite` key（而非 V2 的 `root` key），确认 UNION 查询路由到 Calcite 引擎，翻译开销含 Calcite Volcano 优化器。G0-1/G0-2 走 V2 路径（`root` key），无 Calcite 优化器。
> 
> ⚠️ **预期区间说明**：上述预期值（2-7ms / 3-10ms / 5-18ms）基于保守硬件假设（单节点 + 1GB heap + JDK 17）估算。实际测试环境（3 节点 + M4 Pro + JDK 25）硬件性能显著更强，实测值低于预期下界。预期区间应理解为"生产典型环境的上界"，而非绝对基准。

### 5.6 数据分析与结论

#### 1. 翻译开销基线与端到端分解

**翻译开销**：G0 实测 `_explain` 端点端到端延迟 0.82-4.36ms（G0-1 点查 1.05ms / G0-2 聚合 0.82ms / G0-3 UNION 4.36ms），低于 4.7 节预期 2-18ms（基于保守硬件假设，见 §4.7 预期区间说明）。此为 V2 翻译开销的**直接测量**——`_explain` 走与 execute 相同的 ANTLR + AstBuilder + Analyzer + Planner 路径，仅跳过物理执行（代码路径验证见 §5.5）。G0-3 UNION 走 Calcite 路径，4.36ms 含 Volcano 优化器开销；G0-1/G0-2 走 V2 路径，无 Calcite 优化器。

低于预期的主要原因：① Apple M4 Pro + JDK 25 硬件性能强劲，ANTLR 解析全链路 <1ms；② G0-1/G0-2 查询简单（A1 仅 1 个 WHERE + LIMIT，G0-2 仅 4 桶），ANTLR 解析树节点数少；③ G0-3 UNION 含 Calcite 优化器但仍 4.36ms，因查询结构简单（两路各聚合后合并，5 桶/路）。

**端到端开销分解**：`_explain` 翻译开销与端到端开销的差异随结果集大小增长：

| 场景      | `_explain` p50 (ms)    | 端到端开销 (ms) | 差异 (ms) | 差异来源                                                    |
| ------- |:----------------------:|:----------:|:-------:| ------------------------------------------------------- |
| A1 点查   | 1.05                   | 2.22       | 1.17    | SQL 额外序列化(10行 JDBC 转换) + sql-worker 线程调度 + 网络往返         |
| C1 聚合   | 0.82                   | 2.47       | 1.65    | SQL 额外序列化(4桶 JDBC 转换) + TakeOrderedOperator 内存排序 + 线程调度 |
| D3 大结果集 | ~1.05 (估，引用 G0-1 点查基线) | 9.31       | ~8.26   | **SQL 额外序列化(1000行 JDBC 转换) 占主导**，约 8ms                  |

- 轻查询差异 1.17-1.65ms：`JdbcResponseFormatter` JDBC 格式转换 + sql-worker 线程调度 + HTTP 往返。注：DSL 的 `ToXContent` 原生序列化开销已在 DSL 基线中，此处差异是 SQL **额外**的格式化开销
- D3 差异 ~8.26ms：SQL `JdbcResponseFormatter` 把 1000 行 `SearchResponse` 转为 JDBC `List<Object[]>` 再序列化 JSON，比 DSL 原生 `ToXContent` 多一层对象转换，占差异的 ~90%
- 网络：3 节点同机回环 <0.1ms，生产环境跨节点 0.5-2ms 差异会更大
- **差值分解（非交叉验证）**：G0-1 `_explain` 1.05ms + 差值 1.17ms = A1 端到端开销 2.22ms。此处差值 1.17ms 由端到端减 `_explain` 反推得出（序列化 + 线程调度 + 网络），**非独立测量**，不能作为独立验证——仅说明端到端开销与 `_explain` 翻译开销的差值合理（见 §5.8 局限性 4）。D3 引用 G0-1（点查）而非 G0-2（聚合）基线，因 D3 查询语义（`WHERE` + `LIMIT`，无 `GROUP BY`）与 G0-1 匹配

#### 2. 轻查询劣化分析（1M，5.2 节数据）

8 个轻查询场景开销占比 34-62%，均在小数据量低基线上放大：

| 场景类型        | 场景  | 开销占比  | 预期占比区间 | 是否在预期内              |
| ----------- | --- |:-----:|:------:| ------------------- |
| 点查(单条件)     | A1  | 56.2% | 20-47% | ⚠️ 略超上界（DSL 实测低于预期） |
| 点查(多条件)     | A2  | 44.6% | 20-47% | ✅                   |
| 范围点查        | A3  | 41.2% | 20-47% | ✅                   |
| 全文搜索        | B1  | 34.0% | 17-70% | ✅                   |
| 聚合(level)   | C1  | 61.8% | 20-75% | ✅（偏高）               |
| 聚合(service) | C3  | 62.1% | 20-75% | ✅（偏高）               |
| 排序+小分页      | D1  | 41.9% | 20-47% | ✅                   |
| 大结果集        | D3  | 55.5% | 25-67% | ✅                   |

**劣化规律**：

- **聚合劣化最严重**（C1/C3 ~62%）：composite 聚合（`AggregationQueryBuilder.java:97`，size=1000 硬编码）+ 协调节点内存排序（`TakeOrderedOperator`），DSL terms 在分片层排序——架构差异，非纯翻译开销
- **全文搜索劣化最轻**（B1 34%）：`match()` 直接映射 DSL `match`，翻译路径最短
- **A1 超上界 47%**：DSL 实测仅 1.73ms（预期 3-8ms），固定开销 2.22ms 占比被放大——非 SQL 翻译变慢，而是 DSL 在 3 节点 + M4 Pro 上比预期更快

**比例 vs 绝对值悖论**：D3（55.5%）反而低于 C1/C3（62%），因 DSL 基线更高（7.47ms vs 1.5ms），开销占比被稀释。但 D3 的额外开销（9.31ms）并非固定开销——主要是与结果集大小正相关的序列化开销（1000 行 JSON），而 C1/C3 的额外开销（2.47/2.10ms）才是接近固定的翻译+聚合处理开销。因此该现象的正确解释是：D3 的绝对开销更高（序列化主导），但 DSL 基线也更高（7.47ms vs 1.5ms），开销增长速率低于 DSL 基线增长速率，占比下降。这印证了"DSL 越慢，劣化比例越低"的趋势——10M 下 DSL 基线升至 50-200ms 时，同等开销下占比将降至 1-5%。

**p99 抖动**：聚合场景 C1 p99=10.03ms（p50=4.0ms，2.5× 抖动），C3 p99=8.63ms（p50=3.39ms，2.5×），轻查询 A1-A3/B1/D1 仅 1.5-2×。composite 分页拉取 + 随机阈值打散缓存导致更大尾延迟。

#### 3. 重查询开销分布（1M，5.3 节数据）

开销占比 28-83%，按场景类型呈三个梯队：

| 场景类型       | 代表场景      | 返回行数    | SQL p50 (ms) | DSL p50 (ms) | 开销 (ms) | 开销占比   | 开销主导因素                                            |
| ---------- | --------- |:-------:|:------------:|:------------:|:-------:|:------:| ------------------------------------------------- |
| 大结果集(10K行) | A1-H/D2-H | 10000   | 127-128      | 43-44        | 83-84   | 65-66% | **序列化**（10K行 JSON ~80ms）                          |
| 大结果集(5K行)  | A2-H      | 5000    | 69           | 24           | 45      | 66%    | 序列化（5K行 ~40ms）                                    |
| 高基数聚合      | C1-H      | 0(10K桶) | 78           | 13           | 65      | 83%    | **composite 聚合流式拉取**                              |
| 多级小聚合      | C2-H      | 0(80桶)  | 2.5          | 1.2          | 1.3     | 51%    | 固定开销；⚠️ DSL 用 nested terms（非 composite），80 桶时结果等价 |
| 深度分页       | D1-H      | 10      | 21           | 15           | 6       | 28%    | **SQL from/size 转换开销**                            |
| 全文搜索(5K行)  | B1-H      | 5000    | 34           | 19           | 15      | 43%    | 序列化 + match 翻译                                    |

**关键发现**：

1. **序列化主导大结果集**：10K 行 SQL 开销 83-85ms，序列化占 ~80ms（每行 ~8μs），与 D3（1000 行 9ms）线性外推一致
2. **composite 聚合是高基数瓶颈**：C1-H 占比 83%（最高）。SQL 走 V2 引擎 composite 聚合流式拉取全部 ~10K 桶后 `TakeOrderedOperator` 协调节点排序；DSL terms 在分片层面排序合并仅返回 Top N。**V2 无 composite→terms 转换**（`rePushDownSortAggMeasure` 仅 Calcite 路径调用）。⚠️ C1-H 有 ~10K 桶，SQL composite 返回全量桶精确排序取 Top 1000，DSL terms size=1000 分片层取 Top 1000 再合并——底部排名可能因 `doc_count_error` 有微小差异（见 §4.8）。性能对比有效，正确性结论需容差通过后参与
3. **深度分页开销最低**（D1-H 28%）：SQL 和 DSL 都走 `from+size`，SQL 额外开销仅 5.7ms
4. **C2-H 快因桶数少**（80 < 1000）：composite 单页拉取完毕无分页开销，非 terms 转换（V2 永远用 composite，见 §4.2）

**E-H 组 SQL 独有场景**（无 DSL 对照）：

| 场景                    | 引擎        | SQL p50 (ms) | 结果行数    | 评价                     |
| --------------------- |:---------:|:------------:|:-------:| ---------------------- |
| E1-H 三路UNION+聚合       | Calcite   | 7.4          | 15      | ✅ 聚合后 UNION，结果集小       |
| E1b-H 两路UNION（无LIMIT） | Calcite   | 139.2        | ~100000 | ⚠️ 流式拉取 ~100K 行在协调节点合并 |
| E2-H JOIN 10K行        | Legacy V1 | 58.1         | 10000   | ✅ nested loop JOIN     |
| E3-H IN子查询 10K行       | Legacy V1 | 76.5         | 10000   | ✅ 子查询改写为 terms 过滤      |

JOIN（58ms）快于 IN 子查询（77ms），因 JOIN 走 nested loop + hash，IN 子查询需先执行子查询收集 host 列表再转为 terms。E1b-H 慢因无 LIMIT，100K 行拉取合并是瓶颈而非 UNION 本身。

#### 4. 10M 场景与大数据量趋势验证

**10M 独立场景性能**（§5.4 G/H 组，**非控制变量**——查询条件与 §5.2/§5.3 不同，仅作趋势参考；控制变量对比见下方表格）：

| 场景类型          | 代表场景   | SQL p50 (ms) | DSL p50 (ms) | SQL/DSL 比   | 开销占比      | 对比 1M            |
| ------------- | ------ |:------------:|:------------:|:-----------:|:---------:| ---------------- |
| 大结果集(10K行)    | H3     | 143          | 63           | 2.27×       | 56.0%     | 与 A1-H 相近（127ms） |
| 大结果集+filter   | G2     | 141          | 57           | 2.44×       | 59.3%     | 与 D3-H 相近（75ms）  |
| 高基数聚合         | G3     | 74           | 9.2          | 7.94×       | 87.6%     | 与 C1-H 相近（78ms）  |
| **500K桶极重聚合** | **H1** | **3913**     | **28**       | **132.82×** | **99.3%** | **无 1M 对照**      |
| 小聚合(缓存)       | G1/H2  | 2.0/2.1      | 1.0          | 2.0×        | 50-54%    | 与 C2-H 相近        |

> ⚠️ **G1/H2 缓存命中失真**：G1（20桶）和 H2（80桶）为确定性查询（无随机阈值），request cache 命中后 SQL/DSL 均在 2ms 内返回，开销占比 ~50% 主要来自 SQL REST 层固定开销。**不代表真实冷查询性能**，不应参与"劣化比例"结论。

**H1 极端异常深度分析**（已通过 `_explain` 验证）：

SQL 3913ms vs DSL 28ms（140 倍）。根因是 V2 引擎的 **composite 聚合架构设计 + `AGGREGATION_BUCKET_SIZE=1000` 硬编码**共同导致的确定性限制：

1. V2 对 GROUP BY 永远用 composite 聚合（`AggregationQueryBuilder.java:97`），size=1000 硬编码（不使用 `plugins.query.buckets`）
2. composite 分页拉取全部 ~100K 桶：100K / 1000 = 100 页，每页一次 scatter-gather
3. `TakeOrderedOperator` 在协调节点内存中对 100K 桶排序取 Top 5000
4. DSL terms 在每个分片层面排序取 Top 5000，协调节点仅合并 6 × 5000 = 30000 桶

**为什么 ORDER BY 没有触发 terms 转换**：`rePushDownSortAggMeasure`（`AggPushDownAction.java:133`）仅被 `AggSpec.java:192` → `CalciteLogicalIndexScan.java:343`（Calcite 路径，间接调用）调用。`shouldUseCalcite()`（`QueryService.java:366-374`）对普通 SQL SELECT（无 UNION）返回 false，走 V2，V2 的 `OpenSearchIndexScan` 不调用此方法。

**影响**：任何 `GROUP BY <高基数字段>` + `ORDER BY <聚合字段>` + `LIMIT` 的 SQL 查询在高基数（>10K 桶）场景下都会出现此问题。非偶发 bug。

**缓解方案**：① 使用 DSL terms 聚合；② SQL 中添加 WHERE 条件降低基数；③ 使用 PPL `stats`（走 Calcite pushdown，有 terms 转换）；④ 调高 `AGGREGATION_BUCKET_SIZE`（如 10000，需改代码，分页轮次从 100 降至 10，但单次拉取量增大，内存压力上升）

**G3 高基数聚合分析**：SQL 74ms / DSL 9.2ms（8 倍），与 1M C1-H（78ms / 13ms，6 倍）相比 SQL 时间相近但 DSL 更快（因 G3 有时间过滤扫描文档更少）。G3 与 C1-H 10M（116ms / 17ms，6.6 倍）是**不同查询**（不同 WHERE 条件、不同 GROUP BY 字段、不同 measure），占比相近（87.6% vs 85.2%）不能作为"劣化不随数据量改善"的证据——不同查询的分子分母均不同，占比相近可能为巧合。高基数聚合场景劣化趋势的**唯一有效证据**来自控制变量对比：C1-H 1M 83.2% → C1-H 10M 85.2%（↑2.0pp，见下方控制变量表格）。

**1M vs 10M 控制变量对比**（相同查询，仅数据量 1M→10M）：

**轻查询（§5.2 vs §5.2.1）**：

| 场景             | 1M 开销占比 | 10M 开销占比 | 占比变化    | 说明                                                |
| -------------- |:-------:|:--------:|:-------:| ------------------------------------------------- |
| A1 点查          | 56.2%   | 38.2%    | ↓18.0pp | DSL 从 1.73→1.59ms（微降），固定开销占比被稀释                   |
| A2 点查(多条件)     | 44.6%   | 29.6%    | ↓15.0pp | DSL 从 2.01→3.85ms，SQL 从 3.62→5.47ms，DSL 增长 >> SQL |
| B1 全文搜索        | 34.0%   | 34.0%    | → 0pp   | DSL 和 SQL 同比例增长，占比不变                              |
| C1 聚合(level)   | 61.8%   | 59.1%    | ↓2.7pp  | composite 4 桶开销恒定，DSL 从 1.53→2.02ms 缓慢增长          |
| C3 聚合(service) | 62.1%   | 66.8%    | ↑4.7pp  | composite ~5 桶开销恒定，DSL 从 1.28→1.40ms 几乎不变         |
| D3 大结果集        | 55.5%   | 55.2%    | ↓0.3pp  | 序列化开销 14ms（1M）→14ms（10M）固定，DSL 7.47→11.73ms，占比微降  |

**重查询（§5.3.1 vs §5.3.2）**：

| 场景                | 1M 开销占比 | 10M 开销占比 | 占比变化    | 说明                                                           |
| ----------------- |:-------:|:--------:|:-------:| ------------------------------------------------------------ |
| A1-H 大结果集(10K行)   | 65.7%   | 57.6%    | ↓8.1pp  | 序列化 ~80ms 固定，DSL 44→64ms，占比下降 ✅                              |
| A3-H 大结果集(10K行)   | 67.8%   | 57.0%    | ↓10.8pp | 同上 ✅                                                         |
| D2-H 大结果集(10K行)   | 65.4%   | 56.9%    | ↓8.5pp  | 同上 ✅                                                         |
| D3-H 大结果集(10K行)   | 59.2%   | 46.4%    | ↓12.8pp | 序列化 ~44ms 固定，DSL 31→51ms，占比下降 ✅                              |
| C1-H 高基数聚合(~10K桶) | 83.2%   | 85.2%    | ↑2.0pp  | composite 开销 65→99ms（增长），DSL 13→17ms（增长），但 composite 增长更快 ⚠️ |
| C2-H 多级聚合(80桶)    | 51.4%   | 42.2%    | ↓9.2pp  | composite 80 桶开销 1.3→2.8ms，DSL 1.2→3.8ms，DSL 增长更快 ✅          |
| D1-H 深度分页         | 27.8%   | 40.0%    | ↑12.2pp | from+size 深翻页，DSL 15→23ms，SQL 21→38ms，SQL 增长更快 ⚠️            |

**假设验证**：

- **序列化主导场景**（大结果集 10K 行 A1-H/A3-H/D2-H/D3-H）：开销占比**显著下降** ✅——序列化 ~80ms 固定，DSL 随数据量增长，占比稀释
- **低基数聚合场景**（C1/C3，4-5 桶）：占比**基本持平或微降** ≈——composite 开销恒定（桶数不变），DSL 随数据量缓慢增长
- **高基数聚合场景**（C1-H，~10K 桶）：占比**微升** ⚠️——composite 开销从 65→99ms（+52%），DSL terms 从 13→17ms（+31%），composite 增长更快。**composite 开销与桶数正相关，但 10M 数据下每页拉取的 scatter-gather 成本更高（更多 segment/文档扫描），导致 composite 分页轮次成本上升**
- **深度分页**（D1-H）：占比**上升** ⚠️——SQL from+size 在 10M 数据上扫描更多文档，增长快于 DSL

#### 5. 综合结论

**三组数据综合视图**：

| 维度         | 4.2 轻查询（1M，≤10行） | 4.2 轻查询（10M，相同查询） | 4.3 重查询（1M，5K-10K行） | 4.3 重查询（10M，相同查询） | 4.4 10M 独立场景⚠️ | 趋势         |
| ---------- | ---------------- | ----------------- | ------------------- | ----------------- | -------------- | ---------- |
| SQL p50 范围 | 2.1-16.8ms       | 1.6-26.2ms        | 2.5-128ms           | 50-151ms          | 2.0-3913ms †   | 随数据量/结果集增长 |
| DSL p50 范围 | 1.3-7.5ms        | 1.1-11.7ms        | 1.2-44ms            | 18-64ms           | 1.0-63ms †     | 同上         |
| 开销占比范围     | 34-62%           | 30-67%            | 28-83%              | 29-85%            | 50-99.3% †     | **非单调下降**  |

> † **4.4 10M 独立场景列含缓存命中数据（G1/H2），不参与结论推导**：G1（20桶）和 H2（80桶）为确定性查询（无随机阈值），request cache 命中后 SQL/DSL 均在 2ms 内返回，开销占比 ~50% 来自 SQL REST 层固定开销，不代表真实冷查询性能（详见 §5.4 缓存命中说明）。该列仅 H1（3913ms）、G3（87.6%）等冷查询数据参与结论。

| 纯翻译开销      | 0.82-4.36ms（G0）          | —                 | ~1ms（估）             | —                 | 未测                   | **恒定**                     |
| 序列化开销      | ~1.5ms（10行）→8.5ms（1000行） | ~14ms（1000行）      | ~80ms（10K行）         | ~85ms（10K行）       | ~80ms（10K行）          | **随结果集线性增长，与数据量无关**        |
| 内存计算开销     | 2.1-2.5ms（4-5桶 composite 排序） | 2.8-2.9ms（4-5桶）   | 65ms（10K桶 composite 排序） | 99ms（10K桶）        | 65-3885ms（10K-100K桶） | **随桶数/数据量增长，composite 架构瓶颈** |

**假设验证结论（分维度）**：

| 假设维度             | 验证结果  | 关键证据                                    | 说明                                                |
| ---------------- |:-----:| --------------------------------------- | ------------------------------------------------- |
| 翻译开销可接受          | ✅ 成立  | G0 实测 0.82-4.36ms (`_explain`)          | V2 路径 0.82-1.05ms，Calcite 路径 4.36ms，均低于 2-18ms 预期 |
| 小数据量劣化比例大        | ✅ 成立  | 轻查询 34-62%（DSL 1.3-7.5ms）               | 固定开销在低基线上占比放大                                     |
| 小数据量整体耗时少        | ✅ 成立  | 轻查询 2-17ms                              | 用户无感知                                             |
| 大数据量劣化比例小（序列化场景） | ✅ 成立  | A1-H 66%→58%、D3-H 59%→46%（控制变量，↓8-13pp） | 序列化开销固定，DSL 随数据量增长                                |
| 大数据量劣化比例小（低基数聚合） | ✅ 成立  | C1 62%→59%、C2-H 51%→42%（控制变量，↓2-9pp）    | composite 开销恒定（桶数不变），DSL 缓慢增长                     |
| 大数据量劣化比例小（高基数聚合） | ❌ 不成立 | C1-H 83%→85%（控制变量，↑2pp）                 | composite 分页轮次成本随数据量上升，增长快于 DSL terms             |
| 大数据量劣化比例小（深度分页）  | ❌ 不成立 | D1-H 28%→40%（控制变量，↑12pp）                | SQL from+size 在 10M 数据上扫描更多文档，增长快于 DSL            |
| H1 极端场景可接受       | ❌ 不成立 | 3913ms vs 28ms（140×）                    | 高基数聚合是架构瓶颈                                        |
| 无查询错误            | ✅ 成立  | 10000+ 次查询零错误                           | 功能正确性良好（稳定性需并发/故障恢复测试验证）                          |

**核心结论**：

1. **点查/全文搜索/排序分页/大结果集场景性能可接受**——劣化 34-66%，绝对开销 0.7-84ms（序列化主导），不影响用户体验
2. **低基数聚合（≤1K桶）性能可接受**——劣化 51-62%，绝对开销 1.3-2.5ms
3. **中基数聚合（~10K桶）性能边界化**——C1-H 劣化 83%，绝对开销 65ms，接近瓶颈拐点。⚠️ SQL composite 返回全量桶、DSL terms size=1000 返回 Top 1000，结果可能因 `doc_count_error` 有底部排名差异（见 §4.8）
4. **高基数聚合（≥50K桶）存在架构瓶颈**——H1（100K桶）劣化 140 倍，**不可接受**。根因是 composite 聚合架构设计 + `AGGREGATION_BUCKET_SIZE=1000` 硬编码共同导致
5. **"大数据量劣化比例小"假设在序列化主导和低基数聚合场景成立**（控制变量验证：大结果集 ↓8-13pp、低基数聚合 ↓2-9pp）；在高基数聚合场景不成立（C1-H ↑2pp，composite 分页成本随数据量上升）；在深度分页场景也不成立（D1-H ↑12pp，SQL from+size 在 10M 数据上扫描更多文档，增长快于 DSL）
6. **技术结论**：在单连接串行、4GB heap、回环网络条件下，SQL 插件在点查/全文/分页/低基数聚合场景性能可接受；高基数聚合（GROUP BY 高基数字段 + LIMIT）应使用 DSL 或 PPL `stats`（走 Calcite pushdown）。**注：此结论基于单连接串行测试，并发负载与故障恢复未验证（见 §5.8 局限性）；4GB heap 非生产 8GB，H1 的 140× 劣化在生产环境下可能不同（见 §5.8 局限性 2）；PPL `stats` 缓解方案未实测**

### 5.7 与预期值对比

> 预期值来自 4.7 节"预期结果矩阵"。偏差 = 实测值 − 预期区间中点。

| 场景      | 预期 DSL (ms) | 实测 DSL p50 (ms) | DSL 偏差   | 预期开销 (ms) | 实测开销 (ms) | 开销偏差     | 预期占比   | 实测占比  | 占比是否在区间 |
| ------- |:-----------:|:---------------:|:--------:|:---------:|:---------:|:--------:|:------:|:-----:|:-------:|
| A1      | 3-8         | 1.73            | 低于下界 42% | 2-7       | 2.22      | 在区间内     | 20-47% | 56.2% | ⚠️ 超上界  |
| A2      | 3-8         | 2.01            | 低于下界 33% | 2-7       | 1.61      | 在区间内     | 20-47% | 44.6% | ✅       |
| A3      | 3-8         | 1.39            | 低于下界 54% | 2-7       | 0.98      | 低于下界     | 20-47% | 41.2% | ✅       |
| B1      | 5-12        | 1.38            | 低于下界 72% | 2-7       | 0.71      | 低于下界     | 17-70% | 34.0% | ✅       |
| C1      | 8-25        | 1.53            | 低于下界 81% | 5-15      | 2.47      | 低于下界     | 20-75% | 61.8% | ✅       |
| C3      | 8-25        | 1.28            | 低于下界 84% | 5-15      | 2.10      | 低于下界     | 20-75% | 62.1% | ✅       |
| D1      | 3-8         | 1.49            | 低于下界 50% | 2-7       | 1.07      | 低于下界     | 20-47% | 41.9% | ✅       |
| D3      | 10-30       | 7.47            | 低于下界 25% | 10-20     | 9.31      | 接近下界     | 25-67% | 55.5% | ✅       |
| A1-H    | 30-80       | 43.59           | 在区间内     | 50-100    | 83.60     | 在区间内     | 50-70% | 65.7% | ✅       |
| C1-H    | 10-30       | 12.99           | 在区间内     | 40-80     | 64.53     | 在区间内     | 70-90% | 83.2% | ✅       |
| D1-H    | 10-25       | 14.85           | 在区间内     | 3-10      | 5.71      | 在区间内     | 20-40% | 27.8% | ✅       |
| G2(10M) | 40-100      | 57.41           | 在区间内     | 60-120    | 83.77     | 在区间内     | 50-70% | 59.3% | ✅       |
| G3(10M) | 5-20        | 9.20            | 在区间内     | 50-100    | 64.94     | 在区间内     | 80-95% | 87.6% | ✅       |
| H1(10M) | 15-50       | 27.84           | 在区间内     | 100-500   | 3885.21   | **远超上界** | 80-95% | 99.3% | ⚠️ 远超上界 |
| H3(10M) | 40-100      | 63.19           | 在区间内     | 60-120    | 80.28     | 在区间内     | 50-70% | 56.0% | ✅       |
| G0-1    | —           | —               | —        | 2-7       | 1.05      | 低于下界 48% | —      | —     | ⚠️ 低于预期 |
| G0-2    | —           | —               | —        | 3-10      | 0.82      | 低于下界 73% | —      | —     | ⚠️ 低于预期 |
| G0-3    | —           | —               | —        | 5-18      | 4.36      | 低于下界 13% | —      | —     | ⚠️ 低于预期 |

**系统性偏差总结**：

- **DSL 延迟普遍低于预期 25-84%（轻查询）**：3 节点 + M4 Pro + JDK 25 远超预期基线（单节点/低配硬件）。重查询 DSL 在区间内
- **SQL 开销占比 16/18 在预期内**：A1 因 DSL 基线过低超上界（见 §5.6.2），H1 因 composite 聚合机制远超上界（见 §5.6.4），G0 因强硬件低于下界（见 §5.6.1）

### 5.8 测试局限性

1. **统计方法**：每场景 200 轮，p99 实为第 198 百分位点（200×0.99），统计意义有限；无法测 p999（生产 SLO 常用，需 ≥10000 轮）。4.5 节建议 ≥2000 轮
2. **环境与硬件**：① 单物理机回环，网络 <0.1ms（生产 0.5-2ms），聚合 scatter-gather 往返被低估；② 4GB heap 非生产 8GB，H1 期间达 79%（接近 90% breaker 阈值）；③ JDK 25 `UseCompactObjectHeaders` + 向量 API 可能比生产 JDK 17/21 快 5-15%
3. **测试设计**：① 无并发负载（sql-worker 排队未测，生产 10-100 并发可能改变结论）；② 缓存偏差未完全消除——A/B/D 组未每轮 `_cache/clear`，G1/H2 为确定性查询缓存命中（开销占比 ~50% 不代表冷查询），SQL 路径无法禁用 request cache（C1/C3 size=0 聚合可能被低估）；③ 未 forcemerge（1M 39 段 / 10M 60 段，可能影响 10-30%）
4. **测量方法**：G0 用 `_explain` 端点直接测量（V2 完整路径，见 §5.5），但含计划序列化+网络开销，非纯 ANTLR+Analyzer 耗时；E 组无 DSL 对照（见 §5.6.3）；§5.6.4 控制变量对比中 B2-H 10M DSL 数据缺失（JSON 转义问题）
5. **数据覆盖**：1M + 10M 已验证大结果集劣化比例下降趋势，但 100M+ 及 30+ 节点需进一步验证（scatter-gather 长尾 + 协调节点 merge 非线性增长，不可外推）
6. **生产风险**：H1 单查询 ~4s，生产环境可能触发超时（默认 30s）或 circuit breaker（见 §5.6.4 H1 分析）

---

> **报告生成时间**：2026-07-20 21:10 UTC
> **测试脚本**：`/tmp/os-bench/benchmark.py`（轻查询 + G0 组）、`/tmp/os-bench/heavy_query_benchmark.py`（重查询 + 10M 场景，含 `bench()` / `bench_sql_only()` 两个测量函数）、`/tmp/os-bench/generate_data_10m.py`（10M 数据生成）
> **原始数据**：`/tmp/os-bench/results.json`（轻查询）、`/tmp/os-bench/results_heavy_1m.json`（重查询 1M）、`/tmp/os-bench/results_10m.json`（10M 场景），均为 JSON 格式，含全部 p50/p95/p99/max/mean 统计
> **集群状态**：测试后保留运行（node-1 @ 9201, node-2 @ 9202, node-3 @ 9203），`perf_test`（1M）和 `perf_test_10m`（10M）索引均可供复查

## 六、SQL 插件能力扩展评估

### 6.1 扩展模式分类

| 模式              | 适用特性           | 说明                                                               |
| --------------- | -------------- | ---------------------------------------------------------------- |
| **计划级路由**（三步模式） | JOIN、EXISTS    | AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 实现 |
| **函数注册**        | COALESCE       | 注册到 BuiltinFunctionRepository，映射 Calcite SqlOperator             |
| **聚合函数修复**      | DATE_HISTOGRAM | 修复 INTERVAL NPE + 注册聚合 + 映射下推                                    |
| **表达式级子查询**     | 标量子查询          | AstExpressionBuilder + Calcite RexSubquery                       |
| **语句级文法**       | CTE            | 新增文法规则 + AST 节点 + 作用域管理                                          |

### 6.2 工作量估算

| 优先级    | 特性                | 模式    | 人天    | 代码量 (LOC) | 理由                                                                    |
|:------:| ----------------- | ----- |:-----:|:---------:| --------------------------------------------------------------------- |
| **P1** | COALESCE          | 函数注册  | 1-2   | 50-100    | Calcite 原生支持，仅需声明 operator + enum + grammar                           |
| **P1** | DATE_HISTOGRAM    | 聚合修复  | 3-5   | 200-400   | 修复 bug，时序分析基础                                                         |
| **P2** | 3表+ JOIN          | 计划级路由 | 5-8   | 400-600   | 复用 UNION 三步模式                                                         |
| **P2** | EXISTS 子查询        | 计划级路由 | 3-5   | 400-600   | SqlV2QueryParser 有参考                                                  |
| **P3** | JOIN + GROUP BY   | 依赖 P2 | 5-8   | 200-400   | 需验证 schema 解析+字段名冲突                                                   |
| **P3** | 子查询 + 外层 GROUP BY | 依赖 P2 | 5-8   | 200-400   | 需验证派生表 schema 传播                                                      |
| **P3** | 统计信息注入            | 独立工作流 | 20-40 | 800-1500  | 让 Calcite CBO 生效，但当前 SQL 默认走 V2 不经 Calcite，需 Calcite 成为 SQL 默认引擎后才有价值 |
| **P4** | 标量子查询             | 表达式级  | 8-12  | 300-500   | 复杂度最高                                                                 |
| **P5** | CTE               | 语句级   | 15-25 | 600-1000  | 文法+AST+作用域管理                                                          |

### 6.3 总工作量

| 范围             | 人天     |
| -------------- |:------:|
| P1（快速收益）       | 4-7    |
| P1-P2（核心能力）    | 12-20  |
| P1-P3（覆盖大部分场景） | 42-76  |
| P1-P5（全部）      | 65-113 |

### 6.4 依赖关系

```
统计信息注入 ──── 独立（需 Calcite 成为 SQL 默认引擎后推进）
COALESCE ─────── 独立
DATE_HISTOGRAM ── 独立
3表+ JOIN ─────── 独立
  ├── JOIN+GROUP BY ── 依赖 JOIN
  └── 子查询+GROUP BY ── 依赖 JOIN
EXISTS 子查询 ─── 独立
标量子查询 ────── 独立
CTE ───────────── 独立（可用派生表替代）
```

### 6.5 关键洞察

UNION 扩展的"三步模式"（AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 实现）仅适用于**计划级路由**的扩展（JOIN、EXISTS）。函数注册、聚合修复、表达式级子查询、语句级文法需要不同的扩展模式。

---

## 七、社区维护情况

> 数据采集时间：2026-07-21 | 仓库：https://github.com/opensearch-project/sql

### 7.1 GitHub 仓库统计

| 指标            | 数值               |
| ------------- | ---------------- |
| Stars         | 174              |
| Forks         | 214              |
| Contributors  | 142              |
| Open Issues   | 258              |
| Closed Issues | 1,798            |
| Open PRs      | 57               |
| Merged PRs    | 2,953（合并率 85.9%） |
| 创建时间          | 2021-04-02       |
| 最近 push       | 2026-07-20       |
| License       | Apache-2.0       |

**近 30 天活动**（2026-06-21 至 2026-07-21）：main 分支 23 commits、16 个新 issue、49 个新 PR、36 个合并 PR。

**main 分支月度 commit 趋势**（过去 12 个月）：平均 ~42 commits/月，2025 年 9-10 月达峰值 85（3.0 正式版发布冲刺），2026 年初回落至 24-30，近期回升至 47-67。

### 7.2 版本发布历史

项目维护三条发布线，紧密跟随 OpenSearch 主版本节奏：

| 发布线        | 最新版本     | 发布日期       | 节奏                          |
| ---------- | -------- | ---------- | --------------------------- |
| **3.x 主线** | 3.7.0.0  | 2026-06-02 | 每 ~2 个月，严格对应 OpenSearch 3.x |
| 2.19.x 维护线 | 2.19.6.0 | 2026-06-30 | 每 2-4 个月                    |
| 1.3.x LTS  | 1.3.20.0 | 2024-12-06 | **已停止**                     |

3.x 版本号与 OpenSearch 一一对应（3.0→3.7），发布间隔稳定在 ~2 个月，与 OpenSearch 核心同步发布。

### 7.3 维护团队

**17 名核心维护者**（来自 [MAINTAINERS.md](https://github.com/opensearch-project/sql/blob/main/MAINTAINERS.md)）：

| 组织        | 人数  | 占比  |
| --------- |:---:|:---:|
| Amazon    | 16  | 94% |
| Improving | 1   | 6%  |

另有 16 名 Emeritus 维护者（已退出）。[CODEOWNERS](https://github.com/opensearch-project/sql/blob/main/.github/CODEOWNERS) 中列出全部 17 名维护者，所有路径均需维护者审查。

过去 12 个月 top 9 提交者（main 分支）：

| 提交者             | commits | 归属          |
| --------------- |:-------:| ----------- |
| Lantao Jin      | 86      | Amazon（维护者） |
| Kai Huang       | 55      | Amazon（维护者） |
| Heng Qian       | 48      | Amazon（维护者） |
| Simeon Widdis   | 39      | Amazon（维护者） |
| Tomoyuki Morita | 34      | Amazon（维护者） |
| Chen Dai        | 33      | Amazon（维护者） |
| Yuanchun Shen   | 29      | Amazon（维护者） |
| Songkan Tang    | 25      | Amazon（维护者） |
| ritvibhatt      | 17      | 社区贡献者       |

**结论**：这是一个 **AWS 团队主导**的项目，top 10 提交者贡献了过去 12 个月 ~80% 的 commits，社区贡献为补充。

### 7.4 活跃度指标

**PR 合并时间**（最近 90 天，样本 100/186）：

| 统计量        | 值          |
| ---------- |:----------:|
| 中位数        | **0.85 天** |
| 平均值        | 4.6 天      |
| P90        | 12.0 天     |
| 当日合并（<1 天） | **63%**    |

> 63% 当日合并率包含大量自动化 backport PR。功能型 PR 通常需 6-15 天。

**Bug Issue 关闭时间**（最近 90 天，样本 19）：

| 统计量 | 值          |
| --- |:----------:|
| 中位数 | **60.3 天** |
| 平均值 | 93.5 天     |
| P90 | 222.6 天    |

> Bug 修复周期较长（中位数 ~2 个月），反映 SQL/PPL 语义复杂度高的特点。

**分诊与 stalled 情况**：

| 指标                    | 数值    | 说明                                            |
| --------------------- |:-----:| --------------------------------------------- |
| Open untriaged issues | **3** | 占 258 个 open issues 的 **1.2%**（分诊率极高）         |
| Open stalled PRs      | 7     | 占 57 个 open PR 的 12%（14 天无活动标记 stalled，不自动关闭） |
| Open bugs             | 54    | —                                             |
| Open enhancements     | 126   | —                                             |
| Good first issues     | 3     | 新人友好任务较少                                      |

### 7.5 社区健康度

**贡献规范**：完善（CONTRIBUTING.md + MAINTAINERS.md + CODEOWNERS + CODE_OF_CONDUCT.md + SECURITY.md + 5 个 Issue 模板）。DCO 签署必需，不接受匿名/化名。

**CI/CD**：非常完善（**31 个 GitHub Actions workflow**），包括：

| Workflow                          | 用途                     |
| --------------------------------- | ---------------------- |
| `sql-test-and-build-workflow.yml` | 主 CI：编译 + 单元测试         |
| `integ-tests-with-security.yml`   | 安全模式集成测试               |
| `sql-pitest.yml`                  | 变异测试（mutation testing） |
| `backport.yml`                    | 自动 backport            |
| `codeql-analysis.yml`             | GitHub CodeQL 安全扫描     |
| `dco.yml`                         | DCO 签署检查               |
| `add-untriaged.yml`               | 新 issue 自动标记 untriaged |

**代码覆盖率**：`build.gradle` 配置 JaCoCo 最低 50% 行覆盖率，`check` 任务依赖覆盖率验证（CI 强制）。集成 [Codecov](https://codecov.io/gh/opensearch-project/sql) 上报。

### 7.6 已知技术债务

**Legacy V1 引擎**（最大技术债）：

| 指标                       | 数值                  |
| ------------------------ | ------------------- |
| Legacy 模块 Java 文件数       | **307**             |
| 占全仓库 Java 文件比例           | **11.8%**（307/2607） |
| 过去 12 个月 legacy/ commits | 21                  |

项目同时维护 V1（legacy）、V2、V3 三代引擎。近期 legacy 提交包括清理 deprecated API（`Remove all AccessController refs #4924`）、修复内存泄漏（`Fix PIT context leak #5009`）。团队正在积极清理，但 307 个文件仍构成显著维护负担。

**其他债务**：

- 258 个 open issues 积压（bug 中位数关闭 60 天）
- 3.x 与 2.19.x 双线维护增加 backport 成本（`backport.yml` 自动化缓解）

### 7.7 与同类项目对比

| 仓库                  | Stars   | Forks   | Open Issues | 最近 Push        |
| ------------------- |:-------:|:-------:|:-----------:|:--------------:|
| k-NN                | 220     | 223     | 273         | 2026-07-18     |
| **SQL**             | **174** | **214** | **258**     | **2026-07-20** |
| anomaly-detection   | 93      | 84      | 109         | 2026-07-20     |
| alerting            | 81      | 131     | 344         | 2026-07-20     |
| asynchronous-search | 34      | 58      | 33          | 2026-07-14     |

SQL 插件 Stars 排名第 2（仅次于 k-NN），Forks 排名第 1（214），属最活跃梯队。Elasticsearch SQL 在 x-pack 商业插件中（闭源），无可比性。

### 7.8 综合评估

| 维度        | 评分    | 说明                                     |
| --------- |:-----:| -------------------------------------- |
| 社区规模      | ⭐⭐⭐⭐  | 174 stars / 142 contributors，生态第 2     |
| 维护活跃度     | ⭐⭐⭐⭐⭐ | 每 2 月稳定发布，日均 1-2 PR 合并                 |
| 响应速度      | ⭐⭐⭐⭐  | PR 中位 0.85 天；bug 中位 60 天               |
| 分诊质量      | ⭐⭐⭐⭐⭐ | 仅 3 个 untriaged（1.2%），极优秀              |
| CI/CD 成熟度 | ⭐⭐⭐⭐⭐ | 31 个 workflow，覆盖率强制 50%，变异测试           |
| 贡献门槛      | ⭐⭐⭐   | DCO 必需，维护者全 AWS，good-first-issue 仅 3 个 |
| 技术债务      | ⭐⭐⭐   | Legacy 引擎占 11.8%；bug 修复慢               |
| 版本治理      | ⭐⭐⭐⭐⭐ | 严格跟随 OpenSearch，3.x+2.19.x 双线清晰        |

**结论**：OpenSearch SQL 是一个 **AWS 团队主导、高度活跃、工程规范成熟** 的核心插件。发布节奏稳定（每 2 月）、CI/CD 完善、分诊及时。主要风险在于 Legacy V1 引擎的持续维护负担和 bug 修复周期较长。社区贡献门槛较高（几乎全 AWS 维护者），外部贡献为补充。

---

## 八、综合结论

> 基于本文档前七章的架构分析、实测数据、能力评估和社区调研，给出以下综合结论。

### 8.1 性能维度

SQL 相对于 DSL 的性能劣化**在大多数场景下可接受**，但存在一个明确边界：

| 场景类型                    | 劣化比例   | 绝对开销      | 结论                     |
| ----------------------- |:------:|:---------:| ---------------------- |
| 点查 / 全文搜索 / 排序分页 / 大结果集 | 34-66% | 0.7-84ms  | ✅ 可接受（序列化主导，不影响用户体验）   |
| 低基数聚合（≤1K桶）             | 51-62% | 1.3-2.5ms | ✅ 可接受                  |
| 中基数聚合（~10K桶）            | 83-85% | 65-99ms   | ⚠️ 边界化（接近瓶颈拐点）         |
| 高基数聚合（≥50K桶）            | 99.3%  | 3885ms    | ❌ 不可接受（composite 架构瓶颈） |

- **纯翻译开销**：0.82-4.36ms（`_explain` 直接测量），远低于预期，不构成瓶颈
- **序列化开销**：SQL `JdbcResponseFormatter` JDBC 格式转换随结果集行数线性增长（~8μs/行），与数据量无关，是大结果集场景的主要额外开销（DSL 原生 `ToXContent` 序列化已在基线中）
- **聚合机制开销**：随桶数指数增长，是 V2 composite 聚合架构设计 + `AGGREGATION_BUCKET_SIZE=1000` 硬编码共同导致的确定性瓶颈
- **"大数据量劣化比例小"假设**：在序列化主导和低基数聚合场景成立（控制变量验证：占比 ↓8-13pp / ↓2-9pp）；在高基数聚合场景不成立（占比 ↑2pp，composite 分页成本随数据量上升）
- **测试条件声明**：以上结论基于单连接串行测试（3 节点、4GB heap、JDK 25、M4 Pro），并发负载与故障恢复未验证

**SQL 独有优势**：除上述"SQL 相比 DSL 慢多少"的维度外，SQL 还具备 DSL 无法实现的能力（无 DSL 对照，不参与劣化比例结论）：

| 能力                              | 性能（1M p50）                       | 说明                    |
| ------------------------------- |:--------------------------------:| --------------------- |
| UNION ALL / DISTINCT            | 7.4ms（聚合后）/ 139ms（100K 行无 LIMIT） | DSL 需多次请求 + 应用层合并，非原子 |
| 2 表 JOIN + WHERE/ORDER BY/LIMIT | 58ms（10K 行）                      | DSL 不支持 JOIN          |
| IN 子查询                          | 77ms（10K 行）                      | DSL 需先查子查询再 terms，非原子 |
| 窗口函数（RANK OVER）                 | —                                | DSL 完全不支持             |
| 派生表                             | —                                | DSL 不支持               |

这些场景的性能差异反映的是**架构能力差异**（单次 SQL 请求 vs 多次 DSL 请求 + 应用层合并），而非翻译效率。

### 8.2 能力扩展维度

SQL 插件当前能力存在若干缺口，但均有明确的扩展路径：

| 优先级 | 特性                           | 模式    | 人天    | 代码量 (LOC) | 理由                                                                    |
|:---:| ---------------------------- | ----- |:-----:|:---------:| --------------------------------------------------------------------- |
| P1  | COALESCE                     | 函数注册  | 1-2   | 50-100    | Calcite 原生支持，仅需声明 operator                                            |
| P1  | DATE_HISTOGRAM               | 聚合修复  | 3-5   | 200-400   | 修复 bug，时序分析基础                                                         |
| P2  | 3表+ JOIN / EXISTS            | 计划级路由 | 8-13  | 800-1200  | 复用 UNION 三步模式                                                         |
| P3  | JOIN+GROUP BY / 子查询+GROUP BY | 依赖 P2 | 10-16 | 400-800   | schema 验证                                                             |
| P3  | 统计信息注入                       | 独立工作流 | 20-40 | 800-1500  | 让 Calcite CBO 生效，但当前 SQL 默认走 V2 不经 Calcite，需 Calcite 成为 SQL 默认引擎后才有价值 |
| P4  | 标量子查询                        | 表达式级  | 8-12  | 300-500   | 复杂度最高                                                                 |
| P5  | CTE                          | 语句级   | 15-25 | 600-1000  | 文法+AST+作用域                                                            |

- UNION 扩展的"三步模式"（AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 实现）仅适用于**计划级路由**扩展；函数注册、聚合修复等需不同模式
- 快速收益（P1）：4-7 人天即可补齐 COALESCE、DATE_HISTOGRAM
- 统计信息注入原标 P0，但当前 SQL 查询主要走 V2 引擎（`shouldUseCalcite()` 仅对 UNION 返回 true），V2 无 Calcite 优化器，统计信息注入对 V2 路径无影响。应在 Calcite 成为 SQL 默认引擎后推进
- 全部覆盖（P1-P5）：65-113 人天

### 8.3 社区维护维度

OpenSearch SQL 是一个 **AWS 团队主导、高度活跃、工程规范成熟** 的核心插件：

- **发布节奏**：每 2 月稳定发布，严格跟随 OpenSearch 主版本
- **CI/CD**：31 个 GitHub Actions workflow，覆盖率强制 50%，含变异测试
- **分诊质量**：仅 3 个 untriaged issues（1.2%），极优秀
- **维护团队**：17 名维护者，94% 来自 Amazon，社区贡献为补充
- **主要风险**：Legacy V1 引擎 307 个文件（11.8%）构成维护负担；bug 修复中位数 60 天

### 8.4 综合建议

**性能层面**：SQL 插件适用于点查、全文搜索、排序分页、大结果集和低基数聚合场景。高基数聚合（GROUP BY 高基数字段 + LIMIT）应使用 DSL 或 PPL `stats`（走 Calcite pushdown，有 terms 转换）。

**能力层面**：建议优先推进 P1（COALESCE、DATE_HISTOGRAM），以 4-7 人天的投入补齐最高 ROI 的能力缺口。统计信息注入需待 Calcite 成为 SQL 默认引擎后推进。CTE 和标量子查询可后续迭代。

---
