---
title: Spark Thrift Server与Hive Server2 的区别详解
tags: 大数据
categories: 大数据
abbrlink: 9bc3a6a0
date: 2021-09-02 14:37:34
---

### STS与HS2的区别

最近在做Hadoop集群相关的代理服务，涉及到对thrift server的一些处理，主要是HiveServer2和Spark Thrift Server，两者其实都是提供一个常驻的SQL服务，用来对外提供高性能的SQL引擎能力，不过两者又有些偏差，主要是HS2是独立的Server，可组成集群，而STS是运行在Yarn上的常驻服务，因此会受到很多局限。

STS和HS2真可谓是一个复杂的历史，基本上是一个“忒修斯船”(Ship of Theseus)的故事。最开始的时候，Spark SQL的代码几乎全部都是Hive的照搬，随着时间的推移，Hive的代码被逐渐替换，直到几乎没有原始的Hive代码保留，具体的内容可以参考：

参考：https://en.wikipedia.org/wiki/Ship_of_Theseus

Spark最开始打包的是Shark和SharkServer(Spark和Hive的结合体)。那个时候，这个结合体包含了大量的Hive代码。SharkServer就是Hive，它解析HiveQL，在Hive中进行优化，读取Hadoop的输入格式，到最后Shark甚至在Spark引擎上运行Hadoop风格的MapReduce任务。这一点在当时来说其实很酷的，因为它提供了一种无需进行任何编程就能使用Spark的方式，HQL做到了。不幸的是，MapReduce和Hive并不能完全融入Spark生态系统，2014年7月，社区宣布Shark的开发在Spark1.0的时终止，因为Spark开始转向更多Spark原生的SQL表达式。同时社区将重心转向原生的Spark SQL的开发，并且对已有的Hive用户提供过渡方案Hive on Spark来进行将Hive作业迁移到Spark引擎执行，可以参考：

https://github.com/amplab/shark/wiki/Shark-User-Guidehttps://databricks.com/blog/2014/07/01/shark-spark-sql-hive-on-spark-and-the-future-of-sql-on-spark.html


Spark引入SchemaRDDs，Dataframes和DataSets来表示分布式数据集。有了这些，一个名为Catalyst的全新Spark原生优化引擎引入到Spark，它是一个Tree Manipulation Framework，为从GraphFrames到Structured Streaming的所有查询优化提供依据。Catalyst的出现意味着开始丢弃MapReduce风格的作业执行，而是可以构建和运行Spark优化的执行计划。此外，Spark发布了一个新的API，它允许我们构建名为“DataSources”的Spark-Aware接口。DataSources的灵活性结束了Spark对Hadoop输入格式的依赖（尽管它们仍受支持）。DataSource可以直接访问Spark生成的查询计划，并执行谓词下推和其他优化。

Hive Parser开始被Spark Parser替代，Spark SQL仍然支持HQL，但语法已经大大扩展。Spark SQL现在可以运行所有TPC-DS查询，以及一系列Spark特定的扩展。（在开发过程中有一段时间你必须在HiveContext和SqlContext之间进行选择，两者都有不同的解析器，但我们不再讨论它了。今天所有请求都以SparkSession开头）。现在Spark几乎没有剩下Hive代码。虽然Sql Thrift Server仍然构建在HiveServer2代码上，但几乎所有的内部实现都是完全Spark原生的。

不过Spark Thrift Server的接口和协议都和HiveServer2完全一致，因此我们部署好Spark Thrift Server后，可以直接使用hive的beeline访问Spark Thrift Server执行相关语句，Spark Thrift Server的目的也只是取代HiveServer2，并不是取代整个Hive的SQL引擎体系，因此它依旧可以和Hive Metastore进行交互，获取到hive的元数据。

还有就是Thrift Server也是Spark提供的一种JDBC/ODBC访问Spark SQL的服务，它是基于Hive1.2.1的HiveServer2实现的，只是底层的SQL执行改为了Spark，同时是使用spark submit启动的服务。同时通过Spark Thrift JDBC/ODBC接口也可以较为方便的直接访问同一个Hadoop集群中的Hive表，通过配置Thrift服务指向连接到Hive的metastore服务即可。

并且Spark Thrift Server大量复用了HiveServer2的代码，HiveServer2的架构主要是通过ThriftCLIService监听端口，然后获取请求后委托给CLIService处理。CLIService又一层层的委托，最终交给OperationManager处理。OperationManager会根据请求的类型创建一个Operation的具体实现处理。比如Hive中执行sql的Operation实现是SQLOperation。

