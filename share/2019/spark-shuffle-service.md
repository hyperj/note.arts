# Spark Shuffle Service

在进行 Shuffle 过程，上游 Executor Mapper 阶段，将数据写到本地磁盘，下游 Executor Reducer 阶段，通过上游 Executor 拉取数据；尤其在动态资源分配情形下，Executor 可能被移除或抢占，就需要重算这部分数据

在 YARN 调度模式下，通过在 NodeManager 上启动 YarnShuffleService（ExternalShuffleService） 长时辅助服务，提供统一的 Shuffle 服务，优雅地关闭 Executor，并保留其状态

Shuffle 过程变为：上游 Executor Mapper 阶段，向本地 Shuffle Service 注册，将数据写到本地磁盘，就可以退出；下游 Executor Reducer 通过上游 Executor 的 Shuffle Service 拉取数据

目前主要包含以下三种实现：

- ExternalShuffleService（Standalone）
- YarnShuffleService（YARN）
- MesosExternalShuffleService（Mesos）

## 重要组件（YARN 模式）

- YarnShuffleService：NodeManager 启动的 Spark Shuffle Service 长时辅助服务，管理作业、服务、恢复
- TransportServer：实际提供 Shuffle 的服务
- ExternalShuffleBlockHandler：TransportServer 的 blockHandler，处理 RPC 请求

## 特点与问题（YARN 模式）

- 数据存储在本地磁盘，没有备份
- IO 并发：大量 RPC 请求（M*R）
- IO 吞吐：热点数据、随机读、写放大（3X）
- GC 频繁，影响 NodeManager

## 优化思路

- 数据容错：S3、副本、WAL
- 服务容错：负载均衡、Failover
- IO优化：增大block、减少随机读
- GC调优
- 流控：服务端、客户端（避免失败）
- 监控告警
- 与NodeManager隔离

## Reference

- [Cosco: An Efficient Facebook-Scale Shuffle Service](https://databricks.com/session/cosco-an-efficient-facebook-scale-shuffle-service)
