## Rabbit 

+ 如何保证高可用

  + 普通集群模式
    + queue 的数据放在一个节点上
    + 其他的节点放的是queue的元信息（配置信息。所在节点）、
    + 消费者所链接的节点没有queue数据信息那么该节点会把通过元数据信息把该queue的消息拉过来。
      + 提高吞吐量而已
      + 缺点：mq内部大量的消息传输。没有可用性保障（单点故障）。
  + 镜像集群模式
    + 元数据。消息数据在每个节点都存在queue完整的数据
    + 不是分布式。每个节点都要queue的完整数据。
    + 管理配置平台。可以配置一个策略。

  > 三种模式：单机模式，普通集群模式，镜像集群

## kafka高可用

+ 每台机器都会有一个broker进程（一个kafka节点）0.8之前没有ha机制
+ 副本机制每个partition都有一个副本。多个副本会选举出一个leader对外服务（读写）
+ leader会把数据同步到flower



## kafka

 从本质上来说，kafka都是采用异步的方式来发送消息到broker，但是kafka并不是每次发送消息都会直 接发送到broker上，而是把消息放到了一个发送队列中，然后通过一个后台线程不断从队列取出消息进 行发送，发送成功后会触发callback。kafka客户端会积累一定量的消息统一组装成一个批量消息发送出 去，触发条件是前面提到的batch.size和linger.ms

+ **batch.size**

  生产者发送多个消息到broker上的同一个分区时，为了减少网络请求带来的性能开销，通过批量的方式 来提交消息，可以通过这个参数来控制批量提交的字节数大小，默认大小是16384byte,也就是16kb， 意味着当一批消息大小达到指定的batch.size的时候会统一发送

+ **linger.ms**

   Producer默认会把两次发送时间间隔内收集到的所有Requests进行一次聚合然后再发送，以此提高吞 吐量，而linger.ms就是为每次发送到broker的请求增加一些delay，以此来聚合更多的Message请求。 这个有点想TCP里面的Nagle算法，在TCP协议的传输中，为了减少大量小数据包的发送，采用了Nagle 算法，也就是基于小包的等-停协议。

+ **group.id**
  consumer group是kafka提供的可扩展且具有容错性的消费者机制。既然是一个组，那么组内必然可以有多个消费者或消费者实例(consumer instance)，它们共享一个公共的ID，即group ID。组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区(partition)。当然，每个分区只能由同一个消费组内的一个consumer来消费.

+ **enable.auto.commit**

消费者消费消息以后自动提交，只有当消息提交以后，该消息才不会被再次接收到，还可以配合

auto.commit.interval.ms控制自动提交的频率。当然，我们也可以通过consumer.commitSync()的方式实现手动提交.

+ **auto.offset.reset**

  + `auto.offset.reset=latest`情况下新的消费者将会从其他消费者最后消费的offset处开始消费Topic下的消息
  + `auto.offset.reset= earliest`情况下，新的消费者会从该topic最早的消息开始消费
  + `auto.offset.reset=none`情况下，新的消费者加入以后，由于之前不存在offset，则会直接抛出异常。

+ **max.poll.records**

  此设置限制每次调用poll返回的消息数，这样可以更容易的预测每次poll间隔要处理的最大值。通过调

  整此值，可以减少poll间隔.

### 关于Topic和Partition

+ **Topic**

  在kafka中，topic是一个存储消息的逻辑概念，可以认为是一个消息集合。每条消息发送到kafka集群的

  消息都有一个类别。物理上来说，不同的topic的消息是分开存储的，每个topic可以有多个生产者向它发送消息，也可以有多个消费者去消费其中的消息。

