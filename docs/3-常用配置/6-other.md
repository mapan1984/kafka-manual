# 其他

## 支持的消息大小

### producer

    max.request.size

### broker

    message.max.bytes

    # replica fetcher
    # 不是绝对的最大值，如果消息超过这个设定，最大值仍由 message.max.bytes, max.message.bytes 决定 (?)
    replica.fetch.max.bytes

### topic

    max.message.bytes

### consumer

    fetch.message.max.bytes

    fetch.max.bytes
