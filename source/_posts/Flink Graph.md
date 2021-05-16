---
title: Flink Graph
date: 2021-02-20 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202105161043.png
---

#### StreamGraph

是通过用户的Stream API编写的代码生成最初的图，用来表示程序的拓扑结构
* StreamNode：用来代表operator的类，并具有所有相关的属性，如并发度、入边和出边等。
* StreamEdge：表示连接两个StreamNode的边。


#### JobGraph

StreamGraph经过优化后生成JobGraph，提交给**JobManager**数据结构。主要是将多个符合条件的节点chain在一起作为一个节点，这样可以减少数据在节点之间流动所需的序列化/反序列化/传输消耗。
* JobVertex：经过优化后符合条件的多个StreamNode可能会chain在一起生成一个JobVertex，即一个JobVertex包含一个或多个operator，JobVertex的输入是JobEdge，输出是IntermediateDataSet。
* IntermediateDataSet：表示JobVertex的输出，即经过operator处理产生的数据集。producer是JobVertex，consumer是JobEdge。
* JobEdge：代表了job graph中的一条数据传输通道。source是IntermediateDataSet，target是JobVertex。即数据通过JobEdge由IntermediateDataSet传递给目标JobVertex。

![Graph](http://img3.tbcdn.cn/5476e8b07b923/TB1tA_GJFXXXXapXFXXXXXXXXXX)

#### ExecutionGraph

**JobManager**根据JobGraph生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。
* ExecutionJobVertex：和JobGraph中的JobVertex一一对应。每一个ExecutionJobVertex都有和并发度一样多的ExecutionVertex。
* ExecutionVertex：表示ExecutionJobVertex的其中一个并发子任务，输入是ExecutionEdge，输出是IntermediateResultPartition。
* IntermediateResult：和JobGraph中的IntermediateDataSet一一对应。一个IntermediateResult包含多个IntermediateResultPartition，其个数等于该operator的并发度。
* IntermediateResultPartition：表示ExecutionVertex的一个输出分区，producer是ExecutionVertex，consumer是若干个ExecutionEdge。
* ExecutionEdge：表示ExecutionVertex的输入，source是IntermediateResultPartition，target是ExecutionVertex。source和target都只能是一个。
* Execution：是执行一个ExecutionVertex的一次尝试。当发生故障或者数据需要重算的情况下ExecutionVertex可能会有多个ExecutionAttemptID。一个Execution通过ExecutionAttemptID来唯一标识。JM和TM之间关于task的部署和task status的更新都是通过ExecutionAttemptID来确定消息接受者。

#### 物理执行图

**JobManager**根据ExecutionGraph对Job进行调度后，在各个TaskManager上部署Task后形成的“图”，并不是一个具体的数据结构。
* Task：Execution被调度后在分配的TaskManager中启动对应的Task。Task包裹了具有用户执行逻辑的operator。
* ResultPartition：代表由一个Task的生成的数据，和ExecutionGraph中的IntermediateResultPartition一一对应。
* ResultSubpartition：是ResultPartition的一个子分区。每个ResultPartition包含多个ResultSubpartition，其数目要由下游消费Task数和DistributionPattern来决定。
* InputGate：代表Task的输入封装，和JobGraph中JobEdge一一对应。每个InputGate消费了一个或多个的ResultPartition。
* InputChannel：每个InputGate会包含一个以上的InputChannel，和ExecutionGraph中的ExecutionEdge一一对应，也和ResultSubpartition一对一地相连，即一个InputChannel接收一个ResultSubpartition的输出。