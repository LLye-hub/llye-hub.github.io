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

[Hive架构与源码分析](https://www.cnblogs.com/swordfall/p/13426569.html#auto_id_14)

# 下载安装

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
cd $HIVE_HOME/conf
```

`vi hive-site.xml`编辑内容如下：
```xml
<configuration>
    <!--  以mysql作为hive元数据库  -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://localhost:3306/hivedb?createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8&amp;useSSL=false&amp;serverTimezone=GMT</value>
        <description>hive metastore连接串</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.cj.jdbc.Driver</value>
        <description>Hive metastore JDBC驱动</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
        <description>Mysql登录账号</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>rootroot</value>
        <description>Mysql登录密码</description>
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

    <!--  配置hive用户名、密码  -->
    <property>
        <name>hive.jdbc_passwd.auth.root</name>
        <value>rootroot</value>
    </property>
    <property>
        <name>hive.jdbc_passwd.auth.llye</name>
        <value>rootroot</value>
    </property>

    <!-- hiveserver2 -->
    <!--  配置用户安全认证方式  -->
    <property>
        <name>hive.server2.authentication</name>
        <value>NONE</value>
        <description>
            Expects one of [nosasl, none, ldap, kerberos, pam, custom].
            Client authentication types.
            NONE: no authentication check
            LDAP: LDAP/AD based authentication
            KERBEROS: Kerberos/GSSAPI authentication
            CUSTOM: Custom authentication provider
            (Use with property hive.server2.custom.authentication.class)
            PAM: Pluggable authentication module
            NOSASL:  Raw transport
        </description>
    </property>
    <property>
        <name>hive.server2.custom.authentication.class</name>
        <value>org.apache.hadoop.hive.contrib.auth.CustomPasswdAuthenticator</value>
        <description>配置用于权限认证的类【这里实际没有】</description>
    </property>
    <!-- 指定 hiveserver2 jdbc连接的 host+port -->
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>localhost</value>
        <description>hiveserver2 jdbc连接的 host</description>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
        <description>hiveserver2 jdbc连接的端口号</description>
    </property>
    <!--  配置webUI界面 host+port  -->
    <property>
        <name>hive.server2.webui.host</name>
        <value>localhost</value>
    </property>
    <property>
        <name>hive.server2.webui.port</name>
        <value>10002</value>
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
$HADOOP_HOME/sbin/start-dfs.sh &
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


# beeline连接hiveserver2

**hadoop配置**
```shell
cd $HADOOP_HOME/etc/hadoop
```

`vi core-site.xml`补充内容如下：
```xml
    <property>
      <name>hadoop.proxyuser.root.groups</name>
      <value>*</value>
    </property>
    <property>
      <name>hadoop.proxyuser.root.hosts</name>
      <value>*</value>
    </property>
    <property>
      <name>hadoop.proxyuser.llye.groups</name>
      <value>*</value>
    </property>
    <property>
      <name>hadoop.proxyuser.llye.hosts</name>
      <value>*</value>
    </property>
```

**重启hadoop**
```shell
$HADOOP_HOME/sbin/stop-all.sh &
$HADOOP_HOME/sbin/start-all.sh 
```

**启动metastore**
配置了hive的环境变量，任意文件夹下执行即可
```shell
hive --service metastore
```

**启动hiveserver2**
配置了hive的环境变量，任意文件夹下执行即可
```shell
hiveservice2
# 或
hive --service hiveservice2
```

**beeline连接**
```shell
beeline
beeline> !connect jdbc:hive2://localhost:10000/default
# 或
beeline -u jdbc:hive2://localhost:10000/default 
```


遇到报错问题的参考：
[beeline连接hiveserver2报错：User: root is not allowed to impersonate root](https://blog.csdn.net/qq_16633405/article/details/82190440)
[Hive JDBC：Permission denied: user=anonymous, access=EXECUTE, inode=”/tmp”](https://developer.aliyun.com/article/606803)

# 客户端jdbc连接hive库
**启动metastore和hiveserver2**
```shell
hive --service metastore

hiveservice2
# 或
hive --service hiveservice2
```

`DBeaver`连接，设置jdbc URL：`jdbc:hive2://localhost:10000/default`
