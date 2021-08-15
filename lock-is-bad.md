# 分布式思考：锁是个麻烦，限制它

## 单机单进程下的锁

我们先看最基本的操作系统下，一个进程用到的锁。

在知乎上，我有几个回答，可以作为预备知识，

[锁的本质](https://www.zhihu.com/question/461351872/answer/1930585878)

[多线程竞争锁时，阻塞等待锁释放是如何实现的？](https://www.zhihu.com/question/461385626/answer/1911994921)

[线程的切换还能让线程快吗？](https://www.zhihu.com/question/453408768/answer/1847958654)

[为什么用互斥锁而不是屏蔽exception/interrupt防止多线程同时对同一公共资源进行读写?](https://www.zhihu.com/question/452878311/answer/1819897531)

简而言之，锁是让多线程并发时，保证进程数据结果的一致性。同时，在保证一致性的基础上，提高性能Throughput。

在数据库系统里，这个操作系统锁，是作为底层锁的基本工具，称之为Latch。

## 单机下的数据库上层锁

数据库系统有自己一套更高层的，基于上面Latch的锁机制，针对的对象是数据库内部的对象，比如：行、页、表和索引、数据库。按特性又区分为读写锁、共享和独占锁、意向锁。也具有操作系统锁不具备的一些高级特性，比如：死锁的监测和恢复。

我不是一个这方面有经验的程序员，我的理解是：

和操作系统锁一样，单机数据库锁，也是为了保证数据库数据的一致性，在此基础上，提高并发和吞吐Throughput。

即锁的本质没有变。

1. 由于并发冲突，来保护共享资源的一致性

2. 在保证一致性基础上，并发提高Throughput

更重要的是，Relational数据库还需要对数据做出ACID保证。而ACID里面很多特性，是基于锁的。比如：经典的2PL（一种悲观锁），可以支持Isolation到Serializable。

## 分布式下，锁是个大麻烦

和基于单进程的操作系统锁的单机数据库的锁不同，到了分布式系统里，我们会发现锁不好用。

我们来看分布式锁对于锁的两个目的（本质）可能带来的损害

### 锁是为了保证数据（共享资源）的一致性

如果是单进程的操作系统锁，很多东西你不用考虑。而到了分布式，必须用到网络，也就必须用到通信。

类似LPC（local procedurer call）和RPC（remote procedure call）的差别。

LPC下，你的返回结果一定是预先定义的返回值（deterministic），比如1+1，一定返回结果2，而且Latecny基本是固定的。即它是稳定的、可靠的。

RPC下，你的返回结果还可能包括网络故障、Timeout（因为受限于另外一个机器上的另外一个进程），Latency没有什么保证，即它不是稳定的，也不是可靠的。

因此，分布式锁可能被一个死进程永远霸占，如果你加入Lease属性（即锁有timeout），那么又可能出现两个进程都拥有分布式锁的情况（比如：一个进程GC了）。所以，为了保护资源，我们还必须加入Token防护。一个Redis的例子可以作为参考：[怎样实现redis分布式锁？](https://www.zhihu.com/question/300767410/answer/1931519430)

### 锁在保证数据一致性的前提下，希望提高并发

我们用锁，还有一个目的，是为了提高并发。

大家可以参考我的一个文章：[单线程就比多线程性能差吗？不一定](https://zhuanlan.zhihu.com/p/397039359)。

当我们用操作系统锁，如果只是简单的CAS，我们只付出20ns的代价，如果是mutex并有线程切换，也不过几百us的代价。这对于磁盘这个动不动就以ms计价的介质而言，这个成本是划算的。所以，能提高并发。

但到了网络上，首先是有Round Trip的通信损耗(一般是200us)，而且，还需要依赖其他进程处理锁的速度。比如：另外一个进程拿到锁，处理任务，然后通过网络释放锁，如果是秒级操作，那么，我们的进程也必须等待这个秒级Latency，而且由于网络特性和多机环境，这个还不可靠。

所以，在分布式下，用分布式锁，不仅很难提高并发性，反而损害并发性。

### 分布式锁还需要自己给自己提供分布式保护

分布式下，我们还要考虑分布式锁本身的健壮，即万一提供锁的机器死了怎么办？

提供一套一致性很强的分布式锁是非常困难的，Redis的RedLock还犯了错误。etcd等提供了，但性能测试显示并不好。

因为：[分布式下一致性是有代价的](cost-of-consistency.md)

## 结论

在分布式系统下，对于单机，可以继续用它自己的锁系统，但一旦这个锁要扩展到跨进程、跨网络、参与集群一致性，请谨慎使用。

我的建议：对于分布式锁，不到必须用，尽量不用。

那么对于分布式处理，我们不用锁去保证数据的一致性，那我们是否就不能保证整个集群的数据的可靠性，同时维持一定性能（甚至不能scale-out）？

不是的，请看下一页。