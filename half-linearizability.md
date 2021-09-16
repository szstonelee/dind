# 用Half Linearizability去破解线性一致性(Linearizability)的性能瓶颈

## 准备知识：什么是Linearizability

请参考此文：[什么是线性一致性Linearizability](https://zhuanlan.zhihu.com/p/410217203)

## Linearizability的瓶颈

### Write的瓶颈

首先我们必须保证集群里的多个机器node的数据变化后，是强一致的，即Write的强一致

请参考下文:

[分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015)

我们知道：为了保证数据的强一致，我们受限两个瓶颈：

1. 单点leader无法scale-out

2. 多机的共识

### Read的瓶颈

上面讨论了Write的强一致，那么Read时，是否也存在这个问题

参考[什么是线性一致性Linearizability](https://zhuanlan.zhihu.com/p/410217203)

我们得知，读的时候，为了保证Linearizability，我们同样受这两个瓶颈的限制

1. 单点leader无法scale-out

2. 多机的共识

所以，不管是Write，还是Read，为了Linearizability，我们步步都受这两点的瓶颈限制

## 两个符合Linearizability的极端案例

那么，有没有可能，不受这两个瓶颈限制。

答案是：有，至少两个，但都是特例。

### 特例一：如果集群是一台机器

如果我们假设集群只有一台机器，请问，它是不是Linearizability？

答案：是的，因为它符合[什么是线性一致性Linearizability](https://zhuanlan.zhihu.com/p/410217203)文中的约束

1. Atomic

2. Seems like one copy of data （Actually it has only one copy of data）

请问，这个系统有瓶颈吗？

对于单点瓶颈，可以说有，也可以说没有，因为只有一台机器

对于共识，没有（一台机器唯我独尊，不存在商量、讨论、协商、一致的问题）

### 特例二：如果集群是多台机器，但数据是只读imumutable的

假设我们集群不限制于一台机器node，而是多台，所以我们有多个一样的数据copy在每个node上，然后，我们假定这个数据从此不变了，

请问：它是不是Linearizability？

答案：是的，因为它同样符合上面两个约束

1. Atomic

2. Seems like one copy of data （Actually they are same copies of one immutable data）

请问，这个系统有瓶颈吗？

对于单机瓶颈，没有，因为我们可以到集群里的任意一台机器去读（注意：没有写，因为只读，我们限制了写）

对于共识，也没有，因为数据不变，每个节点node只用原始数据，不需要和其他节点达成一致，也不需要额外的通信

## Half Linearizability的解决之道

Half Linearizability就是参考了上面Linearizability集群的极端两个例子，所做出的方案

### 方案一：sticky to one node and isolated from other clients

类似上面的集群只有一台机器，我们的集群系统虽然有多台系统，但是，客户端只允许绑定集群一台机器。

那么假设：集群是写强一致的，那么对于这个客户，读是否是强一致？

答案：当然是的。

对于这个客户端，它看到的数据，就是满足Linearizability，因为它眼里，只有一台机器（当然，就只有一份数据）

但是相关问题随之而来：

#### 如果这个客户端，和另外一个客户端，有逻辑关系，而另外一个客户端连入了另外一个node，怎么办？

答案：不允许这样

即我们必须将逻辑和相关状态，受限于单个客户端的单个机器的连接

如果一个客户端的逻辑，在某些应用场景下，可以完全的隔离Isolated出来(即不和其他客户端发生关系），那么，我们就可以用这个方法实现Linearizability。

即在这个客户端眼里，它只有一台机器node给它提供服务；如果它需要和其他客户端有关联，是绝对不可以的。即这个客户端，被：

**sticky to one node and isolated from other clients**

注意：可以多个客户端连入并绑定同一机器node，然后让这些连入同一node的客户端之间有关联，这同样可以保证Linearizability。

#### 如果连接的node死了怎么办？

我们当然要切换node（否则就无法实现HA了），但是要注意：切换node，会导致切换时，数据不一致，从而违反Linearizability。

解决方法：

对于切换node，我们要像对待新连接一样去对待，即原来的client，必须抛弃所有的之前的状态记录，对于未完成的Transaction，必须rollback。然后，一切初始化重新来过。

**switch to other node like a new baby**

### 方案二：snapshot

类似上面的极端例子，集群数据只有read-only data时，集群是Linearizability。

我们假设现在集群数据是可变的，但是，提供snapshot，而且，对于某一版本的snapshot，集群所有node的数据都是完全一样的。

这时，如果客户端Client去读数据时，带入snapshot版本参数，那么，这时，还是符合Linearizability。而且，Client可以任意切换node。

但存在下面几个问题：

#### 版本滞后怎么办？

比如：Client到Node A，读版本version=5的数据，然后再到Node B去读另外一个数据，但此时，Node B还没有更新到版本5（比如：只更新到版本4）。

方法是：等 (Wait until the right version of snapshot)

等到版本5的数据到来，因为根据最终一致性的约束，Node B最终会出现这个版本5的数据。

#### 如果很长时间等不到某版本数据怎么办？

比如：上面这个例子中，Node B被Network Partition了，无法更新最新的版本5数据。

方法是：Timeout and Retry

设置一个Timeout时间，到时仍不能更新，那么切换node，到集群里其他node去找这个新版本数据，直到找到（或永远找不到，比如：版本号错误）

## Half Linearizability总结

Half Linearizability并不是完整的线性一致性Linearizability，或者更准确地说，是一种受限（某种约束下）的线性一致性Linearizability。但它可以带来性能的提升（因为不受那两个瓶颈约束）。

其要点在两个：

1. 要么sticky to one node in cluster

2. 要么bound to one snapshot for every node in cluster

附：Half Linearizability和Half Master/Master，

都是我创建的词汇，里面都有Half这个怪词，用于某些特殊的设计模式或方案。想了解Half Master/Master，参考：[从Raft角度看Half Master/Master(两层解耦)](https://zhuanlan.zhihu.com/p/407603154)


