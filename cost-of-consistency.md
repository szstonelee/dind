# 分布式下一致性Consistency的代价cost

## 前言

分布式是个好东西，有了分布式这个东西，我们希望达到：

1. HA, High Availability，高可用性，即如果有一个机器死了，没事，集群里还有另外一个兄弟（而且是双胞胎兄弟，因为有一样的数据）机器顶上

2. Scale-Out，横向扩展，即如果集群一台机器能服务1万个并发客户Client，那么万一来了两万个客户怎么办？简单，再增加一台机器即可

但上面美好的分布式系统的理想，在一致性Consistency的要求下，是有代价cost的，甚至有巨大约束Constraint。

## 什么叫分布式下的一致性

### 数据变化在分布式下的麻烦

打个比方，我在上海开始工作，只有一张银行卡，作为工资卡，上班前，里面有余额100人民币大元，本月工作结束，老板发给我400元月工资（老板好慷慨啊，发的一个月工资是我以前积蓄的四倍！）并存到这个这张卡上，然后老板进一步地照顾我，让我无限期的休假（开除了...）。

为了庆祝此事，我决定到广州去旅游一天，并住了一夜二百五的五星级高级酒店。我认为这没有什么，花掉一个二百五，我还剩下一个二百五。

但我总不希望，我在上海看到银行卡里有五百元，到了广州，却只能看到银行卡里只显示之前的积蓄一百元。即上海增加的数字，只能在上海的银行系统里看得到，到了广州，就看不到了。

这会导致我的信誉破产，至少是人格不稳定吧。

所以，我们希望：分布式系统里，各个机器的数字都一样。一处变化，全局生效。

上面问题很好解决，在上海和广州之间架设一个通信线路，将上海增加的工资数字，也发送给广州的银行系统。

### 数据变化并发时的麻烦

但如果两个地方有并发，就会有麻烦。

比如：我在上海给自己改个名字，叫Tony，然后在广州也给自己改个名字，叫Money。两边同时一起提交申请（假设我克隆了一个自己或者恶作剧，用上自己的孪生兄弟）。

如果上海人名办公室，先将我的名字设置为Tony，然后将这个名字发给广州，广州的也是如此操作，那么我在上海，最后名字会被叫做Money，而在广州，就变成了Tony。未来我拿着一个新的身份证，总有一个地方让我无法住店。

解决方案也有：就是要设置一个中心裁决点，比如北京最高人名办公室。上海和广州都向北京汇报我的改名请求，让北京来裁决。

如果北京先收到上海的请求，然后才收到广州的请求，那么我最终的名字就是Money。反之，就是Tony。

然后北京把这个最终决定，通过一次或两次信函（看两个请求到达北京时间间隔决定，如果同天到达，那么一封信函即可），告诉上海和广州，那么，我的名字的冲突问题就得到彻底地解决。

所以，结论：**对于并发，在分布式系统里，需要一个单点的中心裁决者，leader，来解决冲突**。

其本质在于：由于几方通信的随机性（因为TCP/IP通信是Async的，但TCP在一个连接上是有序ordered的），我们无法在整个系统里，依靠每个机器自己看到的顺序性，来决定整个系统的逻辑的唯一正确性（顺序性）。我们只能用一个单点(leader or master)，根据leader看到的顺序性，来作为整个分布式集群的顺序的一致性。

### 脑裂Brain Split的麻烦

上面还不是分布式仅有的麻烦。我们还有新的麻烦，叫脑裂Brain Split。

还是上面这个改名的案例，如果改名的最终判定权，属于北京最高人名办公室的张主任，但他可能偶有不适，比如修个产假。那么北京张主任不能工作的这段时间，我的改名就无法及时完成，这对于工作效率是不可接受的。

解决方法也有：就是如果北京的张主任病了，那么马上将裁决权转到上海的马主任。而且我们假设两位主任不太可能发生同时休产假，这样就保证了整个系统的有效性Availablility和健壮性HA。

怎么知道北京的张主任不能工作了（因为有可能是北京和上海以及广州的通信因为意外中断了）？在分布式系统里，是由上海和广州等待一个Timeout。如果发现Timeout里没有北京的任何动静（平时在Timeout时间还没有到，要求经常互相打个招呼，说声“我没死”，即发送HeartBeat消息），就认为北京已经网络隔离Network partition。然后由上海和广州一起决定，让上海作为新的裁决者leader。

