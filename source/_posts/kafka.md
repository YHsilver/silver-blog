---
title: Kafka
tags:
  - MQ
  - Kafka
abbrlink: 9a7d5a13
date: 2022-11-29 19:13:42
categories:
description:
cover: http://images.ashery.cn/img/638c91aa16f2c2beb1a86dc2.png
---


# What is Kafka

> Apache Kafka is an open-source distributed event streaming platform used by thousands of companies for high-performance data pipelines, streaming analytics, data integration, and mission-critical applications.

Kafka是一种分布式的，基于发布/订阅的消息系统。

同时也是一个分布式事件流平台。

# As Message Queue

## Kafka总体架构

![img](http://images.ashery.cn/img/638c8eb216f2c2beb1a4d1c2.png)

**关键概念 Key Concept**

-  **消息Record**：Kafka 中的数据单元被称一条Record/Message/Event。`<key,value,timestamp>` 

-  **生产者Producer**： 持续不断的向某个主题发送消息。 

-  **消费者Consumer**：处理生产者产生的消息。 

-  **Broker**: 接收来自生产者的消息，将消息持久化到磁盘，并供消费者拉取消息，一个Broker就是一个Kafka服务器。 

-  **主题Topic**：消息的种类称为主题Topic，一个主题Topic代表了一类消息。在物理上，一个Topic约等于一个append-only immutable log。 

-  **分区Partition**：一个主题可以被分为若干个分区partition，每个分区是一组有序的消息日志。生产者生产的每条消息只会被发送到一个分区中。一个主题中的分区一般来说会部署在多个机器上，由此来实现 kafka 的**伸缩性**，单一主题中的**分区有序（偏序）**，但是无法保证主题中所有的分区有序。 

-  **分区偏移量Offset**：每条消息都有一个当前partition下唯一的64字节的offset，它指明了这条消息的位置。 

-  **副本Replica**：Kafka 中同一条消息能够被拷贝到多个地方以提供数据冗余，这些地方就是所谓的副本，保证消息的**可靠性**。一个Partition会包含多个副本，Leader负责与客户端的交互 ，Follow只负责备份。 

-  **消费者群组Comsumer Group**：多个消费者实例共同组成的一个组，同时消费多个分区以实现**高吞吐**。一个Topic的消息可以被多个消费组消费；一个消费组内，每个分区都只会被组内的一个消费者实例消费，其他消费者实例不能消费它。 

-  **消费者偏移量Comsumer Offset**：表征消费者消费进度，每个消费者都有自己的消费者位移，是一个不断递增的整数值。 

-  **重平衡Rebalance**：消费者组内某个消费者实例挂掉后，其他消费者实例自动重新分配订阅主题分区的过程。Rebalance 是 Kafka 消费者端实现高可用的重要手段。 

Kafka通过Zookeeper管理集群配置，选举leader，以及在Consumer Group发生变化时进行rebalance。

## Producer

### 消息发送流程

![img](http://images.ashery.cn/img/638c8ecb16f2c2beb1a4f188.png)

1.  producer发出消息，被拦截器拦截。拦截器可以实现用户在消息发送前、发送成功后、发送失败时定制自己的逻辑 
2. producer发出消息，被拦截器拦截。拦截器可以实现用户在消息发送前、发送成功后、发送失败时定制自己的逻辑 
3.  消息的key/value被序列化 
4. 如果消息没有指定分区将会由分区器计算消息所属分区 
5.  消息被加到对应的deque的队尾批次ProducerBatch中(如果批次不存在会创建批次) 
6. sender线程不断从RecordAccumulator获取批次，按照目标brokerId分组然后组装成网络请求 
7. NetWorkClient将这些分好组的ClientRequest发到对应的KafkaChannel
8.  KafkaSelector真正执行网络请求 
9.  Kafka集群返回响应 



### Batch Send

批处理是提升性能的一个主要方式，为了允许批量处理，kafka 生产者会尝试在内存中汇总数据，并用一次请求批次提交信息。 批处理不仅仅可以配置指定的消息数量，也可以指定等待特定的延迟时间（如 64k 或 10ms)，这允许汇总更多的数据后再发送，在服务器端也会减少更多的 IO 操作。 batch只针对异步发送接口，同步发送接口消息不会batch，直接ping-pang返回发送结果。



同一个partition里batch的消息，会一次性写入，对应一次io操作。增加batch，会降低io操作次数，降低集群的qps，从而增加集群吞吐；消息batch在一起，压缩率也更高。



默认情况下，一个批次(ProducerBatch)大小为16k，如果一条消息大小大于16k，那么kafka就是一条消息一条消息发送的。



### 消息发送机制

**三种发送模式：**

1.  仅发送：不关心发送结果，调用send接口后，直接返回，kafka内部有重试机制 
2.  同步发送：发送消息后producer会一直阻塞等待broker的响应，如果发生可重试错误Kafka会自动重试。 
3.  异步发送：异步发送消息，producer只发送消息，并不关心返回值，可以注册回调函数对异常情况进行处理。 



Producer发送消息时指定的 acks 参数指定了要有多少个分区副本接收消息，生产者才认为消息是写入成功的。此参数对消息丢失的影响较大

-  如果 acks = 0，这种情况生产者不等待broker响应，不保证消息被同步到broker。这种模式下吞吐量最大，但丢消息的可能性最大。 

-  如果 acks = 1，只要集群的 Leader 接收到消息，就会给生产者返回一条消息，告诉它写入成功。 

-  如果 acks = all，这种情况下是只有当所有参与复制的节点都收到消息时，生产者才会接收到一个来自服务器的消息。 



### 分区策略

1.  如果在ProducerRecord中指定了partition，则按照该值路由 
2.  如果没有指定分区信息，但是消息有key，则按照`Hash(key)`路由 
3.  否则，按照轮询（Round-Robin）策略分区 



### 生产者消息压缩

在消息发送端将消息进行压缩，以减少网络传输带宽和磁盘占用，以CPU 时间去换磁盘空间或者 I/O 传输量，可以在发送端指定压缩策略（比如snappy, gzip...）



## Broker

### Broker处理流程

![img](http://images.ashery.cn/img/638c8ed316f2c2beb1a4fb85.png)



1.  每个broker会在监听的每个端口上启动一个acceptor线程来处理client的连接 
2.  acceptor线程将请求用轮询的方式转发到processor线程(网络线程) 
3.  processor负责获取client的请求并放到请求队列中 
4.  IO线程从请求队列获取请求并处理请求 
5.  IO线程然后将处理的返回值放入processor的响应队列中 
6.  processor将响应队列中的响应返回给client 



### 文件结构

![img](http://images.ashery.cn/img/638c8ede16f2c2beb1a50d99.png)



#### 时间索引文件 .timeIndex  **<timestamp, offset>**

![img](http://images.ashery.cn/img/638c8ee616f2c2beb1a51a5a.jpg)

#### 位置索引文件 .index <offset, position>

![img](http://images.ashery.cn/img/638c8eed16f2c2beb1a52300.png)

#### 消息日志文件 .log

![img](http://images.ashery.cn/img/638c8ef416f2c2beb1a52abe.jpg)



### 主从备份

Kafka 定义了两类副本：领导者副本（Leader Replica）和追随者副本（Follower Replica）。前者对外提供服务，这里的对外指的是与客户端程序进行交互；而后者只是被动地追随领导者副本而已，不能与外界进行交互。Follower只做一件事：向领导者副本发送请求，请求领导者把最新生产的消息发给它，这样它能保持与领导者的同步。

Kafka副本不对外提供服务的意义：如果允许follower副本对外提供读服务（主写从读），首先会存在数据一致性的问题，消息从主节点同步到从节点需要时间，可能造成主从节点的数据不一致。主写从读无非就是为了减轻leader节点的压力，将读请求的负载均衡到follower节点，如果Kafka的分区相对均匀地分散到各个broker上，同样可以达到负载均衡的效果，没必要刻意实现主写从读增加代码实现的复杂程度

Leader负责跟踪所有的follower状态，如果follower“落后”太多或者失效，leader将会把它从replicas同步列表中删除。当所有in sync的follower都将一条消息保存成功，此消息才被认为是“committed”。



#### 主从同步

**LEO&HW概念：**

-  **LEO(last end offset)**: 指的是底层日志文件存储的消息最大偏移量的下一个值 

-  **HW(high water):** 高水位，主从同步中的最小LEO值，消费者可读的范围 

![img](http://images.ashery.cn/img/638c8efb16f2c2beb1a533b8.png)



**更新机制**：

- Follower向leader同步数据时会返回自己的HW，leader会记录这些follower的HW并更新自己的HW为这些HW的最小值。![img](http://images.ashery.cn/img/4a97c8b6f18a219259704210150a3800.svg)

- 同时Follower会对比leader的HW和自身的LEO值，取小作为follwer的HW值。![img](http://images.ashery.cn/img/f1d247f2f1cd558f4eac97fa2db8147a.svg)



#### 故障恢复

Kafka并没有采取quorum 算法来进行Leader的筛选，大多数投票的缺点是，多数的节点挂掉让你不能选择 leader。要冗余单点故障需要三份数据，并且要冗余两个故障需要五份的数据。根据我们的经验，在一个系统中，仅仅靠冗余来避免单点故障是不够的，但是每写 5 次，对磁盘空间需求是 5 倍， 吞吐量下降到 1/5，这对于处理海量数据问题是不切实际的。这可能是为什么 quorum 算法更常用于共享集群配置（如 ZooKeeper ）， 而不适用于原始数据存储的原因。



Kafka 采取了一种稍微不同的方法来选择它的投票集。 Kafka 不是用大多数投票选择 leader 。Kafka 动态维护了一个同步状态的备份的集合 （a set of in-sync replicas）， 简称 ISR ，在这个集合中的节点都是和 leader 保持高度一致的，只有这个集合的成员才 有资格被选举为 leader，一条消息必须被这个集合 所有 节点读取并追加到日志中了，这条消息才能视为提交。这个 ISR 集合发生变化会在 ZooKeeper 持久化，正因为如此，这个集合中的任何一个节点都有资格被选为 leader 。



选择Follower时需要兼顾一个问题，就是server上所已经承载的partition leader的个数，如果一个server上有过多的partition leader，意味着此server将承受着更多的IO压力。zookeeper在选举新leader时，需要考虑到“负载均衡”，partition leader较少的broker将会更有可能成为新的leader。



另一个重要的设计区别是，Kafka 不要求崩溃的节点恢复所有的数据, 新选出的Leader会舍弃HW之后的内容（因为可能还有其他replica没有最新的数据）。



### 日志压缩

Kafka提供了一种机制来为单个 topic partition 的数据日志中的每个 message key 保留最新的已知值。

![img](http://images.ashery.cn/img/638c8f0316f2c2beb1a53dbe.jpg)

## Consumer

### 消费流程

![img](http://images.ashery.cn/img/638c8f0916f2c2beb1a544d8.png)

1.  consumer group的第一台consumer启动时就会选择消费者组协调器coordinator(根据consumerGroupId的哈希值确定) 
2.  consumer向coordinator发出join group请求，coordinator指定第一个发出join group的consumer为leader consumer(这里假定为consumer0) 
3.  leader consumer(consumer0)制定分区消费策略发给coordinator 
4.  coordinator给所有consumer下发分区消费方案 
5.  consumer向broker拉取消息 



### Pull or Push

-  Pull方式：consumer从Broker中主动拉取消息。客户端可以依据自己的消费能力进行消费，但是有一定的延迟性（取决于消费者poll的时间间隔interval） 

-  Push方式：Broker主动推送消息到consumer。及时性很高；但是当客户端消费能力远低于服务端生产能力，那么一旦服务端推送大量消息到客户端时，就会导致客户端消息堆积，处理缓慢，甚至服务崩溃，需要提供消费者限流。并且服务端需要维护每次传输状态（ACK机制），以防消息传递失败进行重试。 

-  推拉结合：Broker先同步一个消息信号给comsumer，通知comsumer有新消息到来，comsumer可根据当前自身的消费能力和情况选择是否拉取最新的消息。 



**Kafka****采用了Pull的方式。**简单的 pull-based 系统的不足之处在于：如果 broker 中没有数据，consumer 可能会在一个紧密的循环中结束轮询，实际上 busy-waiting 直到数据到来。为了避免 busy-waiting，Kafka在 pull 请求中加入参数，使得 consumer 在一个“long pull”中阻塞等待，直到数据到来（还可以选择等待给定字节长度的数据来确保传输长度）。



### 消费组 Consumer Group

`消费者组（Consumer Group）`是由一个或多个消费者实例（Consumer Instance）组成的群组，具有可扩展性和可容错性的一种机制。消费者组内的消费者`共享`一个消费者组ID`Group ID`，组内的消费者共同对一个主题进行订阅和消费，同一个组中的消费者只能消费一个分区的消息，多余的消费者会闲置，派不上用场。如果consumer比partition少，一个consumer会对应于多个partitions，这里需要合理分配consumer数和partition数，否则会导致partition里面的数据被取的不均匀。



### 再均衡Rebalance

增减consumer，broker，partition会导致rebalance，即把分区的所有权通过一个消费者转到其他消费者。



重平衡非常重要，它为消费者群组带来了`高可用性` 和 `伸缩性`，我们可以放心的添加消费者或移除消费者，不过在正常情况下我们并不希望发生这样的行为。在重平衡期间，消费者无法读取消息，造成整个消费者组在重平衡的期间都不可用。另外，当分区被重新分配给另一个消费者时，消息当前的读取状态会丢失，它有可能还需要去刷新缓存，在它重新恢复状态之前会拖慢应用程序。



消费者通过向Coordinator Broker发送心跳来维护自己是消费者组的一员并确认其拥有的分区。对于不同不的消费组来说，其组织协调者可以是不同的。只要消费者定期发送心跳，就会认为消费者是存活的。



重平衡是一把双刃剑，它为消费者群组带来高可用性和伸缩性的同时，还有有一些明显的缺点(bug)，而这些 bug 到现在社区还无法修改。重平衡的过程对消费者组也有极大的影响。因为每次重平衡过程中都会STW，也就是说，在重平衡期间，消费者组中的消费者实例都会停止消费，等待重平衡的完成。而且重平衡这个过程很慢。



### 消费者位移 Consumer Offset

大多数消息系统都在 broker 上保存被消费消息的元数据。也就是说，当消息被传递给 consumer，broker 要么立即在本地记录该事件，要么等待 consumer 的确认后再记录。



要让 broker 和 consumer 就被消费的数据保持一致性也不是一个小问题。如果 broker 在每条消息被发送到网络的时候，立即将其标记为 consumed，那么一旦 consumer 无法处理该消息（可能由 consumer 崩溃或者请求超时或者其他原因导致），该消息就会丢失。 为了解决消息丢失的问题，许多消息系统增加了确认机制：即当消息被发送出去的时候，消息仅被标记为 **sent** 而不是 **consumed**；然后 broker 会等待一个来自 consumer 的特定确认，再将消息标记为 **consumed**。这个策略修复了消息丢失的问题，但也产生了新问题。首先，如果 consumer 处理了消息但在发送确认之前出错了，那么该消息就会被消费两次。第二个是关于性能的，现在 broker 必须为每条消息保存多个状态，还有更棘手的问题要处理，比如如何处理已经发送但一直得不到确认的消息。



Kafka的Consumer通过提交offset来记录消费进度，kafka将消费者提交的offset存在一个叫做`_consumer_offsets`的topic中，所以consumer提交偏移量实际上就是往这个topic发消息。这个`_consumer_offset`在rebalance中非常重要，重新分配consumer到partition的消费关系的时候，新的消费者需要知道当前partition的消费进度。rebalance后可能会出现重复消费和消息丢失的情况。



提交方式

-  自动提交： 每次`poll()`请求的时候会检查是否可以进行位移提交（默认配置是5s一次）， 如果可以，那么就会提交上一轮消费的位移。 

-  同步提交： 该方式的最大问题在于数据是批量处理，当部分数据完成消费， 还没来得及提交offset就被中断， 则会使得下次消费会重复消费那部分已经消费过的数据。 

-  异步提交：`commitAsync`属于异步提交， 也就是不会阻塞线程，比起同步提交`commitSync`具有更好的性能。异步提交时需要注意回调函数中失败时的处理，防止出现ABA的问题。 



## 事务消息



Kafka在0.11版本中引入了事务消息和Exactly Once的投递语义，即**通过事务机制，KAFKA 可以实现对多个 topic 的多个 partition 的原子性的写入**，即处于同一个事务内的所有消息，不管最终需要落地到哪个 topic 的哪个 partition, 最终结果都是要么全部写成功，要么全部写失败（Atomic multi-partition writes）。



Kafka中的事务特性主要用于以下两种场景：

-  **生产者发送多条消息可以封装在一个事务中，形成一个原子操作。**多条消息要么都发送成功，要么都发送失败。 

-  **read-process-write模式：将消息消费和生产封装在一个事务中，形成一个原子操作。**在一个流式处理的应用中，常常一个服务需要从上游接收消息，然后经过处理后送达到下游，这就对应着消息的消费和生成。 



**Kafka的事务和其他的消息队列的事务语义并不相同，他并不支持本地事务执行和发送消息的一致性。**

**比如RocketMQ解决的是本地事务的执行和发消息这两个动作满足事务的约束。**



### 事务流程

**为支持事务机制，kafka引入了两个新的组件，Transaction Coordinator 和 Transaction Log**

-  transaction coordinator 是运行在每个 kafka broker 上的一个模块，是 kafka broker 进程承载的新功能之一（不是一个独立的新的进程） 

-  transaction log 是 kafka 的一个内部 topic（类似 __consumer_offsets ，是一个内部 topic） 

-  transaction log和普通的消息一样也有多个分区，每个分区都有一个 leader，该 leade对应哪个 kafka broker，哪个 broker 上的 transaction coordinator 就负责对这些分区的写操作 



**为支持事务机制，kafka还将日志文件格式进行了扩展，添加了控制消息 control batch**

-  日志中除了普通的消息，还有一种消息专门用来标志事务的状态，它就是控制消息 control batch； 

-  控制消息跟其他正常的消息一样，都被存储在日志中，但控制消息不会被返回给 consumer 客户端，sonsumer获取消息的时候会根据消息的类型来过滤 

-  控制消息共有两种类型：commit 和 abort，分别用来表征事务已经成功提交或已经被成功终止； 

![img](http://images.ashery.cn/img/638c8f1216f2c2beb1a54ffd.jpg)

A: producer发送事务状态消息到coordiantor，开始事务、record分区写入信息等

B: 持久化事务状态

C: producer正常发送消息到broker集群

D: coordinator写入事务消息状态信息

![img](http://images.ashery.cn/img/638c8f1816f2c2beb1a557c2.jpg)

1.  查找Transacrtion Coordinator：Producer向任意一个Broker发送 FindCoordinatorRequest请求来获取Transaction Coordinator的地址。 
2.  事务初始化：Producer 通过 initTransactions API 将 transactional.id 注册到 transactional coordinator，此时 coordinator 会关闭所有有相同 transactional.id 且处于 pending 状态的事务，比如由于Producer崩溃宕机导致的上次事务未完成。 
3.  开启事务，发送消息：通过beginTransaction API 开启事务，并通过 send API 发送消息到目标topic。此时消息对应的 partition 会首先被注册到 transactional coordinator，然后 producer 按照正常流程发送消息到目标 topic 
4.  消息提交/回滚：通过 commitTransaction API 提交事务或通过abortTransaction API回滚事务：此时会向 transactional coordinator 提交请求，开始两阶段提交协议： 

	1. 第一阶段，transactional coordinator 更新内存中的事务状态为 “prepare_commit”，并将该状态持久化到 transaction log 中；
   2. 第二阶段，coordinator 首先写 transaction marker 标记到目标 topic 的目标 partition，这里的 transaction marker，就是我们上文说的控制消息，控制消息共有两种类型：commit 和 abort，分别用来表征事务已经成功提交或已经被成功终止； coordinator 在向目标 topic 的目标 partition 写完控制消息后，会更新事务状态为 “commited” 或 “abort”， 并将该状态持久化到 transaction log 中

5. 要正确消费事务消息，consumer需要通过"isolation.level"来进行配置隔离级别（read_committed）来过滤掉未提交的消息 



## 消息投递语义

-  At Most Once：最多投递一次，消息可能会丢失但绝不重传 

-  At Least Once：最少投递一次，消息可以重传但绝不丢失，默认提供 

-  Exactly Once：每条消息只被传递一次 



Kafka使用**幂等生产者**和**事务机制**可以保证Exactly Once**（前提是系统中只有kafka，无外部参与者）**

- 幂等生产者：producer使用pid和seqId保证消息只会被写入单partition一次 
  - 为了实现Producer的幂等语义，Kafka引入了Producer ID（即PID）和Sequence Number。每个新的Producer在初始化的时候会被分配一个唯一的PID，该PID对用户完全透明而不会暴露给用户。 

  - 对于每个PID，该Producer发送数据的每个<Topic, Partition>都对应一个从0开始单调递增的Sequence Number。 

  - 类似地，Broker端也会为每个<PID, Topic, Partition>维护一个序号，并且每次Commit一条消息时将其对应序号递增。对于接收的每条消息，如果其序号比Broker维护的序号（即最后一次Commit的消息的序号）大一，则Broker会接受它，否则将其丢弃 


![img](http://images.ashery.cn/img/638c8f1e16f2c2beb1a56037.jpg)

上述幂等设计只能保证单个Producer对于同一个<Topic, Partition>的Exactly Once语义。

-  事务消息：事务保证了跨分区的原子写入，结合事务消息，可以实现Producer到Broker的Exactly Once的语义。 

-  Broker到Consumer的：consumer每次消费消息后会向broker确认消费位移`_consumer_offset`，Broker下次只会投递offset之后的数据，如果这个ack丢失，consumer还是会重复消费，如果引入了外部系统，需要外部系统做幂等操作。 



## 零拷贝

Kafka中存在大量的网络数据持久化到磁盘（Producer到Broker）和磁盘文件通过网络发送（Broker到Consumer）的过程。这一过程的性能直接影响Kafka的整体吞吐量。



**传统模式下的四次拷贝与四次上下文切换**

以将磁盘文件通过网络发送为例。传统模式下，先将文件数据读入内存，然后通过Socket将内存中的数据发送出去。

这一过程实际上发生了四次数据拷贝。首先通过系统调用将文件数据读入到内核态Buffer（DMA拷贝），然后应用程序将内存态Buffer数据读入到用户态Buffer（CPU拷贝），接着用户程序通过Socket发送数据时将用户态Buffer数据拷贝到内核态Buffer（CPU拷贝），最后通过DMA拷贝将数据拷贝到NIC Buffer。同时，还伴随着四次上下文切换，如下图所示。



**sendfile和transferTo实现零拷贝**

Linux 2.4+内核通过`sendfile`系统调用，提供了零拷贝。数据通过DMA拷贝到内核态Buffer后，直接通过DMA拷贝到NIC Buffer，无需CPU拷贝。这也是零拷贝这一说法的来源。除了减少数据拷贝外，因为整个读文件-网络发送由一个`sendfile`调用完成，整个过程只有两次上下文切换，因此大大提高了性能。零拷贝过程如下图所示。

![img](http://images.ashery.cn/img/638c8f2416f2c2beb1a568c0.png)





# As Streaming Platform

Kafka Stream是Apache Kafka从0.10版本引入的一个新Feature。它是提供了对存储于Kafka内的数据进行流式处理和分析的功能。

2014 年的时候，Kafka 的三个主要开发人员从 LinkedIn 出来创业，开了一家叫作 Confluent 的公司。和其他大数据公司类似，Confluent 的产品叫作 Confluent Platform。这个产品的核心是 Kafka，分为三个版本：Confluent Open Source、Confluent Enterprise 和 Confluent Cloud，Confluent将对kafka的很多特性做了补充。

## Kafka Connect

用于在Kafka与其他系统（数据库，云服务，文件等）之间执行流式集成的组件。

Kafka Connect 是在 Kafka 0.9.0 版本中添加的，并在后台使用了生产者和消费者API。Connect Service 是 Confluent 平台的一部分，并与 Apache Kafka 一起提供该平台的发行版。**连接器旨在提供一种连接外部系统的简单方法，只需要一个配置文件，而状态的扩展、分布和持久性由框架为开发者自动处理**。[Confluent Hub](https://www.confluent.io/hub/)中已经存在用于JDBC等常见事物的连接器。

Source Connector从外部系统读取数据并将其发送到 Kafka Topic。

Sink Connector 订阅一个或多个 Kafka Topic并将其读取的消息写入外部系统。

![img](http://images.ashery.cn/img/638c8f2a16f2c2beb1a57054.png)



### 自定义Connector

如果开源社区中没有提供当前系统的Connector时，开发者可以基于Kafka提供的API开发适合自己系统的Connector。

![img](http://images.ashery.cn/img/638c8f2f16f2c2beb1a576c9.jpg)

-  **Connector：**通过管理任务来协调数据流的高级抽象，需要实现的最主要方法是`taskConfig(...)`，用于定义Task如何扩展以及Task的配置信息，会通过Worker线程传递给Task。 

-  **Task**：如何将数据从外部系统复制到 Kafka 的实现。这里是真正定义如何解析外部数据并传入Kafka的地方，这里会使用**SourceTaskContext**来与Connector Service进行数据交互，比如文件读取的offset。Task的`poll()`负责具体的数据处理逻辑，该方法会被Woker线程根据系统的容量等因素循环调用。 

-  **Workers：**执行连接器和任务的正在运行的进程。 

-  **Fileter/Transformer：**更改连接器生成或发送到连接器的每条消息的简单逻辑。 

-  **Convertor：**用于在 Connect 和发送或接收数据的系统之间转换数据的代码。 



## Schema Registry

结构化消息需要被序列化之后才能发送，并且会在消费端进行反序列化，序列化的方式有很多种：JSON，Proto，Avro等。



随着系统的演进，消息体的结构往往是会发生变化的，这时候如果消费者没有及时更新，可能会出现Consumer解析消息错误的情况，如果序列化的方式不是自解释的，这时就需要IDL了。



Schema Registry 是一个独立的服务器进程，运行在 Kafka 代理之外的机器上。它的工作是维护一个包含已写入其负责的集群中主题的所有模式的数据库。该“数据库”保存在**内部 Kafka 主题**中，并缓存在 Schema Registry 中以实现低延迟访问。Schema Registry 可以在冗余的高可用性配置中运行，因此如果一个实例发生故障，它仍然可以运行。

![img](http://images.ashery.cn/img/638c8f3416f2c2beb1a57cbf.jpg)

 

## Kafka Streams

Kafka Stream是Apache Kafka从0.10版本引入的一个新Feature，一套Kafka Java Client 的Library API，它是提供了对存储于Kafka内的数据进行**流式**处理和分析的功能。



### 流式计算

一般流式计算会与批量计算相比较。在流式计算模型中，输入是持续的，可以认为在时间上是无界的，也就意味着，永远拿不到全量数据去做计算。同时，计算结果是持续输出的，也即计算结果在时间上也是无界的。流式计算一般对实时性要求较高，同时一般是先定义目标计算，然后数据到来之后将计算逻辑应用于数据。同时为了提高计算效率，往往尽可能采用增量计算代替全量计算。

![img](http://images.ashery.cn/img/638c8f3a16f2c2beb1a583a9.png)![img](http://images.ashery.cn/img/638c8f4216f2c2beb1a58c81.png)

### 为什么需要Kafka Stream

当前已经有非常多的流式处理系统，比如Spark 、Storm、Flink、Hadoop等。Apache Storm发展多年，应用广泛，提供记录级别的处理能力，当前也支持SQL on Stream。而Spark Streaming基于Apache Spark，可以非常方便与图计算，SQL处理等集成，功能强大，对于熟悉其它Spark应用开发的用户而言使用门槛低。另外，目前主流的Hadoop发行版，如MapR，Cloudera和Hortonworks，都集成了Apache Storm和Apache Spark，使得部署更容易。



既然Apache Spark与Apache Storm拥用如此多的优势，那为何还需要Kafka Stream呢？

1.  Spark和Storm都是流式处理框架，而Kafka Stream提供的是一个基于Kafka的流式处理类库。 

1.  部署和管理方式不同：一般来说 Flink 等大数据处理平台都是由公司的大数据团队部署和管理的，如果某个应用项目中的部分处理逻辑需要依赖于这些大数据处理平台的话，那必然涉及团队间的协作问题，如果相关处理不是很复杂的话，其实完全可以通过 Kafka Stream来解决，因此应用研发团队可以完全掌控相关的代码和部署，从而能够快速响应需求变化。 

1.  Kafka Stream充分利用了Kafka的分区机制和Consumer的Rebalance机制，使得Kafka Stream可以非常方便的水平扩展，并且各个实例可以使用不同的部署方式。 

1.  就流式处理系统而言，基本都支持Kafka作为数据源，Kafka基本上是主流的流式处理系统的标准数据源。在大部分流式系统中都已部署了Kafka的情况下，此时使用Kafka Stream的成本非常低。 



### Kafka Stream 整体架构

![img](http://images.ashery.cn/img/638c8f4816f2c2beb1a5945c.png)





**Processor Topology**

基于Kafka Stream的流式应用的业务逻辑全部通过一个被称为Processor Topology的地方执行。它与Storm的Topology和Spark的DAG类似，都定义了数据在各个处理单元（在Kafka Stream中被称作Processor）间的流动方式，或者说定义了数据的处理逻辑（比如计数）。



**并行模型**

Kafka Stream的模型中，最小粒度为Task，而每个Task包含一个特定子Topology的所有Processor。因此每个Task所执行的代码完全一样，唯一的不同在于所处理的数据集互补。



**KTable &  KStream**

-  KStream是一个数据流，可以认为所有记录都通过Insert only的方式插入进这个数据流里。 

-  KTable代表一个完整的数据集，可以理解为数据库中的表。 

![img](http://images.ashery.cn/img/638c8f4d16f2c2beb1a59a89.png)





**State store**

流式处理中，部分操作是无状态的，例如过滤操作。而部分操作是有状态的，需要记录中间状态，如Window操作和聚合计算。State store被用来存储中间状态。它可以是一个持久化的Key-Value存储，也可以是内存中的HashMap，或者是数据库。Kafka提供了基于Topic的状态存储。



**时间窗口**

由于流式数据是在时间上无界的数据，而聚合操作只能作用在特定的数据集，也即有界的数据集上。因此需要通过某种方式从无界的数据集上按特定的语义选取出有界的数据。窗口是一种非常常用的设定计算边界的方式。



**Join操作**

-  `KTable Join KTable` 结果仍为KTable。任意一边有更新，结果KTable都会更新。 

-  `KStream Join KStream` 结果为KStream。必须带窗口操作，否则会造成Join操作一直不结束。 

-  `KStream Join KTable / GlobalKTable`结果为KStream。只有当KStream中有新数据时，才会触发Join计算并输出结果。 



## Kafka Ksql

大多数的流处理技术，需要开发人员使用 Java 或 Scala 等编程语言编写代码。

KSQL 是 Apache Kafka 的数据流 SQL 引擎，它使用 SQL 语句替代编写大量代码去实现流处理任务。

![img](http://images.ashery.cn/img/638c8f5316f2c2beb1a5a2f2.jpg)





> 参考:
>
> [Confluent Documentation | Confluent Documentation](https://docs.confluent.io/home/overview.html)
>
> [www.jasongj.com](http://www.jasongj.com/categories/Kafka/)
>
> https://juejin.cn/post/6844903495670169607

