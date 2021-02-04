---
layout:     post
title:      "为什么要设计和开发RocksDB"
subtitle:   " \"Hello RocksDB\""
date:       2020-02-04 12:00:00
author:     "Dhruba Borthakur"
header-img: "img/post-bg-2015.jpg"
tags:
    - RocksDB
---

> “Yeah It's on. ”

- 翻译自[The History of RocksDB](http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html)

## 过去

现在已经是2011年中旬，我已经从事HDFS/HBase的研发5年了，我也很喜欢Hadoop的生态。Hadoop是一个无需眨眼即可存储和查询数百PB数据的系统。Hadoop始终将重点放在可伸缩性上，且Hadoop的可伸缩性也在突飞猛进地增长。 显然，几年后，我们将能够在单个Hadoop集群中存储EB级数据。在这里，我想面对一个新的挑战...看看我们是否也可以将HDFS的成功案例从数据分析扩展到Query Serving workloads。 

Query Serving workload通常包括数据查找、随机小范围的数据扫描、数据更新，该类型的workloads的核心诉求是低延迟的查询。然而大数据分析查询通常涉及到大批量数据顺序扫描和多个数据集合连接，且这些数据集的更新量极低。因此，我比较了HBase/HDFS和MySQL在运行Query Serving workload方面的差异，HBase的优点是可以在一个数据中心内拥有多个数据副本。我尝试将包含几PB数据集的Query Serving workload从MySQL集群迁移到HBase/HDFS(该数据存储与硬盘上)，经对HBase进行多轮增强后，可使HBase延迟仅是MySQL服务器的两杯，且HBase仅使用三倍多的IOP在相同的硬件上提供相同的工作负载。我们正朝着目标稳步前进，但是Flash Storage的出现打破了这种稳定。

