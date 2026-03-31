# 5.2 监控项

Kafka 通过 JMX 暴露内部监控，具体监控项可以查看：https://kafka.apache.org/documentation/#monitoring

这里列出一些需要关注的监控项的 JMX `MBean` 的 `ObjectName`(`domain:properties...`)，以及 jmx_exporter 将其转换后对应的 promql 查询语句：

JMX ObjectName + attrName 需要转换为 promql 指标，默认的转换规则将

    domain:<beanpropertyName1=beanPropertyValue1, beanpropertyName2=beanPropertyValue2, ...>:<key1, key2, ...>:attrName=value

转换为

    domain_beanPropertyValue1_key1_key2_...keyN_attrName{beanpropertyName2="beanPropertyValue2", ...}: value

## 消息速率(KafkaRequestHandler 统计)

客户端与 broker 之间消息写入/读取速率。

    kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec[,topic=*]

    kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec[,topic=*]

    kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec[,topic=*]

> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="MessagesInPerSec",topic=""}
> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="BytesInPerSec", topic=""}
> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="BytesOutPerSec", topic=""}

broker 与 broker 之间副本同步带来的消息写入/读取速率。

    kafka.server:type=BrokerTopicMetrics,name=ReplicationBytesInPerSec

    kafka.server:type=BrokerTopicMetrics,name=ReplicationBytesOutPerSec

> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="ReplicationBytesInPerSec",}
> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="ReplicationBytesOutPerSec",}

broker 重新分区数据迁移速率：

    kafka.server:type=BrokerTopicMetrics,name=ReassignmentBytesInPerSec

    kafka.server:type=BrokerTopicMetrics,name=ReassignmentBytesOutPerSec

拒绝写入速率

    kafka.server:type=BrokerTopicMetrics,name=BytesRejectedPerSec

## 请求频率(KafkaRequestHandler 统计)

以 topic 维度统计请求频率

    kafka.server:type=BrokerTopicMetrics,name=TotalProduceRequestsPerSec,topic=([-.\w]+)

    kafka.server:type=BrokerTopicMetrics,name=TotalFetchRequestsPerSec,topic=([-.\w]+)

> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="TotalProduceRequestsPerSec", }
> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="TotalFetchRequestsPerSec", }

TotalProduceRequestsPerSec 会在消息加入本地副本日志时统计一次

TotalFetchRequestsPerSec 会在从本地读取消息时统计一次


失败请求频率

    kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec,topic=([-.\w]+)

    kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec,topic=([-.\w]+)

> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="FailedProduceRequestsPerSec", }
> kafka_server_BrokerTopicMetrics_OneMinuteRate{name="FailedFetchRequestsPerSec", }

这里的 Fetch 包含 FetchConsumer 与 FetchFollower，即客户端的拉取与副本的拉取请求速率之和。

## 请求频率(RequestChannel 统计)

RequestChannel 会在处理响应时 `processCompletedSends()` 时记录请求频率

    kafka.network:type=RequestMetrics,name=RequestsPerSec,request={Produce|FetchConsumer|FetchFollower}

> kafka_network_RequestMetrics_OneMinuteRate{name="RequestsPerSec",request="Produce",}

Kafka 的 API 定义读取消息只有 FETCH，RequestChannel 在统计时会将请求分为 `FetchConsumer` 和 `FetchFollower`，即客户端消费和副本同步消费。

请求错误频率，统计失败的请求/响应，如果 `error=NONE` 表示响应是成功的。

    kafka.network:type=RequestMetrics,name=ErrorsPerSec,request=([-.\w]+),error=([-.\w]+)

> kafka_network_RequestMetrics_OneMinuteRate{name="ErrorsPerSec", error!="NONE"}


一般 RequestChannel 统计数会小于 KafkaRequestHandler 统计数，区别在于：

- RequestChannel 统计的是网络层面的指标，记录的是 Broker 接收到的请求数量，即每个请求类型（Produce、FetchConsumer、FetchFollower）每秒多少次
- KafkaRequestHandler 统计的是 Server 层面（逻辑层） 的指标，每次 Fetch 或 Produce 操作针对 Topic/Partition 都会累计计数

因此，如果一个 Fetch 请求包含多个 partition：

- RequestChannel 计 1
- KafkaRequestHandler 可能会计 N（分区数）

## 请求大小

    kafka.network:type=RequestMetrics,name=RequestBytes,request=([-.\w]+)

> kafka_network_RequestMetrics_OneMinuteRate{name="RequestBytes", }

## 网络处理线程空闲率

    kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent

## 请求处理线程空闲率

    kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent

## Topic 数量

只有 Controller 节点上有，表示集群的 topic 数量

    kafka.controller:type=KafkaController,name=GlobalTopicCount

## leader 数量

    kafka.server:type=ReplicaManager,name=LeaderCount

> kafka_server_ReplicaManager_Value{name="LeaderCount"}

只有 Controller 节点上有，表示集群的 topic leader partition 数量

    kafka.controller:type=KafkaController,name=GlobalPartitionCount

## 分区数量

    kafka.server:type=ReplicaManager,name=PartitionCount

> kafka_server_ReplicaManager_Value{name="PartitionCount"}

