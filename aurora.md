# 从分布式Distributed、日志Log、一致性Consistency分析AWS Aurora for MySQL

## 预备知识

建议你先大致读一下，下面的参考资料Reference，特别是头两篇Amazon关于Aurora的两篇论文。

你可以不用全部理解参考资料里的所有文字，或者认为它的每一句话都是对的（我个人认为Amazon的论文里有错误），但是，对AWS Aurora有一个轮廓，有一些基本概念，特别是，带了很多的疑惑和问题再读本文，会有助于理解。

同时，读完本文，再回头去阅读参考资料，会有助于进一步理解参考资料，特别是Amazon的原始论文。而且，本文也不是全面解读论文，特别是segment shard、recover、membership change、backup、performance、multi master这些子命题，都被特别忽略了，因为我想聚焦在本文标题中所讨论的命题，我认为这对于理解Aurora特别关键。

当然，Tony作为本文作者，也可能犯错误，所以，请用批判的精神阅读本文。

## 一、Aurora的架构特点

Aurora基于MySQL(InnoDB)的改造，是一个分布式OLTP的关系型数据库（RDB），它的架构，有如下特点:

### 1-1、计算和存储分离

传统MySQL，存储是单独的磁盘作为硬件支持，和整个数据库位于同一机器（machine or node）上。而Aurora，存储是用单独的机器实现，作为存储层独立存在，然后，通过网络和数据库其他层（即计算层）交互，以实现ACID的支持。

计算层负责：计算机器（Computing node)负责，客户端连接的管理、SQL语句的解析和执行计划（PLAN）的形成、并发事务Transaction的管理和隔离保证（Isolation）、本地大内存作为DB Cache（MySQL里又叫page pool或buffer pool）的管理。最后注意：Computing node也有本地磁盘，而且很重要（用于存储undo log），但整个的数据，并不存储于Computing node的本地磁盘上。

存储层负责：存储机器（Storage node），和传统MySQL类比，它就像一个磁盘，用于存储几乎所有的数据（除了undo log）。但是，因为使用了机器，Storage node有自己的CPU、内存（相比Computing node不大）、以及本地的SSD磁盘（相比Computing node非常大）。因此，Storage node不仅仅是一个磁盘硬件，也就是说，它可以运行程序，接受来自Computing node发来的请求并进行处理，并且在自己的内存里，形成一些数据结构（HashMap、Queue等），和本地SSD磁盘一起，通过网络应答（Client/Server）的模式，对计算层提供存储服务。同时，Storage node之间，也通信（Gossip）。

### 1-2、集中共享式的分布式存储

所谓存储是分布式，指的是整个数据库数据data set，不只存在于一个Storage node上，而是有6个copy，分布在最少6台Storage nodes上。

所谓集中共享式，是指从任何一个Computing node眼里看存储层，都是统一无区别的。数据只有整体一致的六个copy，虚拟成一个统一的整体。

但Aurora是整个数据data set有6个copy，而且每个copy，是做了shard，即segement方式分割数据。每个sengment不允许超过10G，data set超过此大小限制即形成新的segment，segment之间是数据连续分布的。比如：整个dataset是20G，那么是2个segment，又必须有6个copy，则总共有12个segment。这12个segment，至少可以用6个Storage node去承担，也可以最多12个Storage node去成承担（也可以6-12中间任何一个数字）。并且，segment可以后台移动，即从一个Storage node，作为一个整体单位（segment为单位），移动到另外一个Storage node上。

但这和类似Google Spanner、AWS DynomaDB的系统，shard上有本质的差别。

Spanner是先考虑shard（比如：最少2个key就可以分布了），将数据分散到多个node上，然后执行数据处理时，预先用Distributed Transaction做了准备。

而Aurora是先不考虑shard，即10G下，不需要多个segment。同时，做数据处理时，即使数据分布在多个segment上，也不需要Distribute Transaction考虑。即多个segment虚拟连接成好像一个独立的数据库存储空间（可以抽象成一个文件或一个tablespace，所有表table对应的clustered inxde和second index都在里面，即所有的B tree），所以，我们仍然可以用B tree对应的页page，以及任何一个page都只有唯一的Number（即Page No.）标识此page，只是Page No.需要根据node、segment进行一个相应的换算即可。

后面分析时，为了简化理解，我们去除Aurora所使用的shard，即忽略segment的作用。我们简化成只有六个Storage nodes，然后有6个copy，每个copy在一个Storage node上。这对于整个Aurora系统分析，没有任何影响，但你必须理解，Aurora内部，是用segment做了shard的，而且存储集群不只限于6个Storage node。

同时（从现在开始，忽略segment shard），Aurora假定，对于存储层，系统正常工作的前提条件是：不允许超过2个（即大于2）Storage nodes同时发生故障。如果发生少于2的故障，应该在一定的时间内，用其他（非此6个）Storage node替代故障的Storage node。只要这个替代时间足够短（小于某个时限），就认为整个存储层是永远都不会故障的（有效率达到一定数目的几个9，就认为是永久安全，类似UUID的思想）。

### 1-3、Single Master / Multi Slave模式

Aurora只有一个Computing node作为master对外服务（注意：随后的补注2，同时，在Amazon文档里，因为政治正确，它被叫做Primary node或者write node）。然后有最多15个replica（本文用传统的方式，称之为slave），一起组成一个计算集群，对外服务。即你的客户程序（Client App或者App），只能对master发出有读有写的SQL事务，但对于slave，只能用只读事务。

这是一个很传统的Master/Slave模式，它很好地保证了数据的一致性Consistency。

补注：

1. Aurora后期也提供类似PostgresSQL的RDB支持，但较晚，本文只分析MySQL模式。

