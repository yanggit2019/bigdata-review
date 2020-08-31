# kafka 精准一次性

## 重要性

在很多的流处理框架的介绍中, 都会说 kafka 是一个可靠的数据源, 并且推荐使用 kafka 当作数据源来进行使用. 这是因为与其他消息引擎系统相比, kafka 提供了可靠的数据保存及备份机制. 并且通过消费者 offset 这一概念, 可以让消费者在因某些原因宕机而重启后, 可以轻易得回到宕机前的位置. 

而 kafka 作为分布式 MQ, 大量用于分布式系统中, 如消息推送系统, 业务平台系统 (如结算平台), 就拿结算来说, 业务方作为上游把数据传输到结算平台, 如果一份数据被计算, 处理了多次, 产生的后果将会特别严重. 消息队列确保写和读的准确是非常重要的, 但 kafka 的**多分区**的特性使之变得困难.

## 哪些因素影响幂等性

在写入一端: 要知道在分布式系统中, 出现网络分区是不可避免的, 如果 kafka broker 在回复 ack 时, 出现网络故障, kafka 挂掉, 节点宕机等情况, producer 将会重发, 如何保证 producer 重试时不造成**重复**或乱序? 或者 producer 挂了, 新的 producer 并没有旧 producer 的状态数据, 这个时候如何保证 exactly-once? 

在读取一端: 即使 kafka 写入的消息满足了幂等性, consumer 拉取到消息后, 把消息交给线程池 workers, workers 线程对 message 的处理可能包含异步操作, 又会出现以下情况: 

- 先 commit, 再执行业务逻辑: 提交成功, 处理失败 . 造成丢失.
- 先执行业务逻辑, 再 commit: 提交失败, 执行成功. 造成重复执行
- 先执行业务逻辑, 再 commit: 提交成功, 异步执行fail. 造成丢失

针对以上的问题, kafka 在 0.11 版新增了幂等型 producer 和事务型 producer. 前者解决了单会话单分区幂等性等问题, 后者解决了多会话多分区幂等性. 

## 总结

总的来讲, 对于写入消息来说, kafka 可以通过设定 ack 为 0 来保证 at most once, 即至多一次写入, 也就是不重复, 但这样可能因为写入失败而丢失数据. 当然也可以通过设定 ack 为 -1 来保证 at leat once, 即至少一次写入, 也就是不丢失数据, 但这样可能因为副本同步成功后 leader 分区挂掉而无法向 producer 返回 ack 而导致 producer 重复生产数据. kafka 在 0.11 版本引入了幂等性和事务, 幂等性和 `ack = -1` 可以保证单会话单分区的写入的精准一次性, 再加上 kafka 写入事务可以保证跨会话跨分区的精准一次性. 而消费端的幂等性需要结合下游处理消息系统的机制去考虑, 比如是否有事务机制, 是否天然去重或幂等性.

# 消息丢失

## 生产者丢失

前面介绍 kafka 分区和副本的时候, 有提到过一个 producer 客户端有一个 acks 的配置: 

**acks = 0**

producer 是发送之后不管的, 这个时候就很有可能因为网络等原因造成数据丢失, 所以应该尽量避免. 

**acks = 1**

这时 leader 写入成功会发送 ack, 所以有可能在 leader 接收到数据, 但还没同步给其他副本的时候就挂掉了, 这时候数据也是丢失了. 这种时候是客户端以为消息发送成功, 但 kafka 丢失了数据. 

**acks = -1**

要达到最严格的无消息丢失配置, 需要配置 `acks = -1` , 这时会等待 ISR 所有副本同步完成再返回 acks. 

但此时还有可能 ISR 中只有 leader 一个, 写入 leader 后就返回了 acks, 这时出现 leader 宕机, 其他副本也没有同步, 最终导致数据丢失, 变成了和 `ack = 1` 一样的情况.

所以一般 `ack = -1` 需要同时配置 `min.insync.replicas > 1`, 保证 ISR 中至少有一个非 leader 副本.

