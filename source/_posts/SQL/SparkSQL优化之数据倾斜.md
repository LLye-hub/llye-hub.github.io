---
title: SparkSQL优化之数据倾斜
toc: true
tags:
  - 数据倾斜
categories:
  - SQL优化
abbrlink: faab1ad7
date: 2023-02-09 10:19:52
---
# 前言
在Spark作业优化场景中，最常见且比较棘手的就是数据倾斜问题。个人认为，具备数据倾斜调优能力对从事数仓开发人员是必备的基本要求。当然，数据倾斜的场景是比较复杂的，针对不同的数据倾斜有不同的处理方案。

# 如何辨别和定位数据倾斜
从Spark作业的执行计划看，若出现某个task任务比其他task任务执行耗时极其久，比如：某个stage有100个task，其中99个task在1min左右就执行成功，但是有1个task却执行了1个小时甚至更久，这种情况显然是出现了数据倾斜。

数据倾斜问题仅出现在shuffle过程，一些会触发shuffle的算子：distinct、groupByKey、reduceByKey、aggregateByKey、countByKey、join、cogroup、repartition等。
对应提交的SparkSQL中可能有distinct、count(distinct)、group by、partition by、join等关键词。

# 常见的数据倾斜场景及解决方案



# 碰到的数据倾斜案例
## 窗口分组数据倾斜
**倾斜场景**
业务上有一张消息记录表msg_records，sql要求是取下一次回复消息
```sql
WITH msg_tmp as
(
    select  id                  -- 唯一键，消息id
           ,from_chat_id        -- 消息发送者id
           ,to_chat_id          -- 消息接受者id
           ,msg_time            -- 消息时间
    from msg_records
)
select  id
       ,msg_time
       ,first_value(if(type = 'reply',id,null),true) over(partition by from_chat_id,to_chat_id order by msg_time,id rows between 1 following and unbounded following) as reply_msg_id_n1t -- 取下一次回复消息
from
(
    select  id
           ,from_chat_id
           ,to_chat_id
           ,msg_time
           ,'send' as type
    from msg_tmp
    union all
    -- 调转，取返回消息
    select  id
           ,to_chat_id   as from_chat_id
           ,from_chat_id as to_chat_id
           ,msg_time
           ,'reply'      as type
    from msg_tmp
) t1
```

**sql执行分析**
有一个task执行耗时1h
![R2j3w.png](https://i.328888.xyz/2023/02/10/R2j3w.png)
 
**数据倾斜分析**
根据窗口函数的分组```from_chat_id + to_chat_id```分析，数据量出现严重倾斜，表总数据量1亿多，其中，分组```from_chat_id=12 and to_chat_id=81867```的数据量有30w，其他分组数据量至多3w。

另外，分组```from_chat_id=12 and to_chat_id=81867```的数据在业务上可定义为脏数据，且first_value()函数计算出的值全为null。

~~猜测原因是，first_value()函数采取isIgnoreNull策略时，底层的执行原理是：以30w数据量为例，第一条判断是null，从第二条开始往下找，一直找不到，就找到第30w条，也就是30w次；第二条则是从第三条找，找第30w条……。倾斜key窗口全是null就是最坏的情况，299999+299998+……+1，高斯求和为30w*15W，这样看数据量不算很大，但是计算量非常大。（*first_value()函数底层实现原理尚未查到资料验证*）~~

根据上述分析，再对照spark执行计划基本可以定位倾斜原因为**窗口数据倾斜和first_value()函数计算耗时**

**解决方案**
结合业务知识，在sql逻辑中过滤```from_chat_id=12 and to_chat_id=81867```的数据

最终，任务执行耗时从```1h```优化至```10min```



# 参考资料
[美团技术团队：Spark性能优化指南——高级篇](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)