2. Aurora后期也提供多Master（Multi Master）的支持，但提供较晚，而且是基于前期Single Master模式上改造的，本文只分析早期的Single Master模式。

### 1-4、落盘处理和相关log

* master的写盘，只有redo log通信传输到存储层（注意：没有undo log到存储层），也没有master对page的直接落盘（不管是网络上的存储层，还是master的本地磁盘，page落盘在论文里被叫做data落盘）

* master和slave因为没有page的落盘，所以，也就无需double write

* Storage node有相应的page落盘，但Storage node的page落盘，是根据redo计算获得的，Storage node可以不使用double write，因为Storage node允许个别故障。

* master的redo log，无需本地存盘，但undo log需要本地存盘。

* master和slave的通信（只同步write），即有redo log，也有undo log，但没有page直接传送。master和slave之间Storage node，不需要binlog的传输，即master和slave之间是物理同步（redo，redo是针对B tree具体page的物理操作记录），而不是逻辑同步（MySQL binlog里的信息是逻辑操作信息，即SQL语句或针对某个tuple的write）

* slave需要对undo log进行本地存盘，但对于redo log，无需本地存盘。

* master和slave如果请求的page不在DB cache（即本机的内存）里，它们都是直接到存储层，通过网络请求获得此page。

注：本文忽略Aurora对于MySQL frm files文件传送，因为不重要，只相当于某些系统级的meta data的改变。

Log我们还必须详细解释，请接着往下看《Log的妙趣》。

## 二、Log的妙趣

我们将数据库简化成一个字符串值，比如：初始值是，I am Tony。然后我们的事务Transaction修改数据库，加三个词word，先改写为: I am Tony coding for world。然后再增加两个word，改写为：I am Tony coding for world in China。这可以是一个事务分两次执行，也可以是两个事务并发执行（假定追加两个词word的并发Transaction的修改效果，在时间上是后发生）。

我们将每个修改动作作为Log record记录下来，这样上例中就存在两个Log record，分别是a和b：

* log record a: 尾部追加 coding for world
* log record b: 尾部追加 in China

我们发现，在数据库初值的基础上，数据库最后的终值，是每个log record所描述的动作，按顺序作用于前一个值的最后结果，即

```
 ------------                                                                 ---------------------
|  数据库初值  |    +  log record a     +        log record b             =    |      数据库终值      |
|            |                                                               |  I am Tony          |
| I am Tony  |      I am Tony             I am Tony coding for world         |  coding for world   |
|            |      coding for world      in China                           |  in China           |
 ------------                                                                 ---------------------
```
上面的 + 号，相当于apply，就是根据log record的内容，然后做相应的动作（本范例的动作是：尾部追加几个词，append words）。

上面log record的总和，就是redo log。

我们发现有下面几个规律：

* redo log record必须有顺序，而且必须唯一确定，比如：我们如果将log record b放在log record a前面，数据库的终值会变成I am Tony in China coding for world，语句算通顺，但结果不一致。再比如：如果我们log record b出现了两次，则终值会变成：I am Tony coding for world in China in China，语句不通顺了。

* 如果缺少了中间某个log record，我们不能形成正确的终值。比如：如果缺少了log record a，那么终值是：I am Tony in China。语句是通的，但结果不对。

数据库处理遵循上面的原则，即将修改动作log record记录到redo log并存盘。这样，即使数据库终值没有及时存盘（掉电或数据库进程被杀），我们一样可以通过磁盘上的数据库初值，再加上磁盘上的redo log，获得正确的数据库终值。

数据库只所以用redo存盘，是因为整个数据库存盘代价太大（cost is big）。试想一下这个字符串初值长达100G大小。而redo log record不大，但历史（比如：一天）合起来的log record以及对应的DB会很大。所以，我们要redo不断及时存盘（而且是连续的追加方式，无需修改前面的存盘内容，这样磁盘工作效率特别高），而DB就可以适当延时存盘，可以部分存盘（part dirty pages flush，比如：100G的数据库，按16K一个page，分别存盘，因为实际上MySQL的redo log record就是针对具体某个page），可以同一个page被重复多次存盘，也可以合并存盘（check point）。

因此：

1. 我们给每个redo log record分配一个LSN（log sequence number），并保证：LSN是唯一的，且LSN是递增的。这样顺序和唯一性就得以保证

2. 我们让redo log及时落盘，即至少要保证Transaction commit时，必须redo先落盘。这样，发生任何意外，commit的东西，绝对不会丢。

LSN是一个重要的定义，我们再做强调

>LSN: log sequence number，是指每个redo log record的唯一标识，它是递增的，标识了数据库在时间上的连续的修改动作，即按照这个LSN的顺序，不遗漏地进行apply，我们总能从数据库的某个初值，达到未来截止某个时刻的绝对一致的终值。

对于数据库有一个麻烦的地方叫：后悔。后悔的意思：Transactionn允许abort，数据库的值必须roll back到前面一个值，比如上例中，我们有两个Transaction，一个做动作a，一个做动作b，但允许Transaction a后悔，不commit，而是roll back。

如果根据redo，我们发现，实际上，我们只要将redo log record里面的动作反着做，我们就可以得到修改的前值。为了计算简便，我们做一个对应的undo record log（如果动作是完全覆盖式，则必须需要undo log），如下
```
时刻:               time 1            time 2                              time 3

DB cache内存里：    I am Tony          I am Tony coding for world          I am Tony coding for world in China

redo log record:                      append: coding for world            append: in China 
undo log record:                      delete: last three words            delete: last two words

DB in disk:        I am Tony          I am Tony                           I am Tony
```

如果Transaction a，在即time 2之后，time 3以前进行roll back，我们只要在内存I am Tony coding for world这个值，对应的undo log record: delete: last three words，去执行这个undo动作，我们就可以获得I am Tony这个合适的值。

