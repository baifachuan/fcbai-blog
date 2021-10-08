---
title: Apache Kafka 内核深度剖析
tags:
  - Kafka源码
  - 大数据源码
categories:
  - 大数据
abbrlink: b88e2480
date: 2020-03-08 23:36:57
---

## 摘要

目前来说市面上可以选择的消息队列非常多，像 activemq，rabbitmq，zeromq 已经被大多数人耳熟能详，特别像 activemq 早期应用在企业中的总线通信，基本作为企业级 IT 设施解决方案中不可或缺的一部分。目前来说 Kafka 已经非常稳定，并且逐步应用更加广泛，已经算不得新生事物，但是不可否认 Kafka 一枝独秀如同雨后春笋，非常耀眼，今天我们仔细分解一下 Kafka，了解一下它的内幕。以下的内容版本基于当前最新的 Kafka 稳定版本 2.4.0。文章主要包含以下内容：
* Kafka 为什么快
* Kafka 为什么稳
* Kafka 该怎么用
该文章为开篇引导之做，后续会有对应的 HBase，Spark，Kylin，Pulsar 等相关组件的剖析。

<!-- more -->

## Kafka 为什么快
快是一个相对概念，没有对比就没有伤害，因此通常我们说 Kafka 是相对于我们常见的 activemq，rabbitmq 这类会发生 IO，并且主要依托于 IO 来做信息传递的消息队列，像 zeromq 这种基本纯粹依靠内存做信息流传递的消息队列，当然会更快，但是此类消息队列只有特殊场景下会使用，不在对比之列。

因此当我们说 Kakfa 快的时候，通常是基于以下场景：

* 吞吐量：当我们需要每秒处理几十万上百万 message 的时候，相对其他 MQ，Kafka 处理的更快。
* 高并发：当具有百万以及千万的 consumer 的时候，同等配置的机器下，Kafka 所拥有的 Producer 和 Consumer 会更多。
* 磁盘锁：相对其他 MQ，Kafka 在进行 IO 操作的时候，其同步锁住 IO 的场景更少，发生等待的时间更短。

那么基于以上几点，我们来仔细探讨一下，为什么 Kafka 就快了。

#### 消息队列的推拉模型

首先，Kafka 快如果我们单纯站在 Consumer 的角度来看是一个伪命题，因为相比其他 MQ，Kafka 从 Producer 产生一条 Message 到 Consumer 消费这条 Message 来看它的时间一定是大于等于其他 MQ 的，背后的原因涉及到消息队列设计的两种模型：推模型和拉模型，如下图所示：

