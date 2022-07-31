# 从分布式Distributed、日志Log、一致性Consistency分析AWS Aurora for MySQL

## 预备知识

建议你先大致读一下，下面的参考资料Reference。

你可以不用全部理解参考资料里的所有文字，或者认为它的每一句话都是对的（我个人认为Amazon的论文里都有错误），但是，对AWS Aurora有一个轮廓，有一些基本概念，特别是，带了很多的疑惑和问题再读本文，会有助于理解本文。

同时，读完本文，再回头去阅读参考资料，会有助于进一步理解，特别是AWS的原始论文。而且，本文也不是全面解读论文里的AWS，特别是segment shard、recover、membership change这些子命题，都被特别忽略了，因为我想聚焦在本文标题中所讨论的命题，因为我认为这对于理解AWS Aurora特别关键。

当然，Tony作为本文作者，也可能犯错误，所以，请用批判的精神阅读本文。

## 一、Aurora的架构特点

Aurora基于MySQL(InnoDB)的改造，是一个分布式OLTP的关系型数据库（RDB），它的架构，有如下特点:

### 1-1、计算和存储分离

传统MySQL，存储是单独的磁盘作为硬件支持，和整个数据库位于同一机器（machine or node）上。而Aurora，存储是用单独的机器实现，作为存储层独立存在，然后，通过网络和数据库其他层（即计算层）交互，以实现ACID的支持。

计算层负责：客户端连接的管理、SQL语句的解析和执行计划（PLAN）的形成、并发事务Transaction的管理和隔离保证（Isolation）、大内存作为DB Cache（MySQL里又叫Page Pool）的管理。最后注意：计算机器（Computing node）也有本地磁盘，而且很重要（用于存储undo log），但整个的数据，并不存储于Computing node的本地磁盘上。

存储层负责：和传统MySQL类比，它就像一个磁盘，用于存储几乎所有的数据（除了undo log）。但是，因为使用了机器（Storage node），Storage node有自己的CPU、内存（相比Computing node不大）、以及本地的SSD磁盘（相比Computing node非常大）。因此，Storage node不仅仅是一个磁盘硬件，也就是说，它可以运行程序，接受来自Computing node发来的请求并进行处理，并且在自己的内存里，形成一些数据结构（HashMap、Queue等），和本地SSD磁盘一起，通过网络应答（Client/Server）的模式，对计算层提供存储服务。同时，Storage node之间，也通信（Gossip）。

### 1-2、集中共享式的分布式存储

所谓存储是分布式，指的是整个数据库数据data set，不只存在于一个Storage node上，而是有6个copy，分布在最少6台Storage nodes上。

所谓集中共享式，是指从任何一个Computing node眼里看存储层，都是统一无区别的。数据只有整体一致的六个copy，虚拟成一个统一的整体（好似一个文件一样）。

但Aurora是整个数据data set有6个copy，而且每个copy，是做了shard，即segement方式分割数据。每个sengment不允许超过10G，超过即形成新的segment，segment之间是数据连续分布的。比如：整个dataset是20G，那么是2个segment，又必须有6个copy，则总共有12个segment。这12个segment，至少可以用6个Storage node去承担，也可以最多12个Storage node去成承担。并且，segment可以后台移动，即从一个Storage node，作为一个整体单位（segment为单位），移动到另外一个Storage node上。

但这和类似Spanner、AWS DynomaDB的系统，shard上有本质爱的差别。

Spanner是先考虑shard（比如：最少2个key就可以分布了），将数据分散到多个node上，然后执行数据处理时，预先用Distributed Transaction做了准备。

而Aurora是先不考虑Shard，即10G下，不需要多个segment shard。同时，做数据处理时，即使数据分布在多个segmeent上，也不需要Distribute Transaction考虑。即多个segment虚拟连接成好像一个独立的数据库存储文件。

后面分析时，为了简化理解，我们去除Aurora所使用的shard，即忽略segment的作用。我们简化成只有六个Storage nodes，然后有6个copy，每个在一个Storage node上。这对于整个Aurora系统分析，没有任何影响，但你必须理解，Aurora内部，是用segment做了shard的，而且存储集群不只限于6个Storage node。

同时，Aurora假定，对于存储层，系统正常工作的前提条件是：不允许超过2个（即大于2）Storage nodes同时发生故障。如果发生少于2的故障，应该在一定的时间内，用其他替代Storage node，来替代这些故障node。只要这个替代时间足够短（小于某个时限），就认为整个存储层是永远都不会故障的（有效概率达到一定数目的几个9，就认为是永久安全，类似UUID的思想）。