如果在time 3以后，且Transaction b还没有roll back（Trannsaction b可以是committed，也可以是uncommitted)，如果Transaction a此时做roll back，我们必须连续读两个undo log record，然后计算得到，这是从尾部倒退五个单词word，然后开始删除三个单词，即从I am Tony coding for world in China，变回I am Tony in China。

所以，当前内存的DB（或者更准确而言，是DB里的记录，tuple），必须有一个指针，指向对应的undo log record。然后，历史的undo record log，必须形成一个链表，构成历史遍历，这时，任何一个Transaction需要roll back时，都可以从指针开始，遍历这个undo log record链表，然后回复到合适的值。

undo log record形成链表，需要在其内容里，记录了链表里相关（即链表里的前一个）的undo log record的指针。

我们不用对整个数据库，做成整体一个对象的redo log和undo log，如果这样，遍历代价太大。我们可以对整个数据库，按照tuple和page为单位，作为redo和undo的目标对象，然后形成相应的链表。这样，每个tuple都有自己对应的undo链表用于roll back，同时，每个tuple，都有对应的连续的redo log records（无需形成链表，只要前后时间保证），用于crash and recover时形成最终值并可以存盘。

undo log还带来一个MVCC的好处。

我们在undo log record里，记录是哪个Transaction做的动作（如果tuple不是第一次生成的话，即Insert Type）。

我们假设磁盘的初值I am Tony是一个committed value。

试想一下还有另外一个read only transaction，它的Isolation被设置为READ COMMIT级别，即它不允许dirty read（不允许返回uncommited value）。

如果read only transaction读到的时刻是time2，内存的值为I am Tony coding for world，如果Transaction a还没有commit，那么，它必须去读对应的undo record log（通过tuple的指针），然后简单计算获得前值是I am Tony，然后才能保证此值是一个committed value（即最开始磁盘里的初值）。

如果read only transactionn读到的时刻是time 3，内存的值已经变成I am Tony coding for world in China，同时Transaction a和b都没有commit，那么它必须通过undo链表，去计算得到I am Tony这个committed value。但如果Transaction a已经commit，而Transaction b还没有commit，那么它必须顺着这个undo 链表，得到此时的committed value是I am Tony coding for world。

所以，下面的结论对我们后面的分析很重要：

1. undo log支持MVCC，因此可以支持各种Isolation级别的读写。

2. 为了支持各种Isolation，除了需要undo log，还必须知道当时所有的有写Transaction的运行状态，即某个时刻的transaction list。InnoDB里，只有写的Transaction才有Transaction ID（在第一个有写的SQL语句里分配），才会在transaction list里。read only tansaction是无Transaction ID的，不会出现在transaction list里。

注：实际Transaction运行中，不需要实时针对这个transaction list的全局数据做查询，而是一个叫read view的实现，即对当时的transaction list做一个snapshot拍照，然后用vector存储在Transaction本地局部数据。如果Transaction的Isolation被设置为REPEATABLE，那么只做一次read view拍照；如果是Isolation是READ COMMITTED，则事务中有读的每个SQL语句，都会形成一个read view。 

注：上面的redo、undo范例中，是抽象模拟。实际对于任何一个field的更改，redo和undo都是记录整个field value。但是，由于一个tuple是由多个field组成，而redo和undo只记录被修改的field的值，所以，上面的抽象是可类比的，即某个tuple的整值，必须通过tuple初值，经过所有的redo log record计算获得最新值。而对应历史的旧值，或者roll back回到某个旧值，必须对undo log record进行遍历，

## 三、Aurora的写Write : quorum write

### 3-1、quorum write的实现

quorum是一个洋文，它的意思是选举里，必须达到法定人数，才算有效。对于Aurora，它有6个copy，因此系统设定quorum = 4。即4个Storage nodes写成功，才算是quorum write写成功的前提条件（**注意：还不算写成功**）。

前面的描述知道，计算层和存储层时通过网络通信的，而且都是独立的机器node，所以，站在发起write的master眼里，是收到Storage node的response，这个Storage node才算成功，而且收集了至少4个成功返回response的Storage node，master才认为这是写成功的必要条件。

我们又知道，Aurora里，master只发送redo log给Storage node，这个redo log在Computing node内存里，是按LSN顺序不断尾部追加redo log record的，那么Aurora如何处理后面这两个麻烦：其一. 异步的通信；其二，分布的6个Storage nodes。

Aurora的解决办法是：

1. master并不强求上一个redo log record写成功，才发下一个redo log record，即可以乱序。

2. 如果某个redo log record暂时没有收集到4个response，就重试重发，因为假设整个存储层是永久有效的（见上文描述），所以，这个write也是最终能保证收集至少4个response。

3. 当某个redo log record收集了4个response，master可以不再给剩下的2个Storage node发write请求，因为系统允许至多两个Storage node失败（例如：停机维护）。

4. 当之前的所有的redo log record都收集了至少4个response，当前这个redo log record也收集了至少4个response（即必要和前提条件），master才认为当前这个redo log record写成功，即还必须有之前的redo log record连续写成功的约束。

5. redo log record对应的动作在master上是否执行，和这个record是否写成功，没有关系，除非这个动作是transaction commmit。即非commit动作不受quorum write的约束和影响（commit的影响请见下面的《对于computing node (master) 的麻烦》）。

补注：quorum字面上看（以及很多其他系统的实现），并不一定强制要求超过半数。Aurora对于write quorum要求是4，是有特别原因的，详细不作解释，看论文。

### 3-2、quorum write的好处

因为乱序，所以，可以并发。即站在master眼里，上一个redo log record暂时还没有成功，并不影响处理下一个recoord发送给存储层。

