---
title: get_json_object在sql中的高级用法
categories:
  - SQL
abbrlink: 5f45fcd7
date: 2022-12-16 11:46:24
---
# get_json_object在sql中的高级用法
## 语法介绍
```sql
get_json_object(String json_string, String path)
-- return string
```
get_json_object函数是用来根据指定路径提取json字符串中的json对象，并返回json对象的json字符串



## 现有困惑
关于这个函数最常见的用法就是```get_json_object('{"a":"b"}', '$.a')```，返回结果```b```
但```$.a```这种path写法仅适用于简单的多层嵌套json字符串解析，碰到嵌套层有json数组时就难以解析了
比如，要提取下面这段json中的所有```weight```对象的值
```json
{
 "store":
        {
         "fruit":[{"weight":8,"type":"apple"}, {"weight":9,"type":"pear"}],  //json数组
         "bicycle":{"price":19.95,"color":"red"}
         }, 
 "email":"amy@only_for_json_udf_test.net", 
 "owner":"amy" 
}
```
通过```$.store.fruit.weight```路径是无法提取的，```$.store.fruit[0].weight```这种写法仅能获取json数组中第一个json字符串中```weight```对象的值，也总不能用```[0]、[1]、[2]……```的方式无穷尽取值吧

到这里思维就限制住了，遇到这种情况时，以前的方式是通过正则表达式处理
具体实现如下：
首先将```item_properties```按指定分隔符split为array数组，再利用explode函数将array数组的元素逐行输出，最终得到的```item_propertie```即为单个json字符串，可根据```$.```提取指定json对象的值，
```sql
-- item_properties = [{"id":42,"name":"包装","sort":0,"type":1}
--                   ,{"id":43,"name":"种类","sort":0,"type":1}
--                   ,{"id":44,"name":"规格","sort":0,"type":2}
--                   ,{"id":63,"name":"保质期(天)","sort":0,"type":2}
--                   ,{"id":100,"name":"适用年龄","sort":0,"type":2}
--                   ,{"id":101,"name":"储存条件","sort":0,"type":2}]

select get_json_object(item_propertie,'$.id')
from table_a
lateral view explode(split(regexp_replace(substr(item_properties,2,length(item_properties)-2),'\\}\\,\\{','\\}\\|\\|\\{'),'\\|\\|')) tmp as item_propertie
```
但上面这种处理方式存在bug，将json数据split为array数组时，必须保证指定分隔符不出现在单个json字符串中，比如上述case中是用```},{```替换为```}||{```，再以```||```作为分隔符split，如若在单个json字符串中也出现了```},{```或是```||```就会导致解析失败


## 怎么高级了
突然有一天在翻看[hive官方文档](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF?spm=a2c4g.11186623.0.0.3c267254Ka3fUh#LanguageManualUDF-get_json_object)时发现path支持的通配符```*```
[![2Pw2H.png](https://i.328888.xyz/2023/01/17/2Pw2H.png)](https://imgloc.com/i/2Pw2H)
```
$ : 表示根节点
. : 表示子节点
[] : [number]表示数组下标，从0开始
* : []的通配符，返回整个数组
```
所以，一开始的问题应该按如下解法：
```sql
-- jsonArray = {"store":
--                     {
--                     "fruit":[{"weight":8,"type":"apple"}, {"weight":9,"type":"pear"}],
--                     "bicycle":{"price":19.95,"color":"red"}
--                     }, 
--             "email":"amy@only_for_json_udf_test.net", 
--             "owner":"amy"}
select get_json_object(jsonArray, '$.store.fruit[*].weight');
-- return [8,9]
```
笔者个人认为，高级之处在于写法极其清爽，按照以前用正则表达式的处理方法，需要多道处理才能得到结果```[8,9]```，而且其中还有隐性风险，但是现在```$.store.fruit[*].weight```这种极简语法既避免了风险，又清晰易理解