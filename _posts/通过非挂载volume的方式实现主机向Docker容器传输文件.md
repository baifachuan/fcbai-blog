---
title: 通过非挂载volume的方式实现主机向Docker容器传输文件
tags:
- 大数据
- 元数据
  categories: 大数据
  date: 2021-11-22 16:45:21
---

将本本机的文件传输到Docker容器中，常见的可以是volume的形式将本机的某个目录挂载至容器中，但是这样的动作对于已经启动的容器来说会很麻烦，需要重新启动修改容器，还有一种简单的办法可以实现：

通过ps获取容器的ID或者名字：

```shell
docker ps -a
```
找到自己的容器：

```shell
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
a0038ddeffee        mysql:latest        "docker-entrypoint.s…"   About an hour ago   Up About an hour    0.0.0.0:3306->3306/tcp, 33060/tcp   mysql_db_1
```
通过容器名或者ID找到磁盘信息：
```shell
docker inspect -f '{{.Id}}' mysql_db_1

a0038ddeffee0e442c710ebaeeb19ddd38649ac1e9a28a8d7cfc4c82e94eb5e8
```

有了上面这一串符号后，通过cp进行文件传输：

```shell
 docker cp ./test_db a0038ddeffee0e442c710ebaeeb19ddd38649ac1e9a28a8d7cfc4c82e94eb5e8:/root
```

进入容器可以看到对应的文件已经成功上传：
```shell
root@a0038ddeffee:/# ls /root/
test_db
```
上传成功。
