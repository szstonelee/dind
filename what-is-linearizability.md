# 什么是线性一致性Linearizability

## 核心要点

只有两个

1. Atomic：原子性

2. Seems like one copy of data：看上去像一份数据

### 原子性：Atomic

#### All or Nothing

计算机文档中难过，有很多地方，我们都见过原子性Atomic这个词，它的基本语义就是：All or Nothing。即对于某个动作Action（或一连串动作组合成一个更大的超级动作），它的效果是：要么都做了，要么什么都没有做，不会有中间状态。

举个例子：银行转账

一个账户A有人民币100元，另外一个账户B有人民币100元，然后进行一个动作：将A的钱转20元到B账户上。

实际操作中，先从账户A减掉20元，这样A变成了80元(Write 80 to A)，然后，再向B账户加上这20元，这样B账户变成120元(Write 120 to B)。最后给这个动作的请求客户返回结果是成功转账消息。

但如果中间发生意外，比如：账户A在建行，账户B在农行，正好转账时两个银行的网络发生了故障，那么这个动作必须返给请求客户失败的信息，而且两个账户的钱必须和转账前是一致的，即A和B还是100元。

在上面的转账过程中，在某一个时刻，实际账户A和B，可能是下面3个状态之一

1. A=100, B=100
2. A=80, B=100
3. A=80, B=120

这里面，状态1是起始状态，即Init State，对应All or Nothing里的Nothing；状态3是正常结束状态Final State，对应All or Nothing里的All。那个状态2，是中间状态，Partial State，这个中间状态，在时空上一定是存在的，但对于原子性Atomic要求，我们必须保证：

**不管发生任何异常，最终结果，不允许出现中间状态Partial State**

即对于这个例子，银行转账系统不允许出现A是80元，B是100元的中间状态，那样就是银行吞没了客户的20元，是犯罪。

#### Linearizability中Atomic对比事务Transaction中Atomic的不同点

我们知道，在数据库中还有一个事务Transaction，它的要求也有Atomic。

见下图：

