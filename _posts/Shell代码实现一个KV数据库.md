---
title: Shell代码实现一个KV数据库
tags: 编程基础
categories: 编程基础
abbrlink: 353e17a9
date: 2021-11-30 10:20:22
---

从本质上讲，数据库主要是做两件事情：
* 当你给它数据时，它帮你保存数据（存储）；
* 当你查询数据时，它帮你取回数据（检索）；

之前看到一个玩具，说可以用两行shell代码实现一个kv数据库，然后我自己也将其修改了修改，写了下，大概就是这个样子：

```shell
#!/bin/bash
db_set() {
    echo "$1,$2" >> database
}

db_get() {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}

case $1 in
  "db_set")
    db_set $2 $3
    ;;
  "db_get")
    db_get $2
    ;;
  *)
    echo "ERROR"
    ;;
esac
```
确实简陋无比，之前的原创是把function注册到os的env，我觉得还是命令行实在一些，使用方式也比较简单：

```shell
存
./db.sh db_set a dd
./db.sh db_set a dd22
# 取
 ./db.sh db_get a
```
其中
* db_set ，保存键值对数据，即将键值用逗号分隔追加到名为 database 的数据文件；
* db_get ，按键取出对应的数据，即从 database 文件中过滤出最后一行包含指定键的数据；

db_set 操作的性能非常好，因为它只是将数据记录追加到 database 文件的末尾，这通常都很快。
跟 db_set 类似，很多数据库内部也有一个只追加的数据文件，一般叫做操作日志。虽然实际数据库需要考虑更多因素，包括并发控制、磁盘空间重用、错误处理等等，但基本原理都是一样的。
然而，如果数据库中有大量数据，db_get 操作的性能会非常差。因为每次你查询一个键，db_get 都必须扫描整个数据库文件！这是一个典型的O(n)操作，数据库记录数增加一倍，查询开销就会增大一。
