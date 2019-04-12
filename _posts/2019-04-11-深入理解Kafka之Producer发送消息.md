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


<style>
table th:nth-of-type(2) {
	width: 100px;
}
</style>

属性名称   |描述    |默认值
:----------|:-------|:---------------
bootstrap.servers|连接Kafka集群的主机名:端口号列表，理论上来讲，你可以只填写集群中的一个broker主机地址+端口号即可，因为这个配置只是用来初始化连接，找到其中一个Broker，其他的便可以通过zk来发现。但通常我们会把集群的Broker都写上，万一只写一个，而那个Broker挂了呢|默认没有值，必填
key.serializer|消息Key的序列化器实现类|没有默认值，必填
value.serializer|消息Value的序列化器实现类|没有默认值，必填