网络通信里，最影响效率的是：必须等待对方回应，才能处理下一个请求（这样网络带宽的利用率是非常低的）。比如：我们知道App for RDB，一般一个数据库连接上的吞吐Throughput都不够多，不足以达到DB的最高效能，一个很大的原因就是，App必须处理了一个SQL事务，才能发送下一个SQL事务，因为上一个SQL事务的结果，可能是下一个SQL事务的某个条件，它们有相关性，所以，单连接不能并发。要实现RDB的全部效能，我们必须用多个数据库连接去完成，而且这些连接上的并发的SQL事务，没有相关性。

所以，Aurora的quorum write的乱序发送，对于吞吐Throughput有很大好处，因为少了前后的约束。

特别是，非commit的redo log record里的动作，在master上可以直接产生效果，不受quorum write是否成功的约束，这样，可以让master的事务执行基本不受网络和存储的约束，能够很好地并发和快速地执行，这也就意味，master上的DB cache页面内容，可以远远领先于存储层对应的页面内容。注：为了防止超前太多，Aurora加了一个LAL约束，LAL = 10 million，即不能跑太快，超过千万。

### 3-3、quorum write的麻烦和如何解决

#### 对于Storage node的麻烦

Storage node收到master的redo log record，这样，就可以根据本地的存储（page的初值），以及这个redo，形成未来的对应的值。

但在《Log的妙趣》里的分析知道，这个需要保证redo log record是连续，无中间漏洞的。

但quorum write的模式决定了，Storage node收到的redo log record可能是中间有漏洞的，而且master有可能一直都不发来对应这个漏洞的record。

所以，对于Storage node，弥补这个漏洞，需要两点:

1. 需要识别哪些是漏洞

2. 需要到其他Storage node里，将这些丢失的record补回找到，补齐漏洞

先看如何识别漏洞:

如果LSN是严格增1连续的（continously increesing，注意：只是连续increasing，不保证continous，只有单调加1的连续，才是continously increesing），那么问题很简单，比如，某个Storaage node收到的redo log record的LSN是如下的（假设LSN从1开始起始）：
```
1、5、5、2、3、7、6、9
```
我们简单做一个Sort，然后遍历，马上知道漏洞是4和8。

但是，Aurora不是这样处理的（因为LSN并不是严格增1连续的，即continuous，它只保证唯一unique和递增increasing，实际上，LSN是对应record在redo里的字节偏移量，byte offset），所以，Aurora用了另外一个技术，链表，即每个redo log record，都记录了前一个redo log record的LSN，即prev LSN。

这样，Storage node可以简单通过prev LSN，就知道是否有漏洞和漏洞是哪些LSN。

然后，发现漏洞的Storage node，通过gossip通信方式，向其他Storage node询问这些丢失的LSN。之前分析我们知道，因为quorum write模式，保证：总有至少一个其他的Storage node，有丢失的LSN在那里存在。

这样，Storage node也就可以保证：redo log record，经过一段时间的异步（含被动从master获得，以及主动gossip从其他Storage node获得），可以完整一致地，在本机上形成一个链表。我们再用这个连续的无丢失漏洞的redo log，进而可以获得一个（或多个）新的存储数据（因为初值已经在Storage node的本地磁盘保存了）。

你可能问：如果master上的Transaction roll back怎么办？

答案很简单，当Transaction roll back时，也会形成对应的redo log record，Storage node只要执行这些record里的动作，就能回到roll back所希望达到的数据状态。

所以，Storage node必须缓存redo log record，遍历发现漏洞，然后gosssip其他Storage node补齐这些漏洞。

对于无漏洞的连续的redo log record，Storage node可以放心地一个接着一个地进行apply，获得对应的数据库状态。而且状态可以不只一个，如果条件允许，我们可以用新的状态覆盖旧的状态，或者，删除旧的状态，即回收purge（也可以叫GC，Garbage Collection）。为什么不用一个状态表达呢，为什么说条件允许，为什么要purge or GC？我们后面（《slave的读read》）会涉及这个知识点。

补注：Aurora的prev LSN是比较复杂的，正如我们前面讲到的，它还有segment shard，同时需要照顾磁盘存储基于block这个单位（即对应的page，磁盘的block和B tree的page，是一一对应的），所以Aurora里面，是存了三个prev LSN，分别是，基于整个redo log record链表的prev LSN，基于segment的prev LSN，和基于Block的prev LSN。本文为了简单，只笼统地说了一个prev LSN，但这不影响整个概念的阐述。

#### 对于computing node (master) 的麻烦

从前面的分析，我们得知，redo log record必须保证次序完整，才能保证数据库的最终一致性。

站在master角度，稍有麻烦，因为发送是乱序的而且可以重发，所以，某个时刻，不能保证先发先到。因此，master必须记录发送队列queue，以及接收状态（某个recoord收到几个response，注意：实际实现为了效率，是按批packet发送的，所以只需记录packet的状态，我们这里用record作为单位，是为了简化理解）。但是，经过一段时间，master肯定可以收集到所有发出的redo log record的足够（大于或等于4）的回应response，只要master不死（crash），一直活着的master，能保证截止到某一个时刻（即某个LSN），前面的所有的redo log record是连续写成功的。

因此，一直有效（活的）master可以保证redo log record的quorum write，是可以做到：一直连续成功的，而且不断推进的，只是要异步等待一点时间。因此，master可以根据此，判定写存储成功，不断推进master里的并发事务处理并相信相关的落盘保证。

我们来分析一下transaction commit的特别要求：

传统的MySQL，在收到App的commit请求时，必须先生成对应的redo log record（先在内存里，即redo buffer），然后必须保证写盘成功（flush to disk，同时也保证之前的redo log reccord和相关的undo record log也写盘成功），然后才能接着处理后续的相关内容内容，包括解锁、改变Transaction状态（从transaction list里删除此Transaction ID）和返回commit成功信息给客户App。

