# 分布式思考：我们需要Shard吗？

## Shard是个好东西

Shard是个好东西，对于分布式系统，它将数据分开，存储在不同机器上。

这样，我们就可以利用到集群的多机，很好地进行scale-out。

Shard的管理有点麻烦，但通过一些手段我们可以解决：

1. [Consistenct Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)

2. 类似Consistency Hash的思想，采用固定数目的Slot，比如：[Redis Cluster](https://redis.io/topics/cluster-tutorial)

3. 用更复杂的机制，在shard的管理上，采用一个meta data store，来管理各个segment的转移、合并、删除等

## Shard有它的麻烦

Shard对于scale out是个非常好的方案，而且是性能如果到达瓶颈时的最终解决方案。

但是，如果我们的某个操作，涉及两个shard，就是大麻烦，包括：

1. Join

2. Distributed Transaction

对于Join，我们要到两个机器上去拿数据，因此，还需要一个中心店，做聚合Aggregation操作，同时，如果其中一个shard fail了，怎么办？

对于Distributed Transaction，当前主流做法是2PC -- Two Phase Commit。但是，这是个效率很低的解决方案，而且有单点故障。Google虽然通过Precolater做了一定程度的优化，即将undo延后处理，来减少Latency，从而提高Throughput，但是，它仍没有解决单点故障的问题，同时代价还是很大（为了Isolation的支持，还需要通过分布式的硬件时钟，或者集中式的软件时钟来保证顺序）。

## 我的观点

软件学说里有一句名言：**再没有达到瓶颈Bottleneck时，不要随意优化Optimization**

类似地，我认为：**在一个shard没有达到瓶颈Bottleneck时，不要轻易地分片Shard**

因为分片shard带来的麻烦，如Join，Distributed Transaction，都是不可避免的。而如果没有Shard，那这些麻烦都将不用考虑，从而简化我们的分布式系统。

即分布式系统并不只是一开始就进行分片Shard。

## 我们当前一个数据的瓶颈达到了吗？

比如：我们的一个数据库服务器，我们发现它只能最大支持100K qps，于是，我们就认为，它的瓶颈到了，必须shard了。

对吗？

我们看一个正常的互联网应用，其访问基本准从大部分访问是Read，小部分访问是Write。

如果90%的访问是Read，10%的访问是Write，那么，我们完全可以用十个机器来做一个集群，每个机器都有同样一份数据，那么就算访问量大到10倍，即1M qps，只要每台机器能支持100K Write qps，我们是不是就不用Shard。

你会说，Write会破坏一致性，不能保证Write时，十个机器的数据都是一致的。

但如果解决了呢？

[BunnyRedis要解决Redis的一致性Consistency问题](https://zhuanlan.zhihu.com/p/392637293)就是尝试解决这个问题的一个实践。即分布式下，如何保证数据的一致性。

你会再说，这个BunnyRedis不过是解决key/value，这个对于Key/value适用，对于Relational Database不适用。即在我的文章[BunnyRedis的一致性Consistency](https://zhuanlan.zhihu.com/p/392653517)里，我只解决了右半边的问题，没有解决左半边的问题，比如：Read Committed, Repeatable Read, Serializable这些问题。

是的，我正准备这些问题。至于如何解决，因为我还没有做出来，来证明我的观点，所以，我现在暂时还不能说出来（没有产品证明的，不适合发表）。

但是，我认为，这是可以解决的问题。

同时，我前面发表的一些分布式优化，你也可以参考一下，如果你的单shard在这些优化下，都能做到不出现瓶颈，那么，你就不必急着分库分表，做shard。

* [分布式下思考：我们需要fsync吗](do-we-need-fsync.md)

* [分布式下思考：我们需要WAL吗](do-we-need-wal.md)

* [分补水下思考：我们需要磁盘吗](do-we-need-disk.md)

* [分布式下思考：批处理Batch是个好东西](batch-is-good.md)

* [分布式下思考：Locality是个好东西](locality-is-good.md)

* [分布式下思考：锁是个麻烦](lock-is-bad.md)

* [分布式下思考：如何提高性能](how-improve-throughput.md)

* [分布式下思考：少就是多，多就是少](less-is-more.md)