### 1-3、Single Master / Multi Slave模式

Aurora只有一个Computing node（注意：随后的补注2），作为master对外服务（在Amazon文档里，因为政治正确，它被叫做Primary node或者write node）。然后有最多15个replica（本文用传统的方式，称之为slave），一起组成一个计算集群，对外服务。即你的客户程序（Client App或者App），只能对master发出有读有写的SQL事务，但对于slave，只能用只读事务。

这是一个很传统的Master/Slave模式，它很好地保证了数据的一致性Consistency。

补注：

1. Aurora后期也提供类似Postgres SQL的RDB支持，但较晚，本文只分析MySQL模式。

2. Aurora后期也提供多Master的支持，但提供较晚，而且是基于前期单Master模式上改造的，本文只分析早期的单Master模式。

### 1-4、落盘处理和相关log

master的写盘，只有redo log通信传输到存储层（注意：没有undo log到存储层），也没有page直接写盘（不管是网络上的存储层，还是本地磁盘）。

master的redo log，无需本地存盘。但undo log需要本地存盘。

master和slave的通信，即有redo log，也有undo log，但没有page直接传送。

slave需要对undo log进行本地存盘，但对于redo log，无需存盘。

master和slave如果请求的page不在DB cache里，它们都是直接到存储层，通过网络请求获得对应的page。

log我们还必须详细解释，请接着往下看。

## 二、Log的妙趣

我们将数据库简化成一个字符串值，比如：初始值是，I am Tony。然后我们的事务Transaction修改数据库，加三个词word，先改写为: I am Tony coding for world。然后再增加两个word，改写为：I am Tony coding for world in China。这可以是一个事务分两次执行，也可以是两个事务并发执行（假定后一个修改的并发Transaction的修改效果，在时间上是后发生）。

我们将每个修改动作作为Log record记录下来，这样上例中就存在两个Log record，分别是：

* log record a: 尾部追加 coding for world
* log record b: 尾部追加 in China

我们发现，数据库有初值的话，每个修改动作被记录成log record，则数据库最后的值，是每个Action按顺序作用于前一个值的最后结果，因此，我们通过预先存在的一个初值和一个Log，来完整复现最后的数据库的终值，即

```
 ------------                                       -------------
|  数据库初值  |  + log record a + log record b  =   |  数据库终值  |
 ------------                                       -------------
```
上面的 + 号，相当于apply，就是根据log record的内容，然后做相应的动作（本范例的动作是：尾部追加几个词，append words）。

上面啊的log record的总和，就是redo log。

我们发现有下面几个现象：

* log record必须有顺序，而且必须唯一确定，比如：我们如果将log record b放在log record a前面，数据库的终值会变成I am Tony in China coding for world，语句算通顺，但结果不一致。再比如：如果我们log record b出现了两次，则终值会变成：I am Tony coding for world in China in China，语句不通顺了。

* 如果缺少了中间某个log record，我们不能形成正确的终值。比如：如果缺少了log record a，那么终值是：I am Tony in China。语句是通的，但结果不对。

数据库处理遵循上面的原则，即将修改动作log record记录到redo log并存盘。这样，即使数据库终值没有及时存盘（掉电或数据库进程被杀），我们一样可以通过磁盘上的数据库初值，再加上redo log，获得正确的数据库终值。

数据库只所以用redo存盘，是因为整个数据库存盘代价太大（cost is big）。试想一下这个字符串初值长达100G大小。而redo log record不大，但历史（比如：一天）合起来的log record以及对应的DB会很大。所以，我们要redo不断及时存盘（而且是连续的追加方式，无需修改前面的存盘内容），而DB就可以适当延时存盘，可以部分存盘（part dirty pages），可以同一个page被重复多次存盘（in-place update），也可以合并存盘（check point）。

因此：

1. 我们给每个redo log record分配一个LSN（Log sequence number），并保证：LSN是唯一的，且LSN是递增的。这样顺序和唯一性就得以保证

2. 我们让redo log及时落盘，即至少要保证Transaction commit时，必须redo先落盘。这样，发生任何意外，commit的东西，绝对不会丢。

上面对于数据库有一个麻烦的地方叫：后悔。后悔的意思：Transactionn允许abort，数据库的值必须roll back到前面一个值，比如上例中，我们有两个Transaction，一个做动作a，一个做动作b，但允许做动作a的Transaction后悔，不commit，而是roll back。

