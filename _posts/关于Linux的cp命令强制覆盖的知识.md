---
title: 关于Linux的cp命令强制覆盖的知识
tags: 编程基础
categories: 编程基础
abbrlink: fabcae5e
date: 2021-12-09 15:11:26
---

如果对linux日常使用比较多的同学肯定会发现一个现象，就是在使用`cp`命令的时候，即使添加了 `-rf` 参数强制覆盖复制时,系统仍然会提示让你一个个的手工输入 `y` 确认覆盖，所添加的 `rf` 参数是不起作用的。

原因：
```
cp命令被系统设置了别名，相当于cp='cp -i'。
```

通过alias命令查询系统的值也可也看到：
```shell
[root@iZbp10hm1pb36ma9dme4ciZ blog]# alias
alias acme.sh='/root/.acme.sh/acme.sh'
alias cp='cp -i'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias l.='ls -d .* --color=auto'
alias ll='ls -l --color=auto'
alias ls='ls --color=auto'
alias mv='mv -i'
alias rm='rm -i'
alias which='(alias; declare -f) | /usr/bin/which --tty-only --read-alias --read-functions --show-tilde --show-dot'
alias xzegrep='xzegrep --color=auto'
alias xzfgrep='xzfgrep --color=auto'
alias xzgrep='xzgrep --color=auto'
alias zegrep='zegrep --color=auto'
alias zfgrep='zfgrep --color=auto'
alias zgrep='zgrep --color=auto'
```

也就是 `cp` 命令，虽然没有添加任何参数，但系统默认会在我们使用 `cp` 命令时自动添加 `-i` 参数，参数的描述如下：

```
-i, --interactive
          prompt before overwrite
```

`-i` 即交互的缩写方式，也就是在使用 `cp` 命令作文件覆盖操作之前，系统会要求确认提示。

这个是系统的一个保险措施，如果有很多文件要复制，觉得一个一个输入`y` 确认麻烦的话，可以使用如下方法解决:

### 方法一：使用原生的cp命令

```shell
/bin/cp -rf xxxx
```

### 方法二：取消别名

```shell
unalias cp
```

这时再用 `cp -rf` 复制文件时，就不会要求确认。

最好在使用完后再把别名加回来：

```shell
alias cp='cp -i'
```

总体来说还是建议多一步确认，避免减少麻烦，或者自己写一个脚本先把原来的备份，删除，再cp，安全第一。
