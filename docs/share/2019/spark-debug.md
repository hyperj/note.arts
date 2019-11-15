# Spark 源码调试

## 环境

只涉及 Java 及 Scala 代码的编译、测试及调试

- macOS 10.14.5
- Java 8u152
- Scala 2.12.8
- Maven 3.6.0
- SBT 0.13.18
- IDEA 2018.3.4
- Spark 2.4.3

## 命令

### 编译

**maven**

```bash
./build/mvn -DskipTests clean package
```

```bash
./build/mvn -pl :spark-core_2.12 -DskipTests clean package -am
```

- `-pl`，编译指定模块，{groupId}:{artifactId} 或是 dir_path
- `-am`，同时编译依赖模块

```bash
./dev/make-distribution.sh --tgz -Phadoop-2.7 -Phive -Phive-thriftserver -Pyarn -DskipTests
```

**sbt**

```bash
./build/sbt -Pscala-2.12 -Phive -Phive-thriftserver -Pyarn -Dhadoop-2.7 -DskipTests -Dsbt.override.build.repos=false

project core
package
```

### 测试

**maven**

```bash
./build/mvn -pl :core
```

```bash
./build/mvn test -DwildcardSuites=none -Dtest=org.apache.spark.streaming.JavaAPISuite
```

- `-DwildcardSuites`，指定 Scala 测试
- `-Dtest`，指定 Java 测试

```bash
./build/mvn test -pl core -Dtest=none -Dsuites='*DAGSchedulerSuite SPARK-3353'
```

- Scala Test 使用通配符匹配：类、方法、测试名称

**sbt**

```bash
./build/sbt -Pscala-2.12 -Phive -Phive-thriftserver -Pyarn -Dhadoop-2.7 -DskipTests -Dsbt.override.build.repos=false

project core
test
testOnly *DAGSchedulerSuite -- -z "SPARK-3353"
```

### 调试

IDEA：Run > Edit Configurations > + > Remote > Host `localhost`、Port `5005`、Command `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005`

```bash
./bin/spark-submit --driver-java-options "-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005" --class org.apache.spark.examples.SparkPi examples/target/spark-examples_2.11-2.4.3-SNAPSHOT.jar
```

- Standalone 模式，可只启动一个 Executor 进行调试

**maven**

```bash
./build/mvn test -pl core -Dtest=none -Dsuites='*DAGSchedulerSuite' -DdebugForkedProcess=true -DdebuggerPort=5005
```

- `-DdebugForkedProcess=true`，开启 UT 调试，默认端口 5005

**sbt**

```bash
./build/sbt -Pscala-2.12 -Phive -Phive-thriftserver -Pyarn -Dhadoop-2.7 -DskipTests -Dsbt.override.build.repos=false -jvm-debug 5005

project core
set fork in Test := false
testOnly org.apache.spark.scheduler.DAGSchedulerSuite
```

- `-DdebugForkedProcess=true`，开启 UT 调试，默认端口 5005

## 问题

## Reference

- [Spark Github](https://github.com/apache/spark)
- [Building Spark](http://spark.apache.org/docs/latest/building-spark.html)
- [Useful Developer Tools](http://spark.apache.org/developer-tools.html)
- [Using the ScalaTest Maven plugin](http://www.scalatest.org/user_guide/using_the_scalatest_maven_plugin)
- [SBT Testing](https://www.scala-sbt.org/1.x/docs/Testing.html)
- [Java 生产环境 debug](https://segmentfault.com/a/1190000010211498)
