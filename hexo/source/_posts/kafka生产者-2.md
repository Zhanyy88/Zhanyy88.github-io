---
title: kafka生产者-2
tags: [kafka,源码解析]
abbrlink: f42c133b
date: 2021-12-30 17:25:35
categories: Kafka
---

## 客户端消息发送线程



#### 1.从记录收集器获取数据
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
        // The set of topics with unknown leader contains topics with leader election pending as well as
        // topics which may have expired. Add the topic again to metadata to ensure it is included
        // and request metadata update, since there are messages to send to the topic.
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

    // 读取记录收集器，返回的每个主副本节点对应批记录列表，每个批记录对应一个分区
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

    // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
    // loop and try sending more data. Otherwise, the timeout is determined by nodes that have partitions with data
    // that isn't yet sendable (e.g. lingering, backing off). Note that this specifically does not include nodes
    // with sendable data that aren't ready to send since they would cause busy looping.
    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
    if (!result.readyNodes.isEmpty()) {
        log.trace("Nodes with data ready to send: {}", result.readyNodes);
        pollTimeout = 0;
    }
    sendProduceRequests(batches, now);

    // if some partitions are already ready to be sent, the select time would be 0;
    // otherwise if some partition already has some data accumulated but not ready yet,
    // the select time will be the time difference between now and its linger expiry time;
    // otherwise the select time will be the time difference between now and the metadata expiry time;
    this.client.poll(pollTimeout, now);
}
```



++下划线++
++波浪线++{.wavy}
++着重点++{.dot}
++紫色下划线++{.primary}
++绿色波浪线++{.wavy .success}
++黄色着重点++{.dot .warning}
~~删除线~~
~~红色删除线~~{.danger}
==荧光高亮==
[赤橙黄绿青蓝紫]{.rainbow}
[红色]{.red}
[粉色]{.pink}
[橙色]{.orange}
[黄色]{.yellow}
[绿色]{.green}
[靛青]{.aqua}
[蓝色]{.blue}
[紫色]{.purple}
[灰色]{.grey}
快捷键 [Ctrl]{.kbd} + [C]{.kbd .red}
H~2~0
29^th^


