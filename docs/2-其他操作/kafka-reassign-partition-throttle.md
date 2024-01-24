### 迁移计划描述

假设有迁移计划：

``` json
t0, p0: 101, 102 -> 102, 103                 101 -> 103
t0, p1: 102, 103 -> 103, 104                 102 -> 104
t0, p2: 103, 101 -> 104, 101                 103 -> 104
t1, p0: 101, 102, 103 -> 102, 103, 104       101 -> 104
```

根据迁移计划获取添加 leader 限制的副本

| topic | partition | broker | leader |
|-------|-----------|--------|--------|
| t0    | p0        | 101    | √      |
| t0    | p0        | 102    |        |
| t0    | p1        | 102    | √      |
| t0    | p1        | 103    |        |
| t0    | p2        | 103    | √      |
| t0    | p2        | 101    |        |
| t1    | p0        | 101    | √      |
| t1    | p0        | 102    |        |
| t1    | p0        | 103    |        |

根据迁移计划获取添加 follower 限制的副本

| topic | partition | broker |
|-------|-----------|--------|
| t0    | p0        | 103    |
| t0    | p1        | 104    |
| t0    | p2        | 104    |
| t1    | p0        | 104    |

### 限流最小值计算

- leader 流量限制：对每个 broker，可以根据重新分配分区方案，获取该 broker 上涉及到的 topic partition leader 写入流量之和，broker 提供的 fetch 请求响应流量必须大于写入流量，因此设置该 broker 的 `leader.replication.throttled.rate` 大于这个值。
- follower 流量限制：对每个 broker，可以根据重新分配分区方案，获取该 broker 上新增的 topic partition，进一步获取这些 topic partition 在当前 leader 节点写入流量之和，broker 的 fetch 请求流量必须大于写入流量，因此设置该 broker 的 `follower.replication.throttled.rate` 大于这个值。

``` python
# 当前分区分布
current_assignments = [
    {
        "topic": "t0",
        "partition": 0,
        "replicas": [
            101,
            102
        ]
    },
    {
        "topic": "t0",
        "partition": 1,
        "replicas": [
            102,
            103
        ]
    },
    {
        "topic": "t0",
        "partition": 2,
        "replicas": [
            103,
            101
        ]
    },
    {
        "topic": "t1",
        "partition": 0,
        "replicas": [
            101,
            102,
            103
        ]
    }
]

# 目标分区分布
proposed_assignments = [
    {
        "topic": "t0",
        "partition": 0,
        "replicas": [
            102,
            103
        ]
    },
    {
        "topic": "t0",
        "partition": 1,
        "replicas": [
            103,
            104
        ]
    },
    {
        "topic": "t0",
        "partition": 2,
        "replicas": [
            104,
            101
        ]
    },
    {
        "topic": "t1",
        "partition": 0,
        "replicas": [
            102,
            103,
            104
        ]
    }
]


def get_traffic(broker, topic, partition):
    # 从 bcm 获取流量值
    return 1


# 获取 topic partition leader 当前分布
topic_partition_leader = {}

# 获取当前每个 broker 上的 topic partition 分布
current_distribution = {}

current_distribution_map = {}

for partition_assign in current_assignments:
    topic = partition_assign["topic"]
    partition = partition_assign["partition"]
    replicas = partition_assign["replicas"]

    # topic partition 最优 leader 为 replicas 第一个值
    topic_partition_leader.setdefault(topic, {})[partition] = replicas[0]

    for replica in replicas:
        current_distribution.setdefault(replica, {}).setdefault(topic, []).append(partition)

    current_distribution_map.setdefault(topic, {})[partition] = replicas


# 获取迁移后，每个 broker 上的 topic partition 分布
proposed_distribution = {}

proposed_distribution_map = {}

for partition_assign in proposed_assignments:
    topic = partition_assign["topic"]
    partition = partition_assign["partition"]
    replicas = partition_assign["replicas"]

    for replica in replicas:
        proposed_distribution.setdefault(replica, {}).setdefault(topic, []).append(partition)

    proposed_distribution_map.setdefault(topic, {})[partition] = replicas

# 获取每个 broker 上将要增加的 topic partition
add_partitions = {}

# 获取每个 broker 上将要删除的 topic partition
remove_partitions = {}

for broker in proposed_distribution.keys() | current_distribution.keys():
    current_topic_partitions = current_distribution.get(broker, {})
    proposed_topic_partitions = proposed_distribution.get(broker, {})

    for topic in current_topic_partitions.keys() | proposed_topic_partitions.keys():
        current_partitions = current_topic_partitions.get(topic, [])
        proposed_partitions = proposed_topic_partitions.get(topic, [])

        remove_partition = list(set(current_partitions) - set(proposed_partitions))
        if remove_partition:
            remove_partitions.setdefault(broker, {})[topic] = remove_partition

        add_partition = list(set(proposed_partitions) - set(current_partitions))
        if add_partition:
            add_partitions.setdefault(broker, {})[topic] = add_partition


# 重新分配分区过程中，每个 broker 上的 topic partition 分布
in_progress_distribution = {}

for broker, topic_partitions in current_distribution.items():
    for topic, partitions in topic_partitions.items():
        in_progress_distribution.setdefault(broker, {})[topic] = partitions
        if broker in add_partitions and topic in add_partitions[broker]:
            in_progress_distribution.setdefault(broker, {})[topic].extend(add_partitions[broker][topic])

in_progress_distribution_map = {}

for topic, partitions in current_distribution_map.items():
    for partition, replicas in partitions.items():
        in_progress_distribution_map.setdefault(topic, {})[partition] = replicas
        in_progress_distribution_map.[topic][partition].extend(proposed_distribution_map[topic, partition])

# 记录 topic partition 当前流量
topic_partition_in_traffic = {}

# 聚合每个 broker 上涉及的 topic partition leader 流入流量
broker_leader_in_traffic = {}

# 聚合每个 broker 上涉及的 topic partition leader 流出流量
broker_leader_out_traffic = {}

for broker, topic_partitions in current_distribution.items():
    for topic, partitions in topic_partitions.items():
        # 理想状态下，获取 broker - topioc - partition 的流量 BytesInPerSec
        # for partition in partitions:
        #     bytes_in_per_sec = get_traffic(broker, topic, partition)
        #     if broker_leader_in_traffic[broker] is None:
        #         broker_leader_in_traffic[broker] = 0
        #     broker_leader_in_traffic[broker] += bytes_in_per_sec

        # 但是实际只能获取到 broker - topic 维度的流量 BytesInPerSec
        bytes_in_per_sec = get_traffic(broker, topic, None)
        if not broker in broker_leader_in_traffic:
            broker_leader_in_traffic[broker] = 0
        broker_leader_in_traffic[broker] += bytes_in_per_sec  # broker 上所有 topic 写入流量之和

        # TODO: 要提供多少个同步副本？
        if not broker in broker_leader_out_traffic:
            broker_leader_out_traffic[broker] = 0
        broker_leader_out_traffic[broker] += bytes_in_per_sec  # broker 上所有 topic 写入流量之和

        # 记录当前 topic partition 在 leader 节点流量
        # 问题：
        #   1. 当前节点是否为 partition leader ？其实不是问题，只有是 partition leader 的情况下才能获取到流量，不是的话流量为 0
        #   2. 遍历每个副本，意味着不能简单的求和，理想情况下，只有一个副本有值，其他副本都为 0
        #   3. 如果求历史流量的值，可能发生 leader 切换，可能有多个副本有值，所以这里求最大值
        #   4. 无法取得 partition 级别的流量，只能以 topic 级别代替。如果一个 broker 上有多个 partition leader，会导致得到的分区流量偏大
        if not topic in topic_partition_in_traffic:
            topic_partition_in_traffic[topic] = {}

        for partition in partitions:
            traffic = topic_partition_in_traffic[topic].get(partition, 0)
            topic_partition_in_traffic[topic][partition] = max(bytes_in_per_sec, traffic)



# 计算 follower 需要流量
# 这部分主要看哪些节点有新增的 partition
broker_follower_traffic = {}
for broker, topic_partitions in add_partitions.items():
    for topic, partitions in topic_partitions.items():
        # 在 broker 上增加 topic partition
        # 需要获取这个 topic partition leader 当前的流量

        # 理想状态下，获取 broker - topioc - partition 的流量
        # 但是只能获取到 broker -topic 维度的流量 BytesInPerSec，会使得这里流量偏大
        for partition in partitions:
            leader_bytes_in_per_sec = topic_partition_in_traffic[topic][partition]

            # 分区副本的同步速率大于 leader 的写入速率
            if not broker in broker_follower_traffic:
                broker_follower_traffic[broker] = 0
            broker_follower_traffic[broker] += leader_bytes_in_per_sec


throttle = -1

for broker, traffic in broker_leader_in_traffic.items():
    if traffic > throttle:
        throttle = traffic

for broker, traffic in broker_follower_traffic.items():
    if traffic > throttle:
        throttle = traffic
```

