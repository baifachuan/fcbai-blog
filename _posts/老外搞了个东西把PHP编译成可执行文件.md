---
title: 老外搞了个东西把PHP编译成可执行文件
tags: 编程基础
categories: 编程基础
abbrlink: a9c15445
date: 2021-12-03 11:09:44
---

作为把PHP重度使用的俄罗斯人搞了个东西：https://github.com/VKCOM/kphp
能把php编译成可执行文件，编译后的php性能rust，比golang更快。

因为php7.4之后 php有强类型模式了，然后俄罗斯人把强类型php编译成c++利用c++的RAII代替GC保证内存安全，再把c++编译成二进制可执行文件。

这性能比以前facebook hack那种由于要兼容弱类型php代码方式强多了。

作者还给了个使用kphp编写的sdl游戏：https://github.com/zjsxwc/kphp-game

在性能上，计算第 300000 个素数的值时，使用kphp耗时5.3秒，接近rust耗时4.8秒，几乎是golang耗时10.4秒的两倍。

kphp算第30万个素数源码：https://pastebin.ubuntu.com/p/FfZx5bjR3f/
golang算第30万个素数源码：https://paste.ubuntu.com/p/MSNWkJTvz6/
rust算第30万个素数源码：https://paste.ubuntu.com/p/dMpwJ6HdY9/

也就老外舍得这样折腾，不过既然都这样了，干嘛不直接上手c++，也不差这点语法糖的诱惑。

