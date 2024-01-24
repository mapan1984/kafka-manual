# 生产者参数配置

    # 内存缓冲大小，单位 byte
    buffer.memory=33554432

在 Producer 端用来存放尚未发送出去的 Message 的缓冲区大小，默认 32MB。内存缓冲区内的消息以一个个 batch 的形式组织，每个 batch 内包含多条消息，Producer 会把多个 batch 打包成一个 request 发送到 kafka 服务器上。

    # `0.11.0.0` 版本之后废弃
    block.on.buffer.full=false

内存缓冲区满了之后可以选择阻塞发送或抛出异常，由 `block.on.buffer.full` 的配置来决定（`0.11.0.0` 版本之后废弃）。

    max.block.ms=60000

设置 `Producer` 的 `send()`, `partitionsFor()` 等方法的最多阻塞时长。

* 当缓冲区 `buffer.memory` 写满后，Producer `send()` 方法最多阻塞时间不超过 `max.block.ms` 设置的值，超过后 producer 会抛出 `TimeoutException` 异常。
* 元数据拉取超时，也会导致 `send()` 方法超时。


    batch.size=16384

Producer 会把发往同一个 topic partition 的多个消息进行合并，`batch.size` 指明了合并后 batch 大小的上限。如果这个值设置的太小，可能会导致所有的 Request 都不进行 Batch。`batch.size` 增加会增大吞吐量，但是同时也会增加延迟。

    linger.ms=0

producer 合并的消息的大小未达到 `batch.size`，但如果存在时间达到 `linger.ms`，也会进行发送。增加此值可能会增加吞吐量，但同时也会增加延迟。

    # 最大请求大小
    max.request.size

决定了每次发送给 Kafka 服务器请求的最大大小，同时也限制了单条消息的最大大小

    # 请求-响应超时时间，应该大于服务端的 replica.lag.time.max.ms
    request.timeout.ms=30000

    # 发送失败重试次数
    retries
    # 每次重试间隔时间
    retries.backoff.ms

    # 压缩类型
    compression.type=none

默认发送不进行压缩，推荐配置一种适合的压缩算法，可以大幅度的减缓网络压力和Broker的存储压力。

    acks=1

这个配置可以设定发送消息后是否需要 Broker 端返回确认:

* 0：表示 producer 请求发出后立即返回，不需要等待 leader 的任何确认
* 1：表示 producer 发出请求后，leader 需要将 producer 请求消息写入后向 producer 返回成功响应，之后 producer 请求才确认成功并返回
* -1/all：表示 producer 发出请求后，leader 需要将 producer 请求消息写入并等待所有 ISR 副本同步后，向 producer 返回成功响应，之后 producer 请求才确认成功并返回。当 ISR 副本少于 `min.insync.replicas` 设置的值时，producer 在此情况会报 `NotEnoughReplicas` 异常。

从上到下的设置，可靠性依次增强，吞吐量依次降低，延迟依次增加。

## 生产者不丢失数据保证

    block.on.buffer.full = true

生产者消息在实际发送之前是保留在 buffer 中，buffer 满之后生产等待，而不是抛出异常

    acks=all

所有 follower 都响应后才认为消息提交成功（需要注意 broker 的 `min.insync.replicas` 参数）

    retries=Integer.MAX_VALUE

发送失败后持续重试（单独设置这个可能会造成消息重复发送）

    max.in.flight.requests.per.connection=1

单个线程在单个连接上能够发送的未响应请求个数，这个参数设置为 1 可以避免消息乱序，同时可以保证在 retry 是不会重复发送消息，但是会降低 producer io 线程的吞吐量

    unclean.leader.election.enable=false

关闭 unclean leader 选举，即不允许非 ISR 中的副本被选举为 leader

## 幂等性

开启幂等写配置：

    enable.idempotence

示例：

``` java
Properties props = new Properties();
props.put("enable.idempotence", "true");
props.put("acks", "all");  // 当 enable.idempotence 为 true，这里默认为 all
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer producer = new KafkaProducer(props);
```

