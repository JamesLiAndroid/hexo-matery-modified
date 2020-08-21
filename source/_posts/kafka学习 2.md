# kafka学习 2

## 压缩策略

两种例外情况就可能让 Broker 重新压缩消息。

情况一：Broker 端指定了和 Producer 端不同的压缩算法。
情况二：Broker 端发生了消息格式转换

Kafka 会将启用了哪种压缩算法封装进消息集合中
Producer 端压缩、Broker 端保持、Consumer 端解压缩。


解压缩对 Broker 端性能 是有一定影响的，特别是对 CPU 的使用率而言。

四种压缩算法：GZIP、Snappy、LZ4、zstd

即在吞吐量方面：LZ4 > Snappy > zstd 和 GZIP；在压缩比方面，zstd > LZ4 > GZIP > Snappy

何时启用压缩？

1. Producer端完成压缩：Producer 程序运 行机器上的 CPU 资源要很充足

2. CPU 资源充足这一条件，如果你的环境中带宽资源有限，建议开启压缩，建议你开启 zstd 压缩，这样能极大地节省网络资源消耗

解压缩问题：

有条件的话尽量保证 不要出现消息格式转换的情况

## 无消息丢失

原则：
Kafka 只对“已提交”的消息（committed message）做有限度的持久化保证。

已提交的消息：若干个 Broker 成 功地接收到一条消息并写入到日志文件后，它们会告诉生产者程序这条消息已成功提交。

有限度的持久化保证：Kafka 不可能保证在任何情况下 都做到不丢失消息。


* Producer丢失消息

Kafka 依然不认为这条消息属于已提交消息，故对它不做任何持久化保证。

执行完一个操作 后不去管它的结果是否成功。调用 producer.send(msg) 就属于典型的“fire and forget”

解决方式：Producer 永远要使用带有回调通知的发送 API，也就是说不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)

* 消费者程序丢失数据

Consumer 端的位移数据，容易造成数据丢失。没有真正地确认消息是否真的被消费就“盲目”地更新了位移。

解决方式：维持先消费消息（阅读），再更新位移（书签）的顺序。如果是多线程异步处理消费消息，Consumer 程序不要开 启自动提交位移，而是要应用程序手动提交位移。

总结：

1. 不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一 定要使用带有回调通知的 send 方法。
2. 设置 acks = all。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。 如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。 这是最高等级的“已提交”定义。
3. 设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到 的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。
4. 设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪 些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么 它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false， 即不允许这种情况的发生。
5. 设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将 消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。
6. 设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入 到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千 万不要使用默认值 1。
7. 确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂 机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要 在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。
8. 确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设 置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程 处理的场景而言是至关重要的。

问题：

Kafka 还有一种特别隐秘的消息丢失场景：增加主题分区。当增加主题分区后，在某 段“不凑巧”的时间间隔后，Producer 先于 Consumer 感知到新增加的分区，而 Consumer 设置的是“从最新位移处”开始读取消息，因此在 Consumer 感知到新分区 前，Producer 发送的这些消息就全部“丢失”了，或者说 Consumer 无法读取到这些消 息。严格来说这是 Kafka 设计上的一个小缺陷，你有什么解决的办法吗？

解答：新增分区之后，producer先感知并发送数据，消费者后感知，消费者 的offset会定位到新分区的最后一条消息，如果你配置了auto.offset.reset=latest就会这样的

## 拦截器

preHandler、postHandler

多个拦截器可用ArrayList串联：
props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptors);

org.apache.kafka.clients.producer.ProducerInterceptor  

onSend：该方法会在消息发送之前被调用
onAcknowledgement：该方法会在消息成功提交或发送失败之后被调用。（onAcknowledgement 的调用要早于 callback 的调用。值得注意的是，这个方法和 onSend 不是在同一个线程中被调用的， 因此如果你在这两个方法中调用了某个共享可变对象，一定要保证线程安全哦。还有一 点很重要，这个方法处在 Producer 发送的主路径中，所以最好别放一些太重的逻辑进 去，否则你会发现你的 Producer TPS 直线下降。）

org.apache.kafka.clients.consumer.ConsumerInterceptor

onConsume：该方法在消息返回给 Consumer 程序之前调用。
onCommit：Consumer 在提交位移之后调用该方法。

应用场景：Kafka 拦截器可以 应用于包括客户端监控、端到端系统性能检测、消息审计等多种功能在内的场景。

## 生产者如何管理TCP连接

