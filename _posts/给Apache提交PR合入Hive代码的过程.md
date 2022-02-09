---
title: 给Apache提交PR合入Hive代码的过程
tags: 大数据源码
categories: 大数据源码
keywords: 'Hive PR, hive源码,hive贡献代码'
description: 如何给Hive提交PR，如何给Hive贡献代码，如果提交代码给Apache
abbrlink: ba4b1fb8
date: 2022-02-09 20:08:44
---

前面我提过我我在开发组件过程中发现Hive的一个bug，这个bug会在集成Ranger/OpenLDAP或者开启Kerberos模式下出现，因为开启权限后在删除表的时候它会去检查location的权限，从而导致出错。

之前记录的信息在这里：

[点击进入：修复Hive创建外部表的一个设计缺陷](https://baifachuan.com/posts/4cc9b567.html)
[点击进入：Hive创建外部表的一个设计缺陷](https://baifachuan.com/posts/c5888fa1.html)

我也给他们提了issue和提了PR：

https://issues.apache.org/jira/browse/HIVE-25912
https://github.com/apache/hive/pull/2987

在这个过程中比较麻烦的是找reviewer，我提了PR后，有几个Contributor倒是觉得这个修复是合理的，聊了很久。

apache的邮件更新很慢，我又从github找到twitter上，一路艰辛。

有人提到这个问题是不是master上面也有，因为我自己只使用3.1.2所以没有测试master，考虑到修复问题，我又升级了一下我的Hive到最新的4.x.

在升级后使用的时候发现元数据不兼容，也就是这个问题：
```
Exception in thread "main" MetaException(message:Hive Schema version 4.0.0 does not match metastore's schema version 3.1.0 Metastore is not upgraded or corrupt)
```
开始我粗暴的修复hive-site.xml，关闭了这个校验：
```
<property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
</property>
```

但是发现版本差异太大，元数据直接结构就不一样了，于是又重建了元数据。

```
./bin/schematool --dbType mysql -initSchema
```

测试了一下发现这个问题在Master依旧存在，于是又给Master提了一个PR：

https://github.com/apache/hive/pull/3009

老社区合入PR的问题就是人太多，整体的速度就没有那么快。
