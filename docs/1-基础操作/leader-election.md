# 选举 leader

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
