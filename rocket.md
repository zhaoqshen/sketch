Kafka和RocketMQ都是基于发布/订阅模式的消息系统，但是它们的架构设计有所不同。

Kafka的架构设计比较简单，主要由生产者、消费者和Kafka集群三个组件组成。生产者将消息发布到Kafka集群中的Broker节点，然后消费者从Broker节点中获取消息进行消费。Kafka的数据模型是基于Topic和Partition的，每个Topic可以有多个Partition，每个Partition可以在多个Broker节点上复制，保证数据的高可用性。

RocketMQ的架构设计比较复杂，主要由Namesrv、Broker和Producer/Consumer三个角色组成。Namesrv主要负责服务注册和发现，Broker节点负责存储和传输消息，Producer和Consumer分别将消息发送到和从Broker节点中获取消息。RocketMQ也是基于Topic和Partition（Queen）的数据模型，但它采用了一种主从复制的机制，确保了数据的高可用性和容错性

Kafka和RocketMQ都是高可靠性的消息系统，但是它们的可靠性也有所不同。

在数据可靠性方面，Kafka表现更加出色。Kafka采用了多副本机制，每个Partition都有多个副本，当某个Broker节点失效时，可以通过其他副本来保证数据的可用性。而RocketMQ采用的是主从复制机制，当主节点失效时，需要进行主节点选举才能保证数据的可用性，这可能会导致一定的延迟。

在数据一致性方面，Kafka也具有更好的表现。Kafka采用了基于Zookeeper的分布式协调机制，能够确保数据在Producer和Consumer之间的顺序性。而RocketMQ则需要在Producer端对消息进行排序，然后再发送到Broker节点中，这可能会对性能造成一定的影响。

在消息事务方面，RocketMQ的表现要优于Kafka。RocketMQ提供了完整的消息事务机制，能够保证消息在发送和接收过程中的一致性和可靠性。而Kafka并没有提供官方的事务支持，需要开发者自行处理。

在故障恢复方面，Kafka具有更好的表现。Kafka支持自动的故障转移和数据复制机制，能够快速地恢复节点的可用性，保证数据的连续性。而RocketMQ需要手动进行主从切换，可能需要一定的人工干预。



Message:消息传输信息的物理载体生产和消费的最小单位每条消息必须属于一个topic rocketmq 中每条消息都具有唯一的ID且可以带具有业务标示的key

topic ：表示同一类消息的集合每个主题包含若干条消息。是roketmq的最小订阅单位。

quee ： 消息队列是组成topic 的最小单元 一个topic 对应多个Queen。topic 是逻辑概念Queen 是物理存储。consumer 在消费topic 时实际是拉去的Queen的信息

tag：为消息标志用于区分同一主题下的不同类型的消息消费者可以根据不同的tag来实现不同的业务逻辑处理

productgroup ？同一类的消费者组，发送的逻辑相同。如果发送的是】事务消息且原始的生产者崩溃之后服务器会联系同一生产者组的其他生产者提供提交或者消费回溯

。消费者组的消费者实例必须订阅完全相同的topic

kafka的master 和slave 在同一台broker机器上。broker机器上有多个分区每个分区上的master和slave都是选取产生。broker机器具有双重的身份即是是Master分区也是slave分区

rocket上的master和slave不在同一个分区，每台broker不事master就是slave，身份是在配置文件中预定义好的

Broker 向每个nameservier 注册路由信息。namesever与每个broker保持长链接。间隔30s从路由注册表中的故障机器移除。

顺序消息

+ 设置成同步发送

+ 发送到同一个对列

  + messagequeeselect 

  + selectmessagequeebyhash 

rocket 发送3种方式 

	+ 同步  同步等待broker 返回结果 支持失败重试
	+ 异步  不阻塞主线程  不支持失败重试
	+ 单向  不阻塞主线程  不支持失败重试 不支持回调

普通消息发送 采用轮询和故障规避机制 默认采用轮询

+ 轮询 是路由信息topicpushinfo 维护了一个计数器sendwhichqueen 弊端有可能轮询到故障的broker

RoketMq 支持两种消息模式：

+ 广播消费 每条休息都是被消费者组中的消费者消费
+ 集群消费 集群中只会被消费一次

顺序消费时要先获取到processQueen中的独占锁消费成功后会向broker中更新消费节点的信息。并发消费没有锁竞争

消费消息有三种模式

`at-most-once`：消息投递后不论是否成功都不会重复投递。有可能导致消息未被消费。

`at-least-once`:消息投递后向服务器返回ack没有消费者一定不会返回ack、服务器没有返回ack 则进行再次投递。这可能会导致重复消费rocket通过ack来确保至少被消费一次。

`exactly-only-once`:发送阶段不允许重复发送。消费阶段不允许重复消费

