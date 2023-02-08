---
title: hiveSQL之生成连续数字
toc: true
abbrlink: 3ce3d37f
date: 2023-02-08 16:03:36
tags:
 - SQL高级语法
categories:
 - SQL
---

# sql要求
生成100以内的全部整数

# 涉及udtf函数
**posexplode**(ARRAY\<T> a) [官方说明](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF?spm=a2c4g.11186623.0.0.3c267254Ka3fUh#LanguageManualUDF-posexplode(array))	
**Return**: Returns a row-set with two columns (pos int,val T), one row for each element from the array.	
**Description**: posexplode() is similar to explode but instead of just returning the elements of the array it returns the element as well as its position in the original array.

**用法示例**：
有如下一张表myTable
| (array\<int>)myCol |  
| :-: |   
| [100,200,300] | 
| [400,500,600] | 

执行hive sql
```sql
-- 造数据
with myTable as (
    select array(100,200,300) as myCol
    union all 
    select array(300,400,500) as myCol
)
-- 查询sql
SELECT posexplode(myCol) AS (pos, val) FROM myTable
```
得到结果为：
| (int)pos | (int)val |  
| :-: | :-: |   
| 0 | 100 | 
| 1 | 200 | 
| 2 | 300 | 
| 0 | 400 | 
| 1 | 500 | 
| 2 | 600 | 

# sql实现
借助posexplode返回的pos即可实现
```sql
select posexplode(split(space(99), ' ')) as (pos, val)
-- 返回的pos字段即为[0,99]区间的100个整数

-- 或者下面这种写法
select posexplode(split(repeat(',',99), ',')) as (pos, val)
```


# 实例场景
## 数据重复扩容10倍
```sql
-- 造数据
with myTable as (
    select '张三' as name
    union all 
    select '李四' as name
)
-- 将myTable的每行数据重复复制为5行
SELECT name
       ,posexplode(split(space(4), ' ')) AS (pos, val) 
FROM myTable
```
得到结果为：
| name | pos | val |  
| :-: | :-: | :-: |   
| 张三 | 0 |  | 
| 张三 | 1 |  | 
| 张三 | 2 |  | 
| 张三 | 3 |  | 
| 张三 | 4 |  | 
| 李四 | 0 |  | 
| 李四 | 1 |  | 
| 李四 | 2 |  | 
| 李四 | 3 |  | 
| 李四 | 4 |  | 


## 生成指定范围内的连续日期
```sql
with subquery as  (
    select split(space(datediff('2023-1-31','2022-11-30')), ' ')  as x
) 
select 
    date_add('2022-11-30', pos) as new_date
from  
    subquery t
    lateral view 
    posexplode(x) pe as pos, val
```


