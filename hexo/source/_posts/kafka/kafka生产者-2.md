---
title: kafka生产者-2
tags: [kafka,源码解析]
abbrlink: f42c133b
date: 2021-12-30 17:25:35
categories: Kafka
---

## 客户端消息发送线程


### 从记录收集器获取数据
​	&emsp;&emsp;生产者发送的消息在客户端首先被保存到[记录收集器]{.orange}中，[发送线程]{.red}需要发送消息时，从中获取就可以了。不过记录器为了使[发送线程]{.red}更好的工
作。 在[发送线程]{.red}需要数据时，[记录收集器]{.orange}能够按照节点将消息重新分组再发送给[发送线程]{.red}。[发送线程]{.red}从[记录收集器]{.orange}中得到每个节点上需要发送
的批记录列表，为每个客户端请求(CLientRequest)。代码如下：
```java
    // org.apache.kafka.clients.producer.internals.Sender#run(long)
void run(long now) {
    Cluster cluster = metadata.fetch();
    // 获取准备发送数据的分区列表
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    // 如果有任何分区的leader还不知道，强制元数据更新
    if (!result.unknownLeaderTopics.isEmpty()) {
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic);
        this.metadata.requestUpdate();
    }

    // 删除任何我们没有准备好发送的节点
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
        Node node = iter.next();
        if (!this.client.ready(node, now)) {
            iter.remove();
            notReadyTimeout = Math.min(notReadyTimeout, this.client.connectionDelay(node, now));
        }
    }

    // 读取记录收集器，按节点整理好每个分区的批记录
    Map<Integer, List<RecordBatch>> batches = this.accumulator.drain(cluster,
                                                                     result.readyNodes,
                                                                     this.maxRequestSize,
                                                                     now);
    if (guaranteeMessageOrder) {
        // Mute all the partitions drained
        for (List<RecordBatch> batchList : batches.values()) {
            for (RecordBatch batch : batchList)
                this.accumulator.mutePartition(batch.topicPartition);
        }
    }

    List<RecordBatch> expiredBatches = this.accumulator.abortExpiredBatches(this.requestTimeout, now);
    // update sensors
    for (RecordBatch expiredBatch : expiredBatches)
        this.sensors.recordErrors(expiredBatch.topicPartition.topic(), expiredBatch.recordCount);

    sensors.updateProduceRequestMetrics(batches);
    
    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
    if (!result.readyNodes.isEmpty()) {
        log.trace("Nodes with data ready to send: {}", result.readyNodes);
        pollTimeout = 0;
    }
    // 建立客户端请求
    sendProduceRequests(batches, now);
    // 执行真正的网络读写请求
    this.client.poll(pollTimeout, now);
}
```
![发送线程-记录收集器](https://i.bmp.ovh/imgs/2021/12/2383641e3175e836.png)


### 创建生产者客户端请求


```java
/**
 * Create a produce request from the given record batches
 */
private void sendProduceRequest(long now, int destination, short acks, int timeout, List<RecordBatch> batches) {
    Map<TopicPartition, MemoryRecords> produceRecordsByPartition = new HashMap<>(batches.size());
    final Map<TopicPartition, RecordBatch> recordsByPartition = new HashMap<>(batches.size());
    for (RecordBatch batch : batches) {
        TopicPartition tp = batch.topicPartition;
        produceRecordsByPartition.put(tp, batch.records());
        recordsByPartition.put(tp, batch);
    }
    // 构造生产者的请求，
    ProduceRequest.Builder requestBuilder =
            new ProduceRequest.Builder(acks, timeout, produceRecordsByPartition);
    // 回调函数
    RequestCompletionHandler callback = new RequestCompletionHandler() {
        public void onComplete(ClientResponse response) {
            handleProduceResponse(response, recordsByPartition, time.milliseconds());
        }
    };

    String nodeId = Integer.toString(destination);
    ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0, callback);
    // 这里只是将请求暂存
    client.send(clientRequest, now);
    log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
}
```
&emsp;&emsp;
### 客户端网络连接对象
&emsp;&emsp;客户端网络连接对象(Networkclient)管理了客户端和服务端之间的网络通信，包括连接的建立、发送客户端请求、读取客户端响应。
- [ ] ready()。从记录收集器获取准备完毕的节点，并连接所有准备好的节点
- [ ] send()。为每个节点创建一个客户端请求，将请求暂存到节点对应的通道中
- [ ] poll()。轮询动作会真正执行网络请求，比如发送请求给节点、并读取响应。

```java

```