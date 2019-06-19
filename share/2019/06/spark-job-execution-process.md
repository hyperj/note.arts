# Spark 作业执行流程（Cluster Mode On YARN）

- spark-submit

检查 SPARK_HOME，提交给 spark-class，指定执行类

- spark-class

检查 SPARK_HOME，加载 Spark 环境变量，检查 Java 执行路径，检查 Spark 依赖路径，拼接命令，执行命令

- launcher.Main

拼接命令，输出命令

- deploy.SparkSubmit

解析参数，准备环境，调用 child class main（YARN Cluster 模式通过 yarn.Client）

    - createApplication 
    - verifyClusterResources 
    - createContainerLaunchContext/createApplicationSubmissionContext（__app__.jar、__spark_libs__、__spark_conf__、launch_container.sh ...）
    - submitApplication

- User Class Main

SparkSession、SparkContext

Driver: runJob -> DAGScheduler#runJob, submitJob -> DAGSchedulerEventProcessLoop#doOnReceive(JobSubmitted) -> DAGScheduler#handleJobSubmitted, submitStage(Submits stage, but first recursively submits any missing parents, BFS.), submitMissingTasks -> TaskScheduler#submitTasks(TaskSet) -> CoarseGrainedSchedulerBackend#reviveOffers

Executor: CoarseGrainedExecutorBackend#revive -> Executor#launchTask -> TaskRunner#run -> Task#run -> ShuffleMapTask(MapStatus), ResultTask(func():U)#runTask -> CoarseGrainedExecutorBackend#statusUpdate

- 资源注销、清理

## Reference