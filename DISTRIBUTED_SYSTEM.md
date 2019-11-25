# 分布式系统
涉及相关知识

### 分布式事务
事务就是通过它的ACID特性，保证一系列的操作在任何情况下都可以安全正确的执行。
- *Java中我们平时用的最多的就是在service层的增删改方法上添加@Transactional注解，让spring去帮我们管理事务。
它底层会给我们的service组件生成一个对应的proxy动态代理，这样所有对service组件的方法都由它对应的proxy来接管
当proxy在调用对应业务方法比如add()时，proxy就会基于AOP的思想在调用真正的业务方法前执行setAutoCommit（false）打开事务。
然后在业务方法执行完后执行commit提交事务，当在执行业务方法的过程中发生异常时就会执行rollback来回滚事务。*

#### CAP理论
cap理论告诉我们，一个分布式系统不可能同时满足一致性（consistency）,可用性（availability）和分区容错性（partitions problems）
- 一致性是指数据在多个副本之间是否能够保持一致的特性。
- 可用性是指系统提供的服务必须一直处于可用的状态，对于用户的每一个操作请求总是能够在有限的时间内返回结果。
- 分区容错性要求分布式系统在遇到任何网络分区故障时，仍然能够保障对外提供满足一致性和可用性的服务，除非是整个网络都发生了故障。
分区容错性是分布式系统必须要面对和解决的问题，所以系统架构设计师往往根据业务特点在C和A之间找寻平衡。

#### BASE理论
BASE是Basically Availability(基本可用)，Soft status(软状态)，Eventually consistency(最终一致性)三个短语的简称。是对CAP理论中一致性和可用性劝和的结果。**其核心思想是即使无法做到强一致性，但每个应用可以根据自身业务特点，采用适当的方式使系统达到最终一致性**
- 基本可用：系统在出现不可预知的障碍之后，允许损失部分可用性。例子：响应时间上的损失，功能伤的损失（例如大促的降级页面）。
- 弱状态：即允许系统在不同的节点的数据副本之间进行数据同步的过程中存在延迟。
- 最终一致性：系统中所有的副本数据，在经过一段时间的同步后，最终能够达到一个一致的状态。

#### 面对的问题
在分布式系统中，每一个节点都能知道自己在进行的事务操作的结果是成功还是失败，但是却无法直接获得其他分布式节点的操作结果。因此，需要引入一个被称为些调整的组件来统一调度所有分布式节点的执行逻辑。

#### 二阶段提交
常用于计算机网络和数据库领域来保证事务处理过程中的原子性和一致性。*目前，绝大部分的关系型数据库都是采用二阶段提交协议来完成分布式事务处理的*。二阶段协议的整体流程：
**阶段一**
1. 事务询问：协调者向所有的参与者发送事务内容，询问是否可以执行事务提交的操作，并开始等待各个参与者的响应。
2. 执行事务：各个参与者结点执行事务操作，并将undo和Redo信息记入事务日志中。
3. 各参与者向协调者反馈事务询的响应：如果参与者成功执行了事务操作，那么久反馈给协调者yes响应，表示事务可以执行；如果参与者没有成功执行，那么久反馈给协调者No响应，表示事务不可以被执行。
由于上面讲述的内容在形式上近似是协调者组织各个参与者对一次事务操作的投票表态过程，因此二阶段提交协议的阶段一也被称为“投票阶段”，即各个参与者投票表明是否要继续执行接下去的事务提交操作。

