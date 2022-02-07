# Broker 参数配置

## broker id and rack

    # integer 类型，默认 -1。
    # 手动设置应该从 0 开始，每个 broker 依次 +1，手动设置的值不能超过 reserved.broker.max.id
    broker.id=0

    # 如果配置文件中没有指定 broker.id，broker 会自动生成一个 broker.id，默认从 reserved.broker.max.id+1 开始
    reserved.broker.max.id=1000

    # string 类型，默认 null。
    # topic partition 的 replica 分布在不同的 broker 上，但这些 broker 可能在同一个机架/机房/区域内，
    # 如果想让 replica 在不同机架/机房/区域内分布，可以将不同机架/机房/区域的 broker 配置不同的 broker.rack，
    # 配置后，topic partition 的 replica 会分布在不同的 broker.rack
    broker.rack=null

## 网络和 io 操作线程配置

    num.network.threads=9

broker处理消息的最大线程数(主要处理网络io，读写缓冲区数据，基本没有io等待，配置线程数量为cpu核数加1)

    num.io.threads=16

broker处理磁盘IO的线程数(处理磁盘io操作，高峰期可能有些io等待，因此配置需要大些。配置线程数量为cpu核数2倍，最大不超过3倍)

    socket.request.max.bytes=2147483600

socket server可接受数据大小(防止OOM异常)，根据自己业务数据包的大小适当调大。这里取值是int类型的，而受限于java int类型的取值范围

> java int的取值范围为（-2147483648~2147483647）

## log 数据文件刷盘策略

    # 每当producer写入10000条消息时，刷数据到磁盘
    log.flush.interval.messages=10000

    # 每间隔1秒钟时间，刷数据到磁盘
    log.flush.interval.ms=1000

为了大幅度提高producer写入吞吐量，需要定期批量写文件。一般无需改动，如果topic的数据量较小可以考虑减少 `log.flush.interval.ms` 和 `log.flush.interval.messages` 来强制刷写数据，减少可能由于缓存数据未写盘带来的不一致。推荐配置分别message 10000，间隔1s。

> Kafka官方并不建议通过Broker端的log.flush.interval.messages和log.flush.interval.ms来强制写盘，认为数据的可靠性应该通过Replica来保证，而强制Flush数据到磁盘会对整体性能产生影响。

## 日志保留策略配置

    # 日志保留时长
    log.retention.hours=72

日志建议保留三天，也可以更短

## 日志文件

    # 段文件大小
    log.segment.bytes=1073741824

段文件配置1GB，有利于快速回收磁盘空间，重启kafka加载也会加快。

kafka启动时会加载目录(log.dir)下所有数据文件，如果段文件过小，则文件数量比较多。

    # 启动时每个文件夹对应的线程数
    num.recovery.threads.per.data.dir=1

增加 `num.recovery.threads.per.data.dir` 也可以提高加载速度。

## replica复制配置

    # 拉取线程数：fetchers 配置多可以提高follower的I/O并发度，单位时间内leader持有更多请求，相应负载会增大，需要根据机器硬件资源做权衡，建议适当调大；
    num.replica.fetchers=3

    # 拉取消息最小字节：一般无需更改，默认值即可；
    replica.fetch.min.bytes=1

    # 拉取消息最大字节：默认为1MB，根据业务情况调整
    replica.fetch.max.bytes=5242880

    # 拉取消息等待时间：决定 follower 的拉取频率，频率过高，leader会积压大量无效请求情况，无法进行数据同步，导致cpu飙升。配置时谨慎使用，建议默认值，无需配置。
    replica.fetch.wait.max.ms

## 分区数量配置

    num.partitions=1

默认 partition 数量 1，如果topic在创建时没有指定partition数量，默认使用此值。Partition的数量选取也会直接影响到Kafka集群的吞吐性能，配置过小会影响消费性能。

## replica 数配置

    default.replication.factor=1

这个参数指新创建一个topic时，默认的Replica数量，Replica过少会影响数据的可用性，太多则会白白浪费存储资源，一般建议在2~3为宜。

## replica lag

    replica.lag.time.max.ms=10000
    replica.lag.max.messages=4000

## auto rebalance

    auto.leader.rebalance.enable=true
    leader.imbalance.check.interval.seconds=300
    leader.imbalance.per.broker.percentage=10

## offset retention

    offsets.retention.check.interval.ms = 600000
    offsets.retention.minutes = 1440

## 时间戳

0.10.0.0 版本后，kafka 消息增加了 timestamp 字段。

    log.message.timestamp.type=CreateTime/LogAppendTime

producer 写入的消息可以设置 timestamp 字段，(如果没有指定，则使用 producer 客户端的当前时间)。

时间戳有 2 中类型：

1. `CreateTime`: 
2. `LogAppendTime`: 使用 broker 当前时间覆盖 消息携带的 timestamp，

时间戳类型为 `CreateTime`，允许 create time 与当前时间最大的时间差：

    log.message.timestamp.difference.max.ms=9223372036854775807
