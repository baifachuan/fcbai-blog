---
title: 远程调试Hive组件的Ranger插件
tags: 大数据源码
categories: 大数据源码
description: >-
  远程调试Hive组件的Ranger插件,
  远程调试Hive组件的源码，远程调试Hive组件，Hive远程调试源码，如何使用源码启动Hive，HiveServer2源码调试。
keywords: Hive源码， Hive远程调试，Hive源码开发，Hive权限
abbrlink: 6881ba5d
date: 2022-01-29 16:11:34
---

最近在开发一些底层组件，中间有部分操作涉及到和Hive的交互，当前环境的Hive是集成了Ranger作为权限管理服务。

使用beeline连接到Hive server2进行创建表的时候出现：

```shell
CREATE EXTERNAL TABLE `inventory`(
  `inv_item_sk` int,
  `inv_warehouse_sk` int,
  `inv_quantity_on_hand` int)
  PARTITIONED BY (
  `inv_date_sk` int) STORED AS ORC
  LOCATION
  'xxx://fcbai001/';
```
报错：
```shell
Error: Error while compiling statement: FAILED: HiveAccessControlException Permission denied: user [xxx] does not have [READ] privilege on [xxx://fcbai001/] (state=42000,code=40000)
```

看起来是没有权限的问题，由于后面的存储服务是新开发的，而鉴权体系走的ak/sk，按理说和ranger是没有关系的，使用hive命令直接启动一个内嵌的hs2，绕过ranger鉴权的时候，是可以成功执行的。

需要调试一下搞清楚在这样的情况下，ranger是如何拦截的，我感觉虽然这个地方创建的是hive表，但是对LOCATION的校验应该走的是hdfs的api去做最终的验证，但是入口肯定还是在hive-ranger-plugin。

第一步我需要打一个远程断点，因为hive跑在远程服务器上，所以需要开启hive的debug，由于这个地方的调试我只需要hive server2，所以开启hive server2的bubug就行：

```shell
./bin/hiveserver2 --debug
```

不过启动上面的命令的时候可能会碰到一个错误：
```shell
root# ./bin/hiveserver2 --debug
2022-01-29 15:33:47: Starting HiveServer2
Conflicting collector combinations in option list; please refer to the release notes for the combinations allowed
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```
这是因为垃圾回收器冲突了，ParNewGC是新生代的GC算法，G1GC是不区分新生代、老年代的，所以有了G1GC都不需要再指定别的GC算法了。

如果看jvm源码可以看到有这样的代码：

```
bool Arguments::check_gc_consistency() {  
  bool status = true;  
  // Ensure that the user has not selected conflicting sets  
  // of collectors. [Note: this check is merely a user convenience;  
  // collectors over-ride each other so that only a non-conflicting  
  // set is selected; however what the user gets is not what they  
  // may have expected from the combination they asked for. It's  
  // better to reduce user confusion by not allowing them to  
  // select conflicting combinations.  
  uint i = 0;  
  if (UseSerialGC)                       i++;  
  if (UseConcMarkSweepGC || UseParNewGC) i++;  
  if (UseParallelGC || UseParallelOldGC) i++;  
  if (UseG1GC)                           i++;  
  if (i > 1) {  
    jio_fprintf(defaultStream::error_stream(),  
                "Conflicting collector combinations in option list; "  
                "please refer to the release notes for the combinations "  
                "allowed\n");  
    status = false;  
  }  

  return status;  
}  
```
解决办法是修改{HIVE_HOME}/bin/ext/debug.sh，将参数去掉-XX:+UseParallelGC，也就是这个地方：

![image](hive-debug02.png)

只后再开启hive server2的debug：
```
./bin/hiveserver2 --debug
2022-01-29 16:06:55: Starting HiveServer2
Listening for transport dt_socket at address: 8000
```

会监听在8000端口，然后在idea里面创建remote debug连接：

![image](hive-debug001.png)

host配置自己的服务器地址，同时你需要下载ranger的代码，当然如果你本身是调试hive源码而不是ranger的hive plugi，那么你直接使用hive源码即可。

使用idea连接上远程服务器的端口后，服务器会进行下一步的启动，创建sessio：

```
./bin/hiveserver2 --debug
2022-01-29 16:06:55: Starting HiveServer2
Listening for transport dt_socket at address: 8000
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in Äjar:file:/opt/tiger/current/hive-server2/lib/log4j-slf4j-impl-2.15.0.jar!/org/slf4j/impl/StaticLoggerBinder.classÅ
SLF4J: Found binding in Äjar:file:/opt/tiger/current/hadoop/share/hadoop/common/lib/slf4j-log4j12-1.7.30.jar!/org/slf4j/impl/StaticLoggerBinder.classÅ
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type Äorg.apache.logging.slf4j.Log4jLoggerFactoryÅ
Hive Session ID = bd213902-8c9d-45f7-8daa-bdfe90835e42
Hive Session ID = c7decf18-31a9-4f2f-a1ef-dd283a059a98
```

使用beeline访问后，idea会收到断点：

![image](hive-debug.png)

如此，便可开始调试ranger的代码，看一下hive server2的权限到底是哪一步失败了。

如果是debug hive matestore 的话，一样的方式启动：

```
./bin/hive --service metastore --debug
```

然后使用idea去连接即可。
