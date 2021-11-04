# 4 监控

## 副本同步

节点

topic/partition

	kafka_server_FetcherLagMetrics_Value{name="ConsumerLag",clientId="ReplicaFetcherThread-0-4",topic="topnews_server_log_v2",partition="0",}

## 请求/响应队列

    kafka.network:type=RequestChannel,name=RequestQueueSize

Size of the request queue. A congested request queue will not be able to process incoming or outgoing requests.

    kafka.network:type=RequestChannel,name=ResponseQueueSize

Size of the response queue. The response queue is unbounded. A congested response queue can result in delayed response times and memory pressure on the broker.

## 请求次数

    kafka.network:type=RequestMetrics,name=RequestsPerSec,request={Produce|FetchConsumer|FetchFollower}

Request rate.

## 请求耗时

    kafka.network:type=RequestMetrics,name=TotalTimeMs,request={Produce|FetchConsumer|FetchFollower}

Total time in ms to serve the specified request.

    kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request waits in the request queue.

    kafka.network:type=RequestMetrics,name=LocalTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request is processed at the leader.

    kafka.network:type=RequestMetrics,name=RemoteTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request waits for the follower. This is non-zero for produce requests when acks=all.

    kafka.network:type=RequestMetrics,name=ResponseQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time the request waits in the response queue.

    kafka.network:type=RequestMetrics,name=ResponseSendTimeMs,request={Produce|FetchConsumer|FetchFollower}

Time to send the response.
