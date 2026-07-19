# 10M 数据量重查询性能对比验证报告

> 基于 OpenSearch 3.7.0 + SQL Plugin 3.7.0（含 UNION/UNION ALL 扩展）
> 数据量：990 万文档，6 shard，单节点（模拟生产数据量，单节点限制已标注）
> 测试日期：2026-07-18
> 未 forcemerge（模拟生产 segment 分布）

---

## 一、测试环境

| 项目 | 配置 |
|------|------|
| OpenSearch | 3.7.0-SNAPSHOT 单节点 |
| SQL Plugin | 3.7.0.0-SNAPSHOT（含 UNION/UNION ALL 扩展） |
| 索引 | `perf_test_10m`，6 shard 0 replica，9.9M docs，2.8GB |
| 字段数 | 13（含高基数 user_id=100K, session_id=500K, client_ip=100K） |
| forcemerge | **否**（模拟生产多 segment 分布） |
| `plugins.calcite.enabled` | true |
| `plugins.calcite.pushdown.enabled` | true |
| JVM Heap | 512MB |
| slowlog | 开启（`threshold.query.info: 0ms`） |

> ⚠️ **单节点限制**：6 shard 全在单节点，无跨节点 scatter-gather。多节点环境下 DSL 聚合有并行优势，绝对延迟会不同。但数据量 10M 是核心变量，SQL 翻译开销与节点数无关。

---

## 二、完整测试结果

### 全部场景汇总

| 场景 | SQL avg (ms) | SQL p50 | DSL avg (ms) | DSL p50 | 差异 (ms) | 差异 (%) | SQL 引擎 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|------|
| **G1 时间范围+聚合** | **121.7**³ | 115.1 | 50.7³ | 49.8 | +71.0 | +140% | V2 |
| G3 高基数聚合+过滤 | 859.2 | 858.8 | 26.0 | 24.9 | +833.2 | +3205% | V2 |
| **H1 全表500K桶聚合** | **32991**⁴ | 33129 | 143.7⁴ | 118.5 | +32847 | **+22865%** | V2 |
| **H2 全表多级聚合** | **211.4**³ | 209.3 | 104.9³ | 103.3 | +106.5 | +102% | V2 |
| H3 大范围+排序+10K | 173.4 | 171.3 | 71.8 | 71.2 | +101.6 | +142% | V2 |
| E1 三路UNION+聚合 | 12.5 | 12.0 | — | — | — | — | Calcite |
| E2 大表JOIN 10K | 744.1 | 715.4 | — | — | — | — | Legacy V1 |
| E3 IN子查询 10K | 789.5 | 729.3 | — | — | — | — | Legacy V1 |

> ³ G1/H2 数据修正（重测）：原数据（G1 SQL=2.7ms, H2 SQL=4.6ms）因 SQL 路径不尊守 `request_cache=false`，filter cache 命中导致假数据。重测使用随机阈值打散缓存 + DSL 去掉 percentile 保持查询等价。重测结果：G1 SQL=121.7ms, H2 SQL=211.4ms，SQL 实际比 DSL 慢 1.4-2.0x。
> ⁴ H1 重测：512MB heap 下 composite 翻页 500 次导致严重 GC，avg 32991ms（部分查询触发 circuit breaker）。H1 DSL 部分查询也触发 circuit breaker，有效样本仅 3 次。H1 数据可信度低，需更大 heap 重测。

### 按性能差异分类

| 分类 | 场景 | SQL vs DSL | 原因 |
|------|------|:---:|------|
| **SQL 慢 1.4-2.0x** | G1, H2 | SQL 慢 102-140% | V2 composite 翻译开销 + DSL 查询等价后更快 |
| **SQL 明显慢** | H3 | SQL 慢 2.4x | JdbcResponseFormatter 序列化 10K 行开销 ~100ms |
| **SQL 极慢** | G3, H1 | SQL 慢 33-100x | V2 composite 聚合翻页拉取 vs DSL terms Top-N |
| **SQL 独有** | E1, E2, E3 | 无对照 | UNION/JOIN/IN 子查询 |

---

## 三、关键发现

### 3.1 🔴 H1/G3：V2 高基数聚合策略差异

**现象**：
- H1（500K session_id 桶）：SQL 2354ms（5 次手动验证 1941-3015ms）vs DSL 329ms，**SQL 慢 7 倍**
- G3（100K user_id 桶）：SQL 859ms vs DSL 26ms，**SQL 慢 33 倍**

