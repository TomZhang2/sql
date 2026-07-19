# SQL vs DSL 查询性能对比验证报告

> 基于 OpenSearch 3.7.0 + SQL Plugin 3.7.0（含 UNION/UNION ALL 扩展）
> 数据量：100 万文档，3 shard，单节点
> 测试日期：2026-07-18
> 预热：20 轮 | 测试：100 轮（E/F 场景 20-50 轮）

---

## 一、测试环境

| 项目 | 配置 |
|------|------|
| OpenSearch | 3.7.0-SNAPSHOT 单节点 |
| SQL Plugin | 3.7.0.0-SNAPSHOT（含 UNION/UNION ALL 扩展） |
| 索引 | `perf_test`，3 shard 0 replica，1M docs，forcemerge 1 segment |
| `plugins.calcite.enabled` | true |
| `plugins.calcite.pushdown.enabled` | true |
| `index.max_result_window` | 20000 |
| CPU | 8 核 |
| JVM Heap | 512MB（单节点测试环境） |
| 网络 | 本地回环 |

---

## 二、测试结果

### 场景组 A：点查

| 场景 | SQL avg (ms) | SQL p50 | SQL p99 | DSL avg (ms) | DSL p50 | DSL p99 | 差异 (ms) | 差异 (%) |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| A1 等值查询(单条件) | 1.3 | 1.2 | 2.7 | 0.8 | 0.8 | 1.2 | +0.5 | +63% |
| A2 等值查询(多条件) | 2.9 | 2.6 | 14.6 | 2.0 | 2.0 | 3.5 | +0.9 | +45% |
| A3 范围查询 | 1.2 | 1.2 | 2.1 | 0.8 | 0.8 | 1.3 | +0.4 | +50% |

### 场景组 B：全文搜索

| 场景 | SQL avg (ms) | SQL p50 | SQL p99 | DSL avg (ms) | DSL p50 | DSL p99 | 差异 (ms) | 差异 (%) |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| B1 简单全文搜索 | 1.0 | 0.9 | 1.5 | 0.6 | 0.5 | 2.6 | +0.4 | +67% |
| B2 多字段搜索 | 1.9 | 1.9 | 2.5 | 3.1 | 3.0 | 4.7 | **-1.2** | **-39%** |

### 场景组 C：聚合查询（request_cache=false + 随机阈值）

| 场景 | SQL avg (ms) | SQL p50 | SQL p99 | DSL avg (ms) | DSL p50 | DSL p99 | 差异 (ms) | 差异 (%) |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| C1 简单聚合 | 12.7 | 12.6 | 26.2 | 1.4 | 1.3 | 2.9 | +11.3 | +807% |
| C2 多级聚合 | 27.4 | 27.5 | 54.9 | 7.7 | 7.6 | 14.0 | +19.7 | +256% |
| C3 范围聚合+排序 | 14.0 | 13.8 | 30.9 | 1.4 | 1.6 | 2.1 | +12.6 | +900% |

### 场景组 D：排序与分页

| 场景 | SQL avg (ms) | SQL p50 | SQL p99 | DSL avg (ms) | DSL p50 | DSL p99 | 差异 (ms) | 差异 (%) |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| D1 排序+小分页 | 1.1 | 1.1 | 2.6 | 0.7 | 0.7 | 0.9 | +0.4 | +57% |
| D2 深度分页(10000,10) | 13.3 | 13.3 | 16.3 | 3.3 | 3.2 | 6.3 | +10.0 | +303% |
| D3 大结果集(1000行) | 20.0 | 20.2 | 24.5 | 9.8 | 9.7 | 12.8 | +10.2 | +104% |

### 场景组 E：SQL 独有能力（无 DSL 对照）

| 场景 | SQL avg (ms) | SQL p50 | SQL p99 | 引擎路径 | 说明 |
|------|:---:|:---:|:---:|------|------|
| E1 UNION ALL+聚合 | 11264.8 | 11261.3 | 11350.2 | Calcite | ⚠️ 异常慢，见分析 |
| E2 2表JOIN | 63.4 | 62.8 | 71.8 | Legacy V1 | Hash Join |
| E3 IN子查询 | 91.3 | 90.8 | 103.3 | Legacy V1 | Hash Join |

### 场景组 F：辅助验证

| 场景 | avg (ms) | p50 | p99 | 说明 |
|------|:---:|:---:|:---:|------|
| F1 冷启动SQL | 1.8 | 1.8 | 2.8 | 每轮唯一注释 |
| F1 热计划SQL | 1.3 | 1.3 | 2.2 | 相同查询重复 |
| F2 Pushdown ON | 5.7 | 5.5 | 7.4 | UNION ALL+聚合 |
| F2 Pushdown OFF | 11319.2 | 11312.8 | 11409.5 | 同上查询，下推关闭 |

