# 分布式思考：最好不要用CAP去分析问题

## 前言

在很多分布式问题分析中，有人喜欢先介绍CAP或用CAP这个词开篇。作为一个广泛使用的大词（Buzzword），CAP并不是一个很适合分析分布式的工具或词汇，它存在很多自身的问题，下面为你做详细的剖析。

## CAP的本义

CAP，是下面三个单词Word（或短语Phrase）的首字母的组合，如下：

* Consistency，中文意思：一致性
* Availability，中文意思：可用性
* Partition Tolerance，中文意思：分割忍耐性

具体的含义是：

在分布式系统里，你的设计，只能保证从上面三个单词里选取两个，但不可能三个全得。即：

你的分布式系统，要么：

1. CA，但不允许有P。即你的分布式系统，可以即保证一致性Consistency，同时保证可用性Availability，但不能出现系统分割Partition

2. CP，如果你的分布式系统出现了系统分割Partition，你可以保证你的数据的一致性Consistency，但是，此时，你的系统，必须牺牲可用性Availability

3. AP，如果你的分布式系统出现了系统分割Partition，你可以保证你的系统继续可用Availability，但是，此时，你的系统，不能保证数据一致性Consistency

CAP是一种trade off思想，即你不能好处全得，你获得什么，也就意味你失去什么。比如：

* CA，失去P；
* CP，失去A；
* AP，失去C。

下面我们看看，如何攻击上面的这个CAP定义。

## 故意关闭或crash集群部分机器

假设我有一个集群系统，有N台机器，我故意Crash几乎所有的机器，只留下最后一台。

我让最后一台机器继续保持对外服务，同时数据一致（因为数据只有一份，所以一致性Consistency非常容易保证）。

然后，我宣称：CAP破产。因为，我制造了Partition，其中一个Partition只剩一台机器，并对外继续提供Consistency和Availability，然后另外一个Partition是所有崩溃的机器，而且，我忍耐了它。

于是，为了让CAP不破产，CAP需要第一修正案

```
Partition，不能是简单地crash系统。

它必须是基于网络，由网咯产生分割，

而且每个Partition必须至少有1台机器活着。即我们不考虑crash掉的死机器
```

## 故意等待一个足够长的时间

根据上面的修正，我再做一个处理，我让数据包一直retry（比如：TCP重发或重连重发），直到网络恢复（比如：人工检修，一周后恢复网络正常通信），然后集群的数据得到同步，而等待网络数据包的时间（当然，客户也得跟着等），我不认为系统是不有效的（Available），只不过系统是慢了点而已（即使慢一周也可以）。

上面这个做法，又破坏了CAP的金身。

于是，我们还需要再对CAP做一个修正案

```
不可以长时间等待一个数据包，我们必须设置Timeout机制，或者，我们必须设定一些系统监控指标（metric），

如果某个数据包时间过长（或者重试次数超过一个阀值），或者某个metric异常，

我们就认为这个分布式系统是不可用的，即 Not Available
```

## Write Follow Reads

好，我再次搭建一个分布式系统，为了大家易懂，我用两台MySQL作为集群的节点，然后，客户端只写入Key/Value，其中Key是UUID，可以任意选择某个MySQL进行读写。

然后，这两台机器后台通过binlog进行数据同步。

如果发生网络分割，假设这两台机器之间不能通信了，请问，这个系统是否可以继续服务（Availabiliity）？

答案是：可以，你可以继续读写，只是后台binlog同步暂时失效。

那么，数据一致性有问题吗？

我们看数据一致向的几个模型。请参考：[Consistency Models](http://jepsen.io/consistency)里面的一致性模型图。

其中，有一个一致性模型叫：Write Follow Reads，其定义请参考：

[Write Follow Reads的定义](http://jepsen.io/consistency/models/writes-follow-reads)，我抄录其中定义如下：

```
Writes follow reads, also known as session causality, 
ensures that if a process reads a value v, which came from a write w1, 
and later performs write w2, then w2 must be visible after w1. 

Once you’ve read something, you can’t change that read’s past.
```

我们针对上面的系统，来看看Network Partition发生了，我们让集群继续工作，这个Consistency是否失效？

即能否出现下面的不一致情况

```
如果有一个客户（Session），读到一个value，这个value是来自w1，然后这个客户写入w2（可以是针对同一个UUID key）

另外一个客户（Session），可以先读到w2，然后再读到w1
```

显然，上面这个不一致，对于上面的这个MySQL分布式系统是不可能出现的。也就是说，在Network Parititoin时，上面这个分布式系统，继续提供了服务（保持了Availability），同时也保证了一致性Write Follow Reads。

这是不是打脸CAP？

于是，我们还得对CAP追加一个修正案：

```
CAP中的C，Consistenccy，仅限于一致性模型中的Linerizability，即线性一致性
```

## etcd在Netwoork Partition时，继续提供Linerizability服务

我们知道有一个分布式系统etcd，它基于Raft实现，可以对读提供Linerizability。

即使etcd发生了Network Partition，只要其中一个Partition有足够的node（Majority），那么etcd可以继续进行读写，并且提供Linerizability服务。

详细可参考我的另外一个文章：[什么是线性一致性Linearizability](https://zhuanlan.zhihu.com/p/410217203)

于是，我们又得对CAP再做一个修正案

```
CAP里的Availability，不是针对整个系统而言（用一个可以工作的Partition代表整个系统），

而是要求所有的Partition都必须Available and Consistent
```

## 审视经过多重修正案修正的CAP

上面几个修正，我们发现，CAP需要好几个约束

1. 不能针对crash机器，Partition里必须有活的机器

2. 必须有Timeout约束

3. 不能针对所有的一种性模型，而只能是最高等级线性一致性Linerizability

4. 即使针对Linerizability，Availability不是针对整个系统，而必须是所有的Partition

请问，这样的修正后的CAP，还有实际意义吗？

即CAP是针对一个极其狭小的范围的理论论证，从工程角度，没有意义。

## 我们该如何用CAP的思想

其实，CAP的核心思想是Trade Off。

即我们设计分布式系统时（或任何系统时），我们不可能全得，我们要为达到某项目标，在某些方面做些牺牲。除了在Consistency，Availability，Partition这三个维度分析问题，我们也应该考虑其他维度，比如：

1. Cost。我们需要多加几倍的机器，才能保证一台机器crash，整个系统还继续运作

2. Performance。我们是否降低了性能，比如，为了一致性，我们是否需要考虑单点瓶颈和共识通信

3. Complexity。我们是否复杂了整个系统设计，比如：cluster membership的管理

所以，我的建议是：

```
用核心思想，Trade Off，讨论你的分布式系统。

讨论Trade Off时，老老实实分析任何一个得失。

而不要简单用一个大词CAP去解释问题！
```
