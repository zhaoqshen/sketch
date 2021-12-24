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

