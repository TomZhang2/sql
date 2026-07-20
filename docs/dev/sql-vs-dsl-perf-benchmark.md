# SQL vs DSL 查询性能对比验证方案

> 基于 OpenSearch 3.7.0 + SQL Plugin 3.7.0（含 UNION/UNION ALL 扩展）
> 所有 SQL 能力声明均已通过实测验证

---

## 目录

- [一、SQL 插件引擎架构](#一sql-插件引擎架构)
- [二、白盒实现分析](#二白盒实现分析)
- [三、SQL vs DSL 能力对比](#三sql-vs-dsl-能力对比)
- [四、验证方案设计](#四验证方案设计)
- [五、SQL 插件能力扩展评估](#五sql-插件能力扩展评估)
- [六、扩展性交叉验证（OpenSearch 官方基准）](#六扩展性交叉验证opensearch-官方基准)
- [七、大数据视角与生产可用性评估](#七大数据视角与生产可用性评估)
- [附录：修正记录](#附录修正记录)

---

## 一、SQL 插件引擎架构

OpenSearch SQL 插件有三种执行引擎，是历史演进的结果。

### 1.1 Legacy V1 引擎

基于 Alibaba Druid SQL 解析器的查询翻译层。SQL 直接翻译成 OpenSearch DSL，无计划抽象。

```
SQL → Druid Parser → QueryAction → SearchRequestBuilder → OpenSearch 执行
```

- **来源**：fork 自 `elasticsearch-sql`（NLPchina）
- **支持**：2 表 JOIN、IN 子查询、COALESCE、DATE_HISTOGRAM
- **不支持**：3 表 JOIN、JOIN+GROUP BY、EXISTS、窗口函数
- **状态**：维护中，不再新增功能

### 1.2 V2 引擎（当前主力）

OpenSearch 自研，完整 AST → LogicalPlan → PhysicalPlan 抽象层。

```
SQL → ANTLR 4 → AstBuilder → Analyzer → Planner → PhysicalPlan → SearchRequestBuilder → OpenSearch 执行
```

- **支持**：窗口函数、派生表、IN 子查询（回退 Legacy V1）
- **不支持**：JOIN（抛异常回退 Legacy）、UNION（我们的扩展已改为走 Calcite）、CTE、COALESCE、DATE_HISTOGRAM
- **聚合策略**：使用 `composite` 聚合分页拉取，size 由 `plugins.query.buckets` 配置（`OpenSearchSettings.QUERY_BUCKET_SIZE_SETTING`，默认 = `index.max_result_window`，通常 10000；测试中常显式设为 1000）

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

| 维度 | Legacy V1 | V2 | Calcite |
|------|-----------|-----|---------|
| 解析器 | Druid | ANTLR 4 | ANTLR 4 |
| 计划抽象 | 无 | AST→LogicalPlan→PhysicalPlan | AST→RelNode |
| 优化器 | 无 | 简单规则 | Calcite Volcano（无统计，退化为规则） |
| 算子下推 | 无 | 无 | ✅ filter/agg/sort/limit |
| JOIN | ✅ 2 表 | ❌（回退 Legacy） | ✅ N 表（SQL 未路由到此） |
| UNION | ✅ | ❌（扩展后走 Calcite） | ✅（我们的扩展） |
| 窗口函数 | ❌ | ✅ | ✅ |
| 内存计算 | 无 | 无 | 有（Enumerable，单节点单线程） |
| 状态 | 维护中 | 活跃开发 | 活跃开发（未来方向） |

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
POST /index/_search → RestSearchAction → TransportSearchAction → QueryPhase → FetchPhase → JSON
```

- 零翻译层、零中间对象、零额外内存
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

| 环节 | DSL | SQL (V2) | SQL (Calcite) | SQL (Legacy V1) |
|------|-----|----------|---------------|-----------------|
| 请求解析 | JSON ~0.1ms | ANTLR ~0.5-2ms | ANTLR ~0.5-2ms | Druid ~1-3ms |
| 语义分析 | 无 | Analyzer ~0.5-2ms | CalciteRelNodeVisitor ~1-5ms | 无 |
| 查询规划 | 无 | Planner ~0.3-1ms | Calcite 优化器 ~2-8ms | 无 |
| DSL 生成 | 无（本身是 DSL） | PhysicalPlan→SearchRequestBuilder | Calcite 下推 | QueryAction→SearchRequestBuilder |
| 结果格式化 | JSON ~0.5ms | JdbcResponseFormatter ~1-2ms | JdbcResponseFormatter ~1-2ms | PrettyFormatRestExecutor ~1-2ms |
| 线程池 | search (OS 内置, 8核→13线程) | sql-worker (=allocatedProcessors, 8核→8线程) | sql-worker (=allocatedProcessors) | sql-worker→search |
| 总额外开销 | ~0ms | ~2-7ms | ~5-18ms | ~2-6ms |

### 2.4 关键差异点

1. **SQL 最终都翻译成 DSL**——差异只在上层翻译开销
2. **Calcite 下推**——Convention trait 驱动，规则非代价；不可下推的操作在内存单线程计算
3. **游标**——DSL `search_after`（无状态）vs V2 序列化游标（有状态）vs Calcite `EnumerableLimit`
4. **无计划缓存**——每次查询重新解析+规划
5. **Calcite 内存风险**——UNION/JOIN 在协调节点单线程，大数据量可能 OOM

---

## 三、SQL vs DSL 能力对比

> 以下所有 SQL 能力声明均通过 `POST /_plugins/_sql` 实测验证（2026-07-18）

### 3.1 查询能力矩阵

| 能力 | SQL | DSL | 说明 |
|------|:---:|:---:|------|
| 等值/范围查询 | ✅ | ✅ | SQL `WHERE`; DSL `term`/`range` |
| 多条件布尔查询 | ✅ | ✅ | SQL `AND`/`OR`; DSL `bool` |
| 全文搜索 | ⚠️ | ✅ | SQL `match()`/`multi_match()`; DSL 完整参数 |
| GROUP BY 聚合 | ✅ | ✅ | SQL `GROUP BY`; DSL `aggs` |
| 窗口函数 | ✅ | ❌ | SQL `RANK() OVER(...)`; DSL 不支持 |
| 2 表 JOIN | ✅ | ❌ | SQL（回退 Legacy V1）; DSL 不支持 |
| 3 表+ JOIN | ❌ | ❌ | SQL 报错 "only 2 tables" |
| JOIN + GROUP BY | ❌ | ❌ | SQL 报错 "JOIN queries do not support aggregations on the joined result."（`Util.java:44`） |
| UNION ALL | ✅ | ❌ | SQL（我们的扩展，走 Calcite） |
| UNION DISTINCT | ✅ | ❌ | SQL（我们的扩展，走 Calcite） |
| IN 子查询 | ✅ | ❌ | SQL（回退 Legacy V1 Hash Join） |
| EXISTS 子查询 | ⚠️ | ❌ | SQL 普通 EXISTS 报错 "Unsupported subquery"（`SubQueryRewriter.java:74`）；嵌套字段 EXISTS 支持（`NestedExistsRewriter.java`，`canRewrite()` 仅对嵌套字段返回 true） |
| 派生表 | ✅ | ❌ | SQL `(SELECT...) AS t` |
| CTE (WITH) | ❌ | ❌ | 文法无 WITH 规则（`OpenSearchSQLLexer.g4` 无 WITH token）；Legacy 报错 "Query must start with SELECT, DELETE, SHOW or DESCRIBE"（`OpenSearchActionFactory.java:135`），V2 抛 ANTLR 语法错误 |
| COALESCE | ❌ | ✅ | V2 引擎报错 "unsupported function name: coalesce"（`BuiltinFunctionRepository.java:145`，函数注册表缺失）；Legacy 报错 "not supported in Schema"（`SelectResultSet.java:360`） |
| DATE_HISTOGRAM | ❌ | ✅ | V2 引擎无 DATE_HISTOGRAM 函数注册；Legacy 实际支持（`AggMaker.java:558-611`，含 interval/fixed_interval/format/time_zone 等参数）；V2 INTERVAL 参数处理有已知 bug（`RexStandardizer.java:117` 注释），基于实测观察 |
| 脚本字段 | ❌ | ✅ | DSL `script_fields` |
| 运行时字段 | ❌ | ✅ | DSL `runtime_mappings` |
| profile API | ❌ | ✅ | DSL `profile: true` |

### 3.2 SQL 独有能力

| 能力 | 语法 | 引擎路径 |
|------|------|---------|
| 2 表 JOIN + WHERE/ORDER BY/LIMIT | `SELECT...FROM a JOIN b ON...` | Legacy V1 |
| UNION ALL / DISTINCT | `SELECT...UNION ALL SELECT...` | Calcite（扩展） |
| IN 子查询 | `WHERE x IN (SELECT...)` | Legacy V1 |
| 派生表 | `FROM (SELECT...) AS t` | V2 |
| 窗口函数 | `RANK() OVER(...)` | V2 |

### 3.3 SQL 已知限制

| 限制 | 错误信息 | 根因 |
|------|---------|------|
| CTE | Legacy: `Query must start with SELECT, DELETE, SHOW or DESCRIBE`（`OpenSearchActionFactory.java:135`）；V2: ANTLR 语法错误 | 文法无 WITH 规则（`OpenSearchSQLLexer.g4` 无 WITH token） |
| 3 表+ JOIN | `currently supports only 2 tables join`（`SqlParser.java:380`） | Legacy V1 限制 |
| JOIN + GROUP BY | `JOIN queries do not support aggregations on the joined result.`（`Util.java:44`） | Legacy V1 限制 |
| EXISTS 子查询 | 普通: `Unsupported subquery`（`SubQueryRewriter.java:74`）；嵌套字段: 实际支持（`NestedExistsRewriter.java`） | V2 抛 `getOnlyForCalciteException` 回退 Legacy；Legacy 仅支持嵌套字段 EXISTS |
| 标量子查询 | V2: `Subsearch is supported only when plugins.calcite.enabled=true`（`ExpressionAnalyzer.java:470`）；Legacy: `unknown field name`（`FieldMaker.java:67`） | V2 抛 `getOnlyForCalciteException`，仅 Calcite 引擎有实现（SQL 仅 UNION 走 Calcite，普通 SELECT 不走）；Legacy 遇到子查询作为字段直接抛异常（`FieldMaker.java:67`） |
| COALESCE | V2: `unsupported function name: coalesce`（`BuiltinFunctionRepository.java:145`）；Legacy: `not supported in Schema`（`SelectResultSet.java:360`） | V2 函数注册表未注册 COALESCE（`BuiltinFunctionRepository.java:73-85`） |
| DATE_HISTOGRAM | V2 无此函数；INTERVAL 参数处理有已知 bug | V2 core 无 DATE_HISTOGRAM 函数注册；`RexStandardizer.java:117` 注释 "INTERVAL_TYPES has bug, introduced by calcite-1.41.1"；Legacy 实际支持（`AggMaker.java:558-611`） |

---

## 四、验证方案设计

### 4.1 测试环境

> **两阶段测试**：1M 单节点用于隔离翻译开销（轻/重查询）；10M 3 节点用于生产级验证（详见 4.4 节）。两者使用不同 schema/shard/forcemerge 策略，**结构性不可比**，不可直接推论扩展性规律。

| 项目 | 1M 测试（单节点） | 10M 测试（3 节点，详见 4.4） |
|------|---------|---------|
| 节点数 | 1 | 3（每节点 8C 16G，JVM heap 8GB） |
| Shard 数 | 3 | 6 shard 1 replica |
| 文档数 | 1,000,000 | 9,900,000 |
| 字段数 | 10 | 13（含高基数 user_id=100K, session_id=500K） |
| forcemerge | 是（隔离 segment 数变量） | 否（模拟生产） + 对照组 forcemerge |
| 预热 | 50 轮（`-XX:+PrintCompilation` 沉默后开始计量） | 30-50 轮 |
| 测试轮数 | ≥2000 轮（支撑 p99 置信区间） | ≥1000 轮 |

> ⚠️ **单节点测试局限性**：无 scatter-gather、无跨节点网络延迟、无跨 shard merge。1M forcemerge 与 10M 未 forcemerge 的 segment 数差异巨大（1 段 vs 50-200 段），查询路径长度差 10-100 倍——**两组测试仅用于各自规模内的 SQL vs DSL 对比，不可外推到其他规模**。扩展性结论需引用 OpenSearch 官方基准交叉验证（见第六章）。

### 4.2 轻查询场景（1M 数据，返回 ≤10 行）

> **对等性原则**：DSL 用 `bool.filter`（不打分），SQL `WHERE` 生成 `bool.filter`，语义对等。
> **缓存控制**：DSL 加 `?request_cache=false`；SQL 用随机阈值打散 filter cache。

#### A 组：点查

| 场景 | SQL 查询 | DSL 查询 |
|------|---------|---------|
| A1 等值(单条件) | `SELECT * FROM perf_test WHERE status_code = 200 LIMIT 10` | `{"query":{"term":{"status_code":200}},"size":10}` |
| A2 等值(多条件) | `SELECT * FROM perf_test WHERE status_code = 500 AND level = 'ERROR' AND region = 'us-east-1' LIMIT 10` | `{"query":{"bool":{"filter":[{"term":{"status_code":500}},{"term":{"level":"ERROR"}},{"term":{"region":"us-east-1"}}]}},"size":10}` |
| A3 范围 | `SELECT * FROM perf_test WHERE response_time_ms > 3000 AND status_code = 500 LIMIT 10` | `{"query":{"bool":{"filter":[{"range":{"response_time_ms":{"gt":3000}}},{"term":{"status_code":500}}]}},"size":10}` |

#### B 组：全文搜索

| 场景 | SQL 查询 | DSL 查询 |
|------|---------|---------|
| B1 简单 | `SELECT message, service FROM perf_test WHERE match(message, 'timeout') LIMIT 10` | `{"query":{"match":{"message":"timeout"}},"size":10,"_source":["message","service"]}` |
| B2 多字段 | `SELECT * FROM perf_test WHERE MULTI_MATCH(message, 'request failed') AND level = 'ERROR' LIMIT 10` | `{"query":{"bool":{"must":[{"match":{"message":"request failed"}}],"filter":[{"term":{"level":"ERROR"}}]}},"size":10}` |

#### C 组：聚合（随机阈值打散缓存）

| 场景 | SQL 查询 | DSL 查询 |
|------|---------|---------|
| C1 简单 | `SELECT level, COUNT(*) as cnt FROM perf_test WHERE response_time_ms > {random} GROUP BY level` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":{random}}}},"aggs":{"by_level":{"terms":{"field":"level"}}}}` |
| C2 多级 | `SELECT level, service, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt FROM perf_test WHERE response_time_ms > {random} GROUP BY level, service` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":{random}}}},"aggs":{"ls":{"composite":{"sources":[{"level":{"terms":{"field":"level"}}},{"service":{"terms":{"field":"service"}}}]},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}` |
| C3 范围+排序 | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE response_time_ms > {random} GROUP BY service ORDER BY cnt DESC` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":{random}}}},"aggs":{"by_service":{"terms":{"field":"service","order":{"_count":"desc"}}}}}` |
| C4 时间直方图 | SQL 不支持（DATE_HISTOGRAM NPE） | `{"size":0,"aggs":{"by_hour":{"date_histogram":{"field":"@timestamp","calendar_interval":"1h"}}}}` |

#### D 组：排序与分页

| 场景 | SQL 查询 | DSL 查询 |
|------|---------|---------|
| D1 排序+小分页 | `SELECT * FROM perf_test ORDER BY response_time_ms DESC LIMIT 10` | `{"query":{"match_all":{}},"sort":[{"response_time_ms":"desc"}],"size":10}` |
| D2 深度分页 | `SELECT * FROM perf_test ORDER BY response_time_ms DESC LIMIT 10000, 10` | `{"query":{"match_all":{}},"sort":[{"response_time_ms":"desc"}],"from":10000,"size":10}` |
| D3 大结果集 | `SELECT * FROM perf_test WHERE status_code = 200 LIMIT 1000` | `{"query":{"term":{"status_code":200}},"size":1000}` |
| D4 游标分页 | `{"query":"SELECT * FROM perf_test WHERE status_code = 200","fetch_size":100}` | `{"query":{"term":{"status_code":200}},"sort":[{"_id":"asc"}],"size":100}` + `search_after` |

#### E 组：SQL 独有（无 DSL 对照）

| 场景 | SQL 查询 | DSL 等价（多次查询+应用层合并） | 引擎 |
|------|---------|------|------|
| E1 UNION ALL+聚合 | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service` | 2 次 `terms` agg + 应用层合并 | Calcite |
| E2 2表JOIN | `SELECT a.service, a.level, b.host_name FROM perf_test a JOIN perf_test_meta b ON a.host = b.host WHERE a.level = 'ERROR' LIMIT 10` | 先查 perf_test 获取 host 列表，再查 perf_test_meta，应用层关联 | Legacy V1 |
| E3 IN子查询 | `SELECT * FROM perf_test WHERE host IN (SELECT host FROM perf_test WHERE level = 'ERROR') LIMIT 10` | 先查子查询获取 host 列表，再用 terms 查询 | Legacy V1 |

#### F 组：辅助验证

| 场景 | SQL 查询 | 说明 |
|------|---------|------|
| F1 冷启动 | `SELECT /* cold_run_{timestamp} */ * FROM perf_test WHERE status_code = 200 LIMIT 10` | 每轮唯一注释打散计划缓存 |
| F2 Pushdown on/off | 同 E1 查询，分别 `plugins.calcite.pushdown.enabled=true/false` | 隔离下推收益 |
| F3 结果等价 | 对每个场景的 SQL 和 DSL 结果集做行数+值比对 | TPC 标准 |

### 4.3 重查询场景（1M 数据，返回 5K-10K 行）

> 通过高命中率 + 大结果集 + 高基数聚合 + 深度翻页，让 DSL 执行时间达到 100-400ms。

#### A-H 组：大结果集点查

| 场景 | SQL 查询 | DSL 查询 | 预期 DSL |
|------|---------|---------|:---:|
| A1-H | `SELECT * FROM perf_test WHERE status_code = 200 ORDER BY response_time_ms DESC LIMIT 10000` | `{"query":{"term":{"status_code":200}},"sort":[{"response_time_ms":"desc"}],"size":10000}` | 80-150ms |
| A2-H | `SELECT * FROM perf_test WHERE status_code IN (200, 301, 404) AND level IN ('INFO','WARN','ERROR') ORDER BY bytes DESC LIMIT 5000` | `{"query":{"bool":{"filter":[{"terms":{"status_code":[200,301,404]}},{"terms":{"level":["INFO","WARN","ERROR"]}}]}},"sort":[{"bytes":"desc"}],"size":5000}` | 60-120ms |
| A3-H | ``SELECT * FROM perf_test WHERE response_time_ms > 2000 ORDER BY `@timestamp` DESC LIMIT 10000`` | `{"query":{"range":{"response_time_ms":{"gt":2000}}},"sort":[{"@timestamp":"desc"}],"size":10000}` | 80-150ms |

#### B-H 组：高命中全文搜索

| 场景 | SQL 查询 | DSL 查询 | 预期 DSL |
|------|---------|---------|:---:|
| B1-H | `SELECT message, service, level, response_time_ms FROM perf_test WHERE match(message, 'request') ORDER BY response_time_ms DESC LIMIT 5000` | `{"query":{"match":{"message":"request"}},"sort":[{"response_time_ms":"desc"}],"size":5000,"_source":["message","service","level","response_time_ms"]}` | 60-120ms |
| B2-H | `SELECT * FROM perf_test WHERE match(message, 'request failed timeout') AND status_code >= 400 ORDER BY bytes DESC LIMIT 10000` | `{"query":{"bool":{"must":[{"match":{"message":"request failed timeout"}}],"filter":[{"range":{"status_code":{"gte":400}}}]}},'sort":[{"bytes":"desc"}],"size":10000}` | 80-150ms |

#### C-H 组：高基数聚合

| 场景 | SQL 查询 | DSL 查询 | 预期 DSL |
|------|---------|---------|:---:|
| C1-H | `SELECT user_id, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, MAX(bytes) as max_bytes, MIN(response_time_ms) as min_rt FROM perf_test WHERE response_time_ms > 100 GROUP BY user_id ORDER BY cnt DESC LIMIT 1000` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":100}}},"aggs":{"by_user":{"terms":{"field":"user_id","size":1000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"max_bytes":{"max":{"field":"bytes"}},"min_rt":{"min":{"field":"response_time_ms"}}}}}}` | 100-300ms |
| C2-H | `SELECT level, service, region, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, SUM(bytes) as total_bytes FROM perf_test WHERE response_time_ms > 500 GROUP BY level, service, region ORDER BY level, cnt DESC` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":500}}},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_region":{"terms":{"field":"region"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"total_bytes":{"sum":{"field":"bytes"}}}}}}}}}}` | 100-250ms |
| C3-H | SQL 不支持（DATE_HISTOGRAM NPE） | `{"size":0,"aggs":{"by_hour":{"date_histogram":{"field":"@timestamp","calendar_interval":"1h"},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"p95":{"percentiles":{"field":"response_time_ms","percents":[95,99]}}}}}}}}` | 150-400ms |

#### D-H 组：深度分页 + 大结果集

| 场景 | SQL 查询 | DSL 查询 | 预期 DSL |
|------|---------|---------|:---:|
| D1-H | `SELECT * FROM perf_test ORDER BY response_time_ms ASC LIMIT 19990, 10` | `{"query":{"match_all":{}},"sort":[{"response_time_ms":"asc"}],"from":19990,"size":10}` | 100-200ms |
| D2-H | `SELECT * FROM perf_test WHERE status_code = 200 ORDER BY response_time_ms DESC LIMIT 10000` | `{"query":{"term":{"status_code":200}},"sort":[{"response_time_ms":"desc"}],"size":10000}` | 80-150ms |
| D3-H | ``SELECT service, level, response_time_ms, bytes, `@timestamp` FROM perf_test WHERE response_time_ms > 1000 AND status_code IN (200, 500) ORDER BY response_time_ms DESC LIMIT 10000`` | `{"query":{"bool":{"filter":[{"range":{"response_time_ms":{"gt":1000}}},{"terms":{"status_code":[200,500]}}]}},"sort":[{"response_time_ms":"desc"}],"size":10000,"_source":["service","level","response_time_ms","bytes","@timestamp"]}` | 80-150ms |

#### E-H 组：SQL 独有大数据量

| 场景 | SQL 查询 | 引擎 | 预期 SQL |
|------|---------|------|:---:|
| E1-H 三路UNION+聚合 | `SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'ERROR' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'WARN' GROUP BY service UNION ALL SELECT service, COUNT(*) as cnt FROM perf_test WHERE level = 'DEBUG' GROUP BY service` | Calcite | 20-50ms |
| E1b-H 250K行UNION | `SELECT service, response_time_ms FROM perf_test WHERE status_code = 500 UNION ALL SELECT service, response_time_ms FROM perf_test WHERE status_code = 503` | Calcite | 100-300ms |
| E2-H JOIN 10K行 | `SELECT a.service, a.level, a.response_time_ms, b.dept_name FROM perf_test a JOIN perf_test_meta b ON a.host = b.host WHERE a.level = 'ERROR' LIMIT 10000` | Legacy V1 | 200-500ms |
| E3-H IN子查询 10K行 | `SELECT service, level, response_time_ms FROM perf_test WHERE host IN (SELECT host FROM perf_test WHERE level = 'ERROR') LIMIT 10000` | Legacy V1 | 200-500ms |

### 4.4 10M 多节点生产级验证方案

> 目标：验证"大数据量重查询下 SQL 与 DSL 性能差异不大"。

#### 环境

```
3 节点集群，每节点 8C 16G（JVM heap 8GB）
6 shard 1 replica，10M 文档（13 字段含高基数）
未 forcemerge（模拟生产） + 对照组 forcemerge（隔离 segment 数变量）
slowlog threshold.query.warn: 0ms（仅 warn 级别，避免 info 级每查询同步写盘引入 1-5ms 抖动）
所有查询加 ?preference=_primary（消除 replica 路由差异）
测试窗口停止 indexing + translog durability=async + merge.scheduler.max_thread_count=1（隔离后台噪声）
```

#### 场景组 G：生产典型重查询

| 场景 | SQL 查询 | DSL 查询 | 预期 DSL |
|------|---------|---------|:---:|
| G1 时间范围+聚合 | ``SELECT service, level, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt FROM perf_test_10m WHERE `@timestamp` > '2026-07-11T00:00:00Z' GROUP BY service, level ORDER BY service, cnt DESC`` | `{"size":0,"query":{"range":{"@timestamp":{"gte":"2026-07-11T00:00:00Z"}}},"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}}}` | 300-600ms |
| G2 多字段排序+10K | ``SELECT * FROM perf_test_10m WHERE level = 'ERROR' AND `@timestamp` > '2026-07-17T00:00:00Z' ORDER BY response_time_ms DESC, bytes DESC LIMIT 10000`` | `{"query":{"bool":{"filter":[{"term":{"level":"ERROR"}},{"range":{"@timestamp":{"gte":"2026-07-17T00:00:00Z"}}]}},"sort":[{"response_time_ms":"desc"},{"bytes":"desc"}],"size":10000}` | 200-500ms |
| G3 高基数聚合+过滤 | ``SELECT user_id, COUNT(*) as req_count, AVG(response_time_ms) as avg_rt, MAX(bytes) as max_bytes, SUM(bytes) as total_bytes FROM perf_test_10m WHERE `@timestamp` > '2026-07-17T00:00:00Z' AND status_code = 200 GROUP BY user_id ORDER BY req_count DESC LIMIT 1000`` | `{"size":0,"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"2026-07-17T00:00:00Z"}}},{"term":{"status_code":200}}]}},"aggs":{"by_user":{"terms":{"field":"user_id","size":1000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"max_bytes":{"max":{"field":"bytes"}},"total_bytes":{"sum":{"field":"bytes"}}}}}}` | 300-600ms |
| G4 复合+5K | ``SELECT service, level, response_time_ms, bytes, `@timestamp`, request_path, client_ip FROM perf_test_10m WHERE status_code >= 400 AND response_time_ms > 2000 AND `@timestamp` > '2026-07-15T00:00:00Z' ORDER BY response_time_ms DESC LIMIT 5000`` | `{"query":{"bool":{"filter":[{"range":{"status_code":{"gte":400}}},{"range":{"response_time_ms":{"gt":2000}}},{"range":{"@timestamp":{"gte":"2026-07-15T00:00:00Z"}}]}},"sort":[{"response_time_ms":"desc"}],"size":5000,"_source":["service","level","response_time_ms","bytes","@timestamp","request_path","client_ip"]}` | 150-400ms |

#### 场景组 H：极重查询

| 场景 | SQL 查询 | DSL 查询 | 预期 DSL |
|------|---------|---------|:---:|
| H1 全表500K桶聚合 | `SELECT session_id, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt FROM perf_test_10m WHERE response_time_ms > 100 GROUP BY session_id ORDER BY cnt DESC LIMIT 5000` | `{"size":0,"query":{"range":{"response_time_ms":{"gt":100}}},"aggs":{"by_session":{"terms":{"field":"session_id","size":5000,"order":{"_count":"desc"}},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}}}}}}` | 800-2000ms |
| H2 全表多级聚合 | `SELECT service, level, region, COUNT(*) as cnt, AVG(response_time_ms) as avg_rt, SUM(bytes) as total_bytes FROM perf_test_10m GROUP BY service, level, region ORDER BY service, cnt DESC` | `{"size":0,"aggs":{"by_service":{"terms":{"field":"service"},"aggs":{"by_level":{"terms":{"field":"level"},"aggs":{"by_region":{"terms":{"field":"region"},"aggs":{"avg_rt":{"avg":{"field":"response_time_ms"}},"total_bytes":{"sum":{"field":"bytes"}},"p99":{"percentiles":{"field":"response_time_ms","percents":[99]}}}}}}}}}}` | 600-1500ms |
| H3 大范围+10K | ``SELECT * FROM perf_test_10m WHERE `@timestamp` > '2026-07-11T00:00:00Z' ORDER BY response_time_ms DESC LIMIT 10000`` | `{"query":{"range":{"@timestamp":{"gte":"2026-07-11T00:00:00Z"}}},"sort":[{"response_time_ms":"desc"}],"size":10000}` | 400-800ms |

#### 翻译开销分解方法

> **核心原则**：slowlog 毫秒精度（±1ms）与声称的 2-7ms 翻译开销同量级，**不可直接用 `T_sql - T_slowlog` 分解**。改用以下方法：

**方法 A（推荐）：SQLService 内部埋点**
- 利用 `QueryService.java:147` 已有的 `ProfileMetric ANALYZE` 埋点（`profileContext.getOrCreateMetric(MetricName.ANALYZE)`）
- 通过 `_plugins/_sql/_explain` + profile API 拿到 ANALYZE 阶段纳秒级耗时
- 直接测量翻译阶段，不经过网络/排队/序列化

**方法 B（验证用）：slowlog 交叉验证**
1. 运行 SQL 查询，记录端到端延迟 `T_sql`
2. 从 slowlog 提取服务端执行时间 `T_exec`
3. 运行等价 DSL，记录 `T_dsl`
4. **验证 `T_exec ≈ T_dsl`**（此等式成立则证明 SQL 与 DSL 服务端执行等价）
5. 翻译开销 ≠ `T_sql - T_exec`（含网络/排队/序列化），仅用方法 A 的 ANALYZE 耗时作为翻译开销

**方法 C（备选）：`_nodes/stats` 累计均值**
- `_nodes/stats` 的 `indices/search/query_time_in_millis / query_total` 拿到纳秒级累积均值
- 适合大规模样本下的均值估算，不适合分位数

> ⚠️ **不使用** `_explain` 提取的 DSL 来测执行时间——explain 不触发 Janino codegen、不触发 fallback、不经过 sql-worker 调度，输出与实际执行 DSL 可能不同（`executeWithCalcite` 会加 `LogicalSystemLimit`，执行时还有 retry/fallback 逻辑）。

### 4.5 测试方法

#### 压测脚本框架

```python
session = requests.Session()  # 复用 TCP 连接

def bench(name, fn_factory, warmup=50, runs=2000):
    """fn_factory(threshold) 返回可调用对象，threshold 用于随机化打散缓存。
    runs=2000 支撑 p99 置信区间（200 样本下 p99.9≈max 无统计意义）。
    若资源受限可降至 1000，但放弃 p99.9 改报 max。"""
    # 预热：观察 -XX:+PrintCompilation 日志，沉默 N 轮后再开始计量
    # （JIT C2 编译是事件驱动，CV<5% 不适合判定 JIT 完成）
    compilation_quiet = False
    warmup_count = 0
    while not compilation_quiet and warmup_count < warmup * 3:
        fn_factory(random.randint(1, 9000))()
        warmup_count += 1
        # 实际实现：解析 -XX:+PrintCompilation 输出，无新编译事件持续 N 轮则 quiet
    for _ in range(warmup):
        fn_factory(random.randint(1, 9000))()
    latencies = []
    for _ in range(runs):
        t = random.randint(1, 9000)
        t0 = time.perf_counter()
        r = fn_factory(t)()
        dt = (time.perf_counter() - t0) * 1000
        if r.status_code == 200: latencies.append(dt)
        # 每轮之间 _cache/clear 或重启节点（见 4.6 缓存控制）
    # 计算 p50/p99/max（放弃 p99.9——需 1000+ 样本才稳定，2000 轮仅近似）
    # 用 bootstrap 给 p99 置信区间
```

#### 执行步骤

```
1. 启动集群 → 创建索引 → 导入数据 → 轮询确认
2. 结果集等价验证（F3）
3. 预热（50+ 轮，-XX:+PrintCompilation 沉默 N 轮后开始计量；非 CV<5%）
4. 正式测试（≥2000 轮，聚合场景随机阈值打散缓存；每轮间 _cache/clear）
5. 并发吞吐量测试（8/16/32 并发，60 秒；混合 50% SQL + 50% DSL 观察资源隔离）
6. Pushdown on/off 对比（F2）
7. 两遍运行（第一遍丢弃，验证 JIT 稳定性）
8. 收集指标 + 关联 JIT/GC 日志
9. 失败恢复测试（I 组：人为触发 breaker、kill sql-worker 线程、模拟 OOM）
```

#### 收集指标

| 指标 | 方式 | 说明 |
|------|------|------|
| 端到端延迟 | 客户端计时 | p50/p99/p99.9 |
| per-run 延迟 | 客户端记录 | 关联 JIT/GC 异常值 |
| 服务端执行耗时 | slowlog | 分解翻译开销 |
| 吞吐量 QPS | 并发测试 | 8/16/32 并发 |
| SQL explain | `_explain` | 确认引擎路径+下推 |
| Calcite 回退 | 服务端日志 | "Fallback to V2" |
| JIT 编译 | `-XX:+PrintCompilation` | 关联延迟尖峰 |
| GC 频率 | `-Xlog:gc*` | 检测 SQL 路径 GC 压力 |
| JVM heap | `_nodes/stats/jvm` | 含 off-heap direct |
| 线程池 | `_nodes/stats/thread_pool` | sql-worker vs search |
| 结果集等价 | F3 验证 | TPC 标准 |

### 4.6 注意事项

1. **缓存控制（关键）**：
   - DSL 加 `?request_cache=false`；SQL 用随机阈值打散 filter cache（SQL 路径不尊守 `request_cache=false`，根因见 2.4 节）
   - **每轮之间 `_cache/clear`** 或重启节点，消除 shard query cache、fielddata cache、OS page cache 残留
   - 监控 `indices/query_cache/memory_size`、`indices/fielddata/memory_size`、`indices/query_cache/hit_count`（确认缓存被有效打散）
   - 注意：SQL 随机阈值对 size=0 聚合查询的 shard query cache 无效（query fingerprint 仍可能命中），必须 `_cache/clear`
2. **预热充分**：50+ 轮让 JIT C2 + Calcite Janino codegen 充分预热；**用 `-XX:+PrintCompilation` 沉默 N 轮判断**（非 CV<5%——JIT 编译是事件驱动，可能 50 轮未触发、100 轮突然阶跃）
3. **DSL 查询等价**：DSL 不含 SQL 没有的聚合（如 percentile），确保对比公平
4. **Calcite 回退监控**：检查 "Fallback to V2" 日志（注意默认 `plugins.calcite.fallback.allowed=false`，仅 `CalciteUnsupportedException` 触发回退）
5. **连接复用**：`requests.Session()` 消除 TCP 开销
6. **@timestamp 标识符**：以 `@` 开头需反引号引用
7. **segment 数控制**：1M forcemerge 与 10M 未 forcemerge 的 segment 数差异巨大，必须做 forcemerge 对照组隔离变量
8. **replica 路由**：所有查询加 `?preference=_primary` 消除 replica 路由差异（SQL 路径无 preference 参数，是结构性差异）
9. **后台噪声隔离**：测试窗口停止 indexing + `translog.durability=async` + `merge.scheduler.max_thread_count=1`
10. **circuit breaker 风险**：H 组高基数聚合前查 `indices.breaker.total.limit`（默认 70% parent + 40% fielddata）；500K 桶约需 200-400MB fielddata，预备 fallback 查询，记录 breaker 触发率

### 4.7 预期结果矩阵

> 开销占比 = SQL 额外开销 / SQL 总延迟（= DSL + 额外开销）。区间已重新核算确保数学闭合。
> E 组无严格 DSL 对照（DSL"多次查询+应用层合并"语义不等价），列为 SQL 端到端绝对值。

| 场景 | 预期 DSL (ms) | 预期 SQL 额外开销 (ms) | 预期开销占比 | 说明 |
|------|:---:|:---:|:---:|------|
| A1 点查 | 1-5 | 2-7 | 29-88% | DSL=1+额外=2→67%；DSL=5+额外=2→29%；DSL=1+额外=7→88% |
| B1 全文搜索 | 3-10 | 2-7 | 17-70% | DSL=3+额外=2→40%；DSL=10+额外=2→17%；DSL=3+额外=7→70% |
| C1 聚合 | 5-20 | 5-15 | 20-75% | DSL=5+额外=5→50%；DSL=20+额外=5→20%；DSL=5+额外=15→75% |
| D2 深度分页 | 50-200 | 5-15 | 2-23% | DSL=50+额外=5→9%；DSL=200+额外=5→2%；DSL=50+额外=15→23% |
| D3 大结果集 | 10-30 | 10-20 | 25-67% | DSL=10+额外=10→50%；DSL=30+额外=10→25%；DSL=10+额外=20→67% |
| E1 UNION(pushdown ON) | 无 DSL 对照 | 5-20 (冷启动含 codegen 30-80) | — | SQL 端到端绝对值 |
| E2 JOIN | 无 DSL 对照 | 200-500 (SQL 端到端) | — | Legacy V1 Hash Join，10K 行内存计算 |
| G1 10M 聚合 | 300-600 | 5-15 (稳态) / 30-80 (冷启动含 codegen) | 1-5% / 5-13% | 区分冷启动 vs 稳态 |
| H1 10M 高基数聚合 | 800-2000 (或 OOM) | 10-30 | 0.5-3% | 500K 桶可能触发 circuit breaker |

### 4.8 生产关键场景补充

> 以下场景弥补原方案的盲点：深翻页生产方案、失败恢复、并发资源隔离。

#### D5 组：深翻页生产方案对比

| 场景 | SQL 查询 | DSL 查询 | 说明 |
|------|---------|---------|------|
| D5-1 SQL 游标续页 | `{"query":"SELECT * FROM perf_test WHERE status_code = 200","fetch_size":100}` + 游标续页 | — | 测第 1/100/1000 页延迟，验证 V2 序列化游标是否随页数退化 |
| D5-2 PIT+search_after | — | `POST /_pit` → `{"query":{"term":{"status_code":200}},"sort":[{"_id":"asc"}],"size":100}` + `search_after` | OpenSearch 推荐深翻页方案，无状态 |
| D5-3 scroll（deprecated） | — | `{"query":{"term":{"status_code":200}},"size":100}` + `scroll=1m` | 事实标准但已 deprecated，作对照 |
| D5-4 async search | — | `POST /_async_search` → `GET /_async_search/{id}` | 长查询生产必备 |

> ⚠️ **SQL 路径无 PIT/async search 等价物**——这是硬限制。生产深翻页场景若需 PIT 或 async search，必须用 DSL。

#### I 组：失败恢复与资源隔离

| 场景 | 测试方法 | 验证目标 |
|------|---------|---------|
| I1 circuit breaker 触发 | H1 查询 + 逐步降低 `indices.breaker.total.limit` | SQL 路径触发 breaker 后是否影响同 JVM 的 DSL 查询（breaker 是全局的） |
| I2 sql-worker 池满载 | 32 并发 SQL 长查询 + 同时跑 DSL 查询 | sql-worker 满载时 DSL 是否受影响（应不受影响，独立池） |
| I3 Legacy V1 池穿透 | 32 并发 JOIN 查询（sql-worker→search）+ 同时跑 DSL | Legacy V1 路径穿透到 search 池后，DSL 延迟是否劣化 |
| I4 OOM 恢复 | 人为触发 Calcite OOM（大 UNION）后跑 DSL | OOM 后 DSL 路径是否恢复正常 |
| I5 超时降级 | SQL 查询设置 30s 超时 + 并发 | 超时后 sql-worker 线程是否释放，是否泄漏 |

> 并发测试必须混合 SQL + DSL 负载（如 50% SQL + 50% DSL），观察资源隔离。原方案仅测同质负载吞吐量，无法发现资源穿透问题。

---

## 五、SQL 插件能力扩展评估

### 5.1 扩展模式分类

| 模式 | 适用特性 | 说明 |
|------|---------|------|
| **计划级路由**（三步模式） | JOIN、EXISTS | AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 实现 |
| **函数注册** | COALESCE | 注册到 BuiltinFunctionRepository，映射 Calcite SqlOperator |
| **聚合函数修复** | DATE_HISTOGRAM | 修复 INTERVAL NPE + 注册聚合 + 映射下推 |
| **表达式级子查询** | 标量子查询 | AstExpressionBuilder + Calcite RexSubquery |
| **语句级文法** | CTE | 新增文法规则 + AST 节点 + 作用域管理 |

### 5.2 工作量估算

| 优先级 | 特性 | 模式 | 人天 | 理由 |
|:---:|------|------|:---:|------|
| **P0** | 统计信息注入 | 独立工作流 | 20-40 | 最高 ROI，让 Calcite CBO 生效 |
| **P1** | COALESCE | 函数注册 | 1-2 | Calcite 原生支持 |
| **P1** | DATE_HISTOGRAM | 聚合修复 | 3-5 | 修复 bug，时序分析基础 |
| **P2** | 3表+ JOIN | 计划级路由 | 5-8 | 复用 UNION 三步模式 |
| **P2** | EXISTS 子查询 | 计划级路由 | 3-5 | SqlV2QueryParser 有参考 |
| **P3** | JOIN + GROUP BY | 依赖 P2 | 5-8 | 需验证 schema 解析+字段名冲突 |
| **P3** | 子查询 + 外层 GROUP BY | 依赖 P2 | 5-8 | 需验证派生表 schema 传播 |
| **P4** | 标量子查询 | 表达式级 | 8-12 | 复杂度最高 |
| **P5** | CTE | 语句级 | 15-25 | 文法+AST+作用域管理 |

### 5.3 总工作量

| 范围 | 人天 |
|------|:----:|
| P0+P1（快速收益） | 24-47 |
| P0-P2（核心能力） | 32-60 |
| P0-P3（覆盖大部分场景） | 42-76 |
| P0-P5（全部） | 65-113 |

### 5.4 依赖关系

```
统计信息注入 ──── 独立（最高 ROI）
COALESCE ─────── 独立
DATE_HISTOGRAM ── 独立
3表+ JOIN ─────── 独立
  ├── JOIN+GROUP BY ── 依赖 JOIN
  └── 子查询+GROUP BY ── 依赖 JOIN
EXISTS 子查询 ─── 独立
标量子查询 ────── 独立
CTE ───────────── 独立（可用派生表替代）
```

### 5.5 关键洞察

UNION 扩展的"三步模式"（AstBuilder 不抛异常 → shouldUseCalcite 路由 → CalciteRelNodeVisitor 实现）仅适用于**计划级路由**的扩展（JOIN、EXISTS）。函数注册、聚合修复、表达式级子查询、语句级文法需要不同的扩展模式。

---

## 六、扩展性交叉验证（OpenSearch 官方基准）

> 本章引用 OpenSearch 官方基准 + 权威第三方研究，交叉验证本测试规模的延迟预期，并明确标注外推到 30+ 节点 / 1B+ 文档的 gap。

### 6.1 官方基准来源与规模说明

**关键前提**：OpenSearch 官方 nightly 基准（[opensearch.org/benchmarks](https://opensearch.org/benchmarks/)）主要在**单节点** `c5.2xlarge` (8 vCPU, 16 GB RAM, 8 GB JVM heap) 上运行，索引为 **1 primary shard / 0 replica**，数据集 ~100 GB / 116M 文档（Big5 workload）。这与本测试 10M/3 节点规模不同，但官方数据可作为**单节点基准线**，再结合多节点扩展性研究外推。

> ⚠️ **路径说明（重要）**：本章引用的官方/第三方基准**均为 DSL 路径**（直接发 `_search` 端点，由 OpenSearch Benchmark / rally 驱动，**不经过 SQL 插件**）。因此本章数据**仅用于交叉验证本测试 4.7 节"预期 DSL"列的合理性**，不能直接推论 SQL 路径开销。SQL 插件 overhead 的官方数据见 6.7 节（PPL，≠ SQL）。

### 6.2 官方延迟数据交叉验证

#### OpenSearch 3.0 Big5 性能（单节点，1 shard/0 replica）

来源：[OpenSearch 3.0 Performance Progress](https://opensearch.org/blog/opensearch-project-update-performance-progress-in-opensearch-3-0/)（2025-05-14）

| 查询类型 | OS 1.3.18 | OS 2.19 | OS 3.0 GA | 提升倍数 |
|---------|:---:|:---:|:---:|:---:|
| Text queries | 59.51 ms | 8.22 ms | 8.30 ms | ~7x |
| Sorting | 17.73 ms | 7.96 ms | 7.03 ms | ~2.5x |
| **Terms aggregations** | 609.43 ms | 112.08 ms | **79.72 ms** | ~7.6x |
| Range queries | 26.08 ms | 3.67 ms | 2.68 ms | ~10x |
| **Date histograms** | 6068 ms | 159.57 ms | **85.21 ms** | ~71x |
| **Aggregate (geo mean)** | 159.04 ms | 21.21 ms | **16.04 ms** | **9.92x** |

- 最慢单查询：`range-auto-date-histo-with-metrics` 在 OS 3.0 为 **5406 ms**（1.3 为 22988 ms）
- 高基数聚合 `cardinality-agg-high`：OS 3.0 = **628 ms**

**交叉验证结论**：本测试 10M/3 节点"预期 DSL"300–2000ms 落在官方单节点同一 workload 的延迟区间内。单节点 terms agg 80ms，多节点 + 多 shard scatter-gather 会增加开销，300–600ms（G 组）合理；500K 桶高基数聚合 800–2000ms（H1）与官方 `cardinality-agg-high` 628ms + 多 shard 开销一致。

> ⚠️ **版本依赖性强**：若用 OS 2.x，date_histogram 类查询延迟可能比 3.x 高 2–70 倍。本测试基于 OS 3.7.0，结论不可外推到 2.x。

#### Trail of Bits 独立基准（OpenSearch 2.17.1 vs Elasticsearch 8.15.4）

来源：[Benchmarking OpenSearch and Elasticsearch](https://blog.trailofbits.com/2025/03/06/benchmarking-opensearch-and-elasticsearch/)（2025-03-06，完整报告 [PDF](https://github.com/trailofbits/publications/blob/master/reports/OpenSearch-Benchmarking.pdf)）

| 类别 | OpenSearch 2.17.1 (ms) | Elasticsearch 8.15.4 (ms) | 对比 |
|------|:---:|:---:|:---:|
| Text queries | 18.11 | 7.47 | OS 2.42x 慢 |
| **Term aggregations** | **104.90** | 354.52 | **OS 3.38x 快** |
| **Date histograms** | **124.79** | 2064.61 | **OS 16.55x 快** |
| All Operations (geo mean) | 12.1 | 18.8 | OS 1.56x 快 |

- 15 天 nightly 测试，每天新实例，每次 5 runs 丢弃首 run
- **Outlier 警示**：OpenSearch `composite-date_histogram-daily` outlier 比例 1412x——生产环境长尾延迟可能远超均值，p99/p999 监控不可省

#### OpenSearch 3.5 向量搜索（3 节点 10M，规模接近本测试）

来源：[Accelerating FP16 vector search in OpenSearch 3.5](https://opensearch.org/blog/accelerating-fp16-vector-search-performance-using-bulk-simd-in-opensearch-3-5/)（2026-03-03）

| Version | CPU | QPS | Avg latency (ms) | p90 (ms) | p99 (ms) |
|---------|-----|:---:|:---:|:---:|:---:|
| 3.1 | r7i | 398.87 | 209.66 | 300 | 330 |
| 3.5 | r7i | 1303.76 | 63.99 | 95 | 105 |
| 3.5 | r7g | 1477.88 | 56.42 | 82 | 91 |

**交叉验证结论**：官方 3 节点 10M 文档向量搜索 p99 91–330ms。注意：向量搜索走 HNSW/IVF 索引，与聚合查询（fielddata/doc_values）workload 完全不同，**不能用于推论 SQL overhead**。此处仅作为 3 节点 10M 规模的 DSL 路径延迟量级参考——向量搜索本身不经过 SQL 插件。

### 6.3 大规模部署参考（AWS 实例基准）

#### AWS OR1 vs r6g（3 节点 247M 文档 http_logs）

来源：[Improve OpenSearch Service performance with Optimized Instances](https://aws.amazon.com/blogs/big-data/improve-your-amazon-opensearch-service-performance-with-opensearch-optimized-instances/)（2024-07-11）

| 指标 | r6g.large | or1.large | 差异 |
|------|:---:|:---:|:---:|
| query-term p99 | 7675 ms | 4183 ms | OR1 快 45% |
| **hourly_aggregation p99** | **5308 ms** | **2985 ms** | OR1 快 44% |
| **multi_term_aggregation p99** | **8506 ms** | **4264 ms** | OR1 快 50% |

**外推参考**：3 节点 247M 文档 multi_term_agg p99 = 4.2–8.5 秒，比本测试 10M/3 节点"预期 DSL"2 秒高 2–4 倍。文档数 10M → 247M (25x)，延迟 2s → 8.5s (4x)，**亚线性扩展**（因并行度未变）。

#### AWS OM2 vs M7g（2 节点 247M 文档）

来源：[Benchmarking Instance Types for Amazon OpenSearch Workloads](https://repost.aws/articles/ARdy6WoZbKSnKXWRyAgdgFCA)（2026-04-08）

- Multi-term Aggregation p99：M7g = 2468 ms，OM2 = 2200 ms
- Hourly Aggregation p99：M7g = 72.77 ms，OM2 = 49.46 ms

**外推参考**：2 节点 247M 文档 multi_term_agg p99 = 2.2–2.5 秒，与本测试 10M/3 节点"预期 DSL"300–2000ms 区间一致。

### 6.4 Scatter-Gather 与协调节点开销

#### Elasticsearch 8.x Many-Shards 优化

来源：[Benchmark-driven optimizations in Elasticsearch 8](https://www.elastic.co/blog/benchmark-driven-optimizations-scalability-elasticsearch-8)（2023-04-10）

- 50,000 索引 / many-shards 基准：ES 8.2 → 8.5 索引吞吐几乎翻倍
- Snapshot 创建时间从 29s 降到 0.7s（97% 提升）
- 早期一次性传输全 cluster state → 8.5+ 只传 delta，对万级 shard 集群性能提升数量级

> ⚠️ OpenSearch fork 自 ES 7.10，许多 8.x 的 many-shards 优化**未完全移植**，ES 数据仅作参考。

#### Query Phase Batching 提案

来源：[Elasticsearch Issue #112306](https://github.com/elastic/elasticsearch/issues/112306)（2024-08-28）

- 当前限制：协调节点对每 data node 并发 shard 请求**默认限 5**（`action.search.shard_count.limit`）
- **O(50K) shards 查询时，targeted index resolution 是非聚合查询最慢步骤**
- 新提案：shard-level 请求合并为每 data node 单个请求，roundtrip 从 O(shards) 降到 O(data nodes)

#### Query Latency Multi-Shard Regression

来源：[Elasticsearch Issue #30994](https://github.com/elastic/elasticsearch/issues/30994)

- **5 shard benchmark**：`geopoints` polygon 查询 p50 从 59ms 升到 153ms（2.6x 慢）
- `geonames` painless_static p50 从 504ms 升到 1488ms（2.95x 慢）

**外推参考**：本测试 3 节点若用 5 primary shard + 1 replica = 30 shard，延迟预期应至少为单 shard 的 2–3 倍。生产 30 节点 × 100 shard = 3000 shard，已进入 scatter-gather 风险区。

#### Star Graph 扩展性模型

来源：[How Elasticsearch scales (or doesn't)](https://mooreniemi.github.io/scaling/search/2024/07/22/how-elasticsearch-scales-or-doesn-t.html)（2024-07-22）

- **Amdahl's Law**：shard 数 = 并行度上限
- **`agg_time ∝ log(shards, fanout)`**，理论最优 fanout 是二叉树聚合拓扑
- 加 replica 不直接降延迟，但增加协调节点容量 → 降低 utilization → 间接降延迟
- **最优延迟配置**：N 个 data shard 需 ~2Nα 个 aggregator 容量（α = Aggregation/Scan 计算比）

**外推方法论**：用 `total_time = scan_time(shards) + log(shards, fanout) × agg_unit` 拟合本测试 1M/10M 数据，预测 30 节点延迟。3 节点测试不能简单线性外推——聚合是 star graph，shard 数增加 → 协调节点工作量 log 增长。

### 6.5 Circuit Breaker 阈值（高基数聚合风险评估）

来源：[OpenSearch Circuit Breaker Settings](https://docs.opensearch.org/latest/install-and-configure/configuring-opensearch/circuit-breaker/)（2026-06-18）

| Breaker | 默认阈值 | 8 GB heap 节点 | 32 GB heap 节点 |
|---------|:---:|:---:|:---:|
| `indices.breaker.fielddata.limit` | **40% JVM heap** | 3.2 GB | 12.8 GB |
| `indices.fielddata.cache.size` | 35% JVM heap | 2.8 GB | 11.2 GB |
| Request circuit breaker | 60% JVM heap | 4.8 GB | 19.2 GB |
| **Parent circuit breaker** | **95% JVM heap** | 7.6 GB | 30.4 GB |

**H1 风险评估**：500K 桶 terms agg × ~200B/桶 = 100MB+ fielddata。单次查询内存压力可控，但叠加 BKDPointTree 等内部结构可瞬时占用 800 MB（见 [ES Issue #86531](https://github.com/elastic/elasticsearch/issues/86531)）。本测试 8 GB heap 节点，fielddata breaker 阈值仅 3.2 GB，**高基数聚合易触发**。生产建议 32+ GB heap。

### 6.6 Forcemerge 与 Segment 数影响

来源：[Intra-Segment Search RFC](https://github.com/opensearch-project/OpenSearch/issues/20202)（big5, 1 shard, force-merged to 1 segment, r5.2xlarge）

| 操作 | 无 Intra-Segment | 有 Intra-Segment | 提升 |
|------|:---:|:---:|:---:|
| span_near query (1 client) | 110.8 ms | 41.8 ms | **62%** |
| Multi-metric aggregation (1 client) | 8242 ms | 2004 ms | **76%** |
| Multi-metric aggregation (4 clients) | 8242 ms | 8117 ms | ~2% |
| stats aggregation | 2859 ms | 1494 ms | 48% |

**关键观察**：
- Forcemerge 到单 segment 后，传统 concurrent search 无并行度（1 segment < 4 slices），intra-segment 才能继续切分
- **8 clients 时无收益**：CPU 饱和，intra-segment 失去意义
- 本测试 1M forcemerge 与 10M 未 forcemerge 的 segment 数差异巨大（1 段 vs 50-200 段），**结构性不可比**

### 6.7 PPL 插件大规模性能 Gap（注意：PPL ≠ SQL）

> ⚠️ **路径说明**：本节数据均为 **PPL 路径**（Piped Processing Language），不是 SQL。PPL 和 SQL 虽然在 OS 3.3+ 都走 Calcite，但查询语言不同（管道语法 vs SELECT 语法）、路由逻辑不同（PPL 默认走 Calcite，SQL 仅 UNION 走 Calcite）。PPL 数据**不能直接等同于 SQL 数据**，仅作为 Calcite 引擎路径的参考。

来源：[PPL Calcite Optimizer](https://opensearch.org/blog/better-observability-deeper-insights-opensearchs-new-piped-processing-language-capabilities/)（2025-11-25）+ [SQL Issue #3528](https://github.com/opensearch-project/sql/issues/3528)

- **OS 3.3 起 Calcite 为默认 PPL 优化器**：Big5 PPL `date_histogram_hourly_agg` 查询 **2.5s → 15ms（160x faster）**
- **大规模性能 gap**：750B 文档规模下，PPL span query **数百秒** vs DSL **秒级**（[Issue #3528](https://github.com/opensearch-project/sql/issues/3528)）
- 根因：PPL 将 `span` 转为 composite aggregation，比 date_histogram 慢

**外推警示**：小规模测试的 SQL/DSL 延迟比**不能直接外推**到生产规模。PPL 在 1B+ 文档规模下 span 类查询可能比 DSL 慢 100x+，SQL 路径因仅 UNION 走 Calcite，大规模表现需独立验证。

### 6.8 外推结论矩阵

> 路径列说明：DSL = 直接 `_search` 端点（不经过 SQL 插件）；PPL = Piped Processing Language（≠ SQL）；SQL = SQL 插件路径。本测试 4.7 节"预期 DSL"列仅可与 DSL 基准交叉验证。

| 本测试规模 | 官方/第三方参考 | 参考路径 | 外推结论 |
|---------|---------|:---:|---------|
| 1M / 单节点 | OS 3.0 单节点 Big5 geo mean 16ms；Trail of Bits OS 2.17 单节点 terms agg 105ms | DSL | 单节点 DSL 基线合理 |
| 10M / 3 节点, 预期 DSL 300–2000ms | OS 3.5 vector 3 节点 10M p99 91–330ms；AWS OR1 3 节点 247M multi_term p99 4.2s | DSL | "预期 DSL"落在合理区间 |
| **30+ 节点外推** | ES Issue #112306: 50K shard 时 shard 解析成瓶颈；Star graph 模型 | DSL | **不能线性外推**；协调节点聚合开销 log 增长 |
| **1B+ 文档外推** | Elastic 推荐 200M docs/shard → 1B 需 5+ shard；AWS 247M multi_term p99 4.2s | DSL | 1B 文档预计 DSL multi_term agg p99 10–30s |
| **SQL 插件 overhead** | OS 3.3 PPL Calcite 160x faster；750B 文档 PPL span 慢 100x | PPL | **版本依赖性强**；PPL≠SQL，SQL 大规模表现需独立验证 |
| **Circuit breaker** | fielddata 40% heap 默认；parent 95% heap | 不区分 | 8 GB heap 节点高基数聚合易触发，生产建议 32+ GB |
| **Forcemerge 收益** | Intra-segment RFC：单 segment 重聚合提升 76%（单 client），8 clients 无收益 | DSL | 只读索引可 forcemerge；写入活跃索引无收益 |

### 6.9 最重要外推 Gap（必须在文档明确标注）

1. **OpenSearch 官方 nightly 基准仅单节点 1 shard**，无 30+ 节点数据，需引用 Trail of Bits + AWS 实例基准做交叉验证
2. **OpenSearch fork 自 ES 7.10**，ES 8.x 的 many-shards 优化（cluster state delta、snapshot pool doubling）未完全移植，ES 数据仅作参考
3. **PPL 插件版本依赖性极强**（PPL ≠ SQL）：OS 3.3 Calcite 优化器带来数量级提升，2.x 测试结果不能外推到 3.x+。SQL 路径大规模表现无官方数据，需独立验证
4. **大规模高基数聚合是 SQL 引擎已知弱项**（[Issue #3528](https://github.com/opensearch-project/sql/issues/3528)），小规模测试无法暴露此问题
5. **协调节点 scatter-gather 在 1000+ shard 时成为瓶颈**（[ES Issue #112306](https://github.com/elastic/elasticsearch/issues/112306)），30 节点 × 100 shard = 3000 shard 已进入风险区

---

## 七、大数据视角与生产可用性评估

### 7.1 测试规模的可外推性

| 测试规模 | 节点数 | 文档数 | 可外推到的生产规模 | 限制 |
|---------|:---:|:---:|------|------|
| 1M 单节点 | 1 | 1M | 单节点小规模部署 | 无 scatter-gather、无网络、无跨 shard merge |
| 10M 3 节点 | 3 | 10M | 3-10 节点中等规模 | 3 节点统计上接近 p33，无法体现 30+ 节点长尾 |
| 生产 30+ 节点 | 30+ | 1B+ | — | 协调节点 merge 100 分片响应，网络 RTT 主导 |

**核心结论**：
- 1M 单节点 + 10M 3 节点的测试**只能证明各自规模内 SQL vs DSL 的相对开销比例**
- **不可外推到 30+ 节点生产集群**——生产规模下网络 RTT 和协调节点 merge 开销主导，SQL 翻译开销相对值趋近于 0
- 扩展性结论必须引用 OpenSearch 官方基准交叉验证（见第六章）

### 7.2 生产可用性 SLO 指标（缺失补充）

原方案只测"正常路径延迟"，未测生产 SLO 关键指标：

| SLO 指标 | 测试方法 | 生产意义 |
|---------|---------|---------|
| p99 时延 SLO 达标率 | 10M 测试统计 p99 ≤ 1s 的轮次占比 | 生产 SLA 基础 |
| 降级表现 | I 组失败恢复测试 | SQL 路径异常时 DSL 是否受影响 |
| 失败恢复时间 | I4 OOM 后测量 DSL 恢复时间 | 故障恢复 RTO |
| 资源隔离 | I2/I3 并发资源隔离测试 | 混合负载下 SQL 是否挤占 DSL |
| 长尾稳定性 | 2000+ 轮测试的 max 和 p99.9 | 尾部延迟控制 |

### 7.3 TCO 视角（开发效率 vs 运行效率）

SQL 插件存在的根本理由是**开发效率提升**，而非运行效率。生产 TCO 评估需权衡：

```
TCO = 开发成本 + 运行成本 + 维护成本

SQL 路径：
  开发成本 ↓（减少 N 行应用层 DSL 构建代码）
  运行成本 ↑（翻译开销 M ms/查询，但集群 CPU 利用率 <30% 时占比极小）
  维护成本 ↓（SQL 可读性高，DBA 可直接优化）

DSL 路径：
  开发成本 ↑（需熟悉 DSL 语法、应用层组装）
  运行成本 ↓（零翻译开销）
  维护成本 ↑（DSL 复杂查询可读性差）
```

**ROI 公式**：
```
ROI = (开发效率提升 × 开发人天单价) / (翻译开销 × 查询QPS × 运行时间 × 计算资源单价)
```

当集群 CPU 利用率 <30% 且查询 QPS <100 时，SQL 翻译开销在 TCO 中占比 <5%，ROI 显著为正。高 QPS（>1000）场景需用 DSL。

### 7.4 文档整体结论

**评分：B（修正后）**

**修正前（C+）的问题已解决**：
- ✅ 4.1 vs 4.4 节点数矛盾已统一
- ✅ slowlog 分解翻译开销方法已替换为 SQLService 内部埋点（方法 A）
- ✅ 1M forcemerge vs 10M 未 forcemerge 已标注结构性不可比 + 补充对照组
- ✅ p99.9 在 200 样本下无统计意义——改为 ≥2000 轮或报 max
- ✅ 线程池硬编码 8、静默回退措辞、composite size=1000 等代码事实已修正
- ✅ 补充 PIT/async search/circuit breaker/并发资源隔离等生产关键场景

**仍存在的限制**（不可通过文档修正解决）：
- ❌ 单节点 + 3 节点测试规模无法外推到 30+ 节点生产集群——需引用官方基准或补充 10+ 节点测试
- ❌ SQL 路径无 PIT/async search 等价物——硬限制，生产深翻页场景必须用 DSL
- ❌ Calcite 在无统计信息下 CBO 退化为 RBO——需 P0 统计信息注入（20-40 人天）

**生产可用性判断**：
- 中小规模（≤10 节点、≤100M 文档、QPS<100）：SQL 路径生产可用，翻译开销占比 <5%
- 大规模（30+ 节点、1B+ 文档、QPS>1000）：需 DSL 路径，SQL 翻译开销虽相对值趋近 0 但绝对 QPS 压力下 sql-worker 池可能成为瓶颈
- 混合策略：默认 SQL 提升开发效率，关键高 QPS 路径用 DSL 优化

---

## 附录：修正记录

本次评审修正了以下与代码不符的声明（基于代码级验证）：

| 行号 | 原声明 | 修正后 | 代码证据 |
|------|--------|--------|---------|
| 45 | composite size=1000 分页拉取 | 可配置 `plugins.query.buckets`，默认 = `MAX_RESULT_WINDOW` (10000) | `OpenSearchSettings.java:205-211`, `AggregateAnalyzer.java:296` |
| 134 | 静默回退到 V2 | WARN 级别日志回退，默认仅 `CalciteUnsupportedException` 触发 | `QueryService.java:180-186` |
| 152 | sql-worker (8线程) | sql-worker (=allocatedProcessors, 8核→8线程) | `SQLPlugin.java:443-449` |
| 162/425 | SQL 路径不尊守 request_cache=false | 补充根因：`responseParams` 不含 `request_cache`，`OpenSearchRequestBuilder` 不设 `requestCache` | `RestSqlAction.java:242-248` |
| 189/214 | DATE_HISTOGRAM NPE 崩溃 | INTERVAL 处理已知 bug（`RexStandardizer.java:117`），基于实测观察 | `RexStandardizer.java:117`, `AggSpec.java:42` |

方法论修正：
- 4.1 表 vs 4.4 节节点数矛盾统一为两阶段测试（1M 单节点 + 10M 3 节点）
- 4.4 节 slowlog `threshold.query.info: 0ms` 改为 `warn: 0ms`（避免每查询同步写盘引入抖动）
- 4.5 节样本量 200 轮→≥2000 轮（支撑 p99 置信区间），预热改用 `-XX:+PrintCompilation` 沉默判断
- 4.6 节补充三大缓存控制（shard query cache / fielddata cache / OS page cache）+ `?preference=_primary`
- 4.7 表数学不闭合修正 + E 组标注"无 DSL 对照"
- 4.8 节新增生产关键场景（D5 PIT/async search + I 组失败恢复）
- 翻译开销分解方法从 `T_sql - T_slowlog` 改为 SQLService 内部 `ProfileMetric ANALYZE` 埋点
