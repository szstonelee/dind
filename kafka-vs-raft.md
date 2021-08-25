# Kafka模式对比纯Raft模式简表

|  | Kafka模式 | 纯Raft模式 |  |
| -- | -- | -- | -- |
| 一致性瓶颈所做的事 | Only Log | Log + State Machine<br>(B+树 or LSM树) | [分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015) |
| 支持外部的模式 | Half Master/Master | Stateless<br>(计算和存储分离) | [BunnyRedis要解决Redis的一致性Consistency问题](https://zhuanlan.zhihu.com/p/392637293) |
| Batch/Locaility优化 | Log比较容易 | 可做，但难一些，特别是State Machine | [分布式思考：批发Batch是个好东西，用足它](https://zhuanlan.zhihu.com/p/401190110)<br>[分布式思考：近邻Locality是个好东西，用足它](https://zhuanlan.zhihu.com/p/401569843) |
| 磁盘优化 fsync/WAL/No Disk | Kafka已支持，外部系统（State Machine）需自我实现 | 可做，但现在很多未实现 | [分布式思考：我们需要fsync吗？](https://zhuanlan.zhihu.com/p/400099269)<br>[分布式思考：我们需要WAL吗？](https://zhuanlan.zhihu.com/p/400338569)<br>[分布式思考：我们需要磁盘吗？](https://zhuanlan.zhihu.com/p/400480015) |
| Failure忍耐度 | 只剩最后一个broker仍正常 | n/2+1 (n is cluster number)<br>cluster=3: quorum=2; <br>cluster=5: quorum=3 | |
| 忍耐度为2时磁盘/网络最小倍数 | 3倍 | 5倍 | [少就是多，多就是少](https://zhuanlan.zhihu.com/p/402990609) |
| 单机磁盘写放大倍数 | 1 | 最少10(如果LSM树) | |
| 第三方依赖 | Zookeeper(或Raft) | 全部自我实现 | [少就是多，多就是少](https://zhuanlan.zhihu.com/p/402990609) |
| 其他 | 还需进一步实现State Machine，但State Machine可以做到无单点限制 | 全部自我实现，但单点瓶颈过大 | [BunnyRedis](https://zhuanlan.zhihu.com/p/392646113)<br>[Kafka is Database](https://zhuanlan.zhihu.com/p/392645152)<br> [分布式下一致性Consistency的代价cost](https://zhuanlan.zhihu.com/p/399639015)<br>[少就是多，多就是少](https://zhuanlan.zhihu.com/p/402990609)
