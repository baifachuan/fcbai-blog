---
title: Spark的内存管理模块-Tungsten
tags: 大数据
categories: 大数据
abbrlink: 8b39f1a3
date: 2021-12-21 17:31:59
---

对于JVM框架来说，内存管理都逃脱不了GC，但是当一个框架过于复杂，或者优化到达极限的时候，就会考虑JVM的GC机制是否符合当前的运行模式，Tungsten模块就是Spark发现利用JVM本身的内存管理机制。
无法高效的对内容进行管理，主要体现在：

### Java对象空间开销大
以UTF-8编码的字符串abcd为例，仅存储字符串信息需要4 bytes，然而使用Java的String存储则需要48 bytes，使用jol打印出String的内部占用(参考Java Object Layout(jol) )，如：
```
OFFSET  SIZE     TYPE DESCRIPTION                       VALUE 
  0    12      (object header)                           N/A 
 12     4      char[] String.value                       N/A 
 16     4      int String.hash                           N/A 
 20     4      int String.hash32                         N/A 
Instance size: 24 bytes 
```

使用jol打印出abcd占用空间大小，如下：

```
java.lang.String@213214d1d footprint: 
 COUNT       AVG       SUM   DESCRIPTION 
     1        24        24   [C 
     1        24        24   java.lang.String 
     2                  48   (total)
```

### GC带来的时间开销
full gc会stop the world，增加程序耗时，甚至可能出现假死情况，并且这种情况没有日志输出，给问题排查带来一定难。

基于以上考虑，Tungsten项目优化内存的使用，使用off-heap方式管理内存，不再依赖JVM，避免了存储的额外开销及GC的影响，Tungsten主要工作包含以下三个方面：

* Memory Management and Binary Processing: leveraging application semantics to manage memory explicitly and eliminate the overhead of JVM object model and garbage collection。
* Cache-aware computation: algorithms and data structures to exploit memory hierarchy。
* Code generation: using code generation to exploit modern compilers and CPUs。

解释如下：

* Memory Management and Binary Processing: off-heap管理内存，降低对象的开销和消除JVM GC带来的延时。
* Cache-aware computation: 优化存储，提升CPU L1/ L2/L3缓存命中率。
* Code generation: 优化Spark SQL的代码生成部分，提升CPU利用率。

Tungsten设计了一套内存管理机制，而不再是交给JVM托管，Spark的operation直接使用分配的binary data而不是JVM objects。 


