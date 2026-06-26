# kafka 基础操作

## 预设环境变量

预设置环境变量，方便操作：

``` sh
# 将 kafka 命令脚本路径加入到 PATH
export KAFKA_HOME=/usr/local/kafka
export PATH="$PATH:${KAFKA_HOME}/bin"

# zk 连接地址
export ZK_CONNECT="$(hostname):2181"

# kafka 连接地址
export BOOTSTRAP_SERVER="$(hostname):9092"

# 如果有 jaas 认证
export KAFKA_OPTS="-Djava.security.auth.login.config=${KAFKA_HOME}/config/kafka_client_jaas.conf"

# 如果 broker 通过在 kafka-run-class.sh 文件内设置 JMX_PORT，则这里需要设置成不同的 port
# (一般 broker 开启 JMX_PORT 最好在 kafka-server-start.sh 文件内设置，kafka-run-class.sh 文件内的修改会影响到所有命令脚本)
# export JMX_PORT=9997
```

## 基本操作

### topic

创建 topic

=== "ZooKeeper"

    ```sh
    kafka-topics.sh --zookeeper ${ZK_CONNECT} --create --replication-factor 3 --partitions 3 --topic __test
    ```

=== "Bootstrap Server"

    ```sh
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --create --replication-factor 3 --partitions 3 --topic __test
    ```

删除 topic

=== "ZooKeeper"

    ```sh
    kafka-topics.sh --zookeeper ${ZK_CONNECT} --delete --topic __test
    ```

=== "Bootstrap Server"

    ```sh
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --topic __test
    ```

topic 列表

=== "ZooKeeper"

    ```sh
    kafka-topics.sh --zookeeper ${ZK_CONNECT} --list
    ```

=== "Bootstrap Server"

    ```sh
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list
    ```

topic 详情

=== "ZooKeeper"

    ```sh
    kafka-topics.sh --zookeeper ${ZK_CONNECT} --describe --topic test
    ```

=== "Bootstrap Server"

    ```sh
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --topic __test
    ```

修改 topic 分区数

=== "ZooKeeper"

    ```sh
    kafka-topics.sh --zookeeper ${ZK_CONNECT} --alter --topic __test --partitions 5
    ```

=== "Bootstrap Server"

    ```sh
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --alter --topic __test --partitions 5
    ```

> 指定配置文件 --command-config client.properties

### consumer

consumer 列表

=== "ZooKeeper (已废弃)"

    ```sh
    # 记录在 zookeeper 中的消费组（2.x.x 版本以上废弃）
    kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --list
    ```

=== "Bootstrap Server (<= 0.9)"

    ```sh
    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 <= 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list --new-consumer
    ```

=== "Bootstrap Server (> 0.9)"

    ```sh
    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 > 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list
    ```

consumer 详情

=== "ZooKeeper (已废弃)"

    ```sh
    # 记录在 zookeeper 中的消费组（2.x.x 版本以上废弃）
    kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --describe --group $group
    ```

=== "Bootstrap Server (<= 0.9)"

    ```sh
    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 <= 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --new-consumer --describe --group $group
    ```

=== "Bootstrap Server (> 0.9)"

    ```sh
    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 > 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group $group

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group my-group --members

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group my-group --members --verbose

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group my-group --state
    ```

> 指定配置文件 --command-config client.properties


### 生产

生产消息

=== "--broker-list"

    ```sh
    kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic __test
    ```

=== "--bootstrap-server"

    ```sh
    kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test
    ```

> 指定配置文件 --producer.config client.properties

### 消费

消费消息

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning \
     --consumer-property group.id=test_group

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning \
     --group test_group --property print.timestamp=true --property print.key=true

> 指定配置文件 --consumer.config client.properties

#### 消费者选项

* 使用 zookeeper 保存消费组数据（2.x.x 版本以上废弃）: `--zookeeper localhost:2181`
* 使用 __consumer_offsets 保存消费组数据: `--bootstrap-server localhost:9092`
* 指定 group 名:
    * `--group group1`
    * `--consumer-property group.id=group1`
