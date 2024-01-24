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
export KAFKA_OPTS="-Djava.security.auth.login.config=${KAFKA_HOME}/config/kafka_server_jaas.conf"

# 如果 broker 通过在 kafka-run-class.sh 文件内设置 JMX_PORT，则这里需要设置成不同的 port
# (一般 broker 开启 JMX_PORT 最好在 kafka-server-start.sh 文件内设置，kafka-run-class.sh 文件内的修改会影响到所有命令脚本)
# export JMX_PORT=9997
```

## 基本操作

### topic

创建 topic

    kafka-topics.sh --zookeeper ${ZK_CONNECT} --create --replication-factor 3 --partitions 3 --topic __test

    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --create --replication-factor 3 --partitions 3 --topic __test

删除 topic

    kafka-topics.sh --zookeeper ${ZK_CONNECT} --delete --topic __test

    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --topic __test

topic 列表

    kafka-topics.sh --zookeeper ${ZK_CONNECT} --list

    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list

topic 详情

    kafka-topics.sh --zookeeper ${ZK_CONNECT} --describe --topic test

    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --topic __test

修改 topic 分区数

    kafka-topics.sh --zookeeper ${ZK_CONNECT} --alter --topic __test --partitions 5

    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --alter --topic __test --partitions 5

### 生产/消费

生产 消息

    kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic __test

    kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test

消费 消息

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning

    kafka-console-consumer.sh --property print.timestamp=true --property print.key=true \
        --bootstrap-server ${BOOTSTRAP_SERVER} --group test_group --topic test --from-beginning

### consumer

consumer 列表

    # 记录在 zookeeper 中的消费组（2.x.x 版本以上废弃）
    kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --list

    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 <= 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list --new-consumer

    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 > 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list

consumer 详情

    # 记录在 zookeeper 中的消费组（2.x.x 版本以上废弃）
    kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --describe --group $group

    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 <= 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --new-consumer --describe --group $group

    # 记录在 __consumer_offsets 中的消费组，Kafka 版本 > 0.9.x.x
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group $group

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group my-group --members

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group my-group --members --verbose

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group my-group --state

## 消费者选项

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

    kafka-console-consumer.sh --zookeeper ${ZK_CONNECT} --topic test --from-beginning --group group1

    kafka-console-consumer.sh --zookeeper ${ZK_CONNECT} --topic test --consumer-property group.id=group1

KF Group

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic test

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --whitelist ".*"

property

    kafka-console-consumer.sh --property print.timestamp=true --property print.key=true --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning

从指定 partition, offset 开始消费

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic logs --partition 0 --offset 3418783

从指定 partition, offset 开始消费指定数量消息：

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic logs --partition 7 --offset 1340190464 --max-messages 10

## broker 参数(动态)

查看参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --describe

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 0 --describe

设置副本同步的限流参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-name 0 \
        --alter --add-config "leader.replication.throttled.rate=1024,follower.replication.throttled.rate=1024"

删除副本同步限流参数：

    kafka-configs.sh --zookeeper ${ZK_CONNECT} -entity-type brokers  --entity-name 0 \
        --alter --delete-config 'leader.replication.throttled.rate,follower.replication.throttled.rate'

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} -entity-type brokers --entity-name 0 \
        --alter --delete-config 'leader.replication.throttled.rate,follower.replication.throttled.rate'

设置集群级别的参数

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default \
        --alter --add-config 'log.cleaner.threads=2'

查看集群级别的参数

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type brokers --entity-default --describe

## topic 参数

查看参数：

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --describe

    kafka-configs.sh --bootstrap-server ${BOOTSTRAP_SERVER} --entity-type topics --entity-name __test --describe

修改消息大小限制：

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __test \
        --alter --add-config max.message.bytes=4194304

修改消息保留时长：

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name __test \
        --alter --add-config retention.ms=259200000

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

设置副本同步流量限制：

    kafka-configs.sh --zookeeper ${ZK_CONNECT} --entity-type topics --entity-name test-throttled \
        --alter --add-config "leader.replication.throttled.replicas=*,follower.replication.throttled.replicas=*"

删除副本同步流量限制：

    kafka-configs.sh --zookeeper ${ZK_CONNECT} -entity-type topics --entity-name test-throttled \
        --alter --delete-config 'leader.replication.throttled.replicas,follower.replication.throttled.replicas'

## 选举 leader

触发集群内所有 topic partition 的最优 leader 选举:

    $ kafka-preferred-replica-election.sh --zookeeper ${ZK_CONNECT}

触发 `partitions.json` 文件指定的 topic partition 的最优 leader 选举:

    kafka-preferred-replica-election.sh --zookeeper ${ZK_CONNECT} --path-to-json-file partitions.json

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

从 2.4.0 版本开始，推荐使用 `kafka-leader-election.sh` 触发选举

可以通过 `--topic`, `--partition` 参数指定 topic, partition，通过 `--election-type` 指定选举类型为 preferred/unclean

    $ kafka-leader-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic <topic> --partition <partition> --election-type preferred

也可以通过 `--path-to-json-file` 指定文件包含的 topic partition 的最优 leader 选举

    $ kafka-leader-election.sh --bootstrap-server ${BOOTSTRAP_SERVER} --path-to-json-file partitions.json --election-type preferred

## 删除消费组

KF 类型

    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --group console-consumer-97214

ZK 类型

    $ kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --delete --group console-consumer-38645

## 查看 topic offset

最终的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -1 --topic test

最早的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -2 --topic test

## 设置 consumer current offset

重置 offset 到最新位置：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --reset-offsets --to-latest --group $group --topic $topic --execute

设置到指定的 offset：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-offset 6250 --topic $topic --execute

根据时间设置，设置到大于等于该时间的第一个 offset：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-datetime 2019-12-12T16:59:59.000 --topic $topic --execute


## 读取 __consumer_offsets

0.11.0.0 之前版本

    $ kafka-console-consumer.sh --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter" --zookeeper ${ZK_CONNECT} --topic __consumer_offsets

0.11.0.0 之后版本(含)

    $ kafka-console-consumer.sh --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --bootstrap-server ${BOOTSTRAP_SERVER} --topic __consumer_offsets

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
