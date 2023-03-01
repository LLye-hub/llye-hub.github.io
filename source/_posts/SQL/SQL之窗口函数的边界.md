---
title: SQL之窗口函数的边界
tags:
  - 窗口函数
categories:
  - SQL
toc: true
abbrlink: 5af52219
date: 2023-02-28 16:25:40
---
# 前言
窗口函数常用于在SQL数据分析计算各种统计指标，也是日常sql开发中常见的函数了，但是最近发现自己在这方面存在一些误解

比如下面这段sql
```sql
select col1
    ,sum(col2) over(partition by col1 order by col3) as sum1
    ,sum(col2) over(partition by col1) as sum2
from (select * from (VALUES('a',1,4),('a',2,7),('a',3,6)) t(col1,col2,col3)) a
```
第一眼感觉`sum1`和`sum2`字段计算值是一样的，但实际运行出来的结果为(`mysql`+`hiveSQL`)

| col1 | sum1 | sum2 |
| --- | --- | --- |
| a | 1 | 6 |
| a | 4 | 6 |
| a | 6 | 6 |

从执行结果上来看，`sum1`字段为窗口内的累加值，`sum2`字段值为窗口内所有值之和

# 为什么有无order by差异这么大
有人会说，聚合函数`sum()`的窗口内有`order by`子句时，计算结果本就是累加性质。从执行结果上来看，这么说是对的，但是这种解释太流于表面，并没有真正从函数定义上解释为什么

这里重新回顾下窗口函数基本语法：
```sql
<window_function> over (partition by <column_name> order by <column_name> <window_frame>)
```
主要有四个部分：
* window_function：函数，比如：sum、row_number、first_value
* partition by：窗口分区子句
* order by：窗口排序子句
* window_frame：窗口框架，限制窗口的边界大小

对照基本语法，有`order by`子句的执行结果就是计算窗口边界为起始行至当前行的`sum`结果，
即`sum(col2) over(partition by col1 order by col3)`
等同于`sum(col2) over(partition by col1 order by col3 rows between unbounded preceding and current row)` 

# 窗口函数的官方说明
[mysql官方文档](https://dev.mysql.com/doc/refman/8.0/en/window-functions-frames.html) 

[hive官方文档](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics#LanguageManualWindowingAndAnalytics-PARTITIONBYwithonepartitioningcolumn,noORDERBYorwindowspecification)

`mysql`关于窗口函数的window_frame有如下说明：
![6DIOt.png](https://i.328888.xyz/2023/03/01/6DIOt.png)

`hive`关于窗口函数的window_frame有如下说明：
![6DNeJ.png](https://i.328888.xyz/2023/03/01/6DNeJ.png)

根据以上官方说明可知，
当`window_frame`子句和`order by`子句都没有时，窗口计算默认包含窗口内的所有数据，即`window_frame=ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`；
当仅有`order by`子句，没有`window_frame`子句时，窗口计算默认仅包含排序后起始行至当前行的数据，即`window_frame=ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`