如果根据redo，我们发现，实际上，我们只要将redo log record里面的动作反着做，我们就可以得到修改的前值。为了计算简便，我们做一个对应的undo record log，如下
```
时刻:               time 1            time 2                              time 3

DB cache内存里：    I am Tony          I am Tony coding for world          I am Tony coding for world in China

redo log record:                      append: coding for world            append: in China 
undo log record:                      delete: last three words            delete: last two words

DB in disk:        I am Tony          I am Tony                           I am Tony
```

如果已经执行动作a的Transaction，在还没有执行动作b前就后悔roll back（即time 3以前），我们只要在内存I am Tony coding for world这个值，对应的undo log record: delete: last three words，去执行这个undo动作，我们就可以获得I am Tony这个合适的值。如果已经执行动作a的Transaction，在并发的另外一个事务已经执行了动作b发生以后（即time 3以后），做roll back，我们必须连续读两个undo log record，然后计算得到，这是从尾部倒退五个单词word，然后开始删除三个单词，即从I am Tony coding for world in China，变回I am Tony in China。

所以，当前内存的DB，必须有一个指针，指向对应的undo record log。然后，历史的undo record log，必须形成一个链表，构成历史遍历，这时，任何一个Transaction需要roll back时，都可以从指针开始，遍历这个undo record链表，然后回复到合适的值。如果需要（比如：上例中的Update），undo record log里，应该记录是哪个Transaction生成的（不记录Transaction的包括，Insert Type的undo record log，因为它不需要被其他Transaction感知，即不参与MVCC）。

我们不用对整个数据库，做成整体一个对象的redo log和undo log，如果这样，redo log和undo log都太大（想想并发很多Transactionn，运行很长时间，那个undo log遍历链表会变得很大）。我们可以对整个数据库，按照tuple和page为单位，作为redo和undo的目标对象，然后形成相应的链表。这样，每个tuple都有自己对应的undo链表用于roll back，同时，每个tuple，都有对应的连续的redo log records（无需形成链表，只要前后时间保证），用于recover时形成最终值。

undo log还带来一个MVCC的好处。

试想一下还有另外一个read only transaction，它的Isolation被设置为READ COMMIT级别，即它不允许dirty read（不允许返回uncommited value）。

当read only transaction读到内存的I am Tony coding for world时（time 2时刻），如果Transaction a还没有commit，那么，它必须去读对应的undo record log（通过tuple的指针），然后简单计算获得前值是I am Tony，然后才能保证返回值是一个committed value。

如果read only transactionn读到的时刻是time 3，内存的值已经变成I am Tony coding for world in China，同时Transaction a和b都没有commit，那么它必须通过undo链表，去计算得到I am Tony这个committed value。但如果Transaction a已经commit，而Transaction b还没有commit，那么它必须顺着这个undo 链表，同时必须知道Transaction a和Transaction b的commit状态（即在数据库内存里，Transaction a和b是否在active transaction list里），然后得到此时的committed value = I am Tony coding for world。

所以，下面的结论对我们后面的分析很重要：

1. undo log支持MVCC，因此可以支持各种Isolation级别的读写。

2. 为了支持各种Isolation，除了需要undo log，还必须知道当时的Transaction运行状态。对于read only tansaction而言，它只需要知道当时的active write transaction列表。InnoDB里，只有写的Transaction才有Transaction ID（在第一个有写的SQL语句里分配），才会在transaction list里。read only tansactio是无Transaction ID的，不会在transaction list里。

## 三、Aurora的写Write : quorum write

### 3-1、quorum write的意义

quorum是一个洋文，它的意思是选举里，必须达到法定人数，才算有效。对于Aurora，它有6个copy，因此系统设定quorum = 4。即4个Storage nodes写成功，才算是写成功的前提条件。

前面的描述知道，计算层和存储层时通过网络通信的，而且都是独立的机器node，所以，站在发起write的Computing node眼里，是收到Storage node的response，这个Storage node才算成功，而且收集了至少4个成功返回response的Storage node，Computing node才认为这是写成功的必要条件（注意：还不算写成功）。

我们又知道，Aurora里，Computing node只发送redo log给Storage node，这个redo log在Computing node内存里，是按LSN顺序不断尾部追加redo log record的，那么Aurora如何处理：1. 异步的通信；2. 分布的6个Storage nodes，这两个麻烦。

Aurora的处理是：

1. 并不强求上一个redo log record写成功，才发下一个redo log record。即可以乱序

2. 如果某个redo log record暂时没有写成功，就重试重发，因为假设整个存储层是永久有效的（见上文描述），所以，这个write也是最终quorum write成功的

