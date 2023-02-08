---
title: hadoop命令之distcp分布式拷贝
toc: true
abbrlink: bcc5bdf2
date: 2023-02-01 10:06:03
tags: 
  - hadoop命令
  - hdfs文件拷贝
categories: 
  - hadoop
---
# distcp用途
DistCp（分布式拷贝）是用于大规模集群内部和集群之间拷贝的工具。 
使用Map/Reduce实现文件分发，错误处理和恢复，以及报告生成。 
DistCp将文件和目录的列表作为map任务的输入，每个任务会完成源列表中部分文件的拷贝。 

# distcp用法
命令行中可以指定多个源目录
```bash
# 
hadoop distcp source_dir1 [source_dir2 source_dir3……] target_dir
```
集群内拷贝
```bash
# 
hadoop distcp [hdfs://nn:8020]/db/table_a/partition=1 [hdfs://nn:8020]/db/table_b/partition=1
```
不同集群间拷贝，DistCp必须运行在目标端集群上
```bash
# 
hadoop distcp hdfs://nn1:8020/db/table_a/partition=1 hdfs://nn2:8020/db/table_b/partition=1
```

# 常用参数选项
## -overwrite	
源文件覆盖同名目标文件

## -update	
拷贝目标目录下不存在而源目录下存在的文件，当文件大小不一致时，源文件覆盖同名目标文件

## -delete
删除目标目录下存在，但源目录下不存在的文件，需要配合```-update```或```-overwrite```使用

## -p[rbugpcaxt]
控制是否保留源文件的属性，```-p```默认全部保留，常用的为```-pbugp```。
修改次数不会被保留。并且当指定 -update 时，更新的状态不会 被同步，除非文件大小不同（比如文件被重新创建）。
| 标识 | 含义 | 备注 |
| ---- | ----- | ----- |
| r | replication number | 文件副本数 |
| b | block size | 文件块大小 |
| u | user | 用户 |
| g | group | 组 |
| p | permission | 文件权限 |
| c | checksum-type | 校验和类型 |
| a | acl |  |
| x | xattr |  |
| t | timestamp | 时间戳 |

## -m
控制拷贝时的map任务最大个数
如果没使用-m选项，DistCp会尝试在调度工作时指定map数目=min(total_bytes/bytes.per.map,20*num_task_trackers)， 其中bytes.per.map默认是256MB。

# 应用实例
## 表结构一致的两表互相拷贝数据

```bash
#********************************************************************************
# **  功能描述：通过hdfs文件路径拷贝的方式，实现表结构完全相同的表互相拷贝数据
#********************************************************************************
# 指定源路径、目标路径
source_dir=/db/table_a/partition=1
target_dir=/db/table_b/partition=1
db_name=db_a
target_tbl_name=db_a.table_b
# 判断源路径是否存在，不存在则返回
hadoop fs -test -e $source_dir
if [ $? -ne 0 ];then
  echo "源路径$source_dir不存在"
  exit 1
fi
# 判断目标路径是否存在，不存在则创建
hadoop fs -test -e $target_dir
if [ $? -ne 0 ];then
  hadoop fs -mkdir $target_dir
  echo "目标路径$target_dir不存在，创建成功"
fi
# 开始拷贝
echo "开始hdfs文件拷贝，source_dir=$source_dir，target_dir=$target_dir"
hadoop distcp -overwrite -delete -pbugp $source_dir $target_dir
if [ $? -eq 0 ];then
  echo "hdfs文件拷贝成功"
else
  echo "hdfs文件拷贝失败"
  exit -1
fi
# 刷新目标表的metastore信息
hive -database $db_name -v -e "msck repair table $target_tbl_name;"
if [ $? -eq 0 ];then
  echo "$target_tbl_name表的metastore信息刷新成功"
  exit 0
fi
```


# 参考资料
[DistCp使用指南](https://hadoop.apache.org/docs/r1.0.4/cn/distcp.html)
[Hadoop中文网：DistCp](https://hadoop.org.cn/docs/hadoop-distcp/DistCp.html)