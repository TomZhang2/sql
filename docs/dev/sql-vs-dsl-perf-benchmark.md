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
- **聚合策略**：使用 `composite` 聚合（size=1000 分页拉取）

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

> ⚠️ Calcite 失败时静默回退到 V2（`QueryService.java:181-183`），需监控日志。

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
| 线程池 | search (13线程) | sql-worker (8线程) | sql-worker (8线程) | sql-worker→search |
| 总额外开销 | ~0ms | ~2-7ms | ~5-18ms | ~2-6ms |

### 2.4 关键差异点

1. **SQL 最终都翻译成 DSL**——差异只在上层翻译开销
2. **Calcite 下推**——Convention trait 驱动，规则非代价；不可下推的操作在内存单线程计算
3. **游标**——DSL `search_after`（无状态）vs V2 序列化游标（有状态）vs Calcite `EnumerableLimit`
4. **无计划缓存**——每次查询重新解析+规划
5. **Calcite 内存风险**——UNION/JOIN 在协调节点单线程，大数据量可能 OOM
6. **缓存公平性**——SQL 路径不尊守 `request_cache=false`，filter cache 对 SQL 生效；基准测试需用随机阈值打散

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
| JOIN + GROUP BY | ❌ | ❌ | SQL 报错 "no aggregations" |
| UNION ALL | ✅ | ❌ | SQL（我们的扩展，走 Calcite） |
| UNION DISTINCT | ✅ | ❌ | SQL（我们的扩展，走 Calcite） |
| IN 子查询 | ✅ | ❌ | SQL（回退 Legacy V1 Hash Join） |
| EXISTS 子查询 | ❌ | ❌ | SQL 报错 "Unsupported subquery" |
| 派生表 | ✅ | ❌ | SQL `(SELECT...) AS t` |
| CTE (WITH) | ❌ | ❌ | SQL 报错 "must start with SELECT" |
| COALESCE | ❌ | ✅ | SQL 报错 "not supported in Schema" |
| DATE_HISTOGRAM | ❌ | ✅ | SQL NPE 崩溃 |
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
| CTE | `Query must start with SELECT` | 文法无 WITH 规则 |
| 3 表+ JOIN | `only 2 tables join` | Legacy V1 限制 |
| JOIN + GROUP BY | `no aggregations on joined result` | Legacy V1 限制 |
| EXISTS 子查询 | `Unsupported subquery` | V2 不支持 |
| 标量子查询 | `unknown field name` | Druid 限制 |
| COALESCE | `not supported in Schema` | V2 函数注册表缺失 |
| DATE_HISTOGRAM | NPE | V2 INTERVAL 参数解析 bug |

---

## 四、验证方案设计

### 4.1 测试环境

| 项目 | 1M 测试 | 10M 测试 |
|------|---------|---------|
| 节点数 | 1 | 1 |
| Shard 数 | 3 | 6 |
| 文档数 | 1,000,000 | 9,900,000 |
| 字段数 | 10 | 13（含高基数 user_id=100K, session_id=500K） |
| forcemerge | 是 | 否 |
| 预热 | 20 轮（简单）/ 5 轮（复杂） | 3-5 轮 |
| 测试轮数 | 100 轮 / 20 轮 | 10-30 轮 |

> ⚠️ **单节点测试局限性**：无 scatter-gather、无跨节点网络延迟、无跨 shard merge。1M 和 10M 使用不同 schema 和 shard 数，两者不完全可比。

### 4.2 轻查询场景（1M 数据，返回 ≤10 行）

> **对等性原则**：DSL 用 `bool.filter`（不打分），SQL `WHERE` 生成 `bool.filter`，语义对等。
> **缓存控制**：DSL 加 `?request_cache=false`；SQL 用随机阈值打散 filter cache。

#### A 组：点查

| 场景 | SQL | DSL |
|------|-----|-----|
| A1 等值(单条件) | `SELECT * FROM perf_test WHERE status_code=200 LIMIT 10` | `{"term":{"status_code":200},"size":10}` |
| A2 等值(多条件) | `WHERE status_code=500 AND level='ERROR' AND region='us-east-1' LIMIT 10` | `bool.filter` 3 个 term |
| A3 范围 | `WHERE response_time_ms>3000 AND status_code=500 LIMIT 10` | `bool.filter` range+term |

#### B 组：全文搜索

| 场景 | SQL | DSL |
|------|-----|-----|
| B1 简单 | `WHERE match(message,'timeout') LIMIT 10` | `{"match":{"message":"timeout"}}` |
| B2 多字段 | `WHERE MULTI_MATCH(message,'request failed') AND level='ERROR' LIMIT 10` | `bool.must(match)+filter(term)` |

#### C 组：聚合（随机阈值打散缓存）