+ **Partition**

  每个topic可以划分多个分区（每个Topic至少有一个分区），同一topic下的不同分区包含的消息是不同

  的。每个消息在被添加到分区时，都会被分配一个offset（称之为偏移量），它是消息在此分区中的唯

  一编号，kafka通过offset保证消息在分区内的顺序，offset的顺序不跨分区，即kafka只保证在同一个

  分区内的消息是有序的。

  >  每一条消息发送到broker时，会根据partition的规则选择存储到哪一个partition。如果partition规则
  >
  > 设置合理，那么所有的消息会均匀的分布在不同的partition中，这样就有点类似数据库的分库分表的概
  >
  > 念，把数据做了分片处理。

### Topic&Partition的存储

Partition是以文件的形式存储在文件系统中，比如创建一个名为fifirstTopic的topic，其中有3个

partition，那么在kafka的数据目录（/tmp/kafka-log）中就有3个目录

fifirstTopic-0~3， 命名规则是<topic_name>-<partition_id>

### 消息分发策略

+ 自定义分发策略

  在kafka中，一条消息由key、value两部分构成，在发送一条消息时，我们可以指定这个key，那么producer会根据key和partition机制来判断当前这条消息应该发送并存储到哪个partition中。我们可以根据需要进行扩展producer的partition机制。

  需要实现 Partitioner 接口，在配置文件`ProducerConfig.PARTITIONER_CLASS_CONFIG`指定对应的类

+ 消息默认的分发机制

  kafka采用的是hash取模的分区算法。如果Key为null，则会随机分配一个分区。这个随机
  是在这个参数”metadata.max.age.ms”的时间范围内随机选择一个。对于这个时间段内，如果key为
  null，则只会发送到唯一的分区。这个值值哦默认情况下是10分钟更新一次。

  > Metadata:简单理解就是Topic/Partition和broker的映射关系，每一个topic的每一个partition，需要知道对应的broker列表是什么，leader是谁、follower是谁。这些信息都是存储在Metadata这个类里面

### 消费端如何消费指定的分区

```java
/消费指定分区的时候，不需要再订阅
//kafkaConsumer.subscribe(Collections.singletonList(topic));
//消费指定的分区
TopicPartition topicPartition=new TopicPartition(topic,0);
kafkaConsumer.assign(Arrays.asList(topicPartition));
```

### 消息消费原理

+ 如果consumer比partition多，是浪费，因为kafka的设计是在一个partition上是不允许并发的，
  所以consumer数不要大于partition数
+ 如果consumer比partition少，一个consumer会对应于多个partitions，这里主要合理分配
  consumer数和partition数，否则会导致partition里面的数据被取的不均匀。最好partiton数目是
  consumer数目的整数倍
+ 如果consumer从多个partition读到数据，不保证数据间的顺序性，kafka只保证在一个partition
  上数据是有序的，但多个partition，根据你读的顺序会有不同。
+ 增减consumer，broker，partition会导致rebalance，所以rebalance后consumer对应的
  partition会发生变化

### kafka consumer的rebalance

+ 同一个consumer group内新增了消费者
+  消费者离开当前所属的consumer group，比如主动停机或者宕机
+ topic新增了分区（也就是分区数量发生了变化）

### 分区分配策略

+ RangeAssignor（范围分区）

  Range策略是对每个主题而言的，首先对同一个主题里面的分区按照序号进行排序，并对消费者按照字
  母顺序进行排序

  + 假设我们有10个分区，3个消费者
    + C1-0 将消费 0, 1, 2, 3 分区
      C2-0 将消费 4, 5, 6 分区
      C3-0 将消费 7, 8, 9 分区
  + 假设我们有11个分区，3个消费者
    + C1-0 将消费 0, 1, 2, 3 分区
      C2-0 将消费 4, 5, 6, 7 分区
      C3-0 将消费 8, 9, 10 分区

+ RoundRobinAssignor（轮询分区）

  轮询分区策略是把所有partition和所有consumer线程都列出来，然后按照hashcode进行排序。最后通
  过轮询算法分配partition给消费线程。如果所有consumer实例的订阅是相同的，那么partition会均匀
  分布。

+ StrickyAssignor 分配策略
  + 分区的分配尽可能的均匀
  + 分区的分配尽可能和上次分配保持相同

### **coordinator **

