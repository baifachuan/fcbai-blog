---
title: 自己实现兼容Hive客户端的服务器时的一些问题
tags: 大数据源码
categories: 大数据源码
description: 自己动手实现一个thrift服务器，Hive JDBC服务器实现，自己实现兼容Hive客户端的服务器时的一些问题
keywords: Hive服务器，Hive JDBC，SQL服务器，Hive源码，Hive原理，thrift，thrift server， 兼容HIVE JDBC
abbrlink: a16034a0
date: 2022-01-20 15:45:49
---

最近因为一些需求需要自己实现一个兼容Hive JDBC的服务器，支持使用Hive的JDBC或者Beeline连接上来分析，并且获取结果，整个服务的实现倒不是很复杂，主要是实现：

`TCLIService.Iface` 中的所有接口，并且再使用`TThreadPoolServer`来支持启动一个多线程服务器即可，这样就可以实现一个兼容hive jdbc的服务器，大头工作是需要精通thrift和每一个接口参数定义等信息。

我大概花了2周左右的时间写了一个SQL服务器，兼容了JDBC所有接口，使用代码编写测试用例的时候没有问题，使用BI工具例如Superset连接上去也没有问题，但是在使用beeline进行交互式分析的时候发现有问题了。

主要的现象就是执行一个查询后，屏幕会一直刷结果，不会停止。

查阅JDBC和HiveServer2的源码可以发现，所有的API列表是这样的：

* OpenSession
* CloseSession
* GetInfo
* ExecuteStatement
* GetTypeInfo
* GetCatalogs
* GetSchemas
* GetTables
* GetTableTypes
* GetColumns
* GetFunctions
* GetPrimaryKeys
* GetCrossReference
* GetOperationStatus
* CancelOperation
* CloseOperation
* GetResultSetMetadata
* FetchResults
* GetDelegationToken
* CancelDelegationToken
* RenewDelegationToken
* GetQueryId
* SetClientInfo

这也是我们需要实现的所有接口，而使用JDBC执行一个任务的主干流程是：

* OpenSession
* Execute
* GetOperationStatus
* FetchResult
* CloseSession

大体就是这样几个流程，如果是使用beeline的话，会多一些额外的元数据接口，例如GetInfo，GetSchema等信息。

我仔细看了整个Hive JDBC的实现，并且去调试了一下HiveServer2的实现，发现这个自己实现的服务器不停的刷新结果，看起来是Hive JDBC埋下的一个坑，HiveServer2做了兼容实现。

首先在任务执行里面获取数据结果走的是FetchResult这个thrift接口，这个接口除了返回任务执行的真正结果外，也会返回日志，也就是日志和内容的查询使用的其实是同一个接口，对于熟悉Hive JDBC的人来说。

就是你使用`HiveStatement`的`getQueryLog()`这个方法获取日志，或者使用`resultSet`去`next`拿结果的时候背后走的都是`FetchResult`这个API。

这里面的我首先需要理清楚的问题是什么时机去获取结果，看Hive的代码，可以知道在HiveStatement的execute里面会去调用后端服务执行sql任务，并且基于Execute的返回来这样判断：

```java
// The query should be completed by now
if (!stmtHandle.isHasResultSet()) {
  return false;
}
resultSet =  new HiveQueryResultSet.Builder(this).setClient(client).setSessionHandle(sessHandle)
    .setStmtHandle(stmtHandle).setMaxRows(maxRows).setFetchSize(fetchSize)
    .setScrollable(isScrollableResultset).setTransportLock(transportLock)
    .build();
return true;
```

也就是如果服务器执行完sql没有结果，或者这是一个不需要返回结果的sql，例如DDL，那么这里就不会创建resultSet了，直接结束掉，和我们常规经验一致。

如果有结果且需要展示，那么会返回一个resultSet对象，这个对象是不是很熟悉？是的，它就是Java标准的SQL接口中的ResultSet的Hive JDBC实现，当返回这个对象后，其实就进入到我们常见的这个代码中了：

```java
ResultSet rs = hiveStatement.executeQuery(sql);
while (rs.next()) {
    for (int i = 1; i <= rs.getMetaData().getColumnCount(); i++) {
        System.out.print(rs.getString(i) + "\t");
    }
    System.out.println();
}
```

是不是很熟悉的代码？后于我自己的服务器在实现FetchResult的时候，是把查询结果放在某个地方，基于客户端的请求一批次一批次来查询，也就是这里的rs.next会触发的查询。