kafka 可以通过设置 `ack`, `retries` 等参数保证消息不丢失，但是无法消息不重复（即 at least once）。

幂等性解决消息重复的问题，即多次发送同一条消息到 server 端，server 只会记录一次，之后重复发送的消息会被丢弃。

为了判断消息是否重复，Kafka 使用 `producer_id` + `sequence_number` 标记每条消息，由 topic partiton 所在 leader 进行判断并去重。

每个 producer 在初始化时，会向 server 端申请一个唯一的 `producer_id`。之后发送的每条消息，都会关联一个从 0 开始递增的 `sequence_number`，每个 topic partition 都会维护一个单独的 `sequence_number`。

因为每次 producer 初始化都会申请新的 `producer_id`，且 `sequence_number` 是分区维度的，所以只能保证单个会话，单个 partition 的幂等性，重复发送数据时 exactly once

## 事务性

生产者事务 id

    transactional.id

设置事务 id 后，`enable.idempotence` 默认开启

kafka 事务特性主要用于 2 种场景：

1. 将多条消息的发送动作封装在一个事务中，形成原子操作，多条消息要么都发送成功，要么都发送失败。
2. consume-transform-produce loop，将 消费消息-处理消息-发送消息 封装在一个事务中，形成原子操作。常见于流式处理应用，从一个上游接收消息，经过处理后发送给下游。

kafka 事务的实现原理是把全部消息都追加到分区日志中，并将未完成事务的消息标记为未提交。一旦事务提交，这些标记就会被改为已提交。

### 场景 1

示例代码：

``` java
Properties props = new Properties();
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("client.id", "ProducerTranscationnalExample");
props.put("bootstrap.servers", "localhost:9092");
props.put("transactional.id", "test-transactional");
props.put("acks", "all");
KafkaProducer producer = new KafkaProducer(props);
producer.initTransactions();

try {
    String msg = "matt test";
    producer.beginTransaction();
    producer.send(new ProducerRecord(topic, "0", msg.toString()));
    producer.send(new ProducerRecord(topic, "1", msg.toString()));
    producer.send(new ProducerRecord(topic, "2", msg.toString()));
    producer.commitTransaction();
} catch (ProducerFencedException e1) {
    e1.printStackTrace();
    producer.close();
} catch (KafkaException e2) {
    e2.printStackTrace();
    producer.abortTransaction();
}
producer.close();
```

* 原子性：事务保证多个写操作要么全部成功，要么全部失败
* 僵死进程：开启事务的 producer 会向 Transaction 申请一个 producer id，transaction id 与 producer id 一一对应，每个 producer id 对应一个 epoch，当新的 producer 使用相同的 transaction id 开启事务后，会获得相同的 producer id 和更高的 epoch，此时服务端可以根据 epoch 区分新旧 producer，旧的 producer 将不能写入消息。

### 场景 2

``` java
Properties producerProps = new Properties();
producerProps.put("bootstrap.servers", "localhost:9092");
producerProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
producerProps.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

producerProps.put("transactional.id", "test-transactional");
producerProps.put("acks", "all");
KafkaProducer producer = new KafkaProducer(producerProps);

Properties consumerProps = new Properties();
consumerProps.put("bootstrap.servers", "localhost:9092");
consumerProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
consumerProps.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

consumerProps.put("group.id", groupId);
consumerProps.put("enable.auto.commit", "false");
consumerProps.put("isolation.level", "read_committed");
KafkaConsumer consumer = new KafkaConsumer(consumerProps);

consumer.subscribe(Collections.singleton("source_topic"));


producer.initTransactions();

try {
    records = consumer.poll();

    producer.beginTransaction();

    records.forEach(record -> producer.send("target_topic", record));

    producer.sendOffsetsToTransaction();

    producer.commitTransaction();
} catch (ProducerFencedException e1) {
    e1.printStackTrace();
    producer.close();
} catch (KafkaException e2) {
    e2.printStackTrace();
    producer.abortTransaction();
}
producer.close();
```

- https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging
