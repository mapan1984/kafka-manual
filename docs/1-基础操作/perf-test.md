# 性能测试

写入压测

    kafka-producer-perf-test.sh \
     --print-metrics \
     --topic __test \
     --num-records 5000 \
     --throughput -1 \
     --record-size 1024 \
     --producer-props bootstrap.servers=${BOOTSTRAP_SERVER} buffer.memory=67108864 batch.size=10240 linger.ms=10 acks=1

通过文件指定消息 payload 内容

    kafka-producer-perf-test.sh \
     --print-metrics \
     --topic __test \
     --num-records 5000 \
     --throughput -1 \
     --payload-file payload.json \
     --producer-props bootstrap.servers=${BOOTSTRAP_SERVER} buffer.memory=67108864 batch.size=10240 linger.ms=10 acks=1

> 指定客户端参数：--producer.config client.properties

消费压测

    kafka-consumer-perf-test.sh  \
     --broker-list ${BOOTSTRAP_SERVER}  \
     --show-detailed-stats \
     --topic __test \
     --messages 5000 \
     --group test_group

    kafka-console-consumer.sh \
     --bootstrap-server ${BOOTSTRAP_SERVER} \
     --topic __test \
     --consumer-property group.id=test_group \
     --from-beginning

    kafka-consumer-perf-test.sh \
     --bootstrap-server ${BOOTSTRAP_SERVER}  \
     --topic __test \
     --print-metrics \
     --show-detailed-stats \
     --timeout 180000 \
     --fetch-size 10485760 \
     --socket-buffer-size 20971520 \
     --messages 5000 \
     --group test_group

> 指定客户端参数：--consumer.config client.properties
