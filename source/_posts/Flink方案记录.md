---
title: Flink方案记录
date: 2021-02-21 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202105161042.png
---


#### flink 快慢流双流join

* 1、自定义UDF函数，join不到sleep一下
* 2、在join operator处数据等一会再查
* 3、如果没有join到,把数据重新回流,等再次消费后再次join
* 4、如果source端的MQ支持延迟消息，直接使用MQ的延迟功能(例如kafka的时间轮)
* 5、扩展Flink Source,例如在Kafka Connector中加入time.wait属性,当用户设置这个属性就让source的数据等一会
* 6、对于未匹配上的数据可以先导入外部存储,后续进行匹配  
* 7、如果source端是kafka,数据在写入kafka时候设置key（后续的join key）的值,使得同key的数据延迟度降低

#### flink 作业迁移注意事项
##### flink对接kafka, kafka offset如何保证?
背景:kafka与yarn集群同时迁移到新集群,保证flink作业不丢数据。
使用kafka mirror,从原有kafka集群copy数据到新集群,flink作业做savepoint,地址是新集群地址,在新的yarn集群上启动flink作业。
注意:必要时可以使用state processor api进行修改checkpoint信息


#### flink checkpoint失败原因
[ververica连接](https://ververica.cn/developers/flick-checkpoint-troubleshooting-practical-guide/)

总结:
* 1、如果数据存在倾斜或反压,需先解决,再排查checkpoint是否合理
* 2、查看checkpoint详情页面(flink框架自带的web端监控页面)
    * Acknowledged：subtask 对这个 checkpoint 进行ack的个数
    * Latest Acknowledgement：该 operator 的所有 subtask 最后ack的时间
    * End to End Duration：该 operator 的所有 subtask 完成 snapshot 最长的时间
    * State Size：当前 Checkpoint 的 state 大小
    * Buffered During Alignment：在 barrier 对齐阶段积攒了多少数据，如果这个数据过大也间接表示对齐比较慢
    
checkpoint主要分为两种:Decline(拒绝) 和 Expire(过期)

##### Checkpoint Decline
* 1、先定位失败的checkpoint在哪个taskManager：先去jobManager.log查看
* 2、再去对应的taskManager上查看失败原因

Checkpoint Cancel(Decline的一种情况)：当前checkpoint还没有处理完成,下一个checkpoint已经到达,则会通知下游operator cancel掉当前checkpoint

Checkpoint慢

* 1、使用增量checkpoint
* 2、作业反压或者数据倾斜,需要先解决反压或数据倾斜,在看是否调整checkpoint
* 3、barrier对齐慢：
* 4、主线程太忙,导致checkpoint慢：task是单线程的，如果由于操作state较慢导致的整体处理慢,或者处理barrier慢,需要查看某个PID对应的hotmethod
    * 连续多次jstack，查看一直处于Runnable状态的线程有哪些
    * 通过火焰图查看占用CPU最多的栈
* 5、异步线程慢:对于非RocksDBBackend，主要瓶颈来自于网络；对于RocksDBBackend，还需要考虑本地磁盘性能。


#### flink 每分钟（每秒钟）统计最近24小时的数据

flink清洗数据，按key预聚合每分钟或者每秒钟，存入redis，使用redis的zset，score为时间戳，value为值

注：以redis为方案时，需先测试redis zset的性能，写入、读取响应时间是否符合业务场景

#### flink作业反压排查
* 1、先看 flink web ui 面板
* 2、通过metric，发送端占用buffer和接收端占用buffer（outPoolBuffer 与 inPoolBuffer），定位是哪个节点有问题
    * 通过数据分布（ Web UI 各个 SubTask 的 Records Sent 和 Record Received 来确认 ）
    * taskManager所在的 CPU profile
    * taskManager GC

[如何生成火焰图](https://www.cnblogs.com/CarpenterLee/p/7467283.html)

[如何读懂火焰图](https://www.ruanyifeng.com/blog/2017/09/flame-graph.html)

[jstack排查问题](https://blog.csdn.net/moakun/article/details/80086777)

#### flink 反压解决
* 1、sink端反压：调整并行度
* 2、中间operator反压：解决用户代码执行效率
* 3、source端反压：调大并行度

#### flink stream 小文件合并大致流程
TempFileWriter(parallel) ---(InputFile&EndInputFile)--->
    # 以 broadcast 方式发送下游消息(CompactionUnit 或 EndCompaction),下游operator 会依据id选择对应的分区和文件路径
    CompactCoordinator(non-parallel) --(CompactionUnit&EndCompaction)--->
        # 如果目标文（文件的压缩路径）件存在,直接返回
        # 如果是单个文件，原子性修改文件名
        # 如果是多个文件，多文件压缩
        CompactOperator(parallel)---(PartitionCommitInfo)--->
            # 提交metastore 或 success
            PartitionCommitter(non-parallel)