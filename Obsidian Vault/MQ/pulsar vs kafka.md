#### Kafka vs Pulsar 服务端持久化对比

|**特性**|**Kafka**|**Apache Pulsar**|
|---|---|---|
|**服务端架构**|**存算一体**（Broker 即负责计算，也直接把数据写在本地磁盘）|**存算分离**（Broker 仅做无状态计算，BookKeeper 负责持久化）|
|**持久化流程**|Producer $\rightarrow$ Broker $\rightarrow$ 写入 Broker 本地 Page Cache / 磁盘|Producer $\rightarrow$ Broker $\rightarrow$ **写往 Bookie 节点**（BookKeeper 存储层磁盘）|

**在 Pulsar 中，服务端内部又拆成了两层：**

1. **Pulsar Broker（计算层）：** 它是**无状态（Stateless）**的。负责接收 Producer 的请求、处理路由、做内存 Caching 和消息分发。它自己**不存消息数据**到本地磁盘。
2. **Apache BookKeeper / Bookie（存储层）：** 专门负责消息的**顺序日志持久化（Write-Ahead Log）**。Broker 收到消息后，会将其写入 Bookie 集群。只有当 Bookie 确认落盘后，服务端才会给 Producer 返回成功的 ACK。

### 总结

无论是 Kafka 还是 Pulsar：

1. **客户端（Consumer）内存不够：** 只需扩容客户端本身（调大资源或调整拉取参数），不需要动服务端。
2. **持久化归属：** 都发生在服务端处理阶段，客户端不承担存储责任。
3. **架构差异：** Pulsar 将服务端的**计算与存储**彻底拆开，这使得 Pulsar 服务端自身在扩容时（如仅扩容存储能力或仅扩容吞吐能力）比 Kafka 更加敏捷、无需重新迁移（Rebalance）分区数据。