---
title: '解决pytorch进行深度学习的时候出现的：Warning: Cannot set number of intraop threads'
categories: 问题调试
abbrlink: d7eb454a
date: 2020-11-26 21:13:43
tags 问题调试:
---

最近我在使用pytorch构建神经网络，处理图像，遇到一个问题，如下：

```
[W ParallelNative.cpp:206] Warning: Cannot set number of intraop threads after parallel work has started or after set_num_threads call when using native parallel backend (function set_num_threads)
```

看起来只是一个警告，并不是error，所以也就没有关心，但是发现程序在这个警告后就卡死不动了，具体的原因有空我再深入分析一下，解决办法为：

通过设置环境变量：`export OMP_NUM_THREADS=1` 将并行度设置为1，虽然会降低速度，但是因为我是预测环节，所以也不会太慢。
