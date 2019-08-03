# 元数据集成

**类型**

- 审计
- 操作
- 监控
- 度量

**机制**

- Hook & Bridge 机制
- Metrics 机制
- 中间件、网关、工具链
- 拉取，推送、同步
- 实时、离线

## HDFS

- 基于 HDFS Checkpoint 机制
- 基于 QJM 架构，从 JournalNode 拉取数据

## Hive

- Hive 提供的 [Hook](https://github.com/apache/hive/blob/master/ql/src/java/org/apache/hadoop/hive/ql/hooks/Hook.java) 机制

## HBase

- HBase 提供的 [Coprocessor](https://blogs.apache.org/hbase/entry/coprocessor_introduction) 机制

## Kafka

- Kafka 提供的 [ZkUtils](https://github.com/apache/kafka/blob/2.3/core/src/main/scala/kafka/utils/ZkUtils.scala) 工具

## Kylin

- Kylin 提供的 [ResourceTool](https://github.com/apache/kylin/blob/master/core-common/src/main/java/org/apache/kylin/common/persistence/ResourceTool.java) 工具

## Druid

- 通过 MySQL 集成方式，参考 [SQLMetadataConnector](https://github.com/apache/incubator-druid/blob/master/server/src/main/java/org/apache/druid/metadata/SQLMetadataConnector.java)

## ElasticSearch

- 通过 [REST APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html) 方式

## MySQL

- 通过 Canal 进行 binlog 增量订阅&消费

## Other

- Impala
- Redis
- TiDB
- Tair
- YARN
- Spark
- Flink
- Oozie

## Reference

- [Apache Atlas](https://atlas.apache.org/index.html)
- [Navigator Metadata Server Management](https://www.cloudera.com/documentation/enterprise/latest/topics/cn_admcfg_nms_intro.html)
- [How to write a Hive Hook](http://dharmeshkakadia.github.io/hive-hook/)