如果到了Aurora这里，生成的commit redo log record收到了Storage node的四个response，虽然此record被标识某种成功了（success of collecting quorum response），但仍不算write success，必须保证前面的所有的乱序和异步发送的redo log record也标识成功（success of collecting quorum response），此commit redo record才算写成功（success of quorum write），然后才能接着处理解锁、改transaction list以及回应App成功这些动作。

但注意：其他非commit类型的redo log record的动作，master可以继续做，不受任何约束。那些修改某个tuple（对应的page）里的相关内容，可以继续生成redo log record，并执行这些动作的相关效果，包括且不限于下面这些操作：从存储层读某page，修改DB cache里的某个page，分裂和合并（split or merge）某些page以保证整个B tree的完整一致，等等。

所以，quorum write的异步性和分布式，并不影响master的并发的事务的执行效率，除非到了commit这个特别阶段。而且，到了commit阶段，也只影响当前提交commit请求的transaction（和对应的某个数据库连接），其他并发的Transaction并不受影响（除非受数据库的内部锁的影响，因为某个锁可能是正在等待commit的transaction持有的）。

注意：read only transaction的commit不受quorum write success影响，因为read only commit没有生成redo log record（但read only transaction仍受数据库内部锁的约束，不过因为MVCC，锁的影响极小）。

这里，Aurora引入一个重要的概念，**VCL**，其定义如下

>VCL: 就是Computing node（首先获得和唯一判定的：只能是master）看到的最后一个完成quorum write成功的redo log record（即最大的quorum write成功的LSN），即保证之前的（LSN更小的）redo log record都已经quorum write成功。

如果再转义一下，VCL就是站在Computing node视角，针对存储层，落盘成功的最大的LSN，且保证之前的LSN全部落盘成功。

注意：Aurora master实现VCL计算时，不是通过保存所有的redo record log记录的状态进行的，它只要收集所有Storage nodes的redo log record的连续状态，然后简单计算即可获得这个VCL。这样一是简化了master上的状态保存状态，二是可以保证六台Storage node（如果都活的话）都满足VCL，或者至少哪4台Storage node满足VCL。尽管这个通过Storage nodes汇报而计算获得的VCL可能比如果master全部本地缓存全部状态而得到的值要低，但这个VCL已经足够了。

我们再加入一个相关的**VDL**，其定义如下：
>VDL：是截止到VCL的最近的一个mini transaction logic mini commit log reecord点（也是一个LSN，但必须是最后一个mini transaction里面的最后一个LSN）。

因为mini transaction保证了B tree的完整性（否则，如果有split和merge动作只完成一半，整个B tree的遍历traverse会出错，即mini transaction定义了一连串split和merge页面动作，以保护后续的其他事务对B tree可以安全遍历）。而一个mini transaction形成的多个（最少可以一个）redo log records是一个整体，这里面的最后一个LSN，相当于逻辑上的mini transaction logic mini commit log recoord。而VDL就是这样一个logic mini commit log reecord，它最接近VCL。

细节我不描述，详细可参考InnoDB的mini transaction的说明。你只需要知道：

* 一个Transaction是由多个mini transaction组成的（包含其roll back过程）
* mini transaction没有roll back概念，所以一个Transaction如果roll back，只不过是又产生了新的mini transactionn并执行
* 一个mini transaction形成的redo log records，中间不会被另外一个mini transaction插入，即从redo log上看，mini transaction log records是连续的
* 一个mini transaction，一旦开始，一定要完成。这包括：中间执行如果需要锁不会产生死锁（执行前开始前获得的锁可以不算，因为可以发现死锁然后roll back），中间如果splt/merge的页，不会让其他事务读取（比如：通过Lock和Latch实现），即它是个atomic动作，要么开始前do nothing，要么全部完成do all，而且中间执行不受外界影响同时也不对外界的其他事务产生非法错误的影响
* 当一个mini transaction完成后，它保证其他事务可以安全地浏览整个B tree
* 一个mini transaction可以产生最少一条，也可以是多个redo log record，这里面的最后一条相当于mini transaction的logic mini commit
* VDL不过是一个logic mini commit，它最接近VCL
* VDL小于等于VCL，因此像VCL一样，能保证截止到VDL这个LSN，存储上的page数据是肯定存在的

## 四、master的读read

首先，正常情况下（非crash & recover阶段），那么，不管masster，还是slave，如果发现需要读的page不在DB cache里（cache miss），那么它只需要到一台Storage node去读，而且**不是quorum read**。注：crash & recover阶段的quorum read，对于读read，quorum是3，而不是4，因为这可以保证读到最新的值，本文不详细讨论crash和recover以及相关的quorum read。

为什么？

首先，quorum read的cost比一台读，要大不少，因为latency是最慢的那个node，网络带宽占用也大了几倍。

其次，没有必要。

因为Computing node可以知道每个Storage node的完成情况，比如：是哪4台Storage node，保证完成VCL，它可以选择有此完备（complete）数据的其中的一台去读即可。而且，可以做到基本选择最快的那台Storage node去读（算法我不描述，看论文，或者想想sample latency seldomly这个思想即可）。

master只要保证一点，不要读到旧数据stale data即可。那什么是stale data？

比如：

master想要获得page number = 101的一个数据，它开始是在内存里（DB cache），所以master可以自由修改它，并假设产生了三次修改，分别对应redo log record的LSN为5、7、8(还有一个LSN=6的record，但不是针对page 101的)。即Page 101在DB cache里对应的最新的修改的LSN = 8。