但是并不是每一次rs.next都会触发一次后端的FetchResult API调用，而是JDBC客户端会一批次默认50条数据像后端请求，如果50遍历完了再重新请求。

因此当JDBC客户端发起查询请求的时候，我后端记录了一个offset值，记录客户端请求到第几次的50了，当把最后一页的内容都返回后，在请求的时候就会拿到一个空List，客户端会认为没有内容了。

也就退出前面那个rs.next的while循环了，一切结束，同时后端服务器会把这一次任务执行的offset重置为0，是为了解决一个connection提交多个查询的情况，offset是connection共享的，本质在于thrift是异步接口。

但是为啥代码没有问题，beeline会狂刷结果？我尝试写了如下代码：

```java
ResultSet rs = hiveStatement.executeQuery(sql);
while (rs.next()) {
    for (int i = 1; i <= rs.getMetaData().getColumnCount(); i++) {
        System.out.print(rs.getString(i) + "\t");
    }
    System.out.println();
}

while (rs.next()) {
    for (int i = 1; i <= rs.getMetaData().getColumnCount(); i++) {
    System.out.print(rs.getString(i) + "\t");
    }
    System.out.println();
}
```

看起来是不是没有任何变化？只是把while循环拷贝了一遍，正常逻辑来说，一旦退出第一个while循环，第二个while循环是不能进去的，因为rs 这个对象已经被遍历完了。

但是我发现居然进入到第二个while的时候还会有数据出现，而beeline是一个while(true)循环，会无限制执行rs.nex，所以会导致只要有结果就会返回。

那么先分享我自己实现的这个服务器为啥会进入第二个while ， 并且有结果，原因在于执行完第一个while后，后台把offset设置为0，这时候进入第二个rs的时候，它会发现本地的list是nul，也就是HiveQueryResultSet的这个代码：

```java
if (fetchedRows == null || !fetchedRowsItr.hasNext()) {
    TFetchResultsReq fetchReq = new TFetchResultsReq(stmtHandle,
        orientation, fetchSize);
    TFetchResultsResp fetchResp;
    if (transportLock == null) {
      fetchResp = client.FetchResults(fetchReq);
    } else {
      transportLock.lock();
      try {
        fetchResp = client.FetchResults(fetchReq);
      } finally {
        transportLock.unlock();
      }
    }
    Utils.verifySuccessWithInfo(fetchResp.getStatus());
    
    TRowSet results = fetchResp.getResults();
    fetchedRows = RowSetFactory.create(results, protocol);
    fetchedRowsItr = fetchedRows.iterator();
}
```

这里的`client.FetchResults(fetchReq)`又触发了一次调用，而此时后端的offset已经被重置为0，所以又从头开始返回内容，不断往复，这就是为啥beeline会不断持续的刷新的问题。

看到这里我就在想，不应该啊，这个明显应该是客户端来判断是否结束，因为每一次请求服务会给客户端返回内容，同时会返回`hasResultSet`这个标志来告诉前端是否还有后续结果。

再不济客户端自己是按照一批次像后端发起查询的，客户端在发现后端返回的size小于batch size的时候也可以终止查询，这是一个很明显的客户端行为，但是jdbc里面却没有考虑，同时JDBC还维护了一个maxRows，但是我看了下几乎整个流程都没有被使用，只有在事务表下被没有意义的调用过，而这个字段也可以用来判断是否结束。。

然后我就在想这个问题肯定不能只是我碰到，HiveServer2已经实现了，必然对这部分做了单独的处理，于是我看了下hive服务器的实现，发现大概逻辑是把所有的数据内容放到一个list里面，每一次客户端查询一批数据后，就把这一批数据从整个list里面remove掉，这样就能确保整个数据每一条只会被访问一次。

后续无论多少个while ， 此时的服务端的list是空的，自然不会出问题，同时也省去了offse，毕竟，数据都没了，要啥offset，虽然很戳，但是简单，有效，粗暴。

老实说我看到这种实现感觉非常不合理，先不说是否支持数据重播，这种把客户端逻辑侵入到服务端，整体设计会导致服务端的实现不能使用offset的机制，而offset才是数据查找的核心逻辑，至于服务端何时处理数据，不应该由客户端控制。

所以我总是觉得这是jdbc实现的一个坑，然后hive实现了妥协，虽然我也照着差不多的逻辑做了处理，但是这里还是保留一定的意见，打算和hive社区讨论下。