> ⚠️ **数据修正**：原基准测试 H1 SQL 54521ms 是因 512MB heap 不足导致 GC 频繁。手动单次验证稳定在 2-3 秒，差距从 180x 修正为 7x。

**根因**（通过 `_explain` 确认）：
- SQL V2 使用 `composite` 聚合（`size=1000`），对 500K 个 session_id 需要翻页 ~500 次
- 每次翻页是一次独立的 OpenSearch 请求（`composite` 聚合的 after_key 机制）
- DSL 使用 `terms` 聚合（`size=5000`），shard 级直接取 Top 5000，仅 1 次请求
- **这不是翻译开销，也不是内存聚合**——V2 确实下推了聚合到 OpenSearch（`composite` 聚合），但翻页机制导致请求数 = 桶数/1000

**explain 输出确认**：
```
V2 路径: ProjectOperator → TakeOrderedOperator → OpenSearchIndexScan
OpenSearchIndexScan 请求体:
  "aggregations":{"composite_buckets":{"composite":{"size":1000,"sources":[
    {"session_id":{"terms":{"field":"session_id"}}}]},
    "aggregations":{"COUNT(*)":{"value_count":{"field":"_index"}},
                    "AVG(response_time_ms)":{"avg":{"field":"response_time_ms"}}}}}
```
V2 确实下推了 composite 聚合到 OpenSearch，但 `size=1000` 意味着 500K 桶需要 ~500 次翻页请求。

**结论**：**高基数聚合（GROUP BY 高基数字段）的性能差异来自聚合策略选择**——V2 用 `composite`（分页拉取，适合全量遍历），DSL 用 `terms`（Top-N，适合排序取前 N）。当 `LIMIT 5000` 时 DSL 的 `terms(size=5000)` 更高效。这是 V2 的优化空间，而非 SQL 接口的固有缺陷。

### 3.2 G1/H2：重测确认 SQL 比 DSL 慢 1.4-2.0x（原结论已推翻）

**原结论（已推翻）**：SQL 快 82-99%，composite 聚合是结构性优势。

**重测结论**：SQL 比 DSL 慢 1.4-2.0x。原数据受 filter cache 污染 + DSL 查询含 percentile 导致不公平。

**重测方法**：
- 使用随机阈值（1-9000）打散 filter cache
- DSL 去掉 percentile 聚合，保持与 SQL 查询等价（均计算 COUNT + AVG）

**重测数据**：

| 场景 | SQL avg (ms) | DSL avg (ms) | 差异 | 原数据（错误） |
|------|:---:|:---:|:---:|:---:|
| G1（10M, 随机阈值） | 121.7 | 50.7 | SQL 慢 140% | 原 SQL=2.7ms (cache hit) |
| H2（10M, 随机阈值, 无 percentile） | 211.4 | 104.9 | SQL 慢 102% | 原 SQL=4.6ms (cache hit), DSL=379ms (with percentile) |

**根因分析**：
- SQL 额外开销（翻译+composite 翻页）约 70-107ms
- DSL 直接构造 `terms` 聚合请求，无翻译开销
- **不存在 "SQL composite 比 DSL nested aggs 高效" 的结构性优势**——原结论是 filter cache 污染 + 查询不等价的假象

### 3.3 🟡 H3：大结果集额外开销（估算，未实测分解）

**现象**：SQL 173ms vs DSL 72ms，差异 101ms。

**估算分解**（⚠️ 以下为估算值，未通过实测分解验证）：
- DSL 执行时间（含 fetch 10K 行）：~72ms
- SQL 翻译开销（解析+分析+规划）：估算 ~5-15ms
- SQL JdbcResponseFormatter 序列化 10K 行：估算 ~86-96ms（由总额减去翻译估算得出）
- **翻译开销占比**：估算 5-15ms / 72ms ≈ 7-21%
- **总额外开销占比**：101ms / 72ms ≈ 142%

> ⚠️ **可信度说明**：翻译开销和格式化开销均未通过实测分解验证。翻译开销 5-15ms 是估算值（从总额减去格式化估算），格式化 ~90ms 也是估算值（从总额减去翻译估算）。两者相互推导，存在循环论证。需要通过代码级 instrumentation 或 format=raw 对比实测。

**结论**：总额外开销 101ms 是实测的（SQL 总延迟 - DSL 总延迟）。其中翻译开销和格式化开销的**比例未经验证**。大结果集场景的额外开销显著（142%），但无法确定瓶颈是翻译还是格式化。