## 有落后副本的分区数量

`|isr| < |all replicas|` 的 partition 数

    kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions

> kafka_server_ReplicaManager_Value{name="UnderReplicatedPartitions"}

`|isr| = min.insync.replicas` 的 partition 数

    kafka.server:type=ReplicaManager,name=AtMinIsrPartitionCount

`|isr| < min.insync.replicas` 的 partition 数

    kafka.server:type=ReplicaManager,name=UnderMinIsrPartitionCount

## 下线分区数

    kafka.controller:type=KafkaController,name=OfflinePartitionsCount

## 落后的 replica 数

    kafka.cluster:type=Partition,name=UnderReplicated,topic=*,partition=*

## 掉线的 replica 数量

当副本因为日志目录文件系统错误而无法访问时，视为 OfflineReplica。

每个 broker 上的 OfflineReplica 数量：

    kafka.server:type=ReplicaManager,name=OfflineReplicaCount

每个 broker 上的无法访问的 LogDir 数量：

    kafka.server:name=OfflineLogDirectoryCount,type=LogManager

## 当前 leader 不是最优的分区数量

    kafka.controller:type=KafkaController,name=PreferredReplicaImbalanceCount

## 落后消息数量

follower 和 leader replica 之间落后的消息数量。

    kafka.server:type=ReplicaFetcherManager,name=MaxLag,clientId=Replica

> kafka_server_ReplicaFetcherManager_Value{name="MaxLag", clientId="Replica"}

具体到 topic partition 级别落后的消息数量

    kafka.server:type=FetcherLagMetrics,name=ConsumerLag,clientId=([-.\w]+),topic=([-.\w]+),partition=([0-9]+)

> kafka_server_FetcherLagMetrics_Value{name="ConsumerLag",clientId="ReplicaFetcherThread-0-4",topic="test",partition="0",}

## Isr 列表扩缩容速率

    kafka.server:type=ReplicaManager,name=IsrShrinksPerSec

> kafka_server_ReplicaManager_OneMinuteRate{name="IsrExpandsPerSec"}

    kafka.server:type=ReplicaManager,name=IsrExpandsPerSec

> kafka_server_ReplicaManager_OneMinuteRate{name="IsrShrinksPerSec"}

## 当前 broker 是否为 controller

只有当 broker 作为集群 controller 时为 1，整个集群应该只有 1 个 controller

    kafka.controller:type=KafkaController,name=ActiveControllerCount

## Leader 选举速率

    kafka.controller:type=ControllerStats,name=LeaderElectionRateAndTimeMs

## Unclean leader 选举速率

    kafka.controller:type=ControllerStats,name=UncleanLeaderElectionsPerSec

## 请求/响应队列

    kafka.network:type=RequestChannel,name=RequestQueueSize

Size of the request queue. A congested request queue will not be able to process incoming or outgoing requests.

    kafka.network:type=RequestChannel,name=ResponseQueueSize

Size of the response queue. The response queue is unbounded. A congested response queue can result in delayed response times and memory pressure on the broker.

## 请求耗时

Total time in ms to serve the specified request.

    kafka.network:type=RequestMetrics,name=TotalTimeMs,request={Produce|FetchConsumer|FetchFollower}

> kafka_network_RequestMetrics_75thPercentile{name="TotalTimeMs",request=~"Produce|FetchConsumer",}

Time the request waits in the request queue.

    kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request is processed at the leader.

    kafka.network:type=RequestMetrics,name=LocalTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request waits for the follower. This is non-zero for produce requests when acks=all.

    kafka.network:type=RequestMetrics,name=RemoteTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request waits in the response queue.

    kafka.network:type=RequestMetrics,name=ResponseQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time to send the response.

    kafka.network:type=RequestMetrics,name=ResponseSendTimeMs,request={Produce|FetchConsumer|FetchFollower}

## 消息格式转换

转换频率

    kafka.server:type=BrokerTopicMetrics,name=ProduceMessageConversionsPerSec,topic=([-.\w]+)

耗时

    kafka.network:type=RequestMetrics,name=MessageConversionsTimeMs,request={Produce|Fetch}

## gc

    java.lang:name=G1 Young Generation,type=GarbageCollector CollectionCount

    java.lang:name=G1 Young Generation,type=GarbageCollector CollectionTime

## kafka purgatory

    kafka.server:type=DelayedOperationPurgatory,delayedOperation=Produce,name=PurgatorySize

Number of requests waiting in the producer purgatory. This should be non-zero when `acks=all` is used on the producer.

    kafka.server:type=DelayedOperationPurgatory,delayedOperation=Fetch,name=PurgatorySize

Number of requests waiting in the fetch purgatory. This is high if consumers use a large value for `fetch.wait.max.ms`.

## partition log

    kafka.log:type=Log,name=Size,topic=([-.\w]+),partition=([0-9]+)

    kafka.log:type=Log,name=NumLogSegments,topic=([-.\w]+),partition=([0-9]+)

    kafka.log:type=Log,name=LogStartOffset,topic=([-.\w]+),partition=([0-9]+)

    kafka.log:type=Log,name=LogEndOffset,topic=([-.\w]+),partition=([0-9]+)