**同时还需要使用带有回调的 producer api, 来发送数据**. 注意这里讨论的都是异步发送消息, 同步发送不在讨论范围. 

```java
public class send{
    ......
    public static void main(){
        ...
        /*
        *  第一个参数是 ProducerRecord 类型的对象, 封装了目标 Topic, 消息的 kv
        *  第二个参数是一个 CallBack 对象, 当生产者接收到 kafka 发来的 ack 确认消息的时候, 
        *  会调用此 CallBack 对象的 onCompletion() 方法, 实现回调功能
        */
         producer.send(new ProducerRecord<>(topic, messageNo, messageStr),
                        new DemoCallBack(startTime, messageNo, messageStr));
        ...
    }
    ......
}

class DemoCallBack implements Callback {
    /* 开始发送消息的时间戳 */
    private final long startTime;
    private final int key;
    private final String message;

    public DemoCallBack(long startTime, int key, String message) {
        this.startTime = startTime;
        this.key = key;
        this.message = message;
    }

    /**
     * 生产者成功发送消息, 收到 kafka 服务端发来的 ack 确认消息后, 会调用此回调函数
     * @param metadata 生产者发送的消息的元数据, 如果发送过程中出现异常, 此参数为 null
     * @param exception 发送过程中出现的异常, 如果发送成功为 null
     */
    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        if (metadata != null) {
            System.out.printf("message: (%d, %s) send to partition %d, offset: %d, in %d\n",
                    key, message, metadata.partition(), metadata.offset(), elapsedTime);
        } else {
            exception.printStackTrace();
        }
    }
}
```

