# 手动迁移 kafka topic 分区数据

## 目的

目前遇到 2 类使用场景：

1. Kafka 单个节点上挂载多块磁盘，磁盘使用率不均匀，有的磁盘已经 100% 了，有的磁盘还有大量空余，这时可以从使用率高的磁盘移动部分分区到使用率低的磁盘上。
2. kafka 单个节点磁盘故障，数据丢失，但是 topic 分区 leader 并未切换，leader 为 `-1`，如果分区副本数据还在，此时可以将分区副本数据手动移动到 leader 节点上，恢复数据。

> 注意：这里并不会改变分区在 broker 上的分布情况，移动后分区还在原来的 broker 上。操作只是在同一个 broker 不同数据目录下移动分区，或者在分区 leader, follower 所在 broker 之间复制分区数据。如果想改变分区在 broker 上的分布情况，将分区移动到原来没有分布的 broker 上，请参考「基础操作」里的「分区重分配」说明。

## 原理

kafka 数据目录由配置项 `log.dirs`(或 `log.dir`) 指定，数据目录结构如下：

```
- <log.dirs>
    - <topic.name>-<partition.id>
        - <segment.offset>.log
        - <segment.offset>.index
        - <segment.offset>.timeindex
        - ...
        - leader-epoch-checkpoint
    ...
    - cleaner-offset-checkpoint
    - log-start-offset-checkpoint
    - meta.properties
    - recovery-point-offset-checkpoint
    - replication-offset-checkpoint
- ...
```

`log.dir` 目录下，每个 topic 分区以 `<topic.name>-<partition.id>` 为目录存储自身数据，下面有多个 segment 文件与其索引文件:

1. segment 文件(日志分段文件)：文件名为 `<segment.offset>.log`，实际的数据文件，每个 segment 文件都对应 2 个索引文件：
    1. 偏移量索引文件：文件名为 `<segment.offset>.index`，建立消息偏移量(offset)到物理地址之间的映射关系
    2. 时间戳索引文件：文件名为 `<segment.offset>.timeindex`，根据指定的时间戳(timestamp)来查找对应的偏移信息

`log.dir` 目录下除了以 `<topic.name>-<partition.id>` 为名的数据目录，还有 4 个记录 topic partition offset 的文件:

1. `cleaner-offset-checkpoint`
2. `log-start-offset-checkpoint`
3. `recovery-point-offset-checkpoint`
4. `replication-offset-checkpoint`

这些文件内容格式为：

```
0
<record_partition_num>
<topic> <partition.id> <offset>
<topic> <partition.id> <offset>
<topic> <partition.id> <offset>
...
...
```

`<record_partition_num>` 为文件中记录的条数，下面的每一条记录为以空格分隔的 topic 名，partition，和对应 offset。

移动分区数据，就是停止 kafka 服务，打包 `<topic.name>-<partition.id>` 目录数据，移动到其他目录，并修改对应目录的 `recovery-point-offset-checkpoint` 和 `replication-offset-checkpoint` 文件记录

## 步骤示例

假设有 2 个磁盘 `/data1/kafka-logs/` 和 `/data2/kafka-logs`，需要把 `/data1/kafka-logs/` 下 `test_topic` 分区 1 移动到 `/data2/kafka-logs/`

### 1. 停止节点服务

### 2. 移动分区数据

打包数据目录：

    cd /data1/kafka-logs/
    tar zcvf test_topic-1.tar.gz test_topic-1

如果 2 块磁盘是同一个 broker 下

    mv test_topic-1.tar.gz /data2/kafka-logs/
    rm test_topic-1.tar.gz
    rm -rf test_topic-1

### 3. 修改 offset 文件

进入 `/data1/kafka-logs` 目录查看 `replication-offset-checkpoint`, `replication-offset-checkpoint` 文件，假设内容如下:

recovery-point-offset-checkpoint

```
0
102
...
test_topic 1 144458735
...
```

replication-offset-checkpoint

```
0
102
...
test_topic 1 144465512
...
```

进入 `/data2/kafka-logs` 目录查看 `replication-offset-checkpoint`, `replication-offset-checkpoint` 文件，假设内容如下:

recovery-point-offset-checkpoint

```
0
99
...
...
```

replication-offset-checkpoint

```
0
99
...
...
```

#### 3.1 在 /data2/kafka-logs 对应文件下添加 test_topic 的记录

进入 `/data2/kafka-logs` 目录，将 test_topic 的记录添加到对应的文件中，并增加记录数，文件内容修改如下：

recovery-point-offset-checkpoint

```
0
100
...
...
test_topic 1 144458735
```

replication-offset-checkpoint

```
0
100
...
...
test_topic 1 144465512
```

#### 3.2 在 /data1/kafka-logs 对应文件下去除 test_topic 的记录

进入 `/data1/kafka-logs` 目录，将对应的文件中 test_topic 的记录去除，并减少记录数，文件内容修改如下：

recovery-point-offset-checkpoint

```
0
101
...
...
```

replication-offset-checkpoint

```
0
101
...
...
```

> 这里的示例是同一节点不同磁盘之间移动分区，如果是不同节点之间恢复数据，这里的记录不用去除

### 4. 启动节点服务


