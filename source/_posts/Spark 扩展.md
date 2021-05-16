---
title: Spark 扩展功能
date: 2020-10-18 08:31:57
tags: spark
categories: spark
cover: /img/topimg/202105161055.png
---

### SMJ 扩展打印信息
#### 执行SortMergeJoinExec(SparkPlan)时打印左右表信息
```
    // 执行SortMergeJoinExec类中任意位置
    val executionId = sparkContext.getLocalProperty(SQLExecution.EXECUTION_ID_KEY)
    val queryExecution = SQLExecution.getQueryExecution(executionId.toLong)
    // 打印 queryExecution.analyzed
```

#### 执行SortMergeJoinExec(SparkPlan)时输出operator分区数,左右表的输入行数

**SortMergeJoinExec**
```scala
  override lazy val metrics = Map(
    "numOutputRows" -> SQLMetrics.createMetric(sparkContext, "number of output rows"),
    "numPartitions" -> SQLMetrics.createMetric(sparkContext, "number of partitions"),
    "leftNumInputRows" -> SQLMetrics.createMetric(sparkContext, "left number of input rows"),
    "rightNumInputRows" -> SQLMetrics.createMetric(sparkContext, "right number of input rows")
  )
  override def doProduce(ctx: CodegenContext): String = {
    // Inline mutable state since not many join operations in a task
    val leftInput = ctx.addMutableState("scala.collection.Iterator", "leftInput",
      v => s"$v = inputs[0];", forceInline = true)
    val rightInput = ctx.addMutableState("scala.collection.Iterator", "rightInput",
      v => s"$v = inputs[1];", forceInline = true)
    //添加
    val numPartitions = metricTerm(ctx, "numPartitions")
    ctx.addSqlMetric(s"$numPartitions.add(1);")

    val (leftRow, matches) = genScanner(ctx)

    ...
  }
```
**CodeGenerator**
```scala
  // 添加以下内容
  val metricInitializationStatements: mutable.ArrayBuffer[String] = mutable.ArrayBuffer.empty

  def addSqlMetric(metric: String): Unit = {
    metricInitializationStatements += metric
  }

  def initMetric(): String = {
    metricInitializationStatements.mkString("\n")
  }
```
**WholeStageCodegenExec**
```scala
    ...
    def doCodeGen(): (CodegenContext, CodeAndComment) = {
      ...
      public void init(int index, scala.collection.Iterator[] inputs) {
        partitionIndex = index;
        this.inputs = inputs;
        ${ctx.initMutableStates()}
        ${ctx.initPartition()}
        // 添加
        ${ctx.initMetric()}
      }
    }
    ...
```


### Spark 扩展自定义语法

* 复制 SqlBase.g4 文件
* 下载 antlr-4.8-complete.jar
* 添加自定义语法
* 生成文件
```shell script
java -Xms500m -cp antlr-4.8-complete.jar org.antlr.v4.Tool 
-o [antlr 生成的java文件路径]
-package org.asuraspark.sql.antlr4
-visitor -listener
-lib E:\IdeaProjects\asuraspark\asuraspark-sql\src\main\scala\org\asuraspark\sql\antlr4\lib
[.g4文件路径]
```
> TODO
