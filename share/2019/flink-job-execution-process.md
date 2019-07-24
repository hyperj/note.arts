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

## Reference