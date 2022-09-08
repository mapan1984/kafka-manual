# 分区重分配/修改副本数

> Kafka 在逐步去除 zookeeper 依赖，所以不同的版本命令行工具参数存在差异，
> 以下大部分命令给了依赖 zookeeper 和不依赖 zookeeper 的 2 种示例，可以根据自己的 kafka 版本进行选择。

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

## 获取当前集群 broker id 列表

获取当前 broker id 列表：

    $ zookeeper-shell.sh ${ZK_CONNECT} ls /brokers/ids | sed 's/ //g'

## 生成分配方案

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

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --generate --topics-to-move-json-file topics.json --broker-list 1,2,3 | tee plan
    Current partition replica assignment
    {"version":1,"partitions":[{"topic":"statistics","partition":0,"replicas":[3],"log_dirs":["any"]}]}

    Proposed partition reassignment configuration
    {"version":1,"partitions":[{"topic":"statistics","partition":0,"replicas":[1],"log_dirs":["any"]}]}

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --topics-to-move-json-file topics.json --broker-list 1,2,3 --generate | tee plan

命令会给出现在的 partition 分布和目的 partition 分布，将生成的内容分别保存到 `current.json`(用于恢复) `reassign.json`(之后的计划)

    $ sed -n '2p' plan > current.json

    $ sed -n '5p' plan > reassign.json

可以调整 `replicas.json` 的内容，`replicas` 字段的含义是该 partition 分布的 broker id：

1. 通过增加/减少 `replicas` 中的 broker id 可以增加/减少副本（`log_dirs` 包含的项要与 `replicas` 包含的项数目一致）
2. 调整 `replicas` 字段的第一个 broker id 可以指定这个 partition 的优先 leader
3. 通过指定 `log_dirs` 中 `any` 为实际的目录，从而指定分区数据位置

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

*版本低于 1.1.0，分配方案没有 `log_dirs` 字段，可以忽略*

## 执行重新分配

执行 `kafka-reassign-partitions.sh`，指定 `--execute` 参数和 `reassign.json` 文件，执行 partition 重分布：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --execute --reassignment-json-file reassign.json

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --reassignment-json-file reassign.json --execute

执行 `kafka-reassign-partitions.sh`，指定 `--verify` 参数和 `reassign.json` 文件，确认 partition 重分布进度：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --verify --reassignment-json-file reassign.json

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --reassignment-json-file reassign.json --verify

执行 `kafka-reassign-partitions.sh`，指定 `--list` 参数，查看当前正在进行重分配的分区：

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list
    Current partition reassignments:
    test-0: replicas: 2,1,3. adding: 1. removing: 3.
    test-1: replicas: 3,2,1. adding: 2. removing: 1.
    test-2: replicas: 1,3,2. adding: 3. removing: 2.

取消重分配：

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --reassignment-json-file reassign.json --cancel

## 限流

如果 topic 数据量和流量过大，重分区会对集群服务造成比较大的影响，此时可以使用 `--throttle` 参数对重分区限制流量，单位 Byte/s。

比如限制不超过 50MB/s：

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --execute --reassignment-json-file reassign.json --throttle 50000000

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --reassignment-json-file reassign.json --execute --throttle 50000000

重新限制流量为 700MB/s

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --additional --execute --reassignment-json-file reassign.json --throttle 700000000

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --reassignment-json-file reassign.json --execute --additional --throttle 700000000

当分区分配完成后，重新执行 verfiy 会取消限流设置

    $ kafka-reassign-partitions.sh --zookeeper ${ZK_CONNECT} --verify --reassignment-json-file reassign.json

    $ kafka-reassign-partitions.sh --bootstrap-server ${BOOTSTRAP_SERVER} --verify --reassignment-json-file reassign.json

限流实际上设置了 broker 级别的 `leader.replication.throttled.rate`, `follower.replication.throttled.rate` 参数和 topic 级别的 `leader.replication.throttled.replicas`, `follower.replication.throttled.replicas` 参数，限制了副本同步的流量。

### 参考

- https://kafka.apache.org/documentation/#rep-throttle

