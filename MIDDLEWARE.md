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








## RPC
RPC要解决的两个问题：
1. 解决分布式系统中，服务之间的调用问题。
2. 远程调用时，要能够像本地调用一样方便，让调用者感知不到远程调用的逻辑。
**rpc 是什么？就是socket 加动态代理。**

### dubbo


## Elasticsearch
