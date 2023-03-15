---
title: hive本机安装
tags:
  - hive安装
categories:
  - hive
toc: true
abbrlink: 47d5b7b0
date: 2023-03-15 10:49:23
---
**参考文章**

[Hive源码系列（一）hive2.1.1+hadoop2.7.3环境搭建](https://zhuanlan.zhihu.com/p/68748400)

[Hive安装超详细教程](https://juejin.cn/post/7114252763621490719)

**下载安装**

[下载地址](https://dlcdn.apache.org/hive/)

* 解压安装包
```shell
tar -zxvf apache-hive-2.3.9-bin.tar.gz
```

**环境变量配置**

```shell
# 编辑
# HIVE_HOME=/Users/llye/soft/hive-2.3.9
# export PATH=$HIVE_HOME/bin:$PATH
vi ~/.bash_profile 

# 生效
source ~/.bash_profile

# 验证
hive --version
```

**修改配置文件**
```shell
cd $HADOOP_HOME/conf
```

`vi hive-site.xml`编辑内容如下：
```xml
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://localhost:3306/hivedb?createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8&amp;useSSL=false&amp;serverTimezone=GMT</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.cj.jdbc.Driver</value>
    </property>

    <!-- Mysql登录账号 -->
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <!-- Mysql登录密码 -->
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>12345678</value>
    </property>

    <!-- 忽略HIVE 元数据库版本的校验，如果非要校验就得进入MYSQL升级版本 -->
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>

    <property>
        <name>hive.cli.print.current.db</name>
        <value>true</value>
    </property>

    <property>
        <name>hive.cli.print.header</name>
        <value>true</value>
    </property>

    <!-- hiveserver2 -->
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
    </property>

    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>hadoop1</value>
    </property>
</configuration>
```


**下载连接MySQL的驱动包到hive的lib目录下**

`mysql-connector-java-8.0.17.jar`[下载地址](https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.17/mysql-connector-java-8.0.17.jar)


**初始化hive元数据库**
```shell
cd $HIVE_HOME/bin
schematool -initSchema -dbType mysql -verbose
```
 


**验证初始化是否成功**
```sql
-- mysql的hivedb库中，若展示多个数据表，即代表初始化成功
show tables;
```
![JxW5E.png](https://i.328888.xyz/2023/03/15/JxW5E.png)


**启动hive**
```shell
$HADOOP_HOME/sbin/start-dfs.sh
$HADOOP_HOME/sbin/start-yarn.sh
cd $HIVE_HOME 
hive 
```
遇到启动报错`org.apache.hadoop.hdfs.server.namenode.SafeModeException): Cannot create directory /tmp/hive`时，执行命令`hdfs dfsadmin -safemode leave`关闭HDFS安全模式

**验证**
```sql
-- 建表
create table student(id int, name string);
-- 插入数据
insert into table student values(1, 'abc');
-- 查询数据
select * from student;
```
![J73cF.png](https://i.328888.xyz/2023/03/15/J73cF.png)



