---
title: 不同的项目使用不同的git配置
tags: 编程基础
categories: 编程基础
abbrlink: c90f1613
date: 2022-02-16 09:57:47
---

日常使用git时，一般会有全局配置文件的.gitconfig，所有repo会默认使用这个配置。如果需要特殊配置某个repo，只需要修改repo里的.git/config文件即可。但如果需要修改repo数量变多就容易忘，导致提交错误author的commit。

例如，公司repo与个人repo，想使用不同的user.name与user.email，简单操作就是，修改个人repo中.git/config中的配置
```
[user]
name = name1
email = name1@example.com
```
或执行以下语句，修改配置（两种方法结果来看本质上是一样的）
```
git config user.name name1
git config user.email name1@example.com
```
这样就会覆盖全局配置，但repo一多，这种方法就显得效率很低。


个人会用不同的文件夹区分公司与个人的repo，这样就可以通过git conditional include，给不同目录下的repo，设置不同的git config。假设公司repo，会放在work目录下。

新建一个.gitconfig-work，内容如下
```
[user]
name = name2
email = name2@example.com
```
打开全局配置.gitconfig文件，添加目录配置

```
[includeIf "gitdir:~/work/"]
path = .gitconfig-work
```
这样work下的新repo，都会使用.gitconfig-work中的配置覆盖全局配置，从而达到不同目录使用不同的git config


对于work目录下已存在的repo，可执行以下命令查看.gitconfig-work中配置是否生效

```
git config --show-origin --get user.name
git config --show-origin --get xxxx
```

参考：
https://git-scm.com/docs/git-config#_conditional_includes
https://stackoverflow.com/questions/43919191/git-2-13-conditional-config-on-windows
