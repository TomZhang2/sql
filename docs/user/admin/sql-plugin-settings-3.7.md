# OpenSearch SQL Plugin 3.7.0 配置参数详细清单

> 基于源码 `Settings.java`、`OpenSearchSettings.java` 及官方文档 `docs/user/admin/settings.rst` 整理。
> 所有参数均为 **Node Scope**（节点级），绝大多数支持 **动态更新**（无需重启）。

---

## 目录

- [1. SQL 基础设置](#1-sql-基础设置)
- [2. PPL 基础设置](#2-ppl-基础设置)
- [3. Calcite 引擎设置](#3-calcite-引擎设置)
- [4. 查询资源限制](#4-查询资源限制)
- [5. PPL Pattern 命令设置](#5-ppl-pattern-命令设置)
- [6. PPL 子搜索与 Rex 设置](#6-ppl-子搜索与-rex-设置)
- [7. 数据源设置](#7-数据源设置)
- [8. Spark 异步查询引擎设置](#8-spark-异步查询引擎设置)
- [9. 异步查询调度设置](#9-异步查询调度设置)
- [10. 监控指标设置](#10-监控指标设置)
- [11. 非动态设置（需重启）](#11-非动态设置需重启)
- [12. 线程池设置](#12-线程池设置)
- [13. 配置方式](#13-配置方式)
- [14. 使用建议](#14-使用建议)

---

## 1. SQL 基础设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.sql.enabled` | boolean | `true` | ✅ | 1.0 | 启用/禁用 SQL 接口。设为 `false` 时所有 `_plugins/_sql` 请求被拒绝 |
| `plugins.sql.slowlog` | integer | `2` | ✅ | 1.0 | 慢查询日志阈值（秒）。执行时间超过此值的查询记录到 `opensearch.log` |
| `plugins.sql.cursor.keep_alive` | time | `1m` | ✅ | 1.0 | 游标上下文存活时间。游标占用较多资源，建议尽量设小 |

**使用示例：**
```bash
# 禁用 SQL 接口
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.sql.enabled": "false" }
}'

# 设置慢查询阈值为 10 秒
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.sql.slowlog": "10" }
}'

# 设置游标存活时间为 5 分钟
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.sql.cursor.keep_alive": "5m" }
}'
```

---

## 2. PPL 基础设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.ppl.enabled` | boolean | `true` | ✅ | 1.0 | 启用/禁用 PPL 接口。设为 `false` 时所有 `_plugins/_ppl` 请求被拒绝 |
| `plugins.ppl.query.timeout` | time | `300s` (5m) | ✅ | 1.0 | PPL 查询超时时间。超时后查询被终止 |
| `plugins.ppl.syntax.legacy.preferred` | boolean | `true` | ✅ | 3.0 | 优先使用旧版 PPL 语法解析。用于兼容性控制，也传递给 Unified Query Context |

**使用示例：**
```bash
# 设置 PPL 查询超时为 60 秒
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.ppl.query.timeout": "60s" }
}'
```

---

## 3. Calcite 引擎设置

> Calcite 是 3.0 引入的新查询引擎（V3 引擎），3.3.0 起默认启用。
> PPL 查询走 Calcite 路径；SQL 查询中的 UNION 也会路由到 Calcite。

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.calcite.enabled` | boolean | `true` | ✅ | 3.0 | 启用 Calcite 引擎。3.0-3.2 默认 `false`，3.3+ 默认 `true` |
| `plugins.calcite.fallback.allowed` | boolean | `false` | ✅ | 3.1 | Calcite 执行失败时是否允许回退到 V2 引擎。`CalciteUnsupportedException` 始终允许回退，不受此设置限制 |
| `plugins.calcite.pushdown.enabled` | boolean | `true` | ✅ | 3.0 | 启用算子下推优化。将过滤、投影等操作下推到 OpenSearch 存储层执行 |
| `plugins.calcite.pushdown.rowcount.estimation.factor` | double | `0.9` | ✅ | 3.1 | 行数估算因子。下推优化时用于估算查询计划的行数，影响优化器决策 |
| `plugins.calcite.all_join_types.allowed` | boolean | `false` | ✅ | 3.3 | 启用所有 JOIN 类型。默认仅支持 `INNER`/`LEFT`/`SEMI`/`ANTI`；设为 `true` 额外启用 `RIGHT`/`FULL`/`CROSS` |

**使用示例：**
```bash
# 启用 Calcite 引擎（通常已是默认值）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.calcite.enabled": "true" }
}'

# 允许 Calcite 回退到 V2（调试用）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.calcite.fallback.allowed": "true" }
}'

# 启用所有 JOIN 类型（包括 RIGHT/FULL/CROSS）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.calcite.all_join_types.allowed": "true" }
}'
```

---

## 4. 查询资源限制

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.query.memory_limit` | memory size | `85%` | ✅ | 1.0 | 查询堆内存使用上限。超出时终止当前查询。可设为百分比或绝对值（如 `2g`） |
| `plugins.query.size_limit` | integer | `index.max_result_window` (默认 10000) | ✅ | 1.0 | 单次查询返回的最大行数。不能超过索引级 `index.max_result_window` |
| `plugins.query.buckets` | integer | 等于 `plugins.query.size_limit` | ✅ | 3.4 | 单次响应中返回的最大聚合桶数。不能超过 `search.max_buckets`。仅在 Calcite 启用时生效 |
| `plugins.query.field_type_tolerance` | boolean | `true` | ✅ | 2.19 | 是否保留数组字段。`true` 返回完整数组，`false` 仅返回数组第一个元素 |
| `search.max_buckets` | integer | 65536 | ✅ | — | OpenSearch 核心设置，SQL 插件读取此值作为桶数上限 |

**使用示例：**
```bash
# 限制查询返回最多 500 行
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.size_limit": 500 }
}'

# 设置内存限制为 80%
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.query.memory_limit": "80%" }
}'

# 设置聚合桶数上限为 1000
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.buckets": 1000 }
}'
```

---

## 5. PPL Pattern 命令设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.ppl.pattern.method` | string | `SIMPLE_PATTERN` | ✅ | — | Pattern 命令的默认分析方法 |
| `plugins.ppl.pattern.mode` | string | `LABEL` | ✅ | — | Pattern 命令的输出模式 |
| `plugins.ppl.pattern.max.sample.count` | integer | `10` | ✅ | — | Pattern 采样的最大样本数。最小值 0 |
| `plugins.ppl.pattern.buffer.limit` | integer | `100000` | ✅ | — | Pattern 缓冲区大小限制。最小值 50000 |
| `plugins.ppl.pattern.show.numbered.token` | boolean | `false` | ✅ | — | 是否在 Pattern 输出中显示编号 token |

---

## 6. PPL 子搜索与 Rex 设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.ppl.rex.max_match.limit` | integer | `10` | ✅ | — | Rex 命令每个字段的最大匹配数。最小值 1。也传递给 Unified Query Context |
| `plugins.ppl.values.max.limit` | integer | `0` | ✅ | — | Values 命令的最大行数。`0` 表示无限制，`-1` 表示禁用 |
| `plugins.ppl.subsearch.maxout` | integer | `10000` | ✅ | — | 子搜索（subsearch）最大输出行数 |
| `plugins.ppl.join.subsearch_maxout` | integer | `50000` | ✅ | — | JOIN 子搜索最大输出行数 |

---

## 7. 数据源设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.query.datasources.enabled` | boolean | `true` | ✅ | 2.16 | 启用/禁用数据源功能 |
| `plugins.query.datasources.limit` | integer | `20` | ✅ | 2.12 | 集群允许的最大数据源数量 |
| `plugins.query.datasources.uri.hosts.denylist` | list | `[]` (空) | ✅ | — | 数据源 URI 主机黑名单。匹配的 URI 被拒绝连接 |
| `plugins.query.datasources.encryption.masterkey` | string | 无（必须设置） | ❌ | — | 数据源凭据加密主密钥。**Final 设置**，设置后不可更改，必须在 `opensearch.yml` 中配置 |

**使用示例：**
```bash
# 增加数据源数量上限
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.datasources.limit": 50 }
}'

# 设置主机黑名单
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.datasources.uri.hosts.denylist": ["10.0.0.1", "internal.host"] }
}'
```

> ⚠️ `plugins.query.datasources.encryption.masterkey` 必须在 `opensearch.yml` 中设置：
> ```yaml
> plugins.query.datasources.encryption.masterkey: "your-256-bit-secret-key"
> ```

---

## 8. Spark 异步查询引擎设置

> 以下设置适用于通过 Spark 执行的异步查询（Async Query）。

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.query.executionengine.spark.config` | string | 无 | ✅ | — | Spark 执行引擎配置（JSON 字符串） |
| `plugins.query.executionengine.spark.session.limit` | integer | `10` | ✅ | 2.12 | 集群最大并行 Spark 会话数 |
| `plugins.query.executionengine.spark.refresh_job.limit` | integer | `5` | ✅ | 2.12 | 集群最大并行刷新作业数 |
| `plugins.query.executionengine.spark.session_inactivity_timeout_millis` | long | `180000` (3m) | ✅ | 2.12 | Spark 会话不活跃超时时间（毫秒） |
| `plugins.query.executionengine.spark.auto_index_management.enabled` | boolean | `true` | ✅ | 2.12 | 自动管理请求和结果索引。启用时自动删除过期索引文档 |
| `plugins.query.executionengine.spark.session.index.ttl` | time | `30d` | ✅ | 2.12 | 请求索引 TTL。超过此时间的索引文档被自动删除 |
| `plugins.query.executionengine.spark.result.index.ttl` | time | `60d` | ✅ | 2.12 | 结果索引 TTL。超过此时间的索引文档被自动删除 |
| `plugins.query.executionengine.spark.streamingjobs.housekeeper.interval` | time | `15m` | ✅ | 2.13 | 流式作业清理器运行间隔。清理已删除/禁用数据源的流式作业 |

**使用示例：**
```bash
# 增加并行会话数
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.executionengine.spark.session.limit": 200 }
}'

# 设置会话不活跃超时为 10 分钟
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.executionengine.spark.session_inactivity_timeout_millis": 600000 }
}'
```

---

## 9. 异步查询调度设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.query.executionengine.async_query.enabled` | boolean | `true` | ✅ | 2.12 | 启用/禁用异步查询提交 |
| `plugins.query.executionengine.async_query.external_scheduler.enabled` | boolean | `true` | ✅ | 2.17 | 启用外部调度器处理自动刷新查询 |
| `plugins.query.executionengine.async_query.external_scheduler.interval` | string | 无（需显式设置） | ✅ | 2.17 | 外部调度器运行间隔。格式遵循 Spark CalendarInterval（如 `"10 minutes"`、`"1 hour"`） |

**使用示例：**
```bash
# 禁用异步查询
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.executionengine.async_query.enabled": "false" }
}'

# 设置外部调度器间隔为 10 分钟
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": { "plugins.query.executionengine.async_query.external_scheduler.interval": "10 minutes" }
}'
```

---

## 10. 监控指标设置

| 参数 | 类型 | 默认值 | 动态 | 引入版本 | 说明 |
|------|------|--------|:----:|:--------:|------|
| `plugins.query.metrics.rolling_window` | long | `3600` (秒) | ✅ | — | 指标滚动窗口大小（秒）。最小值 2 |
| `plugins.query.metrics.rolling_interval` | long | `60` (秒) | ✅ | — | 指标滚动间隔（秒）。最小值 1 |

---

## 11. 非动态设置（需重启）

以下设置在 `opensearch.yml` 中配置，**不支持动态更新**：

| 参数 | 类型 | 默认值 | 属性 | 说明 |
|------|------|--------|------|------|
| `plugins.query.datasources.encryption.masterkey` | string | 无 | Final, Filtered | 数据源凭据加密主密钥。**必须设置**才能使用数据源创建/更新 API |
| `plugins.query.federation.datasources.config` | file | 无 | Deprecated | 旧版数据源配置文件路径（已废弃，将在未来版本移除） |

---

## 12. 线程池设置

在 `opensearch.yml` 中配置（3.4+ 引入）：

```yaml
thread_pool:
  sql-worker:
    size: 30
    queue_size: 100
  sql_background_io:
    size: 30
    queue_size: 1000
```

| 线程池 | 说明 |
|--------|------|
| `sql-worker` | 查询计算线程池。直接映射可并发执行的查询数，是主要的外部交互池 |
| `sql_background_io` | IO 请求线程池。低开销，用于限制 Calcite 操作对集群的间接负载。一个 `sql-worker` 线程可能产生多个后台 IO 线程 |

---

## 13. 配置方式

### 方式一：`opensearch.yml`（启动时配置）

适用于非动态设置和默认值配置：

```yaml
# opensearch.yml
plugins.calcite.enabled: true
plugins.query.datasources.encryption.masterkey: "your-256-bit-secret-key"

thread_pool:
  sql-worker:
    size: 30
    queue_size: 100
```

### 方式二：Cluster Settings API（动态更新）

适用于所有动态设置，无需重启：

```bash
# transient（临时，重启后失效）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "transient": {
    "plugins.calcite.enabled": "true",
    "plugins.query.size_limit": 10000
  }
}'

# persistent（持久化，重启后保留）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": {
    "plugins.calcite.enabled": "true"
  }
}'
```

### 方式三：SQL Plugin Settings API（动态更新）

SQL 插件专用的设置端点，效果与 Cluster Settings API 相同：

```bash
curl -X PUT "localhost:9200/_plugins/_query/settings" -H 'Content-Type: application/json' -d '{
  "transient": {
    "plugins.sql.enabled": "true"
  }
}'
```

### 查看当前设置

```bash
# 查看所有集群设置
curl "localhost:9200/_cluster/settings?pretty&include_defaults=true"

# 查看指定设置
curl "localhost:9200/_cluster/settings?pretty" | python3 -m json.tool
```

---

## 14. 使用建议

### 生产环境推荐配置

```bash
# 1. 启用 Calcite 引擎（3.3+ 默认已启用）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.calcite.enabled": "true" }
}'

# 2. 启用算子下推（默认已启用，提升查询性能）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.calcite.pushdown.enabled": "true" }
}'

# 3. 设置内存限制为 80%（防止 OOM）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.query.memory_limit": "80%" }
}'

# 4. 设置慢查询日志阈值为 5 秒
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.sql.slowlog": "5" }
}'

# 5. 设置游标存活时间为 3 分钟（平衡资源占用和分页体验）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{
  "persistent": { "plugins.sql.cursor.keep_alive": "3m" }
}'
```

### 场景化建议

| 场景 | 推荐配置 | 原因 |
|------|---------|------|
| **高并发查询** | `thread_pool.sql-worker.size: 50+` | 增加并发查询数 |
| **大数据量分页** | `plugins.sql.cursor.keep_alive: 5m` + `plugins.query.size_limit: 10000` | 避免游标过期，限制单次返回行数 |
| **内存受限环境** | `plugins.query.memory_limit: 70%` | 留更多内存给 OpenSearch 核心 |
| **调试 Calcite 问题** | `plugins.calcite.fallback.allowed: true` | Calcite 失败时回退到 V2，避免查询直接报错 |
| **需要 RIGHT/FULL/CROSS JOIN** | `plugins.calcite.all_join_types.allowed: true` | 启用所有 JOIN 类型（注意性能影响） |
| **使用 UNION ALL** | 确保 `plugins.calcite.enabled: true` | SQL UNION ALL 通过 Calcite 引擎执行 |
| **使用数据源** | 在 `opensearch.yml` 设置 `masterkey` | 必须配置才能使用数据源 API |
| **使用异步查询 (Spark)** | 调整 `session.limit` 和 `session_inactivity_timeout` | 根据负载调整并行会话数和超时 |
| **PPL 长查询** | `plugins.ppl.query.timeout: 600s` | 增加超时时间避免复杂 PPL 被终止 |
| **安全加固** | `plugins.query.datasources.uri.hosts.denylist: ["内网地址"]` | 防止数据源连接到内部主机 |

### 注意事项

1. **`transient` vs `persistent`**：`transient` 重启后失效，`persistent` 持久化。生产环境建议用 `persistent`。
2. **`plugins.calcite.fallback.allowed`**：默认 `false` 意味着 Calcite 不支持的查询会直接报错而不是静默回退。开发环境可设为 `true` 方便调试。
3. **`plugins.query.size_limit`** 不能超过索引级 `index.max_result_window`（默认 10000）。如需更多结果，使用游标分页。
4. **`plugins.query.buckets`** 仅在 Calcite 启用时生效。V2 引擎中聚合桶数固定为 1000。
5. **`plugins.query.datasources.encryption.masterkey`** 是 Final 设置，设置后不可更改。更换密钥需要重新创建所有数据源。
6. **Calcite 引擎对 SQL 的支持范围**：3.7.0 中 SQL 的 UNION/UNION ALL 通过 Calcite 执行（本扩展支持），其他 SQL 语法仍走 V2 引擎。PPL 全面走 Calcite。
7. **线程池配置**：`sql-worker` 的 `size` 决定最大并发查询数。`queue_size` 决定排队等待的查询数。超出的请求会被拒绝。
