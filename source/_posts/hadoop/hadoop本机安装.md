---
title: hadoop本机安装
tags:
  - hadoop安装
categories:
  - hadoop
toc: true
abbrlink: 2cb81866
date: 2023-03-15 10:49:02
---
**参考文章**

[Hive源码系列（一）hive2.1.1+hadoop2.7.3环境搭建](https://zhuanlan.zhihu.com/p/68748400)

[Hadoop【单机安装-测试程序WordCount】](https://cloud.tencent.com/developer/article/1708064)

Hadoop 安装有三种方式：

`单机模式`：安装简单，几乎不用做任何配置，但仅限于调试用途；

`伪分布模式`：在单节点上同时启动 NameNode、DataNode、JobTracker、TaskTracker、Secondary Namenode 等 5 个进程，模拟分布式运行的各个节点；

`完全分布式模式`：正常的 Hadoop 集群，由多个各司其职的节点构成。

本人选择的是`伪分布模式`安装


**下载安装**

[下载地址](https://hadoop.apache.org/)

* 解压安装包
```shell
tar -zxvf hadoop-3.3.1.tar.gz
```

**环境变量配置**

```shell
# 编辑
# HADOOP_HOME=/Users/llye/soft/hadoop-3.3.1
# export PATH=$HADOOP_HOME/bin:$PATH
vi ~/.bash_profile 

# 生效
source ~/.bash_profile

# 验证
hadoop version
```

**修改配置文件**
```shell
cd $HADOOP_HOME/etc/hadoop
```

`vi core-site.xml`编辑内容如下：
```xml
<configuration>
<!-- 配置分布式文件系统的schema和ip以及port,默认8020-->
<property>
<name>fs.defaultFS</name>
<value>hdfs://localhost:8020</value>
</property>
</configuration>
```

`vi hdfs-site.xml`编辑内容如下：
```xml
<configuration>
<!-- 配置副本数，注意，伪分布模式只能是1。-->
<property>
<name>dfs.replication</name>
<value>1</value>
</property>
</configuration>
```

`vi hadoop-env.sh`编辑内容如下：
```xml
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_301.jdk/Contents/Home
```

`vi mapred-site.xml`编辑内容如下：
```xml
<configuration>
<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>

<property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
</configuration>
```

`vi yarn-site.xml`编辑内容如下：
```xml
<configuration>

<!-- Site specific YARN configuration properties -->
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
</configuration>
```

**ssh免密码登录**
```shell
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub>> ~/.ssh/authorized_keys
chmod 0600~/.ssh/authorized_keys
```
以前安装其他软件已操作过，所以此步骤忽略

**格式化namenode**
```shell
hdfs namenode -format
```
忽略`SHUTDOWN_MSG: Shutting down NameNode at localhost/127.0.0.1`

有`INFO common.Storage: Storage directory /tmp/hadoop-llye/dfs/name has been successfully formatted.`即说明操作成功。


**启动**
```shell
$HADOOP_HOME/sbin/start-dfs.sh
$HADOOP_HOME/sbin/start-yarn.sh
```

**验证**

```shell
root@localhost hadoop % jps
11440 
11169 
74514 NameNode # 名称节点
74756 SecondaryNameNode
74999 ResourceManager
75098 NodeManager
75834 Jps
19771 Launcher
74620 DataNode  # 数据节点
```
**访问UI：ip+port**

All Applications：`http://localhost:8088/cluster/apps`

Applications running on this node：`http://localhost:8042/node/allApplications`
![JCU0J.png](https://i.328888.xyz/2023/03/15/JCU0J.png)

Browse Hdfs：`http://localhost:9870/`
![J0HkZ.png](https://i.328888.xyz/2023/03/15/J0HkZ.png)

这里需要注意的是，因为安装的是3.x版本，所以端口号为9870

若安装的是2.x版本，则端口号为50070

![J0ehd.png](https://i.328888.xyz/2023/03/15/J0ehd.png)


**测试程序**

* 测试一
```shell
cd $HADOOP_HOME
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar pi 2 100
```

* 测试二
```shell
cd $HADOOP_HOME

# 创建一个hdfs目录
hdfs dfs -mkdir /wordcount

# 造数据
# hello hadoop
# hello world
# hello hadoop
# hello hangzhou
# hello hangzhou
# hello hadoop
mkdir wordCount
cd wordCount
touch wc.input
vi wc.input
cat wc.input

# 上传本地文件到指定目录
hdfs dfs -put wc.input /wordcount

# 运行mr程序
cd $HADOOP_HOME
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar wordcount /wordcount/ /wordcount/output

# 查看mr计算结果
hadoop fs -cat /wordcount/output/part-r-00000
```

**停止**
```shell
$HADOOP_HOME/sbin/stop-dfs.sh
$HADOOP_HOME/sbin/stop-yarn.sh
```