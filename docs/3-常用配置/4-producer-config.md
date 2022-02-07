# 生产者参数配置

    # 内存缓冲大小，单位 byte
    buffer.memory=33554432

在 Producer 端用来存放尚未发送出去的 Message 的缓冲区大小，默认 32MB。内存缓冲区内的消息以一个个 batch 的形式组织，每个 batch 内包含多条消息，Producer 会把多个 batch 打包成一个 request 发送到 kafka 服务器上。内存缓冲区满了之后可以选择阻塞发送或抛出异常，由 `block.on.buffer.full` 的配置来决定。
    * 如果选择阻塞，在消息持续发送过程中，当缓冲区被填满后，producer立即进入阻塞状态直到空闲内存被释放出来，这段时间不能超过 `max.block.ms` 设置的值，一旦超过，producer则会抛出 `TimeoutException` 异常，因为Producer是线程安全的，若一直报TimeoutException，需要考虑调高buffer.memory 了。

 
    batch.size=16384

Producer会把发往同一个 topic partition 的多个消息进行合并，`batch.size` 指明了合并后 batch 大小的上限。如果这个值设置的太小，可能会导致所有的Request都不进行Batch。

    linger.ms=0

producer 合并的消息的大小未达到 `batch.size`，但如果存在时间达到 `linger.ms`，也会进行发送。

    #  最大请求大小
    max.request.size

决定了每次发送给Kafka服务器请求的最大大小，同时也会限制你一条消息的最大大小也不能超过这个参数设置的值，这个其实可以根据你自己的消息的大小来灵活的调整

    # 发送失败重试次数
    retries
    # 每次重试间隔时间
    retries.backoff.ms

    # 压缩类型
    compression.type=none

默认发送不进行压缩，推荐配置一种适合的压缩算法，可以大幅度的减缓网络压力和Broker的存储压力。

    acks=1

这个配置可以设定发送消息后是否需要Broker端返回确认，设置时需要权衡数据可靠性和吞吐量。

`acks`:
    * 0：表示 producer 请求立即返回，不需要等待 leader 的任何确认
    * -1：表示分区 leader 必须等待消息被成功写入到所有的 ISR 副本中才认为 producer 成功
    * 1：表示 leader 副本必须应答此 producer 请求并写入消息到本地日志，之后 producer 请求被认为成功

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

    enable.idempotence

单个会话，单个 partition 幂等性，重复发送数据时 exactly once

producer id, sequence number

producer id, topic, partition, sequence number

## 事务性

生产者事务 id

    transactional.id

设置事务 id 后，`enable.idempotence` 默认开启

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

## 请求

