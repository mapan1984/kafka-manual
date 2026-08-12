# 诊断工具

## 查看 topic offset

最终的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -1 --topic test

最早的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -2 --topic test

> 指定客户端配置：--command-config conf/kafka.properties

## 读取 __consumer_offsets

=== "< 0.11.0.0"

    ```sh
    $ kafka-console-consumer.sh --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter" --zookeeper ${ZK_CONNECT} --topic __consumer_offsets
    ```

=== ">= 0.11.0.0"

    ```sh
    $ kafka-console-consumer.sh --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --bootstrap-server ${BOOTSTRAP_SERVER} --topic __consumer_offsets
    ```

格式：

    [Group, Topic, Partition]::[OffsetMetadata[Offset, Metadata], CommitTime, ExpirationTime]

分区规则：

    Math.abs(groupID.hashCode()) % numPartitions

## 查看日志/索引文件

查看日志文件

    $ kafka-run-class.sh kafka.tools.DumpLogSegments --files ./00000000000000283198.log --print-data-log

查看索引文件

    $ kafka-run-class.sh kafka.tools.DumpLogSegments --files 0000000000000045.timeindex

## 查看请求使用的 API Version

    $ kafka-broker-api-versions.sh  --bootstrap-server ${BOOTSTRAP_SERVER}

## 查看副本同步 lag

    kafka-replica-verification.sh --broker-list ${BOOTSTRAP_SERVER}

    kafka-replica-verification.sh --broker-list ${BOOTSTRAP_SERVER} --topic-white-list .*

## 获取 broker topic 分区实际目录分布

    kafka-log-dirs.sh --bootstrap-server localhost:9092  --describe [--broker-list "0,1,2"] [--topic-list "t1,t2"]

## 查看特定 consumer-topic 消费信息(1.1.1 后不可用）

``` sh
kafka-consumer-offset-checker.sh  --zookeeper $(hostname):2181 --group flume --topic live-log
```
