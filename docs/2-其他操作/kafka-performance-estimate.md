# kafka 理论极限性能预估

参考：[Best practices for right-sizing your Apache Kafka clusters to optimize performance and cost](https://aws.amazon.com/cn/blogs/big-data/best-practices-for-right-sizing-your-apache-kafka-clusters-to-optimize-performance-and-cost/)


```
max(Tcluster) <= min{
  max(Tstorage) * #brokers / r,
  max(Tcds_network) * #brokers / r,

  max(Tbcc_network) * #brokers / (#consumer_groups + r - 1)
}
```

## 理想集群

假设集群在理想状态，即：

* 所有 topic 的 分区 leader, 副本都均匀分布在每个 broker 上
* 生产者/消费者对 topic 的每个分区的读写流量都是相同的
* 生产消费行为之间是同步的，没有延迟

这时所有的 broker 都承载相同的流量和压力。

假设：

* 集群写入流量：`Tcluster`
* 集群节点数：`#broker`
* 主题副本数：`r`
* 消费组数：`#consumer_groups`


```
节点流入数据 = 生产者写入节点流量 + 副本同步拉取流量
             = Tcluster / #brokers + Tcluster / #brokers * (r - 1)
             = Tcluster / #brokers * r
```

```
节点流出数据 = 消费组读取节点流量 + 副本同步读取流量
             = Tcluster / #brokers * #consumer_groups + Tcluster / #brokers * (r – 1)
             = Tcluster / #brokers * (#consumer_groups + r - 1)
```

## 磁盘瓶颈

```
Tstorage = 节点流入数据
         = 生产者写入流量 + 副本同步写入流量
         = Tcluster / #brokers + Tcluster / #brokers * (r-1)
         = Tcluster / #brokers * r

max(Tcluster) <= max(Tstorage) * #brokers / r

#broker = Tcluster * r / max(Tstorage)
```

## 节点带宽瓶颈

```
Tbcc_network = 消费组读取流量 + 副本同步读取流量
             = Tcluster / #brokers * #consumer_groups + Tcluster / #brokers * (r – 1)
             = Tcluster / #brokers * (#consumer_groups + r - 1)

max(Tcluster) <= max(Tbcc_network) * #brokers / (#consumer_groups + r - 1)

#broker = Tcluster * (#consumer groups + r - 1) / max(Tbcc_network)
```

