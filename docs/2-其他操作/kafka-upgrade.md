---
title: Kafka 滚动升级
tags: [kafka]
---

# kafka 滚动升级

kafka 滚动升级参考[官方文档](https://kafka.apache.org/11/documentation.html#upgrade)，升级过程中需要进行两次滚动重启节点操作，可以保证升级过程中集群服务不中断。

下面以 0.9.0.X 升级到 1.1.1 为例，假设 kafka 安装在 `/usr/local/kafka` 下，并用 systemd 管理：

## 第一步

需要依次替换节点 kafka 到 1.1.1 版本，替换过程中保留节点原有 kafka 配置，并新增配置指定 `inter.broker.protocol.version=0.9.0.X`，`log.message.format.version=0.9.0.X` 与现有集群版本兼容，具体操作步骤如下：

``` sh
# 下载 1.1.1 版本 kafka
wget https://archive.apache.org/dist/kafka/1.1.1/kafka_2.12-1.1.1.tgz -O /tmp/kafka_2.12-1.1.1.tgz
wget https://archive.apache.org/dist/kafka/1.1.1/kafka_2.12-1.1.1.tgz.sha512 -O /tmp/kafka_2.12-1.1.1.tgz.sha512

# 校验 kafka 包
# cd /tmp
# [[ `md5sum kafka_1.1.1.tar.gz` == `cat kafka_1.1.1.md` ]] && echo 'ok' || echo 'no'
# [[ `shasum -a512 kafka_2.12-1.1.1.tgz` == `cat kafka_2.12-1.1.1.tgz.sha512` ]] && echo 'ok' || echo 'no'
# [[ `openssl sha512 kafka_2.12-1.1.1.tgz` == `cat kafka_2.12-1.1.1.tgz.sha512` ]] && echo 'ok' || echo 'no'

# 校验成功后解压 kafka 包
tar -zxvf /tmp/kafka_2.12-1.1.1.tgz -C /tmp

# 保留原有配置
mv /tmp/kafka/config /tmp/kafka/config.back
cp -r /usr/local/kafka/config /tmp/kafka/

# 指定通信协议与消息格式
echo inter.broker.protocol.version=0.9.0.X >> /tmp/kafka/config/server.properties
echo log.message.format.version=0.9.0.X >> /tmp/kafka/config/server.properties

# 替换 kafka 包
mv /usr/local/kafka /usr/local/kafka.back
cp -r /tmp/kafka /usr/local/

# 重启 kafka 节点
# service kafka-server restart
systemctl restart kafka
```

## 第二步

Kafka 版本升级有通信协议和消息格式的变化，经过上一步的替换，集群上的 kafka 包已经替换成 1.1.1 版本，通信协议和消息格式仍然指定为 0.9.0.X 版本，之后需要对这两项配置进行更新，但需要注意以下两点：

1. 更改 `log.message.format.version` 之前，所有消费者必须升级到支持 1.1.1 或者之后的版本
2. `inter.broker.protocol.version`, `log.message.format.version` 更改并重启生效之后，kafka 版本不能进行降级回退

操作步骤：

``` sh
# 指定通信协议与消息格式
sed -i "s/inter.broker.protocol.version=0.9.0.X/inter.broker.protocol.version=1.1-IV0/g" /usr/local/kafka/config/server.properties
sed -i "s/log.message.format.version=0.9.0.X/log.message.format.version=1.1-IV0/g" /usr/local/kafka/config/server.properties

# 重启 kafka 节点
# service kafka-server restart
systemctl restart kafka
```
## 注意事项

* [0.10.0.0 之后消息格式改变，如果客户端版本低于 0.10.0.0 会带来性能损失](https://kafka.apache.org/21/documentation.html#upgrade_10_performance_impact)
