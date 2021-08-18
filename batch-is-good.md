# 分布式思考：批发Batch是个好东西，用足它

## 前言

你家是大户，你有辆五座小轿车，过年了，你要到火车站去接来访的亲戚。如果来了小妹，你可以马上接她，让她快点到家早点休息。但你付出的代价cost是时间和汽油费。你也可以稍微等一下，因为下趟车的舅妈一个小时候后就到，你可以两个人一起接上，唯一的麻烦可能是小妹会埋怨你几句，让她干等1个小时（Latency变长了），但trade off是你省了路上的两趟时间和两次汽油。

但是，如果某一趟来了一大家人，你的叔叔家共三人，你肯定不会分三次，每次只在车中放一个乘客。否则，你就是个大傻子。

计算机世界里，也是同样的道理。

这个优化，我们叫做Batch。

## 磁盘上的Batch

在[Throughput, Bandwidth, Latency](throughput-bandwidth-latency.md)里，我有个测试，对磁盘做同样的写，只是block size不同，一个是4K，一个是1024K，磁盘的Throughput差别竟然大到超过60倍。

这就是Batch的力量。因为如果是4K一次提交，磁盘这个轿车，就只能载4K这样一个人。而1024K一次提交，磁盘这个轿车就可以满载而归。

## 网络上的Batch

网络上同样存在Batch现象。

对于TCP/IP，一个数据包package，如果跑在底层Ethernet协议上，最大可以到1500字节。但如果你只发送一个字节一个包，TCP/IP也得像上面的小轿车一样去跑，而且还得在里面加上40字节的head信息（TCP用20字节、IP用20字节），这相当于上面的汽油。

所以，TCP/IP里有[Nagle算法](https://baike.baidu.com/item/Nagle%E7%AE%97%E6%B3%95/5645172)，专门对这个进行优化。

## 操作系统和应用软件的Batch

比如：RocksDB就提供Batch的专门接口:

对于写，有[WriteBatch](https://github.com/facebook/rocksdb/wiki/Basic-Operations#writes)

对于读，有[MultiGet](https://github.com/facebook/rocksdb/wiki/MultiGet-Performance)

Redis也用到了，可以参考[Redis Pipeline](https://redis.io/topics/pipelining)

Kafka用得更多，比如：

1. IO上的Batch，参考[Kafka Efficiency](https://kafka.apache.org/documentation/#maximizingefficiency)

2. [Batch Compression](https://kafka.apache.org/documentation/#design_compression)

3. [RdlibKafka High Performance](https://github.com/edenhill/librdkafka/blob/master/INTRODUCTION.md#performance)

## 一些常规代码里也经常用到Batch

比如：我们有一个动态数组（假设是整型），传入应该是空的，我们需要初始化连续的数字（从零开始）

C++
```
void init_continuouse_nums(const int count, std::vector<int>& nums)
{
  assert(count > 0 && nums.size() == 0);


  for (int i = 0; i < count; ++i)
  {
    nums.push_back(i);
  }
}
```

Java
```
void initContinuouseNums(int count, ArrayList<Int> nums) {
  Preconditions.checkArgument(count > 0 && nums != null && nums.size() == 0);

  for (int i = 0; i < count; ++i) {
    nums.add(i);
  }
}
```

这个代码是不够优化的，因为动态数组涉及可能发生多次内存的分配，我们完全可以做一次批处理

对于C++，加入下面的代码
```
  nums.reserve(count);
```

对于Java，加入下面的代码
```
  nums.ensureCapacity(count);
```

有兴趣的同学可以benchmark上面不同代码的latency，我想结果对比会让你自己吓一大跳。

## Batch有两种：Pipeline和Group(Aggregation)

我们要知道，Batch有两种，一种是Pipeline，一种是Group(Aggregation)

### 什么是Pipeline

所谓Pipeline，是站在一个客户端角度而言，将许多个请求，不进行一次又一次的提交，而是一起提交。即先buffered第一个请求，直到一定条件（时间到或buffer字节满或者程序配置的请求数），再一起提交。

因为，一般一次提交，客户端都需要等待服务器有了回应response，才能进行下一次请求。

这样，就能提高Throughput，起码省了多次通信的Round Trip（还有服务器也可以进行批处理，省了很多cost）

### 什么是Group(Aggregation)

Group(Aggregation)是站在服务器角度，当收到一个客户提交的请求，并不马上处理，而是先缓存起来（buffered）。等待一个足够的时间或一个足够的限额（比如：buffer超过一定的字节数），然后才一起处理请求（一般返回结果Response也是一起处理）。

### Pipeline和Group的应用场景

Pipeline需要客户端有一定的智能化，也就是说，客户端要buffer请求。

而Group可以省掉客户端的这个智能化，将buffer逻辑放在服务器端，但必须对于多个并发客户才能有效。如果客户端足够多，请求并发足够大，那么Group的效果要好于Pipeline，因为客户端不用等待再批处理发送（客户端因此可能降低Latency，如果和客户端同时也做buffer对比）。

## Batch的trade-off

Batch是个好东西，不代表它没毛病。

首先是复杂了代码，其次是可能加大Latency。不管是基于客户端的Pipeline，还是基于服务器端的Group，都需要等待一定的时间（或者超过一定字节的buffer）。这就会延长单个请求的Latency，特别是第一个请求的Latency。

但如果系统能接受一定程度的Latency牺牲，这个Batch所带来Throughput很可能会有极其大的提升，那么这个trade-off就值得。

## 分布式下的结论

分布式由于有单点和共识的代价，请参考[分布式下一致性的代价](cost-of-consistency.md)，所以，如果是来一个请求就处理一个，我们的牺牲很大，你能观测到的Throughput会非常低。

举个例子：etcd，如果是单次小字节请求，Throughput并不高，低可能到几百，高也就几K。但etcd采用了Batch优化，可以在一些场景下（符合Batch的条件，比如：客户端并发数足够多）将这个Throughput提高到几十K。

所以，**分布式下，我们应该尽量用足Batch，它确实是个好东西**。

## BunnyRedis的实践

BunnyRedis除了用到Redis的Pipeline，以及RocksDB的batch，Kafka的Batch，它还利用了Redis的Transaction实现了Batch功能，提高了Throughput。

因为对于Redis的写命令，BunnyRedis必须通过Kafka做到强一致，而这个逃不掉[分布式下强一致的代价](cost-of-consistency.md)。

但如果做Transaction，BunnyRedis就可以在保证强一致的前提下，让效率得以提升。

比如：一个Transaction如果有10个Write命令，那么如果分开写，BunnyRedis需要通过Kafka做10次同步，但如果放在Transaction里，则只需要一次同步。

测试表明，这可以带来9倍的提高。详细可参考：[通过Pipeline和Transaction提高BunnyRedis的Throughput](https://zhuanlan.zhihu.com/p/392787651)