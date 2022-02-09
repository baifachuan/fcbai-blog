---
title: HiveServer2是如何实现HA的
tags: 大数据源码
keywords: 'Hive高可用, Hive JDBC,  HiveServer2是如何实现HA的, Hive HA'
description: 'HiveServer2是如何实现HA的,Hive JDBC流程, Hive源码调试'
categories: 大数据源码
abbrlink: d103c638
date: 2022-01-19 13:55:07
---

HiveServer2实现HA有个地方的重点实现，一个是HiveServer2本身的服务器相关的信息，HiveServer2会把自己的服务信息注册到ZK中，在ZK中存在这样的目录结构：

```shell
[zk: localhost:2181(CONNECTED) 1] ls /hiveserver2
[serverUri=0.0.0.0:10001;version=2.1.1-cdh6.3.2;sequence=0000000000]
```

可以看到是在`hiveserver2`这个节点下创建了其他节点，节点的名称是多个字段拼接的内容，其次这个节点的内容，也就是data是serverUri也就是0.0.0.0:10001这个值。

当服务器把自己的信息注册到ZK中之后，服务端的工作就完成了，剩下的就是客户端如何从ZK中去解析信息，并且去连接了。

Hive 的JDBC自身实现了连接ZK去获取服务器信息，从而实现高可用的能力，JDBC的实现思路是：

客户端使用`jdbc:hive2://localhost:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2`这样的连接信息去连接服务器。
JDBC的代码会解析这个内容，从中提取出ZK的信息，去查询获取到所有注册在hiveserver2下的服务器信息。也就是ZooKeeperHiveClientHelper代码中的getNextServerUriFromZooKeeper，使用CuratorFramework客户端。
从ZK中去查询到对应的节点信息：

```java
try {
  zooKeeperClient.start();
  serverHosts = zooKeeperClient.getChildren().forPath("/" + zooKeeperNamespace);
  // Remove the znodes we've already tried from this list
  serverHosts.removeAll(connParams.getRejectedHostZnodePaths());
  if (serverHosts.isEmpty()) {
    throw new ZooKeeperHiveClientException(
        "Tried all existing HiveServer2 uris from ZooKeeper.");
  }
  // Now pick a host randomly
  serverNode = serverHosts.get(randomizer.nextInt(serverHosts.size()));
  connParams.setCurrentHostZnodePath(serverNode);
  String serverUri =
      new String(
          zooKeeperClient.getData().forPath("/" + zooKeeperNamespace + "/" + serverNode),
          Charset.forName("UTF-8"));
  LOG.info("Selected HiveServer2 instance with uri: " + serverUri);
  return serverUri;
} 
```

部分代码如上所示，不过这里有一个小地方就是，JDBC客户端代码在获取到ZK下注册的节点信息后，会先把IP等于本地的排掉，也就是：

```java
serverHosts.removeAll(connParams.getRejectedHostZnodePaths());
```

这一行代码，而这个的添加是在：

```java
connParams.getRejectedHostZnodePaths().add(connParams.getCurrentHostZnodePath());
```

* 整体HiveServer2实现HA的思路就非常简单了：
* 服务器先注册自己的信息到ZK，作为临时Node。
* JDBC客户端连接ZK进行遍历。
* JDBC排除本地的Host的Node。
* JDBC随机从剩下的List里面选择一个Server出来进行连接。

因此HA至少需要2个Server注册在ZK，且不可与客户端调用的host相同。