* 指定 topic:
    * `--topic foo`
    * `--whitelist ".*"`
* 指定 partition:
    * `--partition 0`
* 指定 offset:
    * `--from-beginning`
    * `--offset 3418783`
* 消费多少条消息:
    * `--max-messages 10`

ZK Group（2.x.x 版本以上废弃）

    kafka-console-consumer.sh --zookeeper ${ZK_CONNECT} --topic test --consumer-property group.id=group1

    kafka-console-consumer.sh --zookeeper ${ZK_CONNECT} --topic test --from-beginning --group group1

KF Group

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic test

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --whitelist ".*"

property

    kafka-console-consumer.sh --property print.timestamp=true --property print.key=true --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning

从指定 partition, offset 开始消费

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic logs --partition 0 --offset 3418783

从指定 partition, offset 开始消费指定数量消息：

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic logs --partition 7 --offset 1340190464 --max-messages 10


## 性能测试

写入压测

    kafka-producer-perf-test.sh \
     --print-metrics \
     --topic __test \
     --num-records 5000 \
     --throughput -1 \
     --record-size 1024 \
     --producer-props bootstrap.servers=${BOOTSTRAP_SERVER} buffer.memory=67108864 batch.size=10240 linger.ms=10 acks=1

通过文件指定消息 payload 内容

    kafka-producer-perf-test.sh \
     --print-metrics \
     --topic __test \
     --num-records 5000 \
     --throughput -1 \
     --payload-file payload.json \
     --producer-props bootstrap.servers=${BOOTSTRAP_SERVER} buffer.memory=67108864 batch.size=10240 linger.ms=10 acks=1

> 指定客户端参数：--producer.config client.properties

消费压测

    kafka-consumer-perf-test.sh  \
     --broker-list ${BOOTSTRAP_SERVER}  \
     --show-detailed-stats \
     --topic __test \
     --messages 5000 \
     --group test_group

    kafka-console-consumer.sh \
     --bootstrap-server ${BOOTSTRAP_SERVER} \
     --topic __test \
     --consumer-property group.id=test_group \
     --from-beginning

    kafka-consumer-perf-test.sh \
     --bootstrap-server ${BOOTSTRAP_SERVER}  \
     --topic __test \
     --print-metrics \
     --show-detailed-stats \
     --timeout 180000 \
     --fetch-size 10485760 \
     --socket-buffer-size 20971520 \
     --messages 5000 \
     --group test_group

> 指定客户端参数：--consumer.config client.properties

## broker 参数(动态)

查看参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --describe

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default --describe

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 0 --describe

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 1 --describe

设置参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default  \
     --alter --add-config 'log.cleaner.threads=3'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 1  \
     --alter --add-config 'log.cleaner.threads=3'

删除参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER}  -entity-type brokers  --entity-default  \
     --alter --delete-config 'log.cleaner.threads'

### 磁盘阈值

设置参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'disk.used.threshold.enable=true, disk.used.threshold.percent=85'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'disk.usage.check.interval.ms=20'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'log.min.retention.ms=3600000, log.min.retention.bytes=1073741824'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 2  \
     --alter --add-config 'disk.used.threshold.enable=true, disk.used.threshold.percent=50'

删除参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER}  -entity-type brokers  --entity-default  \
     --alter --delete-config 'disk.used.threshold.enable,disk.used.threshold.percent'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER}  -entity-type brokers  --entity-name 1 \
     --alter --delete-config 'disk.used.threshold.enable,disk.used.threshold.percent'

### 磁盘硬限

设置参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'disk.max.used.percent=10'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'disk.min.free.bytes=0'

删除参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER}  -entity-type brokers  --entity-default  \
     --alter --delete-config 'disk.max.used.percent, disk.min.free.bytes'

