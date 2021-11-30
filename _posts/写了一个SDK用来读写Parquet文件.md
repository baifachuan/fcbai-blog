---
title: 写了一个SDK用来读写Parquet文件
tags: 编程基础
categories: 编程基础
abbrlink: 2e6dc139
date: 2021-11-25 15:04:43
---

Parquet是比较常见的数据格式，最近将数据的中间结果以parquet的形式进行了保存，在获取的时候需要读出内容，再以json的方式传递给前端进行使用。
不过在读取文件转json的时候碰到了问题，parquet虽然很常用，使用spark等框架读取基本也就是一两行代码的事，但是由于传输数据给前端的部分是web serve。
所以使用spark这样的框架有些许不合理，考虑使用java代码直接读取，能够拿来读取的库是：
```xml
 <dependency>
    <groupId>org.apache.parquet</groupId>
    <artifactId>parquet-tools</artifactId>
    <version>1.11.2</version>
</dependency>
```
也算是比较官方的库，但是使用该库的to json api将一个parquet的文件转成json后格式是这样的：
```json
data1: value1
data2: value2
models
  map
    key: data3
    value
      array: value3
  map
    key: data4
    value
      array: value4
data5: value5
...
```

但是实际的真是格式是：
```json
"data1": "value1",
"data2": "value2",
"models": {
    "data3": [
        "value3"
    ],
    "data4": [
        "value4"
    ]
},
"data5": "value5"
...
```
这是因为在parquet里面所有的内容都会被包装成SimpleRecord，而我看了下SimpleRecord，里面有prettyPrintJson和toJsonObject，这两个方法的输出是接近需要的。
不过这两方法都是protected，于是想到只需要将这里面的方法重写即可，于是自己手动封装了一个java 的sd，用来read和write parquet文件，考虑到代码简洁我也没有使用lombok这样的工具包了。

地址参考：https://github.com/baifachuan/parquet-java-sdk
