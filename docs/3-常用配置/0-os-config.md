# OS

## 文件描述符限制

每个 broker 至少需要创建 number_of_partitions * (partition_size / segment_size) 个 segment 文件。

推荐文件数限制最少为 100000

修改 `/etc/security/limits.conf` 文件

```
#<domain>  <type>  <item>  <value>

# root 用户限制文件数限制 100000
root soft nofile 100000
root hard nofile 100000

# 其他用户限制文件数限制 100000
*    soft nofile 65535
*    hard nofile 65535
```

## socket buffer

修改 `/etc/sysctl.conf` 文件

```
# 单个 socket buffer 的最大内存限制
net.core.optmem_max = 33554432

# 单个 socket receive buffer 默认内存限制
net.core.rmem_default = 33554432

# 单个 socket receive buffer 最大内存限制
net.core.rmem_max = 67108864
```

## vm.max_map_count

每个 segment 文件对应一对 index/timeindex 文件，每个 index/timeindex 文件消耗一个 map 区域

需要确保有足够的虚拟内存可用于 `mmapped` 文件

修改 `/etc/sysctl.conf` 文件

```
vm.max_map_count=262144
```

## 参考

- https://kafka.apache.org/27/documentation.html#hwandos
- https://developer.confluent.io/learn/kafka-performance/