一致性Model: ![Alt Text](https://jepsen.io/consistency/models/map.svg)

这个图中，左半边是关于Transaction的，右半部（顶端就是Linearizability or Linearizable）是关于非Transaction的，它们都用到了Atomic这个词，那么它们有什么不同？

在讨论Linearizability时，或者更广义地说，上图中左半部（Transaction）和右半部（非Transaction），对于Atomic的中间状态Partial State，其差异在于：

1. 左边部分，Transaction，核心关注点就是这个Partial State，即承认这个Partial State，并重点研究这个Partial State所带来的相互影响，以及各种Isolation下，对于这个影响所做的相关保证（Guarantee）

2. 右边部分，非Transactiion，是忽略这个Partial State，即认为这个Partial State没有任何影响，分析问题时，根本就不考虑这个中间状态。然后在这个基本态度下，看各个一致性模型所做出的各种保证（Guarantee）

所以，分析Linearizability时，对于上面转账这个案例，我们认为只需要考虑两个状态：

1. 起始状态，A=100, B=100
2. 结束状态，A=80, B=120

这也是为什么我们在讨论K/V系统时，都去谈右边的一致性（比如：[BunnyRedis的一致性](https://zhuanlan.zhihu.com/p/392653517)），因为Key/Value，基本操作都是针对一个key，根本就没有Partial State。我们当然不需要考虑左边的Transactionn，因为那个是针对的中间状态及其中间状态互相影响的场景。

#### Atomic结论

对于Linearizability的Atomic，我们只考虑动作前的状态，和动作结束后的状态，我们认为不存在中间状态。

如果是Key/Value系统，它天然的是没有中间状态；

但如果对于转账这种涉及多个key的动作，请你把多个key看成一个大的key，即：

大Key = A账户 和 B账户 的联合

它只有两个状态，账前是：（A=100, B=100），转账后是：（A=80, B=120）

### 看上去像一份数据：Seems like one copy of data

所谓Seems like oone copy of data，是因为在一个集群clusster里，有多个机器nodes组成，每个node都握有一份最终一样（Eventually Consistency）的数据。即数据data有多个拷贝copy，它分布在一个集群中的不同的node上。

但是，我们在用到这个Linearizability集群时，我们要看上去，好像只有一份数据。即我们可能访问多个node的多个copy，但我们看上去，好像只访问了一个node的一份copy。

这又是一个和上面那个一致模型图中左边Transaction极其不同的地方。

在Transaction里，我们大部分情况下，都不关心一个数据有多个copy。而对于多个node共同拥有多个copy数据，则考虑得更少。

即在Transaction语境下，我们考虑更多的是：一个数据对象，被几个并发的过程，共同拥有的应用场景，而且，主要着力于单机单点(single node)上；即使我们讨论Distributed Transaction时，还是很少考虑一个数据在多个机器node上的多个copy，而是基于单个node的Transsaction基础上的某种共识（Consensus）

而Linearizability恰恰是要考虑多个机器node上多个拷贝copy的一致性问题（或者说：保证Guarantee）

所以，这个Linearizability的保证就是：

**多个node上的多份copy，实际使用时，和一个node上一份数据，在一致性方面，没有什么不同**

## 案例分析

记住上面那两个要点，特别是Seems like one copy of data这个保证，我们用具体的案例来分析什么样的系统，符合Linearizability

### 案例一

```
        ****************            ****************
        * node A (R=0) *            * node B (R=0) *
        ****************            ****************

 time:           t1                   t2                  t3
Client:    Write(R=1) to A    OK Response from A    Read(R) from B
```

说明：

node A 和 node B 组成cluster，它们有一个初值R=0。客户Client，在时刻t1向node A发出一个写命令，设置R=1，然后t2时刻，收到A返回的成功（OK）消息，接着t3时刻，向B发出读R命令，请问，如果cluster是Linearizability，那么读到的R是是什么值?

分析：

将上面两个node，虚拟成一个copy of data（即R），如下图


```
               *****************************************            
               * one virtual node for one copy of data *            
               *****************************************           

 time:           t1                   t2               t3
Client:       Write(R=1)         OK Response         Read(R)
```

很显然，答案应该是1。

### 案例二

```
  ****************            ****************
  * node A (R=0) *            * node B (R=0) *
  ****************            ****************

 time:           t1                    t3
Client:    Write(R=1) to A        Read(R) from B
```

问题变化：

和上图不一样的地方，t1发出Write命令到A，但没有等到Response，就在后面的t3时刻发出Read R到B。如果cluster是Linearizability，那么读到的R是什么？

答案：

可能是0，也可能是1

分析：

见下图

```
               *****************************************            
               * one virtual node for one copy of data *            
               *****************************************           

 time:              t1                         t3
Client:          Write(R=1)                  Read(R)
```

首先，我们要认为t1和t3时刻发出的通信包，是属于两个TCP连接，而不是一个。

虽然t3时刻比t1晚，但因为TCP/IP通信的原因，我们不能保证：t1时刻发出的Write TCP包，先于t3时刻发出的Read TCP包到达virtual node。

如果t3时刻发出的Read TCP包先于t1时刻发出的Write TCP包到达这单个的virtual noode，那么读出的值就是0。然后，迟到的Write TCP包才将R改为1.

如果t3时刻发出的Read TCP包晚于t1时刻发出的Write TCP包到达这个virtual node，那么读出的数据就是1。

因此，答案就是：读到的R的值，可能是0，也可能是1，取决于Write和Read命令TCP包到达集群并生效的先后。

### 案例三

```
  ****************            ****************              **************** 
  * node A (R=0) *            * node B (R=0) *              * node C (R=0) *
  ****************            ****************              ****************

 time:           t1                 t3                 t4                     t5     
Client:    Write(R=1) to A     Read(R) from B   response R=1 from B     Read(R) from C
```

问题变化：

和案例三相比，我们的cluster变成了三个节点（初始值R=0）。t1时刻还是Write(R=1)到A，但没有收到Write的Response，就在t3时刻发出Read(R)到B，然后在t4时刻，B返回了Read结果是R=1。在仍没有收到Write的返回结果时，在t5发出Read(R)命令到C，

请问，如果clusster是Linearizability，那么我们从C读到的值是什么？

* 答案选项甲：肯定是0
* 答案选项已：肯定是1
* 答案选项丙：有可能是0，也有可能是1

正确答案是：已，即肯定是1

分析:

```
               *****************************************            
               * one virtual node for one copy of data *            
               *****************************************           


 time:           t1            t3           t4           t5     
Client:      Write(R=1)      Read(R)        R=1        Read(R)
```

如果clusster是Linearizability，它就好像只有一份数据。虽然t1时刻的Write，我们不知道是否成功以及何时成功，但t4时刻的Read Response，已经告诉我们，旧值(R=0)不存在了，所以，t5时刻的Read，一定读到新值，因为看上去，只有一份数据，不管这个Read请求，是发给A，or B，or C的。



