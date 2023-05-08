---
title: hiveSQL之理解explain参数
tags:
  - hiveSQL
categories:
  - hive
toc: true
abbrlink: 2369b6cf
date: 2023-05-08 11:16:49
---

关键字EXPLAIN
使用方法：EXPLAIN SELECT ```````(SQL语句)

解释：
MapReduce:表示当前任务执行所用的计算机引擎是MapReduce

Map Operator Tree:表示当前描述的Map阶段执行的操作信息。

Reduce Operator Tree:表示当前描述的Reduce阶段的操作信息。

MAP:

TableScan:表示对关键字alias声明的结果集。这里代指表明。

Statistics:表示对当前阶段的统计信息。例如数据行数和数据量，这两个都是预估值。

Filter Operator:表示在之前操作（TableScan）的结果集上进行数据的过滤。

Predicate:表示Filter Operator进行过滤时候使用的谓词，例如 pt_dt = 2020-11-11

Select Operator:表示在之前的结果集上对列进行投影，即列筛选。

Expressions:表示需要投影的列，即筛选的列。

OutputColNames:表示输出的列名。

Group By Operator:表示在之前的结果集上分组聚合。

Aggreations:表示分组聚合使用的算法.例如 count（）。

keys:表示分组的列。

Reduce Output Operator :表示当前描述的是对之前的结果聚合后的输出信息，这里表示Map端聚合后的信息。

key expressions/value expressions:MapReduce计算引擎，在Map阶段和Reduce阶段输出的都是键-值对的形式，这里key expression和 key expression 和 value expression 分别描述的就是Map阶段输出的键（key） 和值（value）所用的数据列。key expression指代的就是聚合列。 value expression 指代的就是 聚合的函数。

sort order：表示输出是否进行排序，+表示正序，-表示倒序。

Map-Reduce partition columns:表示Map阶段输出到Reduce阶段的分区列。在HIVE-SQL中可以用distribute by 指代分区的列。

Reduce 阶段关键字和Map阶段的含义一样，不同的如下：

compressed:在 File output operator中这个关键词表示文件输出的结果是否进行压缩，FALSE表示不进行输出压缩。

table:表示正在操作的表。

input format/out putformat:分别表示文件输入和输出的文件类型。

serde:表示读取表数据的序列化和反序列化的方式。

Explain extended， explain的扩展，展示更加详细的信息，

抽象语法树（AST）：是SQL转换成Map Reduce或其他计算引擎的任务中的一个过程，在HIVE3.0版本中，AST会从Explain extended 移除，要查看AST需要使用Explain ast.

作业的依赖关系图，即STAGE DEPENDENCIES其内容和explain所展现的一样。

每个作业的详细信息，即STAGE PLAINS。在打印每个作业的详细信息时，explain extend会打印更多的信息，除了explain打印出的内容，还包括每个标的HDFS读取路径，每个HIVE表的配置信息等。

很重要可以查看出表是否被全表扫描：

explain dependency用于描述一段sql需要的数据来源，输出的是一个json格式的数据，里面包含一下两个部分的内容，

input_partitions：描述一段sql依赖的数据来源表分区，里面存储的分区名的列表，格式如下：

{“partitionname”:“库名@表名 @分区列=分区列的值”} ，如果整段sql包含的所有表都是是非分区表，则显示为空。

input_table：描述一段sql依赖的数据来源表，里面存储的是HIVE表名的列表格式如下：

{“tablename”:“库名@表名 ”，“tabletype”：表的类型（外部表/内部表）}

