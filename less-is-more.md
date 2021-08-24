# 分布式思考：少就是多，多就是少

## Kafka最开始带给我的疑惑

我刚开始读Kafka文档时，发现它有一个做法非常奇怪，就是leader必须让所有的数据都在replica里确认，leader才确认这个数据，然后给客户端committed response。

对比Raft的操作，它是等到大多数member replica的回应，就确认committed这个数据。

数学像我这样烂的人都知道，从[Latency](https://zhuanlan.zhihu.com/p/399883427)角度，Kafka这样做不对呀？

比如：同是一个五集群的Kafka，和同是一个五集群的Raft，进行对比，那么：

Kafka committed latency = max(r1, r2, r3, r4, r5)

Raft committed latency = max(the fastest three in r1, r2, r3, r4, r5)

后来我才发现Kafka的精妙之处。

## Kafka的艺术

### 集群有效性最小的保证

如果我们希望整个集群，支持最多两个节点的故障，集群仍得以正常运行，请问：Kafka和Raft，组成集群，最少需要几台机器？

对于Kafka，答案是三。

对于Raft，答案是五。

### 网络和磁盘的最小消耗

而且，我在之前一篇文章[分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015)写到过，这些数据，最终还是需要在所有的replica里都要过一遍。

因此，假设我们客户端向集群写入1G数据，那么Kafka和Raft消耗在网络和磁盘上的最小数据有多少？

对于Kafka，是3G for network, 3G for disk

对于Raft，是5G for network, 5G for disk

### Latency的艺术

假设我们Kafka和Raft都是五节点集群。

我们知道，正常情况下，每个节点的网络通信都差不多，对于Round Trip而言，大概是200us这个级别。

那么 max(r1, r2, r3, r4, r5) = max(the fastest three in r1, r2, r3, r4, r5)

当发生网络异常时，怎么办？

Kafka做法很简单，开除慢的节点，不要一个老鼠屎，坏了一锅汤。

当开除节点，将产生整个集群membership的变化。这时，就能看出Kafka的妙趣。

membership产生变化，有两种可能：

#### 一、如果member不是leader

Kafka只要leader到Zookeeper那里登记一下，然后就不用理会这件事。最后，即使只剩下leader一台机器，整个集群也运行如故。

Raft，必须保证整个集群的数目维持在quorum，即五台机器，最多只能开除两台。

试想，你是一个team leader，你是愿意去一个团队，可以任意开除下面的员工，但整个team完成KPI任务不受任何影响，还是愿意去一个team，你必须看几个关键员工的脸色，生怕开错了人导致整个team crash（leader当然也会被揪责或因此被开除）？

#### 二、如果member是leader

如果影响的member是leader，差别更大。

对于Raft，必须发起一个选举，选举的过程相当复杂，但它是一个民主选举，即人人参与，而且要找到合适的皇帝leader，万一两个都合适的候选人争皇帝（类似雍正王朝的四王爷和八王爷），还需要有一个冲突机制解决（通过随机休眠来解决，就好比雍正王朝里的争就是不争，不争就是争）

对于Kafka，则是完全粗暴的独裁方式。在整个队伍中，还有一个慈禧太后（controller），如果皇帝死了，她立即指定下一任接班人（比如：让宣统溥仪接替光绪）。

你会问，如果慈禧不在呢？很简单，再换一个慈禧（由其他非leader node到Zookeeper抢先注册，先到先得）。

你会再问，如果存在两个慈禧（controller），甚至两个皇帝（leader），怎么办？

不会发生，因为Kafka还依赖Zookeeper，这个慈禧和皇帝，必须由Zookeeper确定，而在那里，是唯一的。即Zookeeper有点像后面的贵族集团（八旗制度），必须得到贵族集团的认可。即系统最下面还是有底线的民主存在。

但是：

Kafka不用像Raft那样，选举时，搞人人投票，搞冲突机制，它只是用独裁的手段（独裁带来效率的提升），同时简单地一个登记确认即可（法律和民主的底层最终保障）。

## 总结

1. 正常网络情况下，Kafka的All Latency和Raft的Majority Latency是相当的

2. 不正常网络情况下，Kafka开除慢的node，仍然保证足够的（和正常一样的）速度

3. 保证一定集群有效数目，Kafka用的node数目只是Raft的近一半，因此带来网络和磁盘消耗也降低近一半

4. Kafka用controller独裁代替Raft的民主选举，使改朝换代的阵痛（cost）最小

5. 但Trade Off是，Kafka还必须依赖第三方强一致系统（Zookeeper，或最近准备替换的Raft），来实现上面这些特性

即佛教里的禅语：**少就是多，多就是少**

这也是[BunnyRedis](https://zhuanlan.zhihu.com/p/392646113)采用Kafka模式的一个很重要的因素。

