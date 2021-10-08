---
title: 解决Pyenv下_tkinter的ModuleNotFoundError问题
tags: 问题调试
categories: 问题调试
abbrlink: 6cce1c32
date: 2020-07-13 00:37:04
---

```
ModuleNotFoundError: No module named '_tkinter'
```
这个问题其实很早我就遇到了，但是一直没有时间去解决，主要是需要重新安装的东西太多，懒得动手，这几天需要做object detection，绕不过去了，不得不解决这个问题了。

首先这个问题出现在mac os 下面，环境为pyenv管理python版本，使用virtualenvwrapper来封装python的运行隔离。

也就是如果使用mac本机自带的python则不会遇到这个问题，通过查看pyenv的代码可以看到：

```python
build_package_auto_tcltk() {
if is_mac && [ ! -d /usr/include/X11 ]; then
if [ -d /opt/X11/include ]; then
if [[ "$CPPFLAGS" != *-I/opt/X11/include* ]]; then
export CPPFLAGS="-I/opt/X11/include $CPPFLAGS"
fi
else
package_option python configure --without-tk
fi
fi
}
```


也就是说，pyenv 使用插件 python-build 进行编译，而上面的代码就是和 tkinter 相关的，因为 macOS 已经不再内置 X11 了，所以导致 python 编译的时候少了 tk 相关的配置。

可以发现解决这个问题可以有两种办法，一种是安装一下 XQuartz 就可以了，另一种是使用 tcl-tk 。

我使用了tcl-tk，如果使用tcl-tk的话，是这样解决的：

* 首先卸载pyenv中间已经安装了的python的版本（正在使用的版本）
* 然后使用brew安装tcl-tk：` brew install tcl-tk` 
* 添加如下环境变量（请注意下面的8.6为安装的对应的tcl-tk的版本）：
``` shell
export PATH="/usr/local/opt/tcl-tk/bin:$PATH"
export LDFLAGS="-L/usr/local/opt/tcl-tk/lib"
export CPPFLAGS="-I/usr/local/opt/tcl-tk/include"
export PKG_CONFIG_PATH="/usr/local/opt/tcl-tk/lib/pkgconfig"
export PYTHON_CONFIGURE_OPTS="--with-tcltk-includes='-I/usr/local/opt/tcl-tk/include' --with-tcltk-libs='-L/usr/local/opt/tcl-tk/lib -ltcl8.6 -ltk8.6'"
```
* 因为写在过python，所以对应的virtualenvwrapper需要重新安装，对应的env环境也需要重新创建。
* 使用` python -m tkinter -c 'tkinter._test()' `检查是否安装成功。


经过上面的处理，基本问题搞定。
