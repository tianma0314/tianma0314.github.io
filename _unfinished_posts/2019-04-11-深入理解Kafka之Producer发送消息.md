---
layout:     post
title:      Kafka源码解析之Producer发送消息流程详解
subtitle:   从一个简单的例子开始
date:       2019-04-11
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Kafka
    - 源码
---

## 前言

网上关于Kafka的文章恒河沙数，源码解析类的优质文章也不在少数，而之所以会决定写这个源码解析系列，是希望能够对自己学习的过程有一个记录，以备之后有遗忘的时候也能够及时查看。

在系列文章中，会从三个部分阅读Kafka的源码，分别是客户端，服务端和番外，客户端和服务端不用多说，番外主要是关注Kafka的一些新特性，主要会介绍Kafka Streams和Kafka与其他流式处理系统的整合案例。

> 此系列文章基于0.10.1.0版本的Kafka，后面番外中的文章可能会基于更新的版本，如有此种情况，会在文章中说明。

## KafkaProducer介绍

在Kafka 0.8.1版本之后，Kafka用Java重写了生产者API，我们称之为新生产者API,使用示例如下：

```java
public class KafkaProducerTest {

    private static final String KAFKA_BROKERS = "10.191.0.17:9092,10.191.0.18:9092,10.191.0.19:9092";

    public static void main(String[] args) {
        Map<String, Object> kafkaParams = new HashMap<>();
        kafkaParams.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA_BROKERS);
        kafkaParams.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        kafkaParams.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        KafkaProducer<String, String> producer = new KafkaProducer(kafkaParams);

        String topicName = "test-liujun";

        for (int i = 0; i < 1000; i++) {
            String msg = "message " + i;
            producer.send(new ProducerRecord<>(topicName, msg));
        }
    }

}
```

新的生产者API使用起来非常简单，只需两步：首先给生产者设置几个基本属性，指定要发送的topic，然后调用send方法，就能将消息发送至Kafka集群。好的，那么我们就从这两步开始，开始层层拨开生产者实现的面纱。

#### KafkaProducer Config

上边的例子中，我们只设置了bootstrap.servers，key.serializer，value.serializer这三个属性，就能够让Producer找到Kafka集群并将消息正确地发送至集群，但是其实关于Producer重要的属性还有其他几项，下面的列表中列出了KafkaProducer重要的配置项及其作用、默认值等。

![important-kafka-producer-config](/img/important-kafka-producer-config.jpg)

#### KafkaProducer send方法详解

当构造完KafkaProducer对象之后，只要指定topic，就能够通过send方法发送消息到服务器了，下面来让我们进入send方法一探究竟。

```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null);
}

public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

单参数的send方法调用双参数的send方法，这两个方法都是异步发送消息，双参的send方法允许生产者提供一个回调方法，使得生产者可以在消息发送到服务器后再做一些事情（注意：由于回调是在当前I/O线程中执行的，所以耗时的同步操作最好放在自己单独的线程中执行），双参的send方法中调用了doSend方法，这个doSend方法便是真正执行发送消息动作的地方，源码（主要代码）如下：

```java
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        // Ⅰ
        // first make sure the metadata for the topic is available
        ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;
        
        // Ⅱ
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer");
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer");
        }
    
        // Ⅲ
        int partition = partition(record, serializedKey, serializedValue, cluster);
        int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
        ensureValidRecordSize(serializedSize);
        tp = new TopicPartition(record.topic(), partition);
        long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
        log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = this.interceptors == null ? callback : new InterceptorCallback<>(callback, this.interceptors, tp);
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
        
        // Ⅳ
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch... 代码略
}
```

整个发送过程大致分为四个阶段，用罗马数字标出，完成任务分别如下：

1. 发送消息总得知道发到哪啊，核心方法waitOnMetadata获取topic对应的元数据，关于Kafka获取及更新元数据的实现我会在后面的文章中详细说明。
2. 序列化消息的Key和Value。
3. 算出消息应该发到哪个partition，然后对发送的消息做一些验证，然后把消息加到累加器中，这个累加器类似是一个缓冲区，在这简单说明一下，Kafka并不是生产一条数据就发送一条，这样的话对于网络IO的压力是非常大的，Kafka的实现是生产消息放入缓冲区，达到条件后把整个缓冲区内的数据一起发送，简言之就是批量发送，具体实现文章后面会有详细说明。
4. 当满足某些条件后，唤醒生产者的sender线程，sender线程的工作机制我会单独起一篇文章，这里只需要知道sender线程会把整个批次的消息发送至Kafka集群，工作机制是Kafka基于Java NIO封装了自己的网络通信架构。


