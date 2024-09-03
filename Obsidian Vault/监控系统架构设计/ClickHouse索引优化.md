1. `TYPE minmax` 指定了索引类型为`minmax`，这是ClickHouse中一种常用的索引类型，适用于范围查询，千万不能把该类型的索引应用在array或者map类型上。
2. Bloom Filter类型，布鲁过滤器对大量分散的离散数据效果非常好，但是可能会存在false positive（假阳性）数据的问题，该索引可以运用在array或者mao类型上。
3. Set 类型， Set类型主要针对于特定的优先的数量级，类似于枚举。当cardinality数值量小的时候，该索引的性能高。

最佳实践：
1. `minmax`：ALTER TABLE op_metrics.samples ADD INDEX ts_idx(ts) TYPE minmax GRANULARITY 1;
2. `BloomFilter`：ALTER TABLE op_metrics.samples ADD INDEX tag_idx tags TYPE bloom_filter(0.01) GRANULARITY 1;




[[监控系统后续优化]]

reference
https://clickhouse.com/docs/en/optimize/skipping-indexes#bloom-filter-types