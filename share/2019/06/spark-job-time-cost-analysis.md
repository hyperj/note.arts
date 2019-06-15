# Spark 作业耗时分析

## Spark 作业执行流程（Cluster Mode On YARN）

- [Spark 作业执行流程（Cluster Mode On YARN）](spark-job-execution-process.md)

## 外部影响

### 在线调度系统（e.g. Oozie、Airflow）

- 用户提交作业到在线应用调度系统，系统根据应用或系统自身的约束条件，将作业分配到对应应用调度队列排队，执行机将作业提交到对应的集群或服务（资源调度系统）

### 资源调度系统（e.g. YARN、Mesos）

- 根据约束条件，进入对应资源调度队列排队，申请资源，启动ApplicationMaster，ApplicationMaster启动Driver，申请资源，启动Executor
- 资源不足时，满足最小注册资源比例或达到等待时间，开始执行

### 元数据系统（e.g. Hive Metastore）

- 模型元数据信息，Catalog（e.g. databases、tables、columns,、partitions、functions) 

### 存储系统（e.g. HDFS、Alluxio）

- input（get splits）：大文件分割问题；output（commit job）：小文件存储问题
- 文件格式：ORC、Parquet

### 权限/审计（e.g. Ranger、Sentry）

- 鉴权：资源控制粒度、鉴权时机
- 审计

## 内部机制

### 调度

- Job
- Stage
- Task

### Shuffle Service

- read（remote、local）
- write

### DataSource V2

- input（get splits）：大文件分割问题
- output（commit job）：小文件存储问题

**特性**

- Filter Push
- Partitioning
- Transactional

### 任务

- 资源不充足：调度延迟、资源使用率高
- 序列化与反序列化：调整序列化
- 复杂计算：分段计算
- 数据记录数量大：提高并行度
- 数据记录存储大：减少冗余信息
- 数据倾斜（TODO）：加盐、分组

### 其他

- GC
- 本地化
- 资源使用率
- 超时与重试

## Reference

