---
title: 使用go语言进行交叉编译的时候遇到的一些问题
tags: 编程基础
categories: 编程基础
abbrlink: 4862a3b1
date: 2021-08-17 23:39:18
---

最近一直在搞go的开发，开发的程序会部署在一套hadoop集群中，我本地的开发环境是使用macos进行代码编写，然后放到virtualbox中的hadoop集群去运行，而vbox中的机器又是linux。

这就导致我需要使用交叉编译在macos上编译出linux的binary，再去虚拟机中测试，对于go来说默认就支持交叉编译，毕竟方便的多环境编译也是go的优势之一，只需要通过如下配置：

```
CGO_ENABLED=1 go build -o BIN_NAME
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -o BIN_NAME
```
如上第一行是编译当前机器的二进制，第二行代码是编译linux的二进制，不过我在编译的时候碰到一个问题：

```
linux_syscall.c:67:13: error: implicit declaration of function 'setresgid' is invalid in C99 [-Werror,-Wimplicit-function-declaration]
linux_syscall.c:67:13: note: did you mean 'setregid'?
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/unistd.h:593:6: note: 'setregid' declared here
linux_syscall.c:73:13: error: implicit declaration of function 'setresuid' is invalid in C99 [-Werror,-Wimplicit-function-declaration]
linux_syscall.c:73:13: note: did you mean 'setreuid'?
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/unistd.h:595:6: note: 'setreuid' declared here
```

这是因为os本身系统库不兼容的问题，首先因为我是在macos上编译linux的版本，所以需要安装：

```
brew install FiloSottile/musl-cross/musl-cross
```

如果你是在macos上编译运行在win下的exe，那么需要安装：

```
brew install mingw-w64
==> Downloading https://homebrew.bintray.com/bottles/gmp-6.2.1.big_sur.bottle.tar.gz
==> Downloading from https://d29vzk4ow07wi7.cloudfront.net/6a44705536f25c4b9f8547d44d129ae3b3657755039966ad2b86b821e187c32c?response-content-dispositio
```

同时编译的参数也需要调整如下：

```
CGO_ENABLED=1 GOOS=linux  GOARCH=amd64  CC=x86_64-linux-musl-gcc  CXX=x86_64-linux-musl-g++ go build -o BIN_NAME
```

这样就能编译出需要的可以运行的二进制了，不过在运行的时候，linux可能会遇到这个错误：

```
/lib/ld-musl-x86_64.so.1: bad ELF interpreter: No such file or directory
```

这是因为linux没有安装musl的支持，如下安装：

```
wget https://copr.fedorainfracloud.org/coprs/ngompa/musl-libc/repo/epel-7/ngompa-musl-libc-epel-7.repo -O /etc/yum.repos.d/ngompa-musl-libc-epel-7.repo
yum install -y musl-libc-static
```

musl是一个os级别的系统库，像glibc是大家见的最多的，musl也是其中之一，各有所长，上面编译出的是使用的musl接口的二进制，因此需要安装支持，其他几类的介绍如下：

* Glibc glibc = GNU C Library 是GNU项（GNU Project）目，所实现的 C语言标准库（C standard library）。 目前，常见的桌面和服务器中的GNU/Linux类的系统中，都是用的这套C语言标准库。 其实现了常见的C库的函数，支持很多种系统平台，功能很全，但是也相对比较臃肿和庞大。
* uClibc 一个小型的C语言标准库，主要用于嵌入式。 其最开始设计用于uClinux（注：uClinux不支持MMU），因此比较适用于微处理器中。 对应的，此处的u意思是μ，Micro，微小的意思。 uClibc的特点： 
    1. (1)uClibc比glibc要小很多。 
    2. (2)uClibc是独立的，为了应用于嵌入式系统中，完全重新实现出来的，和glibc在源码结构和二进制上，都不兼容。
* EGLIBC EGLIBC = Embedded GLIBC EGLIBC是，（后来）glibc的原创作组织FSF所（新）推出的，glibc的一种变体，目的在于将glibc用于嵌入式系统。 EGLIBC的目标是： 
    1. (1)保持源码和二进制级别的兼容于Glibc 源代码架构和ABI层面兼容 如果真正实现了这个目标，那意味着，你之前用glibc编译的程序，可以直接用eglibc替换，而不需要重新编译。 这样就可以复用之前的很多的程序了。 
    2. (2)降低(内存)资源占用/消耗 
    3. (3)使更多的模块为可配置的（以实现按需裁剪不需要的模块） 
    4. (4)提高对于交叉编译(cross-compilation)和交叉测试(cross-testing)的支持 Eglibc的最主要特点就是可配置，这样对于嵌入式系统中，你所不需要的模块，比如NIS，locale等，就可以裁剪掉，不把其编译到库中，使得降低生成的库的大小了。
* Musl-libc C语言标准库Musl-libc项目发布了1.0版。Musl是一个轻量级的C标准库，设计作为GNU C library (glibc)、 uClibc或Android Bionic的替代用于嵌入式操作系统和移动设备。它遵循POSIX 2008规格和 C99 标准，采用MIT许可证授权，使用Musl的Linux发行版和项目包括sabotage，bootstrap-linux，LightCube OS等等。


备注一个备忘录，因为我之前一直是macos的低版本升级上来的，所以默认使用的是bash shell，不过目前比较推荐的是zsh，macos内置的也是zsh，插件也非常丰富，可以使用如下命令去自由切换：

```
查看shell类型
可以执行命令echo $SHELL，先查看终端类型。
bash: /bin/bash
zsh: /bin/zsh
shell类型切换命令
切换到bash 输入命令：chsh -s /bin/bash
切换到zsh 输入命令：chsh -s /bin/zsh
```

当然可以修改系统级别的，也可以修改当前session级别的配置。
