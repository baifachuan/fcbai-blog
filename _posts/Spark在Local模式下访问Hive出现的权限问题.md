---
title: Spark在Local模式下访问Hive出现的权限问题
tags: 大数据
categories: 大数据
abbrlink: e1936fe1
date: 2021-12-06 10:04:33
---

有个功能需要借助Spark的DataSet部分能力，主要也是不想再造个轮子，所以在某个Spring Boot工程里面启动了一个Spark Session。
用这个对象来处理一些内容，由于这个场景下Spark需要使用Hive的HMS，也就开启了Hive Enable，有同学反馈在Spark执行任务的出现一个错：

```
The root scratch dir: /tmp/hive on HDFS should be writable. Current permissions
```

粗一看就是没有授权，但是实际在Ranger里面授权了，并且通过对应的用户在命令行进行操作可以看到权限是没有问题的。

这个问题的原因在于在Hive Core Site的配置文件里面配置的：

`hive.exec.scratchdir` 这个值的内容是：`/tmp/hive`。

如果是Spark On Yarn则不会有问题，因为整个filesystem是hdfs的对象，而如果是Spark的local模式，则代表整个spark的上下文是本地jvm，和yarn不会有关系，自然也不会和hdfs有交互。

也就是除非`hive.exec.scratchdir`明确写成`hdfs://192.168.19.59:9000/user/hive/tmp`这样，否则默认整个spark程序会使用本地文件系统而非hdfs。

所以上面出错的目录并非指hdfs没有权限，而是本地的`/tmp/hive`没有权限，最后确认实际便是如此。

对于hive来说，`hive.exec.scratchdir` 这个目录下存储不同阶段的map/reduce的执行计划的目录，同时也存储中间输出结果，默认是/tmp/<user.name>/hive。

所以找到原因了要解决问题就比较容易了：
* 修改hive的配置显示配置成hdfs的目录。
* 在自己的Spark Session的配置里面重写`hive.exec.scratchdir`设置成其他地方。

都可以解决问题。