### 3.4 🟢 E1：UNION ALL + 聚合在 10M 数据上表现优秀

**现象**：三路 UNION ALL + 聚合仅 12.5ms。

**分析**：
- Calcite pushdown 将 3 个子查询的聚合下推到 OpenSearch（3 次 shard 级聚合）
- 仅合并 15 行结果（5 service × 3 路）
- 10M 数据量对 pushdown 路径影响极小（聚合在 shard 级执行）

### 3.5 🟢 E2/E3：JOIN/IN 子查询在 10M 数据上可接受

**现象**：E2 JOIN 744ms，E3 IN 子查询 790ms。

**分析**：
- Legacy V1 Hash Join 在 10M 数据上性能合理
- 无 DSL 对照（DSL 不支持 JOIN/IN 子查询）
- 与"两次 DSL 查询 + 应用层关联"相比，SQL 单次查询更便利

---

## 四、与预期对比

### 4.1 预期 vs 实际

| 场景 | 预期 DSL (ms) | 实际 DSL (ms) | 预期 SQL 开销占比 | 实际 SQL 开销占比 | 评估 |
|------|:---:|:---:|:---:|:---:|------|
| G1 | 300-600 | 30.7 | 1-5% | **-91%（SQL 更快）** | ✅ SQL 优于预期 |
| G3 | 300-600 | 26.0 | 1-5% | +3205% | ❌❌ V2 composite 翻页 |
| H1 | 800-2000 | 329 | 0.5-3% | +615% | ❌❌ V2 composite 翻页 |
| H2 | 600-1500 | 379.0 | 0.3-2% | **-99%（SQL 更快）** | ✅✅ SQL 大幅优于预期 |
| H3 | 400-800 | 71.8 | 0.6-3% | +142% | ❌ DSL 快于预期 |
| E1 | — | — | — | — | ✅ 12.5ms |
| E2 | — | — | — | — | ✅ 744ms |
| E3 | — | — | — | — | ✅ 790ms |

### 4.2 预期偏差分析

#### 偏差 1：DSL 在 10M 单节点上仍然较快（26-380ms）

**预期**：10M 数据 DSL 150-2000ms
**实际**：DSL 26-380ms

**原因**：
- 单节点 6 shard 无跨节点网络开销
- OpenSearch 的聚合引擎高度优化（shard 级 terms 聚合很快）
- 512MB heap + 2.8GB 数据，filter cache 命中率高
- 多节点环境下 DSL 会有 scatter-gather 开销，绝对延迟会更高

#### 偏差 2：H1/G3 V2 composite 聚合翻页是性能瓶颈

**预期**：SQL 开销 10-30ms
**实际**：H1 SQL 2354ms（修正后），G3 SQL 859ms

**原因**：V2 使用 `composite` 聚合（`size=1000`）下推到 OpenSearch，但对高基数字段（100K-500K 桶）需要翻页 ~100-500 次。每次翻页是一次独立 OpenSearch 请求。DSL 使用 `terms` 聚合（`size=5000`），一次取 Top-N，仅 1 次请求。

> **修正**：原报告称 V2 做"内存聚合"——这是不准确的。通过 `_explain` 确认 V2 确实下推了 `composite` 聚合到 OpenSearch，问题在于翻页请求数过多。

#### 偏差 3：H2/G1 SQL composite 聚合是未预见的优势

**预期**：SQL 与 DSL 差异不大
**实际**：SQL 快 82-99%

**原因**：V2 对低基数多级 GROUP BY 使用 `composite` 聚合下推，比 DSL 的 nested `terms` 高效 1-2 个数量级。低基数场景 composite 一次翻页即可获取全部桶，无翻页开销。

---

## 五、核心结论

### 5.1 能否得出"SQL 与 DSL 性能差异不大"的结论？

**部分能，部分不能。** 需要按场景分类：

| 场景类型 | 能否说"差异不大"？ | 数据支撑 |
|---------|:---:|------|
| 低基数多级聚合 | ⚠️ SQL 慢 1.4-2.0x | G1/H2 重测：SQL 慢 102-140%（原 "SQL 快 82-99%" 已推翻） |
| 高基数聚合 | ❌ DSL 更快 | H1/G3：DSL 快 33-228x（composite 翻页 + 512MB heap GC） |
| 大结果集+排序 | ⚠️ 总额外开销 142% | H3：总额外 101ms，翻译/格式化比例未验证 |
| UNION+聚合 | ✅ 无对照，SQL 12.5ms | E1：pushdown 高效 |
| JOIN/IN子查询 | ✅ 无对照，SQL 744-790ms | E2/E3：合理 |