Kafka提供了一个角色：coordinator来执行对于consumer group的管理，当consumer group的第一个consumer启动的时候，它会去和kafka server确定谁是它们组的coordinator。之后该group内的所有成员都会和该coordinator进行协调通信

### 如何确定coordinator

消费者向kafka集群中的任意一个broker发送一个GroupCoordinatorRequest请求，服务端会返回一个负载最小的broker节点的id，并将该broker设置为coordinator.

### JoinGroup的过程

在rebalance之前，需要保证coordinator是已经确定好了的，整个rebalance的过程分为两个步骤，Join和Sync

​		join: 表示加入到consumer group中，在这一步中，所有的成员都会向coordinator发送joinGroup的请
求。一旦所有成员都发送了joinGroup请求，那么coordinator会选择一个consumer担任leader角色，
并把组成员信息和订阅信息发送消费者

  	leader选举算法比较简单，如果消费组内没有leader，那么第一个加入消费组的消费者就是消费者
leader，如果这个时候leader消费者退出了消费组，那么重新选举一个leader，这个选举很随意，类似
于随机算法.

  + joinGroup
    + protocol_metadata: 序列化后的消费者的订阅信息
    + leader_id： 消费组中的消费者，coordinator会选择一个座位leader，对应的就是member_id
    + member_metadata 对应消费者的订阅信息
    + members：consumer group中全部的消费者的订阅信息
    + generation_id： 年代信息，类似于之前讲解zookeeper的时候的epoch是一样的，对于每一轮
      rebalance，generation_id都会递增。主要用来保护consumer group。隔离无效的offset提交。也就
      是上一轮的consumer成员无法提交offset到新的consumer group中。

​        每个消费者都可以设置自己的分区分配策略，对于消费组而言，会从各个消费者上报过来的分区分配策
略中选举一个彼此都赞同的策略来实现整体的分区分配，这个"赞同"的规则是，消费组内的各个消费者
会通过投票来决定

          + 	在joingroup阶段，每个consumer都会把自己支持的分区分配策略发送到coordinator
          + 	coordinator手机到所有消费者的分配策略，组成一个候选集
          + 	每个消费者需要从候选集里找出一个自己支持的策略，并且为这个策略投票
          + 	最终计算候选集中各个策略的选票数，票数最多的就是当前消费组的分配策略

### Synchronizing Group State阶段

   完成分区分配之后，就进入了Synchronizing Group State阶段，主要逻辑是向GroupCoordinator发送
SyncGroupRequest请求，并且处理SyncGroupResponse响应，简单来说，就是leader将消费者对应
的partition分配方案同步给consumer group 中的所有consumer

​		每个消费者都会向coordinator发送syncgroup请求，不过只有leader节点会发送分配方案，其他消费者只是打打酱油而已。当leader把方案发给coordinator以后，coordinator会把结果设置到
SyncGroupResponse中。这样所有成员都知道自己应该消费哪个分区。
​         consumer group的分区分配方案是在客户端执行的！Kafka将这个权利下放给客户端主要是因为这
样做可以有更好的灵活性

### consumer group rebalance的过程

+ 对于每个consumer group子集，都会在服务端对应一个GroupCoordinator进行管理，
  GroupCoordinator会在zookeeper上添加watcher，当消费者加入或者退出consumer group时，会修
  改zookeeper上保存的数据，从而触发GroupCoordinator开始Rebalance操作
+ 当消费者准备加入某个Consumer group或者GroupCoordinator发生故障转移时，消费者并不知道
  GroupCoordinator的在网络中的位置，这个时候就需要确定GroupCoordinator，消费者会向集群中的
  任意一个Broker节点发送ConsumerMetadataRequest请求，收到请求的broker会返回一个response
  作为响应，其中包含管理当前ConsumerGroup的GroupCoordinator
+ 消费者会根据broker的返回信息，连接到groupCoordinator，并且发送HeartbeatRequest，发送心
  跳的目的是要要奥噶苏GroupCoordinator这个消费者是正常在线的。当消费者在指定时间内没有发送
  心跳请求，则GroupCoordinator会触发Rebalance操作。