但问题又来了，北京的张主任恢复很快，在我的最终结果还没有出来，他就回到办公室（或者根本就没有病，只是北京的通信恢复了），而且，他不知道裁决权已经转到上海马主任（记住：前面描述过，由于北京的通信中断，现在刚刚恢复，张主任不可能知道刚发生的上海和广州的leader选举）。这时，就出现了某一个时刻，存在两位裁决者leaders。由于上海和广州传递我的申请的时间是完全随机的（由城市之间的通信速度决定，而且城市间的通信可能受外界影响，时快时慢），那么北京张主任和上海马主任的决定完全可能是完全相反的。

你可能时说：这个好解决。我们认定北京的张主任官大，他的决定高于上海的马主任。所有城市，如果没有北京张主任的决定，那么以上海马主任为准。否则，北京张主任的决定，将覆盖上海马主任的决定。

但我是个坏小孩，我不只一次在上海和广州并发提交改名申请，我会连续提交两次（或任意多次），在两个地方并发多次。而且，我还在这中间，故意破坏北京的网络通信几次，让脑裂Brain Split发生。

这时，广州就会陷入麻烦，它可能会收到多个裁决，有北京张主任的，有上海马主任的，由于是两个地方发过来的，时间上完全随机，所以，广州不知道这是脑裂由北京做出的覆盖决定，还是上海替补的最终决定（因为北京张主任有可能确实牺牲了，而且现在到北京的线路还不通）。

结论：**脑裂时，由于有两个裁决者leader，它们的决定可能是冲突的，导致数据不一致。即使我们给leader做个权值rank（即大家默认哪个官更大），也不能解决一致性问题**。

### 如何解决脑裂Brain Split下的一致性问题

如何解决脑裂下的数据一致性问题，方法有，见下面几个约束：

1. leader必须是大多数人选出来的。即某一时刻，要么不存在大多数认可leader，要么只存在一位大多数认可的leader。即使发生脑裂，绝对不可能发生大多数认可的两位leader同时存在（但可以有两位leader同时存在，只是一个是大多数认可的真领袖leader，一个是少数派自以为是的假皇帝leader）

2. 每个裁决公函，都有leader发出，然后需要大家签字

3. 大家签字的原则是：如果这个裁决公函的leader是我心目中的leader，我就签字，否则我就拒签（认为是伪leader）

4. 只有大多数签字认可的裁决公函，才能生效。否则，就是废纸一张

这四个约束，就保证了分布式的一致性。其实质在于：**通过大多数认可的leader，来保证从时间上，只有一位有效的leader。只有所有的裁决请求，经过所有人的签字认可（或拒绝），才能验证这个裁决决定，是由真正的leader发出的。从而，在时间和空间上，保证了顺序的唯一性**。

当然，实际实现中，比这个还要复杂，站在某个leader角度，首先必须给请求唯一编号id而且顺序递增（由leader的单线程决定），另外，每个请求还必须有印记，标记是哪个leader产生的（在Raft里，这个印记，就是term，而且term也必须单调递增，这个通过选举来保证），即任何一个请求，如果从id+term角度看，是唯一的（而且是单调递增的）。同时，选举中，选出的leader要保证不丢有效的信息（committed log）。这都是附加的，我们不做更详细的讨论。

## 分布式下一致性的代价cost分析

通过上面的案例，大家可以发现，如果要完成一致性，我们陷入下面的约束

1. 有单点限制。我们不能通过多个机器，或者多个进程，甚至一个进程下的多个线程（假设没有同步发生的话，线程同步也是非常大的代价，而且id单调递增必须单线程保证）来自由横向扩展scale-out性能。即，我们集群下的多机器，我们每个机器下的多核，在这里，没用！这是个scale-out的瓶颈bottleneck。

2. 需要集群里达成共识。既需要leader是大多数的选择这个共识，也需要每个裁决请求由大伙批准的共识。这些共识，都是负担，因为有大量的网络通信和信息同步。

而以上，是为了达到分布式下一致性必须付出的代价，逃都逃不掉。

所以，分布式下的一致性，限制了我们的集群能力，因为有单点，同时有巨大的网络通信成本。

## 如何解决这个问题

业内有两个现成的方案解决这个问题，一个是Kafka的模式，一个是纯Raft方案。

如果对比起来，我个人的观点，对于Database而言，纯Raft的解决方案并不好。

etcd是纯Raft，但不是面向Database，而是meta-data datastore。所以etcd没有问题。但如果你将纯Raft用于真正的Database开发，或者分布式存储引擎开发，请慎之。

我的观点：

**这个单点和通信共识的限制，在分布式一致性约束下，是无法逃掉的（Kafka也逃不掉），你只能做一件事，就是让这个瓶颈尽量影响小，也就是说，让它尽量做最少的事情**。

而Kafka用这个思想做到了，而纯Raft却没有。

后面，我会给出详细的分析，请耐心等待后面的分析（而且不是一两页能说明清楚的，你需要通读所有的文章）。





