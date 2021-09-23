---
title: kafka答疑
date: 2021-09-23 19:08:11
tags: kafka
categories: kafka
cover: /img/topimg/202109231909.png
---

## kafka消息大小参数
### broker

#### message.max.bytes
> kafka.log.Log#analyzeAndValidateRecords
* 如果生产者设置压缩, 校验batch大小是否 <= `message.max.bytes`
* 调大此参数后, 消费者获取大小也必须增加

### topic

#### max.message.bytes

* 作用和`message.max.bytes`一样, `max.message.bytes`针对指定topic生效, `message.max.bytes`是全局
* 校验batch大小

### producer

#### max.request.size
> org.apache.kafka.clients.producer.KafkaProducer#ensureValidRecordSize

校验单条消息大小是否大于`max.request.size`, 若大于直接抛出异常

### 参考
> https://zhuanlan.zhihu.com/p/142139663
