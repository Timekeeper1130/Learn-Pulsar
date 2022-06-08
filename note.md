# 可以看作是官方文档的简单翻译
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
- 支持topic多种订阅模式（独占、共享和灾备）
- 通过[Apache BookKeeper](https://bookkeeper.apache.org)提供的消息持久化机制保证消息的传递。
- 由轻量级的serverless computing框架Pulsar Functions，实现了流原生的数据处理
- 拥有基于Pulsar Function的serverless connector框架 Pulsar IO，其能够使数据更好的迁入移除Apache Pulsar。
- 当数据老化时，通过分层存储，将数据从热存储转移到冷存储（例如S3和GCS）。
## 1.2 消息传递（Messaging）
Pulsar是基于 [发布-订阅](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 模式(也可以缩写为pub-sub)。在这种模式下，producers发布消息到topics中；consumers订阅这些topic，处理传入的消息，并且当处理消息成功地结束时发送一个ack给broker。  

当一个订阅被创建时，Pulsar会保留所有的消息，即使consumer断开链接。只有当某一个消费者成功处理完毕这些消息，发送了ack后，这些被保留下来的消息才会被丢弃。  

如果一个消息消费失败，并且你希望这个消息能够被再次消费，你可以启用消息重新传递机制来要求broker重新发送这些消息。
### 消息（Message）
消息（Message）是Pulsar的基本“单位”。下表列出了消息包含的一些组件信息。  
|组件（Component）|描述（Description）|
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
### 生产者（Producers）
生产者是一个与topic建立连接，并且可以把消息发布到Pulsar broker上的进程。Pulsar broker将会处理这些消息。
#### 发送模式（Send Modes）
生产者可以选择同步发送（sync）或者异步发送（async）
|模式（Mode）|描述（Description）|
|:---------:|:-----------------|
|同步发送（Sync）|producer每次发送消息后都会等待broker返回ack。如果producer没有收到ack，会将此次发送视为失败。|
|异步发送（Async）|producer会将消息放入阻塞队列中并且马上返回。客户端在后台将消息发送给broker。如果队列满了（可以在配置中设置最大size），当调用API时producer会被阻塞或者立马失败，这取决于传递给producer的参数。|
#### 访问模式（Access Mode）
producer对于topic可以有不同的访问模式
|访问模式（Access mode）|描述（Description）|
|:--------------------:|:-----------------|
|`共享（shared）`|多个producers可以向同一个topic进行消息发布。</br></br>这是**默认**的设置。|
|`独占（Exclusive）`|一个topic只能有一个producer进行消息发布。</br></br>如果已经有一个producer连接了该topic，其他producers尝试往这个topic上发布消息时会立马提示错误。</br></br>当“旧”producer与broker发生网络分区时，“旧”producer会被剔除，“新”producer会被选为下一个独占对象。|
|`等待独占（WaitForExclusive）`|如果已经有一个producer连接了该topic，那么新producer的连接会被挂起（而不是超时），直到新producer获取到`独占（Exclusive）`访问权。</br></br>获得到独占访问权的producer被视为leader。因此，如果你想要让你的应用实现leader选举方案，你可以使用这种访问模式。|
> #### ！小记     
> 一旦一个应用程序成功获取到了`独占（Exclusive）`或`等待独占（WaitForExclusive）`的访问模式，那么可以保证该topic**只会被这个应用实例写入**。任何其他producers尝试该topic生产消息时都会得到一个错误响应或者等待直到它们得到`等待独占（WaitForExclusive）`的访问模式。更多信息，请查看PIP 68: Exclusive Producer。
你可以通过Java客户端API来设置producer的访问模式，对于更多信息，可以查看ProducerBuilder.java文件中的`ProducerAccessMode`。
#### 压缩（Compression）
你可以在producer发布消息的过程中进行消息压缩。Pulsar目前支持以下几种压缩类型。
- [LZ4](https://github.com/lz4/lz4)
- [ZLIB](https://zlib.net/)
- [ZSTD](https://facebook.github.io/zstd/)
- [SNAPPY](https://google.github.io/snappy/)
#### 批量处理（Batching）
当启用批量处理时，producer会将消息积累起来，在一个request中将这些消息一起发送。批量处理的量大小取决于最大消息数和最大发布延迟。因此，积压数量是批量处理的总数，而不是消息的总数。  

在Pulsar中，批（batch）作为存储和追踪的基本单位，而不是单个消息作为存储和追踪的基本单位。Consumer将一个batch拆解为单个消息。但是，即使开启了批处理，延时消息（被配置了参数`deliverAt`或`deliverAfter`）始终会被当但一个独立的消息进行发送。  

通常情况下，当一个consumer确认了batch中的所有消息，这个batch才会被视为确认。这意味着如果**没有**将一个batch中的所有消息进行确认（如意料之外的失败、否定确认或者是确认超时），那么该batch中的所有消息将会被重新发送，即使有部分消息已经被确认过了。

为了避免重新将batch中已经被确认的消息发送给consumer，Pulsar从2.6.0版本开始引入了批量索引确认（batch index acknowledgement）。当批量索引确认启用时，consumer会过滤掉那些已经确认过的batch index，并将这些batch index发送给broker。broker维护且追踪每个batch index的ack状态以防止向consumer发送那些已被确认过的消息。只有当batch中的所有消息被确认时，batch才会被删除。  

默认情况下，批量索引确认是禁用的（`acknowledgmentAtBatchIndexLevelEnable=false`）。你可以在broker端设置参数`acknowledgmentAtBatchIndexLevelEnable`为`true`来启用它。启用批量索引确认会带来更多的内存开销。
#### 分块（Chunking）
消息分块能够使Puslar在producer端将消息进行分块，在consumer端将聚合分块消息，这样能够很好的处理大型负载消息。  

当消息分块启用时，当消息的大小超过了允许的最大载荷（即在broker处的参数配置`maxMessageSize`），消息的工作流会如下所示：
1. producer端将原始消息拆分为分块消息，并且将他们与分块元数据（metadata）单独分开，按顺序发布到broker上。
2. broker会将分块消息同其他普通消息一样，放在一个管理台账上，并且会使用`chunkedMessageRate`参数来记录这个topic中分块消息的速度。
3. consumer端缓存分块消息，并且当收到了一个消息的所有分块时，会将它们聚合起来，放入receiver queue中。
4. 客户端消费从receiver queue中聚合的数据。
##### 局限性
- 分块只对持久化的topic有用。
- 分块只对独占和灾备的订阅类型有用。
- 分块无法与批处理（batching）同时启用。
#### 消费者有序处理连续的分块消息