这时，master因为DB cache满了，需要淘汰（evict）一个page，然后选择了101这个page，随后，事务处理又需要这个page的数据，这时，master必须发出一个到存储层的请求（找那个最快的Storage node去读）。但上面分析了，因为quorum write是异步的，假设此时VCL=6（因为LSN=7还是空洞），此时，从存储层读出的page数据就不对，是个stale data，即没有apply LSN=8的数据。

对于传统MySQL以上不存在问题，因为传统MySQL的算法是：如果要evict某个page，并且发现这个page是dirty page，必须先写盘。未来从本地磁盘读出来，就不会出现stale data。即传统MySQL中，page 101 evict时，已经保证apply了LSN为5、7、8的up-to-now data，即LSN=8的page 101保证落盘，

但Aurora的异步特征（见上面的分析，DB cache apply不受redo log发送是否完成的影响），同时Auror不会做像传统MySQL那样的本地的flush dirty page（前面说过，master不会向Storage nodes发送page信息，而只有redo log信息），就会出现上面这个麻烦。

解决方法也很简单：首先我们需要知道，每个redo log record，都最多只针对一个页面（有些record不针对具体某个page，比如：MLOG_MULTI_REC_END，但这些record不会修改page），对于每个page in DB cache，其上都有LSN号，对应最后一个apply和此page相关的redo log record。Aurora master要evict的dirty page，必须保证已经在存储层落盘，所以只要加上一个evict限定条件：这个dirty page上面的LSN，必须小于或等于VCL。

这时，Aurora的master，就可以放心地evict这个page，未来再需要了， 直接去读存储层的对应的page，由于VCL的保证，那么这个存储在存储层的page，一定是保证同步的up-to-now的page。

Aurora的论文，是用VDL做保证判断。但我个人的理解，对于master，VCL就足够，但由于VDL小于等于VCL，所以，用VDL去做evict约束条件，没有任何问题。

其核心原理在于：在master上，当用这个page LSN <= VCL（or VDL）的约束去evict某个页面page时，我们保证master从存储层读到的page（而且是存储层计算获得的最新up-to-now的page数据），一定是当时evict时的数据状态，然后，我们接着对这个已经存在DB cache内存的page进行操作时（对应了后续产生的redo log record for this page），一定是一致的。这个包括所有针对这个页面的任何写操作，如update、merge、split、delete等。

你可能担心，如果evict然后read的page，此时不保证B tree一致性（traverse会出错），怎么办？对于master，应该没有这个问题，因为master应该设置了相关的锁，保证这些page不会被其他事务访问到，从而维护B tree的一致性（读出page且当时B tree不一致的事务应该是写且上锁的事务，即当时导致B tree不一致的事务，它应该继续工作，将后面的page补齐，从而在本mini transaction里最终让B tree保证一致性）。注意：这个仅对Aurora master有效，对于Aurora slave，其实现机理不同，不能通过VCL保证B tree一致性（请继续阅读，直到《slave的读read》）。

在《Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases》论文中，我人为下面的话是有错的：

```
The guarantee is implemented by evicting a page from the cache only
if its “page LSN” (identifying the log record associated with the
latest change to the page) is greater than or equal to the VDL.
```

首先，greater than，我认为需要改成less than。如果是谈master，VDL可以改为VCL。但如果是slave，则必须是VDL（为什么，往下看，直到《slave的读read》）

## 五、slave的同步（write for slave）

前面说了，Aurora的计算层可以有多个slave，它需要同步master的write。

我们在《Log的妙趣》里分析，只要有redo log，然后一步一步地apply顺序的redo log record，就可以像存储层那样，形成一个一致的data set。这视乎不需要undo log什么事？

这是不对的，slave需要undo log，因为两个原因：

1. master可能crash，我们需要promote slave to master，这需要undo log，用于crash recover
2. slave还需要服务其他（连接到本slave node的App）提交的read only transaction，而这些read only transaction需要MVCC，所以，这也需要undo log。

那slave的write同步，就既需要master产生的redo log，也要redo log对应的undo log（也是master产生），同时，slave必须undo log本地存盘（因为Storage node没有undo log）。

同时上面的分析我们还知道，为了支持MVCC和各种Isolation，我们还需要知道transactionn list，即所有写的事务的Transaction ID列表。这个可以通过分析redo log得到，因为redo log记录了每个Transaction ID的诞生和消亡，但是为了简化slave的计算，Aurora master是将这些信息作为附加信息和redo log一起发给slave的。

然后，我们就需要考虑如何apply redo，我们能否像Storage node那样，任意地自由地在slave上进行apply？

我们不能。为什么？

因为数据库的写操作，有mini transaction的约束。

什么是mini transaction的约束，简而言之，就是为了保证整个B tree的一致性，一些record log record，必须连续且原子地执行，否则，会导致B tree的不一致而引起的非法错误。试想一下，一个tuple的插入、删除和修改，都可能导致B tree的页的分裂split和合并merge，因为牵扯到多个页page，它们必须一起完成。否则，做到一半事务暂停（比如：线程休眠），那么其他事务如果遍历此B tree，就会导致非法。

在InnoDB里定义了mini transaction，就是为了这个目的。这个mini transaction的执行，必须是原子的，即中间不可打断（除非crash）。

因此，当从master传过来的redo log进行apply到slave本机时，我们必须原子性地执行一个个mini transaction。所以，在slave上这个mini transaction的apply，是一个类似stop the world的操作（类比Java的GC），这时，其他read only transaction for slave必须停下来，等这个redo log records of one (or last one) mini transaction完成。只有这个atomic apply完成后，read only transaction for slave才能唤醒，并且并发地继续执行，因为这时，B tree是一致的，不会引起这些read only transction for slave遍历B tree时，发现一个broken B tree，从而导致非法。