![在这里插入图片描述](https://images.gitbook.cn/c06147f0-4335-11ea-b599-ad2da1334987)

对于拉模型来说，Producer 产生 Message 后，会主动发送给 MQ Server，为了提升性能和减少开支，部分 Client 还会设计成批量发送，但是无论是单条还是批量，Producer 都会主动推送消息到 MQ Server，当 MQ Server 接收到消息后，对于拉模型，MQ Server 不会主动发送消息到 Consumer，同时也不会维持和记录消息的 offset，Consumer 会自动设置定时器到服务端去询问是否有新的消息产生，通常时间是不超过 100ms 询问一次，一旦产生新的消息则会同步到本地，并且修改和记录 offset，服务端可以辅助存储 offset，但是不会主动记录和校验 offset 的合理性，同时 Consumer 可以完全自主的维护 offset 以便实现自定义的信息读取。

对于推模型来说，服务端收到 Message 后，首先会记录消息的信息，并且从自己的元信息数据库中查询对应的消息的 Consumer 有谁，由于服务器和 Consumer 在链接的时候建立了长链接，因此可以直接发送消息到 Consumer。

Kafka 是基于拉模型的消息队列，因此从 Consumer 获取消息的角度来说，延迟会小于等于轮询的周期，所以会比推模型的消息队列具有更高的消息获取延迟，但是推模型同样又其问题。首先，由于服务器需要记录对应的 Consumer 的元信息，包括消息该发给谁，offset 是多少，同时需要向 Consumer 推送消息，必然会带来系列的问题：假如这一刻网络不好，Consumer 没有收到，消息没有发成功怎么办？假设消息发出去了，我怎么知道它有没有收到？因此服务器和 Consumer 之间需要首先多层确认口令，以达到至少消费一次，仅且消费一次等特性。

Kafka 此类的拉模型将这一块功能都交由 Consumer 自动维护，因此服务器减少了更多的不必要的开支，因此从同等资源的角度来讲，Kafka 具备链接的 Producer 和 Consumer 将会更多，极大的降低了消息堵塞的情况，因此看起来更快了。

##  OS Page Cache 和 Buffer Cache

太阳底下无新鲜事，对于一个框架来说，要想运行的更快，通常能用的手段也就那么几招，Kafka 在将这一招用到了极致，其中之一就是极大化的使用了 OS 的 Cache，主要是 Page Cache 和 Buffer Cache。对于这两个 Cache，使用 Linux 的同学通常不会陌生，例如我们在 Linux 下执行 free 命令的时候会看到如下的输出：

![在这里插入图片描述](https://images.gitbook.cn/cb9be260-4335-11ea-b4ae-d3af6e5d7b57)

会有两列名为 buffers 和 cached，也有一行名为“-/+ buffers/cache”。这两个信息的具体解释如下：

pagecache：文件系统层级的缓存，从磁盘里读取的内容是存储到这里，这样程序读取磁盘内容就会非常快，比如使用 Linux 的 grep 和 find 等命令查找内容和文件时，第一次会慢很多，再次执行就快好多倍，几乎是瞬间。另外 page cache 的数据被修改过后，也即脏数据，等到写入磁盘时机到来时，会转移到 buffer cache 而不是直接写入到磁盘。我们看到的 cached 这列的数值表示的是当前的页缓存（page cache）的占用量，page cache 文件的页数据，页是逻辑上的概念，因此 page cache 是与文件系统同级的。

 buffer cache：磁盘等块设备的缓冲，内存的这一部分是要写入到磁盘里的 。buffers 列表示当前的块缓存（buffer cache）占用量，buffer cache 用于缓存块设备（如磁盘）的块数据。块是物理上的概念，因此 buffer cache 是与块设备驱动程序同级的。

![在这里插入图片描述](https://images.gitbook.cn/d640f390-4335-11ea-aa55-2f590bdada6f)

两者都是用来加速数据 IO，将写入的页标记为 dirty，然后向外部存储 flush，读数据时首先读取缓存，如果未命中，再去外部存储读取，并且将读取来的数据也加入缓存。操作系统总是积极地将所有空闲内存都用作 page cache 和 buffer cache，当 os 的内存不够用时也会用 LRU 等算法淘汰缓存页。

有了以上概念后，我们再看来 Kafka 是怎么利用这个特性的。首先，对于一次数据 IO 来说，通常会发生以下的流程：
![在这里插入图片描述](https://images.gitbook.cn/1c769590-4336-11ea-9268-457edb399911)

* 操作系统将数据从磁盘拷贝到内核区的 pagecache 
* 用户程序将内核区的 pagecache 拷贝到用户区缓存 
* 用户程序将用户区的缓存拷贝到 socket 缓存中 
* 操作系统将 socket 缓存中的数据拷贝到网卡的 buffer 上，发送数据

可以发现一次 IO 请求操作进行了 2 次上下文切换和 4 次系统调用，而同一份数据在缓存中多次拷贝，实际上对于拷贝来说完全可以直接在内核态中进行，也就是省去第二和第三步骤，变成这样：

![在这里插入图片描述](https://images.gitbook.cn/21cb0670-4336-11ea-b197-a56306ef8720)

正因为可以如此的修改数据的流程，于是 Kafka 在设计之初就参考此流程，尽可能大的利用 os 的 page cache 来对数据进行拷贝，尽量减少对磁盘的操作。如果 kafka 生产消费配合的好，那么数据完全走内存，这对集群的吞吐量提升是很大的。早期的操作系统中的 page cache 和 buffer cache 是分开的两块 cache，后来发现同样的数据可能会被 cache 两次，于是大部分情况下两者都是合二为一的。

Kafka 虽然使用 JVM 语言编写，在运行的时候脱离不了 JVM 和 JVM 的 GC，但是 Kafka 并未自己去管理缓存，而是直接使用了 OS 的 page cache 作为缓存，这样做带来了以下好处：

* JVM 中的一切皆对象，所以无论对象的大小，总会有些额外的 JVM 的对象元数据浪费空间。
* JVM 自己的 GC 不受程序手动控制，所以如果使用 JVM 作为缓存，在遇到大对象或者频繁 GC 的时候会降低整个系统的吞吐量。
* 程序异常退出或者重启，所有的缓存都将失效，在容灾架构下会影响快速恢复。而 page cache 因为是 os 的 cache，即便程序退出，缓存依旧存在。

所以 Kafka 优化 IO 流程，充分利用 page cache，其消耗的时间更短，吞吐量更高，相比其他 MQ 就更快了，用一张图来简述三者之间的关系如下：
![在这里插入图片描述](https://images.gitbook.cn/3dcdbd90-4336-11ea-b943-95709b8744cc)
当 Producer 和 Consumer 速率相差不大的情况下，Kafka 几乎可以完全实现不落盘就完成信息的传输。

##  追加顺序写入

除了前面的重要特性之外，Kafka 还有一个设计，就是对数据的持久化存储采用的顺序的追加写入，Kafka 在将消息落到各个 topic 的 partition 文件时，只是顺序追加，充分的利用了磁盘顺序访问快的特性。

![在这里插入图片描述](https://images.gitbook.cn/48e8c210-4336-11ea-b4ae-d3af6e5d7b57)

Kafka 的文件存储按照 topic 下的 partition 来进行存储，每一个 partition 有各自的序列文件，各个 partition 的序列不共享，主要的划分按照消息的 key 进行 hash 决定落在哪个分区之上，我们先来详细解释一下 Kafka 的各个名词，以便充分理解其特点：

* broker：Kafka 中用来处理消息的服务器，也是 Kafka 集群的一个节点，多个节点形成一个 Kafka 集群。
* topic：一个消息主题，每一个业务系统或者 Consumer 需要订阅一个或者多个主题来获取消息，Producer 需要明确发生消息对于的 topic，等于信息传递的口令名称。
* partition：一个 topic 会拆分成多个 partition 落地到磁盘，在 kafka 配置的存储目录下按照对应的分区 ID 创建的文件夹进行文件的存储，磁盘可以见的最大的存储单元。
* segment：一个 partition 会有多个 segment 文件来实际存储内容。
* offset：每一个 partition 有自己的独立的序列编号，作用域仅在当前的 partition 之下，用来对对应的文件内容进行读取操作。
* leader：每一个 topic 需要有一个 leader 来负责该 topic 的信息的写入，数据一致性的维护。
* controller：每一个 kafka 集群会选择出一个 broker 来充当 controller，负责决策每一个 topic 的 leader 是谁，监听集群 broker 信息的变化，维持集群状态的健康。

![在这里插入图片描述](https://images.gitbook.cn/58ee03f0-4336-11ea-8733-e300e75fa302)
![在这里插入图片描述](https://images.gitbook.cn/5fffdc40-4336-11ea-b599-ad2da1334987)

可以看到最终落地到磁盘都是 Segment 文件，每一个 partion(目录)相当于一个巨型文件被平均分配到多个大小相等 segment(段)数据文件中。但每个段 segment file 消息数量不一定相等，这种特性方便老的 segment file 快速被删除。因为 Kafka 处理消息的力度是到 partition，因此只需要保持好 partition 对应的顺序处理，segment 可以单独维护其状态。

segment 的文件由 index file 和 data file 组成，落地在磁盘的后缀为.index 和.log，文件按照序列编号生成，如下所示：
![在这里插入图片描述](https://images.gitbook.cn/788b1fe0-4336-11ea-b943-95709b8744cc)

其中 index 维持着数据的物理地址，而 data 存储着数据的偏移地址，相互关联，这里看起来似乎和磁盘的顺序写入关系不大，想想 HDFS 的块存储，每次申请固定大小的块和这里的 segment？是不是挺相似的？另外因为有 index 文的本身命名是以 offset 作为文件名的，在进行查找的时候可以快速根据需要查找的 offset 定位到对应的文件，再根据文件进行内容的检索。因此 Kafka 的查找流程为先根据要查找的 offset 对文件名称进行二分查找，找到对应的文件，再根据 index 的元数据的物理地址和 log 文件的偏移位置结合顺序读区到对应 offset 的位置的内容即可。

segment index file 采取稀疏索引存储方式，它减少索引文件大小，通过 mmap 可以直接内存操作，稀疏索引为数据文件的每个对应 message 设置一个元数据指针,它比稠密索引节省了更多的存储空间，但查找起来需要消耗更多的时间，特别是在随机读取的场景下，Kafka 非常不合适。所以因为 Kafka 特殊的存储设计，也让 Kafka 感觉起来，更快。

## Kafka 为什么稳

前面提到 Kafka 为什么快，除了快的特性之外，Kafka 还有其他特点，那就是：稳。Kafka 的稳体现在几个维度：

* 数据安全，几乎不会丢数据。
* 集群安全，发生故障几乎可以 Consumer 无感知切换。
* 可用性强，即便部分 partition 不可用，剩余的 partition 的数据依旧不影响读取。
* 流控限制，避免大量 Consumer 拖垮服务器的带宽。

#### 限流机制

对于 Kafka 的稳，通常是由其整体架构设计决定，很多优秀的特性结合在一起，就更加的优秀，像 Kafka 的 Qutota 就是其中一个，既然是限流，那就意味着需要控制 Consumer 或者 Producer 的流量带宽，通常限制流量这件事需要在网卡上作处理，像常见的 N 路交换机或者高端路由器，所以对于 Kafka 来说，想要操控 OS 的网卡去控制流量显然具有非常高的难度，因此 Kafka 采用了另外一个特别的思路，即：没有办法控制你的网卡通过的流量大小，那么便控制返回给你数据的时间。对于 JVM 程序来说，就是一个 wait 或者 seelp 的事情。

所以对于 Kafka 来说，有一套特殊的时延计算规则，Kafka 按照一个窗口来统计单位时间传输的流量，当流量大小超过设置的阈值的时候，触发流量控制，将当前请求丢入 Kafka 的 Qutota Manager，等到延迟时间到达后，再次返回数据。我们通过 Kafka 的 ClientQutotaManager 类中的方法来看：

![在这里插入图片描述](https://images.gitbook.cn/86207330-4336-11ea-a702-d3ff8e8dc85c)

这几行代码代表了 Kafka 的限流计算逻辑，大概的思路为：假设我们设定当前流量上限不超过 T，根据窗口计算出当前的速率为 O，如果 O 超过了 T，那么会进行限速，限速的公示为：

```X = (O - T)/ T * W ```

X 为需要延迟的时间，让我举一个形象的例子，假设我们限定流量不超过 10MB/s，过去 5 秒（公示中的 W，窗口区间）内通过的流量为 100MB，则延迟的时间为：（100-5*10）/ 10=5 秒。这样就能够保障在下一个窗口运行完成后，整个流量的大小是不会超过限制的。通过 KafkaApis 里面对 Producer 和 Consumer 的 call back 代码可以看到对限流的延迟返回：

![在这里插入图片描述](https://images.gitbook.cn/8ec3e800-4336-11ea-9268-457edb399911)

对于 kafka 的限流来讲，默认是按照 client id 或者 user 来进行限流的，从实际使用的角度来说，意义不是很大，基于 topic 或者 partition 分区级别的限流，相对使用场景更大，ThoughtWroks 曾经帮助某客户修改 Kafka 核心源码，实现了基于 topic 的流量控制。

#### 竞选机制

Kafka 背后的元信息重度依赖 Zookeeper，再次我们不解释 Zookeeper 本身，而是关注 Kafka 到底是如何使用 zk 的，首先一张图解释 Kafka 对 zk 的重度依赖：

![在这里插入图片描述](https://images.gitbook.cn/99ab5960-4336-11ea-b4ae-d3af6e5d7b57)

利用 zk 除了本身信息的存储之外，最重要的就是 Kafka 利用 zk 实现选举机制，其中以 controller 为主要的介绍，首先 controller 作为 Kafka 的心脏，主要负责着包括不限于以下重要事项：

![在这里插入图片描述](https://images.gitbook.cn/af4a0460-4336-11ea-b4ae-d3af6e5d7b57)

也就是说 Controller 是 Kafka 的核心角色，对于 Controller 来说，采用公平竞争，任何一个 Broker 都有可能成为 Controller，保障了集群的健壮性，对于 Controller 来说，其选举流程如下：

* 先获取 zk 的 /cotroller 节点的信息，获取 controller 的 broker id，如果该节点不存在（比如集群刚创建时），* 那么获取的 controller id 为-1。
* 如果 controller id 不为-1，即 controller 已经存在，直接结束流程。
* 如果 controller id 为-1，证明 controller 还不存在，这时候当前 broker 开始在 zk 注册 controller；。
* 如果注册成功，那么当前 broker 就成为了 controller，这时候开始调用 onBecomingLeader() 方法，正式初始化 controller（注意：controller 节点是临时节点，如果当前 controller 与 zk 的 session 断开，那么 controller 的临时节点会消失，会触发 controller 的重新选举）。
* 如果注册失败（刚好 controller 被其他 broker 创建了、抛出异常等），那么直接返回。

其代码直接通过 KafkaController 可以看到：

![在这里插入图片描述](https://images.gitbook.cn/be426a20-4336-11ea-b943-95709b8744cc)

一旦 Controller 选举出来之后，则其他 Broker 会监听 zk 的变化，来响应集群中 Controller 挂掉的情况：

![在这里插入图片描述](https://images.gitbook.cn/d0538190-4336-11ea-a702-d3ff8e8dc85c)

从而触发新的 Controller 选举动作。对于 Kafka 来说，整个设计非常紧凑，代码质量相当高，很多设计也非常具有借鉴意义，类似的功能在 Kafka 中有非常多的特性体现，这些特性结合一起，形成了 Kafka 整个稳定的局面。


## Kafka 该怎么用

虽然 Kafka 整体看起来非常优秀，但是 Kafka 也不是全能的银弹，必然有其对应的短板，那么对于 Kafka 如何，或者如何能用的更好，则需要经过实际的实践才能得感悟的出。经过归纳和总结，能够发现以下不同的使用场景和特点。

* Kafka 并不合适高频交易系统：Kafka 虽然具有非常高的吞吐量和性能，但是不可否认，Kafka 在单条消息的低延迟方面依旧不如传统 MQ，毕竟依托推模型的 MQ 能够在实时消息发送的场景下取得先天的优势。
* Kafka 并不具备完善的事务机制：0.11 之后 Kafka 新增了事务机制，可以保障 Producer 的批量提交，为了保障不会读取到脏数据，Consumer 可以通过对消息状态的过滤过滤掉不合适的数据，但是依旧保留了读取所有数据的操作，即便如此 Kafka 的事务机制依旧不完备，背后主要的原因是 Kafka 对 client 并不感冒，所以不会统一所有的通用协议，因此在类似仅且被消费一次等场景下，效果非常依赖于客户端的实现。
* Kafka 的异地容灾方案非常复杂：对于 Kafka 来说，如果要实现跨机房的无感知切换，就需要支持跨集群的代理，因为 Kafka 特殊的 append log 的设计机制，导致同样的 offset 在不同的 broker 和不同的内容上无法复用，也就是文件一旦被拷贝到另外一台服务器上，将不可读取，相比类似基于数据库的 MQ，很难实现数据的跨集群同步，同时对于 offset 的复现也非常难，曾经帮助客户实现了一套跨机房的 Kafka 集群 Proxy，投入了非常大的成本。
* Kafka Controller 架构无法充分利用集群资源：Kafka Controller 类似于 Es 的去中心化思想，按照竞选规则从集群中选择一台服务器作为 Controller，意味着改服务器即承担着 Controller 的职责，同时又承担着 Broker 的职责，导致在海量消息的压迫下，该服务器的资源很容易成为集群的瓶颈，导致集群资源无法最大化。Controller 虽然支持 HA 但是并不支持分布式，也就意味着如果要想 Kafka 的性能最优，每一台服务器至少都需要达到最高配置。
* Kafka 不具备非常智能的分区均衡能力：通常在设计落地存储的时候，对于热点或者要求性能足够高的场景下，会是 SSD 和 HD 的结合，同时如果集群存在磁盘容量大小不均等的情况，对于 Kafka 来说会有非常严重的问题，Kafka 的分区产生是按照 paratition 的个数进行统计，将新的分区创建在个数最少的磁盘上，见下图：
![在这里插入图片描述](https://images.gitbook.cn/ed457ab0-4336-11ea-a702-d3ff8e8dc85c)
曾经我帮助某企业修改了分区创建规则，考虑了容量的情况，也就是按照磁盘容量进行分区的选择，紧接着带来第二个问题：容量大的磁盘具备更多的分区，则会导致大量的 IO 都压向该盘，最后问题又落回 IO，会影响该磁盘的其他 topic 的性能。所以在考虑 MQ 系统的时候，需要合理的手动设置 Kafka 的分区规则。。

# 结尾
Kafka 并不是唯一的解决方案，像几年前新生势头挺厉害的 pulsar，以取代 Kafka 的口号冲入市场，也许会成为下一个解决 Kafka 部分痛点的框架，下文再讲述 pulsar。

