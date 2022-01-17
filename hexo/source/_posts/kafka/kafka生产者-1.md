---
title: kafka生产者-1
tags: [kafka,源码解析]
abbrlink: 6d254281
date: 2021-12-30 11:15:57
categories: Kafka
---

## 同步和异步发送消息

​	&emsp;&emsp;KafkaProducer只用了一个send方法，就可以完成同步和异步两钟模式的消息发送,这是因为send方法返回的是一个Future。基于Future,我们可以实现同步和异步的消息发送语义。
- [x] 同步。调用send返回Future时，需要立即调用get,因为Future.get在没有返回结果时会一直阻塞。
- [x] 异步。提供一个回调函数，调用send后可以继续发送消息而不用等待，当有结果返回时，会自动执行回调函数。 
```java
public void run() {
    int messageNo = 1;
    while (true) {
        String messageStr = "Message_" + messageNo;
        long startTime = System.currentTimeMillis();
        if (isAsync) { // Send asynchronously
            producer.send(new ProducerRecord<>(topic,
                messageNo,
                messageStr), new DemoCallBack(startTime, messageNo, messageStr));
        } else { // Send synchronously
            try {
                producer.send(new ProducerRecord<>(topic,
                    messageNo,
                    messageStr)).get();
                System.out.println("Sent message: (" + messageNo + ", " + messageStr + ")");
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
        ++messageNo;
    }
}
```
## KafkaProducer的send逻辑
​	&emsp;&emsp;首先序列消息的key和value (消息必须序列化成二进制流的形式才能在网络中传输）,然后为每一条消息选择对应的分区（表示要将息存储到kafka集群的哪个节点上），最后通知发送线程发送消息。

```java
// org.apache.kafka.clients.producer.KafkaProducer#doSend
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
	// first make sure the metadata for the topic is available
    ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
    long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
    Cluster cluster = clusterAndWaitTime.cluster;

    // 序列化消息 key value
    serializedKey = keySerializer.serialize(record.topic(), record.key());  
    serializedValue = valueSerializer.serialize(record.topic(), record.value());
    // 选择这条消息的分区
    int partition = partition(record, serializedKey, serializedValue, cluster);
    // 检验消息大小，< maxRequestSize(max.request.size=1M) && <totalMemorySize(buffer.memory=32M)
    int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
    ensureValidRecordSize(serializedSize);
    // 构造TopicPartition然后追加至记录收集器里
    tp = new TopicPartition(record.topic(), partition);
    RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
    // 追加一条消息到收集后，如果记录收集器满了，通知Sende发送消息
    if (result.batchIsFull || result.newBatchCreated) {
    	log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
        this.sender.wakeup();
    }
    return result.future;
}
```

### 1.为消息选择分区

​	&emsp;&emsp;partition()方法为消息选择一个分区编号。为了保证消息负载均衡地分布到各个服务端节点，对于没有键的消息，通过计数器轮询的方式依次分配到不同的分区上；对于有键的消息，对键计算散列值，然后和主题的分区数进行取模得到分区编号。

```java
// org.apache.kafka.clients.producer.internals.DefaultPartitioner#partition
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    // 获取主题的所以分区，用来实现消息的负责均衡
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    if (keyBytes == null) { // 消息没有key,则均衡发布
        int nextValue = nextValue(topic); // 计数器递增
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
            int part = Utils.toPositive(nextValue) % availablePartitions.size();
            return availablePartitions.get(part).partition();
        } else {
            // no partitions are available, give a non-available partition
            return Utils.toPositive(nextValue) % numPartitions;
        }
    } else { // 消息有key,对消息的key进行散列化后取模。
        // hash the keyBytes to choose a partition
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}
```

### 2.客户端记录收集器

​	&emsp;&emsp;生产者发送的消息先在客户端缓存到记录收集器RecordAccumulator中。
![分区队列](https://i.bmp.ovh/imgs/2021/12/aa98fd6363ff174c.png)

:::info
追加消息步骤如下：
:::

![记录收集器追加消息](https://i.bmp.ovh/imgs/2021/12/2c2602b61c1decc7.png)
    
  &emsp;&emsp;记录收集器的作用是缓存客户端的消息，还需要通过消息发送线程才能将消息发送到服务端。
