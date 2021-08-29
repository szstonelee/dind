# 从Raft角度看Half Master/Master

## 什么是Half Master/Master

Half Master/Master是我创建的词，其是为解决下面的问题：

**为了获得Master/Master的好处，同时，能保证一致性**

### 什么是Master/Master的好处

请参考这个文章: [BunnyRedis要解决Redis的高可用HA问题](https://zhuanlan.zhihu.com/p/392628491)

简而言之，

**我们可以不通过Shard的方式，就横向扩展scale-out写Write，并且是强一致**

因为，Master是可写可读的，但实际生产环境中，一个机器大部分都是读，所以，对于一个Shard(或无Shard)只用一个Master机器实际是浪费了写Write的能力。

但麻烦在于，写Write对于集群一致性有重大挑战

### 如何保证Master/Master的强一致

Master/Master实际是不能保证强一致的，请参考下文

[BunnyRedis要解决Redis的一致性Consistency问题](https://zhuanlan.zhihu.com/p/392637293)

里面有详细的分析，为什么Master/Master不能做到强一致。

我们必须用一个强一致的**非Master/Master**先去达到强一致，然后才可以再在其上，去构建必须依赖这个底层的上层的Master/Master，然后才能保证这个Master/Master的强一致

这就是**Half Master/Master**，即两层结构，下面一层是强一致的非Master/Master，然后基于它，再实现强一致的Master/Master上层（单独的Master/Master是无法实现强一致的）

再抄录文中一段描述文字如下

```
即你的整个集群cluster系统里，必须至少有一个组件（一般是底层组件）不采用master/master模式，
从而保证一致性。

但是其他依赖这个底层集群组件的上层集群组件，可以用master/master模式，
从而可以达到整个系统的强一致，同时获得master/master的好处。
```

## 如何实现一个底层的强一致系统

### 强一致的两个根本

1. 单点leader

2. 多机的共识

可参考这个文章，[分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015)

### 但强一致也必然带来代价

1. 单点不好scale-out

2. 多机共识所带来的通信（和其他）的cost

同样是参考上文，[分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015)

## 从Raft角度看Half Master/Master

Raft为实现强一致，也需要单点leader和多机共识，可以参考下图

```
 ***********************            ***********************
 *     leader node     *            *     other nodes     *
 *                     *            *                     *
 *  1. Replicated Log  *     ...    *  1. Replicated Log  *
 *  2. State Machine   *            *  2. State Machine   *
 ***********************            ***********************
```

假设我们如果只实现Log的强一致，或者State Machine就是Log Data本身，那么可以简化成下面的结构

```
 *************************            *************************
 *      leader node      *            *      other nodes      *
 *                       *            *                       *
 *  only Replicated Log  *     ...    *  only Replicated Log  *
 *************************            *************************
```

然后，再根据下面一个文章，

[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)

我们知道，通过Log，我们就能再次构建数据，也就是那个State Machine，见下图

```
*********************                *********************
*  Stete Machine 1  *                *  Stete Machine n  *
*    (Database)     *       ...      *    (Database)     *
*********************                *********************


                    ******************
                    *   LOG cluster  *
                    ******************
```

上面，就实现了Half Master/Master方式，而如果LOG是强一致，整个系统也是强一致

这个分离的目的，是为了让最下面那层LOG Cluster，所做的事情最小，

因为：

1. LOG cluster受强一致约束，即单点leader和共识，

2. 但State Machine Cluster就没有这些约束，只要它们依赖强一致的Log

## 但是，构建Log强一致集群时，我们可以不用Raft，而用Kafka

```
****************                *****************
* Raft For Log *       VS       * Kafka For Log *
****************                *****************
```

这两种模式的对比，可以参考下面两个文章

1. [分布式思考：少就是多，多就是少](https://zhuanlan.zhihu.com/p/402990609)

2. [Kafka模式对比纯Raft模式简表]()

从上面这两个文章，我们知道，实现强一致Log的底层系统，用Kafka模式更好

## Half Master/Master的好处

1. 强一致的瓶颈，用了最小的负载，因为只涉及Log，而且是Kafka模式

2. 上层的Master/Master，也是强一致，这样就横向扩展scale-out关键的写Write

3. 正因为扩展了Write，我们就可以不做或少做Shard

请参考：[分布式思考：我们需要分片Shard（含分库分表）吗？](https://zhuanlan.zhihu.com/p/403604353)

这样，就带来了第四点好处

4. 我们避免了Shard的麻烦和Cost，包括Join和Distributed Transaction，从而提升了整个集群的性能
