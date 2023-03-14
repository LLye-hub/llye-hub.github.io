---
title: 终端免密登录mysql
tags:
  - 免密登录
categories:
  - shell
toc: true
abbrlink: b31f5f52
date: 2023-03-14 15:29:49
---
参考资料：[Mysql Shell免密登录的思考及实际应用案例](https://aijishu.com/a/1060000000088457)

常见终端登录mysql的方式是通过命令`mysql -u{user} -p{password}`，每次登录都需要输入一长串命令和参数，我觉得麻烦，且这种方式下密码直接暴露出来是不安全的

虽然也可用命令`mysql -u{user} -p` + 手动输入密码，也是麻烦的，而且还要记密码

所以，如果仅用命令`mysql`即可实现登录，那得多方便

从网上搜索后发现，可以通过明文配置文件的方式实现mysql免密登录

具体命令如下：
```shell
# 编辑配置文件
# [mysql]
# user=xxx
# password=xxx
sudo vi /etc/my.cnf

# 查看文件内容
cat  /etc/my.cnf

# 指定mysql server仅从这个配置文件读取参数
mysql --defaults-file=/etc/my.cnf

# 验证读取配置情况
mysql --defaults-file=/etc/my.cnf --print-defaults
```
![9EVXa.png](https://i.328888.xyz/2023/03/14/9EVXa.png)
