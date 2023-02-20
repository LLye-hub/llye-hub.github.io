---
title: hiveSQL之set参数
toc: true
tags:
categories:
  - SQL
abbrlink: 4547a6e2
date: 2023-02-03 16:14:57
---
# [官方传送门](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties)

## hive.merge.mapfiles
Default Value: true
map-only任务结束时合并小文件

## hive.merge.mapredfiles
Default Value: true
map-reduce任务结束时合并小文件


## hive.optimize.cte.materialize.threshold
默认情况下是-1（关闭）；当开启（大于0），比如设置为2，则如果with..as语句被引用2次及以上时，会把with..as语句生成的table物化，从而做到with..as语句只执行一次，来提高效率