[Flash Storage](http://en.wikipedia.org/wiki/Flash_memory)成为现实后，数据集可以从硬盘迁移到闪存，现在出现的问题是HBase是否可以有效使用闪存硬件，我用闪存上的数据对HDFS和HBase进行了基准测试，并将结果发布在[较早的文章]中（http://hadoopblog.blogspot.com/2012/05/hadoop-and-solid-state-drives.html）。结果是2012年的HDFS / HBase存在一些软件瓶颈，因此无法有效使用闪存。很明显，如果数据存储在闪存中，那么我们需要一个新的存储引擎，以便能够有效地为该数据提供随机工作负载。 我开始寻找建立下一代键值存储的技术，这些技术专门用于从快速存储中提供数据



## 为什么需要嵌入式数据库

闪存在性能上与旋转存储根本不同。为了便于讨论，我们假设对硬盘的读取或写入大约需要10毫秒，而对闪存的读取或写入大约需要100微秒。两台机器之间的网络网络等待时间保持在50微秒左右。这些数字并不是一成不变的，您的硬件可能与此不同，但是这些数字说明了两种情况之间的相对差异。客户端希望存储和访问数据库中的数据，有两种选择，它可以将数据存储在本地连接的磁盘上，也可以通过网络将数据存储在连接了磁盘的远程服务器上。如果考虑延迟，那么本地连接的磁盘可以在大约10毫秒内满足读取请求。在客户端-服务器体系结构中，通过网络访问相同的数据会导致10.05毫秒的延迟，而网络所带来的开销只有0.5％的微不足道。考虑到这一事实，很容易理解为什么当前部署的大多数系统都使用访问数据的客户端-服务器模型。 （出于讨论目的，我忽略了网络带宽限制）。

现在，让我们考虑相同的情况，但将磁盘替换为闪存驱动器。 在本地连接的闪存中，数据访问为100微秒，而通过网络访问相同数据为150微秒。 网络数据访问的开销比本地数据访问高50％，而50％的开销是相当大的。 这意味着，与通过网络访问数据的应用程序相比，在应用程序中嵌入式运行的数据库的延迟可能要低得多。 因此，需要[Embedded Database](http://en.wikipedia.org/wiki/Embedded_database)

上述假设并未说明客户端-服务器模型将消失。 客户端-服务器-数据访问模型在数据管理方面具有固有优势，并且在应用程序部署方案中将继续保持突出地位。

## 现在有哪些嵌入式数据库

当然，有许多现有的嵌入式数据库：[BerkeleyDB](http://www.oracle.com/technetwork/products/berkeleydb/overview/index.html)，[KyotoDB](http://fallabs.com/kyotocabinet/ )，[SQLite3](http://www.sqlite.org/)，[leveldb](https://code.google.com/p/leveldb/)等。[开源基准程序](http：/ /leveldb.googlecode.com/svn/trunk/doc/benchmark.html)测试结果似乎表明leveldb是最快的。但并非所有这些都适合在Flash存储器上存储数据,  Flash具有有限的写持久性；更新Flash上的数据块通常会在Flash驱动程序中引入写放大。鉴于我们要在闪存上运行数据库，我们专注于测量写放大率以评估数据库技术。HBase和[Cassandra](http://cassandra.apache.org/)是日志结构合并（[LSM](http://en.wikipedia.org/wiki/Log-structured_merge-tree)）样式数据库，但它要使HBase和Cassandra成为可嵌入的库，将需要大量的工程工作。它们都是具有内置管理，配置和部署的整个生态系统的服务器。我正在寻找一个简单的c/c ++库：leveldb是进行基准测试的明显首选。



## 为什么LevelDB无法满足我们的需求

我开始对leveldb进行基准测试，发现它不适合Querying Server workloads,  Leveldb是一项很酷的工作，但并非为服务器工作负载而设计。 开源的[基准测试结果](http://leveldb.googlecode.com/svn/trunk/doc/benchmark.html)乍看之下很棒，但我很快意识到这些结果是针对数据库比计算机内存小的场景，即整个数据库必须适应于OS页面缓存。 当我在至少比主内存大5倍的数据库上执行相同的基准测试时，性能结果令人沮丧。Leveldb的单线程压缩过程不足以驱动服务器工作负载，频繁的写入停滞导致99％的延迟非常大。 将文件M映射到OS缓存中会带来读取的性能瓶颈。 Leveldb无法使用基础闪存存储提供的所有IO。

另一方面，我看到服务器存储硬件在不同维度上快速发展，例如，一个实验系统，其中存储卷跨10个闪存卡条带化，每秒可提供高达一百万个IO，基于NVRAM的存储每秒可支持数百万次数据访问，我想使用一个可以驱动这些类型的快速存储硬件的数据库。 闪存的自然发展可能会导致我们拥有非常有限的擦除周期的存储，而我预想，对于这些类型的存储，迫切需要一个能够在读取放大，写入放大和空间放大之间进行权衡取舍的数据库。 Leveldb并非旨在实现这些目标，最好的方法是派生leveldb代码并更改其体系结构以满足这些需求。 因此，RocksDB诞生了！



##  RocksDB愿景

- 具备范围扫描、point lookups功能的Key-Value类型数据库
- 针对快速存储进行了优化，例如 闪存和RAM
- 具有完整生产支持的服务器端数据库
- 与CPU内核数和存储IOP线性扩展

RocksDB不是分布式数据库。 它没有内置的容错或复制功能， 它不支持数据分片，需要依赖使用RocksDB的应用实现容错、分片功能。



## RocksDB架构



RocksDB是一个C ++库，可用于持久存储键和值， 键和值是任意字节流。Key以排序的方式存储(`sorted runs`), 

新的写入会出现在存储中的新位置，并且后台压缩线程主要用于消除数据重复、标记数据删除flag, 支持以原子方式将一组Key写入数据库,  支持在向后和向前迭代。RockDB使用“可插拔”架构构建,  这样可以轻松地替换其中的一部分，而不会影响系统的整体体系结构。 这种架构使我充满信心，RocksDB可以轻松地针对不同的工作负载和不同的硬件进行调整。例如:

- 可以插入各种压缩模块（snappy，zlib，bzip等）而无需更改任何RocksDB代码
- 应用程序可以插入自己的压缩过滤器以在压缩过程中处理Keys， 一个示例应用程序可以使用它为数据库中的key实现一个`expiry-time`
-  RocksDB具有可插入的[memtables](http://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)，以便应用程序可以设计自定义数据结构来缓存其写入内容 ，其中一个示例是prefix-hash-memtable，该示例对键的一部分进行哈希处理，而键的其余部分以btree的形式排列
-  [sst](http://www.igvita.com/2012/02/06/sstable-and-log-structured-storage-leveldb/)文件的实现也是可插入的，应用程序可以为它们设计自己的格式 sst文件,  RocksDB支持MergeType记录，该记录允许应用程序通过避免 read-modify-write来构建更高级别的构造(Lists, Counters, etc).

RocksDB当前支持两种压缩方式：`level style`压缩和`universal style`压缩，这两种样式提供了灵活的性能折衷，压缩本质上是多线程的，因此大型数据库可以维持较高的更新率。 我将针对这两种压缩方式的优缺点写一篇单独的文章。 

RocksDB提供了用于增量在线备份的接口，这是任何生产应用所必需的，它支持在key的sub-part设置Bloom过滤器，这是减少范围扫描所需iops的一种可能方法。

RocksDB的api是[stackable](https://github.com/facebook/rocksdb/blob/master/include/utilities/stackable_db.h)，这意味着您可以使用易于使用的高级包装器包装较低级别的api 。 这是未来帖子的主题。

## RocksDB潜在的用例

- 需要低延迟数据库访问的应用程序可以使用RocksDB
-  存储网站的查看历史记录和用户状态的面向用户的应用程序可以将该内容潜在地存储在RocksDB上
-  需要快速访问大数据集的垃圾邮件检测应用程序可以使用RocksDB
- 需要实时扫描数据集的图形搜索查询可以使用RocksDB
-  RocksDB可用于从Hadoop缓存数据，从而允许应用程序实时查询Hadoop数据
-  支持大量插入和删除的消息队列可以使用RocksDB。

## The Road Ahead

- 当整个数据库都装入RAM时，RocksDB能够以内存速度提供数据服务
- 兼容高度多核的计算机系统
-  随着超低耐久性闪存存储的出现，我们期望RocksDB以最小的写放大来存储数据
- 也许在将来的某个时候，会有人将RocksDB移植到Android和iOS平台
- 另一个不错的功能是支持[Column Families]（http://en.wikipedia.org/wiki/Column_family）以更好地支持相关数据的聚类
- 我希望软件程序员和数据库开发人员可以针对其用例使用，增强和自定义RocksDB

RocksDB代码位于http://github.com/facebook/rocksdb中， 您可以加入Facebook Group https://www.facebook.com/groups/rocksdb.dev来参与有关RocksDB的工程设计讨论。

## 著作权声明

本文译自 [The History of RocksDB](http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html)
