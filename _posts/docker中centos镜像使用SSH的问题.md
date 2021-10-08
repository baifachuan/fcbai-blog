---
title: docker中centos镜像使用SSH的问题
tags: 编程基础
categories: 编程基础
abbrlink: 4f623783
date: 2021-05-13 21:30:42
---

今天使用Docker的centos的镜像的时候，需要使用SSH进行免密登陆配置，因为我不想再开一台机器，所以直接打算直接使用当前的镜像，但是在ssh localhost的时候碰到这个问题：

```
centos7:/var/lib/ambari-server/resources/stacks/HDP/2.6 # ssh localhost
ssh: connect to host localhost port 22: Cannot assign requested address
```

明显是因为sshd没有启动，首先需要检查一下ssh有没有装，没有的话使用：
```
yum install openssh-server
yum install openssh-clients
```
安装一下，先。在用用ps -ef验证一下sshd有没有启动，由于是docker里面的centos，所以service和systemctl都不好用。

尝试手动运行/usr/sbin/sshd，报如下错误：

```
Could not load host key: /etc/ssh/ssh_host_rsa_key
Could not load host key: /etc/ssh/ssh_host_ecdsa_key
Could not load host key: /etc/ssh/ssh_host_ed25519_key
sshd: no hostkeys available -- exiting.
```

手动执行/usr/sbin/sshd-keygen -A

再执行/usr/sbin/sshd成功。

为了免密码本机跳本机，执行如下命令：
```
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

至此，执行ssh localhost就能成功。

因为我是想在docker里面安装源码编译的ambari，所以解决上面ssh的问题后，又碰到如下问题：

```
==========================
Creating target directory...
==========================

Command start time 2021-05-11 12:51:27

Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
SSH command execution finished
host=localhost, exitcode=255
Command end time 2021-05-11 12:51:27

ERROR: Bootstrap of host localhost fails because previous action finished with non-zero exit code (255)
ERROR MESSAGE: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).

STDOUT: 
Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
```

这是因为使用的文件不对，默认使用private key，误用使用成了public key。