问题：

- kafka JMX 不提供 topic partition 级别的流量监控
- 重分区过程中，发生 leader 切换，可能导致原来设置的流量限制小于该节点 leader 写入速率
- 分批执行，会使得每个批次需要的实际流量更小
- 生成重分区方案有随机性，意味着预估依据的方案必须与实际执行的方案一致

### 迁移耗时预估

``` python
def get_log_size(broker, topic, partition):
    # 从 bcm 获取 topic, partition 占用大小
    return 1

# 获取要移动的 topic partition log size
topic_partition_log_size = {}
for broker, topic_partitions in remove_partitions.items():
    for topic, partitions in topic_partitions.items():
        for partition in partitions:
            topic_partition_log_size.setdefault(topic, {})[partition] = get_log_size(broker, topic, partition)

# leader 节点需要提供这些 topic partition 的 fetch 流量
broker_leader_log_size = {}

# follower 节点需要新增这些 topic partition
broker_follower_log_size = {}

for broker, topic_partitions in add_partitions.items():
    for topic, partitions in topic_partitions.items():
        for partition in partitions:
            log_size = topic_partition_log_size[topic][partition]

            if not broker in broker_follower_log_size:
                broker_follower_log_size[broker] = 0
            broker_follower_log_size[broker] += log_size

            # leader 节点需要被复制的 topic partition
            # 这里认为最优 leader 为当前 leader，但实际情况可能不同
            leader = topic_partition_leader[topic][partition]

            if not broker in broker_leader_log_size:
                broker_leader_log_size[leader] = 0
            broker_leader_log_size[leader] += log_size

# 获取迁移耗时
max_duration = 0

# 阈值不能设置为最小值，这里增加 20%
throttle = throttle * 1.2

# leader 耗时
for broker, log_size in broker_leader_log_size.items():
    traffic = broker_leader_in_traffic[broker]
    duration = log_size / (throttle - traffic)
    if duration > max_duration:
        max_duration = duration

# follower 耗时
for broker, log_size in broker_follower_log_size.items():
    traffic = broker_follower_traffic[broker]
    duration = log_size / (throttle - traffic)
    if duration > max_duration:
        max_duration = duration
```

问题：

- 这里是限流，但不代表实际的迁移流量和限流一样
- 分批执行，并行度下降，会导致迁移时长增加

