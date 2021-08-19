# 分布式思考：近邻locality是个好东西，用足它

## 前言

虽然我主要谈的都是分布式，以及分布式下的优化。Locality，主要适用于单机。但它非常重要，所以在分布式（甚至任何其他系统设计）里都要重视它。

## 什么是Locality

所谓Locality，就是将东西（data）尽可能放在一起。

所谓远亲不如近邻。

比如：

在Java中，我们知道有一个List接口，让一群Item共同组建一个Collection。然后对外提供一致的接口：如add(), get(index), remove(index)

而我们知道，List是接口类，它至少有两个重要的实现类（concrete class or implementation class），一个是ArrayList，一个是LinkedList。

其中ArrayList，是用一个动态Array去支持这个List接口的具体的数据结构；而LiinkedList，是用链表去支持List接口的具体的数据结构。

而Array，从数据在内存分布的角度看，是紧挨的；而链表则是分散的。

于是我们说：ArrayList的Locality要远远好于LinkedList。

C++下的std::vector和std::list，也是一样的道理。

## 为什么Locality对性能友好

### 从内存看

从内存看，主要是它适合当代的CPU多core的架构体系。

大家可以参考之前的一个文章：[单线程就比多线程性能差吗？不一定](https://zhuanlan.zhihu.com/p/397039359)，里面有对各类操作的时延的大致判断。

如果我用Array存储数据，它的大小在L1 cache里，那么我第一次找需要100ns，因为要到主存Main Memory里去找，但它将缓存在L1 cache，这样第二次找就是不到1ns。

而如果是链表，由于它在主内存里是散落的，而L1 cache大小比较小（几K或几十K而已），所以，没有办法对整个链表进行缓存，所以每次查找都要到主存（或其他级别的CPU cache，如果链表太大，则在其他级别的CPU cache找到的可能性也不大），这样每次的时间就都是100ns。

这里对于第二次开始的每次查找，有至少100倍的差别。

所以，忽略第一次操作的cost，即使我用100次查找在Array里找到一个数据（几乎相当于一个scan），也会比只用1次在链表里找到的快。

所以，Big O在这里不完全适用。或者说，Big O是纯数学的，在实际编程里，我们还需要考虑真实的物理上的计算机是如何操作的。这样才是真正的Big O。

因此，有可能的话，尽量用Array去代替Linked List。特别是在整个数据集(比如：List Collection)都比较小的情况下，可以考虑Big O(N)去取代Big O(1)，反而可能带来更好的效果。如果只有几K，它适合L1 cache；如果有几十K或几百K或几兆，它也适合L2、L3 cache。（或者分段适合，即100M的Array，我也可以看成100个适合L2 cache的子Array，假设L2 cache是1M的话）

### 从磁盘看

在[Throughput, Bandwidth, Latency](https://zhuanlan.zhihu.com/p/399883427)和[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)，都有对磁盘工作机理的一些描述。

对于HDD，如果数据的Locality好，那么我们省了最昂贵的磁头移动（disk seek）。

但如果是SSD，好像不太明显。但是，我们要知道，数据库里经常进行pre-read，也就是说，在读一个数据时，连带对它附近的数据一起读，因为下次可能用到（主要原因是读一个数据，和连续读两个数据相比，在磁盘上的时间消耗差不多，不管是对于HDD，还是对于SSD）。而如果我们的数据如果本来就是紧邻的（如Log数据），那么pre-read的有效性就大大不同。

## 一些案例

### Redis里的ZipList

Redis使用到一个特殊的数据结构，叫[ZipList](https://redis.com/ebook/part-2-core-concepts/01chapter-9-reducing-memory-use/9-1-short-structures/9-1-1-the-ziplist-representation/)，这个数据结构，对于内存Locality非常好，尽管它从纯数学角度的Big O是O(N)。比如：在Hash数据结构上，Redis分两种实现，当Hash比较小的时候，它用ZipList存储，只有Hash变得比较大，Redis才将之转化为真正的Hash Table实现。

BunnyRedis也充分利用了Redis这个数据结构的特点，所以，如果你的Hash是ZipList，那么BunnyRedis将整个Hash存盘。而当Hash从ZipList转为纯Hash Table后，BunnyRedis才对其内部的field进行单独的存盘。详细可参考：[BunnyRedis的Hash的存盘策略](https://github.com/szstonelee/bunnyredis/wiki/Hash-storage)

同时，如果你去看Redis的用法，它非常在意内存的有效利用率。开始我以为它只是为了节省内存，从而在有限的内存空间里尽量多存储数据，现在我觉得它还有一个很重要的意义，就是这种设计思想，会更有可能导致数据Locality变好，进而让速度也得以提升。比如：你看Redis的Hash Table的Rehash，大部分语言，如Java、C++，都是在load factor在75%时（即整个Hash Table的使用量），就进行Rehash，而Redis很特别，它是在load factor为100%时，才考虑进行Rehash，即Redis里的Hash Table，如果数据比较大的时候，里面会有不少的collision，很多时候，一次读就能获得数据还是需要再走一下Hash冲突后的链表，从数学角度这样好像很不科学，但站在现代硬件角度，这样做，不一定性能就差（或者降低不多，而且trade off带来的内存有效利用率更值得）。

也许Hash collision的解决方案[Linear probing](https://en.wikipedia.org/wiki/Linear_probing)，是个更好的解决方案。

### 我自己做的一个SkipList Scan的优化

对于内存，我自己曾经做个尝试，大家知道，数据库中，我们经常有Scan的请求，就是做一段升序或降序的数据扫描。

而在内存里，我们用的升序和降序数据结构一般是树。

SkipList也是这样一个树，但平常它由于随机增、删、改的原因，基本数据是散列的，类似Linked List。但我通过定期（或符合某种条件下的）对树重整，让树里的有序数据在内存分配上尽量靠近，这样就加快了scan的速度，测试发现有近50倍的速度提升。

详细可参考：[Which Skip List is faster](https://hub.fastgit.org/szstonelee/elephant_eye_c_plusplus/blob/master/which_skip_list_is_faster.md)
