---
title: Spark Sql
date: 2020-11-01 08:31:57
tags: spark
categories: sql
cover: /img/topimg/202105161052.png
---


# Spark SQL

![spark sql](https://databricks.com/wp-content/uploads/2015/04/Screen-Shot-2015-04-12-at-8.41.26-AM.png)

![LogicalPlan.png](http://ww1.sinaimg.cn/large/b3b57085gy1gk4adpqbbjj20ry0hggmu.jpg)

`Analysis`

从**SQL**或者**DataFrame API**中解析得到抽象语法树,依据catalog元数据校验语法树(表名、列名,或列类型),将*Unresolved Logical Plan*解析成*Resolved Logical Plan*

多个性质类似的*Rule*组成一个*Batch*,多个*Batch*构成一个*Batchs*,这些batches会由*RuleExecutor*执行,先按一个一个Batch顺序执行,然后对Batch里面的每个*Rule*顺序执行。每个Batch会执行一次会多次。

`Logical Optimizations`

基于规则优化,其中包含谓词下推、列裁剪、常亮折叠等。利用*Rule*(规则)将*Resolved Logical Plan*解析成*Optimized Logical Plan*,同样是由*RuleExecutor*执行

![PhysicalPlan.png](http://ww1.sinaimg.cn/large/b3b57085gy1gk4aesi38sj20r60abt9c.jpg)

`Physical Planning`

前面的*Logical Plan*不能被*Spark*执行,这个过程是把*Logical Plan*转换成多个*Physical Plan*(物理计划),然后利用*Cost Mode*(代价模型)选择最佳的执行计划;

和前面的逻辑计划绑定和优化不一样,这里使用*Strategy*(策略),而前面介绍的逻辑计划绑定和优化经过*transform*动作之后,树的类型没有改变,也就是说:*Expression* 经过 transformations 之后得到的还是 *Expression* ;*Logical Plan* 经过 Transformations 之后得到的还是*Logical Plan*。而到了这个阶段，经过 Transformations 动作之后，树的类型改变了，由*Logical Plan*转换成*Physical Plan*了。
一个*Logical Plan*(逻辑计划)经过一系列的策略处理之后,得到多个物理计划,物理计划在**Spark**是由*SparkPlan*实现的。多个*Physical Plan*再经过*Cost Model*(代价模型,CBO)得到选择后的物理计划(*Selected Physical Plan*)

## CBO
估算所有可能的物理计划的代价，并挑选出代价最小的物理执行计划。

`Cost = rows * weight + size * (1 - weight)`

* rows:记录行数代表了 CPU 代价
* size:代表了 IO 代价
* **spark.sql.cbo.joinReorder.card.weight**

### LogicalPlan统计信息

LogicalPlanStats以trait的方式在每个LogicalPlan中实现
```scala 
/**
 * A trait to add statistics propagation to [[LogicalPlan]].
 */
trait LogicalPlanStats { self: LogicalPlan =>
  def stats: Statistics = statsCache.getOrElse {
    // 开启cbo 统计,只实现了Aggregate、Filter、Join、Project
    // 其余逻辑还是复用SizeInBytesOnlyStatsPlanVisitor
    // 主要统计 rowCount,size,ColumnStat(列统计信息)
    if (conf.cboEnabled) {
      // 除了统计节点的字节数
      statsCache = Option(BasicStatsPlanVisitor.visit(self))
    } else {
      // 仅仅统计节点的大小(以字节为单位)
      statsCache = Option(SizeInBytesOnlyStatsPlanVisitor.visit(self))
    }
    statsCache.get
  }
  /** A cache for the estimated statistics, such that it will only be computed once. */
  protected var statsCache: Option[Statistics] = None
}
```
如果开启**CBO**,在**Optimize**阶段,会通过收集的表信息对InnerJoin sql进行优化,如下图:
![CBO代码截图.png](http://ww1.sinaimg.cn/large/b3b57085gy1gkdhhs318gj20yp0mg0z0.jpg)

`Code Generation`
生成java字节码

前面生成的*Physical Plan*还不能直接交给Spark执行,Spark最后仍然会用一些Rule对SparkPlan进行处理,如下:

*QueryExecution*
```scala
/** A sequence of rules that will be applied in order to the physical plan before execution. */
  protected def preparations: Seq[Rule[SparkPlan]] = Seq(
    PlanSubqueries(sparkSession),                           // 特殊子查询物理计划处理
    EnsureRequirements(sparkSession.sessionState.conf),     // 确保执行计划分区与排序的正确性
    CollapseCodegenStages(sparkSession.sessionState.conf),  // 代码生成
    ReuseExchange(sparkSession.sessionState.conf),          // 节点重用
    ReuseSubquery(sparkSession.sessionState.conf))          // 子查询重用
```

[CodeGenerator](https://github.com/apache/spark/blob/fedbfc7074dd6d38dc5301d66d1ca097bc2a21e0/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/expressions/codegen/CodeGenerator.scala)


**Join Cardinality(基数)**

`Inner Join` : num(A IJ B) = num(A)*num(B)/max(distinct(A.k),distinct(B.k))

`Left-Outer Join` : num(A LOJ B) = max(num(A IJ B),num(A))

`Right-Outer Join` : num(A ROJ B) = max(num(A IJ B),num(B))

`Full-Outer Join` : num(A FOJ B) = num(A LOJ B) + num(A ROJ B) - num(A IJ B)

cost = weight * cardinality + (1.0 - weight) * size

> https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html
> https://databricks.com/blog/2017/08/31/cost-based-optimizer-in-apache-spark-2-2.html


## Spark Join
### BroadcastJoin
![spark-broadcastjoin.png](http://ww1.sinaimg.cn/large/b3b57085gy1gk3ljsv6hbj20n30coaaz.jpg)
#### 匹配条件
* 等值连接
* 是否有提示(hit)
* 匹配join类型
* 表大小是否小于阈值

#### 执行步骤
* 将小表先拉到driver端,然后在广播到所有executor
* `spark.sql.autoBroadcastJoinThreshold`(默认值为10M)

将broadcat的数据逐行hash,存储到**BytesToBytesMap**,遍历**stream**表,逐行取出hash匹配,找出符合条件的数据

### Shuffle Hash Join
![Shuffle Hash Join.png](http://ww1.sinaimg.cn/large/b3b57085gy1gk3lo45m3sj20oc085aay.jpg)
#### 匹配条件
* 等值连接
* 是否优先执行SMJ(SparkConf配置) && 满足join类型 && 表大小 < bhj阈值 * 默认shuffle分区数(200) && 小表大小 * 3 <= 大表大小 `||` leftkey的类型不能被排序

#### 执行步骤
* shuffle:先对join的key分区,将相同的key分布到同一节点
* hash join:对每个分区中的数据进行join操作,现将小表分区构造一张Hash表(HashedRelation),然后根据大表分区中的记录的key值进行匹配

### SMJ
![spark-smj.jpg](http://ww1.sinaimg.cn/large/b3b57085gy1gk36a7fl00j20nv0cb0tl.jpg)
#### 匹配条件
* 等值连接
* leftkey的类型能被排序

#### 执行步骤
* shuffle:先对join的key分区,将相同的key分布到同一节点
* sort:每个分区的两个表排序
* merge:排号序的两张表join,分别遍历两个有序表,遇到相同的key就merge输出（如果右表key大于左表,则左表继续往下遍历;反之右表往下遍历,直至两表key相等,合并结果）

## Spark 连接 Hive

![HiveThriftServer2启动流程.png](http://ww1.sinaimg.cn/large/b3b57085gy1gkf7r39yi3j21gs0u2won.jpg)

最后sql的执行由**SparkSQLOperationManager**中创建的**SparkExecuteStatementOperation**执行并返回结果

**HiveThriftServer2启动构建对象**

![HiveThritfServer2-main.png](http://ww1.sinaimg.cn/large/b3b57085gy1gkf7ks8yk3j21wi16i4qp.jpg)


## 谓词下推源码
> http://spark.coolplayer.net/?p=3452
```scala
// PushDownPredicate

// project里面的field必须是确定的，并且condition的输出和grandChild的输出有交集
case Filter(condition, project @ Project(fields, grandChild))
      if fields.forall(_.deterministic) && canPushThroughCondition(grandChild, condition) =>

// 聚合函数包含的表达式必须是确定的，filter的字段必须要在group by的维度字段里面
case filter @ Filter(condition, aggregate: Aggregate)
      if aggregate.aggregateExpressions.forall(_.deterministic)
        && aggregate.groupingExpressions.nonEmpty =>

// 谓词下推的表达式必须是窗口聚合的分区key，谓词必须是确定性的
case filter @ Filter(condition, w: Window)
      if w.partitionSpec.forall(_.isInstanceOf[AttributeReference]) =>


case filter @ Filter(condition, union: Union) =>

case filter @ Filter(condition, watermark: EventTimeWatermark) =>

// filter的子节点只有部分类型才可以谓词下推，表达式必须是确定性的
case filter @ Filter(_, u: UnaryNode)
        if canPushThrough(u) && u.expressions.forall(_.deterministic) =>
```