---
title: Spark开启动态资源分配
tags: 编程基础
categories: 编程基础
abbrlink: c5ecc311
date: 2022-02-28 18:49:12
---

Spark开启动态资源分配的步骤比较简单，首先确保spark conf中存在：

```
spark.dynamicAllocation.enabled     true
spark.shuffle.service.enabled       true
```

也可也通过`spark.dynamicAllocation.minExecutors 1`设置最小的个数，由于动态资源分配依赖shuffle，所以需要修改yarn的配置`yarn-site.xml`，确保：

```
<configuration>
    <property>
      <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
      <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
    　　<name>yarn.nodemanager.aux-services</name>
    　　<value>spark_shuffle,mapreduce_shuffle</value>
    </property>
    <property>
    　　<name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
    　　<value>org.apache.spark.network.yarn.YarnShuffleService</value>
    </property>
    
    <property>
    　　<name>yarn.nodemanager.aux-services.spark_shuffle.classpath</name>
    　　<value>spark-3.1.2-bin-hadoop3.2/yarn/*</value>
    </property>
</configuration>
```

yarn会开启一个监听7337端口的服务，用来充当shuffle service，如果你中间某个参数缺失，可能会碰到和我一样的问题：
```
2022-02-28 18:29:33,746 WARN cluster.YarnScheduler: Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient resources
```

yarn的日志会出错：

```
spark_shuffle does not exist
```

这就是因为shuffle service不存在，这个错动态分配会出现，但是并不是出现这个错就是动态分配引起的，更多的问题可以看我之前写的文章：https://baifachuan.com/posts/cf1277b5.html