### 5.2 SQL 额外开销范围

> ⚠️ 以下数据中，"翻译开销"为估算值（未通过代码级 instrumentation 实测分解）。

| 开销类型 | 范围 | 说明 | 可信度 |
|---------|:---:|------|:---:|
| **翻译开销**（解析+分析+规划+DSL生成） | **0.4-20ms** | 与查询复杂度正相关（简单查询 0.4-0.9ms，聚合 11-20ms），**非固定值** | ⚠️ 估算 |
| JdbcResponseFormatter 序列化 | 估算 0-90ms | 与结果集大小相关，估算 ~9ms/1K行（未实测） | ⚠️ 估算 |
| V2 composite 翻页（高基数场景） | 0-2354ms | 与桶数强相关（桶数/1000 次翻页请求），非翻译开销 | ✅ 实测 |
| **总额外开销**（SQL总延迟 - DSL总延迟） | **0.4-2354ms** | 实测值，与查询类型强相关 | ✅ 实测 |

> **重要修正**：原报告称 "翻译开销稳定在 5-15ms" 是不准确的。跨三份报告的直接测量显示，SQL 额外开销范围为 0.4-2354ms，与查询类型强相关，**非固定值**。"5-15ms" 是从 H3 总差异减去估算的格式化开销得出的循环推导值。

### 5.3 最终结论（修正后）

```
结论:

1. SQL 的"翻译开销"（解析+分析+规划+DSL生成）范围为 0.4-20ms，
   与查询复杂度正相关（简单查询 <1ms，聚合查询 11-20ms）。
   在 DSL 执行时间 >200ms 的重查询场景下，翻译开销占比通常 <10%。
   但此开销并非固定值，不能简单概括为"5-15ms 稳定"。

2. SQL 的"总额外开销"（SQL总延迟 - DSL总延迟）包含：
   - 翻译开销：0.4-20ms（与查询复杂度正相关）
   - JdbcResponseFormatter 序列化：估算 0-90ms（与结果集大小相关，未实测分解）
   - V2 聚合策略差异：高基数场景 composite 翻页导致 0-2354ms（非翻译开销）
   - filter cache 效应：SQL 路径不尊守 request_cache=false，
     重复查询时 SQL 可能受益于 filter cache（冷查询则无此优势）

3. "SQL 与 DSL 性能差异不大"的结论在以下条件下成立：
   ✅ 中等结果集（≤1000行）：额外开销 <20ms
   ✅ UNION/JOIN/IN子查询：无 DSL 对照，SQL 是唯一选择
   ⚠️ 低基数聚合：需重新测试（原数据受 filter cache 污染）
   ⚠️ 大结果集（10K行）：总额外开销 ~100ms（翻译+格式化，比例未验证）
   ❌ 高基数聚合（GROUP BY 10K+ 桶）：V2 composite 翻页导致 7-33x 性能差距

4. 高基数聚合场景的瓶颈是"V2 聚合策略选择"——
   V2 使用 composite 聚合（size=1000 翻页拉取），DSL 使用 terms 聚合（size=N 一次取 Top-N）。
   可通过优化 V2 引擎解决（对 ORDER BY + LIMIT 场景改用 terms 聚合）。

5. 多节点环境预期：
   - DSL 绝对延迟会上升（scatter-gather + 跨节点 merge）
   - SQL 翻译开销不变
   - 翻译开销占比会进一步下降
   - 但 V2 composite 翻页问题不会因多节点而改善
```

### 5.4 H1 DSL 方法论修正说明

> 原报告 H1 DSL 使用首次执行值（329ms）而非平均值（~60ms），与其他场景方法论不一致。

**原报告**：H1 DSL 取首次值 329ms（后续 filter cache 命中降至 29-31ms），得出 SQL/DSL = 7x。

**问题**：所有其他场景使用平均值（10-15 轮），H1 单独使用首次值，属于方法论不一致。如用平均值：
- DSL 平均 ~60ms（10 轮：329 + 9×30）/ 10
- SQL/DSL = 2354/60 = **39x**（而非 7x）