补注：quorum字面上看（以及很多其他系统的实现），并不一定强制要求超过半数。Aurora对于写的quorum要求是4，是有特别原因的，详细不作解释，看论文。

### 3-2、quorum write的好处

因为乱序，所以，可以并发。即站在Computiing node眼里，上一个redo log record暂时还没有成功，并不影响处理下一个recoord发送给存储层。

网络通信里，最影响效率的是：必须等待对方回应，才能处理下一个请求（这样网络带宽是不满的）。比如：我们知道App for RDB，一般一个数据库连接上的吞吐Throughput都不够多，不足以达到DB的最高效能，一个很大的原因就是，App必须处理了一个SQL事务，才能发送下一个SQL事务，因为上一个SQL事务的结果，可能是下一个SQL事务的某个条件，它们有相关性，所以，单连接不能并发。要实现RDB的全部效能，我们必须用多个数据库连接去完成，而且这些连接上的并发的SQL事务，没有相关性。

所以，Aurora的quorum write的乱序发送，对于吞吐Throughput有很大好处，因为少了前后的成功的约束。

### 3-3、quorum write的麻烦和如何解决

#### 对于computing node (master) 的麻烦

从前面的分析，我们得知，redo log record必须保证次序完整，才能保证数据库的最终一致性。

站在Computing node角度，稍有麻烦，因为发送是乱序的而且可以重发，所以，某个时刻，不能保证先发先到。因此，Computing node必须记录发送队列queue，以及接受状态（某个recoord收到几个response，注意：实际实现为了效率，是按批packet发送的，所以只需记录packet的状态，我们这里用record作为单位，是为了简化理解）。但是，经过一段时间，computing node肯定可以收集到所有发出的redo log record的足够（quorum）回应，只要Computing node一直活着。活着的Computinng node，能保证截止到某一个时刻（即某个LSN），前面的redo log record是连续写成功的。

因此，Computing node可以保证redo log record的写成功，是可以做到：一直连续的，而且不断推进的，只是要异步等待一点时间。因此，Computing node可以根据写存储成功，不断推进计算层的事务处理。

比如：

传统的MySQL，在收到App的Commit请求时，必须先生成对应的redo log record（先在内存里，即redo buffer），然后必须保证写盘成功（flush to disk，同时也保证之前的redo log reccord也写盘成功），然后才能接着处理后续的相关内容内容，包括解锁、改变Transaction状态（从transaction list里删除此Transaction ID）和返回commit成功信息给客户App。

但这个例子，如果到了Aurora这里，生成的commit redo log record收到了Storage node的四个response，虽然此record被标识某种成功了（success of collecting quorum response，即master不用再针对这个record，向其他Storage node发送网络包了），但仍不算write success，必须保证前面的所有的乱序的redo log record也标识成功（success of collecting quorum response），此commit redo record才算写成功（success of quorum write），然后才能接着处理解锁、改transaction list以及回应App成功这些动作。

但注意：其他非commit类型的redo log record的动作，Aurora的comupting node可以继续做，不受任何影响（即不受quorum write success的约束）。那些修改某个tuple（对应的page）里的相关内容，可以继续生成redo log record（并作为一个Mini transaction写入到redo buffer）里，并执行这些动作的相关效果，包括且不限于下面这些动作类型：从存储层读某page，修改DB cache里的某个page，分裂和合并（split or merge）某些page以保证整个B树的完整一致，等等。

所以，quorum write的异步性和分布式，并不影响Computing node里的并发的事务的执行效率，除非到了commit这个特别阶段。而且，到了commit阶段，也只影响当前提交commit请求的transaction（和对应的某个数据库连接），其他并发的Transaction并不受影响（除非受数据库的内部锁的影响，因为某个锁可能是正在等待commit的transaction持有的）。

注意：read only transaction的commit不受quorum write success影响，因为read only commit没有生成redo log record（但read only transaction仍受数据库内部锁的约束，不过因为MVCC，锁的影响极小）。

这里，Aurora引入一个重要的概念，**VCL**，其定义如下

>VCL: 就是Coomputing node（首先获得和唯一判定的：只能是master）看到的最后一个完成quorum write成功的redo log record（即最大的LSN），且之前的redo log record都已经保证quorum write成功。这个只要master简单记录redo log的发送和接受状态，然后经过简单计算可得。

如果再转义一下，VCL就是针对存储层，落盘成功的最大的LSN且保证之前的LSN全部落盘成功。

#### 对于Storage node的麻烦

Storage node收到master的redo log record，这样，就可以根据本地的存储，以及这个redo，形成未来的对应的值。

