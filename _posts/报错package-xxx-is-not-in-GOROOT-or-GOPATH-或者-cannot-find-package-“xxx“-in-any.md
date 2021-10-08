---
title: '报错package xxx is not in GOROOT or GOPATH 或者 cannot find package “xxx“ in any '
tags: 编程基础
categories: 编程基础
abbrlink: ca53ddc5
date: 2021-04-28 22:23:58
---

## GO111MODULE="off"

go的开发环境和生态相比java这类语言来说一直不是那么的稳定，go的项目构建从早期的基于脚本+环境变量的形式到现在基于go mod，中间也经过了不少时间和步骤。

今天在做go的时候，因为之前乱七八糟的环境引起了一个错误：

```
报错package xxx is not in GOROOT or GOPATH 或者 cannot find package “xxx“ in any 
```
现在就仔细聊一聊

在`GO111MODULE="off"`的条件下，并且写的代码不在$GOPATH/src下，也就是说下面的main.go不在$GOPATH/src目录下面，同时我想要使用另一个module里面的内容，并且这个module不是标准库，或者说不在GOROOT里(一般我们不会修改GOROOT中的内容)

![b1](a0001.png)

运行代码会报错：

```
main.go:4:2: cannot find package "module" in any of:
        /usr/local/go/src/module (from $GOROOT)
        /home/linux/go/src/module (from $GOPATH)

```

解决办法：设置GOPATH：`go env -w GOPATH=~/test`

然后在$GOPATH/src目录下写：

![b1](a0002.png)

go的编译器会在$GOPATH/src下面寻找对应的模块，src下的每一个目录都可以对应一个模块，目录中的目录也可以是一个模块，如果，我们需要访问一个目录中的目录中的模块，比如下图：
![b1](a0003.png)
我们需要调用module中的moduleA模块，只需要使用：`import "module/moduleA" `

## GO111MODULE="on"

在`GO111MODULE="on"`的条件下，我们直接调用写好的模块，如下图所示：

![b1](a0004.png)
会直接报错：

```
main.go:4:2: package module is not in GOROOT (/usr/local/go/src/module)
```

解决方案：
* 第一种方式：设置GO111MODULE="off"，然后像上面的那种方式一样设置GOPATH
* 第二种方式：使用go mod，如下：![b1](a0005.png)


首先我们需要初始化一个go.mod，使用：`go mod init test`


```
如果使用这种方式Goland报错，但是可以进行正常编译，那么可以删除当前目录下的.idea目录然后重启项目即可
```

如果我们想要引用嵌套模块也是一样的：

```
import "test/module/moduleA"
```



![b1](a0006.png)
