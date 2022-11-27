---
title: 事务（二）：分布式事务
tags:
  - 数据库
  - Mysql
  - 事务
cover: 'https://pic.imgdb.cn/item/6382506b16f2c2beb1a9fbaf.jpg'
abbrlink: 26567
date: 2022-11-27 21:30:54
---

# 分布式事务

分布式事务常用于多个数据库之前数据一致性的场景

一般来说可分为两种场景，一是一个服务操作多个数据库。比如对于一个支付宝的转账业务来说，由于分库的存在，两方的账户数据可能分布在不同地区的数据，可能一个在杭州，一个在上海。（ps. 对支付宝数据架构感兴趣可以点[这里](https://www.infoq.cn/article/kihsqp_twv16tiipa1lo)）

另一种场景是多个服务操作多个数据库，一个业务操作涉及多个微服务，每个微服务有自己独立的数据库。比如淘宝支付，余额和优惠券由两个微服务维护在不同的数据库，需要保证余额和优惠券同时扣减成功或失败。

# ACID、CAP与BASE理论

ACID理论在前一篇{% post_link transaction '本地事务' %}已经讲过了，ACID在单机事务上表现的很好。但是在分布式场景下，我们几乎无法实现完整意义上的ACID，即CAP理论的存在。

CAP理论指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）最多只能同时实现两点，不可能三者兼顾。（ps. [具体证明](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf)）

- **一致性**（**C**onsistency）：对于数据分布在不同节点上的数据上来说，任何一个节点最新的读都能返回最新的写值或者错误。
- **可用性**（**A**vailability）：每一个请求都能返回一个非错误的响应，代表系统不间断地提供服务的能力。
- **分区容错性**（**P**artition Tolerance）：部分节点因网络原因而彼此失联后，即与其他节点形成“网络分区”时，分区之间不能同步数据，但系统仍能正确地提供服务的能力。

因此我们需要有所取舍，在出现错误的情况下，放弃一个特性来满足其他两个特性。

- **如果放弃分区容忍性**（CA without P）：那么当出现网络分区后，我们不保证对外提供正确的服务，这显然违反了CA，因此放弃了P也放弃了CA，几乎是一个悖论。另一种方式就是保证系统不会出现网络分区，自然也就没有分区容错的必要了，但是由于网络的不可靠性，永远可靠的通信在分布式系统中必定不成立的。进一步，不能使用网络，其实就退回到传统单机集群的概念了，在一台机器上部署多个数据库实例，实例之间通过共享磁盘来避免网络分区，这显然不是我们追求的分布式事务。

- **如果放弃可用性**（CP without A）：意味着我们将假设一旦网络发生分区，不保证当时服务的可用性，节点之间的信息同步时间可以无限地延长。在现实中，选择放弃可用性的 CP 系统情况一般用于对数据质量要求很高的场合中。最经典使用CP的系统就是ZooKeeper了，即任何时刻对zookeeper的访问请求能得到一致性的数据结果，在使用ZooKeeper获取服务列表时，如果正在选举或者集群中半数以上的机器不可用，那么将无法获取数据。

- **如果放弃一致性**（AP without C）：意味着我们将假设一旦发生分区，节点之间所提供的数据可能不一致。选择放弃一致性的 AP 系统目前是设计分布式系统的主流选择，因为 A 通常是建设分布式的目的，如果可用性随着节点数量增加反而降低的话，很多分布式系统可能就失去了存在的价值。目前大多数 NoSQL 库和支持分布式的缓存框架都是 AP 系统，以 Redis 集群为例，如果某个 Redis 节点出现网络分区，那仍不妨碍各个节点以自己本地存储的数据对外提供缓存服务，但这时有可能出现请求分配到不同节点时返回给客户端的是不一致的数据。

  由于AP在分布式系统中应用的最多，相关的研究和实践也比较多。在AP理论上进一步出现了[BASE](https://en.wikipedia.org/wiki/Eventual_consistency)理论，BASE理论是Basically Available(基本可用)，Soft State（软状态）和Eventually Consistent（最终一致性）三个短语的缩写，核心思想是既然无法做到强一致性，那么就退而求其次，保证最终一致性。



# 分布式事务原理

## 单个服务使用多个数据源

本节之所以限定为“单个服务使用多个数据源场景”，主要是考虑在分布式环境中仍追求强一致性的事务处理方案，即追求AP。

### XA规范

1991 年，为了解决分布式事务的一致性问题，[X/Open](https://en.wikipedia.org/wiki/X/Open)组织提出了一套名为[X/Open XA](https://en.wikipedia.org/wiki/X/Open_XA)（XA 是 eXtended Architecture 的缩写）的处理事务架构，其核心内容是定义了全局的事务管理器（Transaction Manager，用于协调全局事务）和局部的资源管理器（Resource Manager，用于驱动本地事务）之间的通信接口。其中，每个资源管理器RM都具有独立性，且不必是同构的。全局事务的原子性由事务管理器来负责，各个模块的交互如下图所示。AP通过TM来声明一个全局事务，然后操作不同RM上的资源，最后通知TM来提交或者回滚全局事务。

![XA](https://pic.imgdb.cn/item/63833ab016f2c2beb1cd73cd.jpg)

### 2PC协议

可以将2PC协议理解为XA规范中TM和RM的具体交互协议。

2PC协议中包含两种角色，**协调者（Coordinator）**和**参与者（Participant）**。协调者负责整个协议的推进，使得多个参与者最终达到一致的决议。参与者响应协调者的请求，根据协调者的请求完成` prepare `操作及 `commit/abort `操作。

2PC将事务分成了准备和提交两个阶段，如图。

<img src="https://pic.imgdb.cn/item/63833e7216f2c2beb1d22e08.png" alt="image-20221127183636545" style="zoom: 33%;"  />

准备阶段

- 协调者：协调者向所有的参与者发起·` prepare request`
- 参与者：参与者收到 `prepare request` 之后，决定是否可以提交，如果可以则持久化` prepare log` 并且向协调者返回` prepare `成功，否则返回` prepare` 失败。

提交阶段

- 协调者：收齐所有参与者的` prepare ack `之后，进入` COMMIT/ROLLBACK` 状态，向所有参与者发送事务 `commit/rollback request`。
- 参与者：收到` commit/rollback request` 之后释放资源解行锁，然后提交 `commit log`，日志持久化完成之后给协调者回复 `commit ok` 消息，最后释放事务上下文并退出。



两段式提交原理简单，并不难实现，但有几个非常显著的缺点：

- **单点问题**：协调者在两段提交中具有举足轻重的作用，协调者等待参与者回复时可以有超时机制，允许参与者宕机，但参与者等待协调者指令时无法做超时处理。一旦宕机的不是其中某个参与者，而是协调者的话，所有参与者都会受到影响。如果协调者一直没有恢复，没有正常发送 Commit 或者 Rollback 的指令，那所有参与者都必须一直等待。
- **性能问题**：两段提交过程中，所有参与者相当于被绑定成为一个统一调度的整体，期间要经过两次远程服务调用，三次数据持久化（准备阶段写` prepare log` ，协调者做状态持久化，提交阶段在日志写入 ` prepare log` ），整个过程将持续到参与者集群中最慢的那一个处理操作结束为止，这决定了两段式提交的性能通常都较差。

### 3PC协议

为了缓解两段式提交协议的一部分缺陷，后续又发展出了“[三段式提交](https://zh.wikipedia.org/wiki/三阶段提交)”（3 Phase Commit，**3PC**）协议。

3PC把原本的两段式提交的准备阶段再细分为两个阶段，分别称为 CanCommit、PreCommit，把提交阶段改称为 DoCommit 阶段。其中，新增的 CanCommit 是一个询问阶段，协调者让每个参与的数据库根据自身状态，评估该事务是否有可能顺利完成。

将准备阶段一分为二的理由是这个阶段是重负载的操作，一旦协调者发出开始准备的消息，每个参与者都将马上开始写重做日志，它们所涉及的数据资源即被锁住，如果此时某一个参与者宣告无法完成提交，相当于大家都白做了一轮无用功。所以，增加一轮询问阶段，如果都得到了正面的响应，那事务能够成功提交的把握就比较大了，这也意味着因某个参与者提交时发生崩溃而导致大家全部回滚的风险相对变小。

因此，在事务需要回滚的场景中，三段式的性能通常是要比两段式好很多的，但在事务能够正常提交的场景中，两者的性能都依然很差，甚至三段式因为多了一次询问，还要稍微更差一些。

同样也是由于事务失败回滚概率变小的原因，在三段式提交中，如果在 PreCommit 阶段之后发生了协调者宕机，即参与者没有能等到 DoCommit 的消息的话，默认的操作策略将是提交事务而不是回滚事务或者持续等待，这就相当于避免了协调者单点问题的风险。三段式提交的操作时序如图所示。

<img src="https://pic.imgdb.cn/item/638344bc16f2c2beb1dde26d.png" alt="image-20221127190326121" style="zoom:50%;" />

从以上过程可以看出，三段式提交对单点问题和回滚时的性能问题有所改善，但是它对一致性风险问题并未有任何改进，在这方面它面临的风险甚至反而是略有增加了的。譬如，进入 PreCommit 阶段之后，协调者发出的指令不是 Ack 而是 Abort，而此时因网络问题，有部分参与者直至超时都未能收到协调者的 Abort 指令的话，这些参与者将会错误地提交事务，这就产生了不同参与者之间数据不一致的问题。

### AT协议

除了上面提到的问题外，以上两种协议（2PC和3PC）还有一个本质上最大的缺陷就是依赖于数据库实现的XA协议，实际上很多NoSQL数据库并没有实现XA协议，因此也不能直接使用2PC和3PC来完成分布式事务。

阿里GTS所提出的[AT 事务模式](https://seata.io/zh-cn/docs/overview/what-is-seata.html)就在一定成都上解决了这个问题。AT 模式是一种无侵入的分布式事务解决方案。在 AT 模式下，用户只需关注自己的业务 SQL，用户的业务 SQL作为一阶段，Seata 框架会自动生成事务的二阶段提交和回滚操作。

其最重要的思想是：基于数据补偿来代替回滚。这种思想在后文的SAGA模式中也会介绍。

![image.png](https://pic.imgdb.cn/item/638348d616f2c2beb1e54545.png)

- 一阶段准备

  在一阶段，Seata 会拦截业务 SQL，首先解析 SQL 语义，找到业务 SQL要更新的业务数据，在业务数据被更新前，将其保存成`before image`，然后执行业务 SQL更新业务数据，在业务数据更新之后，再将其保存成`after image`，最后生成行锁。以上操作全部在一个数据库事务内完成，这样保证了一阶段操作的原子性。

![图片3.png](https://pic.imgdb.cn/item/638348ff16f2c2beb1e57bfd.png)

- 二阶段提交

  因为业务 SQL在一阶段已经提交至数据库， 所以 Seata 框架只需将一阶段保存的快照数据和行锁删掉，完成数据清理即可。

![图片4.png](https://pic.imgdb.cn/item/6383491416f2c2beb1e58f77.png)

- 二阶段回滚

  二阶段如果是回滚的话，Seata 就需要回滚一阶段已经执行的业务 SQL，还原业务数据。回滚方式便是用`before image`还原业务数据；但在还原前要首先要**校验脏写**，对比“数据库当前业务数据”和 “after image”，如果两份数据完全一致就说明没有脏写，可以还原业务数据，如果不一致就说明有脏写，出现脏写就需要转人工处理。

![图片5.png](https://pic.imgdb.cn/item/6383491e16f2c2beb1e5998c.png)

从整体上看是 AT 事务是参照了 XA 两段提交协议实现的，但针对 XA 2PC 的缺陷，即在准备阶段必须等待所有数据源都返回成功后，协调者才能统一发出 Commit 命令而导致的木桶效应，设计了针对性的解决方案。

基于这种补偿方式，分布式事务中所涉及的每一个数据源都可以单独提交，然后立刻释放锁和资源。这种异步提交的模式，相比起 2PC 极大地提升了系统的吞吐量水平。而代价就是大幅度地牺牲了隔离性，甚至直接影响到了原子性。因为在缺乏隔离性的前提下，以补偿代替回滚并不一定是总能成功的，即脏写问题。

通常来说，脏写是一定要避免的，所有传统关系数据库在最低的隔离级别上都仍然要加锁以避免脏写，因为脏写情况一旦发生，人工其实也很难进行有效处理。所以 GTS 增加了一个“全局锁”（Global Lock）的机制来实现写隔离，要求本地事务提交之前，一定要先拿到针对修改记录的全局锁后才允许提交，没有获得全局锁之前就必须一直等待，这种设计以牺牲一定性能为代价，避免了有两个分布式事务中包含的本地事务修改了同一个数据，从而避免脏写。在读隔离方面，AT 事务默认的隔离级别是读未提交（Read Uncommitted），这意味着可能产生脏读（Dirty Read）。也可以采用全局锁的方案解决读隔离问题，但直接阻塞读取的话，代价就非常大了，一般不会这样做。

## 多个服务使用多个数据源

在多个服务使用多个数据源的场景下，由于CAP理论的显示，我们已经没办法实现强一致性了，只能实现基于BASE理论最终一致性的分布式事务。

### 借助消息队列

#### 本地消息表

此方案的核心是将需要分布式处理的任务通过可靠消息队列的方式来异步执行。消息日志可以存储到本地文本、数据库或消息队列，再通过业务规则自动或人工发起重试，一直到所有业务最终完成。适用于对一致性要求不高的场景，实现这个模型时需要注意重试的幂等。

![img](https://pic.imgdb.cn/item/63834f5d16f2c2beb1ed2a39.png)

#### 事务消息

在RocketMQ等消息队列中实现了[事务消息](https://rocketmq.apache.org/docs/featureBehavior/04transactionmessage)，实际上其实是对本地消息表的一个封装，将本地消息表移动到了MQ内部。

![image-20190726114320498](https://pic.imgdb.cn/item/6383520d16f2c2beb1f273ba.jpg)

### TCC

[TCC](https://database.cs.wisc.edu/cidr/cidr2007/papers/cidr07p15.pdf) 是另一种常见的分布式事务机制，它是“Try-Confirm-Cancel”三个单词的缩写。

在具体实现上，TCC 较为烦琐（参与者需要实现三个接口，这也是他最大的缺点），它是一种业务侵入式较强的事务方案，要求业务处理过程必须拆分为“预留业务资源”和“确认/释放消费资源”两个子过程。如同 TCC 的名字所示，它分为以下三个阶段。

- **Try**：尝试执行阶段，完成所有业务可执行性的检查（保障一致性），并且**预留**好全部需用到的业务资源（保障隔离性）。
- **Confirm**：确认执行阶段，不进行任何业务检查，直接使用 Try 阶段准备的资源来完成业务处理。Confirm 阶段可能会重复执行，因此本阶段所执行的操作需要具备幂等性。
- **Cancel**：取消执行阶段，释放 Try 阶段预留的业务资源。Cancel 阶段可能会重复执行，也需要满足幂等性。

<img src="https://pic.imgdb.cn/item/6383531f16f2c2beb1f5f56a.png" alt="tcc" style="zoom:33%;" />



TCC事务机制相比于上面介绍的XA，解决了其几个缺点：

1. 解决了协调者单点问题，由主业务方发起并完成这个业务活动。业务活动管理器也变成多点，支持集群。 

2. 解决了同步阻塞问题：try阶段不会锁定整个资源，将资源转换为业务逻辑形式，粒度变小。

3. 数据一致性问题，有了补偿机制之后，由业务活动管理器控制一致性。

### SAGA

[SAGA](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) 事务模式的历史十分悠久，其核心思想是将长事务拆分为多个本地短事务，由Saga事务协调器协调，如果正常结束那就正常完成，如果某个步骤失败，则根据相反顺序一次调用补偿操作。

SAGA 由两部分操作组成：

- 大事务拆分若干个小事务，将整个分布式事务 T 分解为 n 个子事务，命名为 T~1~，T~2~，… T~n~。每个子事务都应该是或者能被视为是原子行为。
- 为每一个子事务设计对应的补偿动作，命名为 C~1~，C~2~，… C~n~。T~i~与 C~i~必须满足以下条件：
  - T~i~与 C~i~都具备幂等性。
  - T~i~与 C~i~满足交换律，即先执行 T~i~还是先执行 C~i~，其效果都是一样的。
  - C~i~必须能成功提交，即不考虑 C~i~本身提交失败被回滚的情形，如出现就必须持续重试直至成功，或者要人工介入。

如果 T~1~到 T~n~均成功提交，那事务顺利完成，否则，要采取以下两种恢复策略之一：

- 正向恢复（Forward Recovery）：如果 T~i~事务提交失败，则一直对 T~i~进行重试，直至成功为止。
- 反向恢复（Backward Recovery）：如果 T~i~事务提交失败，则一直执行 C~i~对 T~i~进行补偿，直至成功为止

![Saga模式示意图](https://pic.imgdb.cn/item/6383570a16f2c2beb1fe1986.png)



>  **附录：一些开源的分布式事务组件**
>
> 目前开源的分布式事务组件还不是很多，大多都是公司自研自用，目前由两个比较知名的分布式事务组件
>
> [Seata](https://seata.io/zh-cn/index.html)：阿里内部GTS的开源版本，提供了 AT、TCC、SAGA 和 XA 事务模式支持。
>
> [DTM](https://github.com/dtm-labs/dtm)：DTM 是一款基于 Go 语言实现的开源分布式事务管理器，使用HTTP 协议或 gRPC 协议通讯，已支持 TCC 模式、Saga、二阶段消息模式的分布式事务模式。 DTM 从 2021-06-04 发布 0.1 版本，目前有7.9k star，发展飞快，社区活跃度目前也很高。