但我们上面的分析知道，这个需要保证redo log record是连续，无中间漏洞的。

但quorum write的模式决定了，Storage node收到的redo log record可能是中间有漏洞的，而且master有可能一直都不发来这个弥补漏洞的record。

所以，对于Storage node，弥补这个漏洞，需要两点:

1. 需要识别哪些是漏洞

2. 需要到其他Storage node里，将这些丢失的record补回来

先看如何识别漏洞:

如果LSN是严格增1连续的（continously increesing，注意：只是连续increasing，不保证continous，只有单调加1的连续，才是continously increesing），那么问题很简单，比如，某个Storaage node收到的redo log record的LSN是如下的（假设LSN从1开始起始）：
```
1、5、5、2、3、7、6、9
```
我们简单做一个Sort，然后遍历，马上知道漏洞是4和8。

但是，Aurora不是这样处理的（因为LSN并不是简单连续的，即continuous，它只保证唯一和递增，实际上，LSN是对应record在redo里的字节偏移量，byte offset），所以，Aurora用了另外一个技术，链表，即每个redo log record，都记录了前一个redo log record的LSN。

这样，Storage node可以简单通过prev LSN，就知道是否有漏洞，即漏洞是哪些LSN。

然后，发现漏洞的Storage node，通过gossip通信方式，向其他Storage node询问这些丢失的LSN。之前分析我们知道，因为quorum write模式，保证：总有至少一个其他的Storage node，有丢失的LSN在那里存在，并可以获得以弥补漏洞。

这样，Storage node也就可以保证：redo log record，经过一段时间的异步（含被动从master获得，以及主动gossip从其他Storage node获得），可以完整一致地，在本机上形成一个链表。我们再用这个连续的无丢失漏洞的redo log，进而可以获得一个（或多个）新的存储数据（因为旧的数据已经在Storage node的本地磁盘保存了）。

你可能问：如果master上的transaction roll back怎么办？

答案很简单，当Transaction roll back时，也会形成对应的redo log record，Storage node只要执行这些record里的动作，就能回到roll back所希望达到的数据状态。

所以，Storage node必须缓存redo log record，遍历发现漏洞，然后gosssip其他Storage node补齐这些漏洞。

对于无漏洞的连续的redo record log，Storage node可以放心地一个接着一个地进行apply，获得对应的数据库状态。而且状态可以不只一个，如果条件允许，我们可以用新的状态覆盖旧的状态，或者，删除旧的状态，即回收purge（也可以叫GC，Garbage Collection）。为什么不用一个状态表达呢，为什么说条件允许，为什么要purge or GC？我们后面会涉及这个点。

补注：Aurora的prev LSN是比较复杂的，正如我们前面讲到的，它还有segment shard，同时需要照顾磁盘存储基于block这个单位，所以Aurora里面，是存了三个prev LSN，分别是，基于整个redo log record链表的prev LSN，基于segment的prev LSN，和基于Block的prev LSN。本文为了简单，只笼统地说了一个prev LSN，但这不影响整个概念的阐述。

## 四、master的读read

## 五、slave的同步

## 六、slave的读read


## 七、参考资料Reference

* [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Database](https://web.stanford.edu/class/cs245/readings/aurora.pdf)
* [Amazon Aurora: On Avoiding Distributed Consensus for I/Os, Commits, and Membership Changes](https://pages.cs.wisc.edu/~yxy/cs764-f20/papers/aurora-sigmod-18.pdf)
* [An In-Depth Analysis of REDO Logs in InnoDB](https://www.alibabacloud.com/blog/an-in-depth-analysis-of-redo-logs-in-innodb_598965?spm=a2c65.11461447.0.0.bdf6654adSY57h)
* [An In-Depth Analysis of UNDO Logs in InnoDB](https://www.alibabacloud.com/blog/an-in-depth-analysis-of-undo-logs-in-innodb_598966#:~:text=InnoDB%20uses%20Undo%20Log%20to,database%20can%20resume%20services%20first)
* [Deep Dive: InnoDB Transactions and Write Paths](https://mariadb.org/wp-content/uploads/2018/02/Deep-Dive_-InnoDB-Transactions-and-Write-Paths.pdf)
* [AWS Aurora (by 胡明)](https://zhuanlan.zhihu.com/p/391235701)
* [Aurora读写细节分析 (by 叶提)](https://zhuanlan.zhihu.com/p/508928878)
* [MySQL Engine Features InnoDB Based Physical Replication](https://alibaba-cloud.medium.com/mysql-engine-features-innodb-based-physical-replication-a345990bd266)