## Rocket

根据读取操作的情况分为两种情况：

`DefaultMQPushConsumer`:由系统控制读取情况。系统收到消息后自动调用处理函数来处理消息，自动保存 Offset，而且加 入新的 DefaultMQPushConsumer后会自动做负载均衡。

DefaultMQPushConsumer需要 设 置三 个 参数 : 一 是这个 Consumer 的 GroupName，二是 NameServer 的地址和端 口号，三是 Topic 的名称。

Consumer的 GroupName用于把多个 Consumer组织到一起， 提高并发 处理能力， GroupName需要和消息模式 (MessageModel)配合使用。

NameServer 的地址和端口 号，可以填写多个 ，用分号隔开

Topic名称用来标识消息类型， 需要提前创建。 如果不需要消费某 个 Topic 下的所有消息，可以通过指定消息的 Tag 进行消息过滤，

RocketMQ支持两种消息模式: Clustering和Broadcasting。

+ 在 Clustering模式下，同一个 ConsumerGroup(GroupName相同) 里的每 个 Consumer 只消费所订阅消 息 的一部分 内容， 同一个 ConsumerGroup 里所有的 Consumer消费的内容合起来才是所订阅 Topic 内容的整体， 从而达到负载均衡的目的 
+ 在 Broadcasting模式下，同一个 ConsumerGroup里的每个 Consumer都 能消费到所订阅 Topic 的全部消息，也就是一个消息会被多次分发，被 多个 Consumer消费。

DefaultMQPushConsumer的消费流程

DefaultMQPushConsumer主要功能实现在DefaultMQPushConsumerlmpl 类中，消息的处理逻辑是在 pullMessage 这个函数里的 PullCallBack 中 。 在 PullCallBack 函数里有个 switch 语句，根据从 Broker 返回的消息类型做相应的 处理。

“ PushConsumer”中使用“ PullRequest通过“长轮询”方式达到 Push效果的方法，长轮询方式既有 Pull 的优点，又兼具 Push方式的实时性 Push方式是 Server端接收到消息后，主动把消息推送给 Client端，实时 性高。

`DefaultMQPullConsumer`：由使用者自己控制。Pull方式是 Client端循环地从 Server端拉取消息，主动权在 Client手里， 自己拉取到一定量消息后，处理妥当了再接着取。requestHeader.setSuspendTimeoutMillis (brokerSus- pendMaxTimeMillis)，作用是设置Broker最长阻塞时间，默认设置是 15秒，注 意是 Broker在没有新消息的时候才阻塞，有消息会立刻返回 。服务端接到新消息 请求后， 如果队列 里没有 新消息，并不急于返回，通过一个循环不断查看状态，每次 waitForRunning 一段时间 (默认是5秒)， 然后后再Checka 默认情况下当 Broker一直没 有新 消息， 第三次 Check 的时候， 等待时 间超过 Request里面的 Broker­ SuspendMaxTimeMi11is， 就返回空结果。Broker 端 HOLD 住客户端过来的请求一小段时间，在这个时间内有新 消息到达，就利用现有的连接 立 刻返回消息给 Consumer。Broker 即使有大 量 消息积 压 ，也不会主动推 送给 Consumer 。长轮询方式的局限性，是在 HOLD 住 Consumer 请求的时候需要占用资源， 它适合用在消息队列这种客户端连接数可控的场 景 中 。

DefaultMQPushConsumer 的流量控制

RocketMQ定义了一个快照类 ProcessQueue每个 Message Queue 都会有个对应的 ProcessQueue 对象，保存了这个 Message Queue 消息处理状态的快照 ProcessQueue对象里主要的内容是一个 TreeMap 和一个读写锁。 TreeMap 里以 Message Queue 的 Offset作为 Key，以消息内容的引用为 Value，保存了 所有从 MessageQueue 获取到，但是还未被处理的消息; 读写 锁控制着多个线程对 TreeMap 对象的并发访 问。PushConsumer会判断获取但还未处理的消息个数、消 息总大小、 Offset 的跨度，任何一个值超过设定的大小就隔一段时间再拉取消 息，从而达到流量控制的目的 。 此外 ProcessQueue 还可以辅助实现顺序消费的 逻辑。

