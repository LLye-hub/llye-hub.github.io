---
title: 搭建spark on yarn源码调试
tags:
  - spark on yarn
categories:
  - spark
toc: true
abbrlink: 5e1f3fb0
date: 2023-03-21 17:02:06
---
**参考文章**
[Spark3.0.1各种集群模式搭建及spark on yarn日志配置](https://www.cnblogs.com/qixing/p/14017875.html)
[Spark on Yarn集群搭建详细过程](https://www.jianshu.com/p/aa6f3a366727)

# 环境准备
jdk8+Hadoop3.3.1 

# spark的几种部署方式
Spark作为准实时大数据计算引擎，Spark的运行需要依赖资源调度和任务管理，Spark自带了standalone模式资源调度和任务管理工具，运行在其他资源管理和任务调度平台上，如Yarn、Mesos、Kubernates容器等。

spark的搭建和Hadoop差不多，主要有下面几种部署方式：

* Local：多用于本地测试，如在eclipse，idea中写程序测试等。

* Standalone：Standalone是Spark自带的一个资源调度框架，它支持完全分布式。
  
* Yarn：Hadoop生态圈里面的一个资源调度框架，Spark也是可以基于Yarn来计算的。

基于个人学习需求，本文仅记录Local模式部署过程。

# 下载spark源码
[下载地址](https://archive.apache.org/dist/spark/)

**解压**
```shell
tar -zcvf spark-3.2.0-bin-hadoop3.2.tgz
```

**配置环境变量**
```shell
# 编辑
# SPARK_HOME=/Users/llye/workspace/spark-3.2.0-bin-hadoop3.2 
# export PATH=$SPARK_HOME/bin:$PATH
vi ~/.bash_profile 

# 生效
source ~/.bash_profile
```

# 本地local模式
**测试样例**
```shell
cd $SPARK_HOME/bin 
run-example SparkPi 10  # 可计算出结果
```
![iV6lXq.png](https://i.328888.xyz/2023/03/24/iV6lXq.png)

```shell
spark-shell  # 启动成功，说明Local模式部署成功
```
![iV6HUw.png](https://i.328888.xyz/2023/03/24/iV6HUw.png)

启动成功后，访问`http://localhost:4040/` 即可进行web UI监控页面访问

# Standalone模式（未完成，不具参考性）

配置Spark on Yarn集群

**修改spark-env.sh文件**
```shell
cd $SPARK_HOME/conf
cat > spark-env.sh << EOF
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/
SCALA_HOME=/Users/llye/soft/scala/
HADOOP_HOME=/Users/llye/soft/hadoop-3.3.1/
HADOOP_CONF_DIR=/Users/llye/soft/hadoop-3.3.1/etc/hadoop/
YARN_CONF_DIR=/Users/llye/soft/hadoop-3.3.1/etc/hadoop/etc/hadoop/
SPARK_MASTER_HOST=spark    # 主节点机器名称
SPARK_MASTER_PORT=7077     # 默认端口号7077
SPARK_HOME=/Users/llye/workspace/spark-3.2.0-bin-hadoop3.2/
SPARK_LOCAL_DIRS=/Users/llye/workspace/spark-3.2.0-bin-hadoop3.2/
SPARK_LIBARY_PATH=/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home/lib/:/Users/llye/soft/hadoop-3.3.1/lib/native/
EOF
```

**修改slaves配置文件**
```shell
cd $SPARK_HOME/sbin 
vi slaves
# spark001
# spark002
```

**将spark目录发送到其他机器**


创建workers文件，指定Worker节点：
```shell
cd $SPARK_HOME/conf
cat > workers << EOF
worker1
worker2
worker3
EOF
```


# 启动Spark on Yarn集群

```shell
cd $SPARK_HOME/sbin
```

在Spark节点上启动Spark Master节点：
```shell
start-master.sh
```

在Worker节点上启动Spark Worker节点：
```shell
start-worker.sh spark://spark:7077
```

# 登录Spark on Yarn集群

登录Master：

登录Worker：`http://localhost:8081/`