---
title: Flink window
date: 2020-11-22 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202105161048.png
---


# Flink Window

**Keyed Windows**
```scala
stream
      .key(...)
      .window(...)
      [.trigger(...)]
      [.evictor(...)]
      [.allowedLateness(...)]
      [.sideOutputLateData(...)]
      .reduce/aggregate/fold/apply()
      [.getSideOutput(...)]
```
**No-Keyed Windows**
```scala
stream
      .windowAll(...)
      [.trigger(...)]
      [.evictor(...)]
      [.allowedLateness(...)]
      [.sideOutputLateData(...)]
      .reduce/aggregate/fold/apply()
      [.getSideOutput(...)]
```

## Evictor
可以在执行window trigger之前或之后对window中的数据过滤操作

![evictor.png](/img/blog/evictor.png)

## Trigger
用来判断一个窗口是否需要被触发

TriggerResult的四种状态:
* CONTINUE
* FIRE
* PURGE
* FIRE_AND_PURGE

![trigger.png](/img/blog/trigger.png)

## Time
在 Flink 中 Time  可以分为三种 Event-Time, Processing-Time 以及 Ingestion-Time

## Watermark
就是一个时间戳,在整个job中是单调递增的,当一个 operator 入度为 > 1 时, watermark 取最小的那个

生成watermark的方式有两种,一种是周期性生成,另一种是按特定方式(用户自定义方式)生成。

## Window
![window-mechanics.png](/img/blog/window-mechanics.png)

Flink window的内部实现

**org.apache.flink.streaming.runtime.operators.windowing.WindowOperator**

![flink window内部实现.png](/img/blog/window内部实现.png)


### TimeManager

### TimerService

![timerservice.png](/img/blog/timerservice.png)