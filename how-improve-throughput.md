# 分布式下如何提高性能

## 前言

我们在上一页，[分布式下锁是个大麻烦](lock-is-bad.md)，有描述分布式下，我们希望通过分布式锁来实现一致性要求下的并发性能，可能出现的相关麻烦。而且，文中不建议直接使用分布式锁，那么是不是意味：

**谈锁色变**。

不是的。我们还是要从单机、单进程、锁、性能，这些基本东西谈些，看看哪些东西我们可以借鉴，并且能用到分布式下。

所以，我们还是要先复习单机下的并发或如何通过锁等工具提高性能，当然，前提仍然是要保证一致性。

## 单机下的经验

### 单线程不需要锁

大家知道，如何你写的程序是单线程，你无需用到锁。

比如：C标准库中，对于内存分配malloc，可以有不同的编译选项，一个是用锁的，给多线程进程使用，一个是不用锁的，如果你的程序是单线程体系。而且，不用锁的内存标准分配库，要更快些。

我们也知道，Redis是主逻辑、主数据结构的单线程体系，而且运行得飞快。

在我自己的实验中，单线程无锁，可能带给编译器、CPU很好的优化，让单线程的速度和近万的线程的性能相当，可以参考：[单线程就比多线程性能差吗？不一定](https://zhuanlan.zhihu.com/p/397039359)

### 只读、Immutable，不需要锁，还可以多线程并发

如果我们有一个只读的数据结构，即数据不会发生改变（Immutable），那么你就可以放心地用多线程访问，即没有一致性的担忧，同时，又能用到CPU多核的威力。

### 合适的数据结构：可以修改同时多线程无锁并发

比如：我有一个Log，它是C String的数据，即每增加一个字符串，这个字符串的标志是：

1. 字符串里面不会出现0
2. 最后一个字符一定是0

同时，因为它是Log，所以，它又符合Log的特性

1. 只在尾部修改
2. 已添加的，不会改

即Append-Only。

这时，你可以放心大胆地用多线程，而且不用锁，只要任何时间点，只有一个写线程，就能获得高并发的好处。

### 如果用锁，但锁冲突比较低，我们也可以高并发

比如[Java的CurrentHashMap](https://dzone.com/articles/how-concurrenthashmap-works-internally-in-java)，它用shard的方式（或者segment），让一个对多线程不友好的HashMap（因为存在ReHash），让多个线程可以访问这个数据结构，并且降低冲突的可能，从而带来高并发。

很多内存分配库，比如Jemaloc, TCMalloc，都用到了类似的概念。

### COW，copy on write

COW，or copy on write，也是一个经常用到的，提高并发的手段。

当我们的数据需要修改时，它可能不支持多线程下的一致性，那么，我们将这个数据copy一份，然后对其中一份进行修改，那么没有修改的旧数据，仍旧可以提供给其他线程进行只读，这样，就能提高并发。

一个典型案例是：gRPC用到的Protocol Buffers(protobuf)。

## 分布式下如何借鉴

分布式下我们一样可以借鉴上面的思想

### 单机的性能

我们应该可以利用单机的性能，甚至对于单机，可以利用多线程并发。因为单机在集群里，就好像单线程针对单进程一样。

特别是：在集群系统里，经常为了保证一致性，我们有单点的约束。参考：[分布式下一致性的代价](cost-of-consistency.md)。

但是，这里面有一个麻烦，就是单机可能crash，导致数据可能会丢失。

我的想法是：

1. 对于计算，尽量用到单机的性能
2. 对于存储，需要用集群一起参与
3. 万一单机crash，计算重算或者客户端更换连接节点node，重试retry

### 只读

只读是个好东西，如果数据或数据的部分，是只读的，我们可以无障碍地使用多机，这是非常好的sccale-out。

### 适合修改的数据结构

如果单机系统一样，如果有这样的数据结构，我们可以大胆地进行多机分布，而且不用担心一致性。

可以参考之前的一个文章，[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)

### 通过分片shard降低冲突

如果我们能将数据分片shard，而且shard之间的关联很少，那么，我们也可以利用到多机的同时并发。

这对于key-value系统特别合适，比如：Redis Cluster，就是让key通过hash-slot分布到不同机器上。

只要大部分的数据操作模式，不是跨越两个shard的Join，不要求两个shard一起保证一个atomic操作，我们就无负担地并发。

### COW

Copy on Write，是个分布式里经常用到的模式。

我们可以将数据copy到多个机器上，提供读服务。这样，对于读，几乎是无上限的scale-out。

但写时，我们只针对一个机器进行修改，修改完成后，再分发到各个机器上。

## 总结

其实，分布式下并发提高性能并没有特别的地方，都是源自单机单进程多线程的一些思想，只是有更多的约束（比如：网络通信的约束，跨进程就不能共享地址空间的约束），我们需要克服这些困难，用更复杂的代码去做类似的东西。

上面这些手段，不一定全部都适合每个系统，或者没有必要全部用于一个分布式系统（因为复杂度太高），你只要关键的部位，有一处真正用到，就可能带来整个系统的性能极其大的提升。