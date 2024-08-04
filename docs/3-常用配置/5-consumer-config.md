# 消费者参数配置

## consumer 什么时候被认为下线

### 心跳

    session.timeout.ms=10000

会话超时时间，如果发送心跳时间超过这个时间，broker 就会认为消费者掉线

必须在 broker 参数 `group.min.session.timeout.ms` 与 `group.max.session.timeout.ms` 之间

    heartbeat.interval.ms=3000

发送心跳间隔时间，推荐不要高于 `session.timeout.ms` 的 1/3

    # 9 minutes
    connections.max.idle.ms=540000

连接空闲时间超过此配置后关闭

### poll

    max.poll.interval.ms=300000

`poll()` 调用的最大时间间隔，如果距离上一次 `poll()` 调用的时间超过 `max.poll.interval.ms`，消费者会被认为失败

为什么有心跳后还需要设置 `max.poll.interval.ms` 超时机制，因为即使消费者有心跳，也只能证明消费者是存活状态，但是并不能保证消费者正常，比如消费者因为死锁导致永远无法调用 `poll()`

    max.poll.records=500

单次 `poll()` 调用可以拉取的最多消息，增加该值可能会导致 `poll()` 调用间隔增加

此参数并不影响消费者底层 fetch 请求调用的行为，消费者会缓存 fetch 请求拉取的消息，poll 调用从缓存中获取消息

## 批次

    fetch.min.bytes=1

fetch 请求要求服务端返回响应最少应达到的数据量，默认为 1，增加该值可能提高服务端的吞吐量，同时增加延迟

    max.partition.fetch.bytes

fetch 请求要求服务端对每个 partition 返回响应最多达到的数据量（即使超过也会返回，不是硬性的固定上限）

    fetch.max.bytes

fetch 请求要求服务端返回响应最多达到的数据量（即使超过也会返回，不是硬性的固定上限）

    fetch.max.wait.ms=500

fetch 请求要求服务端在返回前最长阻塞时间，如果在这个时间到了仍然没有达到 `fetch.min.bytes` 要求的数据量，仍然返回响应

    request.timeout.ms

客户端请求的超时时间

## 元数据

    # 5 minutes
    metadata.max.age.ms=300000

定时刷新客户端元数据，即使在此期间没有发生 partition leader 切换

## offset commit

    enable.auto.commit

是否启用自动提交

    auto.commit.interval.ms

自动提交间隔

## 事务

设置为读已提交消息，关闭自动提交 offset

    isolation.level=read_uncommitted

    enable.auto.commit=false

`isolation.level` 控制如何读通过事务写的消息，如果设置为 `read_committed`，则只会读取事务已提交的消息与非事务写的消息。如果设置为 `read_uncommitted`(默认值)，则会读取所有消息。

消费者以 offset 顺序读取消息，如果设置了 `read_committed`，`consumer.poll()` 只会返回 last stable offset (LSO) 之前的消息，LSO 是小于第一个 open transaction 的 offset。

消费者设置 `read_committed` 之后，`seekToEnd` 方法也会返回 LSO。

