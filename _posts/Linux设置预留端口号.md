---
title: Linux设置预留端口号
tags: 编程基础
categories: 编程基础
abbrlink: fecbcdd3
date: 2022-02-18 14:38:55
---

预留端口的意思是提前把一部分端口预留下来，避免程序在使用随机端口，或者指定端口的时候发生冲突。

在Linux里面可以通过区间和端口进行设置。

本地临时端口区间配置(2个数: start, end)
```
cat /proc/sys/net/ipv4/ip_local_port_range
32768   61000
```

预留端口配置(可支持逗号分隔的多个数字，比如10000, 10005-10010)

```
cat /proc/sys/net/ipv4/ip_local_reserved_ports
```

有2种方法可以进行配置:

* 修改临时端口范围 ip_local_port_range，因为一个程序的端口问题修改一个机器的临时端口范围，明显减少了临时端口的使用量。代价大。
* 修改预留端口ip_local_reserved_ports，即使没有发生冲突也可以预先设置，防止后续端口被占用。

贴一下参考链接里面的英文：

```
ip_local_reserved_ports - list of comma separated ranges
Specify the ports which are reserved for known third-party
applications. These ports will not be used by automatic port
assignments (e.g. when calling connect() or bind() with port
number 0). Explicit port allocation behavior is unchanged.

The format used for both input and output is a comma separated
list of ranges (e.g. "1,2-4,10-10" for ports 1, 2, 3, 4 and
10). Writing to the file will clear all previously reserved
ports and update the current list with the one given in the
input.
```

参考:
(1) http://www.ttlsa.com/linux/reserved-port-to-avoid-occupying-ip_local_reserved_ports/