Kafka 的所有通信都是基于 TCP 的

作为一个基于报文的协议，TCP 能够被用于多 路复用连接场景的前提是，上层的应用协议（比如 HTTP） 允许发送多条消息。

何时创建：在创建 KafkaProducer 实例时，生产者应用会 在后台创建并启动一个名为 Sender 的线程，该 Sender 线 程开始运行时首先会创建与 Broker 的连接。

TCP 连接是在创建 KafkaProducer 实例时建立的。TCP 连接还可能在两个地方被创建：一个是在更 新元数据后，另一个是在消息发送时。

何时关闭：一种是用户主动 关闭；一种是 Kafka 自动关闭（connections.max.idle.ms 的值有关）。

## 消息交付可靠性保障

* 最多一次（at most once）：消息可能会丢失，但绝不会 被重复发送。
* 至少一次（at least once）：消息不会丢失，但有可能被 重复发送。
* 精确一次（exactly once）：消息不会丢失，也不会被重 复发送。

Kafka 是怎么做到精确一次：幂等性（Idempotence）和事务 （Transaction）

幂等性：安全重试，反正它们也不会破坏我们的系统状态

在命令式编程语言（比如 C）中，若一个子程序是幂等 的，那它必然不能修改系统状态。这样不管运行这个子程 序多少次，与该子程序关联的那部分系统状态保持不变。 在函数式编程语言（比如 Scala 或 Haskell）中，很多纯 函数（pure function）天然就是幂等的，它们不执行任 何的 side effect。

开启：props.put(“enable.idempotence”, ture) 或者 props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CO NFIG，true)

缺陷：只能保证单分区上的幂等性

事务性：实现多分区以及多会话上的消息无重复

提供的安全性保障是经典的 ACID，即原子性（Atomicity）、一致性 (Consistency)、隔离性 (Isolation) 和持久性 (Durability)
隔离性表明并发执行的事务彼此相互隔离，互不影响

开启：开启 enable.idempotence = true。设置 Producer 端参数 transctional.id

幂等性Producer只能保证单分区、单会话上的消息幂等性；而事务能够保证跨分区、跨会话间的幂等性。

## 消费者组

Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制.

1. Consumer Group 下可以有一个或多个 Consumer 实 例。这里的实例可以是一个单独的进程，也可以是同一进 程下的线程。在实际场景中，使用进程更为常见一些。
2. Group ID 是一个字符串，在一个 Kafka 集群中，它标识 唯一的一个 Consumer Group。
3. Consumer Group 下所有实例订阅的主题的单个分区， 只能分配给组内的某个 Consumer 实例消费。这个分区 当然也可以被其他的 Group 消费。

点对点模型和发布 / 订阅模型

Consumer Group机制：如果所有实例都属于同一个 Group， 那么它实现的就是消息队列模型；如果所有实例分别属于不同的 Group，那么它实现的就是发布 / 订阅模型。

确定Consumer的实例数量：理想情况下，Consumer 实例的数量应该等于该 Group 订阅主题的分区总数。

位移管理：对于 Consumer Group 而言，它是一组 KV 对，Key 是分区，V 对应 Consumer 消费该分区的最新位移。如果用

新版本采用了将位移 保存在 Kafka 内部主题的方法。这个内部主题就是让人既爱又恨的 __consumer_offsets。

重平衡：Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 Consumer 如何达成一致，来分配订阅 Topic 的每个分区。比如某个 Group 下有 20 个 Consumer 实例，它订阅了一个具有 100 个分区的 Topic。正常情况下，Kafka 平均会为每个 Consumer 分配 5 个分区。这个分配的过程就叫 Rebalance。

发生条件：

1. 组成员数发生变更。比如有新的 Consumer 实例加入组 或者离开组，抑或是有 Consumer 实例崩溃被“踢 出”组。
2. 订阅主题数发生变更。Consumer Group 可以使用正则 表达式的方式订阅主题，比如
consumer.subscribe(Pattern.compile(“t.*c”)) 就表 明该 Group 订阅所有以字母 t 开头、字母 c 结尾的主 题。在 Consumer Group 的运行过程中，你新创建了一 个满足这样条件的主题，那么该 Group 就会发生 Rebalance。
3. 订阅主题的分区数发生变更。Kafka 当前只能允许增加一 个主题的分区数。当分区数增加时，就会触发订阅该主题 的所有 Group 开启 Rebalance。

缺陷：效率低，STW僵死问题。

