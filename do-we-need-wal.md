# 分布式思考：我们需要WAL吗？

## 前言 

历史上看，是先有单机系统，然后再向分布式演变。我们很多分布式的设计，都源自单机的很多经验。这是没错的，因为分布式本来就是众多单机参与的一个集群系统。我们当然要借鉴单机的经验和单机的基础。

但是，分布式源自单机，但不能思想受限于单机。

我的观点：分布式要学习和复用单机的知识，但在一些领域，要勇敢跳出单机的思维限制。

再看一个思考：我们需要WAL吗？

## 单机为什么需要WAL？

WAL，write ahead log，是为了优化磁盘性能的一个方法，广泛用于数据库系统（你几乎找不到不支持WAL的数据库系统）

磁盘是慢速的，但磁盘的慢速，不是简单的一个点，而是有很复杂的机理。

大家可以参考我之前的一个文章：

[Throughput, Bandwidth, Latency](throughput-bandwidth-latency.md)

里面有一个磁盘在不同Pattern下的测试数据，可以看到，对于不同的block size，磁盘的Throughput有几十倍的差别。

同时，磁盘还有一个写放大问题，比如：如果我们通过B+树在内存保存数据，如果将某个修改的内存Page即使刷盘的话，会有很大倍数的写放大，很不经济。

大家可以参考另外一篇文章

[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)，里面有对MySQL redo的分析：用Log写盘，来替代直接内存Page写盘，所带来的好处。

简而言之，就是利用磁盘对于Log效率高的特性，让数据的修改，分在两次（即冗余），一次在WAL，并很快落盘；一次在你真正dataset（如B+树的内存页），晚些落盘，从而提高整个系统的Throughput。这样，万一发生灾难，你可以用WAL挽救你的数据。

WAL还有另外一个作用，就是ACID中A的保证。因为data page并不能保证Atomic（数据库可能分开写盘、也可能部分Partial写盘）。

于是，你可以看到，在很多数据库系统里，WAL是无处不在，比如：MySQL的redo log，RocksDB的WAL，etcd的WAL。都是这个思想。

## 分布式下还需要WAL吗？

首先，你从上面的分析可以看出，WAL是冗余的。数据被写了两次。

如果理论上不存在灾难，我们完全可以不用WAL。

请你再参考我之前的一个分析：

[分布式下：我们还需要fsync吗](do-we-need-fsync.md)，你会发现，对于WAL，我们可以用同样的思路。因为：

1. 数据已经在内存的data structure里存在多份（多机），我们可以满足用户不丢数据的承诺（只要两台机器不同时死，而且我们认为两台同时死的概率趋近于零）

2. 我们不用急着对内存的data structure刷盘，从而一样获得上面用WAL的那些好处：数据页可以多次修改但一次刷牌，数据如果连续可以做类似Log式的刷盘（比如LSM下的SST就是数据连续的）

所以，我的倾向，在分布式系统里，我们可以去除WAL。

## 一个这样实践的案例BunnyRedis（还有Kafka）

[BunnyRedis](https://zhuanlan.zhihu.com/p/392646113)，就是省掉了WAL。虽然它的磁盘存储用了RocksDB，但RocksDB有开关可以关闭WAL，而BunnyRedis的代码是不写WAL的。

你可能会问，BunnyRedis不是用到了Kafka，那个Kafka Log不就是BunnyRedis的WAL吗？

是的，Kafka的Log，相当于BunnyRedis的WAL。但是，这个WAL并不是常规数据库里的那个意义的WAL。

因为：

1. 理论上，我可以关闭Kafka的写盘，只要至少一台bunny-redis进程活着，就会有一个dataset被bunny-redis进程所拥有。

2. Kafka收到bunny-redis的数据，它并不很快落盘。WAL对于很多数据库系统是至少每秒必须落盘一次（至多丢1秒或2秒的数据），但对于Kafka，完全依赖操作系统的后台刷盘，时间可以到几十分钟。也就是说，Kafka Log, 作为BunnyRedis的WAL，它不在磁盘上，而在内存上。

3. 而且BunnyRedis这个所谓的WAL不会丢数据，只要一台Kafka broker还活着。

对于Kafka而言，它的数据，就是它的Log（所以，Kafka没有WAL）。

而且，Kafka依赖操作系统后台写入磁盘的速度，几乎等同于一般网络的Throughput，这个没有任何瓶颈。

详细可参考Kafka官方说明：[Don't fear the filesystem!](https://kafka.apache.org/documentation/#design_filesystem)
