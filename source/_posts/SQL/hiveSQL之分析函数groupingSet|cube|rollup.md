---
title: hiveSQL之groupBy语句增强语法grouping sets/CUBE/rollup
tags:
  - grouping sets
categories:
  - SQL
toc: true
abbrlink: 67bc5d15
date: 2023-05-05 14:01:45
---
本文详细整理了关于group by子句的增强聚合语法grouping sets/CUBE/rollup的具体用法，语法的[hive官方介绍文档](https://cwiki.apache.org/confluence/display/Hive/Enhanced+Aggregation%2C+Cube%2C+Grouping+and+Rollup) 。

# 测试数据

首先声明使用的hive版本为 `2.3.9`

```sql
drop table if exists travel_data;
create table if not exists travel_data(
      province string,
      city string,
      attraction string,
      star_level int,
      Price double,
      sales int,
      sale_date string
);

insert overwrite table travel_data
select '河南省','郑州市','方特',4,312.22,15789,'2019-02-03'
union all
select '河南省','郑州市','二七广场',4,0,5942,'2019-02-03'
union all
select '河南省','郑州市','河南省博物馆',4,1.22,943,'2019-02-03'
union all
select '河南省','洛阳市','白云山',4,324.44,16843,'2019-02-03'
union all
select '河南省','洛阳市','白马寺',4,23.45,2567,'2019-02-03'
union all
select '河南省','洛阳市','龙门石窟',4,45,15784,'2019-02-03'
union all
select '广东省','深圳市','东部华侨城',4,86,9523,'2019-02-03'
union all
select '广东省','深圳市','欢乐谷',4,54,2573,'2019-02-03'
union all
select '广东省','深圳市','世界之窗',4,34,5644,'2019-02-03'
union all
select '广东省','广州市','长隆',4,46,25673,'2019-02-03'
union all
select '广东省','广州市','广州塔',4,35,9735,'2019-02-03'
;
```

# grouping set语句

官方说明

> The GROUPING SETS clause in GROUP BY allows us to specify more than one GROUP BY option in the same record set. All GROUPING SET clauses can be logically expressed in terms of several GROUP BY queries connected by UNION. Table-1 shows several such equivalent statements. This is helpful in forming the idea of the GROUPING SETS clause. A blank set ( ) in the GROUPING SETS clause calculates the overall aggregate.

grouping set子句可以实现对同一个数据集指定多个group by条件，适合多维聚合场景下使用。其执行效果等同于对多个group by查询进行union all操作。

`SELECT a, b, SUM(c) FROM tab1 GROUP BY a, b GROUPING SETS ( (a,b) )`等同下面语句

> SELECT a, b, SUM(c) FROM tab1 GROUP BY a, b

`SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS ( (a, b), a, b, ( ) )`等同下面语句

> SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b
> UNION
> SELECT a, null, SUM( c ) FROM tab1 GROUP BY a, null
> UNION
> SELECT null, b, SUM( c ) FROM tab1 GROUP BY null, b
> UNION
> SELECT null, null, SUM( c ) FROM tab1

**语法**

1. grouping sets子句必须跟在group by语句后，且出现在grouping sets的字段必须出现在group by语句中，但是出现在group by中字段不一定要出现在grouping sets语句中
2. 出现在group by中但是没有在grouping sets中的字段将会被赋值为null
3. grouping__id字段可以区分不同的聚合粒度，表示当前行数据数据哪个分组集合
4. grouping函数可以处理空值，grouping()接受一个列名作为参数，如果结果对应行使用了参数列做聚合，返回0，此时意味着NULL来自输入数据；否则返回1，此时意味着NULL是grouping sets的占位符。

**测试sql：从省&市聚合维度统计销售数量**

```sql
select
    td.province,
    td.city,
    IF(grouping(td.city) = 0,td.city,'城市') as city2, -- 进行空值判断，替换输出更有实际意义的值
    sum(sales) as sales,
    grouping__id
from
    travel_data td
group by
    td.province,
    td.city
    grouping SETS (td.province,td.city)
order by grouping__id
;
```

查询结果：

![iTpX0x.png](https://i.328888.xyz/2023/05/05/iTpX0x.png)

* 第一列按照province
* 第二列按照city
* 第三列按照city分组，并对空值进行替换
* 第四列按照province或city分组，进行统计计算
* 第五列grouping__id表示当前行数据属于哪个分组，1表示province，2表示city

**测试sql：从省&市、省&日期、省三个聚合维度统计销售数量**

```sql
select
	td.province ,
	td.city,
	sum(sales) as sales,
	td.sale_date ,
	grouping__id
from
	travel_data td
group by
	td.province ,
	td.city,
	td.sale_date
grouping SETS (
	(td.province , td.city)
	,(td.province, td.sale_date)
	,td.province 
	)
order by
	grouping__id
```

# CUBE语句

CUBE函数跟group by语句一起使用，可以对group by的所有字段进行组合再进行聚合计算。

`group by a,b,c with CUBE`执行效果等同于 `group by a, b, c grouping sets ( (a, b, c), (a, b), (b, c), (a, c), (a), (b), (c), ( ))`

**测试sql**

```sql
select
    province ,
    city,
    sum(sales) as sales,
    grouping__id
from
    travel_data
group by
    province,city
with CUBE
order by
    grouping__id
;

-- 或者下面这种写法
select
    province ,
    city,
    sum(sales) as sales,
    grouping__id
from
    travel_data
group by
    CUBE(province,city)
order by
    grouping__id
;
```

查询结果：

![iatksp.png](https://i.328888.xyz/2023/05/06/iatksp.png)

从上面的结果数据可以看到，对聚合字段 `(province,city)`使用CUBE函数后，返回结果有4种聚合维度：`(province,city)`、`(province)`、`(city)`、`()`

# rollup语句

rollup是CUBE的子集，以最左侧的维度为主，从该维度进行层级聚合，可以实现上钻和下钻的效果

`group by a,b,c with rollup`假设层次结构是 "a "向下钻到 "b "向下钻到 "c"，执行效果等同于 `group by a, b, c grouping sets ( (a, b, c), (a, b), (a), ( ))`

**测试sql**

```sql
select
    province ,
    city,
    sale_date ,
    sum(sales) as sales,
    grouping__id
from
    travel_data
group by
    province,city,sale_date
with rollup
order by
    grouping__id

-- 或者下面这种写法
select
	province ,
	city,
	sale_date ,
	sum(sales) as sales,
	grouping__id
from
	travel_data 
group by
	rollup(province,city,sale_date)
order by
	grouping__id
```

查询结果：

![iakvoL.png](https://i.328888.xyz/2023/05/06/iakvoL.png)

从上面的结果数据可以看到，对聚合字段 `(province,city,sale_date)`使用rollup函数后，返回结果有4种聚合维度：`(province,city,sale_date)`、`(province,city)`、`(province)`、`()`

# grouping__id计算方法

从rollup函数的例子可以看到，grouping__id的数值并不是连续的，下面总结下grouping__id计算方法

1. 按group by语句的字段顺序（不理解网上有说法是按字段倒序排序）。所以这里要注意groupby字段顺序变化是会影响grouping__id计算结果的。
2. 对于每个字段，若出现在了当前粒度中，则该字段位置赋值为0，否则为1。
3. 这样就形成了一个二进制数，将这个二进制数转为十进制，即为当前粒度对应的 grouping__id。

以统计粒度 `group by province,city,sale_date`为例，

* 字段顺序为:province,city,sale_date
* 所有聚合维度对应的二进制数为：


|      grouping sets      | 按字段顺序赋值二进制数 | 转换为十进制的grouping__id |
| :-----------------------: | :----------------------: | :--------------------------: |
| province,city,sale_date |          000          |             0             |
|      province,city      |          001          |             1             |
|   province,sale_date   |          010          |             2             |
|        province        |          011          |             3             |
|     city,sale_date     |          100          |             4             |
|          city          |          101          |             5             |
|        sale_date        |          110          |             6             |
|           无           |          111          |             7             |

**测试sql：**

```sql
select
	td.province ,
	city,
	td.sale_date ,
	sum(sales) as sales,
	grouping__id
from
	travel_data td
group by
	CUBE(td.province,td.city,td.sale_date)
order by
	grouping__id
```

查询结果：

![iT55Jb.png](https://i.328888.xyz/2023/05/05/iT55Jb.png)

从上面的结果可以看到，grouping__id的数值与计算规则得出来的一致。hive2.3版本前关于grouping__id的计算方式可能不同，可以参见[其他博客](https://blog.csdn.net/Dax1n/article/details/104308886)

# grouping sets和union all性能对比

**实现逻辑**
如果说 `union all`是先聚合再联合，那么 `grouping sets`就是先联合再聚合。`grouping sets`根据 `N`个分组对每条数据进行计算，不在当前分组的字段置为null，将数据量扩展成原来的 `N`倍，再按 `group by`的字段做聚合计算。

`group by province,city grouping sets ((province,city),province,())`计算效果图如下：

![iacyut.png](https://i.328888.xyz/2023/05/06/iacyut.png)

`…… group by province,city union all …… group by province`计算效果图如下：

![iaRuvE.png](https://i.328888.xyz/2023/05/06/iaRuvE.png)

分析源码，grouping__id在process的时候将newKeysGroupingSets的值赋予具体的行

```java
// org/apache/hadoop/hive/ql/exec/GroupByOperator.java

public void process(Object row, int tag) throws HiveException {
	……
    if (groupingSetsPresent) {
    Object[] newKeysArray = newKeys.getKeyArray();
    Object[] cloneNewKeysArray = new Object[newKeysArray.length];
    for (int keyPos = 0; keyPos < groupingSetsPosition; keyPos++) {
    cloneNewKeysArray[keyPos] = newKeysArray[keyPos];
    }

    for (int groupingSetPos = 0; groupingSetPos < groupingSets.size(); groupingSetPos++) {
    for (int keyPos = 0; keyPos < groupingSetsPosition; keyPos++) {
    newKeysArray[keyPos] = null;
    }

    FastBitSet bitset = groupingSetsBitSet[groupingSetPos];
    // Some keys need to be left to null corresponding to that grouping set.
    // 按照bitSet保留原值，对于group by a, b, c 如果bitSet是010，则表示keyPos为0和2就表示ClearBit，需要保留原值，1其他就为null
    for (int keyPos = bitset.nextClearBit(0); keyPos < groupingSetsPosition;
          keyPos = bitset.nextClearBit(keyPos+1)) {
          newKeysArray[keyPos] = cloneNewKeysArray[keyPos];
      }
    // 这里就是给当前这条数据赋予GROUPING_ID的值
    newKeysArray[groupingSetsPosition] = newKeysGroupingSets[groupingSetPos];
    processKey(row, rowInspector);
    }
    } else {
    processKey(row, rowInspector);
    }  
    }
```

下面通过执行计划分析两种方式的差异。

```sql
explain select
    province,
    city,
    grouping__id
from
    travel_data 
group by
    province,
    city
    grouping SETS ((province,city),province,())
order by grouping__id
;
```

hive执行计划：

```text
Explain                                                                                                      |
-------------------------------------------------------------------------------------------------------------+
STAGE DEPENDENCIES:                                                                                          |
  Stage-1 is a root stage                                                                                    |
  Stage-2 depends on stages: Stage-1                                                                         |
  Stage-0 depends on stages: Stage-2                                                                         |
                                                                                                             |
STAGE PLANS:                                                                                                 |
  Stage: Stage-1                                                                                             |
    Map Reduce                                                                                               |
      Map Operator Tree:                                                                                     |
          TableScan                                                                                          |
            alias: travel_data                                                                               |
            Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE                 |
            Select Operator                                                                                  |
              expressions: province (type: string), city (type: string)                                      |
              outputColumnNames: _col0, _col1                                                                |
              Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE               |
              Group By Operator                                                                              |
                keys: _col0 (type: string), _col1 (type: string), 0 (type: int)                              |
                mode: hash                                                                                   |
                outputColumnNames: _col0, _col1, _col2                                                       |
                Statistics: Num rows: 33 Data size: 1791 Basic stats: COMPLETE Column stats: NONE            |  这里读取数据后按grouping sets的3个分组维度，将数据由11条扩充为33条
                Reduce Output Operator                                                                       |
                 ……
      Reduce Operator Tree:                                                                                  |
        Group By Operator                                                                                    |
          keys: KEY._col0 (type: string), KEY._col1 (type: string), KEY._col2 (type: int)                    |
          mode: mergepartial                                                                                 |
          outputColumnNames: _col0, _col1, _col2                                                             |
          Statistics: Num rows: 16 Data size: 868 Basic stats: COMPLETE Column stats: NONE                   |
          Select Operator                                                                                    |
            expressions: _col0 (type: string), _col1 (type: string), _col2 (type: int)                       |
            outputColumnNames: _col0, _col1, _col2                                                           |
            Statistics: Num rows: 16 Data size: 868 Basic stats: COMPLETE Column stats: NONE                 |
            ……
                                                                                                             |
  Stage: Stage-2                                                                                             |
    Map Reduce                                                                                               |
      Map Operator Tree:                                                                                     |
          TableScan                                                                                          |
            Reduce Output Operator                                                                           |
              key expressions: _col2 (type: int)                                                             |
              sort order: +                                                                                  |
              Statistics: Num rows: 16 Data size: 868 Basic stats: COMPLETE Column stats: NONE               |
              value expressions: _col0 (type: string), _col1 (type: string)                                  |
      Reduce Operator Tree:                                                                                  |
        Select Operator                                                                                      |
          expressions: VALUE._col0 (type: string), VALUE._col1 (type: string), KEY.reducesinkkey0 (type: int)|  这里KEY.reducesinkkey0即为grouping__id
          outputColumnNames: _col0, _col1, _col2                                                             |
          Statistics: Num rows: 16 Data size: 868 Basic stats: COMPLETE Column stats: NONE                   |
          ……                                                                                        |
 
                                 |
                                                                                                             |
  Stage: Stage-0                                                                                             |
    ……                                                                                   
```

从上面的执行计划可以看到，`Stage-1`在读取数据时，在`map`阶段根据`grouping sets`有3个分组维度，将数据量扩充至原来的3倍，然后在`reduce`阶段做`group by province,city`操作。

```sql
explain
select
	province,
	city
from
	travel_data
group by
    province,city
union all
select
    province,
    NULL as city
from
	travel_data
group by
    province 
;
```

hive执行计划：

```text
Explain                                                                                           |
--------------------------------------------------------------------------------------------------+
STAGE DEPENDENCIES:                                                                               |
  Stage-1 is a root stage                                                                         |
  Stage-2 depends on stages: Stage-1, Stage-3                                                     |
  Stage-3 is a root stage                                                                         |
  Stage-0 depends on stages: Stage-2                                                              |
                                                                                                  |
STAGE PLANS:                                                                                      |
  Stage: Stage-1                                                                                  |
    Map Reduce                                                                                    |
      Map Operator Tree:                                                                          |
          TableScan                                                                               |
            alias: travel_data                                                                    |
            Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE      |
            Select Operator                                                                       |
              expressions: province (type: string), city (type: string)                           |
              outputColumnNames: province, city                                                   |
              Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE    |
              Group By Operator                                                                   |
                keys: province (type: string), city (type: string)                                |
                mode: hash                                                                        |
                outputColumnNames: _col0, _col1                                                   |
                Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE  |  这里数据量没有变化
                Reduce Output Operator                                                            |
                  ……
      Reduce Operator Tree:                                                                       |
        Group By Operator                                                                         |
          keys: KEY._col0 (type: string), KEY._col1 (type: string)                                |
          mode: mergepartial                                                                      |
          outputColumnNames: _col0, _col1                                                         |
          Statistics: Num rows: 5 Data size: 271 Basic stats: COMPLETE Column stats: NONE         |
          ……
                                                                                                  |
  Stage: Stage-2                                                                                  |
    Map Reduce                                                                                    |
      Map Operator Tree:                                                                          |
          TableScan                                                                               |
            Union                                                                                 |
              Statistics: Num rows: 10 Data size: 542 Basic stats: COMPLETE Column stats: NONE    |
              ……                  |
          TableScan                                                                               |
            Union                                                                                 |
              Statistics: Num rows: 10 Data size: 542 Basic stats: COMPLETE Column stats: NONE    |
              ……               
  Stage: Stage-3                                                                                  |
    Map Reduce                                                                                    |
      Map Operator Tree:                                                                          |
          TableScan                                                                               |
            alias: travel_data                                                                    |
            Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE      |
            Select Operator                                                                       |
              expressions: province (type: string)                                                |
              outputColumnNames: province                                                         |
              Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE    |
              Group By Operator                                                                   |
                keys: province (type: string)                                                     |
                mode: hash                                                                        |
                outputColumnNames: _col0                                                          |
                Statistics: Num rows: 11 Data size: 597 Basic stats: COMPLETE Column stats: NONE  |  这里数据量没有变化
                Reduce Output Operator                                                            |
                  ……
      Reduce Operator Tree:                                                                       |
        Group By Operator                                                                         |
          keys: KEY._col0 (type: string)                                                          |
          mode: mergepartial                                                                      |
          outputColumnNames: _col0                                                                |
          Statistics: Num rows: 5 Data size: 271 Basic stats: COMPLETE Column stats: NONE         |
            ……             
                                                                                                  |
  Stage: Stage-0                                                                                  |
    ……                     
```

从上面的执行计划可以看到，`Stage-1`和`Stage-3`都是读取数据，再分别按照`group by province,city`和`group by province`做聚合操作，最后在`Stage-2`做`union`操作合并数据。`union all`这种写法对表`travel_data`重复读取两次，查询性能上比`grouping sets`写法要差些。在集群空闲的情况下，对两种写法的sql分别执行5次，得到如下结果：

> grouping sets写法执行5次的耗时:
>
>> select province, city, grouping__id from travel_data group by province, city grouping SETS ((province,city),province) order by grouping__id ;
>>
>
> Time taken: 54.807 seconds, Fetched: 6 row(s)
> Time taken: 56.261 seconds, Fetched: 6 row(s)
> Time taken: 52.671 seconds, Fetched: 6 row(s)
> Time taken: 62.945 seconds, Fetched: 6 row(s)
> Time taken: 57.337 seconds, Fetched: 6 row(s)

> union all写法执行5次的耗时:
>
>> select province,city from travel_data group by province,city union all select province, NULL as city from travel_data group by province;
>>
>
> Time taken: 83.91 seconds, Fetched: 6 row(s)
> Time taken: 94.466 seconds, Fetched: 6 row(s)
> Time taken: 86.253 seconds, Fetched: 6 row(s)
> Time taken: 75.509 seconds, Fetched: 6 row(s)
> Time taken: 88.633 seconds, Fetched: 6 row(s)

可以算出，`grouping sets`写法的平均耗时为56.8s，`union all`写法的平均耗时为85.7s，耗时是前者的1.5倍。

所以，`grouping sets`写法的sql不仅在表达上更加简洁，在查询性能上也更加高效。

# 参考文章

[Hive分析函数详解：GROUPING SETS/CUBE/ROLLUP](https://juejin.cn/post/7223211123961200700)

[从源码深入理解 Spark SQL 中的 Grouping Sets 语句](https://zhuanlan.zhihu.com/p/536981356)

[Hive虚拟列的生成与计算【1】](https://zhuanlan.zhihu.com/p/408391394)
