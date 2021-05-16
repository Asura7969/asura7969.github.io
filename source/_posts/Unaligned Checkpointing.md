---
title: Unaligned Checkpointing
date: 2021-03-08 08:31:57
tags: flink
categories: flink
cover: https://i.loli.net/2021/05/16/K1d5AkHxIVXagZU.png
---


# Unaligned Checkpointing
## 背景

Flink Checkpoint 基于 Chandy-Lamport 算法实现的。

目前的 Checkpoint 算法在大多数情况下运行良好，然而当作业出现反压时，阻塞式的 Barrier 对齐反而会加剧作业的反压，甚至导致作业的不稳定。

## 当前checkpoint机制
![Aligned Checkpoint](http://www.whitewood.me/img/flink-unaligned-checkpoint/img2.barrier-alignment.png)

* 当operator 接收到 第一个 barrier (b1) 时，会把其所在的后续数据写入 buffer ，直到 buffer 写满阻塞 channel
* 同时处理其它未接受到 barrier (b1) 的 channel，这些 channel 的数据会输出到该 operator 的 outputChannel中，往下游节点发送
* 当所有channel接受到 barrier (b1) 后，该 operator 会先往 outputChannel 发送 b1,再把 buffer 中的数据与 channel 中的后续数据输出到下游


## Unaligned Checkpoint

![Unaligned Checkpoint](https://ci.apache.org/projects/flink/flink-docs-release-1.12/fig/stream_unaligning.svg)

![barrier越过的数据](http://www.whitewood.me/img/flink-unaligned-checkpoint/img7.barrier-overtake-data.png)

* 当第一个barrier (b1) 快到达 operator 时，会优先处理 barrier(b1)，开始 checkpoint，将第一个 barrier (b1) 移至 outputChannel 的末端
* operator 继续处理上游的 channel, 同时算子会将 b1 越过的数据写入 checkpoint，并将其它channel（除 b1 所在的channel）后续早于 b1 的数据持续写入 checkpoint

## 注
Unaligned Checkpoint 并不是百分百优于 Aligned Checkpoint，它会带来的已知问题就有:
* 由于要持久化缓存数据，State Size 会有比较大的增长，磁盘负载会加重。
* 随着 State Size 增长，作业恢复时间可能增长，运维管理难度增加。

目前看来，Unaligned Checkpoint 更适合容易产生高反压同时又比较重要的复杂作业。对于像数据 ETL 同步等简单作业，更轻量级的 Aligned Checkpoint 显然是更好的选择。

## 参考
[FLIP-76: Unaligned Checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/stateful-stream-processing.html#unaligned-checkpointing)

[flink官网](https://ci.apache.org/projects/flink/flink-docs-release-1.12/concepts/stateful-stream-processing.html#unaligned-checkpointing)

[时间与精神的小屋](http://www.whitewood.me/2020/06/08/Flink-1-11-Unaligned-Checkpoint-%E8%A7%A3%E6%9E%90/)