**阶段二**
在阶段二中，协调者会根据各个参与者的反馈情况来决定最终是否可以进行事务提交操作，正常情况下，包含以下两种可能。
第一种可能，执行事务提交：假如协调者从所有的参与者获得的反馈都是yes响应，那么就会执行事务提交。
![avatar](https://images2017.cnblogs.com/blog/837877/201709/837877-20170921140232665-672804875.png)
1.发送提交请求:协调者向所有参与者结点发出Commit请求
2.事务提交:参与者接受到Commit请求后，会正式执行事务提交操作，并在完成提交之后释放整个事务执行期间占用的事务资源。
3.反馈事务提交结果:参与者在完成事务提交之后，向协调者发送Ack消息。
4.完成事务:协调者接收到所有参与者反馈的Ack信息后，完成事务。
第二种可能，中断事务：假如任何一个参与者向协调者反馈了No响应，或者在等待超时后，协调者尚无法接收到所有参与者的反馈响应，那么就会中断事务。
![avatar](https://images2017.cnblogs.com/blog/837877/201709/837877-20170921141154837-1451896799.png)
1.发送回滚请求:协调者向所有参与结点发出RollBack请求。
2.事务回滚:参与者接受到RollBack请求后，会利用其在阶段一种记录的Undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源。
3.反馈事务回滚结果:参与者在完成事务回滚之后，向协调者发送Ack消息。
4.中断事务:协调者接收到所有参与者反馈的Ack消息后，完成事务中断。

**二阶段提交的优缺点**
1.优点：原理简单，实现方便。
2.缺点：同步阻塞、单点问题、脑裂、太过保守。
- 同步阻塞：二阶段提交协议存在的最明显也是最大的一个问题就是同步阻塞，这会极大地限制分布式系统的性能。在二阶段提交的执行过程中，所有参与该事务操作的逻辑都处于阻塞状态，也就是说，各个参与者在等待其他参与者响应的过程中，将无法进行其他任何操作。
- 单点问题：可以看出，协调者在二阶段提交协议中起到了非常重要的作用。一旦协调真发生了问题，整个二阶段提交过程将无法运转，更为严重的是，如果协调者是在阶段二中出现问题的话，那么其他参与者将会一直处于锁定事务资源的状态中，而无法继续完成事务操作。
- 数据不一致：在二阶段提交协议的阶段二，即执行事务提交的时候，当协调者向所有参与者发送commit请求之后，发生了局部网络异常或者是协调者在尚未发送完commit请求之前自身发生了崩溃，导致最终只有部分参与者受到了Commit请求。于是这部分收到了Commit请求的参与者就会进行事务的提交，而其他没有收到Commit请求的参与者则无法进行事务提交，于是整个分布式系统便出现了数据不一致现象。
- 太过保守：如果协调者只是参与者进行事务提交询问的过程中，参与者出现故障而导致协调者始终无法获取到所有参与者的响应信息的话，这时协调者只能依靠其自身的超时机制来判断是否需要中断事务，这样的策略显得比较保守。换句话说，二阶段提交协议没有涉及较为完善的容错机制，任何一个结点的失败都会导致整个事务的失败。

#### 三阶段提交
　　3PC，是Three-Phase Commit的缩写，即三阶段提交，是2PC的改进版，其将二阶段提交协议的“提交事务请求”过程一分为二，形成了CanCommit、PreCommit和doCommit三个阶段组成的事务处理协议，其协议涉及如图三所示。
![avatar](https://images2017.cnblogs.com/blog/837877/201709/837877-20170921143225056-919639121.png)
**阶段一：CanCommit**
1.事务询问：协调者向所有的参与者发送一个包含事务内容的canCommit请求，询问是否可以执行事务提交操作，并开始等待参与者的响应。
2.各参与者向协调者反馈事务询问的响应：参与者在接收到来自协调者的canCommit请求后，正常情况下，如果其吱声认为可以顺利执行事务，那么会反馈yes响应，并进入预备状态，否则反馈No响应。
**阶段二：preCommit**
在阶段二中，协调者会根据各个参与者的反馈情况来决定是否可以进行事务的preCommit操作，正常情况下，包含两种可能。
第一种可能执行事务预提交，假如协调者从所有的参与者获得的反馈都是Yes响应，那么就会执行事务预提交
1.发送预提交请求：协调者向所有参与者结点发出preCommit的请求，并进入Prepared阶段
2.事务预提交：参与者接受到preCommit请求后，会执行事务操作，并将Undo和Redo信息记录到事务日志中。
3.各参与者向协调会怎反馈事务执行的响应：如果参与者成功执行了事务操作，那么就会反馈给协调者Ack响应，同时等待最终的指令：提交（commit）或终止（abort）.
第二种可能中断事务，假如任何一个参与者向协调者反馈了No响应，或者在等待超时后，协调者尚无法接收到所有参与者的反馈响应，那么就会终端事务。
1.发送中断请求：协调者向所有参与者结点发出abort请求。
2.中断事务：无论是收到来自协调者的abort请求，或者是在等待协调者请求过程中出现超时，参与者都会中断事务。
**阶段三：doCommit**
该阶段将进行真正的事务提交，会存在以下两种可能的情况。
第一种可能执行提交
1.发送提交请求：进入这一阶段，假设协调者处于正常的工作状态，并且他接受到了来自所有参与者的Ack响应，那么它将从“预提交”状态转换到“提交”状态，并向所有的参与者发送doCommit请求。
2.事务提交：参与者接受到doCommit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源。
3.反馈事务提交结果：参与者在完成事务提交后，向协调者发送Ack消息。
4.完成事务：协调者接受到所有参与者反馈的Ack消息后，完成事务。
第二种可能中断提交，进入这一阶段，假设协调真处于正常工作状态，并且有任意一个参与者向协调者反馈了No响应，或者在等待超时后，协调者尚无法接受到所有参与者的反馈响应，那么就会中断事务。
1.发送中断请求：协调者向所有参与者结点发送abort请求
2.事务回滚：参与者接收到abort请求后，会利用其在阶段二中记录的Undo信息来执行事务回滚操作，并在完成回滚后释放整个事务执行期间占用的资源。
3.反馈事务回滚结果：参与者在完成事务回滚会后，向协调者发送Ack消息。
4.中断事务：协调者接收到所有参与者反馈的Ack消息后，中断事务。
需要注意的是，一旦进入阶段三，有可能出现以下两种故障。
　　(1)协调者出现问题
　　(2)协调者和所有参与者之间的网络出现故障。
无论出现那种情况，最终都会到时参与者无法及时接受到来自协调者的doCommit或是abort命令，针对这种异常情况，参与者都会在等待超时后继续进行执行事务提交。
**三阶段提交的优缺点**
三阶段提交协议的优点：三阶段提交协议在去阻塞的同时也引入了新的问题，那就是在参与者接受到preCommit消息后，如果网络出现了分区，此时协调者所在的节点和参与者无法进行正常的网通信，在这种情况下，该参与者仍然会进行事务的提交，者必然出现数据的不一致。

#### 三阶段提交是“非阻塞”协议。如何解决二阶段提交的阻塞问题
三阶段提交在两阶段提交的第一阶段与第二阶段之间插入了一个准备阶段，
使得原先在两阶段提交中，参与者在投票之后，由于协调者发生崩溃或错误，
而导致参与者处于无法知晓是否提交或者中止的“不确定状态”所产生的可能相当长的延时的问题得以解决。 举例来说，假设有一个决策小组由一个主持人负责与多位组员以电话联络方式协调是否通过一个提案，以两阶段提交来说，主持人收到一个提案请求，打电话跟每个组员询问是否通过并统计回复，然后将最后决定打电话通知各组员。
要是主持人在跟第一位组员通完电话后失忆，而第一位组员在得知结果并执行后老人痴呆，那么即使重新选出主持人，也没人知道最后的提案决定是什么，也许是通过，也许是驳回，不管大家选择哪一种决定，都有可能与第一位组员已执行过的真实决定不一致，老板就会不开心认为决策小组沟通有问题而解雇。
三阶段提交即是引入了另一个步骤，主持人打电话跟组员通知请准备通过提案，以避免没人知道真实决定而造成决定不一致的失业危机。
为什么能够解决二阶段提交的问题呢？
回到刚刚提到的状况，在主持人通知完第一位组员请准备通过后两人意外失忆，即使没人知道全体在第一阶段的决定为何，全体决策组员仍可以重新协调过程或直接否决，不会有不一致决定而失业。
那么当主持人通知完全体组员请准备通过并得到大家的再次确定后进入第三阶段，
当主持人通知第一位组员请通过提案后两人意外失忆，这时候其他组员再重新选出主持人后，
仍可以知道目前至少是处于准备通过提案阶段，表示第一阶段大家都已经决定要通过了，此时便可以直接通过。

#### 实际场景中如何解决分布式事务问题
[本节参考文章](https://github.com/doocs/advanced-java/blob/master/docs/distributed-system/distributed-transaction.mdÂ)
分布式事务的实现主要有以下 5 种方案：
- XA 方案
- TCC 方案
- 本地消息表
- 可靠消息最终一致性方案
- 最大努力通知方案
##### 两阶段提交方案/XA方案
这种分布式事务方案，比较适合单块应用里，跨多个库的分布式事务，而且因为严重依赖于数据库层面来搞定复杂的事务，效率很低，绝对不适合高并发的场景。这个方案，我们很少用，一般来说某个系统内部如果出现跨多个库的这么一个操作，是不合规的。我可以给大家介绍一下， 现在微服务，一个大的系统分成几十个甚至几百个服务。一般来说，我们的规定和规范，是要求每个服务只能操作自己对应的一个数据库。如果你要操作别人的服务的库，你必须是通过调用别的服务的接口来实现，绝对不允许交叉访问别人的数据库。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/distributed-transaction-XA.png)
*JTA实现分布式事务*

##### TCC方案
TCC 的全称是：Try、Confirm、Cancel。
- Try 阶段：这个阶段说的是对各个服务的资源做检测以及对资源进行锁定或者预留。
- Confirm 阶段：这个阶段说的是在各个服务中执行实际的操作。
- Cancel 阶段：如果任何一个服务的业务方法执行出错，那么这里就需要进行补偿，就是执行已经执行成功的业务逻辑的回滚操作。（把那些执行成功的回滚）
这种方案说实话几乎很少人使用，我们用的也比较少，但是也有使用的场景。因为这个事务回滚实际上是严重依赖于你自己写代码来回滚和补偿了，会造成补偿代码巨大，非常之恶心。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/distributed-transaction-TCC.png)

##### 本地消息表
这个大概意思是这样的：
1.A 系统在自己本地一个事务里操作同时，插入一条数据到消息表；
2.接着 A 系统将这个消息发送到 MQ 中去；
3.  B 系统接收到消息之后，在一个事务里，往自己本地消息表里插入一条数据，同时执行其他的业务操作，如果这个消息已经被处理过了，那么此时这个事务会回滚，这样保证不会重复处理消息；
4.B 系统执行成功之后，就会更新自己本地消息表的状态以及 A 系统消息表的状态；
5.如果B系统处理失败了，那么就不会更新消息表状态，那么此时 A 系统会定时扫描自己的消息表，如果有未处理的消息，会再次发送到 MQ 中去，让 B 再次处理；
6.这个方案保证了最终一致性，哪怕 B 事务失败了，但是 A 会不断重发消息，直到 B 那边成功为止。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/distributed-transaction-local-message-table.png)
这个方案说实话最大的问题就在于严重依赖于数据库的消息表来管理事务啥的，如果是高并发场景咋办呢？咋扩展呢？所以一般确实很少用。

##### 可靠消息最终一致性方案
这个的意思，就是干脆不要用本地的消息表了，直接基于 MQ 来实现事务。比如阿里的 RocketMQ 就支持消息事务。
大概的意思就是：
1. A 系统先发送一个 prepared 消息到 mq，如果这个 prepared 消息发送失败那么就直接取消操作别执行了；
2. 如果这个消息发送成功过了，那么接着执行本地事务，如果成功就告诉 mq 发送确认消息，如果失败就告诉 mq 回滚消息；
3. 如果发送了确认消息，那么此时 B 系统会接收到确认消息，然后执行本地的事务；
4. mq 会自动定时轮询所有 prepared 消息回调你的接口，问你，这个消息是不是本地事务处理失败了，所有没发送确认的消息，是继续重试还是回滚？一般来说这里你就可以查下数据库看之前本地事务是否执行，如果回滚了，那么这里也回滚吧。这个就是避免可能本地事务执行成功了，而确认消息却发送失败了。
5. 这个方案里，要是系统 B 的事务失败了咋办？重试咯，自动不断重试直到成功，如果实在是不行，要么就是针对重要的资金类业务进行回滚，比如 B 系统本地回滚后，想办法通知系统 A 也回滚；或者是发送报警由人工来手工回滚和补偿。
6. 这个还是比较合适的，目前国内互联网公司大都是这么玩儿的，要不你举用 RocketMQ 支持的，要不你就自己基于类似 ActiveMQ？RabbitMQ？自己封装一套类似的逻辑出来，总之思路就是这样子的。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/distributed-transaction-reliable-message.png)