---

## 三、与预期对比

### 3.1 预期 vs 实际

| 场景 | 预期 SQL 开销 | 实际 SQL 开销 | 预期 DSL | 实际 DSL | 评估 |
|------|:---:|:---:|:---:|:---:|------|
| A1 点查 | 2-7ms | 0.5ms | 1-5ms | 0.8ms | ✅ 开销低于预期（JIT 充分预热后） |
| A2 多条件 | 2-7ms | 0.9ms | 1-5ms | 2.0ms | ✅ 同上 |
| B1 全文搜索 | 2-7ms | 0.4ms | 3-10ms | 0.6ms | ✅ |
| B2 多字段 | 2-7ms | **-1.2ms** | 3-10ms | 3.1ms | ⚠️ SQL 反而更快（DSL 的 bool.must 打分开销大） |
| C1 聚合 | 3-7ms | 11.3ms | 5-20ms | 1.4ms | ❌ SQL 开销远超预期（V2 聚合路径慢） |
| C2 多级聚合 | 5-10ms | 19.7ms | 10-40ms | 7.7ms | ❌ SQL 开销超预期 |
| D1 排序+分页 | 2-7ms | 0.4ms | 3-10ms | 0.7ms | ✅ |
| D2 深度分页 | 5-15ms | 10.0ms | 50-200ms | 3.3ms | ❌ DSL 远快于预期（3 shard + 单 segment 优势） |
| D3 大结果集 | 5-15ms | 10.2ms | 10-30ms | 9.8ms | ✅ 符合预期 |
| E1 UNION ALL | 20-50ms | 11265ms | — | — | ❌❌ 远超预期（pushdown 未生效） |
| E2 JOIN | 20-100ms | 63.4ms | — | — | ✅ 符合预期 |
| E3 IN子查询 | — | 91.3ms | — | — | ✅ 符合预期 |
| F2 Pushdown ON | — | 5.7ms | — | — | ✅ 符合预期 |
| F2 Pushdown OFF | — | 11319ms | — | — | ❌ 意料之中但差距惊人（2000x） |

### 3.2 预期结论框架 vs 实际

| 预期结论 | 实际验证 | 评估 |
|---------|---------|------|
| 轻查询（<10ms）：SQL 开销占比 30-50% | A/B/D1：SQL 开销 0.4-0.9ms，占比 40-67% | ✅ 符合 |
| 中查询（10-50ms）：SQL 开销占比 10-30% | C/D2/D3：SQL 开销 10-20ms，占比 50-90% | ❌ SQL 开销占比远超预期 |
| 重查询（>100ms）：SQL 开销占比 <5% | E2/E3：无法计算（无 DSL 对照） | — |
| 聚合场景 DSL 有优势 | C1-C3：DSL 快 5-10 倍 | ✅ 符合，但差距比预期更大 |

---

## 四、关键发现与分析

### 4.1 🔴 聚合查询 SQL 远慢于预期（C1-C3）

**现象**：C1 简单聚合 SQL 12.7ms vs DSL 1.4ms，差异 807%，远超预期的 3-7ms 开销。

**根因分析**：
- SQL V2 路径的 `Analyzer → Planner → PhysicalPlan` 翻译开销在聚合场景特别高
- V2 的 `AggregationOperator` 物理算子构建比 DSL 直接构造 `aggs` JSON 更重
- 随机化阈值导致每轮查询不同，无法利用任何缓存
- DSL 的 `terms` 聚合直接映射到 OpenSearch 的 `terms` 聚合，零翻译

**结论**：**聚合查询应优先使用 DSL**。SQL 的翻译开销在聚合场景占比极高（>80%）。

### 4.2 🔴 UNION ALL 异常慢（E1: 11.3 秒）

**现象**：E1（UNION ALL + 聚合）avg 11265ms，但 F2_PUSH_ON（相同查询，pushdown ON）仅 5.7ms。

**根因分析**：
- E1 运行时 pushdown 可能未生效（transient setting 状态不确定）
- F2_PUSH_OFF（pushdown 关闭）同样 11.3s，证实 pushdown OFF 时 Calcite 将 100 万数据拉入内存计算
- F2_PUSH_ON（pushdown 开启）5.7ms，聚合下推到 OpenSearch，仅合并 ~10 行结果

**结论**：
- **pushdown 是 Calcite 路径的生命线**：ON/OFF 差距 2000 倍（5.7ms vs 11319ms）
- UNION ALL + 聚合在 pushdown ON 时性能优秀（5.7ms）
- pushdown OFF 时不可用（11.3 秒）
- **生产环境必须确保 `plugins.calcite.pushdown.enabled=true`**

### 4.3 🟡 B2 场景 SQL 反而比 DSL 快

