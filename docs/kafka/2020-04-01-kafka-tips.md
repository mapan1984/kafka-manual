# kafka 命令行操作

### 预设环境变量

预设置环境变量，方便操作：

``` sh
export KAFKA_HOME=/usr/local/kafka
export PATH="$PATH:${KAFKA_HOME}/bin"
# zk 连接地址
export ZK_CONNECT="$(hostname):2181"
# kafka 连接地址
export BOOTSTRAP_SERVER="$(hostname):9092"

# 如果有 jaas 认证
export KAFKA_OPTS="-Djava.security.auth.login.config=${KAFKA_HOME}/config/kafka_server_jaas.conf"
export JMX_PORT=9997
```

### 基本操作

创建 topic

    $ kafka-topics.sh --create --zookeeper ${ZK_CONNECT} --replication-factor 3 --partitions 3 --topic __test

删除 topic

    $ kafka-topics.sh --zookeeper ${ZK_CONNECT} --delete --topic __test

列出 topic

    $ kafka-topics.sh --list --zookeeper ${ZK_CONNECT}

生产 消息

    $ kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic __test

消费 消息

    $ kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning

描述 topic

    $ kafka-topics.sh --describe --zookeeper ${ZK_CONNECT} --topic test

修改 topic 分区数

    $ kafka-topics.sh  --zookeeper ${ZK_CONNECT} --alter  --partitions 5 --topic bar

### 消费者选项

* 使用 zookeeper 保存消费组数据: `--zookeeper localhost:2181`
* 使用 __consumer_offsets 保存消费组数据: `--bootstrap-server localhost:9092`
* 指定 topic:
    * `--topic foo`
    * `--whitelist ".*"`
* 指定 group 名:
    * `--group group1`
    * `--consumer-property group.id=group1`

ZK Group

    kafka-console-consumer.sh --zookeeper ${ZK_CONNECT} --topic test --from-beginning --group test1
    kafka-console-consumer.sh --zookeeper ${ZK_CONNECT} --topic logs --consumer-property group.id=your_group

KF Group

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic test
    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --whitelist ".*"

property

    kafka-console-consumer.sh --property print.timestamp=true --property print.key=true --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning
    kafka-console-consumer.sh --property print.timestamp=true --property print.key=true --property group.id=__test --bootstrap-server ${BOOTSTRAP_SERVER} --zookeeper ${ZK_CONNECT} --topic logs --from-beginning

从指定 partition, offset 开始消费

    $ kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic logs --offset 3418783 --partition 0

### 列出所有 topic 详情

``` sh
kafka-topics.sh --list --zookeeper ${ZK_CONNECT} > topics.data
while read topic
do
    kafka-topics.sh --describe --zookeeper ${ZK_CONNECT} --topic $topic
done < topics.data
```

### 列出所有 consumer 详情

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

### 重分区/修改副本数

获取当前 broker id 列表：

    $ zookeeper-shell.sh ${ZK_CONNECT} ls /brokers/ids | sed 's/ //g'

创建 `topics.json` 文件，文件内容为需要重分区的 topic，例如：

``` json
{
    "topics": [
        {
            "topic": "statistics"
        }
    ],
    "version": 1
}
```

执行 `kafka-reassign-partitions.sh`，指定 `--generate` 参数和刚才创建的 `topics.json` 文件，通过 `--broker-list` 指定分布的 broker id，生成描述 partition 分布的内容：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT}:2181 --generate --topics-to-move-json-file topics.json --broker-list 1,2,3 | tee plan
    Current partition replica assignment
    {"version":1,"partitions":[{"topic":"statistics","partition":0,"replicas":[3],"log_dirs":["any"]}]}

    Proposed partition reassignment configuration
    {"version":1,"partitions":[{"topic":"statistics","partition":0,"replicas":[1],"log_dirs":["any"]}]}


命令会给出现在的 partition 分布和目的 partition 分布，将生成的内容分别保存到 `current.json`(用于恢复) `reassign.json`(之后的计划)

    $ sed -n '2p' plan > current.json

    $ sed -n '5p' plan > reassign.json