| 场景 | SQL | DSL |
|------|-----|-----|
| C1 简单 | `SELECT level,COUNT(*) WHERE response_time_ms>{random} GROUP BY level` | `terms` agg |
| C2 多级 | `SELECT level,service,COUNT(*),AVG(rt) WHERE rt>{random} GROUP BY level,service` | `composite` agg（与 SQL flat GROUP BY 对等） |
| C3 范围+排序 | `SELECT service,COUNT(*) WHERE rt>{random} GROUP BY service ORDER BY COUNT(*) DESC` | `terms` agg with order |
| C4 时间直方图 | SQL 不支持（NPE） | DSL `date_histogram`（仅 DSL） |

#### D 组：排序与分页

| 场景 | SQL | DSL |
|------|-----|-----|
| D1 排序+小分页 | `ORDER BY response_time_ms DESC LIMIT 10` | `sort` + `size:10` |
| D2 深度分页 | `LIMIT 10000,10`（需 max_result_window:20000） | `from:10000,size:10` |
| D3 大结果集 | `WHERE status_code=200 LIMIT 1000` | `term` + `size:1000` |
| D4 游标分页 | `fetch_size:100` 多页 | `search_after` 多页 |

#### E 组：SQL 独有（无 DSL 对照）

| 场景 | SQL | 引擎 |
|------|-----|------|
| E1 UNION ALL+聚合 | 两路聚合 UNION ALL | Calcite |
| E2 2表JOIN | `JOIN perf_test_meta ON host` | Legacy V1 |
| E3 IN子查询 | `WHERE host IN (SELECT...)` | Legacy V1 |

#### F 组：辅助验证

| 场景 | 说明 |
|------|------|
| F1 冷启动 | 每轮唯一注释 `/* cold_run_{ts} */` 打散计划缓存 |
| F2 Pushdown on/off | 对比 Calcite 下推开关，隔离下推收益 |
| F3 结果等价 | 验证 SQL 和 DSL 返回相同结果（TPC 标准） |

### 4.3 重查询场景（1M 数据，返回 5K-10K 行）

> 通过高命中率 + 大结果集 + 高基数聚合 + 深度翻页，让 DSL 执行时间达到 100-400ms。

#### A-H 组：大结果集点查

| 场景 | SQL | DSL | 预期 DSL |
|------|-----|-----|:---:|
| A1-H | `WHERE status_code=200 ORDER BY rt DESC LIMIT 10000` | term+sort+size:10K | 80-150ms |
| A2-H | `WHERE status_code IN(...) AND level IN(...) ORDER BY bytes DESC LIMIT 5000` | terms+sort+size:5K | 60-120ms |
| A3-H | `WHERE rt>2000 ORDER BY @timestamp DESC LIMIT 10000` | range+sort+size:10K | 80-150ms |

#### B-H 组：高命中全文搜索

| 场景 | SQL | DSL | 预期 DSL |
|------|-----|-----|:---:|
| B1-H | `WHERE match(message,'request') ORDER BY rt DESC LIMIT 5000` | match+sort+size:5K | 60-120ms |
| B2-H | `WHERE match(message,'request failed timeout') AND status_code>=400 ORDER BY bytes DESC LIMIT 10000` | bool+sort+size:10K | 80-150ms |

#### C-H 组：高基数聚合

| 场景 | SQL | DSL | 预期 DSL |
|------|-----|-----|:---:|
| C1-H | `GROUP BY user_id(100K桶) + 4指标 LIMIT 1000` | terms(size:1000)+4 sub-aggs | 100-300ms |
| C2-H | `GROUP BY level,service,region(100桶) + 多指标` | nested aggs + percentile | 100-250ms |
| C3-H | SQL 不支持 | `date_histogram` + 二级聚合 | 150-400ms（仅 DSL） |

#### D-H 组：深度分页 + 大结果集

| 场景 | SQL | DSL | 预期 DSL |
|------|-----|-----|:---:|
| D1-H | `ORDER BY rt ASC LIMIT 19990,10` | sort+from:19990 | 100-200ms |
| D2-H | `WHERE status_code=200 ORDER BY rt DESC LIMIT 10000` | term+sort+size:10K | 80-150ms |
| D3-H | `WHERE rt>1000 AND status_code IN(200,500) ORDER BY rt DESC LIMIT 10000` | bool+sort+size:10K | 80-150ms |

#### E-H 组：SQL 独有大数据量

| 场景 | SQL | 引擎 | 预期 SQL |
|------|-----|------|:---:|
| E1-H 三路UNION+聚合 | 3 路聚合 UNION ALL | Calcite | 20-50ms |
| E1b-H 250K行UNION | 两路大结果集 UNION ALL | Calcite | 100-300ms |
| E2-H JOIN 10K行 | `JOIN perf_test_meta LIMIT 10000` | Legacy V1 | 200-500ms |
| E3-H IN子查询 10K行 | `WHERE host IN (SELECT...) LIMIT 10000` | Legacy V1 | 200-500ms |