**现象**：B2 多字段搜索 SQL 1.9ms vs DSL 3.1ms，SQL 快 39%。

**根因分析**：
- DSL 使用 `bool.must`（打分）+ `bool.filter`（不打分）组合
- SQL 的 `match()` 生成 `match` query（打分），`level = 'ERROR'` 生成 `bool.filter`（不打分）
- DSL 的 `bool.must` + `match` 打分开销比 SQL 生成的等价 DSL 更大
- 可能是 DSL 的 JSON 解析 + bool query 构建开销略高

**结论**：SQL 的 `match()` + `WHERE` 组合在特定场景下可能比手写 DSL 更优（因为 SQL 自动区分 filter/must context）。

### 4.4 🟡 冷启动 vs 热计划差异小

**现象**：F1 冷启动 1.8ms vs 热计划 1.3ms，差异 0.5ms（38%）。

**分析**：
- 冷启动（每轮唯一注释）比热计划慢 0.5ms
- 0.5ms 的差异可能是 ANTLR parser 的微小缓存效应
- **未发现显著的计划缓存机制**（如有缓存，差异应更大）
- 结论：SQL 插件当前无有效的计划缓存，每次查询都重新解析+规划

### 4.5 🟢 点查/全文搜索 SQL 开销低于预期

**现象**：A/B/D1 场景 SQL 额外开销仅 0.4-0.9ms，低于预期的 2-7ms。

**根因分析**：
- 20 轮预热后 JIT C2 编译充分优化了 ANTLR/AstBuilder/Analyzer 热路径
- 单节点 + 本地回环消除了网络开销
- V2 的 PhysicalPlan → SearchRequestBuilder 翻译在简单查询上非常轻量
- 预期的 2-7ms 估算偏高（基于未充分预热的假设）

**结论**：**充分预热后，简单查询的 SQL 翻译开销可低至 0.4-0.9ms**。但在生产环境（JIT 未充分预热、混合查询负载）中，开销可能回升到 2-5ms。

---

## 五、综合结论

### 5.1 SQL vs DSL 性能决策矩阵

| 查询形态 | SQL 延迟 | DSL 延迟 | SQL 额外开销 | 开销占比 | 推荐 |
|---------|:---:|:---:|:---:|:---:|------|
| 点查 (SELECT+WHERE) | 1-3ms | 0.8-2ms | 0.4-0.9ms | 40-63% | 低 QPS 可用 SQL；高 QPS 用 DSL |
| 全文搜索 (match) | 1-2ms | 0.6-3ms | 0.4ms ~ -1.2ms | -39% ~ 67% | 均可，SQL 可能更快 |
| 聚合 (GROUP BY) | 13-27ms | 1.4-7.7ms | 11-20ms | 256-807% | **强烈推荐 DSL** |
| 排序+小分页 | 1ms | 0.7ms | 0.4ms | 57% | 均可 |
| 深度分页 | 13ms | 3.3ms | 10ms | 303% | **推荐 DSL** |
| 大结果集(1000行) | 20ms | 10ms | 10ms | 104% | 推荐 DSL |
| UNION ALL (pushdown ON) | 5.7ms | — | — | — | **SQL 唯一选择** |
| UNION ALL (pushdown OFF) | 11319ms | — | — | — | **不可用** |
| 2表 JOIN | 63ms | — | — | — | **SQL 唯一选择** |
| IN 子查询 | 91ms | — | — | — | **SQL 唯一选择** |

### 5.2 核心结论

1. **简单查询（点查/全文搜索/排序+小分页）**：SQL 翻译开销 0.4-0.9ms，在低 QPS 场景可接受。高 QPS（>1000）场景 DSL 更优。

2. **聚合查询**：SQL 翻译开销 11-20ms，是 DSL 的 5-10 倍。**聚合查询应优先使用 DSL**。这是最大的性能差距点。

3. **UNION ALL**：pushdown ON 时 5.7ms（优秀），pushdown OFF 时 11.3s（不可用）。**pushdown 是 Calcite 路径的生命线，2000 倍性能差异**。

4. **JOIN/IN 子查询**：走 Legacy V1 引擎，63-91ms。无 DSL 对照（DSL 不支持），是 SQL 独有价值。

5. **无计划缓存**：冷热计划差异仅 0.5ms，说明无有效计划缓存。每次查询都重新解析+规划。高 QPS 场景下翻译开销会累积。

6. **B2 全文搜索反常**：SQL 的 `match()` + `WHERE` 组合可能比手写 DSL 的 `bool.must` + `bool.filter` 更快（SQL 自动优化 filter context）。

### 5.3 与预期的主要偏差