**修正**：H1 的 7x 差距是乐观估计。实际上 DSL 在 filter cache 命中后极快（30ms），而 SQL composite 翻页不受 filter cache 影响（每页是不同请求），因此实际差距可能为 39x。但生产环境首次查询（冷 cache）时 DSL 也是 329ms，此时差距为 7x。

**结论**：H1 差距范围为 **7x（冷 cache）到 39x（热 cache）**，取决于 cache 状态。

### 5.5 与 1M 轻查询/重查询的对比

| 维度 | 1M 轻查询 | 1M 重查询 | 10M 重查询 |
|------|---------|---------|---------|
| DSL 延迟范围 | 0.6-3ms | 6-60ms | 26-380ms |
| SQL 额外开销（实测总额） | 0.4-20ms | 1-78ms | 101-2354ms |
| SQL 额外开销占比 | 40-800% | 18-262% | 1-615% |
| SQL 最优场景 | B2 全文搜索(-39%) | C2-H(待重测) | E1 UNION(12.5ms) |
| SQL 最差场景 | C1 聚合 | A2 多条件 | H1 高基数聚合 |
| 核心瓶颈 | DSL 太快 | DSL 仍较快 | V2 composite 翻页 |
| 数据可信度 | ⚠️ filter cache 未控制 | ⚠️ C2-H 查询不等价 | ⚠️ G1/H2 缓存污染+查询不等价 |

**趋势**：随数据量增大，DSL 执行时间增长，SQL 翻译开销占比总体下降。但：
- V2 composite 聚合翻页问题在高基数场景下恶化
- filter cache 效应使 SQL 重复查询偏快（冷查询则无此优势）
- 512MB heap 的 GC 压力可能影响部分测量结果

---

## 六、建议

### 6.1 短期（用户侧）

| 场景 | 推荐 | 理由 |
|------|------|------|
| 低基数多级聚合 | **待重测** | 原 "SQL 快 10-100x" 受缓存污染+查询不等价，手动验证后 SQL 实际慢 ~2x |
| 高基数聚合（10K+桶） | **DSL** | V2 composite 翻页请求数 = 桶数/1000，DSL terms 一次取 Top-N |
| 大结果集（10K行） | **DSL** | 总额外开销 ~100ms（翻译+格式化，比例未验证） |
| 中等结果集（≤1K行） | **均可** | 额外开销 <20ms |
| UNION/JOIN/子查询 | **SQL** | DSL 不支持 |
| 点查+小结果集 | **DSL** | 额外开销占比高但绝对值小 |

### 6.2 长期（引擎优化侧）

| 优化项 | 优先级 | 预期效果 |
|--------|:---:|------|
| V2 对 ORDER BY+LIMIT 场景改用 terms 聚合而非 composite | P0 | 消除 H1/G3 的 7-39x 性能差距 |
| JdbcResponseFormatter 流式序列化 | P1 | 减少 10K+ 行结果集的格式化开销（估算 ~90ms） |
| 计划缓存（PreparedStatement） | P2 | 消除 0.4-20ms 翻译开销（高频查询） |
| 统计信息注入 Calcite | P2 | 让 Calcite CBO 选择更优计划 |

---

## 附录：测试数据说明

### 数据分布

| 字段 | 类型 | 基数 | 说明 |
|------|------|:---:|------|
| level | keyword | 4 | INFO/WARN/ERROR/DEBUG |
| service | keyword | 5 | 5 个服务 |
| host | keyword | 20 | 20 台主机 |
| region | keyword | 5 | 5 个区域 |
| status_code | integer | 9 | 200(35%)/301/302/400/401/403/404(12%)/500(12%)/502/503 |
| response_time_ms | integer | 1-10000 | 均匀随机 |
| user_id | keyword | 100,000 | 高基数 |
| session_id | keyword | 500,000 | 极高基数 |
| message | text | 15 | 15 种消息 |
| request_path | keyword | 10 | 10 个路径 |
| client_ip | ip | ~65K | 随机 IP |

### 测试方法论

- **预热**：3 轮（重查询每轮 0.1-55 秒，3 轮足够 JIT 预热）
- **测试轮数**：10-15 轮（E 组 5-10 轮，因单次执行时间长）
- **request cache**：禁用（`?request_cache=false`）
- **连接复用**：`requests.Session()`
- **超时**：120 秒/请求
- **未 forcemerge**：模拟生产 segment 分布
- **单节点限制**：6 shard 全在单节点，无跨节点网络开销。多节点环境 DSL 绝对延迟会更高，SQL 翻译开销占比会更低。
