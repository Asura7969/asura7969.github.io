---
title: Spark 任务调度
date: 2020-11-01 08:31:57
tags: spark
categories: spark
cover: /img/topimg/202105161053.png
---



# 概述
主要分为以下四个部分:
* 1、构建DAG
提交的 job 将首先被转化为RDD并通过RDD之间的依赖关系构建DAG,提交到调度系统

* 2、切分stage
`DAGScheduler`负责接受由RDD构成的DAG，将一系列RDD划分到不同的`Stage`(`ResultStage` 和 `ShuffleMapStage` 两种),给`Stage`中未完成的`Partition`创建不同类型的task(`ResultTask` 和 `ShuffleMapTask`),
`DAGScheduler`最后将每个`Stage`中的task以TaskSet的形式提交给`TaskScheduler`继续处理。

* 3、调度task
`TaskScheduler`负责从`DAGScheduler`接受`TaskSet`,创建`TaskSetManager`对`TaskSet`进行管理,并将`TaskSetManager`添加到调度池,最后将对Task调度提交给后端接口(`SchedulerBackend`)处理。

* 4、执行task
执行任务,并将任务中间结果和最终结果存入存储体系。


### Task 本地行级别
获取task的本地性级别时,都会等待一段时间,超过时间会退而求其次。

* 1、PROCESS_LOCAL(本地进程)
* 2、NODE_LOCAL(本地节点)
* 3、NO_PREF(没有最佳位置)
* 4、RACK_LOCAL(本地机架)
* 5、ANY(任何)


### TaskSchedulerImpl调度流程
![TaskSchedulerImpl调度流程.png](/img/blog/TaskSchedulerImpl调度流程.png)