Aurora解决这个问题的方法很简单，master将redo log按mini transaction分割，按chunk方式（即不允许发送一半的mini trannsaction redo log records）发过给slave，因为redo log里，任何一个在master上的mini transaction形成的redo log records，中间都绝对不会出现其他mini transaction生成的redo log record，即master在redo log的生成中，已经保证了，它是按mini transaction连续的。

我们又知道，在Storage node里apply redo时，需要得到初值（Storage node的本地硬盘上已有），然后才能获得终值（按page为单位目标）。

那么slave上，如果其DB cache里没有对应的page，是否需要到存储层去读出其初值，然后进行apply呢？

答案是：不用。因为我们的目的是保证B tree的一致性，如果此page不在slave的DB cache里，我们无需apply，也一样保证B tree的一致性（slave假设未来不在其DB cache里的page，可以从存储层读到绝对一致的page）。

这带来什么好处？这样，所有在slave的apply工作，都是针对内存而去，没有IO（除了undo log，但undo log的落盘很快），这将使slave的atomic apply for all redo log records of one mini transactionn这个动作的cost非常低，从而使stop the world的代价绝对小，从而让read only transactions for slave可以更早地进入下一步并发工作，从而带来整个slave的吞吐Throughput得到提高。

slave write的意义何在？

```
slave write，相当于照搬了master某个内存快照，但只作用于slave内存里的对应的page，而且保证了B tree的一致性
```

但是，你会想到一个问题，如果read only transaction在slave上执行时，需要一个page，而这个page不在DB cache，它难道不需要到存储层去读盘吗？

当然要去存储层读，但怎么读，是个非常tricky的东西，请往下看。

## 六、slave的读read

当read only transaction需要某个不在slave DB cache的page时，slave必须到存储层去获得这个page。注：不在DB cache的原因可能是：1. 它可能本来就不在，2. 开始在，接着被slave evict了（在slave上，evict时，不考虑像master那样的约束）。

那slave能否像master那样，根据VCL的限定去某个Storage node上拿最新up-to-now的page？

不能！

首先，和上面的《master的读read》分析一样，slave不可以拿VCL之后的page，即不可以拿Storage node还无法形成的page。

其次，我们在《slave的同步（write for slave）》里分析了，slave上的read only transction，还必须受mini transaction这个约束。即如果我们从Storage node拿到的up-to-now的page（因为其截止点是Storage node看到的VCL），不能保证B tree的一致性，那么会让read only transaction遍历B tree时非法。所以，我们必须拿截止到某个VDL的页面。

最后，slave从存储层拿到的页面，应该和那个时刻（即slave write完成的最后一个LSN）master里面的DB cache完全一模一样的数据。

注意这个事实和麻烦：**slave上的VDL和master的VDL，在绝对某个时间上（理论上master和slave统一看到的物理上的唯一时钟），是不一样的**。

slave是异步地接收master的redo log，所以，slave认为的VDL和master的VDL，在某个时刻，并不一样，但有一点可以做到，即slave认定的VDL，一定小于或等于master的VDL，master通过chunk方式发来的redo log records的最后一条，就是slave认定的那个VDL，而且此LSN，master可以保证小于或等于master自己认知的VDL（超过了master不发即可）。

因此，我们可以定义一个slave VDL，即从master发来的VDL，它就是master通过chunk模式（mini transaction），所对应的最后一个LSN，而且一定小于或等于master自己的VDL。

即slave在DB cache里面必须做到：它apply了某个redo log records of last one mini transaction后，再加上这个从Storage node拿到的page，它的内存状态，必须和那个时刻（slave VDL）的master的内存状态，一模一样。

如果此page已经在slave的DB cache里，我们apply了这些redo log records，我们肯定可以保证此时slave page是和master（对应slave VDL时刻）是完全一样的。这个在上面的write for slave分析中已经阐述过了。

但上面分析说了，slave apply是可以略过（omit or skip）那些不在DB cache for slave的页面page，所以，当slave到Storage node去读某个page时，它必须读到截止到slave VDL时刻的页面值，即：如果从master角度看，这些slave从Storage读出的page，必须是master在slave VDL时刻完全一模一样的内存page（注意：master DB cache里，此时，即物理上slave write时刻，可能有更新的页面内容，而且，这个对应更新页面内容的redo log，可能也已经发给了Storage node）。

这个如何解决？

当slave从Storage node去读page时，它必须传入slave VDL作为参数，而Storage node在形成这个page时，不能用up-to-now的最新值，而必须是截止到slave VDL的page，而我们上面《Log的妙趣》分析了，我们可以通过page的初始值，然后再apply redo log records，获得任何一个时刻的page。因此，Storage node只要apply到小于和等于slave VDL的redo log records，就能返回合适的page值。

所以，Storage node必须保持redo log records的链表，这样，才能形成各个版本的page（有点类似MVCC）。

那么，在从Storage nodee加载这个页面到slave DB cache这个异步过程中，master又发来了新的针对这个page的redo log records，怎么办？

此时，slave必须将这些redo log records保留下来（不能忽略omit or skip，即不能认为这个page不在slave的DB cache里），形成针对这个page的临时链表，当存储层返回page数据后，slave立刻apply这个临时链表：redo log records for this page，此时，就生成了，针对新的slave VDL时刻的page，并保证和master那个时刻的一致性。

上面的分析还带来一个问题，Storage node如果一直保留redo log records，不会撑破Storage nodes的内存？

