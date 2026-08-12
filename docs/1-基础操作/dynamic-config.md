# broker / topic 动态参数

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