Spark Thrift Server做的事情就是实现自己的CLIService——SparkSQLCLIService，接着也实现了SparkSQLSessionManager以及SparkSQLOperationManager。另外还实现了一个处理sql的Operation——SparkExecuteStatementOperation。这样，当Spark Thrift Server启动后，对于sql的执行就会最终交给SparkExecuteStatementOperation了。

Spark Thrift Server其实就重写了处理sql的逻辑，其他的请求处理就完全复用HiveServer2的代码了。比如建表、删除表、建view等操作，全部使用的是Hive的代码。


对于STS中的SQL执行，Spark Thrift Server的启动其实是通过spark-submit将HiveThriftServer2提交给集群执行的。因此执行start-thriftserver.sh时可以传入spark-submit的参数表示提交HiveThriftServer2时的参数。另外，因为HiveThriftServer2必须要在本地运行，所以提交时的deployMode必须是client，如果设置成cluster会报错。HiveThriftServer2运行起来后，就等于是一个Driver了，这个Driver会监听某个端口，等待请求，也就是这样：

![image](001.jpeg)

所以HiveThriftServer2程序运行起来后就等于是一个长期在集群上运行的spark application，通过yarn或者spark history server页面我们都可以看到对应的任务。

既然HiveThriftServer2就是Driver，那么运行SQL就很简单了。Spark Thrift Server收到请求后最终是交给SparkExecuteStatementOperation处理，SparkExecuteStatementOperation拿到SQLContext，然后调用SQLContext.sql()方法直接执行用户传过来的sql即可。后面的过程就和我们直接写了一个Main函数然后通过spark-submit提交到集群运行是一样的，一张图总结一下STS和HS2的区别：

![image](0002.png)

![image](003.png)

Hive Server2 在大部分的Hadoop发行版中自然就会启动，而STS这不会，因为考虑到其不稳定性，都需要自行维护，不过使用方式也很简单：
```
命令为：
./sbin/start-thriftserver.sh

参数信息：
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...

或者
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
```

然后则可以使用beeline进行访问：

```
./bin/beeline

beeline> !connect jdbc:hive2://localhost:10000

beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>

./bin/spark-sql

```

### Spark Thrift Server的优点
* 在大部分场景下，性能要比Hive on spark好，而且好很多
* SparkSQL的社区活跃度也很高，基本每月都会发布一个版本，因此性能还会不断提高

### Spark Thrift Server的缺点
* 因为HiveThriftServer2是以Driver的形式运行在集群的。因此它能使用的集群资源就和单个Application直接挂钩。如果spark集群没开启动态资源，那么Spark Thrift Server能得到的资源就始终都是固定的，这时候设置太大也不好，设置太小也不好。即使开启了动态资源，一般集群都会设置maxExecutor，这时还是无法很好的利用集群的所有资源。如果将集群所有的资源都分配给了这个Application，这样像yarn、mesos这些资源调度器就完全没有存在的意义了…因此，单就这一点，Spark Thrift Server就不是一个合格的企业级解决方案。
* 从https://issues.apache.org/jira/browse/SPARK-11100 官方的回答来看，spark官方对于Spark Thrift Server这套解决方案也不是很满意。这也可以理解，毕竟Spark Thrift Server只是对HiveServer2进行的一些小改造。
* Spark Thrift Server目前还是基于Hive的1.2版本做的改造，因此如果MetaStore的版本不是1.2，那么也可能会有一些兼容性的潜在问题。
* 不支持用户模拟，即Thrift Server并不能以提交查询的用户取代启动Thrift Server的用户来执行查询语句，具体对应到Hive的hive.server2.enable.doAs参数不支持，参考：https://issues.apache.org/jira/browse/SPARK-5159https://issues.apache.org/jira/browse/SPARK-11248https://issues.apache.org/jira/browse/SPARK-21918
* 因为上述第一点不支持用户模拟，导致任何查询都是同一个用户，所有没办法控制Spark SQL的权限，所以当你需要Spark SQL也要做权限控制的时候，只有自己去实现ranger的plugin了。
* 单点问题，所有Spark SQL查询都走唯一一个Spark Thrift节点上的同一个Spark Driver，任何故障都会导致这个唯一的Spark Thrift节点上的所有作业失败，从而需要重启Spark Thrift Server。
* 并发差，上述第三点原因，因为所有的查询都要通过一个Spark Driver，导致这个Driver是瓶颈，于是限制了Spark SQL作业的并发度。

不过所以网易自己做了一个Thrift服务取名Kyuubi，有兴趣的可以去看看：https://github.com/apache/incubator-kyuubi 目前我也正在打算使用它。