`DefaultMQPullConsumer`

一 个 Topic 包括多个 Message Queue，如果这个 Consumer 需要获取 Topic 下所有的消息，就 要遍历多有的 Message Queue。

维护 Offsetstore

从一个 Message Queue 里拉取消息的时候，要传人 Offset参数( long类型 的值)，随着不断读取消息 ， Offset会不断增长 。 这个时候由用户负责把 Offset 存储下来，根据具体情况可以存到内存里、写到磁盘或者数据库里等 

拉取消息的请求发出后，会返回: FOUND、 NO MATCHED MSG、 NO NEW MSG、 OFFSET ILLEGAL 四种状态，需要根据每个状态做不同的处理 。比较重要的两个状态是 FOUNT 和 NO NEW MSG ，分别表示获取到消息和没 有新的消息 。

Consumer 的启动、关闭流程

+ PullConsumer，使用者主动权很高，可以根据实际需要暂停、停止、启动消费过程。需要注意的是 Offset 的保存，要在程序的异常处理部分增加把 Offset 写人磁盘方 面的处理，记准了每个 Message Queue 的 Offset，才能保证消息消 费 的准确性 。

+ DefaultMQPushConsumer 的退出， 要调用 shutdown() 函数， 以便 释放资 源、保存 Offset 等 。 这个调用要加到 Consumer 所在应用的退出逻辑中 

PushConsumer在启动的时候 ，会做各种配置检查，然后连接 NameServer 获取 Topic 信息，启动时如果遇到异常，比如无法连接 NameServer，程序仍然 可以正常启动不报错(日 志里有 WARN 信息 )。 在单机环境下可以测试这种情 况，启动 DefaultMQPushConsumer 时故 意 把 NameServer 地址填错，程序仍然 可以正常启动，但是不会收到消息 。 RocketMQ 集群可以有多 个 NameServer、 Broker，某个机器出异常后整体服务依然可用。 所以 DefaultMQPushConsumer 被设计成当发现某个连接异常时不立刻退出，而是不断尝试重新连接。如果需要在 DefaultMQPushConsumer启动的时候，及时暴露配置问题，该 如何操作呢? 可以在 Consumer.start()语句后调用: Consumer.fetchSubscribeMe- ssageQueues(”TopicName”)，这 时如果配 置信息写得不准确，或者当 前服务不可 用，这个语句会报 MQC!ientException 异 常 。



延迟消息的使用方法是在创建 Message对象时，调用 setDelayTimeLevel ( int level) 方法设置延迟时间， 然后再把这个消息发送 出去。

一个 Topic会有多个 Message Queue，如果使用 Producer的默认配置，这 个 Producer 会轮流向各个 Message Queue 发 送 消息 。 Consumer 在消费消息的 时候，会根据负载均衡策略，消费被分配到的 Message Queue，如果不经过特 定的设置，某条消息被发往 l哪个 Message Queue，被哪个 Consumer 消费是未 知的。

把同 一 类型 的消息都发 往 相同的 Message Queue， 该怎 么办呢 ? 可以用 Message­ QueueSelector，

RocketMQ 采用两阶段提交 的方式实现事务消息， RocketMQ 依赖将数据顺序写到磁盘这个 特征来提高性能，却需要更改第一阶段消息的状态，这样会造成磁盘 Catch 的脏页过 多， 降低系统的性能 。



首先来明确一下 Offset 的含义， RocketMQ 中， 一 种类型的消息会放到 一 个 Topic 里，为了能够并行， 一 般一个 Topic 会有多个 Message Queue (也可以 设置成一个)， Offset是指某个 Topic下的一条消息在某个 Message Queue里的 位置，通过 Offset的值可以定位到这条消息，或者指示 Consumer从这条消息 开始向后继续处理 。