事务消息。 2阶段提交

事务消息发送步骤：
1. 发送方将半事务消息发送至RocketMQ服务端。
2. RocketMQ服务端将消息持久化之后，向发送方返回Ack确认消息已经发送成功。由于消息为半事务消息，在未收到生产者对该消息的二次确认前，此消息被标记成“暂不能投递”状态。
3. 发送方开始执行本地事务逻辑。
4. 发送方根据本地事务执行结果向服务端提交二次确认（Commit 或是 Rollback），服务端收到Commit 状态则将半事务消息标记为可投递，订阅方最终将收到该消息；服务端收到 Rollback 状态则删除半事务消息，订阅方将不会接受该消息。

事务消息回查步骤：
1. 在断网或者是应用重启的特殊情况下，上述步骤4提交的二次确认最终未到达服务端，经过固定时间后服务端将对该消息发起消息回查。
2. 发送方收到消息回查后，需要检查对应消息的本地事务执行的最终结果。
3. 发送方根据检查得到的本地事务的最终状态再次提交二次确认，服务端仍按照步骤4对半事务消息进行操作。

Roecket 高吞吐量

数据存储的核心有两部分组成CommitLog 

数据存储文件和consumerQueen 消费对列文件生产者将消息发送broker中。broker 会把所有的消息文件存储到CommitLog中在由ommitLog 转发到consumerQueen提供给各个消费者消费

Ç文件是UN年初消息的数据文件所有的topic的的信息。消息数据写入CommitLog 是加锁串行写入

Roecket为了保证消息发送的高吞吐量使用单个文件保持topic信息。从而保证消息存储完全是顺序写。但是这样给文件读带来困难。

当消息到达CommitLog会通过异步线程几乎实时的转发给消费对列文件每个CommitLog的大小默认是1g写满后再写新文件。文件命名按照总字节的偏移量的offset来命名，文件固定命名为20为不足20位前面补0.

是负责存储消息队列的文件。在消息写入CommitLo时会异步转发到consumerQueen然后提供consumer消费

consumerQueen存储的数据位CommitLog的offset 消息大小size 消息tag的hash。

每个topic在broker中应多个Queen。默认是4个消费对列的quee。每条大小默认20b。默认每个文件存储30w条记录命名长度也是20位不足20位前面补0。

在集群模式下broker 会记录每个消费对列的消费的offset定位到consumerQueen里的的应记录并通过CommitLog offset 来定位到

CommitLog中的该条信息。

 消息跳跃读取

+ 检查要读取的数据是不是在pageCash中
+ 如果没有命中Cache中操作系统就要从磁盘中读取对应的数据页到内存中并且将该数据页的之后的连续几页都读取到Cache中在将所需要的数据返回给应用
+ 如果命中。上次缓存的数据有效。操作系统默认是顺序度盘则继续扩大缓存数据范围

消息发送的高可用：namesever 每隔10 秒就会扫描所有的bokker信息。心跳超过120s才认为boker不可用解决发送失败的方式是重试机制和故障延迟机制，

消息存储机制：消息持久化（刷盘）主从复制 读写分离

消息消费高可用：消费重试机制 和ack

重试机制 最多重试3次 如果是RonoteException 是MQclintExcption  会重试 Broker 和线程的IntercptExtion异常不会重试 且重新抛出异常        

故障规避机制：

+  故障规避机制是用来解决broker出现了故障生产者不能及时感知导致消息发送失败的问题。默认是不开启的。如果在开启的状态下消息发送失败的broker暂时排除到选则列表之外。规避的时间是衰减的如果broker 一直不可用会被nameserver检测到并在更新生产者路由信息的时候排除掉。
+ 判断消息队列是否在消息队列中不再故障列表中代表可用
+ 故障列failtiemTable 中判断当前的时间是否大于故障规避的开始时间failtiemTable 存储了响应时间故障规避开始的时间0代表没有故障

同步刷盘异步刷盘

putRequest 是提交任务 request.waitForfulish 会同步等待刷盘任务完成

分为两个队列，写队列和读队列 在刷盘之前会把写队列的数据放到读队列。刷盘的时候依次读取读队列的数据写入磁盘写入完成之后情

读写任务分开的原因是刷盘是不影响其他任务提交到列表

异步刷盘有分为有缓冲区没有缓冲区

不开启：没间隔500毫秒尝试去刷盘。者间隔500ms仅仅是尝试实际刷盘还得满足一些前提条件，即距离上次刷盘时间超过10s或者写入内存数据超过4页（16kb）。

开启：Roeket 会申请一块和CommitLog大小相同大小昨缓冲池。数据会先写入缓冲池。提交线程每隔500ms尝试体检到待刷盘。使用缓冲池的原因是多条信息合并写入从而提高io性能