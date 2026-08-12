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

## topic

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

## 生产

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

## 消费

消费消息

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning \
     --consumer-property group.id=test_group

    kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topic __test --from-beginning \
     --group test_group --property print.timestamp=true --property print.key=true

> 指定配置文件 --consumer.config client.properties

### 消费者选项

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

## consumer group

### consumer 列表

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

### consumer 详情

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

### 删除消费组

=== "ZooKeeper"

    ```sh
    $ kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --delete --group console-consumer-38645
    ```

=== "Bootstrap Server"

    ```sh
    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --group console-consumer-97214
    ```

#### 删除主题订阅关系

    $ kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete-offsets --group group1 --topic test:0,1 --topic foo

> 2.4.0 版本后新增 [KIP-496](https://cwiki.apache.org/confluence/display/KAFKA/KIP-496%3A+Administrative+API+to+delete+consumer+offsets)

### 设置 consumer current offset

重置 offset 到最新位置：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --reset-offsets --to-latest --group $group --topic $topic --execute

设置到指定的 offset：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-offset 6250 --topic $topic --execute

根据时间设置，设置到大于等于该时间的第一个 offset：

    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --group $group --reset-offsets --to-datetime 2019-12-12T16:59:59.000 --topic $topic --execute