Offset 的类结构，主要分为本地文件类型和 Broker代存 的类型两种 。 对于 DefaultMQPushConsurner来说，默认是 CLUSTERING 模 式，也就是同一个 Consumer group 里的多个消费者每人消费一部分，各自收到 的消息内容不一样 。 这种情况下，由 Broker 端存储和控制 Offset 的值，使用 RemoteBrokerOffsetStore 结构 。

DefaultMQPushConsumer里的 BROADCASTING模式下，每个 Consumer 都收到这个 Topic 的全部消息，各个 Consumer 间相互没有干扰， RocketMQ 使 用 LocalfileOffsetStore，把 Offset存到本地 。

如果 PullConsumer，我们就要自己处理 OffsetStore了 `LocalFileOffsetStore`

DefaultMQPushConsumer类里有个函数用来设置从哪儿开始消费 消 息: 指定offset 时间 时间戳格式是精确到秒的 。

注意设置读取位置不是每次都有效，它的优先级默认在 Offset Store后面 ， 比如 在 DefaultMQPushConsumer 的 BROADCASTING 方式 下 ，默 认 是 从 Broker 里读取某个 Topic 对 应 ConsumerGroup 的 Offset， 当读 取不到 Offset 的时候， ConsumeFromWhere 的设置才生效 。 大部分情况下这个设置在 Consumer Group初次启动时有效。 如果 Consumer正常运行后被停止， 然后再启动， 会 接着上次的 Offset开始消费， ConsumeFromWhere 的设置元效。



NameServer是整个消息队列中 的状态服务器，集群的各个组件通过它来了 解全局的信息 。 同时午，各个角色的机器都要定期 向 NameServer上报自己的状 态，超 时不上报的 话， NameServer 会认为 某个机器出故障不可用了，其他的组 件会把这个机器从可用列表里移除 。

NameServer本身是无状态的，也就 是说 NameServer 中的 Broker、 Topic 等状态信息不会持久存储，都是由各个角色 定时上报并存储到内存中的

RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成的， 消息真正的物理存储文件是 CommitLog.

ConsumeQueue 是消息的逻辑队 列，类似数据库的索引文件，存储的是指向物理存储的地址 。 每个 Topic 下 的每个 Message Queue都有一个对应的 ConsumeQueue 文件 。 文件地址在 ${$storeRoot}\consumequeue\${topicName}\${queueld}\${fileName}

ConsumeQueue 组成

CommitLog_Offset  CommitLog 中的物理便宜量

Size 消息大小

message tagHashcode 消息标签的tag 哈希值。

CommitLog 以物理文件的方式存放，每台 Broker上的 CommitLog被本 机器所有 ConsumeQueue 共 享

在 CommitLog 中，一个消息的存储长度是不固定的， RocketMQ 采取一些机制，尽量 向 CommitLog 中顺序写 ，但是随机读 。 ConsumeQueue 的 内容也会被写到磁盘里作持久存储 。

存储机制这样设计有以下几个好处:

+ CommitLog 顺序 写 ，可以大大提 高写人效率 
+ 虽然是随机读，但是利用操作系统的 pagecache 机制，可以批量地从磁 盘读取，作为 cache存到内存中，加速后续的读取速度
+ 为了保证完全的顺序写，需要 ConsumeQueue 这个中间结构 ，因为 ConsumeQuue 里只存偏移量信息，所以尺寸是有限的，在实际情况中，大部 分的 ConsumeQueue 能够被全部读人内存，所以这个中间结构的操作速度很快， 可以认为是内存读取的速度 。 此外为了保证 CommitLog 和 ConsumeQueue 的一 致性， CommitLog 里存储了 Consume Queues、 Message  key、 Tag 等所有信息， 即使 ConsumeQueue 丢失，也可以通过 commitLog 完全恢复出来 。
