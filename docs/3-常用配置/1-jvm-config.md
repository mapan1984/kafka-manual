# JVM

    -Xmx6g -Xms6g \
    -XX:MetaspaceSize=96m \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=20 \
    -XX:InitiatingHeapOccupancyPercent=35 \
    -XX:G1HeapRegionSize=16M \
    -XX:MinMetaspaceFreeRatio=50 \
    -XX:MaxMetaspaceFreeRatio=80 \
    -XX:+ExplicitGCInvokesConcurrent

## 使用 G1 垃圾回收器

    -XX:+UseG1GC

## 堆内存

kafka 并不需要过高的堆内存，一般堆内存的大小不需要超过 6G。

    KAFKA_HEAP_OPTS='-Xmx6G -Xms6G'

相反，kafka 性能与文件读写性能直接相关，这要求操作系统保留足够的内存作为 page cache，jvm 使用的内存越少，操作系统可用的 page cache 空间越大。

> 确保堆内存最小值（ Xms ）与最大值（ Xmx ）的大小是相同的，防止程序在运行时改变堆内存大小， 这是一个很耗系统资源的过程。
