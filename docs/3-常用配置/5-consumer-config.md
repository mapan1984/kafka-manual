# 消费者参数配置

<!--
    num.consumer.fetchers=1

启动Consumer的个数，适当增加可以提高并发度。

    fetch.wait.max.ms=100

在Fetch Request获取的数据至少达到 `fetch.min.bytes` 之前，允许等待的最大时长。
-->

    fetch.min.bytes=1

每次Fetch Request至少要拿到多少字节的数据才可以返回。

    enable.auto.commit

是否启用自动提交。

    auto.commit.interval.ms

自动提交间隔

## consumer 什么时候被认为下线

### 心跳

    session.timeout.ms=10000

会话超时时间，如果发送心跳时间超过这个时间，broker 就会认为消费者掉线

    heartbeat.interval.ms=3000

发送心跳间隔时间，推荐不要高于 `session.timeout.ms` 的 1/3

### poll

    max.poll.interval.ms=300000

`poll()` 调用的最大时间间隔，如果距离上一次 `poll()` 调用的时间超过 `max.poll.interval.ms`，消费者会被认为失败

    max.poll.records=500

单次 `poll()` 调用可以拉取的最多消息

## 事务

设置为读已提交消息，关闭自动提交 offset

    isolation.level=read_committed

    enable.auto.commit=false


`isolation.level` 控制如何读通过事务写的消息，如果设置为 `read_committed`，则只会读取事务已提交的消息；如果设置为 `read_uncommitted`(默认值)，则会读取所有消息。非事务写的消息在任何配置下都会返回。