更详细的代码可以参考这里: [kafka生产者分析——kafkaproducer](https://link.zhihu.com/?target=https%3A//binglau7.github.io/2017/12/18/kafka%E7%94%9F%E4%BA%A7%E8%80%85%E5%88%86%E6%9E%90%E2%80%94%E2%80%94kafkaproducer/). 

我们之前提到过, producer 发送到 kafka broker 的时候, 是有多种可能会失败的, 而回调函数能准确告诉你是否确认发送成功, 当然这依托于 acks 和 `min.insync.replicas` 的配置. 而当数据发送丢失的时候, 就可以进行手动重发或其他操作, 从而确保生产者发送成功. 

## kafka 内部丢失

有些时候, kafka 内部因为一些不大好的配置, 可能会出现一些极为隐蔽的数据丢失情况: 

- `replication.factor` 配置参数, 这个配置决定了副本的数量, 默认是1. 注意这个参数不能超过broker的数量. 说这个参数其实是因为如果使用默认的 1, 或者不在创建 topic 的时候指定副本数量 (也就是副本数为1) , 那么当一台机器出现磁盘损坏等情况, 那么数据也就从 kafka 里面丢失了. 所以 replication.factor这个参数最好是配置大于 1, 比如说 3. 

- `unclean.leader.election.enable` 参数, 这个参数是在主副本挂掉, 然后在ISR集合中没有副本可以成为leader的时候, 要不要让进度比较慢的副本成为leader的. 不用多说, 让进度比较慢的副本成为leader, 肯定是要丢数据的. 虽然可能会提高一些可用性, 但如果你的业务场景丢失数据更加不能忍受, 那还是将 `unclean.leader.election.enable=false` 吧. 

## 消费者丢失

消费者丢失的情况, 其实跟消费者位移处理不当有关. 消费者位移提交有一个参数, `enable.auto.commit`, 默认是 `true`, 决定是否要让消费者自动提交位移. 如果开启, 那么 consumer 每次都是先提交位移, 再进行消费, 比如先跟broker说这 5 个数据我消费好了, 然后才开始慢慢消费这 5 个数据. 

这样处理的话, 好处是简单, 坏处就是漏消费数据, 比如你说要消费 5 个数据, 消费了2个自己就挂了. 那下次该consumer 重启后, 在 broker 的记录中这个 consumer 是已经消费了5个的. 

所以最好的做法就是配置 `enable.auto.commit=false`, 改为手动提交位移, 在每次消费完之后再手动提交位移信息. 当然这样又有可能会重复消费数据, 毕竟 exactly-once 处理一直是一个问题呀. 遗憾的是 kafka 目前没有保证 consumer 幂等消费的措施, 如果确实需要保证 consumer 的幂等, 可以对每条消息维持一个全局的 id, 每次消费进行去重, 当然耗费这么多的资源来实现 exactly-once 的消费到底值不值, 那就得看具体业务了. 

## 无消息丢失小结

那么到这里先来总结下无消息丢失的主要配置吧: 

**具体生产者机制以及参数说明见  [Kafka Producer](https://www.devtalking.com/articles/kafka-practice-4/), 这篇文章非常好.**

### 生产者端

1

`retries = MAX_VALUE`  

无限重试，直到你意识到出现了问题. `retries` 定义了生产者收到异常后重试的次数, 默认为 0, 另一个与之相关的参数是 `retry.backoff.ms`, 决定了两次重试之间的时间间隔, 默认为 100ms.

2

`max.in.flight.requests.per.connection = 1`

这个参数默认是 5, 意思是在被 Broker 阻止前, 未通过 acks 确认的发送请求最大数, 也就是在 Broker 处排队等待 acks 确认的 Message 数量。所以刚才那个场景，第一条和第二条 Message 都在 Broker 那排队等待确认放行，这时第一条失败了，等重试的第一条 Message 再来排队时，第二条早都通过进去了，所以排序就乱了。

如果想在设置了 `retries` 还要严格控制 Message 顺序，可以把 `max.in.flight.requests.per.connection` 设置为 1。让 Broker 处永远只有一条 Message 在排队，就可以严格控制顺序了。但是这样做会严重影响性能（接收 Message 的吞吐量）。

3

producer 的 `acks=-1`, 同时 `min.insync.replicas>1`, 并且使用带有回调的 producer api 发生消息 `KafkaProducer.send(record, callback)`. 

4

callback 逻辑中显式关闭 producer：close(0)  注意: 设置此参数是为了避免消息乱序

5

`replication.factor>1`, 或者创建 topic 的时候指定大于1的副本数. 

6

`min.insync.replicas = 2` 且 `replication.factor > min.insync.replicas`. 如果两者相等, 当一个副本挂掉了分区也就没法正常工作了. 通常设置 `replication.factor = min.insync.replicas + 1` 即可.

7

`unclean.leader.election.enable=false`, 防止定期副本 leader 重新选举. 关闭 unclean leader 选举，即不允许非 ISR 中的副本被选举为 leader，以避免数据丢失.

### 消费者端

自动提交位移 `enable.auto.commit=为false`, 在消费完后手动提交位移. 



# exactly-once

## 幂等性

幂等这个词最早起源于函数式编程, 意思是一个函数无论执行多少次都会返回一样的结果. 比如说让一个数加1就不是幂等的, 而让一个数取整就是幂等的. 因为这个特性所以幂等的函数适用于并发的场景下. 

但幂等在分布式系统中含义又做了进一步的延申, 比如在kafka中, 幂等性意味着一个消息无论重复多少次, 都会被当作一个消息来持久化处理. 

## at most once 和 at least once

最多一次就是保证一条消息只发送一次, 这个其实最简单, 异步发送一次然后不管就可以, 缺点是容易丢数据, 所以一般不采用. 

至少一次语义是 kafka 默认提供的语义, 它保证每条消息都能至少接收并处理一次, 缺点是可能有重复数据. 

前面有介绍过 ack 机制, 当设置 producer 客户端的 ack 是 1 或 -1 的时候:

- broker 接收到消息就会跟 producer 确认. 但producer发送一条消息后, 可能因为网络原因消息超时未达, 这时候producer客户端会选择重发, broker回应接收到消息, 但很可能最开始发送的消息延迟到达, 就会造成消息重复接收. 
- broker 写入成功但发送 ack 时 leader 宕机了, 导致 producer 没有收到 ack 从而重新发送消息, 导致消息重复.

## 单会话幂等的 producer

kafka 的 producer 默认是支持最少一次语义, 也就是说不是幂等的, 这样在一些比如支付等要求精确数据的场景会出现问题, 在 0.11.0 后, kafka提供了让 producer 支持幂等的配置 `props.put("enable.idempotence", ture)`.

在创建 producer 客户端的时候, 添加这一行配置, producer 就变成幂等的了. 注意开启幂等性的时候, ack就自动是 -1 了, 如果这时候手动将 ack 设置为 0, 那么会报错. 

kafka 的幂等性实现其实就是将原来下游需要做的去重放在了数据上游. 为解决 producer 重试引起的乱序和重复. kafka 增加了 pid 和 seq. 开启幂等性的 producer 在初始化的时候会被分配一个 PID. producer 中发往同一Partition的每个 RecordBatch 都有一个单调递增的 seq; Broker 端而会对 `<PID, Partition, Seq>` 做缓存, 每 Commit 都会更新 lastSeq. 这样 recordBatch 到来时, broker 会先检查 recordBatch 再保存数据: 如果 batch 中 baseSeq (第一条消息的 seq) 比 broker 维护的序号 (lastSeq) 大 1, 则保存数据, 否则不保存 (inSequence方法). 即当具有相同主键的消息提交时, broker 只会持久化一条.

> producerStateManager.scala

但是! 幂等的 producer 也并非万能. 有两个主要是缺陷: 

- 幂等性的 producer 仅做到单分区上的幂等性, 即单分区消息不重复, 多分区无法保证幂等性. 
- 只能保持单会话的幂等性, 无法实现跨会话的幂等性, 也就是说如果 producer 挂掉再重启, pid 会变化, 无法保证两个会话间的幂等 (新会话可能会重发) . 因为 broker 端无法获取之前的状态信息, 所以无法实现跨会话的幂等. 

此处引申**kafka producer 对有序性做了哪些处理**.

## 跨会话事务的 producer

Kafka 从 0.11 版本开始引入了事务支持. 事务可以保证 Kafka 在 Exactly Once 语义的基础上, 生产和消费可以跨分区和会话, 要么全部成功, 要么全部失败.

在单会话幂等性中介绍, kafka 通过引入 pid 和 seq 来实现单会话幂等性, 但正是引入了 pid, 当应用重启时, 新的 producer 并没有 old producer 的状态数据, 可能重复保存. 当遇到上述幂等性的缺陷无法解决的时候, 可以考虑使用事务了. 事务可以支持多分区的数据完整性, 原子性. 并且支持跨会话的 exactly-once 处理语义, 也就是说如果 producer 宕机重启, 依旧能保证数据只处理一次. 

幂等性解决了单会话单分区的精准一次性, kafka producer transaction 结合之前的幂等性, 就可以跨会话, 跨分区做到**精准一次性地写入**.

为了实现跨分区跨会话的事务, 需要引入一个全局唯一的 Transaction ID (用户给的), 并将 Producer 获得的 pid 和 transaction ID 绑定. 这样当 producer 重启后就可以通过正在进行的 transaction ID 获得原来的 pid. 

为了管理 Transaction，Kafka 引入了一个新的组件 Transaction Coordinator. Producer 就是通过和Transaction Coordinator 交互获得 Transaction ID 对应的任务状态。Transaction Coordinator 还负责将事务所有写入 Kafka 的一个内部 Topic, 这样即使整个服务重启, 由于事务状态得到保存, 进行中的事务状态可以得到恢复, 从而继续进行. 

开启事务也很简单, 首先需要开启幂等性, 即设置 `enable.idempotence=true`. 然后对 producer 发送代码做一些小小的修改. 

```java
//初始化事务
producer.initTransactions();
try {
    //开启一个事务
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    //提交
    producer.commitTransaction();
} catch (kafkaException e) {
    //出现异常的时候, 终止事务
    producer.abortTransaction();
}
```

**kafka 事务通过隔离机制来实现多会话幂等性**

kafka 事务引入了 transaction.id 和 Epoch, 设置 transactional.id后, 一个 transactionId 只对应一个 pid, 且Server 端会记录最新的 Epoch 值. 这样有新的 producer 初始化时, 会向 TransactionCoordinator 发送 InitPIDRequest 请求,  TransactionCoordinator 已经有了这个 transactionId 对应的 meta, 会返回之前分配的 PID, 并把 Epoch 自增 1 返回, 这样当 old producer 恢复过来请求操作时, 将被认为是无效 producer 抛出异常.  如果没有开启事务, TransactionCoordinator 会为新的 producer 返回 new pid, 这样就起不到隔离效果, 因此无法实现多会话幂等. 

```scala
private def maybeValidateAppend(producerEpoch: Short, firstSeq: Int, offset: Long): Unit = {
    validationType match {
      case ValidationType.None =>

      case ValidationType.EpochOnly =>
        checkProducerEpoch(producerEpoch, offset)

      case ValidationType.Full => //开始事务, 执行这个判断
        checkProducerEpoch(producerEpoch, offset)
        checkSequence(producerEpoch, firstSeq, offset)
    }
}

private def checkProducerEpoch(producerEpoch: Short, offset: Long): Unit = {
    if (producerEpoch < updatedEntry.producerEpoch) {
      throw new ProducerFencedException(s"Producer's epoch at offset $offset is no longer valid in " +
        s"partition $topicPartition: $producerEpoch (request epoch), ${updatedEntry.producerEpoch} (current epoch)")
    }
  }
```

但无论开启幂等还是事务的特性, 都会对性能有一定影响, 这是必然的. 所以kafka默认也并没有开启这两个特性, 而是交由开发者根据自身业务特点进行处理. 

## consumer 端幂等性

对于 Consume 而言, 事务的保证就会相对较弱, 尤其是无法保证 Commit 的信息被精确消费. 这是由于 Consumer 可以通过 offset 访问任意信息, 而且不同的 Segment File 生命周期不同, 同一事务的消息可能会出现重启后被删除的情况.

### kafka 消息重复消费场景

consumer 消费了数据之后, 每隔一段时间, 会把自己消费过的消息的 offset 提交一下, 代表我已经消费过了, 下次我要是重启啥的, 你就让我继续从上次消费到的 offset 来继续消费吧. 

但是凡事总有意外, 比如我们之前生产经常遇到的, 就是你有时候重启系统, 看你怎么重启了, 如果碰到点着急的, 直接kill进程了, 再重启. 这会导致 consumer 有些消息处理了, 但是没来得及提交 offset, 尴尬了. 重启之后, 少数消息会再次消费一次. 

![img](https://pic2.zhimg.com/80/v2-b316ae7a505798e4d67ad2be290b4a3d_1440w.jpg)

如上所述, consumer 拉取到消息后, 把消息交给线程池 workers, workers 对 message 的 handle 可能包含异步操作, 又会出现以下情况: 

- 先 commit, 再执行业务逻辑: 提交成功, 处理失败 . 造成丢失
- 先执行业务逻辑, 再 commit: 提交失败, 执行成功. 造成重复执行
- 先执行业务逻辑, 再 commit: 提交成功, 异步执行 fail. 造成丢失

### 如何保证消息重复消费后的幂等性

其实重复消费不可怕, 可怕的是你没考虑到重复消费之后, 怎么保证幂等性. 

![img](https://pic3.zhimg.com/80/v2-3a45a3e9601826738c9752bd7f033998_1440w.jpg)

举个例子吧. 假设你有个系统, 消费一条往数据库里插入一条, 要是你一个消息重复两次, 你不就插入了两条, 这数据不就错了? 但是你要是消费到第二次的时候, 自己判断一下已经消费过了, 直接扔了, 不就保留了一条数据? 

一条数据重复出现两次, 数据库里就只有一条数据, 这就保证了系统的幂等性.

**那所以第二个问题来了, 怎么保证消息队列消费的幂等性?**

对此我们常用的方法时, workers 取到消息后先执行如下代码: 

```java
if(cache.contain(msgId)){
  // cache中包含msgId, 已经处理过
    continue;
}else {
  lock.lock();
  cache.put(msgId,timeout);
  commitSync();
  lock.unLock();
}
// 后续完成所有操作后, 删除cache中的msgId, 只要msgId存在cache中, 就认为已经处理过. Note: 需要给cache设置有消息
```

其实还是得结合业务来思考, 我这里给几个思路: 

1. 比如你拿个数据要写库, 你先根据主键查一下, 如果这数据都有了, 你就别插入了, update 一下好吧

2. 比如你是写 redis, 那没问题了, 反正每次都是 set, 天然幂等性

3. 比如你不是上面两个场景, 那做的稍微复杂一点, 你需要让生产者发送每条数据的时候, 里面加一个全局唯一的 id, 类似订单 id 之类的东西, 然后你这里消费到了之后, 先根据这个 id 去比如 redis 里查一下, 之前消费过吗? 如果没有消费过, 你就处理, 然后这个 id 写 redis. 如果消费过了, 那你就别处理了, 保证别重复处理相同的消息即可. 

4. 还有比如基于数据库的唯一键 (见文 6) 来保证重复数据不会重复插入多条, 我们之前线上系统就有这个问题, 就是拿到数据的时候, 每次重启可能会有重复, 因为 kafka 消费者还没来得及提交 offset, 重复数据拿到了以后我们插入的时候, 因为有唯一键约束了, 所以重复数据只会插入报错, 不会导致数据库中出现脏数据

# 引申: kafka producer 对有序性做了哪些处理

假设我们有 5 个请求, batch1, batch2, batch3, batch4, batch5；如果只有batch2 ack failed, 3, 4, 5 都保存了, 那 2 将会随下次 batch 重发而造成重复. 我们可以设置 `max.in.flight.requests.per.connection=1` (客户端在单个连接上能够发送的未响应请求的个数) 来解决乱序, 但降低了系统吞吐. 

新版本 kafka 设置 `enable.idempotence=true` 后能够动态调整 `max.in.flight.requests.per.connection`. 默认情况下 `max.in.flight.requests.per.connection=5`. 当重试请求到来且时, batch 会根据 seq 重新添加到队列的合适位置, 并配置 `max.in.flight.requests.per.connection=1`, 这样它前面的 batch 序号都比它小, 只有前面的都发完了, 它才能发. 

# References

1. [Kafka生产者分析——KafkaProducer]([https://binglau7.github.io/2017/12/18/Kafka%E7%94%9F%E4%BA%A7%E8%80%85%E5%88%86%E6%9E%90%E2%80%94%E2%80%94KafkaProducer/](https://binglau7.github.io/2017/12/18/Kafka生产者分析——KafkaProducer/))
2. [Kafka生产者分析——RecordAccumulator]([https://binglau7.github.io/2017/12/24/Kafka%E7%94%9F%E4%BA%A7%E8%80%85%E5%88%86%E6%9E%90%E2%80%94%E2%80%94RecordAccumulator/](https://binglau7.github.io/2017/12/24/Kafka生产者分析——RecordAccumulator/))
3. [大厂面试kafka, 一定会问到的幂等性](https://zhuanlan.zhihu.com/p/77302957)
4. [kafka实现无消息丢失与精确一次语义 (exactly once) 处理](https://zhuanlan.zhihu.com/p/113630718)
5. [阿里一面面经详解: kafka消息重复消费场景, 你怎么看待996? ](https://zhuanlan.zhihu.com/p/128493705)
6. [唯一索引（unique index）和非唯一索引（普通索引）（index） 区别](https://blog.51cto.com/57388/1868848)
7. [Kafka无消息丢失配置](https://www.cnblogs.com/huxi2b/p/6056364.html)