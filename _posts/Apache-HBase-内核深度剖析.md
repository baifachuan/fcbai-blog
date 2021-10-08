---
title: Apache HBase 内核深度剖析
tags:
  - HBase源码
  - 大数据源码
categories: 大数据
abbrlink: d1b2768e
date: 2020-03-08 23:36:03
---

# 摘要

前面一篇文章介绍了 Kafka 的具体内容，今天讲述一下 HBase 相关的知识。首先 HBase 作为大数据发展初期伴随 Google 三大论文问世的一个组件，但是即便在今天依旧被广泛的应用，今天我们来仔细的分析一下 HBase 的内部原理，了解一下 HBase 的具体内幕，以便更好的在工作中使用它，懂它才能更好的使用它。以下内容涉及到的源码基于 HBase 的 Master 分支编译出的最新的 3.0.0 版本。

<!-- more -->

# HBase 相关算法与数据结构基础知识

#### 跳跃表

暂时先不说跳跃表是什么，在 Java 里面有一个 Map 叫：ConcurrentSkipListMap，通过对 HBase 的源码跟踪我们发现在这些地方使用了它：
![在这里插入图片描述](https://images.gitbook.cn/567139f0-4d43-11ea-aeaa-49f91881cc0c)

简单的列了几个，但是观察这几个类所在的模块就可以发现，HBase 从客户端，到请求处理，到元数据再到文件存储贯穿 HBase 的整个生命周期中的各个重要环节，都能看到它的身影，Map 那么多，为何偏偏 HBase 选择了这个，接下来我们仔细分析下。

在算法概念里面有一种数据结构叫跳跃表，顾名思义，之所以叫跳跃表就是因为在查找的时候可以快速的跳过部分列表，用来提升查找效率，跳跃表的查找效率可以和二叉树相比为 O(log(N))，这个算法的实现在 Java 中的 ConcurrentSkipListMap 就是实现跳跃表的相关算法。

首先我们看一个有序的全量表：
![在这里插入图片描述](https://images.gitbook.cn/5fdd7d00-4d43-11ea-95be-cf33507ef652)
假设我们要从中找出<C，E，H>可以发现需要比较的次数（比较次数不是循环次数）为<3，5，8>共计 16 次，可以看到对这样一个有序链表进行查找比较的次数会非常多，那么有没有办法对这种查找做优化。当然是有的，对于这种查找耳熟能详从数据结构的基础课程开始大家就知道二叉树，折半查找，值查找都属于解决这类问题的方法，自然跳跃表也是解决这类问题的方法之一。

跳跃表的思路和如今大部分大数据组件像 kylin 对海量数据下的快速查找的解决思路非常相似，都是通过某种逻辑提前将部分数据做预处理，然后查找的时候进行快速匹配，典型的空间换时间，那么对于跳跃表来说，它的预处理的方式如下：
![在这里插入图片描述](https://images.gitbook.cn/6a318800-4d43-11ea-9f6e-1bdc6229ab3f)

可以看到，跳跃表是按照层次构造的，最底层是一个全量有序链表，依次向上都是它的精简版，而在上层链表中节点的后续下一步的节点是以随机化的方式进行的，因此在上层链表中可以跳过部分列表，也叫跳跃表，特点如下：

* 链表层从上往下查找
* 跳跃表由很多层组成
* 每一层都是一个有序链表
* 最底层是全量有序链表
* 如果一个元素出现在上层的某个节点那么它一定会出现在下层的链表
* 每一个元素都有两个指针，一个指向当前链表的下一个元素，一个指向下一层链表的相同节点

假设根据前面的图表我们要查询 G 这个字母，那么在上面的跳跃表中经过的路径如下：
![在这里插入图片描述](https://images.gitbook.cn/72683140-4d43-11ea-95be-cf33507ef652)


其中红色代表查找所走过的路径。

#### LSM 树
前面讲到了跳跃表的原理，在 HBase 中较大规模的使用了跳跃表，就是为了增快其查找效率，除了跳跃表之外 HBase 还使用到了 LSM 树，LSM 树本质上和 B+相似，是一种存储在磁盘上的数据的索引格式，但是差异点在于 LSM 对写入非常高效，实现来说就是无论什么样的写入 LSM 都是当成一次顺序写入，这一点和 HDFS 的优点正好契合，HDFS 不支持随机写，支持顺序写。LSM 数据存储在两个地方，一个是磁盘上一个是内存中，内存中同样使用的跳跃表，内存中是多个有序的文件。

![在这里插入图片描述](https://images.gitbook.cn/7b7eda40-4d43-11ea-8823-b14c07bf0c6c)


HBase 对 LSM 的应用采用了如上的结构方式，对于 HBase 具体的存储文件的分析，在后面专门针对 HBase 的存储部分进行深入的分析。

#### 布隆过滤器

布隆过滤器解决的问题是，如何快速的发现一个元素是否存在于某个集合里面，最简单的办法就是在集合或者链表上查找一遍，但是考虑到在大数据场景下，数据量非常大，即便不考虑性能，也不见得会有足够多的机器来加载对应的集合。所以需要一种新的思路去解决此类问题，那么布隆过滤器就是一种，它的思想为：

* 由一个长度为 N 的数组构成，每一个元素为 0 或者 1，默认都为 0
* 对集合的每个元素做 K 次哈希，到第 i 次的时候对 N 取一个模，会得到一个 index
* 将数组中的 array[index]变为 1
![在这里插入图片描述](https://images.gitbook.cn/836603f0-4d43-11ea-bb35-2725b4649300)

上图是长度为 18，进行 3 次哈希得到的结果，那么在 HBase 中是如何利用布隆过滤器的呢，首先从操作来说，HBase 的 Get 就经过布隆过滤器，同时 HBase 支持度对不同的列设置不同的布隆过滤器。

![在这里插入图片描述](https://images.gitbook.cn/8b3252a0-4d43-11ea-9aca-f5e54cebdcef)

可以看到对 HBase 来讲可以启用或者禁用过滤器，对于不同的过滤器的实现分别在不同的	在查询的时候分别根据不同的过滤器采用不同的实现类：
![在这里插入图片描述](https://images.gitbook.cn/9e4c4e90-4d43-11ea-8823-b14c07bf0c6c)
所以可以通过如上的代码找到对应的过滤器实现，甚至可以新增自己的过滤器。

# HBase 读写操作

前面提到 HBase 的相关算法，现在我们讲一下 HBase 的整个操作的读写流程。首先，摆出 HBase 的架构图，如下所示：

![在这里插入图片描述](https://images.gitbook.cn/a4593a00-4d43-11ea-a325-5b2189984a9b)

从这个图可以看到，HBase 的一个写操作，大的流程会经过三个地方：1. 客户端，2. RegionServer 3. Memstore 刷新到磁盘。也就是说对于 HBase 的一次写入操作来讲，数据落到 Memstore 就算写入完成，那么必然需要考虑一个问题，那就是没有落盘的数据，万一机器发生故障，这部分数据如何保障不丢失。解析来我们逐步分解一下这三个部分。

客户端：HBase 的客户端和服务器并不是单一的链接，而是在封装完数据后，通过请求 HMaster 获取该次写入对应的 RegionServer 的地址，然后直接链接 RegionServer，进行写入操作，对于客户端的数据封装来讲，HBase 支持在客户端设置本地缓存，也就是批量提交还是实时提交。因为 HBase 的 hbase:meta 表中记录了 RegionServer 的信息，HBase 的数据均衡是根据 rowkey 进行分配，因此客户端会根据 rowkey 查找到对应的 RegionServer，定义在 Connection 中：
![在这里插入图片描述](https://images.gitbook.cn/ac207fa0-4d43-11ea-9f6e-1bdc6229ab3f)
而实现在：AsyncRegionLocator
![在这里插入图片描述](https://images.gitbook.cn/b1e03570-4d43-11ea-b4bd-6f2b05277b74)

RegionServer 写入：当客户端拿到对应的 RegionServer 后，便和 HMaster 没有关系了，开始了直接的数据传输，我们前面提到一个问题，那就是 HBase 如何防止数据丢失，毕竟 HBase 的写入是到内存，一次请求就返回了，解决这个问题是通过 WAL 日志文件来解决的，任何一次写入操作，首先写入的是 WAL，这类日志存储格式和 Kafka 类似的顺序追加，但是具有时效性，也就是当数据落盘成功，并且经过检查无误之后，这部分日志会清楚，以保障 HBase 具有一个较好的性能，当写完日志文件后，再写入 Memstore。
那么在 RegionServer 的写入阶段会发生什么呢？首先我们知道，HBase 是具有锁的能力的，也就是行锁能力，对于 HBase 来讲，HBase 使用行锁保障对同一行的数据的更新要么都成功要么都失败，所以在 RegionServer 阶段，会经过以下步骤：

1. 申请行锁，用来保障本次写入的事务性
2. 更新 LATEST_TIMESTAMP 字段，HBase 默认会保留历史的所有版本，但是查询过滤的时候始终只显示最新的数据，然后进行写入前提条件的检查：
![在这里插入图片描述](https://images.gitbook.cn/c0017510-4d43-11ea-aeb5-6d255c028296)
以上相关操作的代码都在 HRegion，RegionAsTable 中，可以以此作为入口去查看，所以这里就不贴大部分的代码了。
3. 写入 WAL 日志文件，在 WALProvider 中定义了两个方法：
![在这里插入图片描述](https://images.gitbook.cn/d0f16510-4d43-11ea-b4bd-6f2b05277b74)
append 用来对每一次的写入操作进行日志追踪，因为有事物机制，所以 HBase 会将一次操作中的所有的 key value 变成一条日志信息写入日志文件，aync 用来同步将该日志文件落盘到 HDFS 的文件系统，入场中间发生失败，则立即回滚。

4. 写入 Memstore，释放锁，本次写入成功。

所以可以看到对于 HBase 来讲写入通过日志文件再加 Memstore 进行配合，最后 HBase 自身再通过对数据落盘，通过这样一系列的机制来保障了写入的一套动作。

讲完了 HBase 的写入操作，再来看看 HBase 的读取流程，对于读来讲，客户端的流程和写一样，HBase 的数据不会经过 Master 进行转发，客户端通过 Master 查找到元信息，再根据元信息拿到 meta 表，找到对应的 Region Sever 直接取数据。对于读操作来讲，HBase 内部归纳下来有两种操作，一种是 GET，一种是 SCAN。GET 为根据 rowkey 直接获取一条记录，而 SCAN 则是根据某个条件进行扫描，然后返回多条数据的过程。可以看到 GET 经过一系列的判断，例如检查是否有 coprocessor hook 后，直接返回了存储数据集的 List：

![在这里插入图片描述](https://images.gitbook.cn/dd12b600-4d43-11ea-b4bd-6f2b05277b74)

那么我们再看 SCAN 就不那么一样了，可以看到，对于 SCAN 的操作来讲并不是一次的返回所有数据，而是返回了一个 Scanner，也就是说在 HBase 里面，对于 Scan 操作，将其分成了多个 RPC 操作，类似于数据的 ResultSet，通过 next 来获取下一行数据。

![在这里插入图片描述](https://images.gitbook.cn/e6af57e0-4d43-11ea-95be-cf33507ef652)


# HBase 文件格式

前面讲了 HBase 的操作流程，现在我们看下 HBase 的存储机制，首先 HBase 使用的 HDFS 存储，也就是在文件系统方面没有自身的文件管理系统，所以 HBase 仅仅需要设计的是文件格式，在 HBase 里面，最终的数据都是存储在 HFile 里面，HFile 的实现借鉴了 BigTable 的 SSTable 和 Hadoop 的 TFile，一张图先展示 HFile 的逻辑结构：

![在这里插入图片描述](https://images.gitbook.cn/ed9cdf50-4d43-11ea-aeaa-49f91881cc0c)

可以看到 HFie 主要由四个部分构成:
* Scanned block section：顾名思义，表示顺序扫描 HFile 时所有的数据块将会被读取，包括 Leaf Index Block 和 Bloom Block。
* Non-scanned block section：表示在 HFile 顺序扫描的时候数据不会被读取，主要包括 Meta Block 和* Intermediate Level Data Index Blocks 两部分。
* Load-on-open-section：这部分数据在 HBase 的 region server 启动时，需要加载到内存中。包括 FileInfo、Bloom filter block、data block index 和 meta block index。 
* Trailer：这部分主要记录了 HFile 的基本信息、各个部分的偏移值和寻址信息。

对于一个 HFile 文件来讲，最终落盘到磁盘上的时候会将一个大的 HFile 拆分成多个小文件，每一个叫做 block 块，和 HDFS 的块相似，每一个都可以自己重新设定大小，在 HBase 里面默认为 64KB，对于较大的块，在 SCAN 的时候可以在连续的地址上读取数据，因此对于顺序 SCAN 的查询会非常高效，对于小块来讲则更有利于随机的查询，所以块大小的设置，也是 HBase 的调参的一个挑战，相关的定义在源码里面使用的 HFileBlock 类中，HFileBlock 的结构如下所示：

![在这里插入图片描述](https://images.gitbook.cn/f490cf60-4d43-11ea-a2a8-bf3f3a718133)

每一个 block 块支持两种类型，一种是支持 Checksum 的，一种是不支持 Checksum 的，通过参数 usesHBaseChecksum 在创建 block 的时候进行设置：

![在这里插入图片描述](https://images.gitbook.cn/fdfcc450-4d43-11ea-a2a8-bf3f3a718133)

HFileBlock 主要包含两个部分，一个是 Header 一个是 Data，如下图所示：
![在这里插入图片描述](https://images.gitbook.cn/0d066820-4d44-11ea-aeb5-6d255c028296)

BlockHeader 主要存储 block 元数据，BlockData 用来存储具体数据。前面提到一个大的 HFile 会被切分成多个小的 block，每一个 block 的 header 都相同，但是 data 不相同，主要是通过 BlockType 字段来进行区分，也就是 HFile 把文件按照不同使用类型，分成多个小的 block 文件，具体定义在 BlockType 中，定义了支持的 Type 类型：

![在这里插入图片描述](https://images.gitbook.cn/131ca260-4d44-11ea-bb35-2725b4649300)

下面我们仔细分解一下 HBase 的 Data 部分的存储，HBase 是一个 K-V 的数据库，并且每条记录都会默认保留，通过时间戳进行筛选，所以 HBase 的 K-V 的格式在磁盘的逻辑架构如下所示：
![在这里插入图片描述](https://images.gitbook.cn/1b15a660-4d44-11ea-b4bd-6f2b05277b74)

每个 KeyValue 都由 4 个部分构成，而 Key 又是一个复杂的结构，首先是 rowkey 的长度，接着是 rowkey，然后是 ColumnFamily 的长度，再是 ColumnFamily，之后是 ColumnQualifier，最后是时间戳和 KeyType（keytype 有四种类型，分别是 Put、Delete、 DeleteColumn 和 DeleteFamily），而 value 相对简单，是一串纯粹的二进制数据。

最开始的时候我们介绍了布隆过滤器，布隆过滤器会根据条件减少和跳过部分文件，以增加查询速度：
![在这里插入图片描述](https://images.gitbook.cn/24a06df0-4d44-11ea-bb27-bdb4967fc6e0)
每一个 HFile 有自己的布隆过滤器的数组，但是我们也会发现，这样的一个数组，如果 HBase 的块数足够多，那么这个数组会更加的长，也就意味着资源消耗会更多，为了解决这个问题，在 HFile 里面又定义了布隆过滤器的块，用来检索对应的 Key 需要使用哪个数组：

![在这里插入图片描述](https://images.gitbook.cn/29a38a80-4d44-11ea-9aca-f5e54cebdcef)
一次 get 请求进来，首先会根据 key 在所有的索引条目中进行二分查找，查找到对应的 Bloom Index Entry，就可以定位到该 key 对应的位数组，加载到内存进行过滤判断。

# HBase RegionServer

聊完了 HBase 的流程和存储格式，现在我们来看一下 HBase 的 RegionServer，RegionServer 是 HBase 响应用户读写操作的服务器，内部结构如下所示：
![在这里插入图片描述](https://images.gitbook.cn/326312d0-4d44-11ea-9f6e-1bdc6229ab3f)
一个 RegionServer 由一个 HLog，一个 BlockCache 和多个 Region 组成，HLog 保障数据写入的可靠性，BlockCache 缓存查询的热点数据提升效率，每一个 Region 是 HBase 中的数据表的一个分片，一个 RegionServer 会承担多个 Region 的读写，而每一个 Region 又由多个 store 组成。store 中存储着列簇的数据。例如一个表包含两个列簇的话，这个表的所有 Region 都会包含两个 Store，每个 Store 又包含 Mem 和 Hfile 两部分，写入的时候先写入 Mem，根据条件再落盘成 Hfile。

RegionServer 管理的 HLog 的文件格式如下所示：

![在这里插入图片描述](https://images.gitbook.cn/37ff1400-4d44-11ea-aeb5-6d255c028296)

HLog 的日志文件存放在 HDFS 中，hbase 集群默认会在 hdfs 上创建 hbase 文件夹，在该文件夹下有一个 WAL 目录，其中存放着所有相关的 HLog，HLog 并不会永久存在，在整个 HBase 总 HLog 会经历如下过程：

* HLog 构建：任何写入操作都会先记录到 HLog，因此在发生写入操作的时候会先构建 HLog。
* HLog 滚动：因为 HLog 会不断追加，所以整个文件会越来越大，因此需要支持滚动日志文件存储，所以 HBase 后台每间隔一段时间（默认一小时）会产生一个新的 HLog 文件，历史 HLog 标记为历史文件。
* HLog 失效：一旦数据进入到磁盘，形成 HFile 后，HLog 中的数据就没有存在必要了，因为 HFile 存储在 HDFS 中，HDFS 文件系统保障了其可靠性，因此当该 HLog 中的数据都落地成磁盘后，该 HLog 会变为失效状态，对应的操作是将该文件从 WAL 移动到 oldWAl 目录，此时文件依旧存在，并未进行删除。
* HLog 删除：hbase 有一个后台进程，默认每间隔一分钟会对失效日志文件进行判断，如果没有任何引用操作，那么此时的文件会被彻底的从物理删除。

对于 RegionServer 来讲，每一个 RegionServer 都是一个独立的读写请求服务，因此 HBase 可以水平增加多个 RegionServer 来达到水平扩展的效果，但是多个 RegionServer 之间并不存在信息共享，也就是如果一个海量任务计算失败的时候，客户端重试后，链接新的 RegionServer 后，整个计算会重新开始。


# HBase 怎么用

虽然 HBase 目前使用非常广泛，并且默认情况下，只要机器配置到位，不需要特别多的操作，HBase 就可以满足大部分情况下的海量数据处理，再配合第三方工具像 phoenix，可以直接利用 HBase 构建一套 OLAP 系统，但是我们还是要认识到 HBase 的客观影响，知道其对应的细节差异，大概来说如果我们使用 HBase，有以下点需要关心一下：

1. 因为 HBase 在 RegionServer 对写入的检查机制，会导致客户端在符合条件的情况下出现重试的情况，所以对于较为频繁的写入操作，或者较大数据量的写入操作，推荐使用直接产生 HFlie 然后 load 到 HBase 中的方式，不建议直接使用 HBase 的自身的 Put API。
2. 从使用来讲如果业务场景导致 HBase 中存储的列簇对应的数据量差异巨大，那么不建议创建过多的列簇，因为 HBase 的存储机制会导致不同列簇的数据存储在同一个 HBase 的 HFile 中，但是 split 机制在数据量增加较大的情况下，会发生拆分，则会导致小数据量的列簇被频繁的 split，反而降低了查询性能。
3. RegionServer 是相互独立的，所以如果想要让集群更加的稳定高效，例如如果想实现 RegionServer 集群，达到信息共享，任务增量计算，需要自己修改 RegionServer 的代码。
4. 对于 HBase 来讲，很多场景下，像如果 Region 正在 Split，或者 Mem 正在 Dump，则无法进行对应的操作，此时错误信息会被以异常的形式返回到客户端，再由客户端进行重试，因此在使用过程中，需要结合我们的应用场景，考虑如何设置类似于 buffer 大小的参数，以尽可能少的降低因为内部操作引起的客户端重试，特别是在使用类似 opentsdb 的这类集成 hhbase 的数据的情况下。

# 结尾
HBase 有着非常庞大的架构体系，和较为不错的使用体验，因此使用一篇文章通常很难讲述清楚整个 HBase 内幕，但是我们可以根据理解一点点逐步的渗透到 HBase 内部，了解这个组件背后的原理，这样当我们在使用它的时候就会变得更加的得心应手。经验也不是一日构成，需要我们日复一日的不断联系，在 HBase 不断推出的新版下，琢磨和理解它的原理和架构。Apache 下有非常多的组件可以实现差不多的功能，但是每一个组件又有着自己独特的特点，本章我们介绍了 HBase，后续会逐步分解介绍像 Kylin，HDFS，Yarn，以及 Atlas 等组件。

