# 附录：批量脚本

## 列出所有 topic 详情

``` sh
kafka-topics.sh --list --zookeeper ${ZK_CONNECT} > topics.data
while read topic
do
    kafka-topics.sh --describe --zookeeper ${ZK_CONNECT} --topic $topic
done < topics.data
```

## 列出所有 consumer 详情

ZK

``` sh
kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --list > zk.data
while read group
do
    echo ==================== zk group name: $group ===============================
    kafka-consumer-groups.sh --zookeeper ${ZK_CONNECT} --describe --group $group
    echo
    echo
done < zk.data
```

KF `>` 0.9.x.x

``` sh
kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list > kf.data
while read group
do
    echo ==================== kf group name: $group ===============================
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --describe --group $group
    echo
    echo
done < kf.data
```

KF `<=` 0.9.x.x

``` sh
kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER} --list --new-consumer > kf.data
while read group
do
    echo ==================== kf group name: $group ===============================
    kafka-consumer-groups.sh --bootstrap-server ${BOOTSTRAP_SERVER}  --new-consumer --describe --group $group
    echo
    echo
done < kf.data
```

## 批量生产消息（测试）

从文件获取内容写入

``` sh
while [[ -f /etc/hosts ]]; do
     kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic test < /etc/hosts
done
```

写随机字符

``` sh
while true; do
    # random="$(dd if=/dev/urandom bs=1048587 count=1)"
    random="$(od -vAn -N100000 -tu4 < /dev/urandom)"
    echo $random | kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic test
done

while true; do
    # random="$(dd if=/dev/urandom bs=1048587 count=1)"
    random="$(od -vAn -N100000 -tu4 < /dev/urandom)"
    echo "${random}1 ${random}2" | kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic test --sync
done

while true; do
    # random="$(dd if=/dev/urandom bs=1048587 count=1)"
    # random="$(od -vAn -N100000 -tu4 < /dev/urandom)"
    random=$(date +%s%N | sha256sum | head -c 10)
    echo "${random}1 ${random}2" | kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic test --sync
done
```

写 json 内容

``` sh
while true; do
     echo '{"firstname": "name", "lastname": "preferred programming language"}' | kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic test
done

while true; do
     name="$(od -vAn -N4 -tu4 < /dev/urandom)"
     fname="f${name// /}"
     lname="l${name// /}"
     echo "{\"firstname\": \"${fname}\", \"lastname\": \"${lname}\"}" | kafka-console-producer.sh --broker-list ${BOOTSTRAP_SERVER} --topic test
done
```

## 批量创建/删除 topic（测试）

``` sh
for ((i=100; i < 200; i++)) {
    kafka-topics.sh --zookeeper ${ZK_CONNECT} --create --replication-factor 3 --partitions 3 --topic "topic-${i}"
}

for ((i=100; i < 200; i++)) {
   kafka-topics.sh --zookeeper ${ZK_CONNECT}  --delete --topic "topic-${i}"
}
```

``` sh
for ((i=1; i < 200; i++)) {
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --create --replication-factor 3 --partitions 2 --topic "topic_${i}"
}

for ((i=1; i < 200; i++)) {
    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --topic "topic_${i}"
}
```

清空全部主题

``` sh
kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --command-config conf/kafka.properties --list > ts

cur_dir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

while read line; do
    if [[ ${line} == \#* || -z ${line} ]]; then
      echo skip comment line: $line
      continue
    fi

    kafka-topics.sh --bootstrap-server ${BOOTSTRAP_SERVER} --delete --topic $line

done < ts
```

> 指定配置文件 --command-config conf/kafka.properties

## 统计 kafka consumer lag 的总和

``` sh
kafka-consumer-groups.sh --zookeeper $(hostname):2181 --describe --group logstash \
      | tail -n +2 \
      | head -n 32 \
      | awk -F ',' 'BEGIN {sum=0} {sum+=$6} END {print sum}'
```

## 批量删除限流配置

``` bash
#!/bin/bash

while true; do
  echo "Press <CTRL+C> to exit."

  # 循环遍历 broker ID 1 到 5
  for broker_id in {1..5}; do
    kafka-configs.sh --bootstrap-server "${BOOTSTRAP_SERVER}" \
      --entity-type brokers \
      --entity-name "${broker_id}" \
      --alter \
      --delete-config 'leader.replication.throttled.rate,follower.replication.throttled.rate' \
      --command-config /usr/local/ams-worker/conf/kafka.properties
  done

  sleep 60
done
```

