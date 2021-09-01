# 作为数据的Log和WAL的本质区分

## 问题的由来

在计算机世界里，定义一个东西很难。因为大家有不同的理解，而针对中文，又产生了一个新的麻烦，原因两个：

1. 计算机的发源和当前现状，都主要来自英文，所以，需要从英文转译成中文。本来英文就有多个近似词义的表达（比如：master vs leader），中文翻译又可以一翻多（因为存在多个译者），这就让中国用户的理解更容易出现混乱。

2. 中文本身的特点，具有广义性。即可以：比喻、联想、发散。相比而言，英文在表达上更收敛，但一样存在一词多意，而且你会发现，对于同一object，英文会有很多词去描述，而里面有非常细微的差别（而且只有美国人，有时甚至只有英国人知道，i.e., 美国人的英语不如英国人）。比如：好。我们用英文夸一个想法很好是：Your idea is good. 但实际上，这个good在英文很多场景下，只是中文的“一般般”或“麻麻地”的意思。

在我之前的一些文章里，很多涉及Log，同时又用到一个WAL（Write Ahead Log），因此，也容易产生混淆

比如：

[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)，通篇都是Log。然后又提到redo是一个log

[分布式思考：我们需要WAL吗？](https://zhuanlan.zhihu.com/p/400338569)，通篇都是WAL，但又提到Kafka没有WAL

因此，我们有必要给Log和WAL做个准确的区分

## 那么Log (as dataset，作为真正的数据) 和 WAL 到底有什么区别？

### 1. 落盘次数

数据data可以放在WAL里，也可以放在dataset里，一般而言，它们最后都需要写到磁盘上（落盘）。

如果用到WAL，则数据需要落盘两次，一次到这个WAL对应的log文件里（或文件组），一个到dataset对应的文件（或文件目录）里（而且一般不连续，以方便查询，比如B+树或LSM树）。即WAL是冗余的。

所以，[BunnyRedis](https://zhuanlan.zhihu.com/p/392646113)，作为一个整体的系统（请参考[BunnyRedis架构](https://zhuanlan.zhihu.com/p/392652895)），它是存在WAL的，因为数据落盘两次，一次是在Kafka里，一次是在bunny-redis下的RocksDB目录下。

所以，站在BunnyRedis整个系统来看，Kafak里存的数据，就是WAL，而RocksDB里存的数据，就是dataset（或者叫State Machine）。

但是，BunnyRedis从结构上看，是两个子系统（都是集群），一个是Kafka集群，一个是bunny-redis集群。

如果我们只看Kafka这个集群（即不考虑bunny-redis集群），它只落盘一次，所以，它没有WAL，或者说，Log就是它的dataset，即dataset == Log。

如果我们只看bunny-redis集群（尽管它必须依赖Kafka，并且把Kafka的数据当做它的WAL），它本身的存储，即RocksDB，并没有写两次（虽然RocksDB是支持自身WAL的，但可以关闭，而bunny-redis代码中是关闭RocksDB的自身的WAL）。所以，bunny-redis也是没有WAL的。

### 2. 落盘的时间紧急度

如果是WAL，它主要目的，是为了利用Log的特性快速落盘并防止灾难时丢失太多的数据。

如果你一点数据也不想丢，那么你每次写入WAL（先到内存WAL buffer），都要立刻fsync到磁盘上。

如果你只想丢一点数据，一般设置是每秒fsync WAL，这样，最多丢失1秒到2秒的数据，但性能相比上面的每次写WAL都fsync有很大的提升。一般而言，[Throughput](https://zhuanlan.zhihu.com/p/399883427)都有10倍的提升。

你当然可以设置WAL落盘用操作系统后台写入的方式（flush from page cache），而这一般是分钟级，但也意味你可能丢失分钟级的数据，所以，这个做法很少见，或者说，失去了WAL的意义，因为dataset自己本身就是通过后台分钟级的落盘。

所以，判断WAL还是Log as dataset，还有一个出发点，看落盘的时间的即时性。如果很快落盘，一般是WAL，如果不是，那一般是dataset。

所以，从这个角度看：

Kafka的Log落盘是后台分钟级（当然，你可以配置改变）。所以，Log对于Kafka，是dataset，而不是WAL。

bunny-redis的RocksDB落盘也是后台分钟级，RocksDB对于bunny-redis，也是dataset，而不是WAL。

### 3. 落盘是否顺序(order or sequence)

数据data写入是有时间先后序的，比如：t1时刻早于t2时刻，那么t1时刻对应的data，也应该早于t2时刻的data。

对于WAL，它应该保证落盘的时序，和数据写入的时序，是一致的，即t1时刻产生的WAL应该早于t2时刻产生的WAL先落盘。

但对于dataset，则没有这个强制要求。有可能t2时刻对应的dataset先落盘（或者部分先落盘，比如B+树里的page页）。

而如果你依赖操作系统的后台写入，它是不保证时序的，即操作系统眼里，page cache是dataset。

从这个角度看：

Kafka写入的Log，是dataset；bunny-redis，它写入的RocksDB数据，也是dataset。

你可能要问，那么Kafak里的数据，怎么会是bunny-redis集群的WAL呢？你的Kafka落盘时序可能不同呀。

是的，从这个角度看，BunnyRedis的WAL（即Kafka Log dataset）和其他数据库系统是有很大区别的。

即BunnyRedis的WAL，从时序的角度看，只有内存上的保证（但是，它是多个Kafka机器上的内存保证，所以，我们不用担心一台机器crash，而丢失WAL），而不是磁盘上的保证。

Log，它具备其他dataset不具备的一个特点，就是：Append Only。

因此，对于Log这样的数据，我们可以进行编号，而且顺次递增，而且中间无间隔。

这样，即使落盘的Log可能由于时序的不同，导致在磁盘文件里并不连续，我们可以通过这个Log编号，发现哪些是前面的数据，哪些是后面的数据，哪些是中间丢失的数据，从而避免数据的混乱。

这也是Log的一个妙趣，好好利用它!

