# 可以看作是官方文档的~~简单翻译~~渣翻
版本：2.10.0  
始：2022-06-06  
终：  
状态：DOING
# 1. 概念与架构
## 1.1. 预览（Overview）
Pulsar是一个多租户，高性能的服务器间消息传递解决方案，最早由雅虎开发，现在Pulsar由[Apache软件基金会](https://www.apache.org)管理。  

罗列一下Pulsar的特性：
- 本地的一个Pulsar实例中支持多集群部署，集群间可以做到跨地域无缝复制消息。
- 拥有极低地发布和端到端延迟。
- 简单的客户端API，支持Java，Go，Python和C++
- 支持topic多种订阅类型（独占、共享和灾备）
- 通过[Apache BookKeeper](https://bookkeeper.apache.org)提供的消息持久化机制保证消息的传递。
- 由轻量级的serverless computing框架Pulsar Functions，实现了流原生的数据处理
- 拥有基于Pulsar Function的serverless connector框架 Pulsar IO，其能够使数据更好的迁入移除Apache Pulsar。
- 当数据老化时，通过分层存储，将数据从热存储转移到冷存储（例如S3和GCS）。
## 1.2 消息传递（Messaging）
Pulsar是基于 [发布-订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 模式(也可以缩写为pub-sub)。在这种模式下，producers发布消息到topics中；consumers订阅这些topic，处理传入的消息，并且当处理消息成功地结束时发送一个ack给broker。  

当一个订阅被创建时，Pulsar会保留所有的消息，即使consumer断开链接。只有当某一个消费者成功处理完毕这些消息，发送了ack后，这些被保留下来的消息才会被丢弃。  

如果一个消息消费失败，并且你希望这个消息能够被再次消费，你可以启用消息重新传递机制来要求broker重新发送这些消息。
### 1.2.1 消息（Message）
消息（Message）是Pulsar的基本“单位”。下表列出了消息包含的一些组成信息。  
|组成（Component）|描述（Description）|
|:---:|:---|
|Value/data payload|消息携带数据。所有Pulsar消息都包含原始字节，即使消息也可以符合数据模式。|
|Key|消息可以随意地被key所标记，这对一些事情十分有用，比如topic压缩。|
|Properties|用户自定义的键值对（可选）。|
|Producer name|标记着生产这个消息的生产者名称，如果未设置生产者名称，将会使用默认生产者名称。|
|Topic name|标记着这个消息会被发往哪个topic中。|
|Schema version|标记着生产消息时使用的schema版本号。|
|Sequence ID|在topic中，每个Pulsar消息都属于一个有序序列，消息的序列ID可以由producer初始化，来指明其在序列中的顺序，也可以自定义。</br>序列ID可以用来消除重复的消息。如果 `brokerDeuplicationEnabled`被设置为`true`的话，那么每个消息的序列ID在每个topic（非分区）或者一个分区中唯一。|
|Message ID|消息的消息ID会在被bookies持久化时分配。消息ID指明了消息在ledger中的特殊位置，以及其在Pulsar集群中是唯一的。|
|Publish time|一个消息被发布时的时间戳，时间戳会自动由producer赋值。|
|Event time|一个由应用程序赋值给消息的可选时间戳。比如，应用程序可以选择在这个消息被处理时，给这个属性赋上一个时间。如果没有设置event time，它的值为`0`|

消息的最大默认大小为 5MB 。你可以在配置中设置消息的最大大小。
- 在`broker.conf`文件中
```
# message的最大长度（byte）
maxMessageSize=5242880
```
- 在`bookkeeper.conf`文件中
```
# netty的最大大小（byte），接收到的任何大于此值的消息都将被拒绝。默认值为5MB。
nettyMaxFrameSizeBytes=5253120
```
对于更多的Pulsar消息的信息，可以查看Pulsar binary protocol，
### 1.2.2 生产者（Producers）
生产者是一个与topic建立连接，并且可以把消息发布到Pulsar broker上的进程。Pulsar broker将会处理这些消息。
#### 1.2.2.1 发送模式（Send Modes）
生产者可以选择同步发送（sync）或者异步发送（async）
|模式（Mode）|描述（Description）|
|:---------:|:-----------------|
|同步发送（Sync）|producer每次发送消息后都会等待broker返回ack。如果producer没有收到ack，会将此次发送视为失败。|
|异步发送（Async）|producer会将消息放入阻塞队列中并且马上返回。客户端在后台将消息发送给broker。如果队列满了（可以在配置中设置最大size），当调用API时producer会被阻塞或者立马失败，这取决于传递给producer的参数。|
#### 1.2.2.2 访问模式（Access Mode）
producer对于topic可以有不同的访问模式
|访问模式（Access mode）|描述（Description）|
|:--------------------:|:-----------------|
|`共享（shared）`|多个producers可以向同一个topic进行消息发布。</br></br>这是**默认**的设置。|
|`独占（Exclusive）`|一个topic只能有一个producer进行消息发布。</br></br>如果已经有一个producer连接了该topic，其他producers尝试往这个topic上发布消息时会立马提示错误。</br></br>当“旧”producer与broker发生网络分区时，“旧”producer会被剔除，“新”producer会被选为下一个独占对象。|
|`等待独占（WaitForExclusive）`|如果已经有一个producer连接了该topic，那么新producer的连接会被挂起（而不是超时），直到新producer获取到`独占（Exclusive）`访问权。</br></br>获得到独占访问权的producer被视为leader。因此，如果你想要让你的应用实现leader选举方案，你可以使用这种访问模式。|
> #### ！注意
> 一旦一个应用程序成功获取到了`独占（Exclusive）`或`等待独占（WaitForExclusive）`的访问模式，那么可以保证该topic**只会被这个应用实例写入**。任何其他producers尝试该topic生产消息时都会得到一个错误响应或者等待直到它们得到`等待独占（WaitForExclusive）`的访问模式。更多信息，请查看PIP 68: Exclusive Producer。
你可以通过Java客户端API来设置producer的访问模式，对于更多信息，可以查看ProducerBuilder.java文件中的`ProducerAccessMode`。
#### 1.2.2.3 压缩（Compression）
你可以在producer发布消息的过程中进行消息压缩。Pulsar目前支持以下几种压缩类型。
- [LZ4](https://github.com/lz4/lz4)
- [ZLIB](https://zlib.net/)
- [ZSTD](https://facebook.github.io/zstd/)
- [SNAPPY](https://google.github.io/snappy/)
#### 1.2.2.4 批量处理（Batching）
当启用批量处理时，producer会将消息积累起来，在一个request中将这些消息一起发送。批量处理的量大小取决于最大消息数和最大发布延迟。因此，积压数量是批量处理的总数，而不是消息的总数。  

在Pulsar中，批（batch）作为存储和追踪的基本单位，而不是单个消息作为存储和追踪的基本单位。Consumer将一个batch拆解为单个消息。但是，即使开启了批处理，延时消息（被配置了参数`deliverAt`或`deliverAfter`）始终会被当但一个独立的消息进行发送。  

通常情况下，当一个consumer确认了batch中的所有消息，这个batch才会被视为确认。这意味着如果**没有**将一个batch中的所有消息进行确认（如意料之外的失败、否定确认或者是确认超时），那么该batch中的所有消息将会被重新发送，即使有部分消息已经被确认过了。

为了避免重新将batch中已经被确认的消息发送给consumer，Pulsar从2.6.0版本开始引入了批量索引确认（batch index acknowledgement）。当批量索引确认启用时，consumer会过滤掉那些已经确认过的batch index，并将这些batch index发送给broker。broker维护且追踪每个batch index的ack状态以防止向consumer发送那些已被确认过的消息。只有当batch中的所有消息被确认时，batch才会被删除。  

默认情况下，批量索引确认是禁用的（`acknowledgmentAtBatchIndexLevelEnable=false`）。你可以在broker端设置参数`acknowledgmentAtBatchIndexLevelEnable`为`true`来启用它。启用批量索引确认会带来更多的内存开销。
#### 1.2.2.5 分块（Chunking）
消息分块能够使Puslar在producer端将消息进行分块，在consumer端将聚合分块消息，这样能够很好的处理大型负载消息。  

当消息分块启用时，当消息的大小超过了允许的最大载荷（即在broker处的参数配置`maxMessageSize`），消息的工作流会如下所示：
1. producer端将原始消息拆分为分块消息，并且将他们与分块元数据（metadata）单独分开，按顺序发布到broker上。
2. broker会将分块消息同其他普通消息一样，放在一个managed-ledger上，并且会使用`chunkedMessageRate`参数来记录这个topic中分块消息的速度。
3. consumer端缓存分块消息，并且当收到了一个消息的所有分块时，会将它们聚合起来，放入receiver queue中。
4. 客户端消费从receiver queue中聚合的数据。
##### 局限性
- 分块只对**持久化**的topic有用。
- 分块只对**独占**和**灾备**的订阅类型有用。
- 分块无法与**批处理（batching）** 同时启用。
#### 1.2.2.6 消费者有序处理连续的分块消息
下图显示了拥有一个producer的topic，producer向topic发送一批大的分块消息和普通的非分块消息。Producer发布消息M1，M1有三个分块M1-C1，M1-C2和M1-C3，Broker会在managed-ledger中存储这三个分块消息，并且把他们以同样的顺序传输到consumer上（consumer为独占或灾备类型）。Consumer在内存中缓存收到的分块消息，当收到所有分块消息时，会将它们聚合成一整个消息M1，然后将原始消息M1发送给客户端。
<div align="center">
  <img src="/imgs/producer/chunking-01.png"></img>
</div>

#### 1.2.2.7 消费者有序处理不连续的分块消息
当多个消费者往同一个topic发布多个分块消息时，Broker会将所有来自不同的producer的分块消息保存到同一个managed-ledger中。分块们在managed-ledger可以不连续。如下图所示，Producer1发布了消息M1并且分为了三个块M1-C1，M1-C2和M1-C3。Producer2发布了消息M2并且分为了三个块M2-C1，M2-C2以及M2-C3。所有分块消息在该消息内是连续的，但在managed-ledger内可能不是连续的。
<div align="center">
  <img src="/imgs/producer/chunking-02.png"></img>
</div>

> #### ！注意
> 在这种情况下，不连续的分块消息可能会给consumer端带来一定的内存压力，因为consumer需要为每个大消息保留一定的缓冲区来将这些分块消息合并为原来的大消息。你可以通过配置参数`maxPendingChunkedMessage`来限制consumer端并发维持的最大消息分块数。当达到阈值时，consumer将会丢弃pending的消息通过静默ack或者要求broker过一段时间重新传递，以此来优化内存利用率。

#### 1.2.2.8 启用消息分块
**必要条件**：通过设置`enableBatching`参数为`false`来禁用批处理（batching）。  

消息分块的特性默认是关闭的。如果要启用它，可以在创建producer时设置`chunkingEnabled`参数为`true`。

> #### ！注意
> 如果consumer未能在指定时间能收到一个消息的所有分块，那么这些已收到的分块会过期。默认时间时1分钟。有关`expireTimeOfIncompleteChunkedMessage`参数的更多信息，可以查看org.apache.pulsar.client.api.

### 1.2.3 消费者（Consumers）
消费者是一个通过订阅topic与其建立连接，然后接收消息的进程。  

一个consumer向broker发送流许可申请（flow permit request）来获取消息。在消费者端有一个队列，用来接收从broker端推送过来的消息。你可以设置参数`receiverQueueSize`来设置这个队列的最大值（默认大小是`1000`）。每当`consumer.receive()`被调用时，就会从缓冲区获取一条消息。
#### 1.2.3.1 接收模式（Receive modes）
消息可以从brokers处异步（async）或同步（sync）接收。
|模式（Mode）|描述（Description）|
|:---------:|:-----------------|
|同步接收|同步接收在消息之前会一直被阻塞。|
|异步接收|异步接收会立马返回一个future value，例如Java中的`CompletableFuture`，当收到消息时，这个future会被立刻完成。|
#### 1.2.3.2 监听（Listeners）
客户端为consumer提供了监听器的实现类。例如，Java客户端提供了MessageListener接口。当收到一个新消息时，该接口下的`received`方法会被调用。
#### 1.2.3.3 确认（Acknowledgement）
当consumer成功消费一个消息时，它会向broker发送一个确认。这条消息是被永久保存的，只有当所有订阅者都确认了这条消息，这条消息才会被删除。如果你希望这些消息被consumer确认后依旧保留下来，你需要配置消息保留策略。  

对于批量消息，你可以启用批量索引确认来防止已经被ack的消息重复发送给消费者。详细内容请查看批量处理（batching）。

可以通过下面两种方式之一来进行消息确认：
- 独立消息确认。使用独立消息确认，消费者会为每一条消息发送一个ack给broker。
- 累计消息确认。使用累计消息确认，消费者**只会**确认收到的最后一个消息。所有之前（包含此条）的消息都不会再被发送给这个consumer（类似tcp的ack？）

如果你想使用独立消息确认，可以使用下面的API。
```
consumer.acknowledge(msg);
```
如果你想用累计消息确认，可以使用下面的API。
```
consumer.acknowledgeCumulative(msg);
```
> #### ！注意
> 累计消息确认无法用于共享订阅类型，因为共享订阅类型涉及到多个consumers，这些consumers可以访问到同一个订阅，如果使用累计消息确认，可能会将别的consumer消费的消息给确认掉。在共享订阅类型中，消息总是独立确认。
#### 1.2.3.4 否定确认（Negative acknowledgement）
否定确认机制允许你向broker发送一个提醒，这个提醒意味着consumer没有处理消息。当一个consumer消费消息失败，并且需要重新消费它时，consumer会向broker发送一个否定确认（nack），这会让broker重新发送这条消息给consumer。  

消息可以否定地独立确认，也可以否定地累计确认，这取决于消费者使用的订阅模式。  

在独占和灾备订阅类型下，consumers只会对他们收到的最后一条消息进行否定确认。  

在共享或者Key共享的订阅类型下，consumers可以对消息进行单条的否定确认。  

请注意，对于已有排序的订阅类型（如独占、灾备和Key共享模式）进行否定确认，可能会导致失败的消息没办法像原始顺序一样发送给consumers。  

如果你打算对消息使用否定确认，请确保你的否定确认在确认超时之前发送出去。  

使用下面的API来进行否定确认。  
```
Consumer<byte[]> consumer = pulsarClient.newConsumer()
                            .topic()
                            .subscriptionName("sub-negative-ack")
                            .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                            .negativeAckRedeliveryDelay(2, TimeUnit.SECONDS)// the default value is 1 min
                            .subscribe();
                       
Message<byte[]> message = consumer.receive();

// call the API to send negative acknowledgement
consumer.negativeAcknowledge(message);

message = consumer.receive();
consumer.acknowledge(message);
```
为了让那些重新传递的消息有不同的延迟，你可以通过**重传补偿机制（Negative Redelivery Backoff）** 来设置最大重试传递消息次数。使用下面的API来启用`Negative Redelivery Backoff`。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer()
                            .topic(topic)
                            .subscriptionName("sub-negative-ack")
                            .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                            .negativeAckRedeliveryBackoff(MultiplierRedeliveryBackoff.builder()
                                                          .minDelayMs(1000)
                                                          .maxDelayMs(60 * 1000)
                                                          .build())
                            .subscribe();
```
消息的重传行为会像下表一样。
|重传次数|重传延迟|
|:------|:------|
|1|10 + 1 seconds|
|2|10 + 2 seconds|
|3|10 + 4 seconds|
|4|10 + 8 seconds|
|5|10 + 16 seconds|
|6|10 + 32 seconds|
|7|10 + 60 seconds|
|8|10 + 60 seconds|

> #### ！注意
> 如果批处理（batching）开启，一批内的消息全部都会被重新传递给consumer。
#### 1.2.3.5 确认超时（Acknowledgement timeout）
确认超时机制允许你设置一个范围时间，consumer客户端会去追踪未确认的消息。当这些消息超过了确认时间（`ackTimeout`），客户端会发送`redeliver unacknowledged messages`请求给broker，然后broker会重新发送这些未被确认的消息给consumer。  

你可以配置确认超时机制来重传那些超过了`ackTimeout`却未ack的消息，或者运行一个timer task去检查在每个`ackTimeoutTickTime`间ack超时的消息。  

你也可以使用重传补偿机制（redelivery backoff mechanism），通过不同的延迟来多次重传消息。  

如果你想要用重传补偿，你可以使用如下API。
```
consumer.ackTimeout(10, TimeUnit.SECOND)
        .ackTimeoutRedeliveryBackoff(MultiplierRedeliveryBackoff.builder()
        .minDelayMs(1000)
        .maxDelayMs(60000)
        .multiplier(2).build())
```
消息的重传行为会像下表一样。
|重传次数|重传延迟|
|:------|:------|
|1|10 + 1 seconds|
|2|10 + 2 seconds|
|3|10 + 4 seconds|
|4|10 + 8 seconds|
|5|10 + 16 seconds|
|6|10 + 32 seconds|
|7|10 + 60 seconds|
|8|10 + 60 seconds|

> #### ！注意
> - 如果批处理（batching）开启，一批内的消息全部都会被重新传递给consumer。
> - 与确认超时相比，使用否定确认会更好。首先，确认超时很难设置一个超时值。其次，当broker重新发送这些确认超时的消息时，也许这些消息不需要再被重复消费了（超时，并没有消费失败）。

使用如下API来启用确认超时。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer()
                .topic(topic)
                .ackTimeout(2, TimeUnit.SECONDS) // the default value is 0
                .ackTimeoutTickTime(1, TimeUnit.SECONDS)
                .subscriptionName("sub")
                .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                .subscribe();

Message<byte[]> message = consumer.receive();

// wait at least 2 seconds
message = consumer.receive();
consumer.acknowledge(message);
```
#### 1.2.3.6 消息重试topic（Retry letter topic）
重试topic允许你保存一些被consumer消费失败的数据，并且过一段时间consumer会再次尝试消费它们。在这种方式下，你可以自定义每个消息的重传间隔时间。Consumers在所在的原始topic也会自动订阅消息重试topic。一旦某个消息超过了重试最大次数，并且这个消息还是没法被消费时，它会被移动至死信topic（dead letter topic）。  

消息重试topic的概念如下图。
<div align="center">
  <img src="/imgs/consumer/retry-letter-topic.svg"></img>
</div>
使用消息重试topic与使用消息延迟传递不同，即使他们俩都旨于过一段时间消费消息。消息重试topic是处理消费失败的消息，以确保这些数据不会丢失，而延迟消息传递是单纯的为了在未来的某个特定时间点进行消息传递。  

默认情况下，消息自动重试是关闭的。你可以设置`enableRetry`为`true`来为consumer启用消息重试。  

如何使用消息重试API可以参考下面示例，当重试次数达到`maxRedeliverCount`时，未被消费的消息会被移动至死信topic。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                .topic("my-topic")
                .subscriptionName("my-subscription")
                .subscriptionType(SubscriptionType.Shared)
                .enableRetry(true)
                .deadLetterPolicy(DeadLetterPolicy.builder()
                        .maxRedeliverCount(maxRedeliveryCount)
                        .build())
                .subscribe();
```
默认的消息重试topic会以下面格式命名。
```
<topicname>-<subscriptionname>-RETRY
```
使用Java客户端来命名你的消息重试topic
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
        .topic("my-topic")
        .subscriptionName("my-subscription")
        .subscriptionType(SubscriptionType.Shared)
        .enableRetry(true)
        .deadLetterPolicy(DeadLetterPolicy.builder()
                .maxRedeliverCount(maxRedeliveryCount)
                .retryLetterTopic("my-retry-letter-topic-name")
                .build())
        .subscribe();
```
在消息重试topic中的消息会有一些特殊的属性，这些属性由客户端自动创建
|特殊属性|描述|
|:------|:---|
|`REAL_TOPIC`|对应的实际topic名称。|
|`ORIGIN_MESSAGE_ID`|原始的消息ID，这对消息追踪很重要。|
|`RECONSUMETIMES`|消息重试次数|
|`DELAY_TIME`|消息重试间隔时间|
##### 例子
```
REAL_TOPIC = persistent://public/default/my-topic
ORIGIN_MESSAGE_ID = 1:0:-1:0
RECONSUMETIMES = 6
DELAY_TIME = 3000
```
使用下面的API将消息放在重试队列中。
```
consumer.reconsumeLater(msg, 3, TimeUnit.SECONDS);
```
使用下面的API来为`reconsumerLater`函数增加一些自定义参数属性。在下一次消费时，自定义的属性可以通过message#getProperty获取。
```
Map<String, String> customProperties = new HashMap<String, String>();
customProperties.put("custom-key-1", "custom-value-1");
customProperties.put("custom-key-2", "custom-value-2");
consumer.reconsumeLater(msg, customProperties, 3, TimeUnit.SECONDS);
```
> #### ！注意
> - 目前，消息重试topic可以在共享订阅类型中启用。
> - 与否定确认（nack）相比，消息重试topic更适合需要大量重试的消息。因为在消息重试topic中的消息会被持久化到BookKeeper中，而那些由于被nack，需要重传的消息被缓存在客户端。
#### 1.2.3.7 死信topic（Dead letter topic）
死信topic允许你继续消费消息，即使一些消息没法被成功消费。这些被消费失败的消息会被保存到一个特殊的topic中，这个topic称为死信topic。你可以决定如何处理死信topic中的数据。  

在Java客户端中启用默认的死信topic。
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                .topic("my-topic")
                .subscriptionName("my-subscription")
                .subscriptionType(SubscriptionType.Shared)
                .deadLetterPolicy(DeadLetterPolicy.builder()
                      .maxRedeliverCount(maxRedeliveryCount)
                      .build())
                .subscribe();
```
死信topic的默认以下格式
```
<topicname>-<subscriptionname>-DLQ
```
使用Java客户端来自定义你的死信topic名
```
Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                .topic("my-topic")
                .subscriptionName("my-subscription")
                .subscriptionType(SubscriptionType.Shared)
                .deadLetterPolicy(DeadLetterPolicy.builder()
                      .maxRedeliverCount(maxRedeliveryCount)
                      .deadLetterTopic("my-dead-letter-topic-name")
                      .build())
                .subscribe();
```
默认情况下，在DLQ创建的时候不会有任何订阅存在，这可能会导致你丢失消息。为了给DLQ自动初始化创建一个订阅，你可以自定义`initialSubscriptionName`参数。如果设置了这个参数，但broker的`allowAutoSubscriptionCreation`是禁用的，那么DLQ producer将会创建失败。  

死信topic目前会由确认超时、否定确认或者消息重传topic触发。
> #### ！注意
> 目前来说，死信toic允许在share和key_share的订阅类型下使用。
### 1.2.4 主题（Topics）
像其他发布-订阅的系统一样，Pulsar中的topic是将producers的消息运输给consumers间的通道。Topic有良好的URL命名风格。
```
{persistent|non-persistent}://tenant/namespace/topic
```
|Topic名称组成|描述|
|:-----------|:--|
|`persistent`/`non-persistent`|用来标识topic的类型。Pulsar支持两种类型的topic：persistent和non-persistent。默认情况下是persistent，所以你如果你没有指定某种类型，那么这个topic为persistent类型。persistent的topic，所有消息都会被持久化到硬盘上（如果broker非单机，那么这些消息还会被持久化到多块硬盘上），然而non-persistent的topics不会被持久化到硬盘上。|
|`tenant`|实例中的topic租户。租户对Pulsar的多租户至关重要，并分布在集群中。|
|`namespace`|将相关联的topic作为一个组来管理，是管理Topic的基本单元。大多数topic的配置都是在namespace的级别执行。 每个租户里面可以有一个或者多个namespace。|
|`topic`|名称的最后一部分。topic名称在Pulsar实例中没有特殊意义。|
#### 不需要去显示创建新topics
在Pulsar中，你不需要去显示创建topics。当一个客户端尝试去往一个不存在的topic写/从一个不存在的topic中读的时候，Pulsar会在提供的topic名称中的命名空间下自动创建一个topic。如果客户端创建topic时未指定tenant或namespace，那么该topic会被自动创建到default tenant和namespace中。你也可以指定tenant和namespace创建topic，比如`persistent://my-tenant/my-namespace/my-topic`.`persistent://my-tenant/my-namespace/my-topic`意味着`my-topic`这个topic会被创建在命名空间为`my-namespace`以及租户名称为`my-tenant`下。
### 1.2.5 命名空间（Namespaces）
命名空间是一个租户（tenant）下的逻辑术语。一个tenant通过admin API创建命名空间（namespace）。举个例子，一个tenant有不同的应用，那么可以为每个应用创建一个namespace来将他们隔离开来。namespace允许应用创建和管理多个topics。topic `my-tenant/app1`是`my-tenant`租户下，应用程序为`app1`的命名空间。你可以在一个namespace下创建多个topic。
### 1.2.6 订阅（Subscriptions）
订阅是一种配置规则，用于决定如何将消息传递给消费者。在Pulsar中，有四种订阅规则可以使用：exclusive（独占），shared（共享），failover（灾备）和key_shared（key共享）。这些类型如下图所示。
<div style="margin: 0 auto">
  <img src="/imgs/subscription/subscription-types.png" />
</div>

> #### Pub-Sub或Queuing
> 在Pulsar中，你可以灵活地使用不同的订阅类型
> - 如果你想要在消费者中实现传统的“fan-out pub-sub messaging”，你可以为每个消费者subscription设定一个独一无二的名称。这是独占订阅类型。
> - 如果你想要在消费者中实现“message queuing”，可以在多个消费者中共享相同的subscription（shared，failover，key_shared）
> - 如果你想要实现这两种效果，请将exclusive与其他订阅类型相结合。
 
#### 订阅类型（Subscription types）
当一个subscription没有consumer时，它的订阅类型是未定义的（undefined）。当一个consumer连入subscription时，它的类型才会被定义，可以通过更改消费者配置并重启所有消费来修改订阅类型。

##### 独占（Exclusive）
在**独占（Exclusive）** 模式下，只有单个consumer被允许链接到subscription中。如果多个consumer使用同一个subscription订阅了topic，那么会发生错误。注意如果topic被分区了，那么所有分区的消息只会被一个consumer消费。
> exclusive是默认的订阅类型
<div style="margin: 0 auto">
  <img src="/imgs/subscription/exclusive.png" />
</div>

##### 灾备（Failover）
在**灾备（Failover）** 模式下，多个消费者可以链接到同一个subscription。可以为非分区或每个分区选择一个主消费者来接受消息。当主消费者失去连接时，所有（未确认和后续）的消息都将会传递给下一个消费者。

对于存在分区的topic，broker将会对消费者通过优先级和名称字典顺序进行排序。然后broker将会尝试将不同分区内的消息平均分配给高优先级的consumer。

对于未分区的topic，broker将会按照订阅的顺序选择consumer。

例如：一个topic有15个分区以及3个consumer。每个consumer会消费5个分区，每个分区都会有1个活跃的consumer以及4个等待的consumer。

在下图中，**Consumer-B-0**是主consumer，当它失去连接时，**Consumer-B-1**会代替他去接收消息。
<div style="margin: 0 auto">
  <img src="/imgs/subscription/failover.png" />
</div>

##### 共享（Shared）
在*shared*或*round robin*类型中，多个consumers可以连接到同一个subscription中。消息会通过轮询的方式传递给各个消费者，也有一些指定的消息会被发送给指定的消费者。当一个消费者失去连接，所有已经发送给它但未确认的消息将重新安排发送给其余消费者。

在下图中，**Consumer-C-1** 和 **Consumer-C-2** 可以订阅到同一个topic，当然 **Consumer-C-3** 和其他消费者也可以。

> ###### Shared类型的限制
> 当使用Shared类型时，请注意
> - 消息的顺序无法被保证
> - 无法使用累计消息确认
<div style="margin: 0 auto">
  <img src="/imgs/subscription/shared.png" />
</div>

##### Key共享（Key_Shared）
在Key_Shared类型下，多个消费者可以连接到同一个subscription中，消息会在consumers中分发传递，具有相同key的消息只会发给一个consumer。无论这个消息被重新发送过多少次，它都只会发送给相同的consumer。当一个一个消费者连接或失去连接时，这会导致服务消费者更改一部分消息key。
<div style="margin: 0 auto">
  <img src="/imgs/subscription/key-shared.png" />
</div>

注意当consumers使用Key_Shared的订阅类型时，你需要对producers做出 **禁用batching**或**使用key-based batching**的操作。理由如下：
- 1.broker通过消息的key来进行消息分发，但默认的batching可能无法将具有相同key的消息打包到同一批中。
- 2.batch中的第一个消息的key作为整个batch中所有消息的key，这可能会导致一些上下文错误。
- 
key-based batching就是用于解决上述问题。这个batching会保证producers将具有相同key的消息保存到同一batch中。没有key的消息将会打包到同一个batch中，这个batch没有key。当broker分发这些batch信息时，他会使用`NON_KEY`来作为key。此外，每个consumer只会收到一个key或者收到一个相同key的batch信息。默认情况下，你可以限制producer往batch中打包信息的数量。
下面是在Key_Shared订阅类型下使用key-based batching的例子，使用pulsar client：
```
Producer<byte[]> producer = client.newProducer()
                                .topic("my-topic")
                                .batcherBuilder(BatcherBuilder.KEY_BASED)
                                .create();
```

> ###### Key_Shared类型的限制
> 当你使用Key_Shared类型时，请注意：
> - 你需要为消息指定一个key
> - 你无法使用累计消息确认
