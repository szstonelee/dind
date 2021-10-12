# 内存、网络、磁盘性能比较

## IO分类

狭义讲，IO分两类：

1. 网络 Network

2. 磁盘 Disk or Storage

广义讲，还应该加上主板内存Main Memory，有时我们也叫RAM，或者Memory

## IO效率定义

效率，我这里理解为性能Performance，它可以有很多指标，但一般我们主要观察两个

1. Latency 时延

2. Throughput (吞吐) or Bandwidth （带宽）

Latency是从时间的角度看，即一个动作，需要多少时间完成。

Throughput（含Bandwidth）是一段时间完成的动作数，一般是1秒钟完成多少数量，然后，我们将每个IO动作所产生的效果字节化，以方便统一对比。

## 各IO的Latency

随着硬件的提高，值每年能都在变化，大家可参考

https://colin-scott.github.io/personal_website/research/interactive_latency.html

里面数据有些容易迷惑人，但我们抓几个重要的：

1. 内存访问Latency = 100 ns

2. 局域网LAN的包的一来一回，100~200 us (2020年，文中是Datacenter，且是500us)，同时我们不考虑广义网WAN的Round Trip，因为受影响因素太多，而且Latency很大（一般估计几十ms到几秒)

3. SSD的一次访问时延 几十个us到几百个us（2020年，文中是16us）

## Latency、Throughput、Bandwidth，如何定义，有什么关系

请参考下面这个文章：https://zhuanlan.zhihu.com/p/399883427

## 各IO的Bandwidth

Memory；几个GB/s到100GB/s，一般我们认为至少10GB/s

Network：网卡（NIC）有多种规格，有10Mb/s, 100Mb/s，1Gb/s，10Gb/s, 20G/b, 100Gb/s。我们一般除10换算成Byte（因为还有校验位），所以范围是1MB/s到10GB/s。而超过1GB/s，一般用RDMA网络，所以我们一般认为，高端的常规网卡是1GB/s（万兆网卡），低端的是100MB/s（千兆网卡）

Disk: 有HDD和SSD，这里重点是SSD，因为它比HDD快，是未来的趋势。

SSD又有SATA， SAS, NVMe(PCIe)这些接口，而SSD内部可以用NAND和Optane。然后厂家列出的Bandwidth，还有不同模式，比如：是读Read还是写Write，是顺序Sequential的，还是随机Random的

但一般而言，我们认为最好的SSD是1-2GB/s这个量级，一般的SSD是几百兆Byte/s

## 为什么不列出各IO的Throughput？

因为Throughput受你的应用的模式的影响。

打个比方，我们访问主存Memory，我们知道100ns只访问一次，如果每次我们只获得1个字节的有效数据，那么请问，你的Throughput是多少？

答案是：10MB/s。

这超越了我们的理解（内存不可能这么低效），关键是，我们不知道我们的程序读写内存时，有效字节的数量。

如果你说，假设主存的每次访问都是100%有效，但另外一个问题就来了，CPU cache。

即你每次访问主存，系统都会将读到主存的内容缓存到CPU cache，如果你下次再读，或者读相邻的另外地址内容，你拿的数据来自CPU cache，而不是Main Memory。而CPU cache有多级，比如L1到L3，而且各个机器的CPU cache大小配置不一样。

而L1的Latency是小于1ns的，L2的Latency一般是几个ns。

但不管如何，我们通过上面内存的Bandwidth以及CPU cache可以大致可以有下面这个信心：

**内存Memory的Throughput一般会远远高于磁盘Disk和网络Network的Throughput**

## 评估性能到底用哪个指标？

答案是：一般而言，Throughput

Latency和Throughput有一定的关联，某些情况下，甚至是线性相关，但上面那个文章 https://zhuanlan.zhihu.com/p/399883427 以及上面那些分析，我们发现一个大麻烦：

**Throughput是不可控的，它受很多因素影响**

## 磁盘和网络的Throughput在数据库应用的比较

从上面可以看出，如果用Bandwidth，磁盘和网络好像差不多，都是GB/s。

但我们知道，我们要真正比较的指标是Throughput。

而对于数据库应用，磁盘在这里又有一个大的消耗。

因为我们放在磁盘上的数据，如果是数据库应用，它必须能支持很快的查询。

所以，我们不能简单地将数据存储在磁盘上（比如Log），我们必须将这些数据转成一个可以快速查询的数据结构。

一般是B+树或者LSM树（还有一个特例，Hash Table存储，但它受限很多，未广泛使用）

如果是B Tree或LSM Tree的话，我们就有一个读放大和写放大的问题。

简而言之，你如果存入1M的数据库数据到磁盘上，如果转成上面这两个树，它将产生很多损耗，即磁盘真正为1M Byte有效数据的读写可能是几十兆字节，也可能是几百兆字节：

1. 浪费（比如：Page不能占满）

2. 重复（比如：需要后台不停地压缩、或者读一个key需要查多次）

3. 高效模式不可用（比如：不能用sequential模式，而只能用random模式）

以上，对于磁盘，对于数据库应用，将使Bandwidth真正能转为磁盘的Throughput的有效利用率非常低。一般认为，至少10倍以上的浪费。

而网络不存在这个问题。

所以，我们可以得到下面这个结论：

**一般而言，对于数据库应用，磁盘的Throughput低于网络Throughput，即瓶颈在磁盘**

## 最后的结论

一般而言，我们看Throughput来判定性能

对于数据库应用，我们可以假设

**Memory Throughput > Network Throughput > Disk(SSD) Throughput**

而且这个大于号，是至少10倍的差别。

所以，对于数据库的应用，关键是优化磁盘的Throughput，但这个受影响的因素太多了，也就是数据库这么麻烦的地方。

## 补充：为什么不在单机上App和Redis并用（问题里所涉及）

问题连接：https://www.zhihu.com/question/47589908/answer/2166566017

将你的应用App，和Redis放在一个机器上，不是不可以。但单机的麻烦在于：

1. 你的App和Redis都会竞争一个机器的资源，包括CPU和内存
2. 如果你的App有问题，会让Redis一起损坏，比如App导致OS奔溃或内存耗尽

但也不是不能这样做，如果你的App非常在意Latency的话，将App和Redis放在一个机器上，可以达到这个效果，可以走Unix Socket，将使Latency降低，这样Round Trip，从几百个us可以降低到几十个us


