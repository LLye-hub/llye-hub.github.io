---
title: mysql和hiveSQL的语法差别
toc: true
abbrlink: e912f2da
date: 2023-02-17 16:06:16
tags:
categories:
- SQL
---
最近在牛客网上刷sql题，但编程语言居然只支持mysql，一些函数用法上与平时工作使用的hiveSQL有较大差别，所以在这篇博客中整理一下两种语法的函数使用差异

[mysql内置函数](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html)

[hive内置函数](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF?spm=a2c4g.11186623.0.0.3c267254Ka3fUh#LanguageManualUDF-get_json_object)

# 日期、时间函数
| 函数用途 | mysql函数 | mysql用法 | hive函数 | hiveSQL用法 | 
| --- | --- | --- | --- | --- |
| 日期、时间格式化 | date_format | date_format('2008-08-08 22:23:01', '%Y%m%d%H%i%s') | date_format | date_format('2008-08-08 22:23:01', 'yyyyMMddHHmmss') | 
| 日期、时间加 | date_add | date_add('2008-08-08 22:23:01',interval 1 day/hour/minute/second/microsecond/week/month/quarter/year)，返回dateTime格式 | date_add | date_add('2008-08-08 22:23:01',1)，只加days，返回date格式 | 
| 日期、时间减 | date_sub | date_sub('2008-08-08 22:23:01',interval 1 day/hour/minute/second/microsecond/week/month/quarter/year)，返回dateTime格式 | date_sub | date_sub('2008-08-08 22:23:01',1)，只加days，返回date格式 | 
| 日期相差 | datediff | datediff('2008-08-08 22:22:00','2008-08-07 22:23:00') | datediff | datediff('2008-08-08 22:22:00','2008-08-07 22:23:00') | 








