# KRaft 模式

## 配置

### controller

- [静态]增加 `controller.quorum.voters`配置，broker 使用此地址与 controller 连接，controller 使用此地址与 quorum 集群其他节点交互
- [动态]增加 `controller.quorum.bootstrap.servers`配置，controller 成员不需要在配置中固化，但是需要配合 kafka-storage.sh, kafka-metadata-quorum.sh 初始化和动态管理 controller 成员
- 增加 `controller.listener.names=BROKER_CONTROL`配置，broker 使用此协议与 controller 连接
- [可选] 增加 `sasl.mechanism.controller.protocol`配置，broker 使用此 SASL 机制与 controller 连接（如果协议需要 SASL 认证）

```
process.roles=controller

node.id=1001

# dynamic
# controller.quorum.bootstrap.servers=192.168.0.1:9091,192.168.0.2:9091,192.168.0.3:9091

# static
controller.quorum.voters=1001@192.168.0.1:9091,1002@192.168.0.2:9091,1003@192.168.0.3:9091

listeners=CONTROLLER://:9091
# advertised.listeners=CONTROLLER://localhost:9093

controller.listener.names=CONTROLLER

listener.security.protocol.map=CONTROLLER:PLAINTEXT

log.dirs=/data/kraft-controller-logs
metadata.log.dir=/data/kraft-controller-metadata-logs
```

### broker

```
process.roles=broker

node.id=1

# dynamic
# controller.quorum.bootstrap.servers=192.168.0.1:9091,192.168.0.2:9091,192.168.0.3:9091

# static
controller.quorum.voters=1001@192.168.0.1:9091,1002@192.168.0.2:9091,1003@192.168.0.3:9091

controller.listener.names=CONTROLLER

listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

metadata.log.dir=/data/kraft-controller-metadata-logs
```

## 启动服务

### 生成 cluster.id

生成 `cluster.id`:

```
$ kafka-storage.sh random-uuid
fvgkxVK9TuCMr5wKSs921Q

$ KAFKA_CLUSTER_ID=fvgkxVK9TuCMr5wKSs921Q
```

```
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
```

### 格式化存储目录：

#### 静态

```
$ kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/server.properties
```

#### 动态初始化集群

```
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID \
    --initial-controllers "0@node1:9093,1@node2:9093,2@node3:9093" \
    -c config/server.properties
```

#### 动态单节点引导

首先启动第一个 controller

```
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID \
    --standalone \
    -c config/server.properties
```

然后启动其他 controller，主要需要使用 `--no-initial-controllers` 禁止 controller 生成自己的数据

```
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID \
    --no-initial-controllers \
    -c config/server.properties
```

动态配置下添加 controller

```
kafka-metadata-quorum.sh --bootstrap-controller 192.168.0.1:9091 \
    --command-config config/server.properties \
    add-controller
```

动态配置下移除 controller

```
kafka-metadata-quorum.sh --bootstrap-controller 192.168.0.1:9091 \
    --command-config config/server.properties \
    remove-controller --controller-id <id> --controller-directory-id <directory-id>
```

### 启动服务

```
bin/kafka-server-start.sh config/server.properties
```

## kraft 集群状态

连接 broker

```
$ kafka-metadata-quorum.sh --bootstrap-server ${BOOTSTRAP_SERVER} describe --replication
NodeId    LogEndOffset    Lag    LastFetchTimestamp    LastCaughtUpTimestamp    Status
1         18357           0      1720441330777         1720441330777            Leader
2         18357           0      1720441330387         1720441330387            Observer
3         18357           0      1720441330387         1720441330387            Observer


$ kafka-metadata-quorum.sh --bootstrap-server  ${BOOTSTRAP_SERVER} describe --status
ClusterId:              fvgkxVK9TuCMr5wKSs921Q
LeaderId:               1
LeaderEpoch:            2
HighWatermark:          18743
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [1]
CurrentObservers:       [2,3]
```

连接 controller

```
kafka-metadata-quorum.sh --bootstrap-controller localhost:9091 describe --status

kafka-metadata-quorum.sh --bootstrap-controller localhost:9091 describe --replication

kafka-metadata-quorum.sh --bootstrap-controller localhost:9091 describe --human-readable  --replication
```

查看数据：

```
$ kafka-dump-log.sh --cluster-metadata-decoder --files 00000000000000000000.log

$ kafka-dump-log.sh --cluster-metadata-decoder --files 00000000000000007314-0000000001.checkpoint
```

交互式查看数据

```
$ kafka-metadata-shell.sh  --snapshot 00000000000000000000.log

$ kafka-metadata-shell.sh  --snapshot 00000000000000007314-0000000001.checkpoint
```
