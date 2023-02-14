---
title: SparkSQL无法处理hive表中的空ORC文件
categories:
  - SQL
tags:
  - HDFS
  - ORC
abbrlink: 1f69e18b
date: 2022-12-16 17:56:46
toc: true
---

# 为什么碰到这个问题
起因是在使用SparkSQL查询表时，遇到报错：java.lang.RuntimeException: serious problem at OrcInputFormat.generateSplitsInfo
![AfHSH.png](https://i.328888.xyz/2022/12/19/AfHSH.png)
![AfbiQ.png](https://i.328888.xyz/2022/12/19/AfbiQ.png)
之后，换了hiveSQL执行成功，但这并不算排查成功，排查应尽可能追根究底，以后才能做到举一反三，所以基于网上资料和个人理解写了这篇博客

# 问题分析
## 定位问题
根据报错的java类名+方法名（OrcInputFormat.generateSplitsInfo），可以判断问题出现在读取orc文件阶段。

## 查看HDFS文件
查看表存储路径下的文件，发现有1个空文件
![AfjbE.png](https://i.328888.xyz/2022/12/19/AfjbE.png)

## 为什么会有空文件（待详细补充）
1、sparkSQL建表
2、表写入数据时，sql最后做了distribute by操作，产生了空文件

sparksql读取空文件的时候，因为表是orc格式的，导致sparkSQL解析orc文件出错。但是用hive却可以正常读取。

## 网上搜罗的解决办法
问题原因基本清晰了，就是读取空文件导致的报错，如果非得用SparkSQL执行查询语句，这里提供几种解决方案：

#### 1、修改表存储格式为parquet
这种方法是网上查询到的，但在实际数仓工作中，对于已在使用中的表来说，删表重建操作是不允许的，所以不推荐

#### 2、参数设置：`set hive.exec.orc.split.strategy=ETL`
既然已经定位到是空文件读取的问题，那就从文件读取层面解决。

自建集群`Spark`源码：
```java
// org/apache/hadoop/hive/ql/io/orc/OrcInputFormat.java
switch(context.splitStrategyKind) {
    case BI:
    // BI strategy requested through config
    splitStrategy = new BISplitStrategy(context, fs, dir, children, isOriginal,
        deltas, covered);
    break;
    case ETL:
    // ETL strategy requested through config
    splitStrategy = new ETLSplitStrategy(context, fs, dir, children, isOriginal,
        deltas, covered);
    break;
    default:
    // HYBRID strategy
    if (avgFileSize > context.maxSize) {
        splitStrategy = new ETLSplitStrategy(context, fs, dir, children, isOriginal, deltas,
            covered);
    } else {
        splitStrategy = new BISplitStrategy(context, fs, dir, children, isOriginal, deltas,
            covered);
    }
    break;
}

// ./repository/org/spark-project/hive/hive-exec/1.2.1.spark2/hive-exec-1.2.1.spark2.jar!/org/apache/hadoop/hive/conf/HiveConf.class
HIVE_ORC_SPLIT_STRATEGY("hive.exec.orc.split.strategy", "HYBRID", new StringSet(new String[]{"HYBRID", "BI", "ETL"}), "This is not a user level config. BI strategy is used when the requirement is to spend less time in split generation as opposed to query execution (split generation does not read or cache file footers). ETL strategy is used when spending little more time in split generation is acceptable (split generation reads and caches file footers). HYBRID chooses between the above strategies based on heuristics.")      
  

```
也就是说，默认是HYBRID（混合模式读取，根据平均文件大小和文件个数选择ETL还是BI模式）。
+ BI策略以文件为粒度进行split划分
+ ETL策略会将文件进行切分，多个stripe组成一个split
+ HYBRID策略为：当文件的平均大小大于hadoop最大split值（默认256 * 1024 * 1024）时使用ETL策略，否则使用BI策略。 

ETLSplitStrategy和BISplitStrategy两种策略在对getSplits方法采用了不同的实现方式，BISplitStrategy在面对空文件时会出现空指针异常，ETLSplitStrategy则帮我们过滤了空文件。
```java
// org.apache.hadoop.hive.ql.io.orc.OrcInputFormat.BISplitStrategy#getSplits
public List<OrcSplit> getSplits() throws IOException {
  List<OrcSplit> splits = Lists.newArrayList();
  for (FileStatus fileStatus : fileStatuses) {
    String[] hosts = SHIMS
        .getLocationsWithOffset(fs, fileStatus) // 对空文件会返回一个空的TreeMap
        .firstEntry()  // null
        .getValue()    // NPE
        .getHosts();
    OrcSplit orcSplit = new OrcSplit(fileStatus.getPath(), 0, fileStatus.getLen(), hosts,
                                     null, isOriginal, true, deltas, -1);
    splits.add(orcSplit);
  }
  // add uncovered ACID delta splits
  splits.addAll(super.getSplits());
  return splits;
}

// org.apache.hadoop.hive.ql.io.orc.OrcInputFormat.ETLSplitStrategy#getSplits
public List<SplitInfo> getSplits() throws IOException {
    List<SplitInfo> result = Lists.newArrayList();
    for (FileStatus file : files) {
    FileInfo info = null;
    if (context.cacheStripeDetails) {
        info = verifyCachedFileInfo(file);
    }
    // ignore files of 0 length（此处对空文件做了过滤）
    if (file.getLen() > 0) {
        result.add(new SplitInfo(context, fs, file, info, isOriginal, deltas, true, dir, covered));
    }
    }
    return result;
}
```
 本质上是一个BUG，`Spark2.4`版本中解决了这个问题。
```java
// org.apache.hadoop.hive.ql.io.orc.OrcInputFormat.BISplitStrategy#getSplits
public List<OrcSplit> getSplits() throws IOException {
    List<OrcSplit> splits = Lists.newArrayList();
    for (FileStatus fileStatus : fileStatuses) {
    String[] hosts = SHIMS.getLocationsWithOffset(fs, fileStatus).firstEntry().getValue()
        .getHosts();
    OrcSplit orcSplit = new OrcSplit(fileStatus.getPath(), 0, fileStatus.getLen(), hosts,
        null, isOriginal, true, deltas, -1);
    splits.add(orcSplit);
    }

    // add uncovered ACID delta splits
    splits.addAll(super.getSplits());
    return splits;
}
```
了解了spark读取orc文件策略，那么就设置避免混合模式使用根据文件大小分割读取，不根据文件来读取
```sql
set hive.exec.orc.split.strategy=ETL
```
经测试无效。原因分析：
1、参数未生效
2、hdfs文件有两个，大小为49B和7.45G，文件的平均大小肯定是大于256M的，所以按默认HYBRID策略规则应本就是采取的ETL策略split ORC文件

#### 3、参数设置：`spark.sql.hive.convertMetastoreOrc=true`
关于参数的[官方介绍](https://spark.apache.org/docs/2.3.3/sql-programming-guide.html#orc-files)
>Since Spark 2.3, Spark supports a vectorized ORC reader with a new ORC file format for ORC files. To do that, the following configurations are newly added. The vectorized reader is used for the native ORC tables (e.g., the ones created using the clause USING ORC) when spark.sql.orc.impl is set to native and spark.sql.orc.enableVectorizedReader is set to true. For the Hive ORC serde tables (e.g., the ones created using the clause USING HIVE OPTIONS (fileFormat 'ORC')), the vectorized reader is used when spark.sql.hive.convertMetastoreOrc is also set to true.

经测试有效。若仍报错，可尝试搭配spark.sql.orc.impl=native使用。



# 补充知识

![Af23F.jpeg](https://i.328888.xyz/2022/12/19/Af23F.jpeg)
hive.exec.orc.split.strategy参数控制在读取ORC表时生成split的策略。对于一些较大的ORC表，可能其footer较大，ETL策略可能会导致其从hdfs拉取大量的数据来切分split，甚至会导致driver端OOM，因此这类表的读取建议使用BI策略。对于一些较小的尤其有数据倾斜的表（这里的数据倾斜指大量stripe存储于少数文件中），建议使用ETL策略。
另外，spark.hadoop.mapreduce.input.fileinputformat.split.minsize参数可以控制在ORC切分时stripe的合并处理。具体逻辑是，当几个stripe的大小小于spark.hadoop.mapreduce.input.fileinputformat.split.minsize时，会合并到一个task中处理。可以适当调小该值，以此增大读ORC表的并发。

# 参考资料
[SPARK查ORC格式HIVE数据报错NULLPOINTEREXCEPTION](https://www.freesion.com/article/8054484645/)
[SparkSQL读取ORC表时遇到空文件](https://blog.csdn.net/weixin_45240507/article/details/124689323?spm=1001.2101.3001.6650.7&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-7-124689323-blog-100524131.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-7-124689323-blog-100524131.pc_relevant_default&utm_relevant_index=7)