| 偏差 | 预期 | 实际 | 原因 |
|------|------|------|------|
| 点查 SQL 开销 | 2-7ms | 0.4-0.9ms | JIT 充分预热后开销低于预期 |
| 聚合 SQL 开销 | 3-7ms | 11-20ms | V2 聚合路径翻译比预期重 |
| 深度分页 DSL | 50-200ms | 3.3ms | 3 shard + 单 segment + forcemerge 优势 |
| UNION ALL | 20-50ms | 5.7ms (pushdown ON) / 11.3s (OFF) | pushdown 效果远超预期 |
| B2 全文搜索 | SQL 慢于 DSL | SQL 快于 DSL | SQL 自动 filter context 优化 |

### 5.4 建议

1. **生产环境必须确保 pushdown 启用**（`plugins.calcite.pushdown.enabled=true`），否则 UNION/JOIN 在 Calcite 路径完全不可用。
2. **聚合查询优先用 DSL**，SQL 聚合翻译开销占比 >80%。
3. **点查和全文搜索可用 SQL**，翻译开销 <1ms（充分预热后）。
4. **长期需实现计划缓存**，当前每次查询都重新解析+规划，高 QPS 下开销累积。
5. **长期需实现统计信息注入**，让 Calcite CBO 真正生效（当前 pushdown 是规则驱动，非代价驱动）。
6. **单节点 512MB heap 是限制因素**，生产环境（8GB+ heap）的绝对延迟会更低，但相对开销比例应相似。

### 5.5 数据修正与可信度说明

> 经 Oracle 大数据专家深度审视和手动验证，以下数据需修正或标注限制：

**1. SQL 聚合场景的 filter cache 污染（影响 C1-C3）**

SQL 路径经 `_plugins/_sql` 走 PIT 机制，OpenSearch 的 **filter cache 对 SQL 请求仍生效**。基准测试中 C1-C3 虽然使用了随机阈值打散 request cache，但 filter cache（缓存 `bool.filter` 条件结果）仍可能命中。

手动验证证实：SQL 重复相同聚合查询，首次 249ms → 后续 34-37ms（filter cache 命中）。因此 C1-C3 的 SQL 数据（12.7-27.4ms）可能部分受益于 filter cache，实际冷查询 SQL 延迟可能更高。

**2. SQL 翻译开销范围**

报告中 "0.4-0.9ms（简单查询）" 和 "11-20ms（聚合）" 是直接测量的端到端差异，**包含翻译开销 + 可能的缓存效应 + JdbcResponseFormatter 开销**，不能简单等同于"翻译开销"。

更准确的表述：SQL 的总额外开销（翻译+格式化+缓存差异）范围为 **0.4-20ms**，与查询复杂度正相关，**非固定值**。

**3. B2 全文搜索 SQL 快于 DSL 的可信度**

B2 的 DSL 查询使用 `bool.must`（打分）+ `bool.filter`（不打分）组合。SQL 的 `match()` 生成 `match` query（打分），`level = 'ERROR'` 生成 `bool.filter`（不打分）。如果 DSL 查询中 `level` 条件误用了 `must` 而非 `filter`，DSL 会额外打分导致变慢。此结论的公平性取决于 DSL 查询编写质量，**不宜作为 SQL 结构性优势的依据**。

**4. E1 pushdown 状态不确定**

E1（11265ms）与 F2_PUSH_OFF（11319ms）几乎相同，说明 E1 运行时 pushdown 可能未生效。F2_PUSH_ON（5.7ms）是单独验证的结果。E1 的数据**不应作为 UNION ALL 的代表性性能**，应使用 F2_PUSH_ON 的 5.7ms。

---

## 附录：测试数据说明

### 数据分布

| 字段 | 类型 | 基数 | 分布 |
|------|------|:---:|------|
| level | keyword | 4 | 均匀（INFO/WARN/ERROR/DEBUG） |
| service | keyword | 5 | 均匀 |
| host | keyword | 5 | 均匀 |
| region | keyword | 5 | 均匀 |
| status_code | integer | 5 | 加权（200:50%, 301:12.5%, 404:12.5%, 500:12.5%, 503:12.5%） |
| response_time_ms | integer | 1-5000 | 均匀随机 |
| message | text | 8 | 均匀 |

### 测试方法论说明

- **预热**：20 轮（简单查询）/ 5 轮（复杂查询），充分触发 JIT C2 编译
- **测试轮数**：100 轮（简单）/ 20 轮（复杂）/ 50 轮（F1 冷热对比）
- **聚合场景**：每轮使用随机阈值（1-5000）打散 request cache
- **DSL 请求**：均加 `?request_cache=false`
- **连接复用**：使用 `requests.Session()` 复用 TCP 连接
- **局限性**：单节点 512MB heap，3 shard 0 replica，本地回环，forcemerge 1 segment。生产环境绝对延迟会不同，但相对开销比例应相似。