### 远程流量

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'remote.log.manager.copy.max.bytes.per.second=36700160'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'remote.log.manager.fetch.max.bytes.per.second=209715200'

### 线程数

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'num.replica.fetchers=2'

### 事务

设置参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --add-config 'transactions.enable=false'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 1 \
     --alter --add-config 'transactions.enable=false'

删除参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
     --alter --delete-config 'transactions.enable'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 1 \
     --alter --delete-config 'transactions.enable'

### 限流

设置参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 0 \
     --alter --add-config "leader.replication.throttled.rate=1024, follower.replication.throttled.rate=1024"

删除参数：

=== "ZooKeeper"

    ```sh
    kafka-configs.sh --zookeeper ${ZK_CONNECT} -entity-type brokers  --entity-name 0 \
     --alter --delete-config 'leader.replication.throttled.rate, follower.replication.throttled.rate'
    ```

=== "Bootstrap Server"

    ```sh
    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} -entity-type brokers --entity-name 0 \
     --alter --delete-config 'leader.replication.throttled.rate, follower.replication.throttled.rate'
    ```

## topic 参数

查看参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --describe

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name test --describe

> 指定客户端参数：--command-config client.properties

### 消息大小限制

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __test \
        --alter --add-config max.message.bytes=4194304

### 保留时长

设置参数：

=== "ZooKeeper"

    ```sh
    kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name test \
     --alter --add-config retention.ms=259200000
    ```

=== "Bootstrap Server"

    ```sh
    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name test \
     --alter --add-config 'retention.ms=7200000'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name test \
     --alter --add-config 'local.retention.bytes=1073741820'
    ```

删除参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name test \
     --alter --delete-config 'retention.ms'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name test \
     --alter --delete-config 'local.retention.bytes'

修改 __consumer_offsets 保留策略：

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __consumer_offsets --describe

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __consumer_offsets \
        --alter --delete-config cleanup.policy

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __consumer_offsets \
        --alter --add-config retention.ms=2592000000
    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __consumer_offsets \
        --alter --add-config cleanup.policy=delete

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __consumer_offsets \
        --alter --delete-config retention.ms
    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __consumer_offsets \
        --alter --add-config cleanup.policy=compact

### 保留大小

    kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name test \
      --alter --add-config retention.bytes=3298534883328

### 限流

设置副本同步流量限制：

    kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name test-throttled \
        --alter --add-config "leader.replication.throttled.replicas=*,follower.replication.throttled.replicas=*"

删除副本同步流量限制：

    kafka-configs.sh --zookeeper ${ZK_CONNECT} -entity-type topics --entity-name test-throttled \
        --alter --delete-config 'leader.replication.throttled.replicas,follower.replication.throttled.replicas'

### 磁盘阈值

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name test \
     --alter --add-config 'min.retention.bytes=2147483648, min.retention.ms=3600000'


## 选举 leader

=== "< 2.4 (kafka-preferred-replica-election)"

    触发集群内所有 topic partition 的最优 leader 选举:

    ```sh
    $ kafka-preferred-replica-election.sh --zookeeper ${ZK_CONNECT}
    ```

    触发 `partitions.json` 文件指定的 topic partition 的最优 leader 选举:

    ```sh
    kafka-preferred-replica-election.sh --zookeeper ${ZK_CONNECT} --path-to-json-file partitions.json
    ```

    使用 BOOTSTRAP_SERVER 地址连接

    ``` sh
    kafka-preferred-replica-election.sh --bootstrap-server ${BOOTSTRAP_SERVER}
    ```

    ``` sh
    kafka-preferred-replica-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --path-to-json-file partitions.json
    ```

    > 指定客户端配置：--admin.config conf/kafka.properties


