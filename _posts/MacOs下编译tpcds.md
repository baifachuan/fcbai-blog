---
title: MacOs下编译tpcds
tags: 编程基础
categories: 编程基础
abbrlink: 34f97a60
date: 2022-02-24 19:00:27
---

tpcds是针对linux写的代码，进入tool目录执行`make`即可编译，在mac下编译的时候会碰到如下问题。

### fatal error: 'values.h' file not found

错误：

```
./porting.h:46:10: fatal error: 'values.h' file not found
#include <values.h>
         ^~~~~~~~~~
1 error generated.
make: *** [mkheader.o] Error 1
```

values.h 是GNU的库，在mac下可使用如下头文件替换：

```
#include <limits.h>
#include <float.h>
```

### fatal error: 'malloc.h' file not found

错误：

```
dist.c:41:10: fatal error: 'malloc.h' file not found
#include <malloc.h>
         ^~~~~~~~~~
1 error generated.
make: *** [dist.o] Error 1
```

mac下的malloc头文件移到了sys下，因此修改为：

```
#include <sys/malloc.h>
```


### error: use of undeclared identifier 'MAXINT'

错误：

```
genrand.c:87:12: error: use of undeclared identifier 'MAXINT'
      s += MAXINT;
           ^
genrand.c:117:36: error: use of undeclared identifier 'MAXINT'
   return ((double) res / (double) MAXINT);
                                   ^
genrand.c:149:26: error: use of undeclared identifier 'MAXINT'
           Z = (M * Z) % MAXINT;
                         ^
genrand.c:151:23: error: use of undeclared identifier 'MAXINT'
        M = (M * M) % MAXINT;
                      ^
genrand.c:187:53: error: use of undeclared identifier 'MAXINT'
           fres += (double) (next_random (stream) / MAXINT) - 0.5;
                                                    ^
genrand.c:234:53: error: use of undeclared identifier 'MAXINT'
           fres += (double) (next_random (stream) / MAXINT) - 0.5;
                                                    ^
genrand.c:291:68: error: use of undeclared identifier 'MAXINT'
                (double) ((double) next_random (stream) / (double) MAXINT) -
                                                                   ^
genrand.c:591:16: error: use of undeclared identifier 'MAXINT'
        skip = MAXINT / MAX_COLUMN;
               ^
8 errors generated.
make: *** [genrand.o] Error 1
```

这里是宏没有定义，因为mac和linux的h文件差异，部分宏mac并没有，因此直接自己添加即可：


```
vi genrand.h
define MAXINT 4096000
```

再次make即可生成对应的二进制工具文件。
