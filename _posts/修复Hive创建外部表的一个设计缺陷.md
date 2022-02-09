---
title: 修复Hive创建外部表的一个设计缺陷
tags: 大数据源码
categories: 大数据源码
keywords: Hive源码，Hive外接表，Hive删除表失败
description: 修复Hive创建外部表的一个设计缺陷，Hive删除外接表失败如何进行修复
abbrlink: 4cc9b567
date: 2022-02-07 10:39:17
---

前面的文章我提到Hive的一个缺陷，我测试的版本主要是存在于3.1.X的版本，问题这一张图可以直接概括：

![image](hive-bugs.png)

也可以翻一下之前的文章：https://baifachuan.com/posts/c5888fa1.html

我也给Hive提了一个issue，链接在这里：https://issues.apache.org/jira/browse/HIVE-25912 

顺带修复了一下，MR在这里：https://github.com/apache/hive/pull/2987

修复思路我开始是让它删除成功，所以在：common/src/java/org/apache/hadoop/hive/common/FileUtils.java的746行左右添加了这个检查：

```java
if(path.getParent() == null) {
  // no file/dir to be deleted, because of the path is root dir, hive table forbid set the location to root dir.
  return;
}
```

后来觉得加在这里不太合适，不严谨，于是添加到创建表的地方，也就是路径不合理的话就抛出异常，修复的位置是：standalone-metastore/src/main/java/org/apache/hadoop/hive/metastore/HiveMetaStore.java

在`create_table_core`方法里面添加了如下内容：

```java
if (!MetaStoreUtils.validateTblStorage(tbl.getSd())) {
    throw new InvalidObjectException(tbl.getTableName()
        + " location must not be root path");
}
```
其中`validateTblStorage`的方法如下：

```java
 /*
   * Check the table storage location must not be root path.
   */
  static public boolean validateTblStorage(StorageDescriptor sd) {
    return !(StringUtils.isNotBlank(sd.getLocation())
            && new Path(sd.getLocation()).getParent() == null);
  }
```

这样的话，创建如果是不合法的表，就会失败，演示一下就是这样：。

![image](hive-bugs-001.png)

![image](hive-bugs002.png)

继续和社区讨论一下如何把这个合进去。
