---
title: Flink SQL LookupJoin With KeyBy
date: 2021-04-21 08:31:57
tags: flink
categories: sql
cover: /img/topimg/20210515223410.jpg
---

# 背景

维表 Join 的场景下因为维表经常发生变化尤其是新增维度，而 Join 操作发生在维度新增之前，经常导致关联不上。
如果 Join 不到，则暂时将数据缓存起来之后再进行尝试，并且可以控制尝试次数，能够自定义延迟 Join 的规则。

[Flink SQL 在字节跳动的优化与实践](https://segmentfault.com/a/1190000039084980)

先看下实现后的效果图:
![flink-sql-lookupJoinNoKeyBy.png](/img/blog/flink-sql-lookupJoinNoKeyBy.png)

![flink-sql-lookupJoinWithKeyBy.png](/img/blog/flink-sql-lookupJoinWithKeyBy.png)


# 实现思路
## Flink sql 整个的执行流程梳理

![PlannerBase.png](/img/blog/FlinkStreamProgram.png)

![FlinkStreamProgram.png](/img/blog/FlinkStreamProgram.png)

## 在哪实现(where)?
实现的地方很多,图1-1 中最引人注意的是 `TEMPORAL_JOIN_REWRITE`(至少我是这么想的...), 但后来实现过程中由于对 `calcite API` 不熟悉,实在无奈,只能另辟蹊径了。

最后看到 `PHYSICAL_RWRITE`, 先大致看了下这个规则下**Rule**的实现过程, 主要就是对已生成的 **physical node** 进行重写。

## 怎么实现(how)?
接下来就是依葫芦画瓢了

![flink-sql 添加KeyByLookupJoinRule.png](/img/blog/KeyByLookupJoinRule.png)

[KeyByLookupRule 实现](https://github.com/Asura7969/asuraflink/blob/main/asuraflink-sql/src/main/scala/com/asuraflink/sql/rule/KeyByLookupRule.scala)

由于实现过程需要 `temporalTable` 和 `calcOnTemporalTable`, 而 **CommonLookupJoin** 中并没有获取这两个对象的方法,因此只能自己手动添加了
![CommonLookupJoin add method.png](/img/blog/CommonLookupJoin.png)

## 如何校验?
本次校验就仅对 **JdbcLookupTableITCase** 该测试类测试
![flink join keyby.png](/img/blog/flinkLookupJoinWithKeyByTest.png)

# 注意事项
* 本次实践基于flink1.12.0, 需要修改源码
* 该实现不一定是最优的
* 如有不足,还请大佬指出