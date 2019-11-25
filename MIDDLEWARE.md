# 中间件

## zookeeper

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
