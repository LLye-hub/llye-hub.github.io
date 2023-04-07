---
title: hiveSQL之全面认识窗口函数
tags:
  - hiveSQL
categories:
  - SQL
toc: true
abbrlink: ed2327bc
date: 2023-03-31 10:46:52
---
本文内容来自文章[Hive SQL大厂必考常用窗口函数及面试题](https://mp.weixin.qq.com/s/VnQT-bidnJDduoLfJeRhxA)

受岗位性质和工作内容影响，在我从事数仓开发工作至今，对于窗口函数的使用场景都很基础，常用的也只有row_number、sum、max/min，
偶尔碰到些其他场景，因为不熟悉，可能就需要反复查看官方文档确认。

所以在上面文章阅读过程中，基于个人理解，重新梳理写了本文

# 窗口函数概述
[hive官方介绍](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics)

窗口函数也称为OLAP函数，是数据分析最常用到的函数，熟练的掌握窗口函数的各种用法和骚操作对从事数据工作者是很重要的。

与聚合函数将多条记录聚合为一条不同，窗口函数每条记录都会执行，执行前后数据量不变，且窗口函数兼具分组和排序两种功能。

## 窗口函数用法

## 基本语法
```xml
<窗口函数> over ([partition by <列名>] [order by <排序列名>] [window_frame])
```
其中：

<窗口函数>: 指需要使用的分析函数，如row_number()、sum()等。

over() : 用来指定函数执行的窗口范围，这个数据窗口大小可能会随着行的变化而变化。
如果括号中什么都不写，则意味着窗口包含满足where条件的所有行，窗口函数基于所有行进行计算

window_frame: 在分组窗口基础上，可以进一步指定窗口计算边界

## 设置窗口

### 1）partition by子句
窗口划分分组条件
```sql
SELECT
    uid,
    score,
    sum(score) OVER(PARTITION BY uid) AS sum_score
FROM exam_record
```

### 2）order by子句
窗口排序条件
```sql
SELECT
    uid,
    score,
    sum(score) OVER(ORDER BY uid) AS sum_score
FROM exam_record
```


### 3）指定窗口大小
指定窗口大小，又称为窗口框架。框架是重新定义窗口计算边界，框架有两种范围限定方式：

* 一种是使用 ROWS 子句，通过指定当前行之前或之后的固定数目的行来限制分区中的行数。

* 另一种是使用 RANGE 子句，按照排列序列的当前值，根据相同值来确定分区中的行数。

语法`ORDER BY 字段名 RANGE|ROWS 边界规则0 | [BETWEEN 边界规则1 AND 边界规则2]`，边界规则的可取值如下：

* `current row`：当前行
* `n preceding`：当前行及往前n行数据
* `unbounded preceding`：第一行至当前行数据
* `n following`：当前行及往后n行数据
* `unbounded following`：当前行至最后一行数据

需要注意的是，
* 使用框架时必须有order by子句
* 若仅有order by子句而未指定框架，则默认框架语句为`range unbounded preceding and current row`，
  [详情见文章](https://llye-hub.github.io/posts/5af52219.html)

### 4）window_name
给窗口指定一个别名`WINDOW my_window_name AS (PARTITION BY uid ORDER BY score)`，
适用于一个窗口被多次使用，可以使sql简洁清晰，也易于维护
```sql
SELECT
    uid,
    score,
    rank() OVER my_window_name AS rk_num,
    row_number() OVER my_window_name AS row_num,
    dense_rank() OVER my_window_name AS dr_num
FROM exam_record
WHERE score>=60
ORDER BY uid
WINDOW my_window_name AS (PARTITION BY uid ORDER BY score)
```

## 窗口函数分类

窗口函数：
* first_value: 返回计算窗口内按排序条件的第一个值，语法`first_value(exp_str,true|false)`
* last_value: 返回计算窗口内按排序条件的最后一个值，语法`last_value(exp_str,true|false)`
* lag: 返回相对当前行，第前n行的数据，语法`lag(exp_str,offset,defval) over(partition by .. order by …)`
* lead: 返回相对当前行，第后n行的数据，语法`lead(exp_str,offset,defval) over(partition by .. order by …)`

配合over语句使用的聚合函数：
* sum
* count([distinct])
* max
* min
* avg

分析函数：
* row_number: 连续排序——1、2、3、4
* rank: 并列跳号排序——1、1、3、4
* dense_rank: 并列连续排序——1、1、2、3
* percent_rank: 将某个数值在数据集中的rank()排位作为数据集的百分比值返回，每行按照公式(rank-1) / (rows-1)进行计算，百分比值的范围为 0 到 1。
  可用于计算值在数据集内的相对位置。语法`percent_rank(exp_str)`
* cume_dist: 如果按升序排列，则统计：小于等于当前值的行数/总行数。
             如果是降序排列，则统计：大于等于当前值的行数/总行数。
             语法`cume_dist(exp_str)`
* ntiles: 将分组数据按照顺序平均切分成n组，并返回当前切片值。语法`ntiles(n)`。
          如果不能平均分配，则优先分配较小编号的切片，并且各个切片中能放的行数最多相差 1。
          可简单理解为，有 n 个桶，按编号 1-n 的顺序逐个将分组数据放到每个桶内，直至数据分配完毕。








