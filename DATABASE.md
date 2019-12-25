# 数据库
涉及相关知识

## 数据库mysql
### 事务相关
#### 事务的特性
事务的特性ACID,原子性，一致性，隔离性，持久性。
- 原子性：整个数据库事务是不可分割的工作单位。只有使事务中所有的数据库操作执行都成功，才算整个事务成功。
- 一致性：主要指数据库从一种一致性状态到达另一种一致性状态。
- 隔离性：一个事务的影响在该事务提交之前对其他事务不可见。--通过锁来实现。
举个例子，现实中两次转账是应该是不会相互影响的，比如A账户向B账户转账两次，每次都转账5元，那么最后一定是A账户少10元，B账户多10元。假设两次转账操作分别对应为T1和T2,如何两次转账是顺序操作的，那么如下图所示，
![avatar](https://user-gold-cdn.xitu.io/2019/4/1/169d7fd5bcbcc82f?imageView2/0/w/1280/h/960/ignore-error/1)
但是，T1和T2可能交错执行
![avatar](https://user-gold-cdn.xitu.io/2019/4/1/169d7fd5bd2cf68b?imageView2/0/w/1280/h/960/ignore-error/1)
这样转账就发生了问题。所以数据库的设计要保障其他的状态转换不应该影响本次的状态转换，这个规则称之为隔离性。
- 持久性：事务一旦提交，其结果就是永久的。即使发生宕机等故障，数据库也能将数据恢复。
其中，隔离性由锁机制实现，原子性，一致性和持久性由redo和undo日志来完成。



#### 事务的隔离级别
因为事务是具有隔离性的，理论上某个事务对某个数据进行访问时，其他的事务应该进行排队，在事务提交后，其他事务才能继续访问这条数据，但是这样的话对性能的影响太大。


##### 事务并发的时候可能会遇到的问题
事务在不保证串行的情况下可能出现的问题。
1.脏写
当两个事务同时尝试去更新某一条数据记录时，就肯定会存在一个先一个后。而当事务A更新时，事务A还没提交，事务B就也过来进行更新，覆盖了事务A提交的更新数据，这就是脏写。
2.脏读
如果一个事务A向数据库写了数据，但事务还没提交或终止，另一个事务B就看到了事务A写进数据库的数据，这就是脏读。
3.不可重复读
如果一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值，那就意味着发生了不可重复读，实例如下图所示：
![avatar](https://user-gold-cdn.xitu.io/2019/4/18/16a2f5b32bc1f76b?imageView2/0/w/1280/h/960/ignore-error/1)
4.幻读
如果一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来，那就意味着发生了幻读，示意图如下：
![avatar](https://user-gold-cdn.xitu.io/2019/4/18/16a2f5b32d7b9ada?imageView2/0/w/1280/h/960/ignore-error/1)
问题的严重性排序：脏写 > 脏读 > 不可重复读 > 幻读


##### 事务的隔离等级
我们上边所说的舍弃一部分隔离性来换取一部分性能在这里就体现在：设立一些隔离级别，隔离级别越低，越严重的问题就越可能发生。
读未提交，读已提交，可重复读，可串行化
SQL标准中规定，针对不同的隔离级别，并发事务可以发生不同严重程度的问题，具体情况如下：
| 隔离级别 | 脏读 | 不可重读读 | 幻读 |
| ------ | ------ | ------ | ------ |
| 读未提交 | Possible | Possible | Possible |
| 读已提交 | Not Possible | Possible | Possible |
| 可重复读 | Not Possible | Not Possible | Possible |
| 串行化 | Not Possible | Not Possible | Not Possible |

##### 事务隔离等级实现的方式
为什么有了事务这东西，还需要乐观锁悲观锁？事务是粗粒度的概念、乐观锁悲观锁可以更细粒度的控制；
比如抢票，假设余票只有1张；隔离级别可以保证事务A和事务B不能读到对方的数据，也不能更新对方正在更新的数据，但是事务A和事务B都认为还有1张余票，于是出票，并更新为0；
事务解决了并发问题，已经不存在并发问题了；
但是事务B读取的是过时数据，依据过时数据做了业务处理；
所以需要乐观锁或者悲观锁，来记录一个信息：当前已经读取的数据，是不是已经过时了！
事务有这么几种实现方式：锁协议、MVCC、时间戳排序协议、有效性检查协议，锁协议是事务的一种实现方式，事务 = 用锁封装的一个函数，可以重用而已，但是这几个事务的函数覆盖面太粗粒度了，所以有时候我们还得借助于锁来进行细粒度控制；
事务不能保证每个操作结果正确，售票时超卖还是会发生。
事务保证整个操作的成一个组，要么全做要么全不做 但是不能保证多个事务同时读取同一个数据
数据对象被加上排它锁时，其他的事务不能对它读取和修改；加了共享锁的数据对象可以被其他事务读取，但不能修改
事务可以用锁实现，可以保证一致性和隔离性，但是锁用来保证并发性；
隔离性和并发性有点类似，但是隔离性只是保证不会出现相互读取中间数据，却无法解决并发的问题

#### redo日志
[参考链接](https://juejin.im/post/5c3c5c0451882525487c498d)
##### redo log 概述
重做日志用来实现事务的持久性，即ACID中的D，由两部分组成：
- 一是内存中的重做日志缓冲(redo log buffer)  易丢失
- 二是重做日志文件(redo log file) 持久的
InnoDB是事务的存储引擎，其通过Force Log at Commit 机制实现事务的持久性，即当事务提交commit时，必须先将事务的所有日志写入到重做日志文件进行持久化，待事务COMMIT操作完成才算完成。

##### redo log 流程
redo的整体流程，以更新事务为例，redo log的流转流程如下图所示
![avatar](https://user-gold-cdn.xitu.io/2019/1/14/1684bc76cb5c3552?imageView2/0/w/1280/h/960/ignore-error/1)
1.先将原始数据从磁盘读到内存中，修改数据的内存拷贝；
2.生成一条重做日志并写入redo log buffer,记录的是数据被修改后的值；
3.当事务commit时，将redo log buffer中的内容刷到redo log file,对redo log file 采用追加写的方式；
4.定期将内存中修改的数据率新到磁盘中；

##### redo日志的作用
主要作用是用于数据的崩溃恢复

##### redo日志的写入时机
- log buffer空间不够时
- 事务提交时
- 后台线程不停的刷盘
- 正常关闭服务器时


#### undo日志
##### undo日志的概念
undo log主要记录的是数据的逻辑变化，为了在发生错误时回滚之前的操作，需要将之前的操作都记录下来，然后在发生错误时才可以回滚。

##### undo log的作用
undo是一种逻辑日志，有两个作用：
- 事务回滚
- mvcc
undo日志用于事务的回滚操作进而保障了事务的原子性。需要注意的是，undo页面的修改，同样需要记录redo日志。

### 分库分表
[相关介绍](https://github.com/Meituan-Dianping/Zebra/wiki/Zebra%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB%E4%BB%8B%E7%BB%8D)
当读压力太大，单台mysql实例扛不住时，此时DBA一般会将数据库配置成集群，一个master(主库)，多个slave(从库)，master将数据通过binlog的方式同步给slave，可以将slave节点的数据理解为master节点数据的全量备份。
### mysql主从同步方式
主从同步采用复制的方式，是Mysql数据库提供的一种高可用高性能的解决方案，一般用建立大型的应用。整体来说，复制的工作原理分为一下的三个步骤：
- 主服务器把数据更改记录到二进制日志（binglog）中。
- 从服务器把主服务器的二进制日志复制到自己的中继日志中。
- 从服务器重做中继日志中的日志，把更改应用到自己的数据库上，以达到数据的最终一致性。
从服务器有两个线程，一个是I/O线程，负责读取主服务上的二进制日志，并将其保存为中继日志；另一个是SQL线程，复制执行中继日志。
主从复制流程如下：
![avatar](https://upload-images.jianshu.io/upload_images/65300-b9a980010912df2f.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/640)


#### 半同步复制
用来解决主库数据丢失的问题；主库写入binlog日志之后，就会将强制此时立即将数据同步到从库，从库将日志写入自己本地的 relay log 之后，接着会返回一个 ack 给主库，主库接收到至少一个从库的 ack 之后才会认为写操作完成了。

#### 并行复制
用来解决主从同步延时的问题，指的是从库开启多个线程，并行读取 relay log 中不同库的日志，然后并行重放不同库的日志，这是库级别的并行。（前提是进行了分库）

一般来说，如果主从延迟较为严重，有以下解决方案：
- 分库，将一个主库拆分为多个主库，每个主库的写并发就减少了几倍，此时主从延迟可以忽略不计。
- 打开 MySQL 支持的并行复制，多个库并行复制。如果说某个库的写入并发就是特别高，单库写并发达到了 2000/s，并行复制还是没意义。
- 重写代码，写代码的同学，要慎重，插入数据时立马查询可能查不到。
- 如果确实是存在必须先插入，立马要求就查询到，然后立马就要反过来执行一些操作，对这个查询设置直连主库。不推荐这种方法，你要是这么搞，读写分离的意义就丧失了。

高并发的场景下回出现严重的主从不一致。要解决这个问题，要将上面的sql线程拆分成多个线程处理，
![avatar](https://upload-images.jianshu.io/upload_images/65300-3f3f56abaa9730d2.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/640)
coordinate对应这原来的 sql线程，具体执行binlog复制的是worker,分发策略的原则是
- 不能造成更新覆盖，同一行的更新必须被分发到同一个worker中
- 同一个事务不能被被拆开，必须被分配到同一个worker中
[具体的分发策略](https://www.jianshu.com/p/a7c7d5c42417)



## 秒杀场景的解决方案
- 在sql加上判断防止数据变为负数
- 数据库加唯一索引防止用户重复购买
- redis预减库存减少数据库访问　内存标记减少redis访问　请求先入队列缓冲，异步下单，增强用户体验


### 整体流程解决方案
```mermaid
  graph TD
  B(redis判断是否已经秒杀到) --> C(内存里面判断产品是否结束)
  C --> D(redis的decr预见库存)
  D --> E(发送mq消息)
  E --> F(消费mq并查询秒杀商品库存)
  F --> G(redis判断是否已经秒杀到)
  G --> |事务开始|H(减库存下订单写入秒杀订单sql加上判断防止数据变为负数)
  H --> |事务结束|I(结束)
```

## 缓存

**为什么使用缓存**
- 高性能：对于一些需要复杂操作耗时查出来的结果，且确定后面不怎么变化，但是有很多读请求，那么直接将查询出来的结果放在缓存中，后面直接读缓存就好。
- 高并发：mysql 单机支撑到 2000QPS 也开始容易报警了。缓存单机支撑的并发量轻松一秒几万十几万。

**用了缓存之后有什么不良后果**
- [缓存与数据库双写不一致](#redisJump1)
- [缓存雪崩、缓存穿透](#redisJump2)
- [缓存并发竞争](#redisJump3)


<span id = "redisJump1">**数据库与缓存的双写一致性问题**</span>
1.[Cache Aside Pattern](https://www.liangzl.com/get-article-detail-30109.html)
![avatar](https://oscimg.oschina.net/oscnet/3389b4d1aa144c753563b4948a1660eafc1.jpg)

2.为什么是先删除缓存再更新数据库,而不是反过来
![avatar](https://oscimg.oschina.net/oscnet/42596b40fad8ec4ed884572ae3b7c9b515a.jpg)

3.并发读写的一致性问题
![avatar](https://oscimg.oschina.net/oscnet/0175e0029981448e46341cf3d7cc9fbe93c.jpg)

4.并发读写不一致的解决方案
![avatar](https://oscimg.oschina.net/oscnet/b2018cd45ae3e19f5123addc53ad17ba753.jpg)
读请求和写请求串行化，串到一个内存队列里去，这样就可以保证一定不会出现不一致的情况. 一般来说，如果你的系统不是严格要求缓存+数据库必须一致性的话，缓存可以稍微的跟数据库偶尔有不一致的情况，最好不要做这个方案.串行化之后，就会导致系统的吞吐量会大幅度的降低，用比正常情况下多几倍的机器去支撑线上的一个请求。

<span id = "redisJump2">**缓存雪崩、缓存穿透、缓存击穿**</span>
[参考](https://baijiahao.baidu.com/s?id=1619572269435584821&wfr=spider&for=pc)
1.缓存雪崩
缓存雪崩是指在设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失效，导致所有的查询都落在数据库上，造成了缓存雪崩。举个例子来说，对于系统 A，假设每天高峰期每秒 5000 个请求，本来缓存在高峰期可以扛住每秒 4000 个请求，但是缓存机器意外发生了全盘宕机。缓存挂了，此时 1 秒 5000 个请求全部落数据库，数据库必然扛不住，它会报一下警，然后就挂了。此时，如果没有采用什么特别的方案来处理这个故障，DBA 很着急，重启数据库，但是数据库立马又被新的流量给打死了。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/redis-caching-avalanche.png)

2.缓存雪崩的事前事中事后的解决方案如下：
- 事前：redis 高可用，主从+[哨兵](#sentinelJump1)，redis cluster，避免全盘崩溃。
- 事中：本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死。
- 事后：redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

用户发送一个请求，系统 A 收到请求后，先查本地 ehcache 缓存，如果没查到再查 redis。如果 ehcache 和 redis 都没有，再查数据库，将数据库中的结果，写入 ehcache 和 redis 中。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/redis-caching-avalanche-solution.png)
//todo ehcache和hystrix相关

3.缓存穿透
缓存穿透，是指查询一个数据库一定不存在的数据。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/redis-caching-penetration.png)
解决方法：解决方式很简单，每次系统 A 从数据库中只要没查到，就写一个空值到缓存里去，比如 set -999 UNKNOWN。然后设置一个过期时间，这样的话，下次有相同的 key 来访问的时候，在缓存失效之前，都可以直接从缓存中取数据。

4.缓存击穿
缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。
解决方法：不同场景下的解决方式可如下：
- 若缓存的数据是基本不会发生更新的，则可尝试将该热点数据设置为永不过期。
- 若缓存的数据更新不频繁，且缓存刷新的整个流程耗时较少的情况下，则可以采用基于 redis、zookeeper 等分布式中间件的分布式互斥锁，或者本地互斥锁以保证仅少量的请求能请求数据库并重新构建缓存，其余线程则在锁释放后能访问到新缓存。
- 若缓存的数据更新频繁或者缓存刷新的流程耗时较长的情况下，可以利用定时线程在缓存过期前主动的重新构建缓存或者延后缓存的过期时间，以保证所有的请求能一直访问到对应的缓存。




<span id = "redisJump3">**缓存并发竞争**</span>


### Redis
#### rehash
  [渐进式 rehash 策略](http://redisbook.com/preview/dict/incremental_rehashing.html):为了避免 rehash 对服务器性能造成影响， 服务器不是一次性将 ht[0] 里面的所有键值对全部 rehash 到 ht[1] ， 而是分多次、渐进式地将 ht[0] 里面的键值对慢慢地 rehash 到 ht[1] 。
##### rehash 的详细步骤：
- 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
- 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。
- 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一。
- 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。

##### 触发扩容的条件
- 如果哈希表ht[0]中保存的key个数与哈希表大小的比例已经达到1:1，即保存的节点数已经大于哈希表大小，且redis服务当前允许执行rehash
- 或者保存的节点数与哈希表大小的比例超过了安全阈值（默认值为5）
则将哈希表大小扩容为原来的两倍
- [redis渐进式rehash机制](https://luoming1224.github.io/2018/11/12/[redis%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0]redis%E6%B8%90%E8%BF%9B%E5%BC%8Frehash%E6%9C%BA%E5%88%B6/)

##### redis的java客户端jedis
[jedis实例是非线程安全的](https://www.jianshu.com/p/5e4a1f92c88f),常常通过JedisPool连接池去管理实例，在多线程情况下让每个线程有自己独立的jedis实例。Redis所有单个命令的执行都是原子性的，这与它的单线程机制有关；
jedi实例不是线程安全的原因：

##### <span id = "sentinelJump1">redis 哨兵模式</span>
[参考](https://www.jianshu.com/p/06ab9daf921d)
主从切换技术的方法是：当主服务器宕机后，需要手动把一台从服务器切换为主服务器，这就需要人工干预，费事费力，还会造成一段时间内服务不可用。这不是一种推荐的方式，更多时候，我们优先考虑**哨兵模式。**
1.哨兵模式概念
哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例。
![avatar](https://upload-images.jianshu.io/upload_images/11320039-57a77ca2757d0924.png?imageMogr2/auto-orient/strip|imageView2/2/w/507)
- 通过发送命令，让Redis服务器返回监控其运行状态，包括主服务器和从服务器。
- 当哨兵监测到master宕机，会自动将slave切换成master，然后通过发布订阅模式通知其他的从服务器，修改配置文件，让它们切换主机。

然而一个哨兵进程对Redis服务器进行监控，可能会出现问题，为此，我们可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式。

用文字描述一下故障切换（failover）的过程。假设主服务器宕机，哨兵1先检测到这个结果，系统并不会马上进行failover过程，仅仅是哨兵1主观的认为主服务器不可用，这个现象成为**主观下线**。当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由一个哨兵发起，进行failover操作。切换成功后，就会通过发布订阅模式，让各个哨兵把自己监控的从服务器实现切换主机，这个过程称为**客观下线**。这样对于客户端而言，一切都是透明的。

2.Redis配置哨兵模式
![avatar](https://upload-images.jianshu.io/upload_images/11320039-3f40b17c0412116c.png?imageMogr2/auto-orient/strip|imageView2/2/w/747)

## Hbase
[参考一](https://zhuanlan.zhihu.com/p/30411575)
[参考二](https://zhuanlan.zhihu.com/p/30414252)
[参考三](https://www.cnblogs.com/panpanwelcome/p/8716483.html)
### 简介
Hbase是一个运行在Hadoop集群的数据库。Hbase和传统意义上的关系型数据库(RDBMS)有所不同，他牺牲了传统数据库的ACID要求，达到超大容量和更快的处理速度。并且，hbase上面存储的数据不需要严格符合表结构，它可以存储非结构化和板结构化的数据。

### 关系型数据库 vs Hbase-(数据存储模型)
**关系型数据库的优点**
- 关系型数据库提供标准持久化模型
- SQL已经成为数据操作的事实标准
- 关系型数据库支持并发事务
- 关系型数据库已经有了很多成熟工具

**缺点**：纵向扩展困难，需要购买更强大的服务器

**NoSQL能带来什么改变？**
除了纵向扩展(vertical scale)，还有一种扩展方式， 就是横向扩展(horizontal scale) . 横向扩展采购多台便宜的PC服务器组成大集群，提供服务，这种方式使用普通PC服务器，相对便宜，同时大集群也比单机服务器可靠。

- mysql:MySQL：关系型数据库，主要面向OLTP，支持事务，支持二级索引，支持sql，支持主从、Group Replication架构模型（本文全部以Innodb为例，不涉及别的存储引擎）。

- hbase: HBase：基于HDFS，支持海量数据读写（尤其是写），支持上亿行、上百万列的, **面向列** 的分布式NoSql数据库。天然分布式，主从架构，不支持事务，不支持二级索引，不支持sql。

**列存储和行存储的区别**
列式存储(Columnar or column-based)是相对于传统关系型数据库的行式存储(Row-basedstorage)来说的。简单来说两者的区别就是**如何组织表**：
![avatar](https://images2018.cnblogs.com/blog/591290/201804/591290-20180404101912975-230833010.png)
 行式存储下一张表的数据都是放在一起的，但列式存储下都被分开保存了
  存储形式 | 行式存储 | 列式存储
 :-: | :-: | :-:
 优点 | 数据被保存在一起;INSERT/UPDATE容易| 查询时只有涉及到的列会被读取;投影(projection)很高效;任何列都能作为索引
 缺点 | 选择(Selection)时即使只涉及某几列，所有数据也都会被读取 | 选择完成时，被选择的列要重新组装；INSERT/UPDATE比较麻烦

 //todo 任何列都能做索引

 ### hbase的数据模型
 HBase根据主键(row key)，分片，存储数据。HBase的主键(row key)和关系型数据库的主键(primary key)作用非常类似。 HBase上面的所有行按照主键的值排序，即主键小的行排在表的前面。HBase的数据按照主键(row key)有序排列是HBase区别于其他分布式数据库的重要特征
![avatar](https://pic3.zhimg.com/80/v2-48a0b6c84dc40e080c8146f277195552_hd.jpg)

下图中, HBase表按照主键(row key)的范围, 分成不同的区域(region). 然后把这些区域(region)分配给不同的区域服务器(region server)管理. 这种分配方式由HBase自动控制，自动调节区域服务器(region server)中的存储容量, 这样就实现了存储数据的无限扩展。
![avatar](https://pic4.zhimg.com/80/v2-4b9ea19a833cc4323829779da0a94f27_hd.jpg)

下图显示了列簇(column family)和HBase表之间的对应关系，不同的列簇(column)在物理上被存储在不同的文件中。每个列簇文件都可以被单独访问
![avatar](https://pic1.zhimg.com/80/v2-0d391fa6ddb96f88b1f383ed613afef8_hd.jpg)

数据存储在HBase表格的cell中。cell中包含key和value以及一些其他的信息（如version, type等）。其中key部分包括row key，column family，column qualifier, timestamp。并且对于每一个值，key和value和一起存储在列簇(column family)里。如下图所示:
![avatar](https://pic4.zhimg.com/80/v2-e0356a0f104da4f03585b8282f1117f7_hd.jpg)

列簇文件里面的数据按照Cell的方式存储。 每个Cell被称为一个键值对(KeyValue)。 Key包括的项目比较多，有主键(rowkey)，列簇的名称(column family), column qualifier和时间戳(timestamp)。 由这四项的组成一个Key，通过这个Key就可以在列簇文件中查询到Value.

从逻辑上看，Cell好像存储在表中的一行。但是在实际物理存储中，HBase的一行由有多个列簇（column family)的对应Cell拼接而成。

下图所示， 在上面的表显示了一个Hbase表结构， 从逻辑上看和关系型数据库类似，也是按行组织。但在物理存储中，整个表是由2个列簇文件组成。如果要修改某行中的一项（实际上是一个Cell）, 需要指定主键(rowkey)，列簇的名称(column family), column qualifier和时间戳(timestamp)形成一个Key， 然后在实际的列簇文件中设置Value。
![avatar](https://pic4.zhimg.com/80/v2-089cd3a04f7121009dcbbb227a5e8bdf_hd.jpg)

如前所述， 定位一个Cell, 只需要5项数据拼成Key: <表名称, 主键名称, 列簇名称, 列名称, 时间戳> . HBase的表是一个稀疏表。 如果一项(column)没有数据，它就不会被存储. 此外, Cell是有版本信息的，如果只用4项数据做Key, <表名称, 主键名称, 列簇名称, 列名称>可以查询到一个Cell的多个版本.

![avatar](https://pic4.zhimg.com/80/v2-19bd07f9e19ddaa88ce0d38830920b47_hd.jpg)

HBase默认支持多版本，
Put操作既是一个插入操作也可以是一个修改操作，每个Cell都有自己的版本。删除操作不会真的做删除，只是写入一个删除的标志。 任何查询都不会返回有有删除标记的Cell.
Get操作可以任意指定的版本的数据。如果请求没有指定版本号，会返回最新的版本。HBase允许用户配置最多允许多少版本同时存在，HBase默认为同一个Cell最多存储3个版本。当超过3个版本后，最老版本的Cell会被最终清理。
![avatar](https://pic4.zhimg.com/80/v2-3cd0bb4f57acbf13787742dfda022043_hd.jpg)

### 系统架构
![avatar](http://i9.hexunimg.cn/2011-10-17/134285162.jpg)
**client**
包含访问Hbase的接口，
client维护着一些cache来加快对Hbase的访问，比如regione的位置信息。

**Zookeeper**：
- 保证任何时候，集群中只有一个master
- 存贮所有Region的寻址入口。
- 实时监控Region Server的状态，将Region server的上线和下线信息实时通知给Master
- 存储Hbase的schema,包括有哪些table，每个table有哪些column family

**HMaster**:
- 为Region server分配region，负责region server的负载均衡
- 管理用户对Table的增、删、改、查操作
- 发现失效的region server并重新分配其上的region
- GFS上的垃圾文件回收
- 在HRegionServer停机后，负责失效HRegionServer 上的Regions迁移
可以看到，**client访问Hbase上数据的过程并不需要master参与**(寻址访问zookeeper和region server，数据读写访问regione server)，**master仅仅维护者table和region的元数据信息，负载很低**。







## Elasticsearch