### 4.4 10M 多节点生产级验证方案

> 目标：验证"大数据量重查询下 SQL 与 DSL 性能差异不大"。

#### 环境

```
3 节点集群，每节点 8C 16G（JVM heap 8GB）
6 shard 1 replica，10M 文档（13 字段含高基数）
未 forcemerge（模拟生产）
slowlog 开启（threshold.query.info: 0ms）
```

#### 场景组 G：生产典型重查询

| 场景 | SQL | DSL | 预期 DSL |
|------|-----|-----|:---:|
| G1 时间范围+聚合 | 7 天数据按 service×level 聚合 | range+nested aggs | 300-600ms |
| G2 多字段排序+10K | ERROR 日志多字段排序 | bool+sort+size:10K | 200-500ms |
| G3 高基数聚合+过滤 | 按 user_id(100K桶) 聚合 Top1000 | terms(size:1000)+4 aggs | 300-600ms |
| G4 复合+5K | 多条件+排序+5000 行 | bool+sort+size:5K | 150-400ms |

#### 场景组 H：极重查询

| 场景 | SQL | DSL | 预期 DSL |
|------|-----|-----|:---:|
| H1 全表500K桶聚合 | 按 session_id 聚合 Top5000 | terms(size:5000) | 800-2000ms |
| H2 全表多级聚合 | service×level×region + percentile | nested aggs+percentile | 600-1500ms |
| H3 大范围+10K | 7 天数据排序取 10K | range+sort+size:10K | 400-800ms |

#### 翻译开销分解方法

通过 slowlog 分解：
1. 运行 SQL 查询，记录端到端延迟 `T_sql`
2. 从 slowlog 提取服务端执行时间 `T_exec`
3. 翻译开销 = `T_sql - T_exec`
4. 运行等价 DSL，记录 `T_dsl`
5. 验证 `T_exec ≈ T_dsl`

> ⚠️ **不使用** `_explain` 提取的 DSL 来测执行时间——explain 输出与实际执行 DSL 可能不同。

### 4.5 测试方法

#### 压测脚本框架

```python
session = requests.Session()  # 复用 TCP 连接

def bench(name, fn_factory, warmup=50, runs=200):
    """fn_factory(threshold) 返回可调用对象，threshold 用于随机化打散缓存"""
    for _ in range(warmup):
        fn_factory(random.randint(1, 9000))()
    latencies = []
    for _ in range(runs):
        t = random.randint(1, 9000)
        t0 = time.perf_counter()
        r = fn_factory(t)()
        dt = (time.perf_counter() - t0) * 1000
        if r.status_code == 200: latencies.append(dt)
    # 计算 p50/p99/p99.9
```

#### 执行步骤

```
1. 启动集群 → 创建索引 → 导入数据 → 轮询确认
2. 结果集等价验证（F3）
3. 预热（50 轮，滑动窗口 CV <5% 判断完成）
4. 正式测试（200 轮，聚合场景随机阈值打散缓存）
5. 并发吞吐量测试（8/16/32 并发，60 秒）
6. Pushdown on/off 对比（F2）
7. 两遍运行（第一遍丢弃，验证 JIT 稳定性）
8. 收集指标 + 关联 JIT/GC 日志
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

1. **缓存控制**：DSL 加 `?request_cache=false`；SQL 用随机阈值打散 filter cache（SQL 路径不尊守 `request_cache=false`）
2. **预热充分**：50 轮让 JIT C2 + Calcite Janino codegen 充分预热；用滑动窗口 CV 判断
3. **DSL 查询等价**：DSL 不含 SQL 没有的聚合（如 percentile），确保对比公平
4. **Calcite 回退监控**：检查 "Fallback to V2" 日志
5. **连接复用**：`requests.Session()` 消除 TCP 开销
6. **@timestamp 标识符**：以 `@` 开头需反引号引用

### 4.7 预期结果矩阵

| 场景 | 预期 DSL (ms) | 预期 SQL 额外开销 (ms) | 预期开销占比 |
|------|:---:|:---:|:---:|
| A1 点查 | 1-5 | 2-7 | 40-63% |
| B1 全文搜索 | 3-10 | 2-7 | 20-67% |
| C1 聚合 | 5-20 | 5-15 | 25-80% |
| D2 深度分页 | 50-200 | 5-15 | 3-30% |
| D3 大结果集 | 10-30 | 10-20 | 33-100% |
| E1 UNION(pushdown ON) | — | 5-20 | — |
| E2 JOIN | — | 60-100 | — |
| G1 10M 聚合 | 300-600 | 5-15 | 1-5% |
| H1 10M 高基数聚合 | 800-2000 | 10-30 | 0.5-3% |

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
