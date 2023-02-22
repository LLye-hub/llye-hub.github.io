---
title: hive、Spark和Maxcompute的SQL语法对比分析
toc: true
abbrlink: e6b1209
date: 2023-02-21 10:33:16
tags:
- Maxcompute
categories:
- SQL
---
# having 差异
#### 差异点
hive和spark支持窗口函数后带having
maxcomputer 的having语法只支持 在 group 和 distinct 后使用

#### 举例
```sql
select order_id
,sum(trd_amt) over(partition by province) as trd_amt_std
from order
having trd_amt_std>0
```

以上sql在hive中可以运行，但是在maxcomputer中会提示错误，错误如下：
![gtHTp.png](https://i.328888.xyz/2023/02/21/gtHTp.png)


#### 替换方案
在语句中使用子查询，将having替换为where
```sql
select *
from
(select order_id
,sum(trd_amt) over(partition by province) as trd_amt_std
from order
) a
where a.trd_amt_std>0
```


# maxcomputer - cross join 超过一定条数后，依然会提示笛卡尔积风险
#### 差异点
hive可以使用 cross join语法来表示笛卡尔积关联
maxcomputer 的cross join，在条数超过一定数据量后，会提示笛卡尔积风险

#### 举例
```sql
select a.*,b.*
from
(select * from table_a
) a
cross join
(select * from table_b
) b
```

以上sql在hive中可以运行，但是在maxcomputer中会提示错误，错误如下：
![xfOrA.png](https://i.328888.xyz/2023/02/22/xfOrA.png)

#### 替换方案
在左右笛卡尔积表中新增常量字段，用于关联
```sql
select a.*,b.*
from
(select *,1 as cro_col from table_a
) a
cross join
(select *,1 as cro_col from table_b
) b
on a.cro_col=b.cro_col
```
# 不等值join 差异
#### 差异点
1、spark 支持不等值join语法
2、hive 2.2.0版本之前不支持不等值语法，[2.2.0及以后支持不等值join语法](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Joins)
![gtrN5.png](https://i.328888.xyz/2023/02/21/gtrN5.png)
3、[maxcomputer不支持不等值语法](https://help.aliyun.com/document_detail/73783.html)
![gt3h8.png](https://i.328888.xyz/2023/02/21/gt3h8.png)

#### 举例
测试sql
```sql
with table_a as (
select 1 as id_a
,'testa' as value_a

    union all
 
    select 4 as id_a
    ,'testd' as value_a
)
,table_b as (
select 3 as id_b
,'testc' as value_b

    union all
 
    select 2 as id_b
    ,'testb' as value_b
)
select table_a.id_a
,table_a.value_a
,table_b.id_b
,table_b.value_b
from  table_a
left join table_b
on table_a.id_a < table_b.id_b
```
sql说明 :该sql准备了两张表table_a和table_b用于连接测试
使用left join on语法，但是关联关系使用的是 < 不等值关联符号

**maxcomputer运行结果**
![xfcBz.png](https://i.328888.xyz/2023/02/22/xfcBz.png)
maxcomputer会报异常：  FAILED: ODPS-0130071:[15,4] Semantic analysis exception - expect equality expression (i.e., only use '=' and 'AND') for join condition without mapjoin hint

提示的是期望join的是等值表达式

**hive1.2.1运行结果**
![gtmHF.png](https://i.328888.xyz/2023/02/21/gtmHF.png)

hive会报错： Error while compiling statement: FAILED: SemanticException [Error 10017]: line 15:3 Both left and right aliases encountered in JOIN 'id_b'

提示的是在join中遇到左右别名

不得不说，hive的错误信息有点云里雾里，其实就是不等值join造成的。

**hive2.2.3运行结果**
![gtpeH.png](https://i.328888.xyz/2023/02/21/gtpeH.png)

hive 2.2.0+版本顺利得到正确结果

**spark运行结果**

![xfGXy.png](https://i.328888.xyz/2023/02/22/xfGXy.png) 

spark2.3也顺利得到结果

#### 替换方案
针对不等值join的替换方案有两种

1、针对小表，使用mapjoin，避免join操作

2、将on的不等值关联语句放入where语句中

由于mapjoin避免shuffle，性能较好，再可以的情况下，优先使用方案1

**1、针对小表，使用mapjoin，避免join操作**
maxcomputer中的mapjoin hint语法为： /*+ mapjoin(<table_name>) */ ，详情请查看[mapjoin hint](https://help.aliyun.com/document_detail/73785.htm?spm=a2c4g.11186623.0.0.534268c4fHc5iD#concept-bf5-tkb-wdb)
```sql
with table_a as (
select 1 as id_a
,'testa' as value_a
)
,table_b as (
select 2 as id_b
,'testb' as value_b
)
select /*+ mapjoin(table_b) */
table_a.id_a
,table_a.value_a
,table_b.id_b
,table_b.value_b
from  table_a
left join table_b
on table_a.id_a<table_b.id_b
```
可以看到，使用mapjoin hint语法后，sql在maxcomputer中运行正确，顺利拿到了预期结果
![gt79P.png](https://i.328888.xyz/2023/02/21/gt79P.png)

**2、将on的不等值关联语句放入where语句中**
inner join 比较简单
```sql
with table_a as (
select 1 as id_a
,'testa' as value_a
,1 as join_col

    union all
 
    select 4 as id_a
    ,'testd' as value_a
    ,1 as join_col
)
,table_b as (
select 2 as id_b
,'testb' as value_b
,1 as join_col

    union all
 
    select 3 as id_b
    ,'testc' as value_b
    ,1 as join_col
)
select
table_a.id_a
,table_a.value_a
,table_b.id_b
,table_b.value_b
from  table_a
inner join table_b
on table_a.join_col=table_b.join_col
where table_a.id_a<table_b.id_b
```
可以看到，将<判断语句放入where后，sql在maxcomputer运行正确，顺利拿到了预期结果
![gteAX.png](https://i.328888.xyz/2023/02/21/gteAX.png)


left join 比较复杂，建议使用map hint，实在没办法在使用此方案
```sql
with table_a as (
select 1 as id_a
,'testa' as value_a
,1 as join_col

    union all
 
    select 4 as id_a
    ,'testd' as value_a
    ,1 as join_col
)
,table_b as (
select 2 as id_b
,'testb' as value_b
,1 as join_col

    union all
 
    select 3 as id_b
    ,'testc' as value_b
    ,1 as join_col
)
-- 能关联上的部分
,join_part as (
select
table_a.id_a
,table_a.value_a
,table_b.id_b
,table_b.value_b
from  table_a
inner join table_b
on table_a.join_col=table_b.join_col
where table_a.id_a<table_b.id_b
)
-- 以自己为主表，left join能关联上的部分，实现 left join不等值效果
select table_a.id_a
,table_a.value_a
,join_part.id_b
,join_part.value_b
from  table_a
left join join_part
on table_a.id_a=join_part.id_a
```
可以看到，将<判断语句放入where后，sql在maxcomputer运行正确，顺利拿到了预期结果
![gt6hJ.png](https://i.328888.xyz/2023/02/21/gt6hJ.png)


# array_contains 差异
#### 差异点
spark的array_contains支持类型的隐式转换
hive和maxcomputer array_contains不支持，只支持同类型使用

#### 举例
测试sql
```sql
select array_contains(split("1,2,3,4",","),1)
```
**sql说明**
该sql首先使用split一个字符串获取一个array对象用于测试，之后使用array_contains函数进行判断
split后的array对象为一个string数组，而判断被包含的数字【1】为一个int 对象

**maxcomputer运行结果**
![xf5D8.png](https://i.328888.xyz/2023/02/22/xf5D8.png)
maxcomputer会报异常：  FAILED: ODPS-0130071:[1,44] Semantic analysis exception - invalid type INT of argument 2 for function array_contains, expect STRING, implicit conversion is not allowed

提示的是array_contains第二个参数期望的是string，但是传入的是int，隐式类型转换不支持

**hive运行结果**
![gtdOA.png](https://i.328888.xyz/2023/02/21/gtdOA.png)

hive会报错： Error while compiling statement: FAILED: SemanticException [Error 10016]: line 1:43 Argument type mismatch '1': "string" expected at function ARRAY_CONTAINS, but "int" is found

提示的是array_contains函数期望的是string，但是传入的是int，类型不匹配

**spark运行结果**

![xf1DJ.png](https://i.328888.xyz/2023/02/22/xf1DJ.png)

spark能顺利产出结果，结果为true，那么为什么spark可以成功呢？

大概率是spark智能的将1从int转换为了string类型，使得类型得以匹配，通过explain查看物理执行计划来验证

![gtJAq.png](https://i.328888.xyz/2023/02/21/gtJAq.png)

在上图标红的地方可以看到，spark在物理执行计划层面，将int的1隐式的转换为了string类型，验证了我们一开始的猜想。

#### 替换方案
既然知道了在hive和maxcomputer中是类型不匹配导致的array_contains函数报错，那么只需要显示的将类型进行转换即可
```sql
select array_contains(split("1,2,3,4",","),cast(1 as string))
```

# 字段类型转换 ARRAY<> to STRING
#### 差异点
spark的array_contains支持类型的隐式转换

hive和maxcomputer array_contains不支持，只支持同类型使用

#### 举例
测试sql
```sql
select cast(array(1, 2, 3, 4) as string) as array_to_string;
```

**maxcompute运行结果**

![xfQ3c.png](https://i.328888.xyz/2023/02/22/xfQ3c.png)

maxcompute报异常：FAILED: ODPS-0130141:[1,8] Illegal implicit type cast - cannot cast from ARRAY<INT> to STRING

提示的是 ARRAY<>类型字段  不能强制转换为 STRING 类型

**hive运行结果**

![xyi6V.png](https://i.328888.xyz/2023/02/22/xyi6V.png)

hive报异常：SQL语义错误: Error while compiling statement: FAILED: ClassCastException org.apache.hadoop.hive.serde2.typeinfo.ListTypeInfo cannot be cast to org.apache.hadoop.hive.serde2.typeinfo.PrimitiveTypeInfo

提示的是不同类型不能强转

**spark运行结果**

![gtPZa.png](https://i.328888.xyz/2023/02/21/gtPZa.png)

spark能顺利产出结果

**替换方案**
使用 [array_join函数](https://help.aliyun.com/document_detail/293597.htm?spm=a2c4g.11186623.0.0.5be36f60Wo6eDb#section-pc4-90e-0rl) 将array的元素拼接成字符串，再在首尾加上 '[ ' 和 ']' 字符可以还原spark上的运行结果
```sql
select concat('[',array_join(array(1, 2, 3, 4),','),']') as array_to_string;
```
![gtTOx.png](https://i.328888.xyz/2023/02/21/gtTOx.png)

📢注意
macxcompute的array_join函数默认会忽略null元素，可在array_join函数中设置 nullreplacement 参数替代NULL元素
![gtazk.png](https://i.328888.xyz/2023/02/21/gtazk.png)
![gt1aL.png](https://i.328888.xyz/2023/02/21/gt1aL.png)



# 日期格式to_date('xxx','yyyyMMddHHmmss')
#### 差异点
hive语法中，to_date函数用法为：to_date(string timestamp)，返回DATE类型，格式为 yyyy-mm-dd ，仅有一个参数，支持用format格式解析

spark语法中，to_date函数用法为：to_date(date_str[, fmt]) ，返回DATE类型，格式为 yyyy-mm-dd ，支持用format格式解析日期

maxcompute语法中，to_date函数用法为：to_date(string <date>, string <format>)，返回DATETIME类型，格式为 yyyy-mm-dd hh:mi:ss ，支持用format格式解析日期



📢这里要注意的是，虽然spark和maxcompute中，to_date函数都支持用format格式解析日期，format格式是有差异的，主要表现在 分钟 位的格式

spark的format格式：yyyy为4位数的年，MM为2位数的月，dd为2位数的日，HH为24小时制的时，mm为2位数的分钟，ss为2位数的秒，ff3为3位精度毫秒
maxcompute的format格式：yyyy为4位数的年，mm为2位数的月，dd为2位数的日，hh为24小时制的时，mi为2位数的分钟，ss为2位数的秒，ff3为3位精度毫秒


#### 举例
测试sql
```sql
select to_date('20221118123456','yyyyMMddHHmmss'),to_date('2022-11-18 12:34:56','yyyy-MM-dd HH:mm:ss');
```
**maxcompute运行结果**
![gtY0p.png](https://i.328888.xyz/2023/02/21/gtY0p.png)

maxcompute报异常： FAILED: ODPS-0121095:Invalid arguments - format string has second part, but doesn't have minute part : yyyyMMddHHmmss

**hive运行结果**
![gtqXU.png](https://i.328888.xyz/2023/02/21/gtqXU.png)

hive报异常： Arguments length mismatch ''yyyyMMddhhmmss'': to_date() requires 1 argument, got 2

提示的是to_date函数仅有1个参数

去掉format参数后的运行结果为：
![gtuJv.png](https://i.328888.xyz/2023/02/21/gtuJv.png)

从结果可以看到，to_date不能解析 yyyyMMddhhmmss 和 yyyyMMdd 格式

**spark运行结果**
![gWiD3.png](https://i.328888.xyz/2023/02/21/gWiD3.png)

spark能顺利产出结果

#### 替换方案
format格式修改：yyyy为4位数的年，mm为2位数的月，dd为2位数的日，hh为24小时制的时，mi为2位数的分钟，ss为2位数的秒，ff3为3位精度毫秒

修改后的能正常产出结果：
![gW4i8.png](https://i.328888.xyz/2023/02/21/gW4i8.png)

另，常见使用to_date报错sql为 date_format(date_add(to_date(pay_time,'yyyyMMddHHmmss'),2),'yyyyMMddHHmmss') ，解读sql的作用是对 pay_time 加 2 天，建议用 UDF 修改这段sql为 yt_date_add(pay_time,2)，修改后简洁明了

# date日期函数
#### 差异点
spark和hive的date函数支持将标准的日期string转换为date类型

maxcomputer date函数只支持标准的日期string，带时分秒的时间string不支持

#### 举例
测试sql
```sql
select date('2022-12-21'),date('2022-12-21 01:22:01');
```

**maxcompute运行结果**
![gWt0Q.png](https://i.328888.xyz/2023/02/21/gWt0Q.png)

maxcomputer对标准的日期string【2022-12-21】转换正确

但是对带时分秒的string转为错误，直接为null

**hive运行结果**
![gWWXE.png](https://i.328888.xyz/2023/02/21/gWWXE.png)

结果符合预期

**spark运行结果**
![gWkKC.png](https://i.328888.xyz/2023/02/21/gWkKC.png)

spark能顺利产出结果

#### 替换方案
如果是为了格式转换，使用自定义 yt_date_format 函数

如果是为了获取date类型，使用 to_date函数
```sql
select yt_date_format('2022-12-21 01:22:01','yyyy-MM-dd')
,to_date('2022-12-21 01:22:01');
```
![gWCDP.png](https://i.328888.xyz/2023/02/21/gWCDP.png)

# from_unixtime函数
#### 差异点
spark和hive的from_unixtime函数将时间戳转换成格式化string类型，当时间戳为负数时，正常转换

maxcomputer from_unixtime函数转换负数时间戳时，存在时间便宜

#### 举例
测试sql
select '1018-10-15 00:00:00'  -- yyyyMMddHHmmss 时间戳
,unix_timestamp('1018-10-15 00:00:00') --时间戳
,from_unixtime(unix_timestamp('1018-10-15 00:00:00'),'yyyyMMddHHmmss') --转换格式


**maxcompute运行结果**
![gWHiJ.png](https://i.328888.xyz/2023/02/21/gWHiJ.png)

可以看到，原先日期为 '1018-10-15 00:00:00',转换成yyyyMMddHHmmss格式原本期望为  10181015000000

但是实际结果为10181008235417,和预期不符合

**hive运行结果**
![gWObc.png](https://i.328888.xyz/2023/02/21/gWObc.png)

hive结果符合预期

**spark运行结果**
![gWb6A.png](https://i.328888.xyz/2023/02/21/gWb6A.png)

spark产出结果正确

#### 替换方案
使用自定义 yt_date_format 函数
```sql
select '1018-10-15 00:00:00' -- yyyyMMddHHmmss 时间戳
,unix_timestamp('1018-10-15 00:00:00') --时间戳
,yt_date_format('1018-10-15 00:00:00','yyyyMMddHHmmss') --转换格式
```
![gWICN.png](https://i.328888.xyz/2023/02/21/gWICN.png)
使用自定义udf后正确

# concat_ws差异
#### 差异点
spark的concat_ws会支持类型的隐式转换

hive和maxcomputer concat_ws不支持，只支持同类型使用

#### 举例
测试sql
```sql
select concat_ws(",",array(1,2,3))
```

**maxcompute运行结果**
![gWNgV.png](https://i.328888.xyz/2023/02/21/gWNgV.png)

报错提示数据类型不对，concat_ws只能处理ARRAY<STRING>数据类型，而sql中是ARRAY<INT>数据类型，[官方文档](https://help.aliyun.com/document_detail/455513.html?spm=a2c4g.11186623.0.0.28686fd3N35cvE) 中有详细说明
![gWmSz.png](https://i.328888.xyz/2023/02/21/gWmSz.png)


**hive运行结果**
![gWBVw.png](https://i.328888.xyz/2023/02/21/gWBVw.png)

报错提示数据类型不对，与maxcompute一个意思，concat_ws传入数组必须是Array<int>类型

**spark运行结果**

![gWXba.png](https://i.328888.xyz/2023/02/21/gWXba.png)

spark执行结果符合预期

#### 替换方案
使用阿里云提供的array_join函数
```sql
select array_join(array(1,2,3),",");
```
![gWegp.png](https://i.328888.xyz/2023/02/21/gWegp.png)

