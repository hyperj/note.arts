# Flink 开发实践

## 基础概念

- Source、Transformation、Sink
- Window：Tumbling Window、Sliding Window、Session Window、Global Window
- State：Keyed State、Operator State；Managed State、Raw State
- StateBackend：Memory、Fs、RocksDB
- Checkpoint、Savepoint

## 开发优化

- 参数化配置信息
- 设置 parallelism、name、uid
- 处理晚到数据
- 获取丢弃数据
- 并发度控制
- TM数量、内存

## 问题排查

- 延迟：慢节点、大消息、资源不足、数据增长、节点异常、处理异常、依赖服务异常
- 失败：资源不足/超额、环境异常、依赖服务异常、节点故障、处理失败、OOM

**排查**

- UI
- Metrics
- Log

## Reference