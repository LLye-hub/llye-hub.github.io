title: hive之常用元数据表
tags:

- hive元数据
  categories:
- hive
  toc: true
  date: 2023-05-16 11:30:36

---

hive元数据信息通常存储在关系型数据库，常见的是MySQL数据库。hive元数据信息存储在MySQL库的57张表中。
![Vi0EjJ.png](https://i.328888.xyz/2023/05/16/Vi0EjJ.png)

# 存储Hive版本的元数据表（VERSION）


| VER_ID | SCHEMA_VERSION | VERSION_COMMENT            |
| -------- | ---------------- | ---------------------------- |
| 1      | 2.3.0          | Hive release version 2.3.0 |

该表比较重要，如果出现问题，根本进入不了Hive-Cli。比如该表不存在，当启动Hive-Cli时候，就会报错”Table ‘hive.version’ doesn’t exist”。

# Hive数据库相关的元数据表（DBS、DATABASE_PARAMS）

## DBS

DBS表存储的是hive中所有库的基本信息


| DB_ID | DESC                  | DB_LOCATION_URI                                   | NAME    | OWNER_NAME | OWNER_TYPE |
| ------- | ----------------------- | --------------------------------------------------- | --------- | ------------ | ------------ |
| 1     | Default Hive database | hdfs://localhost:8020/user/hive/warehouse         | default | public     | ROLE       |
| 6     |                       | hdfs://localhost:8020/user/hive/warehouse/test.db | test    | llye       | USER       |

## DATABASE_PARAMS

DATABASE_PARAMS表存储数据库的相关参数，在CREATE DATABASE时候用`WITH DBPROPERTIES(property_name=property_value, …)`指定的参数。

表字段有：`DB_ID`、`PARAM_KEY`、`PARAM_VALUE`

# Hive表和视图相关的元数据表（TBLS、TABLE_PARAMS、TBL_PRIVS）

## TBLS

TBLS表存储了表/视图的基本信息，表字段有：`TBL_ID `、`CREATE_TIME `、`DB_ID`、`LAST_ACCESS_TIME`(上次访问时间)、`OWNER`、`RETENTION`(保留字段)、`SD_ID`(序列化配置信息)、`TBL_NAME`、`TBL_TYPE`、`VIEW_EXPANDED_TEXT`(视图的详细HQL语句)、`VIEW_ORIGINAL_TEXT`(视图的原始HQL语句)。


| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER | RETENTION | SD_ID | TBL_NAME    | TBL_TYPE      | VIEW_EXPANDED_TEXT                                                                                                                                                                                                                                       | VIEW_ORIGINAL_TEXT                      | IS_REWRITE_ENABLED |
| -------- | ------------- | ------- | ------------------ | ------- | ----------- | ------- | ------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | -------------------- |
| 2      | 1678874852  | 1     | 0                | llye  | 0         | 2     | student     | MANAGED_TABLE |                                                                                                                                                                                                                                                          |                                         | 0                  |
| 13     | 1683273717  | 6     | 0                | llye  | 0         | 13    | travel_data | MANAGED_TABLE |                                                                                                                                                                                                                                                          |                                         | 0                  |
| 16     | 1684216889  | 1     | 0                | llye  | 0         | 16    | test_query  | VIRTUAL_VIEW  | (¶select¶\`travel_data\`.\`province\`, \`travel_data\`.\`city\`, \`travel_data\`.\`attraction\`, \`travel_data\`.\`star_level\`, \`travel_data\`.\`price\`, \`travel_data\`.\`sales\`, \`travel_data\`.\`sale_date\`¶from¶ \`test\`.\`travel_data\`) | (¶select¶ *¶from¶ test.travel_data) | 0                  |

## TABLE_PARAMS

TABLE_PARAMS表存储了表/视图的属性信息，表字段有：`TBL_ID`、`PARAM_KEY`(totalSize,numRows,EXTERNAL)、`PARAM_VALUE`

|TBL_ID|PARAM_KEY|PARAM_VALUE|
|------|---------|-----------|
|2|COLUMN_STATS_ACCURATE|{"BASIC_STATS":"true"}|
|2|numFiles|1|
|2|numRows|1|
|2|rawDataSize|5|
|2|totalSize|6|
|2|transient_lastDdlTime|1678874879|
|13|COLUMN_STATS_ACCURATE|{"BASIC_STATS":"true"}|
|13|numFiles|1|
|13|numRows|11|
|13|rawDataSize|597|
|13|totalSize|608|
|13|transient_lastDdlTime|1683273753|
|16|transient_lastDdlTime|1684216889|


## TBL_PRIVS

TBL_PRIVS表存储了表/视图的授权信息，表字段有：`TBL_GRANT_ID`、`CREATE_TIME`、`GRANT_OPTION`、`GRANTOR`(授权执行用户)、`GRANTOR_TYPE`、`PRINCIPAL_NAME`(被授权用户)、`PRINCIPAL_TYPE`、`TBL_PRIV`、`TBL_ID`

|TBL_GRANT_ID|CREATE_TIME|GRANT_OPTION|GRANTOR|GRANTOR_TYPE|PRINCIPAL_NAME|PRINCIPAL_TYPE|TBL_PRIV|TBL_ID|
|------------|-----------|------------|-------|------------|--------------|--------------|--------|------|
|5|1678874852|1|llye|USER|llye|USER|INSERT|2|
|6|1678874852|1|llye|USER|llye|USER|SELECT|2|
|7|1678874852|1|llye|USER|llye|USER|UPDATE|2|
|8|1678874852|1|llye|USER|llye|USER|DELETE|2|
|39|1683273717|1|llye|USER|llye|USER|INSERT|13|
|40|1683273717|1|llye|USER|llye|USER|SELECT|13|
|41|1683273717|1|llye|USER|llye|USER|UPDATE|13|
|42|1683273717|1|llye|USER|llye|USER|DELETE|13|
|46|1684216889|1|llye|USER|llye|USER|INSERT|16|
|47|1684216889|1|llye|USER|llye|USER|SELECT|16|
|48|1684216889|1|llye|USER|llye|USER|UPDATE|16|
|49|1684216889|1|llye|USER|llye|USER|DELETE|16|


# Hive文件存储信息相关的元数据表（SDS、SD_PARAMS、SERDES、SERDE_PARAMS）

由于HDFS支持的文件格式很多，而建Hive表时候也可以指定各种文件格式，Hive在将HQL解析成MapReduce时候，需要知道去哪里，使用哪种格式去读写HDFS文件，而这些信息就保存在这几张表中。

## SDS

SDS表保存文件存储的基本信息，如INPUT_FORMAT、OUTPUT_FORMAT、是否压缩等。TBLS表中的SD_ID与该表关联，可以获取Hive表的存储信息。

表字段有：`SD_ID`、`CD_ID`(字段信息id)、`INPUT_FORMAT`(文件输入格式)、`IS_COMPRESSED`(是否压缩)、`IS_STOREDASSUBDIRECTORIES`(是否以子目录存储)、`LOCATION`(HDFS路径)、`NUM_BUCKETS`(分桶数量)、`OUTPUT_FORMAT`(文件输出格式)、`SERDE_ID`(序列化类id)。

|SD_ID|CD_ID|INPUT_FORMAT|IS_COMPRESSED|IS_STOREDASSUBDIRECTORIES|LOCATION|NUM_BUCKETS|OUTPUT_FORMAT|SERDE_ID|
|-----|-----|------------|-------------|-------------------------|--------|-----------|-------------|--------|
|2|2|org.apache.hadoop.mapred.TextInputFormat|0|0|hdfs://localhost:8020/user/hive/warehouse/student|-1|org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat|2|
|13|13|org.apache.hadoop.mapred.TextInputFormat|0|0|hdfs://localhost:8020/user/hive/warehouse/test.db/travel_data|-1|org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat|13|
|16|16|org.apache.hadoop.mapred.TextInputFormat|0|0|NULL|-1|org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat|16|


## SD_PARAMS

SD_PARAMS表保存Hive存储的属性信息，在创建表时候使用`STORED BY ‘storage.handler.class.name’ [WITH SERDEPROPERTIES (…)`指定。

表字段有：`SD_ID`、`PARAM_KEY`、`PARAM_VALUE`


## SERDES

SERDES表存储序列化使用的类信息

表字段有：`SERDE_ID`、`NAME`(序列化类别名)、`SLIB`(序列化类)。

|SERDE_ID|NAME|SLIB|
|--------|----|----|
|2|NULL|org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe|
|13|NULL|org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe|
|16|NULL|NULL|


## SERDE_PARAMS

SERDE_PARAMS表存储了hive表序列化的一些属性信息，比如:行、列分隔符

表字段有：`SERDE_ID`、`PARAM_KEY`、`PARAM_VALUE`

|SERDE_ID|PARAM_KEY|PARAM_VALUE|
|--------|---------|-----------|
|2|serialization.format|1|
|13|serialization.format|1|


# Hive表字段相关的元数据表（COLUMNS_V2）

COLUMNS_V2表存储了hive表各字段的基本信息。

表字段有：`CD_ID`、`COMMENT`、`COLUMN_NAME`、`TYPE_NAME`(字段类型)、`INTEGER_IDX`(字段顺序)。

|CD_ID|COMMENT|COLUMN_NAME|TYPE_NAME|INTEGER_IDX|
|-----|-------|-----------|---------|-----------|
|2|NULL|id|int|0|
|2|NULL|name|string|1|
|13|NULL|attraction|string|2|
|13|NULL|city|string|1|
|13|NULL|price|double|4|
|13|NULL|province|string|0|
|13|NULL|sale_date|string|6|
|13|NULL|sales|int|5|
|13|NULL|star_level|int|3|
|16|NULL|attraction|string|2|
|16|NULL|city|string|1|
|16|NULL|price|double|4|
|16|NULL|province|string|0|
|16|NULL|sale_date|string|6|
|16|NULL|sales|int|5|
|16|NULL|star_level|int|3|


# Hive表分区相关的元数据表（PARTITIONS、PARTITION_KEYS、PARTITION_KEY_VALS、PARTITION_PARAMS）

## PARTITIONS

PARTITIONS表存储hive表分区的基本信息。

表字段有：`PART_ID`、`CREATE_TIME`、`LAST_ACCESS_TIME`、`PART_NAME`、`SD_ID`(分区存储ID)、`TBL_ID`、`LINK_TARGET_ID`。

|PART_ID|CREATE_TIME|LAST_ACCESS_TIME|PART_NAME|SD_ID|TBL_ID|
|-------|-----------|----------------|---------|-----|------|
|1|1684221232|0|dt=1|18|17|
|2|1684221294|0|dt=2|19|17|

## PARTITION_KEYS

PARTITION_KEYS表存储hive表分区的字段信息。

表字段有：`TBL_ID`、`PKEY_COMMENT`、`PKEY_NAME`、`PKEY_TYPE`、`INTEGER_IDX`(分区字段顺序)。

|TBL_ID|PKEY_COMMENT|PKEY_NAME|PKEY_TYPE|INTEGER_IDX|
|------|------------|---------|---------|-----------|
|17|NULL|dt|string|0|


## PARTITION_KEY_VALS

PARTITION_KEY_VALS表存储hive表分区字段值。

表字段有：`PART_ID`、`PART_KEY_VAL`(分区字段值)、`INTEGER_IDX`(分区字段值顺序)。

|PART_ID|PART_KEY_VAL|INTEGER_IDX|
|-------|------------|-----------|
|1|1|0|
|2|2|0|


## PARTITION_PARAMS

PARTITIONS表存储hive表分区的属性信息。

表字段有：`PART_ID`、`PARAM_KEY`(numFiles，numRows)、`PARAM_VALUE`。

|PART_ID|PARAM_KEY|PARAM_VALUE|
|-------|---------|-----------|
|1|COLUMN_STATS_ACCURATE|{"BASIC_STATS":"true"}|
|1|numFiles|1|
|1|numRows|11|
|1|rawDataSize|4290|
|1|totalSize|1221|
|1|transient_lastDdlTime|1684221233|
|2|COLUMN_STATS_ACCURATE|{"BASIC_STATS":"true"}|
|2|numFiles|1|
|2|numRows|4|
|2|rawDataSize|1564|
|2|totalSize|1040|
|2|transient_lastDdlTime|1684221294|


# 其他不常用的元数据表

DB_PRIVS：数据库权限信息表。通过GRANT语句对数据库授权后，将会在这里存储。  
IDXS：索引表，存储Hive索引相关的元数据。  
INDEX_PARAMS：索引相关的属性信息。  
TBL_COL_STATS：表字段的统计信息。使用ANALYZE语句对表字段分析后记录在这里。  
TBL_COL_PRIVS：表字段的授权信息。  
PART_PRIVS：分区的授权信息。  
PART_COL_PRIVS：分区字段的权限信息。  
PART_COL_STATS：分区字段的统计信息。  
FUNCS：用户注册的函数信息。  
FUNC_RU：用户注册函数的资源信息。  
……  

# 参考资料

[Hive 元数据表结构详解](https://mp.weixin.qq.com/s?__biz=MzA3ODUxMzQxMA==&mid=2663993556&idx=1&sn=0e5291bd63426d747f32a7fd05128caa&scene=21#wechat_redirect)