可以调整 `replicas.json` 的内容，`replicas` 字段的含义是该 partition 分布的 broker id：
1. 通过增加/减少 `replicas` 中的 broker id 可以增加/减少副本（`log_dirs` 包含的项要与 `replicas` 包含的项数目一致）
2. 调整 `replicas` 字段的第一个 broker id 可以指定这个 partition 的优先 leader

``` json
{
    "partitions": [
        {
            "log_dirs": [
                "any", "any", "any"
            ],
            "partition": 0,
            "replicas": [
                1, 2, 3
            ],
            "topic": "statistics"
        }
    ],
    "version": 1
}
```

*低版本没有 `log_dirs` 字段，可以忽略*

执行 `kafka-reassign-partitions.sh`，指定 `--execute` 参数和 `reassign.json` 文件，执行 partition 重分布：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --execute --reassignment-json-file reassign.json

执行 `kafka-reassign-partitions.sh`，指定 `--verify` 参数和 `reassign.json` 文件，确认 partition 重分布进度：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --verify --reassignment-json-file reassign.json

如果 topic 数据量和流量过大，重分区会对集群服务造成比较大的影响，此时可以对重分区限制流量，比如限制不超过 50MB/s：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --execute --reassignment-json-file reassign.json --throttle 50000000

参考：
- https://kafka.apache.org/documentation/#rep-throttle

### 读取 __consumer_offsets

0.11.0.0之前版本

    $ kafka-console-consumer.sh --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter" --zookeeper ${ZK_CONNECT} --topic __consumer_offsets

0.11.0.0之后版本(含)

    $ kafka-console-consumer.sh --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --bootstrap-server ${BOOTSTRAP_SERVER} --topic __consumer_offsets

格式：

    [Group, Topic, Partition]::[OffsetMetadata[Offset, Metadata], CommitTime, ExpirationTime]

分区规则：

    Math.abs(groupID.hashCode()) % numPartitions

### 修改 topic 参数

保留大小

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __test --add-config max.message.bytes=4194304

保留时间

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __test --add-config retention.ms=259200000

修改 __consumer_offsets 保留策略

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --describe --entity-type topics --entity-name __consumer_offsets

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __consumer_offsets --delete-config cleanup.policy

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __consumer_offsets --add-config retention.ms=2592000000
    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __consumer_offsets --add-config cleanup.policy=delete

    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __consumer_offsets --delete-config retention.ms
    $ kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --entity-type topics --entity-name __consumer_offsets --add-config cleanup.policy=compact

### 查看日志/索引文件

查看日志文件

    $ kafka-run-class.sh kafka.tools.DumpLogSegments --files ./00000000000000283198.log --print-data-log

查看索引文件

    $ kafka-run-class.sh kafka.tools.DumpLogSegments --files 0000000000000045.timeindex

### 重新平衡 leader

    $ kafka-preferred-replica-election.sh --zookeeper ${ZK_CONNECT}

### 删除消费组

KF 类型

    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --group console-consumer-97214

ZK 类型

    $ kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --delete --group console-consumer-38645

### 查看请求使用的 API Version

    $ kafka-broker-api-versions.sh  --bootstrap-server ${BOOTSTRAP_SERVER}

### 查看副本同步 lag

    kafka-replica-verification.sh --broker-list ${BOOTSTRAP_SERVER}

    kafka-replica-verification.sh --broker-list ${BOOTSTRAP_SERVER} --topic-white-list .*

### 查看 topic offset

最终的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -1 --topic test

最早的 offset

    $ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list ${BOOTSTRAP_SERVER} --time -2 --topic test

### 设置 consumer current offset

    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-datetime 2019-12-12T16:59:59.000 --topic $topic --execute

### SCRAM 用户

    kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --add-config 'SCRAM-SHA-256=[password=admin_pass],SCRAM-SHA-512=[password=admin_pass]' --entity-type users --entity-name admin

    kafka-configs.sh --zookeeper ${ZK_CONNECT} --alter --delete-config 'SCRAM-SHA-512' --entity-type users --entity-name admin
