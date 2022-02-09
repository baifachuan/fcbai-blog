---
title: Hive创建外部表的一个设计缺陷
tags: 大数据源码
categories: 大数据源码
keywords: Hive删除外部表失败，state=08S01，code=1，Hive如何删除外部表
description: >-
  解决Hive删除外部表失败的问题，Hive如何删除外部表，org.apache.hadoop.hive.ql.exec.DDLTask.
  MetaException(message:java.lang.NullPointerException)
abbrlink: c5888fa1
date: 2022-01-30 14:06:38
---

最近在集成hive的时候发现一个问题，现象是这样的，我需要创建一个外部表：

```sql
 CREATE EXTERNAL TABLE `inventory3`(
    `inv_item_sk` int,
    `inv_warehouse_sk` int,
    `inv_quantity_on_hand` int)
    PARTITIONED BY (
    `inv_date_sk` int) STORED AS ORC
    LOCATION
    'hdfs://master-1:8020/';
```

就是一个普通的外部表创建，使用beeline连接hive server2去创建也会成功，但是删除的时候却不行，会有如下问题：

```shell
0: jdbc:hive2://master> drop table inventory3;
INFO  : Compiling command(queryId=hive_20220130135301_09abe5e5-7538-4137-b0f7-559f998a3820): drop table inventory3
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20220130135301_09abe5e5-7538-4137-b0f7-559f998a3820); Time taken: 0.075 seconds
INFO  : Executing command(queryId=hive_20220130135301_09abe5e5-7538-4137-b0f7-559f998a3820): drop table inventory3
INFO  : Starting task [Stage-0:DDL] in serial mode
ERROR : FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. MetaException(message:java.lang.NullPointerException)
INFO  : Completed executing command(queryId=hive_20220130135301_09abe5e5-7538-4137-b0f7-559f998a3820); Time taken: 0.097 seconds
Error: Error while processing statement: FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. MetaException(message:java.lang.NullPointerException) (state=08S01,code=1)
```

虽然我知道具体的原因，但是还是想看一下源码的流程，所以必须要远程debug一下，远程debug hive的方式我之前写了一篇文章：https://baifachuan.com/posts/6881ba5d.html

通过断点的形式，发现在删除表达是时候会进入：`common/src/java/org/apache/hadoop/hive/common/FileUtils.java`的`checkDeletePermission`方法，最终在这个位置：

```
// check if sticky bit is set on the parent dir
FileStatus parStatus = fs.getFileStatus(path.getParent());
if (!shims.hasStickyBit(parStatus.getPermission())) {
  // no sticky bit, so write permission on parent dir is sufficient
  // no further checks needed
  return;
}
```
会出错，也就是path.getParent() 会返回null，从而导致fs.getFileStatus(path.getParent())出现NPE，这里的原因是因为创建表的时候指定的地址是：`hdfs://master-1:8020/`。

这个地址是跟地址，所以上面在path.getParent(）的时候返回的是null，从业务流程上来说，我能创建成功，但是无法删除，这是产品缺陷，hive的修，从修复来说，有2个地方可以修复。

一个是在hive server2的代码里面进行检查，如果是外部表，且地址是跟的话直接拒绝创建，当然也可以在ranger的hive plugin里面去鉴别也行。

另一种是直接修复hdfs的底层fs client，让Path这个对象在getParent的时候返回"/"而不是nul，不过我更倾向于前者修复方式。

因为最近集成到这部分工作，所以打算自己修复一下，然后给hive 提个MR。
