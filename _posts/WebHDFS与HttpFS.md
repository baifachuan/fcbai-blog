---
title: WebHDFS与HttpFS
tags: 大数据
categories: 大数据
abbrlink: cc701e04
date: 2021-06-15 19:11:36
---

操作HDFS我们可以有多种办法，使用命令行或者SDK方式操作HDFS文件居多，而SDK的形式大部分是依赖第三方的产品来实现。

实际上HDFS提供基于Web的rest api来对文件进行操作，目前使用最多的基本就是httpfs和webhdfs两个工具。

WebHDFS服务内置在HDFS中，不需额外安装、启动，只需要在hdfs-site.xml打开WebHDFS开关，此开关默认打开，如下：
```
<property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
</property>
```
便可以通过如下方式进行使用了：连接NameNode的50070端口进行文件操作。
```
curl "http://ctrl:50070/webhdfs/v1/?op=liststatus&user.name=root" | python -mjson.tool
```

httpfs也是Hadoop自带，不需要额外安装。默认服务未启动，需要手工启动，启动的时候需要做一些配置，主要为：


* httpfs-site.xml：有配置文件httpfs-site.xml，此配置文件一般保存默认即可，无需修改。
* hdfs-site.xml：需要增加如下配置，其他两个参数名称中的root代表的是启动hdfs服务的OS用户，应以实际的用户名称代替。
```
<property>  
    <name>hadoop.proxyuser.root.hosts</name>  
    <value>*</value>  
</property>  
<property>  
<name>hadoop.proxyuser.root.groups</name>  
    <value>*</value>  
</property>
```

启动命令：

```
sbin/httpfs.sh start
sbin/httpfs.sh stop
```
这样使用：

```
curl "http://ctrl:14000/webhdfs/v1/?op=liststatus&user.name=root" | python -mjson.tool
```
他俩最大的不同，我直接贴官方原文吧：
```
WebHDFS vs HttpFs Major difference between WebHDFS and HttpFs: WebHDFS needs access to all nodes of the cluster and when some data is read it is transmitted from that node directly, whereas in HttpFs, a singe node will act similar to a “gateway” and will be a single point of data transfer to the client node. So, HttpFs could be choked during a large file transfer but the good thing is that we are minimizing the footprint required to access HDFS.
```

已经描述的很清楚了。
