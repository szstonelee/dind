# 从Raft角度看Half Master/Master(两层解耦)

## 什么是Half Master/Master

Half Master/Master是我创建的词，其是为解决下面的问题：

**为了获得Master/Master的好处，同时，能保证一致性**

### 什么是Master/Master的好处

请参考这个文章: [BunnyRedis要解决Redis的高可用HA问题](https://zhuanlan.zhihu.com/p/392628491)

简而言之，

**我们可以不通过Shard的方式，就横向扩展scale-out写Write，并且是强一致**

因为，Master是可写可读的，但实际生产环境中，一个机器大部分都是读，所以，对于一个Shard(或无Shard)只用一个Master机器实际是浪费了写Write的能力。

但麻烦在于，写Write对于集群一致性有重大挑战。

### 如何保证Master/Master的强一致

纯粹的Master/Master实际是不能保证强一致的，请参考下文

[BunnyRedis要解决Redis的一致性Consistency问题](https://zhuanlan.zhihu.com/p/392637293)

里面有详细的分析，为什么Master/Master不能做到强一致。

我们必须用一个强一致的**非Master/Master**先去达到强一致并作为底层，然后才可以再在其上，去构建必须依赖这个底层的上层的Master/Master，然后才能保证这个Master/Master组件的强一致。

这就是**Half Master/Master**，即两层结构，下面一层是强一致的非Master/Master，然后基于它，再实现强一致的Master/Master上层。

再抄录文中一段描述文字如下

```
即你的整个集群cluster系统里，必须至少有一个组件（一般是底层组件）不采用master/master模式，
从而保证一致性。

但是其他依赖这个底层集群组件的上层集群组件，可以用master/master模式，
从而可以达到整个系统的强一致，同时获得master/master的好处。
```

## 从Raft角度实现一个Half Master/Master

### 预备知识

可参考这个文章，[分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015)

强一致的两个根本

1. 单点leader

2. 多机的共识

但强一致也必然带来代价

1. 单点不好scale-out

2. 多机共识所带来的通信（和其他）的cost

### Raft如何演化成Half Master/Master

我们知道Raft是一个强一致系统。

Raft为实现强一致，也需要单点leader和多机共识，可以参考下图

```
                            Raft
 ***********************            ***********************
 *     leader node     *            *     other nodes     *
 *                     *            *                     *
 *  1. Replicated Log  *     ...    *  1. Replicated Log  *
 *  2. State Machine   *            *  2. State Machine   *
 ***********************            ***********************
```

假设我们如果只实现Log的强一致，或者State Machine就是Log Data本身，那么上图可以简化成下图

```
                              Raft
 *************************            *************************
 *      leader node      *            *      other nodes      *
 *                       *            *                       *
 *  only Replicated Log  *    ...     *  only Replicated Log  *
 *************************            *************************
```

然后，再根据下面一个文章，

[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)

我们知道，通过Log，我们就能再次构建数据库，也就是那个State Machine，见下图

```
                    Half Master/Master

*********************                *********************
*  Stete Machine 1  *                *  Stete Machine n  *
*    (Database)     *       ...      *  (same Database)  *
*********************                *********************

                       解耦Decouple

                ****************************
                *   only LOG Raft cluster  *
                ****************************
```

上面，就实现了Half Master/Master方式，即State Machine cluster是Master/Master模式，但必须依赖**非Master/Master**的Log cluster。

如果LOG是强一致，整个系统也是强一致，i.e., 我们获得了一个强一致的Database。

这是第一层解耦，即

**State Machine被移出成为独立一层，只让Log受分布式强一致的代价的约束**

## 更进一步，构建Log强一致集群时，不用Raft模式，而用Kafka模式

```
          可选两种模式: 构建受强一致约束的Log集群

****************                *****************
* Raft For Log *       VS       * Kafka For Log *
****************                *****************
```

这两种模式的对比，可以参考下面两个文章

1. [分布式思考：少就是多，多就是少](https://zhuanlan.zhihu.com/p/402990609)

2. [Kafka模式对比纯Raft模式简表](https://zhuanlan.zhihu.com/p/405228466)

从上面这两个文章，我们知道，实现强一致Log的底层系统，用Kafka模式更好。

这是第二层解耦（从Kafka模式看），即

**meta data，i.e., membership of cluster，被移出成为独立一层**

## Half Master/Master的好处

1. 强一致的瓶颈，用了最小的负载，因为只涉及Log，而且是Kafka模式

2. 上层的Master/Master，也是强一致，这样就可以横向扩展(scale-out)关键的写Write

3. 正因为扩展了Write，我们就可能不做或少做Shard

请参考：[分布式思考：我们需要分片Shard（含分库分表）吗？](https://zhuanlan.zhihu.com/p/403604353)

这样，就带来了第四点好处

4. 如果我们可以避免Shard，我们也就避免Shard的麻烦和Cost，包括Join和Distributed Transaction

## 两层解耦的意义

### 第一层解耦的意义

将State Machine解耦，从而让State Machine不受分布式一致性的约束，降低了cost。

而State Machine，相比Log，一般更复杂，做的事更多，更难优化。

### 第二层解耦的意义

虽然Log作为data不能不受分布式一致性的约束，但我们再次针对Log做第二层解耦，让meta data分离出去。

这个meta data，就是membership of cluster，比如：谁是leader，谁是controller，哪个可以加入cluster，哪个可以离开cluster，维持HA所需要的最小的quorum。

这将使Log as data保持分布式下强一致的代价cost和约束进一步降低。

从而，让第一层解耦出来的下层Log强一致基础系统，进一步优化，提高效率。

以上，就是两层解耦的真正意义，也是Half Master/Master的精髓。

附：预更多了解Half Master/Master模式，请参考下文

[从Raft角度看Half Master/Master(两层解耦)](https://zhuanlan.zhihu.com/p/407603154)