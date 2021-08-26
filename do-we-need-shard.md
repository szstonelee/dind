# 分布式思考：我们需要分片Shard（含分库分表）吗？

## Shard是个好东西

Shard是个好东西，对于分布式系统，它将数据分开，存储在不同机器上。

这样，我们就可以利用到集群的多机，很好地进行scale-out。

Shard的管理会增加新的复杂度，但通过一些手段我们可以解决：

1. [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)

2. 类似Consistency Hash的思想，采用固定数目的Slot，比如：[Redis Cluster](https://redis.io/topics/cluster-tutorial)

3. 用更复杂的机制，在Shard的管理上，采用一个meta data store，来管理各个segment的转移、合并、删除等

## Shard有它的麻烦

Shard对于scale out是个非常好的方案，而且是性能如果到达瓶颈时的最终解决方案。

但是，如果我们的某个操作，涉及两个Shard，就是大麻烦，包括：

1. Join

2. Distributed Transaction

对于Join，我们要到两个机器上去拿数据，因此，还需要一个中心点，做聚合Aggregation操作，同时，如果其中一个Shard fail了，怎么办？

对于Distributed Transaction，当前主流做法是2PC -- Two Phase Commit。但是，这是个效率很低的解决方案，而且有单点故障。Google虽然通过Precolater做了一定程度的优化，即将undo（含redo）延后处理，来减少Latency，从而提高Throughput，但是，它仍没有解决单点故障的问题，同时代价还是很大（为了Isolation的支持，还需要通过分布式的硬件时钟，或者集中式的软件时钟来保证顺序）。

## 一般情况下，Shard应该是最后的解决手段

软件学说里有一句名言：**在没有达到瓶颈Bottleneck时，不要随意优化Optimization**

类似地，我认为：**在一个Shard（无Shard）没有达到瓶颈Bottleneck时，不要轻易地分片Shard**

因为分片Shard带来的麻烦，如Join，Distributed Transaction，都是不可避免的（还包括最上面的Shard管理的复杂度）。而如果没有Shard，那这些麻烦都将不用考虑，从而简化我们的分布式系统。

分布式系统并不只是一开始就应该立刻进行分片Shard，或者说，分布式系统并不等于Shard（而应该是性能问题解决不了才Shard、或者很多时候是最后最终Shard）。我们应该简化系统，在系统能解决问题的前提下，用简单的方法（甚至老式的方法）去解决问题，而不是一开始就高、大、上。

## 我们当前一个Shard（无Shard）的瓶颈达到了吗？

比如：我们的一个数据库服务器，我们发现它只能最大支持100K qps，于是，我们就认为，它的瓶颈到了，必须Shard了。

对吗？

我们看一个正常的互联网应用，其访问基本遵从一个规律：大部分访问是Read，小部分访问是Write。

如果90%的访问是Read，10%的访问是Write，那么，我们完全可以用十个机器来做一个集群，每个机器都有同样一份数据，那么就算访问量大到10倍，即1M qps，只要每台机器能支持100K Write qps，我们是不是就不用Shard？

你会说，Write会破坏一致性，不能保证Write时，十个机器的数据都是一致的。

但如果解决了呢？

[BunnyRedis要解决Redis的一致性Consistency问题](https://zhuanlan.zhihu.com/p/392637293)就是尝试解决这个问题的一个实践（Half Master/Master）。即分布式下，不用单一master来保证数据的一致性。

你会再说，这个BunnyRedis不过是解决key/value，这个对于Key/value适用，对于Relational Database不适用。即在我的文章[BunnyRedis的一致性Consistency](https://zhuanlan.zhihu.com/p/392653517)里，我只解决了右半边的问题，没有解决左半边的问题，比如：Read Committed, Repeatable Read, Serializable这些问题。

是的，我正准备解决这些问题。至于如何解决，因为我还没有做出来产品，来证明我的观点，所以，不适合发表（Talk is cheap, show me your code）。

但是，我认为，这是可以用类似的解决方案进行解决的问题。

同时，我前面发表的一些分布式优化，你也可以参考一下，如果你的单Shard（即无Shard）在这些优化下，都能做到不出现瓶颈，那么，你就不必急着分库分表，做Shard。

* [分布式思考：我们需要fsync吗](https://zhuanlan.zhihu.com/p/400099269)

* [分布式思考：我们需要WAL吗](https://zhuanlan.zhihu.com/p/400338569)

* [分布式思考：我们需要磁盘吗](https://zhuanlan.zhihu.com/p/400480015)

* [分布式思考：批处理Batch是个好东西](https://zhuanlan.zhihu.com/p/401190110)

* [分布式思考：Locality是个好东西](https://zhuanlan.zhihu.com/p/401569843)

* [分布式思考：锁是个麻烦](https://zhuanlan.zhihu.com/p/401716856)

* [分布式思考：如何提高性能](https://zhuanlan.zhihu.com/p/402660843)

* [分布式思考：少就是多，多就是少](https://zhuanlan.zhihu.com/p/402990609)

## 我个人关于Shard的几个观点

1. 我们不应该在单Shard（无Shard）还没有到瓶颈时，就急着去做Shard，因为这会带来复杂性，特别是Shard之间有交互

2. 在瓶颈判断时，我们不应该用整体（read + write）的负载（overhead）作为判断依据，而应该特别关注write，因为read几乎是无限扩展的，难点在write

3. 我们不应该用master/slave（因此只有单一master）去作为必须Shard的判断，因为强一致不一定就是master/slave，还有half master/master

4. 我们应该让成为瓶颈的负载尽量最小，而不应该让可能出现瓶颈的组件做过多的事，一个案例就是：Kafka模式和纯Raft的对比（下一文章会有描述）

5. 当Shard数据之间没有交互时，我们可以放心地、大胆地、自由地，去做Shard，这是Shard最佳的应用场景

6. 分布式下的事务 不等于 分布式事务，i.e., 分布式 + 事务 != 分布式事务

7. 分布式 不等于 Shard，i.e., 分布式 != Shard

## 我理想中的一个BunnyRedis集群的极限

```
                                                   ** Your Redis Clients **

                                                             |
                                                             |  Read/Write commands
                                                             |                                                            

**********************************************************                           ********************            
*                    NIC for Redis Clients               *                           *                  *
*                                                        *                           *                  *
*                  bunny-redis 1 (local SSD)             *                           *  bunny-redis n   *
*                                                        *          ......           *                  *
*                       NIC for Kafka                    *                           *                  *
**********************************************************                           ********************

                                                |
                                                |  only Write commands
                                                |

    ****************************************************************
    *                                                              *
    *     NIC-1    ...     NIC-8                                   *
    *       (for bunny-redis)                                      *
    *                                                              *                     ****************
    *                                                              *        commit       *              *
    *                                                     NIC-1    *     -------------   *   broker 2   *
    *                                       (for broker2)  ...     *                     *              *
    *                                                     NIC-3    *                     ****************
    *                                                              *
    *                      Kafka broker 1                          *
    *                                                              *
    ****************************************************************
```

如上图，每个broker的NIC带宽都是100Gb/s（可以粗略认为10GB/s），这样，对于一个Kafka broker 1，可以接收25GB/s的来自bunny-redis的写Write，同时有至少4倍的放大（包括：1个收，给2个bunny-redis的同步，1个给broker 2的同步），正好达到内存带宽的100GB/s的限制。

如果每个Write命令的大小是1K（比如：一个Redis set命令），那么这个集群的Bandwidth，对于写是25M qps，对于读几乎无上限。

只有超过这个极限，我才开始考虑对BunnyRedis做Shard，即多个BunnyRedis集群通过Shard再形成一个更大的集群（可以通过Proxy实现）。

而且上面的架构还是保证了：

1. HA，即无单点故障

2. 强一致

3. dataset可以大于内存的限制，即支持磁盘

注：上面有一个难点我没有想清楚，就是如何让多个NIC能虚拟成一个NIC对OS使用，否则，当前Kafka架构还不能支持多网卡NIC的聚合（Aggregation）。但如果集群系统每个机器只用一个100Gb/s的网卡，可以推测 Max qps for Write = 2M。Max qps for Read = unlimited。即使只用一个网卡，我们也可以达到一个很大的上限，而在这个上限之前，无需做Shard。这会大大降低系统的复杂度。

巴菲特的合伙人，查理·芒格，投资比亚迪的那个人，喜欢反向思维。就是：传统的思路是这样，但我如果不这样，会如何。这个文章，以及之前的几个文章，

* [分布式思考：我们需要fsync吗？](https://zhuanlan.zhihu.com/p/400099269)
* [分布式思考：我们需要WAL吗？](https://zhuanlan.zhihu.com/p/400338569)
* [分布式思考：我们需要磁盘吗？](https://zhuanlan.zhihu.com/p/400480015)，

都是尝试向这位老先生学习的结果。

巴菲特认为芒格比他聪明、更会投资，我认为巴菲特不是谦虚，只是大家都喜欢用最后的而且简单的金钱数字来衡量一个人成就、但忘了一个事实，芒格年轻时有一大家子人要去养，而巴菲特老爹死了，巴菲特都舍不得一口好棺材（注意：并不是批评巴菲特，而是巴菲特的理念，就是要尽可能省出任何一分钱的本金去投资，所以巴菲特能成功的地方，其实大家都做不到，包括他的破车和破房子）。