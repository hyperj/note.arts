# Flink 作业执行流程（Cluster Mode On YARN）

- flink

获取执行路径，获取配置信息，环境变量，日志配置，执行命令，提交给 CliFrontend

- CliFrontend

加载环境及配置信息，初始化，解析并执行请求

**run**

    解析配置信息，生成运行时信息
    
    构建 PackagedProgram
    
    执行 runProgram，获取 ClusterDescriptor，计算并行度，生成 JobGraph，提交作业
    
    清理资源及文件

**build**

- code -> transformations List
- StreamGraphGenerator -> StreamGraph
- StreamingJobGraphGenerator -> JobGraph
- ExecutionGraphBuilder -> ExecutionGraph

**resource**

- 查找已分配 slot 是否符合条件
- 查找 slotPool 是否有空闲的
- 向 RM 组件申请 slot，向 YARN 的 RM 申请 container
- YARN 初始化 container，通知 container 申请成功
- 部署 TM ，向 RM 注册
- RM 向 TM 申请 slot
- TM 向 JM 注册，并提供 slot
- JM 向 TM 的 slot 部署 task

## Reference