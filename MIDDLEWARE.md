# 中间件

## zookeeper
[官方参考文档](http://zookeeper.apache.org/doc/r3.5.4-beta/zookeeperReconfig.html#sc_reconfig_modifying)

### leader选举
Leader选举是保证分布式数据一致性的关键所在。当Zookeeper集群中的一台服务器出现以下两种情况之一时，需要进入Leader选举。
- 服务器初始化启动。
- 服务器运行期间无法和Leader保持连接。
#### 服务器启动时的leader选举
选举流程如图
![avatar](https://img-blog.csdn.net/20180128213955030?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmd5dXFpYW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1. 每个server会发起一个投票
- **每个Server发出一个投票**。由于是初始情况，ZK1和ZK2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时ZK1的投票为(1, 0)，ZK2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。
- **接受来自各个服务器的投票**。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。
- **处理投票**。针对每一个投票，服务器都需要将别人的投票和自己的投票进行比较，规则如下
a. 优先检查ZXID。ZXID比较大的服务器优先作为Leader。
b. 如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器。
　　对于ZK1而言，它的投票是(1, 0)，接收ZK2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时ZK2的myid最大，于是ZK2胜。ZK1更新自己的投票为(2, 0)，并将投票重新发送给ZK2。
- **统计投票**。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于ZK1、ZK2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出ZK2作为Leader。
- **改变服务器状态**。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。当新的Zookeeper节点ZK3启动时，发现已经有Leader了，不再选举，直接将直接的状态从LOOKING改为FOLLOWING。

#### 运行期间Leader重新选举
选举流程如图
![avatar](https://img-blog.csdn.net/20180128214003138?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmd5dXFpYW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1. 变更状态。Leader挂后，余下的非Observer服务器都会讲自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程。
2. 每个Server会发出一个投票。在运行期间，每个服务器上的ZXID可能不同，此时假定ZK1的ZXID为124，ZK3的ZXID为123；在第一轮投票中，ZK1和ZK3都会投自己，产生投票(1, 124)，(3, 123)，然后各自将投票发送给集群中所有机器。
3. 接收来自各个服务器的投票。与启动时过程相同。
4. 处理投票。与启动时过程相同，由于ZK1事务ID大，ZK1将会成为Leader。
5. 统计投票。与启动时过程相同。
6. 改变服务器的状态。与启动时过程相同。

### leader 选举算法分析
zk为leader选取提供了三种算法，分别是leaderElection，UDP版本的FastElection和TCP版本的FastElection。从3.4.0版本开始，zk只保留了TCP版本的FastElection,下面介绍的就是易tcp版本的快速选举流程分析。

#### 选取算法分析
zk集群会在两种情况下进入leader选举：服务器初始化启动和服务器运行期间无法与leader联系。当服务器进入选举流程时，集群可能存在两种状态：集群中本来就存在一个leader,集群中确实不存在leader。针对于第一种情况，通常是这台服务器启动较晚造成的，这时当它试图去选举leader的时候会被告知leader已经存在，对于该机器来说，仅仅需要与leader连接，并同步状态即可。当集群内没有leader时，流程如下：
**开始第一次投票**
服务器启动或者zk的leader节点挂了之后，集群处于looking状态。当一台服务器处于looking状态时，他会向集群中其他机器发送投票消息，消息内容包括SID和ZXID。第一次投票的时候，每台机器将自己作为被推举的对象进行投票。
**变更投票**
变更投票的规则是：
- 如果vote_zxid大于self_zxid，就认可当前收到的投票，并再次将该投票发送出去。
- 如果vote_zxid小于self_zxid，就坚持自己的投票，不做任何更改。
- 如果vote_zxid等于self_zxid，那么就比较两者的SID,如果vote_sid大于self_sid,那就认可当前接受到的投票，并再次将该投票发送出去。
- 如果vote_zxid等于self_zxid，并且vote_sid小于self_sid,那么就坚持自己的投票，不做变更。
结合上面规则，给出下面的集群变更过程。
![avatar](https://upload-images.jianshu.io/upload_images/4728488-9e0b653137cccbc7.png)

**确定leader**
经过第二轮投票后，集群中的每台机器都会再次接收到其他机器的投票，然后开始统计投票，如果一台机器收到了超过半数的相同投票，那么这个投票对应的SID机器即为Leader。此时Server3将成为Leader。

### zk对于选举算法的实现细节
**1. 服务器的四种状态**
LOOKING,LEADEING,FOLLOWING,OBSERVING。

**2. 投票的数据结构**
*id*：被推举的Leader的SID。
*zxid*：被推举的Leader事务ID。
*electionEpoch*：逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列，每次进入新一轮的投票后，都会对该值进行加1操作。
*peerEpoch*：被推举的Leader的epoch。
*state*：当前服务器的状态。

**3.QuorumCnxManager：网络I/O**
　　每台服务器在启动的过程中，会启动一个QuorumPeerManager，负责各台服务器之间的底层Leader选举过程中的网络通信。
(1)消息队列
QuorumCnxManager内部维护了一系列的队列，用来保存接收到的、待发送的消息以及消息的发送器，**除接收队列以外，其他队列都按照SID分组形成队列集合**，如一个集群中除了自身还有3台机器，那么就会为这3台机器分别创建一个发送队列，互不干扰。
- recvQueue:消息接收队列，用于存放那些从其他服务器接收到的消息。
- queueSendMap:消息发送队列，用于保存那些待发送的消息，按照SID进行分组。
- 发送器集合，每个SenderWorker消息发送器，都对应一台远程Zookeeper服务器，负责消息的发送，也按照SID进行分组。
- lastMessageSent：最近发送过的消息，为每个SID保留最近发送过的一个消息。

(2)建立连接
为了能够相互投票，Zookeeper集群中的所有机器都需要两两建立起网络连接。QuorumCnxManager在启动时会创建一个ServerSocket来监听Leader选举的通信端口(默认为3888)。开启监听后，Zookeeper能够不断地接收到来自其他服务器的创建连接请求，在接收到其他服务器的TCP连接请求时，会进行处理。为了避免两台机器之间重复地创建TCP连接，Zookeeper只允许SID大的服务器主动和其他机器建立连接，否则断开连接。在接收到创建连接请求后，服务器通过对比自己和远程服务器的SID值来判断是否接收连接请求，如果当前服务器发现自己的SID更大，那么会断开当前连接，然后自己主动和远程服务器建立连接。一旦连接建立，就会根据远程服务器的SID来创建相应的消息发送器SendWorker和消息接收器RecvWorker，并启动。

(3) 消息的接受与发送
- 消息接收：由消息接收器RecvWorker负责，由于Zookeeper为每个远程服务器都分配一个单独的RecvWorker，因此，每个RecvWorker只需要不断地从这个TCP连接中读取消息，并将其保存到recvQueue队列中。
- 消息发送：由于Zookeeper为每个远程服务器都分配一个单独的SendWorker，因此，每个SendWorker只需要不断地从对应的消息发送队列中获取出一个消息发送即可，同时将这个消息放入lastMessageSent中。在SendWorker中，一旦Zookeeper发现针对当前服务器的消息发送队列为空，那么此时需要从lastMessageSent中取出一个最近发送过的消息来进行再次发送，这是为了解决接收方在消息接收前或者接收到消息后服务器挂了，导致消息尚未被正确处理。同时，Zookeeper能够保证接收方在处理消息时，会对重复消息进行正确的处理。

**4. 选举算法的核心**
- 外部投票：特指其他服务器发来的投票。
- 内部投票：服务器自身当前的投票。
- 选举轮次：Zookeeper服务器Leader选举的轮次，即logicalclock。
- PK：对内部投票和外部投票进行对比来确定是否需要变更内部投票。
(1)选票管理
- sendqueue：选票发送队列，用于保存待发送的选票。
- recvqueue：选票接收队列，用于保存接收到的外部投票。
- WorkerReceiver：选票接收器。其会不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到recvqueue中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票。
- WorkerSender：选票发送器，不断地从sendqueue中获取待发送的选票，并将其传递到底层QuorumCnxManager中。

(2)算法核心
![avatar](https://upload-images.jianshu.io/upload_images/4728488-5dc4118fed8d3152.png)
![avatar](http://note.youdao.com/yws/public/resource/1f840315a7019698a1e3140e76bff42b/xmlnote/B42F0F7048EA4E729FE15DA94D941772/16819)
![avatar](https://img2018.cnblogs.com/blog/1496926/201910/1496926-20191004181511814-142904818.png)

1. 自增选举轮次。Zookeeper规定所有有效的投票都必须在同一轮次中，在开始新一轮投票时，会首先对logicalclock进行自增操作。

2. 初始化选票。在开始进行新一轮投票之前，每个服务器都会初始化自身的选票，并且在初始化阶段，每台服务器都会将自己推举为Leader。

3. 发送初始化选票。完成选票的初始化后，服务器就会发起第一次投票。Zookeeper会将刚刚初始化好的选票放入sendqueue中，由发送器WorkerSender负责发送出去。

4. 接收外部投票。每台服务器会不断地从recvqueue队列中获取外部选票。如果服务器发现无法获取到任何外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效的连接，如果没有连接，则马上建立连接，如果已经建立了连接，则再次发送自己当前的内部投票。

5. 判断选举轮次。在发送完初始化选票之后，接着开始处理外部投票。在处理外部投票时，会根据选举轮次来进行不同的处理。

- 外部投票的选举轮次大于内部投票。若服务器自身的选举轮次落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次(logicalclock)，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票。最终再将内部投票发送出去。
- 外部投票的选举轮次小于内部投票。若服务器接收的外选票的选举轮次落后于自身的选举轮次，那么Zookeeper就会直接忽略该外部投票，不做任何处理，并返回步骤4。
- 外部投票的选举轮次等于内部投票。此时可以开始进行选票PK。

6. 选票PK。在进行选票PK时，符合任意一个条件就需要变更投票。

- 若外部投票中推举的Leader服务器的选举轮次大于内部投票，那么需要变更投票。

- 若选举轮次一致，那么就对比两者的ZXID，若外部投票的ZXID大，那么需要变更投票。

- 若两者的ZXID一致，那么就对比两者的SID，若外部投票的SID大，那么就需要变更投票。

7. 变更投票。经过PK后，若确定了外部投票优于内部投票，那么就变更投票，即使用外部投票的选票信息来覆盖内部投票，变更完成后，再次将这个变更后的内部投票发送出去。

8. 选票归档。无论是否变更了投票，都会将刚刚收到的那份外部投票放入选票集合recvset中进行归档。recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票（按照服务队的SID区别，如{(1, vote1), (2, vote2)…}）。

9. 统计投票。完成选票归档后，就可以开始统计投票，统计投票是为了统计集群中是否已经有过半的服务器认可了当前的内部投票，如果确定已经有过半服务器认可了该投票，则终止投票。否则返回步骤4。

10. 更新服务器状态。若已经确定可以终止投票，那么就开始更新服务器状态，服务器首选判断当前被过半服务器认可的投票所对应的Leader服务器是否是自己，若是自己，则将自己的服务器状态更新为LEADING，若不是，则根据具体情况来确定自己是FOLLOWING或是OBSERVING。

以上10个步骤就是FastLeaderElection的核心，其中步骤4-9会经过几轮循环，直到有Leader选举产生。
![avatar](https://img-blog.csdn.net/20180228151845627)

### zk的数据与存储
在zk中，数据存储分为两个部分：内存数据存储和磁盘数据存储。其中磁盘数据又可以分为快照和事务日志。

#### 内存数据
zk的数据模式是一个树结构，在内存数据库中，存储了整个书结构的内容，包括所有的节点路径、节点数据以及ACL信息，zk会定时将这个数据存储到磁盘上。

**DateTree**
树结构的数据结构
![avatar](https://upload-images.jianshu.io/upload_images/11345047-6983f333cfce1e51.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

**DataNode**
dataNode是数据存储的最小单元，数据结构如上图所示，dataNode内部除了保存了节点的数据内容（data[]）、ACL列表（acl）和节点状态（state）,还记录了父节点的饮用和子节点列表两个属性。

**zk的database**
Zookeeper的内存数据库，负责管理Zookeeper的所有会话，DataTree存储和事务日志。它会定时向磁盘dump快照数据（snapCount主要控制），服务器启动时，会通过磁盘上的事务日志和快照数据文件恢复成完整的内存数据库。

### 磁盘文件存储
[参考](https://www.jianshu.com/p/220cd7f0382f)
zookeeper总共提供了两种持久化文件，分别是内存快照SnapShot和事务日
- FileTxnLog：事务日志文件，以日志追加的方式维护。这种日志有点类似MySQL数据库中的binlog，zookeeper会把所有涉及到修改内存数据结构的操作日志记录到该log中，也就是说，zookeeper会把每一个事务操作诸如添加、删除都会记录到这个日志文件中，当zookeeper出现异常时，可以借助该事务日志进行数据恢复。日志文件它主要是负责实时记录对服务端的每一次的事务操作日志
- FileSnap：内存快照文件，保存内存数据库某一时刻的状态，所以数据不一定是最新的。

zk服务其启动的时候，首先会进行数据初始化，将磁盘中数据，加载到内存中，恢复现场。具体流程如下图所示
![avatar](https://upload-images.jianshu.io/upload_images/11345047-8246f433dba2a103.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

### 数据同步
zk的同步通常分为四种：直接差异化同步（diff同步），先回滚在差异化同步（trunc+diff同步），仅回滚同步（trunc同步）和全量同步（snap同步）

ZK 集群服务器启动之后，会进行 2 个动作：
- 选举 Leader：分配角色
- Learner 向 Leader 服务器注册：数据同步。 数据同步，本质：将没有在 Learner 上执行的事务，同步给 Learner。
数据同步的流程如下：
![avatar](https://upload-images.jianshu.io/upload_images/11345047-74e172eae7ad8454.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)
**1.获取learner状态**
在注册learner的最后阶段，learner服务器会发送给Leader服务器一个ACKEPOCH数据包，Leader会从这个数据包中解析出该leaner的currentEpoch和lastZxid。

**2.数据同步初始化**
首先从Zookeeper内存数据库中提取出事务请求对应的提议缓存队列proposals，同时完成peerLastZxid(该Learner最后处理的ZXID)、minCommittedLog(Leader提议缓存队列commitedLog中最小的ZXID)、maxCommittedLog(Leader提议缓存队列commitedLog中的最大ZXID)三个ZXID值的初始化。
- 直接差异化同步(DIFF同步，peerLastZxid介于minCommittedLog和maxCommittedLog之间)。Leader首先向这个Learner发送一个DIFF指令，用于通知Learner进入差异化数据同步阶段，Leader即将把一些Proposal同步给自己，针对每个Proposal，Leader都会通过发送PROPOSAL内容数据包和COMMIT指令数据包来完成，
- 先回滚再差异化同步(TRUNC+DIFF同步，Leader已经将事务记录到本地事务日志中，但是没有成功发起Proposal流程)。当Leader发现某个Learner包含了一条自己没有的事务记录，那么就需要该Learner进行事务回滚，回滚到Leader服务器上存在的，同时也是最接近于peerLastZxid的ZXID。
- 仅回滚同步(TRUNC同步，peerLastZxid大于maxCommittedLog)。Leader要求Learner回滚到ZXID值为maxCommittedLog对应的事务操作。
- 全量同步(SNAP同步，peerLastZxid小于minCommittedLog或peerLastZxid不等于lastProcessedZxid)。Leader无法直接使用提议缓存队列和Learner进行同步，因此只能进行全量同步。Leader将本机的全量内存数据同步给Learner。Leader首先向Learner发送一个SNAP指令，通知Learner即将进行全量同步，随后，Leader会从内存数据库中获取到全量的数据节点和会话超时时间记录器，将他们序列化后传输给Learner。Learner接收到该全量数据后，会对其反序列化后载入到内存数据库中。

### zookeerper一致性协议：Zab协议
[参考](https://www.jianshu.com/p/2bceacd60b8a)

#### 什么是zab协议
Zab协议 的全称是 Zookeeper Atomic Broadcast （Zookeeper原子广播）。Zookeeper 是通过 Zab 协议来保证分布式事务的最终一致性。
- zab协议是为分布式协调服务zookeeper专门设计的一种**支持崩溃恢复**的**原子广播协议**，是Zookeeper保证数据一致性的核心算法。zab借鉴了paxos算法，但又不像paxos那样，是一种通用的分布式一致性算法。**它是特别为zookeeper设计的支持崩溃的原子广播协议。**
- 在Zookeeper中主要依赖Zab协议来实现数据一致性，基于该协议，zk实现了一种主备模型（即Leader和Follower模型）的系统架构来保证集群中各个副本之间数据的一致性。这里的主备系统架构模型，就是指只有一台客户端（Leader）负责处理外部的写事务请求，然后Leader客户端将数据同步到其他Follower节点。

Zookeeper 客户端会随机的链接到 zookeeper 集群中的一个节点，如果是读请求，就直接从当前节点中读取数据；如果是写请求，那么节点就会向 Leader 提交事务，Leader 接收到事务提交，会广播该事务，只要超过半数节点写入成功，该事务就会被提交。

#### zab协议的特性
- **zab协议确保那些已经在leader服务器上提交（commit）的事务最终被所有的服务器提交。**
- Zab 协议需要确保**丢弃那些只在 Leader 上被提出而没有被提交的事务**。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-d32b630b65a7a0b2.png?imageMogr2/auto-orient/strip|imageView2/2/w/470)

#### Zab协议实现的作用
1）使用一个单一的主进程（Leader）来接收并处理客户端的事务请求（也就是写请求），并采用了Zab的原子广播协议，将服务器数据的状态变更以 事务proposal （事务提议）的形式广播到所有的副本（Follower）进程上去。
2）保证一个全局的变更序列被顺序引用。
Zookeeper是一个树形结构，很多操作都要先检查才能确定是否可以执行，比如P1的事务t1可能是创建节点"/a"，t2可能是创建节点"/a/bb"，只有先创建了父节点"/a"，才能创建子节点"/a/b"。为了保证这一点，Zab要保证同一个Leader发起的事务要按顺序被apply，同时还要保证只有先前Leader的事务被apply之后，新选举出来的Leader才能再次发起事务。
3）当主进程出现异常的时候，整个zk集群依旧能正常工作。

#### zab协议原理
Zab协议要求每个 Leader 都要经历三个阶段：发现，同步，广播。
- 发现：要求zookeeper集群必须选举出一个 Leader 进程，同时 Leader 会维护一个 Follower 可用客户端列表。将来客户端可以和这些 Follower节点进行通信。
- 同步：Leader 要负责将本身的数据与 Follower 完成同步，做到多副本存储。这样也是提现了CAP中的高可用和分区容错。Follower将队列中未处理完的请求消费完成后，写入本地事务日志中。
- 广播：Leader 可以接受客户端新的事务Proposal请求，将新的Proposal请求广播给所有的 Follower。

#### zab协议的内容
Zab 协议包括两种基本的模式：崩溃恢复 和 消息广播
**协议过程**
- 当整个集群启动过程中，或者当 Leader 服务器出现网络中弄断、崩溃退出或重启等异常时，Zab协议就会 进入崩溃恢复模式，选举产生新的Leader。
- 当选举产生了新的 Leader，同时集群中有过半的机器与该 Leader 服务器完成了状态同步（即数据同步）之后，Zab协议就会退出崩溃恢复模式，进入消息广播模式。
- 这时，如果有一台遵守Zab协议的服务器加入集群，因为此时集群中已经存在一个Leader服务器在广播消息，那么该新加入的服务器自动进入恢复模式：找到Leader服务器，并且完成数据同步。同步完成后，作为新的Follower一起参与到消息广播流程中。

**协议状态切换**
当Leader出现崩溃退出或者机器重启，亦或是集群中不存在超过半数的服务器与Leader保存正常通信，Zab就会再一次进入崩溃恢复，发起新一轮Leader选举并实现数据同步。同步完成后又会进入消息广播模式，接收事务请求。

**保证消息有序**
在整个消息广播中，Leader会将每一个事务请求转换成对应的 proposal 来进行广播，并且在广播 事务Proposal 之前，Leader服务器会首先为这个事务Proposal分配一个全局单递增的唯一ID，称之为事务ID（即zxid），由于Zab协议需要保证每一个消息的严格的顺序关系，因此必须将每一个proposal按照其zxid的先后顺序进行排序和处理。
//**todo zk全局单递增唯一id生成策略。**


**zookeeper是如何保证事务的顺序一致性的？**
zookeeper采用了递增的事务Id来标识，所有的proposal都在被提出的时候加上了zxid，zxid实际上是一个64位的数字，高32位是epoch用来标识leader是否发生改变，如果有新的leader产生出来，epoch会自增，低32位用来递增计数。当新产生proposal的时候，会依据数据库的两阶段过程，首先会向其他的server发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么就会开始执行。

##### 消息广播
1.在zookeeper集群中，数据副本的传递策略就是采用消息广播模式。zookeeper中农数据副本的同步方式与二段提交相似，但是却又不同。二段提交要求协调者必须等到所有的参与者全部反馈ACK确认消息后，再发送commit消息。要求所有的参与者要么全部成功，要么全部失败。二段提交会产生严重的阻塞问题。
2.Zab协议中 Leader 等待 Follower 的ACK反馈消息是指“只要半数以上的Follower成功反馈即可，不需要收到全部Follower反馈”。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-447433fdf7a1d7d6.png?imageMogr2/auto-orient/strip|imageView2/2/w/555)

**消息广播具体步骤**
1）客户端发起一个写操作请求。

2）Leader 服务器将客户端的请求转化为事务 Proposal 提案，同时为每个 Proposal 分配一个全局的ID，即zxid。

3）Leader 服务器为每个 Follower 服务器分配一个单独的队列，然后将需要广播的 Proposal 依次放到队列中取，并且根据 FIFO 策略进行消息发送。

4）Follower 接收到 Proposal 后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后向 Leader 反馈一个 Ack 响应消息。

5）Leader 接收到超过半数以上 Follower 的 Ack 响应消息后，即认为消息发送成功，可以发送 commit 消息。

6）Leader 向所有 Follower 广播 commit 消息，同时自身也会完成事务提交。Follower 接收到 commit 消息后，会将上一条事务提交。

##### 崩溃恢复
zookeeper 采用 Zab 协议的核心，就是只要有一台服务器提交了 Proposal，就要确保所有的服务器最终都能正确提交 Proposal。这也是 CAP/BASE 实现最终一致性的一个体现。

Leader 服务器与每一个 Follower 服务器之间都维护了一个单独的 FIFO 消息队列进行收发消息，使用队列消息可以做到异步解耦。 Leader 和 Follower 之间只需要往队列中发消息即可。如果使用同步的方式会引起阻塞，性能要下降很多。

一旦 Leader 服务器出现崩溃或者由于网络原因导致 Leader 服务器失去了与过半 Follower 的联系，那么就会进入崩溃恢复模式。

在 Zab 协议中，为了保证程序的正确运行，整个恢复过程结束后需要选举出一个新的 Leader 服务器。因此 Zab 协议需要一个高效且可靠的 Leader 选举算法，从而确保能够快速选举出新的 Leader 。

Leader 选举算法不仅仅需要让 Leader 自己知道自己已经被选举为 Leader ，同时还需要让集群中的所有其他机器也能够快速感知到选举产生的新 Leader 服务器。

**崩溃恢复主要包括两部分：Leader选举 和 数据恢复**

##### Zab 协议如何保证数据一致性
假设两种异常情况：
1、一个事务在 Leader 上提交了，并且过半的 Folower 都响应 Ack 了，但是 Leader 在 Commit 消息发出之前挂了。
2、假设一个事务在 Leader 提出之后，Leader 挂了。

要确保如果发生上述两种情况，数据还能保持一致性，那么 Zab 协议选举算法必须满足以下要求：

Zab 协议崩溃恢复要求满足以下两个要求：
1）确保已经被 Leader 提交的 Proposal 必须最终被所有的 Follower 服务器提交。
2）确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal。

根据上述要求
Zab协议需要保证选举出来的Leader需要满足以下条件：
1）新选举出来的 Leader 不能包含未提交的 Proposal 。
即新选举的 Leader 必须都是已经提交了 Proposal 的 Follower 服务器节点。
2）新选举的 Leader 节点中含有最大的 zxid 。
这样做的好处是可以避免 Leader 服务器检查 Proposal 的提交和丢弃工作。

##### zab如何做数据同步
1）完成 Leader 选举后（新的 Leader 具有最高的zxid），在正式开始工作之前（接收事务请求，然后提出新的 Proposal），Leader 服务器会首先确认事务日志中的所有的 Proposal 是否已经被集群中过半的服务器 Commit。

2）Leader 服务器需要确保所有的 Follower 服务器能够接收到每一条事务的 Proposal ，并且能将所有已经提交的事务 Proposal 应用到内存数据中。等到 Follower 将所有尚未同步的事务 Proposal 都从 Leader 服务器上同步过啦并且应用到内存数据中以后，Leader 才会把该 Follower 加入到真正可用的 Follower 列表中。

##### Zab 数据同步过程中，如何处理需要丢弃的 Proposal

在 Zab 的事务编号 zxid 设计中，zxid是一个64位的数字。

其中低32位可以看成一个简单的单增计数器，针对客户端每一个事务请求，Leader 在产生新的 Proposal 事务时，都会对该计数器加1。而高32位则代表了 Leader 周期的 epoch 编号。

    epoch 编号可以理解为当前集群所处的年代，或者周期。每次Leader变更之后都会在 epoch 的基础上加1，这样旧的 Leader 崩溃恢复之后，其他Follower 也不会听它的了，因为 Follower 只服从epoch最高的 Leader 命令。

每当选举产生一个新的 Leader ，就会从这个 Leader 服务器上取出本地事务日志充最大编号 Proposal 的 zxid，并从 zxid 中解析得到对应的 epoch 编号，然后再对其加1，之后该编号就作为新的 epoch 值，并将低32位数字归零，由0开始重新生成zxid。

Zab 协议通过 epoch 编号来区分 Leader 变化周期，能够有效避免不同的 Leader 错误的使用了相同的 zxid 编号提出了不一样的 Proposal 的异常情况。

基于以上策略
当一个包含了上一个 Leader 周期中尚未提交过的事务 Proposal 的服务器启动时，当这台机器加入集群中，以 Follower 角色连上 Leader 服务器后，Leader 服务器会根据自己服务器上最后提交的 Proposal 来和 Follower 服务器的 Proposal 进行比对，比对的结果肯定是 Leader 要求 Follower 进行一个回退操作，回退到一个确实已经被集群中过半机器 Commit 的最新 Proposal。

#### zab的四个阶段

**1、选举阶段（Leader Election）**
节点在一开始都处于选举节点，只要有一个节点得到超过半数节点的票数，它就可以当选准 Leader，只有到达第三个阶段（也就是同步阶段），这个准 Leader 才会成为真正的 Leader。

Zookeeper 规定所有有效的投票都必须在同一个 轮次 中，每个服务器在开始新一轮投票时，都会对自己维护的 logicalClock 进行自增操作。

每个服务器在广播自己的选票前，会将自己的投票箱（recvset）清空。该投票箱记录了所受到的选票。
例如：Server_2 投票给 Server_3，Server_3 投票给 Server_1，则Server_1的投票箱为(2,3)、(3,1)、(1,1)。（每个服务器都会默认给自己投票）

前一个数字表示投票者，后一个数字表示被选举者。票箱中只会记录每一个投票者的最后一次投票记录，如果投票者更新自己的选票，则其他服务器收到该新选票后会在自己的票箱中更新该服务器的选票。

这一阶段的目的就是为了选出一个准 Leader ，然后进入下一个阶段。
协议并没有规定详细的选举算法，后面会提到实现中使用的 Fast Leader Election。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-cb0c776cef667fcb.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)

**2、发现阶段（Descovery）**
在这个阶段，Followers 和上一轮选举出的准 Leader 进行通信，同步 Followers 最近接收的事务 Proposal 。
一个 Follower 只会连接一个 Leader，如果一个 Follower 节点认为另一个 Follower 节点，则会在尝试连接时被拒绝。被拒绝之后，该节点就会进入 Leader Election阶段。

这个阶段的主要目的是发现当前大多数节点接收的最新 Proposal，并且准 Leader 生成新的 epoch ，让 Followers 接收，更新它们的 acceptedEpoch。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-c75701e220688a8e.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)

**3、同步阶段（Synchronization）**
同步阶段主要是利用 Leader 前一阶段获得的最新 Proposal 历史，同步集群中所有的副本。
只有当 quorum（超过半数的节点） 都同步完成，准 Leader 才会成为真正的 Leader。Follower 只会接收 zxid 比自己 lastZxid 大的 Proposal。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-594a86e8224affba.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)

**4、广播阶段（Broadcast）**
到了这个阶段，Zookeeper 集群才能正式对外提供事务服务，并且 Leader 可以进行消息广播。同时，如果有新的节点加入，还需要对新节点进行同步。
需要注意的是，Zab 提交事务并不像 2PC 一样需要全部 Follower 都 Ack，只需要得到 quorum（超过半数的节点）的Ack 就可以。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-6c9e4297627e4570.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)

#### 协议实现
协议的 Java 版本实现跟上面的定义略有不同，选举阶段使用的是 Fast Leader Election（FLE），它包含了步骤1的发现指责。因为FLE会选举拥有最新提议的历史节点作为 Leader，这样就省去了发现最新提议的步骤。

实际的实现将发现和同步阶段合并为 Recovery Phase（恢复阶段），所以，Zab 的实现实际上有三个阶段。
Zab协议三个阶段：

1）选举（Fast Leader Election）
2）恢复（Recovery Phase）
3）广播（Broadcast Phase）

Fast Leader Election（快速选举）
前面提到的 FLE 会选举拥有最新Proposal history （lastZxid最大）的节点作为 Leader，这样就省去了发现最新提议的步骤。这是基于拥有最新提议的节点也拥有最新的提交记录

成为 Leader 的条件：
  1）选 epoch 最大的
  2）若 epoch 相等，选 zxid 最大的
  3）若 epoch 和 zxid 相等，选择 server_id 最大的（zoo.cfg中的myid）

节点在选举开始时，都默认投票给自己，当接收其他节点的选票时，会根据上面的 Leader条件 判断并且更改自己的选票，然后重新发送选票给其他节点。当有一个节点的得票超过半数，该节点会设置自己的状态为 Leading ，其他节点会设置自己的状态为 Following。
![avatar](https://upload-images.jianshu.io/upload_images/1053629-75683fa04d349414.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)

Recovery Phase（恢复阶段）
这一阶段 Follower 发送他们的 lastZxid 给 Leader，Leader 根据 lastZxid 决定如何同步数据。这里的实现跟前面的 Phase 2 有所不同：Follower 收到 TRUNC 指令会终止 L.lastCommitedZxid 之后的 Proposal ，收到 DIFF 指令会接收新的 Proposal。
history.lastCommitedZxid：最近被提交的 Proposal zxid
history.oldThreshold：被认为已经太旧的已经提交的 Proposal zxid
![avatar](https://upload-images.jianshu.io/upload_images/1053629-613f99cec1c34e2e.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)

### zk的使用场景
zk能够很好的保证分布式环境中数据的一致性。所以用来解决分布式一致性的利器。

#### 数据发布/订阅
发布者将数据发布到zk的一个或者一系列节点上，供订阅着进行数据订阅，进而达到动态获取数据的目的，实现配置信息的集中式管理和数据的动态更新。

发布订阅系统一般有两种设计模式，分别是推模式和拉模式。
- 推模式：服务器主动将数据更新发送给所订阅的客户端；
- 拉模式：客户端主动发起请求获取最新的数据，通常客户端都采用定时轮训的拉取方式。
- 推拉结合的方式(zk)：客户端向服务端注册自己需要关注的节点，一旦该节点的数据发生变更，那么服务端就会向相应的客户端发送watcher事件通知，客户端接受到这个消息通知后，需要向服务端主动获取最新的数据。


**分布式协调**
这个其实是 zookeeper 很经典的一个用法，简单来说，就好比，你 A 系统发送个请求到 mq，然后 B 系统消息消费之后处理了。那 A 系统如何知道 B 系统的处理结果？用 zookeeper 就可以实现分布式系统之间的协调工作。A 系统发送请求之后可以在 zookeeper 上对某个节点的值注册个监听器，一旦 B 系统处理完了就修改 zookeeper 那个节点的值，A 系统立马就可以收到通知，完美解决。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/zookeeper-distributed-coordination.png)
**分布式锁**
举个栗子。对某一个数据连续发出两个修改操作，两台机器同时收到了请求，但是只能一台机器先执行完另外一个机器再执行。那么此时就可以使用 zookeeper 分布式锁，一个机器接收到了请求之后先获取 zookeeper 上的一把分布式锁，就是可以去创建一个 znode，接着执行操作；然后另外一个机器也尝试去创建那个 znode，结果发现自己创建不了，因为被别人创建了，那只能等着，等第一个机器执行完了自己再执行。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/zookeeper-distributed-lock-demo.png)
**元数据/配置信息管理**
zookeeper 可以用作很多系统的配置信息的管理，比如 kafka、storm 等等很多分布式系统都会选用 zookeeper 来做一些元数据、配置信息的管理，包括 dubbo 注册中心不也支持 zookeeper 么？
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/zookeeper-meta-data-manage.png)
**HA高可用性**
这个应该是很常见的，比如 hadoop、hdfs、yarn 等很多大数据系统，都选择基于 zookeeper 来开发 HA 高可用机制，就是一个重要进程一般会做主备两个，主进程挂了立马通过 zookeeper 感知到切换到备用进程。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/zookeeper-active-standby.png)


## ETCD

### ETCD简介
etcd是一个高可用的键值存储系统，主要用于共享配置和服务发现。它的目标是构建一个高可用的分布式键值(key-value)数据库。etcd是由CoreOS开发并维护的，灵感来自于 ZooKeeper 和 Doozer，它使用Go语言编写，并通过Raft一致性算法处理日志复制以保证强一致性。Raft是一个来自Stanford的新的一致性算法，适用于分布式系统的日志复制，Raft通过选举的方式来实现一致性，在Raft中，任何一个节点都可能成为Leader。

**etcd是分布式键值数据库，那么和Redis有什么区别？**
etcd的红火来源于kurbernetes用etcd做服务发现，而redis的兴起则来源于memcache缓存本身的局限性。

- etcd是一种分布式存储，更强调的是各个节点之间的通信，同步，确保各个节点上数据和事务的一致性，使得服务发现工作更稳定，本身单节点的写入能力并不强。
- redis更像是内存型缓存，虽然也有cluster做主从同步和读写分离，但节点间的一致性主要强调的是数据，并不在乎事务，因此读写能力很强，qps甚至可以达到10万+
- 两者都是k-v存储，但redis支持更多的存储模式，包括KEY，STRING，HMAP，SET，SORTEDSET等等，因此redis本身就可以完成一些比如排序的简单逻辑。而etcd则支持对key的版本记录和txn操作和client对key的watch，因此适合用做服务发现。
- 日常使用中，etcd主要还是做一些事务管理类的，基础架构服务用的比较多，容器类的服务部署是其主流。而redis广泛地使用在缓存服务器方面，用作mysql的缓存，通常依据请求量，甚至会做成多级缓存，当然部分情况下也用做存储型redis做持续化存储。

**etcd与zookeeper都可以用做服务发现，zk和etcd的区别是什么？**
zk和etcd都是符合cp的原则 两者的区别可以[参考这里](https://cloud.tencent.com/developer/article/1138664)
**简单来说**
zk的优点：
- 非阻塞完整快照（保证最终一致性）。
- 高效的内存管理。
- 可靠（已经存在了很长时间）。
- 简化的 API。
- 自动 ZooKeeper 连接管理与重试。
- 完整的，经过充分测试的 ZooKeeper recipes 实现。
- 一个可以更容易地编写新的 ZooKeeper Recipes 的框架。
- 通过 ZooKeeper watches 支持事件。
- 在网络分区中，少数和多数分区都将进行 leader 选举。因此，少数分区将停止运行。你可以在这里阅读更多。

zk的缺点：
- 由于 ZooKeeper 是用Java编写的，它继承了Java 的一些缺点（即垃圾收集导致暂停）。
- 创建快照时（将数据写入磁盘），ZooKeeper 读取/写入操作暂时中止。只有我们启用了快照才会发生这种情况。否则，ZooKeeper 将作为内存中的分布式存储运行。
- 对于每个新的 watch 请求，ZooKeeper 都会打开一个新的 socket 连接。这使得 ZooKeeper 更加复杂，因为它必须实时管理大量打开的 socket 连接。


etcd的优点：

- 创建快照时用增量快照避免了暂停，这在 ZooKeeper 中是个问题。
- 由于使用堆外存储，所以没有垃圾回收导致的暂停。
- Watches 被重新设计，将旧的事件模型替换为在关键时间间隔内持久监听和事件复用的模型。因此，不会为每个 watch 分配一个 socket 连接。它在相同的连接上复用事件。
- 与 ZooKeeper 不同，etcd 可以使用当前的 watch 持续监听。一旦 watch 被触发，就不需要设置一个新的 watch。
- etcd3 拥有一个滑动窗口来保留旧事件，以便断开连接不会导致所有事件丢失。

etcd的缺点：
- 请注意，如果客户端超时或者客户端与 etcd 成员之间出现网络中断，客户端的运行状态可能不确定。当有 leader 选举时，etcd 也可能会中断运行。在此事件中，etcd 不会发送终止响应给客户端的未决请求。
- 网络分区时，如果 leader 在少数分区中，序列化的读取请求仍然可以处理。


### etcd的使用场景
[经典使用场景](https://blog.csdn.net/bbwangj/article/details/82584988)
*A highly-available key value store for shared configuration and service discovery.*

很多人第一反应可能是一个键值存储仓库，却没有重视官方定义的后半句，用于配置共享和服务发现。

实际上，etcd作为一个受到ZooKeeper与doozer启发而催生的项目，除了拥有与之类似的功能外，更专注于以下四点。
- 简单：基于HTTP+JSON的API让你用curl就可以轻松使用。
- 安全：可选SSL客户认证机制。
- 快速：每个实例每秒支持一千次写操作。
- 可信：使用Raft算法充分实现了分布式。

**用etcd的场景默认处理的数据都是控制数据，对于应用数据，只推荐数据量很小，但是更新访问频繁的情况。**

**1.场景一：服务发现（Service Discovery）**
要解决服务发现的问题，需要有下面三大支柱，缺一不可。
- 一个强一致性、高可用的服务存储目录：基于 Raft 算法的 etcd 天生就是这样一个强一致性高可用的服务存储目录。
- 一种注册服务和监控服务健康状态的机制:用户可以在 etcd 中注册服务，并且对注册的服务设置key TTL，定时保持服务的心跳以达到监控健康状态的效果。
- 一种查找和连接服务的机制：通过在 etcd 指定的主题下注册的服务也能在对应的主题下查找到。为了确保连接，我们可以在每个服务机器上都部署一个 Proxy 模式的 etcd，这样就可以确保能访问 etcd 集群的服务都能互相连接。




服务发现示意图
![avatar](https://static001.infoq.cn/resource/image/1b/7d/1beabef5a1168cdc43766903e65f907d.jpg)


- **微服务协同工作架构中，服务动态添加。**
随着 Docker 容器的流行，多种微服务共同协作，构成一个相对功能强大的架构的案例越来越多。透明化的动态添加这些服务的需求也日益强烈。通过服务发现机制，在 etcd 中注册某个服务名字的目录，在该目录下存储可用的服务节点的 IP。在使用服务的过程中，只要从服务目录下查找可用的服务节点去使用即可。
![avatar](https://static001.infoq.cn/resource/image/e7/d8/e7d6918c1c9b7c9f2829779966ffb5d8.jpg)
*微服务协同工作示意图*

- **PaaS 平台中应用多实例与实例故障重启透明化。**
PaaS 平台中的应用一般都有多个实例，通过域名，不仅可以透明的对这多个实例进行访问，而且还可以做到负载均衡。但是应用的某个实例随时都有可能故障重启，这时就需要动态的配置域名解析（路由）中的信息。通过 etcd 的服务发现功能就可以轻松解决这个动态配置的问题。
![avatar](https://static001.infoq.cn/resource/image/ec/68/ecea22c641122ee933e4d1a2f4c6ce68.jpg)
*云平台多示例透明化*

**2.场景二：消息发布与订阅**
在分布式系统中，最适用的一种组件间通信方式就是消息发布与订阅。即构建一个配置共享中心，数据提供者在这个配置中心发布消息，而消息使用者则订阅他们关心的主题，一旦主题有消息发布，就会实时通知订阅者。通过这种方式可以做到分布式系统配置的集中式管理与动态更新。

- **应用中用到的一些配置信息放到 etcd 上进行集中管理**。这类场景的使用方式通常是这样：应用在启动的时候主动从 etcd 获取一次配置信息，同时，在 etcd 节点上注册一个 Watcher 并等待，以后每次配置有更新的时候，etcd 都会实时通知订阅者，以此达到获取最新配置信息的目的。
- 分布式搜索服务中，索引的元信息和服务器集群机器的节点状态存放在 etcd 中，供各个客户端订阅使用。使用 etcd 的key TTL功能可以确保机器状态是实时更新的。
- 分布式日志收集系统。这个系统的核心工作是收集分布在不同机器的日志。收集器通常是按照应用（或主题）来分配收集任务单元，因此可以在 etcd 上创建一个以应用（主题）命名的目录 P，并将这个应用（主题相关）的所有机器 ip，以子目录的形式存储到目录 P 上，然后设置一个 etcd 递归的 Watcher，递归式的监控应用（主题）目录下所有信息的变动。这样就实现了机器 IP（消息）变动的时候，能够实时通知到收集器调整任务分配。
- 系统中信息需要动态自动获取与人工干预修改信息请求内容的情况。通常是暴露出接口，例如 JMX 接口，来获取一些运行时的信息。引入 etcd 之后，就不用自己实现一套方案了，只要将这些信息存放到指定的 etcd 目录中即可，etcd 的这些目录就可以通过 HTTP 的接口在外部访问。
![avatar](https://static001.infoq.cn/resource/image/5f/0b/5fb77bf6f5751c45f44cbe9df8ee250b.jpg)

**3.场景三：负载均衡**
在场景一中也提到了负载均衡，本文所指的负载均衡均为软负载均衡。分布式系统中，为了保证服务的高可用以及数据的一致性，通常都会把数据和服务部署多份，以此达到对等服务，即使其中的某一个服务失效了，也不影响使用。由此带来的坏处是数据写入性能下降，而好处则是数据访问时的负载均衡。因为每个对等服务节点上都存有完整的数据，所以用户的访问流量就可以分流到不同的机器上。
- etcd 本身分布式架构存储的信息访问支持负载均衡。etcd 集群化以后，每个 etcd 的核心节点都可以处理用户的请求。所以，把数据量小但是访问频繁的消息数据直接存储到 etcd 中也是个不错的选择，如业务系统中常用的二级代码表（在表中存储代码，在 etcd 中存储代码所代表的具体含义，业务系统调用查表的过程，就需要查找表中代码的含义）。
- 利用 etcd 维护一个负载均衡节点表。etcd 可以监控一个集群中多个节点的状态，当有一个请求发过来后，可以轮询式的把请求转发给存活着的多个状态。类似 KafkaMQ，通过ZooKeeper来维护生产者和消费者的负载均衡。同样也可以用 etcd 来做ZooKeeper的工作。
![avatar](https://static001.infoq.cn/resource/image/67/be/6782904921fa103f42f30113fbf0babe.jpg)
*负载均衡*

**4.场景四：分布式通知与协调**
这里说到的分布式通知与协调，与消息发布和订阅有些相似。都用到了 etcd 中的 Watcher 机制，通过注册与异步通知机制，实现分布式环境下不同系统之间的通知与协调，从而对数据变更做到实时处理。实现方式通常是这样：不同系统都在 etcd 上对同一个目录进行注册，同时设置 Watcher 观测该目录的变化（如果对子目录的变化也有需要，可以设置递归模式），当某个系统更新了 etcd 的目录，那么设置了 Watcher 的系统就会收到通知，并作出相应处理。

- 通过 etcd 进行低耦合的心跳检测。检测系统和被检测系统通过 etcd 上某个目录关联而非直接关联起来，这样可以大大减少系统的耦合性。
- 通过 etcd 完成系统调度。某系统有控制台和推送系统两部分组成，控制台的职责是控制推送系统进行相应的推送工作。管理人员在控制台作的一些操作，实际上是修改了 etcd 上某些目录节点的状态，而 etcd 就把这些变化通知给注册了 Watcher 的推送系统客户端，推送系统再作出相应的推送任务。
- 通过 etcd 完成工作汇报。大部分类似的任务分发系统，子任务启动后，到 etcd 来注册一个临时工作目录，并且定时将自己的进度进行汇报（将进度写入到这个临时目录），这样任务管理者就能够实时知道任务进度。

![avatar](https://static001.infoq.cn/resource/image/38/97/38bee3d541dd88e6f772e64beab92697.jpg)
*分布式协同工作*

**5.场景五：分布式锁**
因为 etcd 使用 Raft 算法保持了数据的强一致性，某次操作存储到集群中的值必然是全局一致的，所以很容易实现分布式锁。锁服务有两种使用方式，一是保持独占，二是控制时序。
- 保持独占即所有获取锁的用户最终只有一个可以得到。etcd 为此提供了一套实现分布式锁原子操作 CAS（CompareAndSwap）的 API。通过设置prevExist值，可以保证在多个节点同时去创建某个目录时，只有一个成功。而创建成功的用户就可以认为是获得了锁。
- 控制时序，即所有想要获得锁的用户都会被安排执行，但是获得锁的顺序也是全局唯一的，同时决定了执行顺序。etcd 为此也提供了一套 API（自动创建有序键），对一个目录建值时指定为POST动作，这样 etcd 会自动在目录下生成一个当前最大的值为键，存储这个新的值（客户端编号）。同时还可以使用 API 按顺序列出所有当前目录下的键值。此时这些键的值就是客户端的时序，而这些键中存储的值可以是代表客户端的编号。

![avatar](https://static001.infoq.cn/resource/image/46/dc/46ff86e2e2c2157bc3f0409845f0e1dc.jpg)
*分布式锁*

6.场景六：分布式队列
分布式队列的常规用法与场景五中所描述的分布式锁的控制时序用法类似，即创建一个先进先出的队列，保证顺序。

另一种比较有意思的实现是在保证队列达到某个条件时再统一按顺序执行。这种方法的实现可以在 /queue 这个目录中另外建立一个 /queue/condition 节点。

- condition 可以表示队列大小。比如一个大的任务需要很多小任务就绪的情况下才能执行，每次有一个小任务就绪，就给这个 condition 数字加 1，直到达到大任务规定的数字，再开始执行队列里的一系列小任务，最终执行大任务。
- condition 可以表示某个任务在不在队列。这个任务可以是所有排序任务的首个执行程序，也可以是拓扑结构中没有依赖的点。通常，必须执行这些任务后才能执行队列中的其他任务。
- condition 还可以表示其它的一类开始执行任务的通知。可以由控制程序指定，当 condition 出现变化时，开始执行队列任务。

![avatar](https://static001.infoq.cn/resource/image/a0/73/a08ce82fb3bee55d0e31d6e2d062a273.jpg)
*分布式队列*

**7.场景七：集群监控与 Leader 竞选**
通过 etcd 来进行监控实现起来非常简单并且实时性强。

1.前面几个场景已经提到 Watcher 机制，当某个节点消失或有变动时，Watcher 会第一时间发现并告知用户。
2.节点可以设置TTL key，比如每隔 30s 发送一次心跳使代表该机器存活的节点继续存在，否则节点消失。

这样就可以第一时间检测到各节点的健康状态，以完成集群的监控要求。

另外，使用分布式锁，可以完成 Leader 竞选。这种场景通常是一些长时间 CPU 计算或者使用 IO 操作的机器，只需要竞选出的 Leader 计算或处理一次，就可以把结果复制给其他的 Follower。从而避免重复劳动，节省计算资源。
这个的经典场景是搜索系统中建立全量索引。如果每个机器都进行一遍索引的建立，不但耗时而且建立索引的一致性不能保证。通过在 etcd 的 CAS 机制同时创建一个节点，创建成功的机器作为 Leader，进行索引计算，然后把计算结果分发到其它节点。
![avatar](https://static001.infoq.cn/resource/image/2d/8c/2dc062ceed9f882ab99ff41f1ca7b18c.jpg)

**8.场景八：为什么用 etcd 而不用ZooKeeper？**
阅读了“ZooKeeper 典型应用场景一览”一文的读者可能会发现，etcd 实现的这些功能， ZooKeeper都能实现。那么为什么要用 etcd 而非直接使用ZooKeeper呢？

相较之下，ZooKeeper有如下缺点：

- 复杂。ZooKeeper的部署维护复杂，管理员需要掌握一系列的知识和技能；而 Paxos 强一致性算法也是素来以复杂难懂而闻名于世；另外，ZooKeeper的使用也比较复杂，需要安装客户端，官方只提供了 Java 和 C 两种语言的接口。
- Java 编写。这里不是对 Java 有偏见，而是 Java 本身就偏向于重型应用，它会引入大量的依赖。而运维人员则普遍希望保持强一致、高可用的机器集群尽可能简单，维护起来也不易出错。
- 发展缓慢。Apache 基金会项目特有的“Apache Way”在开源界饱受争议，其中一大原因就是由于基金会庞大的结构以及松散的管理导致项目发展缓慢。

而 etcd 作为一个后起之秀，其优点也很明显。

- 简单。使用 Go 语言编写部署简单；使用 HTTP 作为接口使用简单；使用 Raft 算法保证强一致性让用户易于理解。
- 数据持久化。etcd 默认数据一更新就进行持久化。
- 安全。etcd 支持 SSL 客户端安全认证。


### etcd 实现原理解读
![avatar](https://static001.infoq.cn/resource/image/cf/94/cf0851c4bcbd2555d09674e7e2a07394.jpg)

从 etcd 的架构图中我们可以看到，etcd 主要分为四个部分。

- HTTP Server： 用于处理用户发送的 API 请求以及其它 etcd 节点的同步与心跳信息请求。
- Store：用于处理 etcd 支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件处理与执行等等，是 etcd 对用户提供的大多数 API 功能的具体实现。
- Raft：Raft 强一致性算法的具体实现，是 etcd 的核心。
- WAL：Write Ahead Log（预写式日志），是 etcd 的数据存储方式。除了在内存中存有所有数据的状态以及节点的索引以外，etcd 就通过 WAL 进行持久化存储。WAL 中，所有的数据提交前都会事先记录日志。Snapshot 是为了防止数据过多而进行的状态快照；Entry 表示存储的具体日志内容。


通常，一个用户的请求发送过来，会经由 HTTP Server 转发给 Store 进行具体的事务处理，如果涉及到节点的修改，则交给 Raft 模块进行状态的变更、日志的记录，然后再同步给别的 etcd 节点以确认数据提交，最后进行数据的提交，再次同步。










### Raft共识算法
[raft共识算法](https://www.jianshu.com/p/8e4bbe7e276c)

#### 1.Raft节点状态
节点有三种状态：Follower，Candidate，Leader，状态之间是互相转换的，可以参考下图
![avatar](https://upload-images.jianshu.io/upload_images/2736397-458eb385e8ccc1c6.png?imageMogr2/auto-orient/strip|imageView2/2/w/605)
每个节点上都有一个倒计时器 (Election Timeout)，时间随机在 150ms 到 300ms 之间。有几种情况会重设 Timeout：
- 收到选举的请求
- 收到 Leader 的 Heartbeat (后面会讲到)

在 Raft 运行过程中，最主要进行两个活动：
- 选主 Leader Election
- 复制日志 Log Replication

#### 2. 选主 Leader Election
选主分为三种情况
- 正常情况下选主
- Leader 出故障情况下的选主
- 多个 Candidate 情况下的选主

##### 2.1正常情况下的选主
![avatar](https://upload-images.jianshu.io/upload_images/2736397-63072559e6b9d35f.png?imageMogr2/auto-orient/strip|imageView2/2/w/671)
假设现在有如图5个节点，5个节点一开始的状态都是 Follower。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-c639092cc6cd0804.png?imageMogr2/auto-orient/strip|imageView2/2/w/670)
在一个节点倒计时结束 (Timeout) 后，这个节点的状态变成 Candidate 开始选举，它给其他几个节点发送选举请求 (RequestVote)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-1ad7ee7ae8fff9cb.png?imageMogr2/auto-orient/strip|imageView2/2/w/668)
其他四个节点都返回成功，这个节点的状态由 Candidate 变成了 Leader，并在每个一小段时间后，就给所有的 Follower 发送一个 Heartbeat 以保持所有节点的状态，Follower 收到 Leader 的 Heartbeat 后重设 Timeout。

这是最简单的选主情况，只要有超过一半的节点投支持票了，Candidate 才会被选举为 Leader，5个节点的情况下，3个节点 (包括 Candidate 本身) 投了支持就行。

##### 2.2 Leader 出故障情况下的选主
![avatar](https://upload-images.jianshu.io/upload_images/2736397-b7cf7a276aa5bf96.png?imageMogr2/auto-orient/strip|imageView2/2/w/680)
一开始已经有一个 Leader，所有节点正常运行。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-25775188b6b66321.png?imageMogr2/auto-orient/strip|imageView2/2/w/680)
Leader 出故障挂掉了，其他四个 Follower 将进行重新选主。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-b24776c5b473ce7a.png?imageMogr2/auto-orient/strip|imageView2/2/w/670)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-b0c6f7d0350db3d2.png?imageMogr2/auto-orient/strip|imageView2/2/w/674)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-d3e2c1b65e0cd570.png?imageMogr2/auto-orient/strip|imageView2/2/w/670)
4个节点的选主过程和5个节点的类似，在选出一个新的 Leader 后，原来的 Leader 恢复了又重新加入了，这个时候怎么处理？在 Raft 里，第几轮选举是有记录的，重新加入的 Leader 是第一轮选举 (Term 1) 选出来的，而现在的 Leader 则是 Term 2，所有原来的 Leader 会自觉降级为 Follower
![avatar](https://upload-images.jianshu.io/upload_images/2736397-249223e23550d8eb.png?imageMogr2/auto-orient/strip|imageView2/2/w/676)

##### 2.3 多个 Candidate 情况下的选主
![avatar](https://upload-images.jianshu.io/upload_images/2736397-7e8c1550477c6f38.png?imageMogr2/auto-orient/strip|imageView2/2/w/644)
假设一开始有4个节点，都还是 Follower。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-235369e90df6c4dd.png?imageMogr2/auto-orient/strip|imageView2/2/w/648)
有两个 Follower 同时 Timeout，都变成了 Candidate 开始选举，分别给一个 Follower 发送了投票请求。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-8a96dd1604c08fc5.png?imageMogr2/auto-orient/strip|imageView2/2/w/660)
两个 Follower 分别返回了ok，这时两个 Candidate 都只有2票，要3票才能被选成 Leader。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-7844d9465c816ada.png?imageMogr2/auto-orient/strip|imageView2/2/w/654)
两个 Candidate 会分别给另外一个还没有给自己投票的 Follower 发送投票请求。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-8424138e1c39373d.png?imageMogr2/auto-orient/strip|imageView2/2/w/648)
但是因为 Follower 在这一轮选举中，都已经投完票了，所以都拒绝了他们的请求。所以在 Term 2 没有 Leader 被选出来。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-f487f6e9c0cddb70.png?imageMogr2/auto-orient/strip|imageView2/2/w/648)
这时，两个节点的状态是 Candidate，两个是 Follower，但是他们的倒计时器仍然在运行，最先 Timeout 的那个节点会进行发起新一轮 Term 3 的投票。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-25086b76d62d09b1.png?imageMogr2/auto-orient/strip|imageView2/2/w/650)
两个 Follower 在 Term 3 还没投过票，所以返回 OK，这时 Candidate 一共有三票，被选为了 Leader。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-cdd1f533e0bbb33a.png?imageMogr2/auto-orient/strip|imageView2/2/w/654)
两个 Follower 已经投完票了，拒绝了这个 Candidate 的投票请求。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-10176f4f4d60f401.png?imageMogr2/auto-orient/strip|imageView2/2/w/650)
Leader 进行 Heartbeat， Candidate 收到后状态自动转为 Follower，完成选主。
以上是 Raft 最重要活动之一选主的介绍，以及在不同情况下如何进行选主。

#### 3. 复制日志 Log Replication
##### 3.1 正常情况下复制日志
Raft 在实际应用场景中的一致性更多的是体现在不同节点之间的数据一致性，客户端发送请求到任何一个节点都能收到一致的返回，当一个节点出故障后，其他节点仍然能以已有的数据正常进行。在选主之后的复制日志就是为了达到这个目的。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-2615f4223329848d.png?imageMogr2/auto-orient/strip|imageView2/2/w/664)
一开始，Leader 和 两个 Follower 都没有任何数据。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-33453ff94de067d1.png?imageMogr2/auto-orient/strip|imageView2/2/w/668)
客户端发送请求给 Leader，储存数据 “sally”，Leader 先将数据写在本地日志，这时候数据还是 Uncommitted (还没最终确认，红色表示)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-1251c82292264ef0.png?imageMogr2/auto-orient/strip|imageView2/2/w/664)
Leader 给两个 Follower 发送 AppendEntries 请求，数据在 Follower 上没有冲突，则将数据暂时写在本地日志，Follower 的数据也还是 Uncommitted。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-8e4fe60b92e5f4dd.png?imageMogr2/auto-orient/strip|imageView2/2/w/666)
Follower 将数据写到本地后，返回 OK。Leader 收到后成功返回，只要收到的成功的返回数量超过半数 (包含Leader)，Leader 将数据 “sally” 的状态改成 Committed。( 这个时候 Leader 就可以返回给客户端了)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-59b9c16018de6a8f.png?imageMogr2/auto-orient/strip|imageView2/2/w/674)
Leader 再次给 Follower 发送 AppendEntries 请求，收到请求后，Follower 将本地日志里 Uncommitted 数据改成 Committed。这样就完成了一整个复制日志的过程，三个节点的数据是一致的

##### 3.2 Network Partition 情况下进行复制日志
在 Network Partition 的情况下，部分节点之间没办法互相通信，Raft 也能保证在这种情况下数据的一致性。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-2d0423a3a1466d99.png?imageMogr2/auto-orient/strip|imageView2/2/w/718)
一开始有 5 个节点处于同一网络状态下。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-9eb9ee3f7e900ac0.png?imageMogr2/auto-orient/strip|imageView2/2/w/718)
Network Partition 将节点分成两边，一边有两个节点，一边三个节点。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-ee03f4d0b8cebfaa.png?imageMogr2/auto-orient/strip|imageView2/2/w/700)
两个节点这边已经有 Leader 了，来自客户端的数据 “bob” 通过 Leader 同步到 Follower。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-89370d272ddf1a5f.png?imageMogr2/auto-orient/strip|imageView2/2/w/718)
因为只有两个节点，少于3个节点，所以 “bob” 的状态仍是 Uncommitted。所以在这里，服务器会返回错误给客户端.
![avatar](https://upload-images.jianshu.io/upload_images/2736397-3ade6c4d64aea90f.png?imageMogr2/auto-orient/strip|imageView2/2/w/726)
另外一个 Partition 有三个节点，进行重新选主。客户端数据 “tom” 发到新的 Leader，通过和上节网络状态下相似的过程，同步到另外两个 Follower。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-c684006e9f9ecc63.png?imageMogr2/auto-orient/strip|imageView2/2/w/720)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-ca63d0cef1b5702a.png?imageMogr2/auto-orient/strip|imageView2/2/w/696)
![avatar](https://upload-images.jianshu.io/upload_images/2736397-ca63d0cef1b5702a.png?imageMogr2/auto-orient/strip|imageView2/2/w/696)
因为这个 Partition 有3个节点，超过半数，所以数据 “tom” 都 Commit 了。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-cd0f310dbf16f12f.png?imageMogr2/auto-orient/strip|imageView2/2/w/716)
网络状态恢复，5个节点再次处于同一个网络状态下。但是这里出现了数据冲突 “bob" 和 “tom".
![avatar](https://upload-images.jianshu.io/upload_images/2736397-da5f3690cb880c78.png?imageMogr2/auto-orient/strip|imageView2/2/w/708)
三个节点的 Leader 广播 AppendEntries.
![avatar](https://upload-images.jianshu.io/upload_images/2736397-938d63d89875df5a.png?imageMogr2/auto-orient/strip|imageView2/2/w/720)
两个节点 Partition 的 Leader 自动降级为 Follower，因为这个 Partition 的数据 “bob” 没有 Commit，返回给客户端的是错误，客户端知道请求没有成功，所以 Follower 在收到 AppendEntries 请求时，可以把 “bob“ 删除，然后同步 ”tom”，通过这么一个过程，就完成了在 Network Partition 情况下的复制日志，保证了数据的一致性。
![avatar](https://upload-images.jianshu.io/upload_images/2736397-c97a5fcc7b75a398.png?imageMogr2/auto-orient/strip|imageView2/2/w/744)

Raft 是能够实现分布式系统强一致性的算法，每个系统节点有三种状态 Follower，Candidate，Leader。实现 Raft 算法两个最重要的事是：选主和复制日志。
关于etcd还可以[参考这里](https://www.infoq.cn/article/coreos-analyse-etcd/),etcd 算法的动画可以[参考这里](http://thesecretlivesofdata.com/raft/)








## 服务之RPC
该章节[参考这里](https://juejin.im/post/5a5ee63d518825732914748c)
### 相关背景
随着业务的发展、用户量的增长，系统数量增多，调用依赖关系也变得复杂，为了确保系统高可用、高并发的要求，系统的架构也从单体时代慢慢迁移至服务SOA时代，**根据不同服务对系统资源的要求不同，我们可以更合理的配置系统资源，使系统资源利用率最大化**。
![avatar](https://user-gold-cdn.xitu.io/2018/1/17/16102b35e64b0107?imageView2/0/w/1280/h/960/ignore-error/1)
- **单一应用架构** 当网站流量很小时，只需一个应用，将所有功能都部署在一起，以减少部署节点和成本。此时，用于简化增删改查工作量的 数据访问框架(ORM) 是关键。
- **垂直应用架构** 当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，将应用拆成互不相干的几个应用，以提升效率。
此时，用于加速前端页面开发的 Web框架(MVC) 是关键。
- **分布式服务架构**
当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。此时，用于提高业务复用及整合的 分布式服务框架(RPC) 是关键。
- **流动计算架构**
当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。此时，用于提高机器利用率的 资源调度和治理中心(SOA) 是关键。

### 服务带来的挑战
系统的复杂度上升、服务依赖关系、服务性能监控、全链路日志、容灾、断路器、限流等。

根据现在团队的业务系统情况，首先我们要**梳理出现存的问题是什**么：
- 多种调用传输方式：HTTP方式、WebService方式；
- 服务调用依赖关系：人工记录，查看代码分析；
- 服务调用性能监控：日志记录，人工查看时间；
- 服务与应用紧耦合：服务挂掉，应用无法可用；
- 服务集群负载配置：Nginx配置，存在单点问题；

在去选择技术框架时，技术框架最基本要解决上面现存问题，同时我们也要确认出我们的期望，**要达到的目标是什么**：
- 支持当前业务需求，这是最最基本的条件；
- 服务避免单点问题，去中心化；
- 服务高可用、高并发，解耦服务依赖；
- 服务通用化，支持异构系统调用服务；
- 服务依赖关系自维护，可视化(阿里云AHAS)；
- 服务性能监控自统计，可视化（鹰眼）；
- 服务需自带注册、发现、健康检查、负载均衡等特性；
- 开发人员关注度高，上手快，简单轻量，低侵入；

#### mas(微服务)和soa(面向服务架构的区别和联系)
[参考](https://blog.csdn.net/qq_15071263/article/details/84027556)
##### 相似之处
- 都是面向服务
- 都是基于HTTP协议

##### 区别和联系
传统的SOA 一般是大而全的单块架构，MSA 是很分散的服务。
一般情况下，SOA需要对整个系统进行规范约束，但是MSA的每个服务都可以有自己的开发语言和开发方式，灵活性比SOA更高。

###### 基于SOA的架构
1、易于部署，只需要扔war包就可以了
2、易于伸缩，只需要在负载均衡下部署应用的拷贝即可
3、拥有较为庞大的代码库，在理解业务时，会造成困扰
4、当项目随着时间的变化越来越大的时候，IDE的速度会变慢
5、Web容器超载，应用变大时，Web容器的启动时间变长
6、在持续部署上存在问题，当你只需要更新某一个组件时，必须重新部署整个应用
7、伸缩性不好，单块架构只能在一个维度伸缩
8、难以调整开发规模
9、需要对一个技术栈长期投入，比如使用Java，那么脱离JVM，Java的开发环境就不能给项目开发组件，并且，项目的迁移也受限于语言和框架层面，或者面临技术升级时，在某些情况下不得不重写整个应用

##### 基于微服务的架构
在优点上
1、每个微服务都相对较小
2、易于开发者理解
3、IDE反应更快，开发更搞笑
4、Web 容器启动更快
5、每个服务可以独立开发，部署
6、易于伸缩开发组织架构
7、提升故障隔离
8、消除了单一技术栈的长期投入

在缺点上
1、需要分布式系统发额外复杂度
2、测试更加困难
3、实现跨服务的用例需要开发者之间的细致协作
4、生产环境的部署复杂度增加
5、更大的内存开销

**两者之间最关键的区别在于**，微服务专注于以自治的方式产生价值。但是两种架构背后的意图是不同的：SOA尝试将应用集成，一般采用中央管理模式来确保各应用能够交互运作。微服务尝试部署新功能，快速有效地扩展开发团队。它着重于分散管理、代码再利用与自动化执行。

功能 | SOA | 微服务
:-: | :-: | :-:
组件大小 | 大块业务逻辑 | 单独任务或小块业务逻辑
耦合 | 通常松耦合| 总是松耦合
公司架构 | 任何类型 | 小型、专注于功能交叉的团队
管理 | 着重中央管理 | 着重分散管理
目标 | 确保应用能够交互操作 | 执行新功能，快速拓展开发团队

微服务并不是一种新思想的方法。它更像是一种思想的精炼，一种SOA的精细化演进，并且更好地利用了先进的技术以解决问题，例如容器与自动化等。所以对于我们 去选择服务技术框架时，并不是非黑即白，而是针对SOA、MSA两种架构设计同时要考虑到兼容性，对于现有平台情况架构设计，退则守SOA，进则攻MSA，阶段性选择适合的。

### 框架
现在业界比较成熟的服务框架有很多，比如：Hessian、CXF、Dubbo、Dubbox、Spring Cloud、gRPC、thrift等技术实现，都可以进行远程调用，具体技术实现优劣参考以下分析，这也是具体在技术方案选择过程中的重要依据。

#### 服务框架对比
**dubbo**
  是阿里巴巴公司开源的一个Java高性能优秀的服务框架，使得应用可通过高性能的 RPC 实现服务的输出和输入功能，可以和 Spring框架无缝集成。

- Dubbox和Dubbo本质上没有区别，名字的含义扩展了Dubbo而已，以下扩展出来的功能，也是选择Dubbox很重要的考察点。
- 支持REST风格远程调用（HTTP + JSON/XML)；
- 支持基于Kryo和FST的Java高效序列化实现；
- 支持基于Jackson的JSON序列化；
- 支持基于嵌入式Tomcat的HTTP remoting体系；
- 升级Spring至3.x；
- 升级ZooKeeper客户端；
- 支持完全基于Java代码的Dubbo配置；

**spring cloud**
Spring Cloud完全基于Spring Boot，是一个非常新的项目。Spring Cloud 为开发者提供了在分布式系统（配置管理，服务发现，熔断，路由，微代理，控制总线，一次性token，全局琐，leader选举，分布式session，集群状态）中快速构建的工具，使用Spring Cloud的开发者可以快速的启动服务或构建应用。
**缺点**:是项目很年轻，很少见到国内业界有人在生产上成套使用，一般都是只有其中一两个组件。
![avatar](https://img-blog.csdn.net/20180312234753534?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTUwMDEyMjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

以上RPC框架功能比较：
框架 | Hessian | Montan | rpcx | gRPC | Thrift |Dubbo|Dubbox| Spring Cloud
:-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
开发语言 | 跨语言 | Java | Go | 跨语言 | 跨语言 | Java | Java | Java
分布式(服务治理) | × | √ | √ | × | × | √ | √ | √
多序列化框架支持	|hessian|	√(支持Hessian2、Json,可扩展)	| √ |	×只支持protobuf)	|×(thrift格式)	|√|	√	|√
多种注册中心|	×	|√|	√	|×|	×	|√|	√|	√
管理中心	|×|	√	|√|	×	|×|	√	|√	|√
跨编程语言	|√	|×(支持php client和C server)	|×|	√|	√	|×|	×	|×
支持REST	|×	|×	|×	|×|	×	|×	| √	|√
关注度	|低	|中	|低	|中	|中|	中	|高	|中
上手难度	|低|	低	|中	|中	|中	|低	|低	|中
运维成本	|低	|中	|中	|中	|低	|中	|中	|中
开源机构	|Caucho	|Weibo	|Apache	|Google	|Apache	|Alibaba	|Dangdang	|Apache

**实际场景中的选择**

- Spring Cloud：Spring全家桶，用起来很舒服，只有你想不到，没有它做不到。可惜因为发布的比较晚，国内还没出现比较成功的案例，大部分都是试水，不过毕竟有Spring作背书，还是比较看好。
- Dubbox：相对于Dubbo支持了REST，估计是很多公司选择Dubbox的一个重要原因之一，但如果使用Dubbo的RPC调用方式，服务间仍然会存在API强依赖，各有利弊，懂的取舍吧。
- Thrift：如果你比较高冷，完全可以基于Thrift自己搞一套抽象的自定义框架吧。
- Montan：可能因为出来的比较晚，目前除了新浪微博16年初发布的，
- Hessian：如果是初创公司或系统数量还没有超过5个，推荐选择这个，毕竟在开发速度、运维成本、上手难度等都是比较轻量、简单的，即使在以后迁移至SOA，也是无缝迁移。
- rpcx/gRPC：在服务没有出现严重性能的问题下，或技术栈没有变更的情况下，可能一直不会引入，即使引入也只是小部分模块优化使用。

### dubbo
而对于远程调用如果没有分布式的需求，其实是不需要用这么重的框架，只有在分布式的时候，才有Dubbo这样的分布式服务框架的需求，说白了就是个远程服务调用的分布式框架，**其重点在于分布式的治理。**

#### dubbo的核心功能
1.**Remoting:远程通讯**，提供对多种NIO框架抽象封装，包括“同步转异步”和“请求-响应”模式的信息交换方式。
2.**Cluster: 服务框架**，提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。
3.**Registry: 服务注册中心**，基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。

#### dubbo的组件角色
![avatar](https://user-gold-cdn.xitu.io/2018/3/20/16241d6d49a76c43?imageView2/0/w/1280/h/960/ignore-error/1)

**调用关系说明：**
1.服务容器Container负责启动，加载，运行服务提供者。
2.服务提供者Provider在启动时，向注册中心注册自己提供的服务。
3.服务消费者Consumer在启动时，向注册中心订阅自己所需的服务。
4.注册中心Registry返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
5.服务消费者Consumer，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
6.服务消费者Consumer和提供者Provider，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心Monitor。
![avatar](https://user-gold-cdn.xitu.io/2018/3/20/16241d6d49d9ccf0?imageView2/0/w/1280/h/960/ignore-error/1)

#### dubbo的总体框架
ubbo最大的特点是按照分层的方式来架构，使用这种方式可以使各个层之间解耦合（或者最大限度地松耦合）。所以，我们横向以分层的方式来看下Dubbo的架构，如图所示：
![avatar](https://user-gold-cdn.xitu.io/2018/3/20/16241d6d49c35881?imageView2/0/w/1280/h/960/ignore-error/1)

Dubbo框架设计一共划分了10个层，而最上面的Service层是留给实际想要使用Dubbo开发分布式服务的开发者实现业务逻辑的接口层。图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口， 位于中轴线上的为双方都用到的接口。

- 配置层（Config）：对外配置接口，以ServiceConfig和ReferenceConfig为中心，可以直接new配置类，也可以通过Spring解析配置生成配置类。
- 服务代理层（Proxy）：服务接口透明代理，生成服务的客户端Stub和服务器端Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。
- 服务注册层（Registry）：封装服务地址的注册与发现，以服务URL为中心，扩展接口为RegistryFactory、Registry和RegistryService。可能没有服务注册中心，此时服务提供方直接暴露服务。
- 集群层（Cluster）：封装多个提供者的路由及负载均衡，并桥接注册中心，以Invoker为中心，扩展接口为Cluster、Directory、Router和LoadBalance。将多个服务提供方组合为一个服务提供方，实现对服务消费方来透明，只需要与一个服务提供方进行交互。
- 监控层（Monitor）：RPC调用次数和调用时间监控，以Statistics为中心，扩展接口为MonitorFactory、Monitor和MonitorService。
- 远程调用层（Protocol）：封将RPC调用，以Invocation和Result为中心，扩展接口为Protocol、Invoker和Exporter。Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。Invoker是实体域，它是Dubbo的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起invoke调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
- 信息交换层（Exchange）：封装请求响应模式，同步转异步，以Request和Response为中心，扩展接口为Exchanger、ExchangeChannel、ExchangeClient和ExchangeServer。
- 网络传输层（Transport）：抽象mina和netty为统一接口，以Message为中心，扩展接口为Channel、Transporter、Client、Server和Codec。
- 数据序列化层（Serialize）：可复用的一些工具，扩展接口为Serialization、 ObjectInput、ObjectOutput和ThreadPool。

从上图可以看出，Dubbo对于服务提供方和服务消费方，从框架的10层中分别提供了各自需要关心和扩展的接口，构建整个服务生态系统（服务提供方和服务消费方本身就是一个以服务为中心的）。

**根据官方提供的，对于上述各层之间关系的描述，如下所示：**
1.在RPC中，Protocol是核心层，也就是只要有Protocol + Invoker + Exporter就可以完成非透明的RPC调用，然后在Invoker的主过程上Filter拦截点。
2.图中的Consumer和Provider是抽象概念，只是想让看图者更直观的了解哪些类分属于客户端与服务器端，不用Client和Server的原因是Dubbo在很多场景下都使用Provider、Consumer、Registry、Monitor划分逻辑拓普节点，保持统一概念。
3.而Cluster是外围概念，所以Cluster的目的是将多个Invoker伪装成一个Invoker，这样其它人只要关注Protocol层Invoker即可，加上Cluster或者去掉Cluster对其它层都不会造成影响，因为只有一个提供者时，是不需要Cluster的。
[dubbo之cluster层](https://segmentfault.com/a/1190000017089603)
![avatar](https://upload-images.jianshu.io/upload_images/12016719-16d235b1ddf7b489.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/550)
**各节点关系：**
- 这里的Invoker是Provider的一个可调用Service的抽象，Invoker封装了Provider地址及Service接口信息；
- Directory代表多个Invoker，可以把它看成List，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更；
- Cluster将Directory中的多个Invoker伪装成一个 Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个；
- Router负责从多个Invoker中按路由规则选出子集，比如读写分离，应用隔离等；
- LoadBalance负责从多个Invoker中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选；
- Cluster经过目录，路由，负载均衡获取到一个可用的Invoker，交给上层调用。

4.Proxy层封装了所有接口的透明化代理，而在其它层都以Invoker为中心，只有到了暴露给用户使用时，才用Proxy将Invoker转成接口，或将接口实现转成Invoker，也就是去掉Proxy层RPC是可以Run的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。

5.而Remoting实现是Dubbo协议的实现，如果你选择RMI协议，整个Remoting都不会用上，Remoting内部再划为Transport传输层和Exchange信息交换层，Transport层只负责单向消息传输，是对Mina、Netty、Grizzly的抽象，它也可以扩展UDP传输，而Exchange层是在传输层之上封装了Request-Response语义。
6.Registry和Monitor实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。

#### 服务调用流程
![avatar](https://user-gold-cdn.xitu.io/2018/3/20/16241d6d49cc5515?imageView2/0/w/1280/h/960/ignore-error/1)

#### 注册/注销服务
![avatar](https://user-gold-cdn.xitu.io/2018/3/20/16241d6d49b32422?imageView2/0/w/1280/h/960/ignore-error/1)

#### 订阅/取消服务
![avatar](https://user-gold-cdn.xitu.io/2018/3/20/16241d6d49a8c8d2?imageView2/0/w/1280/h/960/ignore-error/1)

#### dubbo的工作流程
- 第一步：provider 向注册中心去注册
- 第二步：consumer 从注册中心订阅服务，注册中心会通知 consumer 注册好的服务
- 第三步：consumer 调用 provider
- 第四步：consumer 和 provider 都异步通知监控中心
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/dubbo-operating-principle.png)

从上到下：服 配 代 注 聚 监 协 交 传 序

#### dubbo 支持哪些通信协议？支持哪些序列化协议？说一下 Hessian 的数据结构？PB 知道吗？为什么 PB 的效率是最高的？

**序列化**:就是把数据结构或者是一些对象，转换为二进制串的过程。
**反序列化**：是将在序列化过程中所生成的二进制串转换成数据结构或者对象的过程。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/serialize-deserialize.png)

##### dubbo支持不同的通信协议
- dubbo
认就是走 dubbo 协议，单一长连接，进行的是 NIO 异步通信，基于 hessian 作为序列化协议。使用的场景是：传输数据量小（每次请求在 100kb 以内），但是并发量很高。
为了要支持高并发场景，一般是服务提供者就几台机器，但是服务消费者有上百台，可能每天调用量达到上亿次！此时用长连接是最合适的，就是跟每个服务消费者维持一个长连接就可以，可能总共就 100 个连接。然后后面直接基于长连接 NIO 异步通信，可以支撑高并发请求。

**长连接**，通俗点说，就是建立连接过后可以持续发送请求，无须再建立连接。
**短连接**，每次要发送请求之前，需要先重新建立一次连接。

###### dubbo通信模型
[nio单一长连接--dubbo通信模型实现](https://www.jianshu.com/p/13bef2795c44)
**BIO通信缺陷**
BIO的完整连接示意图如下：
![avatar](https://upload-images.jianshu.io/upload_images/3727888-76b1222bc2d3a0c3.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/920/format/webp)

- 缺陷一、IO阻塞
可以看出，在BIO中，除了建立连接比较耗时之外，在客户端将数据传输到服务端之前，服务端的IO（输入流）阻塞，然后在服务端将返回值传输回来之前，客户端的IO（输入流）阻塞。**也就是说在一次RPC调用开始到完成之前，这个连接一直被此次调用所占用，但是实际上这次调用中，真正需要网络连接的只有中间的数据传输过程，在客户端写出和服务端读取并执行远端方法这两个时间点，其实网络连接是空闲的**。这就是BIO连接中浪费了网络资源的地方。

- 缺陷二、大量连接
1. 由于BIO的IO阻塞，导致每次RPC调用会占用一个连接。而正因为如此，为了减少频繁创建连接消耗的时间，引入了连接池（此处的连接池指普通的HTTP连接池，非异步连接池）的概念，连接池解决了频繁创建连接的资源消耗，但是没有解决根本性的阻塞问题。而且在服务消费者（客户端）数量远大于服务提供者（服务端）数量的时候，会导致服务提供者建立了大量的连接，而本身由于硬件资源的限制，单机最大连接数是有限的（这个限制以前是1w，也就是以前的C10K问题，据说近几年已经提升至50万个连接），所以在服务消费者过多，而服务提供者数量过少的情况下，服务提供者有因为过多的连接而被拖垮的风险（这需要极大的并发数，每秒上百万次的调用）。当然，要解决这个问题，增加机器，从而增加服务提供者数量是可以解决的，但是没有充分利用单机性能。
2. 建立大量连接的另一个弊端，是操作系统频繁的线程上下文切换，因为连接数过多，线程切换频繁，会消耗大量的资源，而且这些切换可能不是必要的。比如当前建立了大量的连接，可能大部分处于阻塞状态，根本没有挨个挨个切换的必要。但是因为操作系统任务调度时并不会忽略阻塞状态的线程，所以造成浪费。

**NIO单一长连接实现分析**
*长连接宏观简介*
![avatar](https://upload-images.jianshu.io/upload_images/3727888-2632e469660f2db2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
NIO由三种角色组成，Selector、SocketChannel、Buffer。

*SocketChannel*: 相当于BIO中的Socket，分为SocketChannel和ServerSocketChannel两种，是真正建立连接并传输数据的管道。这个管道不同于BIO的Socket的点就是，这个管道可以被多个线程共用，线程A使用这个管道写出数据了之后，线程B还可以使用这个管道写出数据，不再被某一次调用所独占。所以就可以不需要像BIO一样建立那么多的连接，一个客户端的一个连接就够了（当然，实际应用中因为机器都是多核，实际上建立核数个连接个人感觉是比较好的）。

*Buffer*:是用来与SocketChannel互通数据的对象，本质上是一块内存区域。SocketChannel是不支持直接读写数据的，所有的读写操作必须通过Buffer来实现。值得一提的是，我们经常说在JVM中有的时候会使用虚拟机之外的内存，说的就是NIO中的Buffer，在分配内存的时候可以选择使用虚拟机外内存，减少数据的复制。

*Selector*:是用来监控SocketChannel的事件的，其实是实现非阻塞的关键。NIO是基于事件的，将BIO中的流式传输改为了事件机制。BIO中，一个连接拥有一个输入／输出流，只要数据传输完成，流就可以读取数据。在NIO中，Selector定义了四种事件，OP_READ、OP_WRITE、OP_CONNECT、OP_ACCEPT。当服务端或者客户端收到写入完成的一次数据时，会触发OP_READ事件，此时可以从连接中读取数据。同理，当可以往连接中写入数据的时候，触发OP_WRITE事件（但是一般情况下这个事件没有必要，因为连接一般都是可写的）。客户端与服务端建立连接的时候，客户端会收到OP_CONNECT事件，而服务端会触发OP_ACCEPT事件。通过这一系列事件将数据的发送与读写解耦，实现异步调用。将一个SocketChannel+一个事件绑定在一个Selector上，Selector本质上是轮询每一个SocketChannel，如果没有事件触发，那么线程阻塞，如果有事件触发，返回对应的SocketChannel，以便进行后续的处理。

**使用NIO设计RPC调用分析**
前面提到，由于NIO的SocketChannel是非阻塞的，所以不再需要连接池，使用一个连接就够了。
![avatar](https://upload-images.jianshu.io/upload_images/3727888-2a3e097144cd7a11?imageMogr2/auto-orient/strip|imageView2/2/w/1072/format/webp)

但是如果真的使用NIO来进行RPC调用的话，会有数据和调用方对应不上的问题，如下图：
![avatar](https://upload-images.jianshu.io/upload_images/3727888-a93021a1bbf552f8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
每次调用的返回值必须与调用方对应上，为此，Dubbo的设计是给每个请求设计一个请求id，在发送请求与发送返回值时都带上这个id。详细思路如下图：
![avatar](https://upload-images.jianshu.io/upload_images/3727888-098e4b3fb2b736f2.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
业务线程在发出请求之前，需要存储一个请求对象，同时挂起相应的业务线程（挂起不会被任务调度，所以不存在线程切换消耗），这个请求对象包含了此次请求的id，然后在获取服务端返回的数据的时候，解析出这个id，通过这个id取出请求对象，并唤醒对应的线程。

#### 如何基于dubbo进行服务治理、服务降级、失败重试以及超时重试
**服务治理**
1.调用链路自动生成
一个大型的分布式系统，或者说是用现在流行的微服务框架来说，分布式系统由大量的服务组成。那么这些服务之间是如何相互调用的？调用链路是什么？
需要基于dubbo做的分布式系统中，对各个服务之间的调用自动记录下来，然后自动将各个服务之间的依赖关系和调用链路生成出来，做成一张图，现实出来，这样大家才可以看到。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/dubbo-service-invoke-road.png)

2.服务访问压力以及时长统计
需要自动统计各个接口和服务之间的调用次数以及访问延时，而且要分成两个级别
- 一个级别是接口粒度，就是每个服务的每个接口每天被调用多少次，TP50h/TP90/TP99，三个档次的请求延时分别是多少；
**tp的含义**：指在一段时间内，统计该方法每次调用所消耗的时间，并将这些时间按从小到大的顺序进行排序，并取出结果为：总次数*指标数=对应TP指标的序号，再根据序号取出对应排序好的时间，即TP指标值。
- 第二个级别从源头的入口开始，一个完整的请求链路经过几十个服务之后，完成一次请求，每天全链路走多少次，全链路的请求延时的TP50/TP90/TP99分别是多少。

这些东西都搞定了之后，后面才可以看到当前系统的主要压力在哪，如何来进行优化和扩容。

3.其他
- 服务分层（避免循环依赖）
- 调用链路失败监控和报警
- 服务鉴权
- 每个服务的可用性监控（接口调用的成功率？几个9？99.99%,99.9%,99%）

**服务降级**
比如说服务A调用服务B,结果服务B挂了，服务A重试几次后调用服务B还是不行，那么直接降级，走一个备用的逻辑，给客户返回响应。

**失败重试和超时重试**
所谓失败重试，就是 consumer 调用 provider 要是失败了，比如抛异常了，此时应该是可以重试的，或者调用超时了也可以重试。

//todo dubbo的filter 涉及到dubbo的责任链模式
[参考](https://www.jianshu.com/p/c5ebe3e08161)

**分布式接口的顺序性如何保证**
首先，个人建议是，你们从业务逻辑上设计的这个系统最好不要有这种顺序性的保证，因为一旦引入顺序性保证，比如适应分布式锁，会导致系统复杂度上升，而且会带来效率低下，热点数据压力过大的问题。

下面我给个我们用过的方案吧，简单来说，首先你得用 dubbo 的一致性 hash 负载均衡策略，将比如某一个订单 id 对应的请求都给分发到某个机器上去，接着就是在那个机器上，因为可能还是多线程并发执行的，你可能得立即将某个订单 id 对应的请求扔一个内存队列里去，强制排队，这样来确保他们的顺序性。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/distributed-system-request-sequence.png)

但是这样引发的后续问题就很多，比如说要是某个订单对应的请求特别多，造成某台机器成热点怎么办？解决这些问题又要开启后续一连串的复杂技术方案......曾经这类问题弄的我们头疼不已，所以，还是建议什么呢？

最好是比如说刚才那种，一个订单的插入和删除操作，能不能合并成一个操作，就是一个删除，或者是其它什么，避免这种问题的产生。

## Elasticsearch

### ES简介
[参考](https://blog.csdn.net/laoyang360/article/details/79293493?spm=a2c4e.10696291.0.0.6c5019a4RdoGp6)
#### ELK stack
ELK stack由最早期的最核心的Elasticsearch ,集合Logstash,kibana,beats等发展而来，形成ELK stack体系。如下图所示
![avatar](https://yqfile.alicdn.com/7cc0ee14e7d1098ccf448cfff37309ef475c6221.png)

#### Elasticsearch认知
Elasticsearch 为开源的，分布式的，基于restful API、支持PB甚至更改数量级的搜索引擎工具。
相对于mysql,给出如下的对应关系表会更好的理解一些：
![avatar](https://yqfile.alicdn.com/e164c17310623ce6dfdf169e0c88368772ff343f.png)
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/es-index-type-mapping-document-field.png)

##### Elasticsearch 比传统的关系性数据库(oracle,mysqls)非关系型数据库（mongodb）对比
如下是传统的关系型数据库（如Oracle、MySQL）、非关系型的数据库（如 Mongo）所做不到的：
1.传统的关系型数据库虽然能支持类型“like 待检索词”模糊语句匹配，但无法进行全文检索（分词检索）。
2.非关系型数据库 Mongo 虽能进行简单的全文检索，但对中文支持的不好、数据量大性能会有问题，这点是在实际应用中总结出的。
**mongodb的全文检索原理**
[mongodb的全文检索依靠的是标签和索引](http://www.voidcn.com/article/p-zlcjehfe-de.html)

#### Logstash认知
[logstash最佳实践](https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/input/file.html)
可以把 Logstash 理解成流入、流出 Elasticsearch 的传送带。

支持：不同类型的数据或实施数据流经过 Logstash 写入 ES 或者从 ES 中读出写入文件或对应的实施数据流。包括但不限于：
- 本地或远程文件；
- Kafka 实时数据流——核心插件有 logstashinputkafka/logstashoutputkafka；
- MySQL、Oracle 等关系型数据库——核心插件有 logstashinputjdbc/logstashouputjdbc；
- Mongo 非关系型数据库——核心插件有 logstashinputmongo/logstashoutputmongo；
- Redis 数据流；

#### kibana认知
Kibana 是 ES 大数据的图形化展示工具。集成了 DSL 命令行查看、数据处理插件、继承了 x-pack（收费）安全管理插件等。

#### Beats 认知

Beats 是一个开源的用来构建轻量级数据汇集的平台，可用于将各种类型的数据发送至 Elasticsearch 与 Logstash。

#### ELK stack
**场景一：使用 ES 作为业务系统的后端。**
此时，ES 的作用类似传统业务系统中的 MySQL、PostgreSQL、Oracle 或者 Mongo 等的基础关系型数据库或非关系型数据库的作用。

我们举例说明。使用 ES 对基础文档进行检索操作，如将传统的 word 文档、PDF 文档、PPT 文档等通过 Openoffice 或者 pdf2htmlEX 工具转换为 HTML，再将 HTML 以JSON 串的形式录入到 ES，以对外提供检索服务。

**场景二：在原有系统中增加 ES、Logstash、Kibana等。**

原有的业务系统中存在 MySQL、Oracle、Mongo 等基础数据，但想实现全文检索服务，就在原有业务系统基础的加上一层 ELK。

举例一，将原有系统中 MySQL 中的数据通过 logstashinputjdbc 插件导入到 ES 中，并通过 Kibana 进行图形化展示。

举例二，将原有存储在 Hadoop HDFS 中的数据导入到 ES 中，对外提供检索服务。

**场景三：使用 ELK Stack 结合现有工具对外提供服务。**

举例一，日志检索系统。将各种类型的日志通过 Logstash 导入 ES 中，通过 Kibana 或者 Grafana 对外提供可视化展示。

举例二，通过 Flume 等将数据导入 ES 中，通过 ES 对外提供全文检索服务。

**场景四：其他综合业务场景**

主要借助 ES 强大的全文检索功能实现，如分页查询、各类数据结果的聚合分析、图形化展示（饼图、线框图、曲线图等）。

举例说明，像那些结合实际业务的场景，如安防领域、金融领域、监控领域等的综合应用。

**大量数据的写入和读取解决方案**
1、存储数据时按有序存储；
2、将数据和索引分离；
3、压缩数据；

#### ES的定义
ES=elaticsearch简写， Elasticsearch是一个开源的高扩展的分布式全文检索引擎，它可以近乎实时的存储、检索数据；本身扩展性很好，可以扩展到上百台服务器，处理PB级别的数据。
Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。
**数据模型**
![avatar](https://upload-images.jianshu.io/upload_images/1604849-f0256300e34c6ccf.png?imageMogr2/auto-orient/strip|imageView2/2/w/973)

#### ES的核心概念
1）Cluster：集群。
ES可以作为一个独立的单个搜索服务器。不过，为了处理大型数据集，实现容错和高可用性，ES可以运行在许多互相合作的服务器上。这些服务器的集合称为集群。

2）Node：节点。
形成集群的每个服务器称为节点。

3）Shard：分片。
当有大量的文档时，由于内存的限制、磁盘处理能力不足、无法足够快的响应客户端的请求等，一个节点可能不够。这种情况下，数据可以分为较小的分片。每个分片放到不同的服务器上。
当你查询的索引分布在多个分片上时，ES会把查询发送给每个相关的分片，并将结果组合在一起，而应用程序并不知道分片的存在。即：这个过程对用户来说是透明的。

4）Replia：副本。
为提高查询吞吐量或实现高可用性，可以使用分片副本。
副本是一个分片的精确复制，每个分片可以有零个或多个副本。ES中可以有许多相同的分片，其中之一被选择更改索引操作，这种特殊的分片称为主分片。
当主分片丢失时，如：该分片所在的数据不可用时，集群将副本提升为新的主分片。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/es-cluster.png)
你搞一个索引，这个索引可以拆分成多个 shard，每个 shard 存储部分数据。拆分多个 shard 是有好处的:
**一是支持横向扩展**，比如你数据量是 3T，3 个 shard，每个 shard 就 1T 的数据，若现在数据量增加到 4T，怎么扩展，很简单，重新建一个有 4 个 shard 的索引，将数据导进去；
**二是提高性能**，数据分布在多个 shard，即多台服务器上，所有的操作，都会在多台机器上并行分布式执行，提高了吞吐量和性能。

接着就是这个 shard 的数据实际是有多个备份，就是说每个 shard 都有一个 primary shard，负责写入数据，但是还有几个 replica shard。primary shard 写入数据之后，会将数据同步到其他几个 replica shard 上去。
通过这个 replica 的方案，每个 shard 的数据都有多个备份，如果某个机器宕机了，没关系啊，还有别的数据副本在别的机器上呢。高可用了吧。


#### ES要解决的问题
1）检索相关数据；
2）返回统计结果；
3）速度要快。

#### ES的工作原理
当ElasticSearch的节点启动后，它会利用多播(multicast)(或者单播，如果用户更改了配置)寻找集群中的其它节点，并与之建立连接。这个过程如下图所示：
![avatar](https://img-blog.csdn.net/20160818205953345)

##### ES的写入过程
- 客户端选择一个 node 发送请求过去，这个 node 就是 coordinating node（协调节点）。
- coordinating node 对 document 进行路由，将请求转发给对应的 node（有 primary shard）。
- 实际的 node 上的 primary shard 处理请求，然后将数据同步到 replica node。
- coordinating node 如果发现 primary node 和所有 replica node 都搞定之后，就返回响应结果给客户端。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/es-write.png)

#### ES的读数据过程
可以通过 doc id 来查询，会根据 doc id 进行 hash，判断出来当时把 doc id 分配到了哪个 shard 上面去，从那个 shard 去查询。

- 客户端发送请求到任意一个 node，成为 coordinate node。
- coordinate node 对 doc id 进行哈希路由，将请求转发到对应的 node，此时会使用 round-robin 随机轮询算法，在 primary shard 以及其所有 replica 中随机选择一个，让读请求负载均衡。
- 接收请求的 node 返回 document 给 coordinate node。
- coordinate node 返回 document 给客户端。

#### es 搜索数据过程
es 最强大的是做全文检索，就是比如你有三条数据：
```java
java真好玩儿啊
java好难学啊
j2ee特别牛
```
你根据 java 关键词来搜索，将包含 java的 document 给搜索出来。es 就会给你返回：java真好玩儿啊，java好难学啊。
- 客户端发送请求到一个 coordinate node。
- 协调节点将搜索请求转发到所有的 shard 对应的 primary shard 或 replica shard，都可以。
- query phase：每个 shard 将自己的搜索结果（其实就是一些 doc id）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果。
- fetch phase：接着由协调节点根据 doc id 去各个节点上拉取实际的 document 数据，最终返回给客户端。

写请求是写入 primary shard，然后同步给所有的 replica shard；读请求可以从 primary shard 或 replica shard 读取，采用的是随机轮询算法。

#### 更新操作
更新操作其实就是先读然后写
![avatar](https://upload-images.jianshu.io/upload_images/1604849-c438f2d30c8c8fbd.png?imageMogr2/auto-orient/strip|imageView2/2/w/910/format/webp)
更新流程：

客户端将更新请求发给Node1
1.Node1根据文档ID(_id字段)计算出该文档属于分片shard0,而其主分片在Node上，于是将请求路由到Node3
2.Node3从p0读取文档，改变source字段的json内容，然后将修改后的数据在P0重新做索引。如果此时该文档被其他进程修改，那么将重新执行3步骤，这个过程如果超过retryon_confilct设置的重试次数，就放弃。
3.如果Node3成功更新了文档，它将并行的将新版本的文档同步到Node1 Node2的副本分片上重新建立索引，一旦所有的副本报告成功，Node3向被请求的Node1节点返回成功，然后Node1向client返回成功

#### 更新和删除
由于segments是不变的，所以文档不能从旧的segments中删除，也不能在旧的segments中更新来映射一个新的文档版本。取之的是，每一个提交点都会包含一个.del文件，列举了哪一个segment的哪一个文档已经被删除了。 当一个文档被”删除”了，它仅仅是在.del文件里被标记了一下。被”删除”的文档依旧可以被索引到，但是它将会在最终结果返回时被移除掉。

文档的更新同理：当文档更新时，旧版本的文档将会被标记为删除，新版本的文档在新的segment中建立索引。也许新旧版本的文档都会本检索到，但是旧版本的文档会在最终结果返回时被移除。


#### 写数据底层原理
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/es-write-detail.png)
[Elasticsearch基础整理](https://www.jianshu.com/p/e8226138485d)
- **write**:
先写入内存 buffer，在 buffer 里的时候数据是搜索不到的；同时将数据写入 translog 日志文件。
![avatar](https://upload-images.jianshu.io/upload_images/1604849-de89cebe9a0976da.png?imageMogr2/auto-orient/strip|imageView2/2/w/627/format/webp)
这时候还不能会被检索到，而数据必须被refresh到segment后才能被检索到。
- **refresh**:
默认情况下es每隔1s执行一次refresh,太耗性能，可以通过index.refresh_interval来修改这个刷新时间间隔。
整个refresh具体做了如下事情:
1.所有在内存缓冲区的文档被写入到一个新的segment中，但是没有调用fsync，因此数据有可能丢失,此时segment首先被写到内核的文件系统中缓存os cache中
2.segment被打开是的里面的文档能够被见检索到
3.清空内存缓冲区in-memory buffer，清空后如下图
![avatar](https://upload-images.jianshu.io/upload_images/1604849-5e6ef487fe33a943.png?imageMogr2/auto-orient/strip|imageView2/2/w/912/format/webp)
*补充*：
1.每隔 1 秒钟，es 将 buffer 中的数据写入一个新的 segment file，每秒钟会产生一个新的磁盘文件 segment file，这个 segment file 中就存储最近 1 秒内 buffer 中写入的数据。
2.但是如果 buffer 里面此时没有数据，那当然不会执行 refresh 操作，如果 buffer 里面有数据，默认 1 秒钟执行一次 refresh 操作，刷入一个新的 segment file 中。
3.操作系统里面，磁盘文件其实都有一个东西，叫做 os cache，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 os cache，先进入操作系统级别的一个内存缓存中去。只要 buffer 中的数据被 refresh 操作刷入 os cache中，这个数据就可以被搜索到了。
- **flush**:
随着translog文件越来越大时要考虑把内存中的数据刷新到磁盘中，这个过程叫flush
1.把所有在内存缓冲区中的文档写入到一个新的segment中
2.清空内存缓冲区
3.往磁盘里写入commit point信息
4.文件系统的page cache(segments) fsync到磁盘
5.删除旧的translog文件，因此此时内存中的segments已经写入到磁盘中,就不需要translog来保障数据安全了，flush后效果如下
![avatar](https://upload-images.jianshu.io/upload_images/1604849-34ea52e9cc4409e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/582/format/webp)
- **segment合并**：
通过每隔一秒的自动刷新机制会创建一个新的segment，用不了多久就会有很多的segment。segment会消耗系统的文件句柄，内存，CPU时钟。最重要的是，每一次请求都会依次检查所有的segment。segment越多，检索就会越慢。
ES通过在后台merge这些segment的方式解决这个问题。小的segment merge到大的
这个过程也是那些被”删除”的文档真正被清除出文件系统的过程，因为被标记为删除的文档不会被拷贝到大的segment中。
![avatar](https://upload-images.jianshu.io/upload_images/1604849-ea07c8a178cdb69e.png?imageMogr2/auto-orient/strip|imageView2/2/w/861/format/webp)
1.当在建立索引过程中，refresh进程会创建新的segments然后打开他们以供索引。
2.merge进程会选择一些小的segments然后merge到一个大的segment中。这个过程不会打断检索和创建索引。一旦merge完成，旧的segments将被删除。
![avatar](https://upload-images.jianshu.io/upload_images/1604849-2cc44a497a148b1c.png?imageMogr2/auto-orient/strip|imageView2/2/w/799/format/webp)

#### 倒排索引
在搜索引擎中，每个文档都有一个对应的文档 ID，文档内容被表示为一系列关键词的集合。例如，文档 1 经过分词，提取了 20 个关键词，每个关键词都会记录它在文档中出现的次数和出现位置。

那么，倒排索引就是关键词到文档 ID 的映射，每个关键词都对应着一系列的文件，这些文件中都出现了关键词。

举个例子：
有以下文档：
DocId | Doc |
:-: | :-:
1   | 谷歌地图之父跳槽 Facebook
2   | 谷歌地图之父加盟 Facebook
3   | 谷歌地图创始人拉斯离开谷歌加盟 Facebook
4   | 谷歌地图之父跳槽 Facebook 与 Wave 项目取消有关
5   | 谷歌地图之父拉斯加盟社交网站 Facebook

对文档进行分词之后，得到以下倒排索引。
WordId | Word | DocIds
:-: | :-: | :-:
1	  |谷歌	 |1,2,3,4,5
2	  |地图	 |1,2,3,4,5
3	  |之父	 |1,2,4,5
4	  |跳槽	 |1,4
5	  |Facebook	|1,2,3,4,5
6	  |加盟	 |2,3,5
7	  |创始人 |3
8	  |拉斯	  |3,5
9	  |离开	  |3
10	|与	  |4
..	|..	   |..

#### es实现高可用的原理
es 集群多个节点，会自动选举一个节点为 master 节点，这个 master 节点其实就是干一些管理的工作的，比如维护索引元数据、负责切换 primary shard 和 replica shard 身份等。要是 master 节点宕机了，那么会重新选举一个节点为 master 节点。

如果是非 master节点宕机了，那么会由 master 节点，让那个宕机节点上的 primary shard 的身份转移到其他机器上的 replica shard。接着你要是修复了那个宕机机器，重启了之后，master 节点会控制将缺失的 replica shard 分配过去，同步后续修改的数据之类的，让集群恢复正常。

说得更简单一点，就是说如果某个非 master 节点宕机了。那么此节点上的 primary shard 不就没了。那好，master 会让 primary shard 对应的 replica shard（在其他机器上）切换为 primary shard。如果宕机的机器修复了，修复后的节点也不再是 primary shard，而是 replica shard。

其实上述就是 ElasticSearch 作为分布式搜索引擎最基本的一个架构设计。

5）全文检索。

全文检索就是对一篇文章进行索引，可以根据关键字搜索，类似于mysql里的like语句。
全文索引就是把内容根据词的意义进行分词，然后分别创建索引，例如”你们的激情是因为什么事情来的” 可能会被分词成：“你们“，”激情“，“什么事情“，”来“ 等token，这样当你搜索“你们” 或者 “激情” 都会把这句搜出来。

#### ES特点和优势
1）分布式实时文件存储，可将每一个字段存入索引，使其可以被检索到。
2）实时分析的分布式搜索引擎。
分布式：索引分拆成多个分片，每个分片可有零个或多个副本。集群中的每个数据节点都可承载一个或多个分片，并且协调和处理各种操作；
负载再平衡和路由在大多数情况下自动完成。
3）可以扩展到上百台服务器，处理PB级别的结构化或非结构化数据。也可以运行在单台PC上（已测试）
4）支持插件机制，分词插件、同步插件、Hadoop插件、可视化插件等。

#### ES在那些场景下面可以代替传动的db
1.个人以为Elasticsearch作为内部存储来说还是不错的，效率也基本能够满足，在某些方面替代传统DB也是可以的，前提是你的业务不对操作的事性务有特殊要求；而权限管理也不用那么细，因为ES的权限这块还不完善。
2.由于我们对ES的应用场景仅仅是在于对某段时间内的数据聚合操作，没有大量的单文档请求（比如通过userid来找到一个用户的文档，类似于NoSQL的应用场景），所以能否替代NoSQL还需要各位自己的测试。
3.如果让我选择的话，我会尝试使用ES来替代传统的NoSQL，因为它的横向扩展机制太方便了。


#### es 在数据量很大的情况下（数十亿级别）如何提高查询效率啊？
**性能优化的杀手锏——filesystem cache**
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/es-search-process.png)
- 比如说你现在有一行数据。id,name,age .... 30 个字段。但是你现在搜索，只需要根据 id,name,age 三个字段来搜索。如果你傻乎乎往 es 里写入一行数据所有的字段，就会导致说 90% 的数据是不用来搜索的，结果硬是占据了 es 机器上的 filesystem cache 的空间，单条数据的数据量越大，就会导致 filesystem cahce 能缓存的数据就越少。其实，仅仅写入 es 中要用来检索的少数几个字段就可以了，比如说就写入 es id,name,age 三个字段，然后你可以把其他的字段数据存在 mysql/hbase 里，我们一般是建议用 es + hbase 这么一个架构。
- **hbase的特点是适用于海量数据的在线存储，就是对 hbase 可以写入海量数据，但是不要做复杂的搜索，做很简单的一些根据 id 或者范围进行查询的这么一个操作就可以了**。从 es 中根据 name 和 age 去搜索，拿到的结果可能就 20 个 doc id，然后根据 doc id 到 hbase 里去查询每个 doc id 对应的完整的数据，给查出来，再返回给前端。
- 写入 es 的数据最好小于等于，或者是略微大于 es 的 filesystem cache 的内存容量。然后你从 es 检索可能就花费 20ms，然后再根据 es 返回的 id 去 hbase 里查询，查 20 条数据，可能也就耗费个 30ms，可能你原来那么玩儿，1T 数据都放 es，会每次查询都是 5~10s，现在可能性能就会很高，每次查询就是 50ms。

**数据预热**
假如说，哪怕是你就按照上述的方案去做了，es 集群中每个机器写入的数据量还是超过了 filesystem cache 一倍，比如说你写入一台机器 60G 数据，结果 filesystem cache 就 30G，还是有 30G 数据留在了磁盘上。
其实可以做数据预热。
- 举个例子，拿微博来说，你可以把一些大V，平时看的人很多的数据，你自己提前后台搞个系统，每隔一会儿，自己的后台系统去搜索一下热数据，刷到 filesystem cache 里去，后面用户实际上来看这个热数据的时候，他们就是直接从内存里搜索了，很快。
- 或者是电商，你可以将平时查看最多的一些商品，比如说 iphone 8，热数据提前后台搞个程序，每隔 1 分钟自己主动访问一次，刷到 filesystem cache 里去。
- 对于那些你觉得比较热的、经常会有人访问的数据，最好做一个专门的缓存预热子系统，就是对热数据每隔一段时间，就提前访问一下，让数据进入 filesystem cache 里面去。这样下次别人访问的时候，性能一定会好很多。

#### 冷热分离
- es 可以做类似于 mysql 的水平拆分，就是说将大量的访问很少、频率很低的数据，单独写一个索引，然后将访问很频繁的热数据单独写一个索引。最好是将冷数据写入一个索引中，然后热数据写入另外一个索引中，这样可以确保热数据在被预热之后，尽量都让他们留在 filesystem os cache 里，别让冷数据给冲刷掉。

#### document 模型设计
- 对于 MySQL，我们经常有一些复杂的关联查询。在 es 里该怎么玩儿，es 里面的复杂的关联查询尽量别用，一旦用了性能一般都不太好。
- 最好是先在 Java 系统里就完成关联，将关联好的数据直接写入 es 中。搜索的时候，就不需要利用 es 的搜索语法来完成 join 之类的关联搜索了。
- document 模型设计是非常重要的，很多操作，不要在搜索的时候才想去执行各种复杂的乱七八糟的操作。es 能支持的操作就那么多，不要考虑用 es 做一些它不好操作的事情。如果真的有那种操作，尽量在 document 模型设计的时候，写入的时候就完成。另外对于一些太复杂的操作，比如 join/nested/parent-child 搜索都要尽量避免，性能都很差的。

#### 分页性能优化
类似于 app 里的推荐商品不断下拉出来一页一页的

类似于微博中，下拉刷微博，刷出来一页一页的，你可以用 scroll api，关于如何使用，自行上网搜索。

- scroll 会一次性给你生成所有数据的一个快照，然后每次滑动向后翻页就是通过游标 scroll_id 移动，获取下一页下一页这样子，性能会比上面说的那种分页性能要高很多很多，基本上都是毫秒级的。
- 但是，唯一的一点就是，这个适合于那种类似微博下拉翻页的，不能随意跳到任何一页的场景。也就是说，你不能先进入第 10 页，然后去第 120 页，然后又回到第 58 页，不能随意乱跳页。所以现在很多产品，都是不允许你随意翻页的，app，也有一些网站，做的就是你只能往下拉，一页一页的翻。
- 初始化时必须指定 scroll 参数，告诉 es 要保存此次搜索的上下文多长时间。你需要确保用户不会持续不断翻页翻几个小时，否则可能因为超时而失败。
- 除了用 scroll api，你也可以用 search_after 来做，search_after 的思想是使用前一页的结果来帮助检索下一页的数据，显然，这种方式也不允许你随意翻页，你只能一页页往后翻。初始化时，需要使用一个唯一值的字段作为 sort 字段。

#### es 生产集群的部署架构是什么？每个索引的数据量大概有多少？每个索引大概有多少个分片？
但是如果你确实没干过，也别虚，我给你说一个基本的版本，你到时候就简单说一下就好了。
- es 生产集群我们部署了 5 台机器，每台机器是 6 核 64G 的，集群总内存是 320G。
- 我们 es 集群的日增量数据大概是 2000 万条，每天日增量数据大概是 500MB，每月增量数据大概是 6 亿，15G。目前系统已经运行了几个月，现在 es 集群里数据总量大概是 100G 左右。
- 目前线上有 5 个索引（这个结合你们自己业务来，看看自己有哪些数据可以放 es 的），每个索引的数据量大概是 20G，所以这个数据量之内，我们每个索引分配的是 8 个 shard，比默认的 5 个 shard 多了 3 个 shard。
