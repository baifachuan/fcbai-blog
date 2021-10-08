---
title: 使用 Let's Encrypt 免费的让自己的域名变成HTTPS
tags: 编程基础
categories: 编程基础
abbrlink: c0d4f39e
date: 2020-06-25 13:12:09
---

自己服务器从2012年就开始折腾，曾经作为发烧友也做了非常多的折腾，也算各种东西都折腾了一遍，曾经输出的文章也到达了数千篇，期间遇到过服务器被黑，网站挂上了赌博的宣传，于是开始研究别人如何入侵进行防护，和对方斗智斗勇，也经历过使用第三方的插件，由于插件问题，模棱两可的描述，在对文章进行迁移的时候，把整个数据库给删掉了，从此养成了万事先检查代码的习惯。

后来慢慢的写的少了，想的多了，做的多了，即便偶尔写点东西，似乎都觉得不甚满意，于是不再像曾经那样随便各种copy书写文章，偶尔写点东西，也输出在了公司的洞见和官方账号上，于是也就懒得自己折腾自己的博客了。


最近以为服务器活动于是又搞了服务器，和域名，图省事，简单做了个博客，个人比较喜欢hackernews的布局和风格，所以自己写了这个UI，模仿着来，比较简单简约。

因为服务器开通了，域名有了，所以打算将其新增HTTPS证书，之前也是使用Let's Encrypt，不过是使用的原始命令操作的，现在发现有其更加简单的方式，所以也算记录下，毕竟后面每间隔三个月需要重新颁发一次，还需要再操作。

![image](b001.jpg)

Let's Encrypt 是一个由非营利性组织 互联网安全研究小组（ISRG）提供的免费、自动化和开放的证书颁发机构（CA）。

简单的说，借助 Let's Encrypt 颁发的证书可以为我们的网站免费启用 HTTPS(SSL/TLS) 。

Let's Encrypt免费证书的签发/续签都是脚本自动化的，官方提供了几种证书的申请方式方法，点击此处 快速浏览。

官方推荐使用 Certbot 客户端来签发证书，这种方式可参考文档自行尝试，不做评价。

我这里直接使用第三方客户端 acme.sh 申请，据了解这种方式可能是目前 Let's Encrypt 免费证书客户端最简单、最智能的 shell 脚本，可以自动发布和续订 Let's Encrypt 中的免费证书。

#### 安装 acme.sh

安装很简单，一条命令：

```shell
curl https://get.acme.sh | sh
```
整个安装过程进行了以下几步，了解一下即可：

* 把 acme.sh 安装到当前用户的主目录$HOME下的.acme.sh文件夹中，即~/.acme.sh/，之后所有生成的证书也会放在这个目录下；
* 创建了一个指令别名`alias acme.sh=~/.acme.sh/acme.sh`，这样我们可以通过`acme.sh`命令方便快速地使用 `acme.sh` 脚本；
* 自动创建`cronjob`定时任务, 每天 0:00 点自动检测所有的证书，如果快过期了，则会自动更新证书。

安装命令执行完毕后，执行`acme.sh --version`确认是否能正常使用acme.sh命令。

```shell
https://github.com/Neilpang/acme.sh
v2.7.9
```

如有版本信息输出则表示环境正常；如果提示命令未找到，执行`source ~/.bashrc`命令重载一下环境配置文件。

整个安装过程不会污染已有的系统任何功能和文件，所有的修改都限制在安装目录`~/.acme.sh/`中。


#### 生成证书

据 acme.sh 官方文档介绍，其实现了 acme 协议支持的所有验证协议，一般有两种方式验证：http 和 dns 验证。

也就是我们有两种选择签发证书，这里我直接选择 http 验证方式，另外一种方式本篇不做介绍，可参考文档自行尝试。

签发证书也很简单，一条命令：

```shell
acme.sh --issue -d baifachuan.com -d www.baifachuan.com -w 服务器上安装网站的绝对路径
```

简单解释下这条命令涉及的几个参数：

* --issue是 acme.sh 脚本用来颁发证书的指令；
* -d是--domain的简称，其后面须填写已备案的域名；
* -w是--webroot的简称，其后面须填写网站的根目录。

![image](b002.png)
![image](b003.png)

生成的证书放在了`/root/.acme.sh/baifachuan.com`目录。另外，可以通过下面两个常用acme.sh命令查看和删除证书：

```shell
# 查看证书列表
acme.sh --list 

# 删除证书
acme.sh remove <SAN_Domains>
至此，证书就下载成功。
```

#### 安装证书

我的站点是由 Nginx 承载的，所以本节内容重点记录如何将证书安装到 Nginx，其他 webserver 请参考 acme.sh 文档自行实践。废话不多说，进入本节正题。

上一小节，生成的证书放在了/root/.acme.sh/baifachuan.com目录，因为这是 acme.sh 脚本的内部使用目录，而且目录结构可能会变化，所以我们不能让 Nginx 的配置文件直接读取该目录下的证书文件。

正确的做法就是使用--installcert命令，指定目标位置，然后证书文件会被 copy 到相应的位置。

一条命令即可解决：

```shell
acme.sh  --installcert -d baifachuan.com \
         --key-file /etc/nginx/ssl/baifachuan.com.key \
         --fullchain-file /etc/nginx/ssl/fullchain.cer \
         --reloadcmd "service nginx force-reload"
```

![image](b004.png)

这里我将证书放到了/etc/nginx/ssl/目录下。最后一步就是，修改 Nginx 配置文件启用 ssl，修改完成后需要重启下 Nginx，这一块不再详述。



#### 更新证书
目前 Let's Encrypt 的证书有效期是90天，时间到了会自动更新，您无需任何操作。 今后有可能会缩短这个时间， 不过都是自动的，不需要关心。

但是，您也可以强制续签证书：

```shell
acme.sh --renew -d example.com --force
```

#### 更新 acme.sh

目前由于 acme 协议和 letsencrypt CA 都在频繁的更新, 因此 acme.sh 也经常更新以保持同步。

升级 acme.sh 到最新版：

```shell
acme.sh --upgrade
```
如果不想手动升级,，可以开启自动升级：

```shell
acme.sh  --upgrade  --auto-upgrade
```
也可以随时关闭自动更新：

```shell
acme.sh --upgrade  --auto-upgrade  0
```
