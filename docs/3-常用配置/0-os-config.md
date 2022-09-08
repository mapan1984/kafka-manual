# OS

## 文件描述符限制

## vm.max_map_count

取保有足够的虚拟内存可用于 `mmap`ped 文件

    sysctl -w vm.max_map_count=262144

在 `/etc/sysctl.conf` 通过修改 `vm.max_map_count` 永久设置


## 参考

- https://developer.confluent.io/learn/kafka-performance/
