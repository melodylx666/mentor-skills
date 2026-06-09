# 最佳实践参考：Spark SQL 参数调优

> 本示例展示 mentor-guide 在处理"参数调优"类问题时的输出风格 —— 以结构化参考手册形式组织最佳实践，并对每条建议标注信源等级。
>
> 问题：**"Spark SQL 参数调优最佳实践"**

---

## 核心调优思路

Spark SQL 调优围绕 **"少读、少传、少倾斜"** 三个目标展开：减少数据扫描、减少 Shuffle 开销、避免数据倾斜。

[[Spark SQL 调优实战](https://www.cnblogs.com/robots2/p/19842064) — `[C]`]

---

## 一、AQE（Adaptive Query Execution）— 必开选项

AQE 是 Spark 3.x 最核心的优化特性，能在运行时根据统计信息动态优化执行计划。

[[Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`]

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `spark.sql.adaptive.enabled` | `true` | AQE 总开关 |
| `spark.sql.adaptive.coalescePartitions.enabled` | `true` | 自动合并 Shuffle 后的小分区 |
| `spark.sql.adaptive.advisoryPartitionSizeInBytes` | `64MB` | 合并后目标分区大小 |
| `spark.sql.adaptive.coalescePartitions.minPartitionNum` | `25` | 合并后最小分区数 |
| `spark.sql.adaptive.coalescePartitions.initialPartitionNum` | `1000` | Shuffle 初始分区数 |
| `spark.sql.adaptive.skewJoin.enabled` | `true` | 自动处理倾斜 Join |
| `spark.sql.adaptive.skewJoin.skewedPartitionFactor` | `5` | 倾斜分区判定因子 |
| `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` | `256MB` | 倾斜分区阈值 |
| `spark.sql.adaptive.optimizeSkewsInRebalancePartitions.enabled` | `true` | 优化重平衡中的倾斜 |
| `spark.sql.adaptive.localShuffleReader.enabled` | `true` | 优化 Shuffle 读取 |

**参考来源**：
- [Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`
- [AWS EMR Spark 优化](https://docs.aws.amazon.com/en_us/emr/latest/ReleaseGuide/emr-spark-performance.html) — `[A]`

---

## 二、资源分配参数

| 参数 | 推荐值原则 | 说明 |
|------|-----------|------|
| `spark.executor.memory` | 每 Executor 4GB~16GB | 避免过大导致 GC 压力 |
| `spark.executor.cores` | 3~5 个 | 过多核会导致 IO 争抢 |
| `spark.executor.instances` | 根据集群规模计算 | 单节点 Executor 数 = 可用核数 / executor.cores |
| `spark.driver.memory` | 2GB~8GB | `collect()` 大结果时需加大 |
| `spark.memory.fraction` | `0.6`（默认） | Execution + Storage 占总堆内比例 |
| `spark.memory.storageFraction` | `0.5`（默认） | Storage 占统一内存比例 |
| `spark.executor.memoryOverhead` | executor.memory × 0.1 或 1GB | 堆外内存 |
| `spark.sql.autoBroadcastJoinThreshold` | `10MB~100MB` | 广播 Join 阈值 |
| `spark.serializer` | KryoSerializer | 比 Java 序列化快 10x |
| `spark.sql.shuffle.partitions` | 由 AQE 接管 | 手动设置按 cores × 2~3 |

**参考来源**：
- [Spark SQL优化方案总结](https://blog.csdn.net/winterPassing/article/details/148333112) — `[C]`
- [大数据技术之Spark性能调优](https://blog.csdn.net/2301_80395772/article/details/161481717) — `[C]`

---

## 三、读取与分区优化

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `spark.sql.files.maxPartitionBytes` | `128MB~256MB` | 读取时单个分区最大字节数 |
| `spark.sql.files.openCostInBytes` | `4MB~8MB` | 打开文件的估算开销 |
| `spark.sql.parquet.mergeSchema` | `false` | 非必要时关闭 |

**存储格式建议**：
- 优先使用 **Parquet / ORC** 列式存储，支持谓词下推和列裁剪
- 避免使用 CSV / JSON 文本格式
- 按高频过滤字段（如日期）做分区

**参考来源**：
- [Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`
- [Apache Spark 性能优化](https://blog.csdn.net/NIIT0532/article/details/148799136) — `[C]`

---

## 四、Join 策略优化

```sql
-- 手动广播 Hint
SELECT /*+ BROADCAST(users) */
       o.order_id, u.user_name, o.amount
FROM orders o JOIN users u ON o.user_id = u.user_id;
```

**常用 Hint**：

| Hint | 用法 | 适用场景 |
|------|------|---------|
| `BROADCAST` | `SELECT /*+ BROADCAST(t) */ ...` | 大表 Join 小表 |
| `MERGE` | `SELECT /*+ MERGE(t) */ ...` | 两表都大，且已分桶 |
| `SHUFFLE_HASH` | `SELECT /*+ SHUFFLE_HASH(t) */ ...` | 大表 Join 中表 |
| `SHUFFLE_REPLICATE_NL` | `SELECT /*+ SHUFFLE_REPLICATE_NL(t) */ ...` | 非等值 Join |

**参考来源**：
- [Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`
- [Spark SQL 进阶实践](https://juejin.cn/post/7626269582288076819) — `[C]`

---

## 五、缓存与存储

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `spark.sql.inMemoryColumnarStorage.compressed` | `true` | 内存缓存启用压缩 |
| `spark.sql.inMemoryColumnarStorage.batchSize` | `10000` | 列式缓存批大小，防 OOM |

- 多次使用的中间结果用 `.cache()` 或 `.persist(StorageLevel.MEMORY_AND_DISK)`
- 用完及时 `.unpersist()`

**参考来源**：
- [Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`
- [Apache Spark 性能优化](https://blog.csdn.net/NIIT0532/article/details/148799136) — `[C]`

---

## 六、动态资源分配

```properties
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.minExecutors=5
spark.dynamicAllocation.maxExecutors=100
spark.dynamicAllocation.initialExecutors=5
spark.dynamicAllocation.executorIdleTimeout=60s
```

**适用场景**：集群资源共享、负载波动大。

**参考来源**：
- [Apache Spark 性能优化](https://blog.csdn.net/NIIT0532/article/details/148799136) — `[C]`

---

## 七、GC 优化

```bash
spark-submit \
  --conf "spark.executor.extraJavaOptions=-XX:+UseG1GC -XX:MaxGCPauseMillis=200" \
  --conf "spark.driver.extraJavaOptions=-XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

大内存 Executor 推荐 G1GC 替代默认的 Parallel GC，减少 Full GC 暂停时间。

**参考来源**：
- [Spark SQL优化方案总结](https://blog.csdn.net/winterPassing/article/details/148333112) — `[C]`
- [大数据技术之Spark性能调优](https://blog.csdn.net/2301_80395772/article/details/161481717) — `[C]`

---

## 八、数据倾斜处理

### 自动方案（推荐）

开启 AQE 后，Spark 3.x+ 自动处理倾斜：

```properties
spark.sql.adaptive.skewJoin.enabled=true
```

[[Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`]

### 手动方案：加盐

适用于 AQE 无法自动处理的场景，核心思路：小表扩容 + 大表打盐。

[[Spark SQL 调优实战](https://www.cnblogs.com/robots2/p/19842064) — `[C]`]

```sql
WITH orders_salted AS (
  SELECT *, pmod(hash(user_id, order_id), 8) AS salt
  FROM orders
),
users_expanded AS (
  SELECT *, salt
  FROM users
  LATERAL VIEW explode(array(0,1,2,3,4,5,6,7)) tmp AS salt
)
SELECT o.order_id, o.user_id, o.amount, u.user_name
FROM orders_salted o
JOIN users_expanded u ON o.user_id = u.user_id AND o.salt = u.salt;
```

**参考来源**：
- [Apache Spark Performance Tuning](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) — `[A]`
- [Spark SQL 调优实战](https://www.cnblogs.com/robots2/p/19842064) — `[C]`
- [Spark SQL 进阶实践](https://juejin.cn/post/7626269582288076819) — `[C]`

---

## 九、调优路径总结

```
发现问题（慢/OOM/倾斜）
  └→ 看 Spark UI（Stage 耗时、Shuffle 量、GC）
      └→ 用 explain 分析执行计划
          ├→ SQL 写法问题 → 优化 SQL（谓词下推、列裁剪、避免 select *）
          ├→ Join 策略不对 → 加 Broadcast Hint / 调整阈值
          ├→ 数据倾斜 → 开启 AQE / 加盐
          ├→ 分区不合理 → 调整 maxPartitionBytes / 用 AQE 自动合并
          └→ 资源不足 → 调整 memory / cores / instances
```

> **最重要原则**：先优化 SQL 再调参数，SQL 层面的优化收益远大于参数调整。

**参考来源**：
- [Spark SQL 调优实战](https://www.cnblogs.com/robots2/p/19842064) — `[C]`
- [Spark SQL 进阶实践](https://juejin.cn/post/7626269582288076819) — `[C]`

---

## 信源汇总

| 序号 | 信源 | 等级 |
|------|------|------|
| 1 | [Apache Spark Performance Tuning (4.1.0)](https://spark.apache.org/docs/4.1.0/sql-performance-tuning.html) | `[A]` |
| 2 | [AWS EMR — Optimize Spark Performance](https://docs.aws.amazon.com/en_us/emr/latest/ReleaseGuide/emr-spark-performance.html) | `[A]` |
| 3 | [Spark SQL 调优实战：少读、少传、少倾斜](https://www.cnblogs.com/robots2/p/19842064) | `[C]` |
| 4 | [Spark SQL优化方案总结](https://blog.csdn.net/winterPassing/article/details/148333112) | `[C]` |
| 5 | [大数据技术之Spark性能调优](https://blog.csdn.net/2301_80395772/article/details/161481717) | `[C]` |
| 6 | [Apache Spark 性能优化](https://blog.csdn.net/NIIT0532/article/details/148799136) | `[C]` |
| 7 | [Spark SQL 进阶实践](https://juejin.cn/post/7626269582288076819) | `[C]` |

> **等级说明**（参考 [source-credibility.md](../references/source-credibility.md)）：
> - `[A]` 权威 — 官方文档 / 官方技术博客
> - `[C]` 参考 — 个人博客 / 社区分享，建议在测试环境验证后再用于生产

---

*本示例为参考资料型输出，展示 mentor-guide 在处理"最佳实践参考"类问题时的结构化输出风格。所有信源链接均为真实可查内容。*
