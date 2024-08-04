# Broker 参数配置

## broker id and rack

    # integer 类型，默认 -1
    # 手动设置应该从 0 开始，每个 broker 依次 +1，手动设置的值不能超过 reserved.broker.max.id
    broker.id=0

    # 如果配置文件中没有指定 broker.id，broker 会自动生成一个 broker.id，默认从 reserved.broker.max.id+1 开始
    reserved.broker.max.id=1000

    # string 类型，默认 null
    # topic partition 的 replica 分布在不同的 broker 上，但这些 broker 可能在同一个机架/机房/区域内，
    # 如果想让 replica 在不同机架/机房/区域内分布，可以将不同机架/机房/区域的 broker 配置不同的 broker.rack，
    # 配置后，topic partition 的 replica 会分布在不同的 broker.rack
    broker.rack=null

## 网络和 io 操作线程配置

    num.network.threads=9

broker 用于接收来自网络的请求并向网络发送响应的线程数，对应 kafka 的 Processor 线程数，处理已连接 socket 数据读写

> 可以动态调整，每次动态调整范围为 [currentSize / 2, currentSize * 2]

    num.io.threads=16

broker 处理请求的线程数，对应 kafka 的 Handler 线程数，进行实际的 kafka 动作

> 可以动态调整，每次动态调整范围为 [currentSize / 2, currentSize * 2]

    socket.receive.buffer.bytes=102400

broker server socket 的 SO_RCVBUF 大小，默认 100kB

## log 数据文件刷盘策略

    # 每当producer写入10000条消息时，刷数据到磁盘
    log.flush.interval.messages=10000

    # 每间隔1秒钟时间，刷数据到磁盘
    log.flush.interval.ms=1000

> Kafka 官方并不建议通过 Broker 端的 log.flush.interval.messages 和 log.flush.interval.ms 来强制写盘，认为数据的可靠性应该通过 Replica 来保证，而强制 flush 数据到磁盘会对整体性能产生影响

## 日志保留策略配置

    # 日志保留时长
    log.retention.hours=72

    # 单个 partition 的日志保留大小
    log.retention.bytes

    # 检查日志是否需要清理的时间间隔
    log.retention.check.interval.ms

    # 从文件系统中删除文件之前等待的时间
    file.delete.delay.ms=60000

## 日志文件

    # 段文件大小，默认 1G
    log.segment.bytes=1073741824

    # 启动时加载每个文件夹内 segment 文件对应的线程数
    num.recovery.threads.per.data.dir=1

    # 即使段文件大小没有达到回滚的大小，超过此时间设置，段文件也会回滚
    log.roll.hours

## replica复制配置

    # 连接其他 broker 拉取线程数
    num.replica.fetchers=1

注意这里是连接每个 broker 的线程数，也就是说，当 fetcher 线程数设置为 x 时，如果集群有 n 个节点，每个节点有 x * (n - 1) 个 fetcher 用来连接其他 n - 1 个节点

> 可以动态调整，每次动态调整范围为 [currentSize / 2, currentSize * 2]

    # 拉取消息最小字节
    replica.fetch.min.bytes=1

    # 拉取消息最大字节，默认为1MB，根据业务情况调整
    replica.fetch.max.bytes=5242880

    # 拉取消息等待时间
    replica.fetch.wait.max.ms

## 分区数量配置

    num.partitions=1

默认 partition 数量 1，如果 topic 在创建时没有指定 partition 数量，默认使用此值

## replica 数配置

    default.replication.factor=1

默认的 replica 数量，如果 topic 在创建时没有指定 partition 数量，默认使用此值

## replica lag

    replica.lag.time.max.ms=10000
    replica.lag.max.messages=4000

## auto rebalance

    # 启用自动平衡 leader
    auto.leader.rebalance.enable=true

    # 检查自动平衡 leader 的时间间隔
    leader.imbalance.check.interval.seconds=300

    # 允许每个 broker leader 不平衡的比例
    leader.imbalance.per.broker.percentage=10

## offset retention

    offsets.retention.check.interval.ms = 600000
    offsets.retention.minutes = 1440

## 时间戳

    log.message.timestamp.type=CreateTime/LogAppendTime

0.10.0.0 版本后，kafka 消息增加了 timestamp 字段，表示消息的时间戳，有 2 种类型：

1. `CreateTime`: producer 端发送消息时给消息设置的 timestamp 字段，理解为消息创建时间。
2. `LogAppendTime`: 使用 broker 接收消息的时间作为消息的 timestamp（会覆盖消息本身携带的 timestamp 字段)，理解为消息写入时间。

时间戳类型为 `CreateTime` 时，允许 create time 与当前时间最大的时间差：

    log.message.timestamp.difference.max.ms=9223372036854775807

## 限流

    leader.replication.throttled.rate

表示 leader 节点对来自副本复制的读流量限制，搭配 topic 参数 `leader.replication.throttled.replicas` 使用。

    follower.replication.throttled.rate

表示 follower 节点复制副本的写流量限制，搭配 topic 参数 `follower.replication.throttled.replicas` 使用。

假设 broker 1 上 leader.replication.throttled.rate 设置为 512KB，topic A 分区 0 分布在 broker 1 和 broker 2 上，broker 1 上为分区 leader，设置 topic A 级别的 leader.replication.throttled.replicas=0:1，则 broker 2 上 topic A 分区 0 从 broker 1 上同步分区 0 数据的速度被限制为 512KB。

多个副本同时进行同步，都会占用 leader 的限流阈值

### 示例

在 broker 1 上限制 topic A 的分区 0, 1, 2 的 leader read 总速率为 1024 B/s

    # broker 1 增加 broker 级别配置
    leader.replication.throttled.rate=1024

    # topic A 增加 topic 级别配置
    leader.replication.throttled.replicas=0:1,1:1,2:1

## log.cleaner

compact 清理策略相关

    log.cleaner.delete.retention.ms

    log.cleaner.enable

## 压缩

broker 端默认使用生成者的压缩策略，当生产者发送的消息 RecordBatch 压缩时，broker 端不需要解压，直接写入

    # 'gzip', 'snappy', 'lz4', 'zstd'
    compression.type=producer

以下 3 种情况，broker 需要对生产者的压缩消息解压并重新压缩:

1. 当 broker 端使用了和生产者不同的压缩算法
2. broker 端消息格式与生产者不一致时
3. broker 目标消息格式是 V0，需要为每条消息重新分配绝对 offset，因此也需要进行解压

当消费组从 broker 读取消息时，broker 会把压缩消息直接发出，消费者读到压缩的消息后，可以根据 RecordBatch attributes 字段得知消息压缩算法，自行解压。

## 事务

__transaction_state 主题配置：

    # __transaction_state 的 min.insync.replicas，默认 2 (优先级高于主题级别配置)
    transaction.state.log.min.isr=2

    # __transaction_state 的分区数，默认 50 (部署后不应该更改)
    transaction.state.log.num.partitions=50

    # __transaction_state 的副本数，默认 3
    transaction.state.log.replication.factor=3

其他配置

    # 事务超时时间，默认 15分钟
    transaction.max.timeout.ms=900000

    # transaction coordinator 多久没有收到 transaction 状态更新后将 transaction id 视为过期
    # 默认 7 天
    transactional.id.expiration.ms=604800000

    # rollback 超时 transaction 的时间间隔，默认 10 秒
    transaction.abort.timed.out.transaction.cleanup.interval.ms=10000

    # remove 过期 transaction 的时间间隔，默认 1 小时
    transaction.remove.expired.transaction.cleanup.interval.ms=3600000