=== ">= 2.4 (kafka-leader-election)"

    从 2.4.0 版本开始，推荐使用 `kafka-leader-election.sh` 触发选举

    可以通过 `--topic`, `--partition` 参数指定 topic, partition，通过 `--election-type` 指定选举类型为 preferred/unclean

    ```sh
    $ kafka-leader-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic <topic> --partition <partition> --election-type preferred
    ```

    也可以通过 `--path-to-json-file` 指定文件包含的 topic partition 的最优 leader 选举

    ```sh
    $ kafka-leader-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --path-to-json-file partitions.json --election-type preferred
    ```

    对全部主题分区触发选举

    ``` sh
    kafka-leader-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --election-type UNCLEAN --all-topic-partitions

    kafka-leader-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --election-type PREFERRED --all-topic-partitions
    ```

    > 指定配置文件 --admin.config conf/kafka.properties

`partitions.json` 文件内容如下：

``` json
{
    "partitions": [
        {
            "partition": 1,
            "topic": "__test"
        }
    ]
}
```

## 删除消费组

=== "ZooKeeper"

    ```sh
    $ kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --delete --group console-consumer-38645
    ```

=== "Bootstrap Server"

    ```sh
    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --group console-consumer-97214
    ```

### 删除主题订阅关系

    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete-offsets --group group1 --topic test:0,1 --topic foo

> 2.4.0 版本后新增 [KIP-496](https://cwiki.apache.org/confluence/display/KAFKA/KIP-496%3A+Administrative+API+to+delete+consumer+offsets)

## 查看 topic offset

最终的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -1 --topic test

最早的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -2 --topic test

> 指定客户端配置：--command-config conf/kafka.properties

## 设置 consumer current offset

重置 offset 到最新位置：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --reset-offsets --to-latest --group $group --topic $topic --execute

设置到指定的 offset：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-offset 6250 --topic $topic --execute

根据时间设置，设置到大于等于该时间的第一个 offset：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-datetime 2019-12-12T16:59:59.000 --topic $topic --execute


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

<!--
## 列出所有 topic 详情

``` sh
kafka-topics.sh --list --zookeeper ${ZK_CONNECT} > topics.data
while read topic
do
    kafka-topics.sh --describe --zookeeper ${ZK_CONNECT} --topic $topic
done < topics.data
```
-->

<!--
## 列出所有 consumer 详情

ZK

``` sh
kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --list > zk.data
while read group
do
    echo ==================== zk group name: $group ===============================
    kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --describe --group $group
    echo
    echo
done < zk.data
```

KF `>` 0.9.x.x

``` sh
kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list > kf.data
while read group
do
    echo ==================== kf group name: $group ===============================
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group $group
    echo
    echo
done < kf.data
```

KF `<=` 0.9.x.x

``` sh
kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list --new-consumer > kf.data
while read group
do
    echo ==================== kf group name: $group ===============================
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --new-consumer --describe --group $group
    echo
    echo
done < kf.data
```
-->

<!--
## 查看特定 consumer-topic 消费信息(1.1.1 不可用）

kafka-consumer-offset-checker.sh  --zookeeper $(hostname):2181 --group flume --topic live-log)
-->


<!--
## 统计 kafka consumer lag 的总和

kafka-consumer-groups.sh --zookeeper $(hostname):2181 --describe --group logstash \
      | tail -n +2 \
      | head -n 32 \
      | awk -F ',' 'BEGIN {sum=0} {sum+=$6} END {print sum}'
-->


<!--
## 批量删除限流配置

``` bash
#!/bin/bash

while true; do
  echo "Press <CTRL+C> to exit."

  # 循环遍历 broker ID 1 到 5
  for broker_id in {1..5}; do
    kafka-configs.sh --bootstrap-server "${BOOTSTRAP_SERVER}" \
      --entity-type brokers \
      --entity-name "${broker_id}" \
      --alter \
      --delete-config 'leader.replication.throttled.rate,follower.replication.throttled.rate' \
      --command-config /usr/local/ams-worker/conf/kafka.properties
  done

  sleep 60
done
```
-->