##### 最大努力通知方案
1. 系统 A 本地事务执行完之后，发送个消息到 MQ；
2. 这里会有个专门消费 MQ 的最大努力通知服务，这个服务会消费 MQ 然后写入数据库中记录下来，或者是放入个内存队列也可以，接着调用系统 B 的接口；
3. 要是系统 B 执行成功就 ok 了；要是系统 B 执行失败了，那么最大努力通知服务就定时尝试重新调用系统 B，反复 N 次，最后还是不行就放弃。














### 分布式主键生成
#### 基于数据库的自增id
这个就是说你的系统里每次得到一个 id，都是往一个库的一个表里插入一条没什么业务含义的数据，然后获取一个数据库自增的一个 id。拿到这个 id 之后再往对应的分库分表里去写入。
- 优点：方便简单，谁都会用
- 缺点：缺点就是单库生成自增 id，要是高并发的话，就会有瓶颈的。

#### 设置数据库 sequence 或者表自增字段步长
可以通过设置数据库 sequence 或者表的自增字段步长来进行水平伸缩。
比如说，现在有 8 个服务节点，每个服务节点使用一个 sequence 功能来产生 ID，每个 sequence 的起始 ID 不同，并且依次递增，步长都是 8。
![avatar](https://raw.githubusercontent.com/doocs/advanced-java/master/images/database-id-sequence-step.png)

#### UUID
好处就是本地生成，不要基于数据库来了；不好之处就是，UUID 太长了、占用空间大，作为主键性能太差了；更重要的是，UUID 不具有有序性，会导致 B+ 树索引在写的时候有过多的随机写操作（连续的 ID 可以产生部分顺序写），还有，由于在写的时候不能产生有顺序的 append 操作，而需要进行 insert 操作，将会读取整个 B+ 树节点到内存，在插入这条记录后会将整个节点写回磁盘，这种操作在记录占用空间比较大的情况下，性能下降明显。
**如果使用文件名称，或者编号适合UUID,但是不适合主键。**

#### 获取系统当前时间
  这个就是获取当前时间即可，但是问题是，并发很高的时候，比如一秒并发几千，会有重复的情况，这个是肯定不合适的。基本就不用考虑了。

#### snowflake 算法
snowflake 算法是 twitter 开源的分布式 id 生成算法，采用 Scala 语言实现，是把一个 64 位的 long 型的 id，1 个 bit 是不用的，用其中的 41 bit 作为毫秒数，用 10 bit 作为工作机器 id，12 bit 作为序列号。
- 1 bit：不用，为啥呢？因为二进制里第一个 bit 为如果是 1，那么都是负数，但是我们生成的 id 都是正数，所以第一个 bit 统一都是 0。
- 41 bit：表示的是时间戳，单位是毫秒。41 bit 可以表示的数字多达 2^41 - 1，也就是可以标识 2^41 - 1 个毫秒值，换算成年就是表示**69年**的时间。
- 10 bit：记录工作机器 id，代表的是这个服务最多可以部署在 2^10台机器上哪，也就是1024台机器。但是 10 bit 里 5 个 bit 代表机房 id，5 个 bit 代表机器 id。意思就是最多代表 2^5个机房（32个机房），每个机房里可以代表 2^5 个机器（32台机器）。
- 12 bit：这个是用来记录同一个毫秒内产生的不同 id，12 bit 可以代表的最大正整数是 2^12 - 1 = 4096，也就是说可以用这个 12 bit 代表的数字来区分同一个毫秒内的 4096 个不同的 id。
snowflake算法的实现：
```java
public class IdWorker {

    private long workerId;
    private long datacenterId;
    private long sequence;

    public IdWorker(long workerId, long datacenterId, long sequence) {
        // sanity check for workerId
        // 这儿不就检查了一下，要求就是你传递进来的机房id和机器id不能超过32，不能小于0
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(
                    String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (datacenterId > maxDatacenterId || datacenterId < 0) {
            throw new IllegalArgumentException(
                    String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        }
        System.out.printf(
                "worker starting. timestamp left shift %d, datacenter id bits %d, worker id bits %d, sequence bits %d, workerid %d",
                timestampLeftShift, datacenterIdBits, workerIdBits, sequenceBits, workerId);

        this.workerId = workerId;
        this.datacenterId = datacenterId;
        this.sequence = sequence;
    }

    private long twepoch = 1288834974657L;

    private long workerIdBits = 5L;
    private long datacenterIdBits = 5L;

    // 这个是二进制运算，就是 5 bit最多只能有31个数字，也就是说机器id最多只能是32以内
    private long maxWorkerId = -1L ^ (-1L << workerIdBits);

    // 这个是一个意思，就是 5 bit最多只能有31个数字，机房id最多只能是32以内
    private long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    private long sequenceBits = 12L;

    private long workerIdShift = sequenceBits;
    private long datacenterIdShift = sequenceBits + workerIdBits;
    private long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private long sequenceMask = -1L ^ (-1L << sequenceBits);

    private long lastTimestamp = -1L;

    public long getWorkerId() {
        return workerId;
    }

    public long getDatacenterId() {
        return datacenterId;
    }

    public long getTimestamp() {
        return System.currentTimeMillis();
    }

    public synchronized long nextId() {
        // 这儿就是获取当前时间戳，单位是毫秒
        long timestamp = timeGen();

        if (timestamp < lastTimestamp) {
            System.err.printf("clock is moving backwards.  Rejecting requests until %d.", lastTimestamp);
            throw new RuntimeException(String.format(
                    "Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        if (lastTimestamp == timestamp) {
            // 这个意思是说一个毫秒内最多只能有4096个数字
            // 无论你传递多少进来，这个位运算保证始终就是在4096这个范围内，避免你自己传递个sequence超过了4096这个范围
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }

        // 这儿记录一下最近一次生成id的时间戳，单位是毫秒
        lastTimestamp = timestamp;

        // 这儿就是将时间戳左移，放到 41 bit那儿；
        // 将机房 id左移放到 5 bit那儿；
        // 将机器id左移放到5 bit那儿；将序号放最后12 bit；
        // 最后拼接起来成一个 64 bit的二进制数字，转换成 10 进制就是个 long 型
        return ((timestamp - twepoch) << timestampLeftShift) | (datacenterId << datacenterIdShift)
                | (workerId << workerIdShift) | sequence;
    }

    private long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    private long timeGen() {
        return System.currentTimeMillis();
    }

    // ---------------测试---------------
    public static void main(String[] args) {
        IdWorker worker = new IdWorker(1, 1, 1);
        for (int i = 0; i < 30; i++) {
            System.out.println(worker.nextId());
        }
    }

}

```
### 分布式限流
#### 限流算法
##### 计数器算法
计数器算法的思想很简单，每当一个请求到来时，我们就将计数器加一，当计数器数值超过阈值后，就拒绝余下请求。会有所谓的突刺现象，一秒钟特别忙，另一秒钟特别的闲。



### 分布式锁
#### 用数据库实现分布式锁
- 基于表记录
  当我们想要获得锁的时候，就可以在该表中增加一条记录，想要释放锁的时候就删除这条记录。

```mysql
CREATE TABLE `database_lock` (
	`id` BIGINT NOT NULL AUTO_INCREMENT,
	`resource` int NOT NULL COMMENT '锁定的资源',
	`description` varchar(1024) NOT NULL DEFAULT "" COMMENT '描述',
	PRIMARY KEY (`id`),
	UNIQUE KEY `uiq_idx_resource` (`resource`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据库分布式锁表';
```
当我们想要获得锁时，可以插入一条数据：
```mysql
INSERT INTO database_lock(resource, description) VALUES (1, 'lock');
```
当需要释放锁的时，可以删除这条数据：
```mysql
DELETE FROM database_lock WHERE resource=1;
```
这种实现方式非常的简单，但是需要注意以下几点：
- 这种锁没有失效时间，一旦释放锁的操作失败就会导致锁记录一直在数据库中，其它线程无法获得锁。这个缺陷也很好解决，比如可以做一个定时任务去定时清理。
- 这种锁的可靠性依赖于数据库。建议设置备库，避免单点，进一步提高可靠性。
- 这种锁是非阻塞的，因为插入数据失败之后会直接报错，想要获得锁就需要再次操作。如果需要阻塞式的，可以弄个for循环、while循环之类的，直至INSERT成功再返回。
- 这种锁也是非可重入的，因为同一个线程在没有释放锁之前无法再次获得锁，因为数据库中已经存在同一份记录了。想要实现可重入锁，可以在数据库中添加一些字段，比如获得锁的主机信息、线程信息等，那么在再次获得锁的时候可以先查询数据，如果当前的主机信息和线程信息等能被查到的话，可以直接把锁分配给它。

#### 乐观锁
乐观锁大多数是基于数据版本(version)的记录机制实现的。在更新过程中，会对版本号进行比较，如果是一致的，没有发生改变，则会成功执行本次操作；如果版本号不一致，则会更新失败。
库存模型用下面的一张表optimistic_lock来表述，参考如下：
```mysql
CREATE TABLE `optimistic_lock` (
	`id` BIGINT NOT NULL AUTO_INCREMENT,
	`resource` varchar(64) NOT NULL COMMENT '锁定的资源',
  `resource_num` int NOT NULL COMMENT `资源的库存`,
	`version` int NOT NULL COMMENT '版本信息',
	`created_at` datetime COMMENT '创建时间',
	`updated_at` datetime COMMENT '更新时间',
	`deleted_at` datetime COMMENT '删除时间',
	PRIMARY KEY (`id`),
	UNIQUE KEY `uiq_idx_resource` (`resource`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据库分布式锁表';
```
在使用乐观锁之前要确保表中有相应的数据，比如：
```mysql
INSERT INTO optimistic_lock(resource, resource_num,version, created_at, updated_at) VALUES(20, 1, CURTIME(), CURTIME());
```
操作流程如下：
- STEP1 - 获取资源： SELECT resource, version FROM optimistic_lock WHERE id = 1
- STEP2 - 执行业务逻辑
- STEP3 - 更新资源：UPDATE optimistic_lock SET resource = resource -1, version = version + 1 WHERE id = 1 AND version = oldVersion
我们通过上述sql语句还可以看到，数据库锁都是作用于同一行数据记录上，这就导致一个明显的缺点，在一些特殊场景，如大促、秒杀等活动开展的时候，大量的请求同时请求同一条记录的行锁，会对数据库产生很大的写压力。所以综合数据库乐观锁的优缺点，乐观锁比较适合并发量不高，并且写操作不频繁的场景。

#### 悲观锁
在查询语句后面增加FOR UPDATE，数据库会在查询过程中给数据库表增加悲观锁，也称排他锁。具体步骤如下：
  - STEP1 - 获取锁：SELECT * FROM database_lock WHERE id = 1 FOR UPDATE;。
- STEP2 - 执行业务逻辑。
- STEP3 - 释放锁：COMMIT。
在使用悲观锁的同时，我们需要注意一下锁的级别。MySQL InnoDB引起在加锁的时候，只有明确地指定主键(或索引)的才会执行行锁 (只锁住被选取的数据)，否则MySQL 将会执行表锁(将整个数据表单给锁住)。
缺点很明显，即每次请求都会额外产生加锁的开销且未获取到锁的请求将会阻塞等待锁的获取，在高并发环境下，容易造成大量请求阻塞，影响系统可用性。另外，悲观锁使用不当还可能产生死锁的情况。
####基于数据库实现分布式锁的缺点
- 锁没有失效时间,一旦解锁操作失败,就会导致锁记录一直在数据库中,其他线程无法再获得到锁
- 锁是非阻塞的,数据的insert操作,一旦插入失败就会直接报错。没有获得锁的线程并不会进入排队队列
  要想再次获得锁就要再次触发获得锁操作
- 锁是非重入的,同一个线程在没有释放锁之前无法再次获得该锁

#### 用zk实现分布式锁
假如当前有一个父节点为/lock,我们可以在这个父节点下面创建子节；zookeeper提供了一个可选的有序特性
例如我们可以创建子节点“/lock/node-”并且指明有序,那么zookeeper在生成子节点时会根据当前的子节点数量自动添加整数序号
也就是说如果是第一个创建的子节点,那么生成的子节点为/lock/node-0000000000,下一个节点则为/lock/node-0000000001,依次类推

临时节点：客户端可以建立一个临时节点,在会话结束或者会话超时后,zookeeper会自动删除该节点

事件监听：在读取数据时,我们可以同时对节点设置事件监听,当节点数据或结构变化时,zookeeper会通知客户端
当前zookeeper有如下四种事件：
1）节点创建
2）节点删除
3）节点数据修改
4）子节点变更

获取分布式锁的流程  ------- 假设所空间的根节点为/lock

1.客户端连接zookeeper,并在/lock下创建临时的且有序的子节点
第一个客户端对应的子节点为/lock/lock-0000000000,第二个为/lock/lock-0000000001,以此类推

2.--避免羊群效应--客户端获取/lock下的子节点列表,判断自己创建的子节点是否为当前子节点列表中序号最小的子节点,
如果是则认为获得锁,否则监听刚好在自己之前一位的子节点删除消息,获得子节点变更通知后重复此步骤直至获得锁

3.实行业务代码

4.流程完成后,删除对应的子节点并释放锁  （Watch机制）
对应分布式开源包Curator,就是使用分布式锁实现的基于zookeeper分布式锁

#### 使用redis实现分布式锁
一般是使用 setnx(set if not exists) 指令，只允许被一个客户端占坑。先来先占， 用完了，再调用 del 指令释放茅坑。
```java
// 这里的冒号:就是一个普通的字符，没特别含义，它可以是任意其它字符，不要误解
> setnx lock:codehole true
OK
... do something critical ...
> del lock:codehole
(integer) 1
```
##### 问题1-超时问题
超时问题,通过设计超时时间，随机版本号，lua脚本保证匹配删除问题的原子性操作；
##### 问题2-可重入性
基于ThreadLocal 和引用计数保证可重入性
##### 问题3-集群failover
  主节点挂掉时，从节点会取而代之，客户端上却并没有明显感知。原先第一个客户端在主节点中申请成功了一把锁，但是这把锁还没有来得及同步到从节点，主节点突然挂掉了。然后从节点变成了主节点，这个新的节点内部没有这个锁，所以当另一个客户端过来请求加锁时，立即就批准了。这样就会导致系统中同样一把锁被两个客户端同时持有，不安全性由此产生。
加锁时，它会向过半节点发送 set(key, value, nx=True, ex=xxx) 指令，只要过半节点 set 成功，那就认为加锁成功。释放锁时，需要向所有节点发送 del 指令。不过 Redlock 算法还需要考虑出错重试、时钟漂移等很多细节问题，同时因为 Redlock 需要向多个节点进行读写，意味着相比单实例 Redis 性能会下降一些。
##### redlock使用场景
如果你很在乎高可用性，希望挂了一台 redis 完全不受影响，那就应该考虑 redlock。不过代价也是有的，需要更多的 redis 实例，性能也下降了，代码上还需要引入额外的 library，运维上也需要特殊对待，这些都是需要考虑的成本，使用前请再三斟酌。
redlock流程，为了取到锁，客户端应该执行以下操作:
  - 获取当前Unix时间，以毫秒为单位。
  - 依次尝试从5个实例，使用相同的key和具有唯一性的value（例如UUID）获取锁。当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。
  - 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（N/2+1，这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
  - 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
  - 如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）。

  redis分布式锁的发展

```mermaid
  graph LR
  B(使用分布式锁setnx) --> C(为了解决主从failover的问题redlock)
  C --> D(redlock严格的执行时序性)
```

[马丁·科勒普曼关于redlock的分析](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) 中介绍redlock是一个严格要求时序性的算法
扩展
- jvm的stw