解决方法也很简单：在所有的slave中，总有一个最小的read only transaction，它起始的read snapshot是基于某个slave VDL，对于这个最小的VDL，它之前的redo log records，Storage node都不用保存（因为未来发来的slave VDL参数只可能比这个大）。而随着read only transaction for slave的结束，这个slave VDL一定是向前推进的（slave VDL会越来越大）。这个不断推进的最小的slave VDL，在Aurora里，被叫做Protection Group Minimum Read Point LSN (PGMRPL)。master和所有的slave交互，获得这个PGMRPL，然后发给Storage nodes，然后Storage node就可以根据PGMRPL，去做相关的清理purge工作，即从Storage node内存里的链表里，删除PGMRPL之前的redo log record，还可以删除相应的磁盘page image，也就是说，Storage node可以做purge或GC了。

补注：再解释一下，为什么master的读可以只受VCL的约束，而slave却要受VDL的约束，是因为master上的事务，是真事务，是有锁保障其安全的，即使某个时刻B tree不一致，因为master的锁机制，它保护不一致的B tree，然后再继续执行的mini transaction中修补这个不一致同时防止其他事务读到这个broken B tree（只需要保护B tree里的broken part）。但slave上，并不是真正的transaction在执行，slave write只是apply atomic redo records from master，slave只有apply时，才能用stop the world的方式（即slave并不用真正的事务锁），保证B tree由不一致转化为一致，如果中间切换read only transaction for slave来执行，就打破了这个atomic约束而且没有真正的写事务锁来保护，让read only transaction for slave可能遍历到不一致的B tree。

## 七、我猜测的Storage node的数据结构

提醒：下面是我根据上面的分析，对Storage node的数据结构的猜测，而且是忽略segment shard的（考虑segment，也可以推理出相应的数据结构）

```
NOTE: LSN is single digit (0-9)，Page Number (Page No.) is 3-digits (101, 202)

Storage node memory:
--------------------

input queue from master:  LSN = 2（101）、3（202)、5（202）、2（101）、4（101）、7(101)、9（202）

PGMRPL from master: 0

gossip queue: 7 (prev is 6) 、9（prev is 8）    so holes are LSN = 6 、8

HashMap: (key is page number, value is a list of redo log records for this page and guarantee sequence without hole）  

101:  2 -> 4 -> nullptr

202： 3 -> 5 -> nullptr


Storage node disk:
-------------------

Page(101) image: base LSN = 0
Page(202) image: base LSN = 1
```

当PGMRPL增加后，我们可以修改HashMap里的list，同时回收Disk里旧的image，用新的image替代。

我们可以在Disk上针对某个LSN缓存一个对应的image版本（如果master或slave发来计算请求的话），这样，就简化了计算，不用针对重复的LSN请求每次都重复计算。相应的数据结构就不再详述，这不过是个Dynamic Programming算法。

## 八、对于Aurora的”瓶颈在网络“的理解 

我认为Aurora提出的瓶颈在网络的说法是正确的。

请参考我写的一个文章：[《内存、网络、磁盘性能比较》](https://zhuanlan.zhihu.com/p/420534288)，一般而言，瓶颈在磁盘。

但由于Aurora采用了计算和存储分离的架构，而且存储是分布式的（而且有segment shard），所以，用多个Storage node来统一提供存储服务，所以，这里，磁盘已经不受限制了。

那么自然，整个Aurora的系统的上限，来自网络，即其最大的Throughput，最后受限于网络带宽的限制。

我自己的实践，Half Master/Master的一个案例，[BunnyRedis](https://www.zhihu.com/column/c_1431329604070342656)，其最大上限也是网络，而且只针对写。所以，在我的文章里[《分布式思考：我们需要分片Shard（含分库分表）吗？》](https://zhuanlan.zhihu.com/p/403604353)，如果假设一个qps平均是1K Byte的话，那么，针对BunnyRedis，这个写的上限是25M qps(两千多万)，如果读和写的比例是十比一的话，整个BunnnyRedis系统的上限是：近三亿qps。

## 九、Aurora性能的提升、如何超越Aurora的性能

在Aurora的测试中，对比传统MySQL，其读写有十倍以上的提升。

那么能否相比Aurora，针对支持ACID的RDB，再有至少十倍的提升呢？

我认为是可以做到的，详细可参考我的一个文章：[从Raft角度看Half Master/Master(两层解耦)](https://zhuanlan.zhihu.com/p/407603154)。

还有一个未来可能发表的文章《Half Master/Master模式在RDB OLTP下的性能⽐较分析》。

## 十、参考资料Reference

* [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Database](https://web.stanford.edu/class/cs245/readings/aurora.pdf)
* [Amazon Aurora: On Avoiding Distributed Consensus for I/Os, Commits, and Membership Changes](https://pages.cs.wisc.edu/~yxy/cs764-f20/papers/aurora-sigmod-18.pdf)
* [An In-Depth Analysis of REDO Logs in InnoDB](https://www.alibabacloud.com/blog/an-in-depth-analysis-of-redo-logs-in-innodb_598965?spm=a2c65.11461447.0.0.bdf6654adSY57h)
* [An In-Depth Analysis of UNDO Logs in InnoDB](https://www.alibabacloud.com/blog/an-in-depth-analysis-of-undo-logs-in-innodb_598966#:~:text=InnoDB%20uses%20Undo%20Log%20to,database%20can%20resume%20services%20first)
* [Deep Dive: InnoDB Transactions and Write Paths](https://mariadb.org/wp-content/uploads/2018/02/Deep-Dive_-InnoDB-Transactions-and-Write-Paths.pdf)
* [AWS Aurora (by 胡明)](https://zhuanlan.zhihu.com/p/391235701)
* [Aurora读写细节分析 (by 叶提)](https://zhuanlan.zhihu.com/p/508928878)
* [MySQL Engine Features InnoDB Based Physical Replication](https://alibaba-cloud.medium.com/mysql-engine-features-innodb-based-physical-replication-a345990bd266)