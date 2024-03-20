# Topic 参数配置

## 时间戳

    message.timestamp.type=LogAppendTime/CreateTime

定义消息的时间戳类型为 `CreateTime` 或者 `LogAppendTime`

    message.timestamp.difference.max.ms

当消息时间戳类型为 `CreateTime` 时，可以接受的创建时间与当前时间的最大时间差

## 限流

    leader.replication.throttled.replicas

表示需要限流的 leader partition replica 列表。格式为 `partitionId:brokerId,partitionId:brokerId,...`，或者使用 `*` 代表此 topic 的全部副本。搭配 broker 参数 `leader.replication.throttled.rate` 使用

    follower.replication.throttled.replicas

表示需要限流的 follower partition replica 列表。格式为 `partitionId:brokerId,partitionId:brokerId,...`，或者使用 `*` 代表此 topic 的全部副本。搭配 broker 参数 `follower.replication.throttled.rate` 使用
