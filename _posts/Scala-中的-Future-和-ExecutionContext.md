---
title: Scala 中的 Future 和 ExecutionContext
tags: 编程基础
categories: 编程基础
abbrlink: 16f45118
date: 2021-05-06 22:41:47
---

Scala 这个语言可以用得很复杂、也可以用得很简洁。它在 Java 并发和 OO 之上做了进一步的抽象，将代码量大大降低。

最近流行的很多项目都使用 Scala，比如 Akka，Spark，Kafka，Spray，Play Framework，足见它是一门生产力很高的语言。

这里主要总结了下一些并发模式和并发线程池需要注意的地方。


## Scala 中的 Future 并发模式
一切皆 Future:
```
val mFuture = future {
  Thread sleep 1000
  "result"
}
val result = Await result (mFuture, 3 seconds)
```
Callback 的几种方式：
```
f.onComplete {
  case Success(result) =>
  case Failure(ex) =>
}

Await result (mFuture, 5 seconds)
```
Timeout Fallback:
```
val searchFuture = search(worktime)
val fallback = after(timeout, context.system.scheduler) {
  Future successful s"$worktime ms > $timeout"
}

Future firstCompletedOf Seq(searchFuture, fallback)
```

Future 之后的运算 map / filter 成其他 Future：
```
f.map { r1 =>
    ...
    r2
}
```
多个 Future 链的处理 flatMap:
```
val nestedFuture: Future[Future[Boolean]] = heatWater(Water(25)).map {
  water => temperatureOkay(water)
}
val flatFuture: Future[Boolean] = heatWater(Water(25)).flatMap {
  water => temperatureOkay(water)
}
```
多个 Future 的链合并的另一种方式：
```
val f = for {
  result1 <- remoteCall1
  result2 <-  remoteCall2
} yield List(result1, result2)
```
假如 future 定义在 for 之前则会并发执行，否则会顺序执行。另外，假如顺序执行 result1 可以作为参数传递到 remoteCall2 中。

转换 Future List 成 List Future (map to)：
```
Future.sequence(list) //(并发执行)
```

转换 Future List 成单个 Future (map to)：
```
Future.reduce(list)(f)
```

Option 和 getOrElse 经常用在 Future 执行中。

## Scala 中的并发任务执行体 ExecutionContext

Scala 中的 ExecutionContext 和 Java 的线程池的概念非常相似。都是执行具体 Task 的执行体。

Scala 中默认的线程池：
```
import scala.concurrent.ExecutionContext.Implicits.global
```

是最方便的做法，如果不考虑优化和性能，在所有需要 ExecutionContext 的地方引用即可。当然这在生产环境是行不通的，原因是假如有 Task Block 了整个 global 线程池，应用将变得不可响应，即使 Block 不一定发生在本应用中，比如数据库的操作引起的 blocking 。

可以修改默认线程池的大小：
```
-Dscala.concurrent.context.numThreads=8 -Dscala.concurrent.context.maxThreads=8
```
自定义线程池
创建固定大小线程池

```
implicit val ec = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(10))
```
假如熟悉 Java 线程池的话，线程池的创建和 Java 中完全一样，可以套用。

Scala & Akka 中的 Dispatcher
定义一个 Dispatcher：
```
my-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "fork-join-executor"
  # Configuration for the fork join pool
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 2
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 2.0
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```

或者这样：
```
my-thread-pool-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "thread-pool-executor"
  # Configuration for the thread pool
  thread-pool-executor {
    # minimum number of threads to cap factor-based core number to
    core-pool-size-min = 2
    # No of core threads ... ceil(available processors * factor)
    core-pool-size-factor = 2.0
    # maximum number of threads to cap factor-based number to
    core-pool-size-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```
在 Scala 代码中引用引用之前定义的 Dispatcher：
```
implicit val executionContext = system.dispatchers.lookup("my-dispatcher")
```
给某个 Actor 指定 dispatcher：

```
val myActor = context.actorOf(Props[MyActor].withDispatcher("my-dispatcher"), "myactor1")
```
## 最佳实践

将不同类型的运算进行 dispatcher 隔离：
给 blocking I/O 创建单独的线程池：
因为 JDBC 没有 non-blocking API，所以为 DB R/W Heavey R/W 这些 Block 操作创建单独的 dispatcher。并在不同的 dispatcher 中执行不同类型的 Future 计算。

```
object Contexts {
  implicit val simpleDbLookups: ExecutionContext = Akka.system.dispatchers.lookup("contexts.simple-db-lookups")
  implicit val expensiveDbLookups: ExecutionContext = Akka.system.dispatchers.lookup("contexts.expensive-db-lookups")
  implicit val dbWriteOperations: ExecutionContext = Akka.system.dispatchers.lookup("contexts.db-write-operations")
  implicit val expensiveCpuOperations: ExecutionContext = Akka.system.dispatchers.lookup("contexts.expensive-cpu-operations")
}
```
同理，需要给 CPU 密集计算创建单独的线程池。

