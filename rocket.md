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







