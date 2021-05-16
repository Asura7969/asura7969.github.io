---
title: Flink Sql解析(一)
date: 2020-11-17 08:31:57
tags: flink
categories: sql
cover: /img/topimg/202105161050.png
---


# Flink Sql

**1. Parse**：语法树解析，把sql语句转换成为一个抽象语法树（AST），在Calcite中用SqlNode来表示

**2. Validate**：语法校验，根据元数据信息进行校验，例如查询的表、使用的函数是否存在等，校验完成后仍然是SqlNode构成的语法树

**3. Optimize**：查询计划优化
* 首先将SqlNode语法树转换成关系表达式，RelNode构成的逻辑树
* 然后使用优化器基于规则进行等价交换，例如我们熟悉的谓词下推、列裁剪等，经过优化后得到最有查询计划

**4. Execute**：将逻辑查询计划翻译成物理执行计划，生成对应的可执行代码，提交运行

### Flink parser
**org.apache.flink.table.delegation.Planner**
* **getParser():** 将SQL字符串转换为Table API特定的对象，例如*Operation tree*
* 提供plan，优化并且转换 *Operation( ModifyOperation )* 成可运行的**Transformation**
```java
@Internal
public interface Planner {
    Parser getParser();
    List<Transformation<?>> translate(List<ModifyOperation> modifyOperations);
    ...
}
```
**org.apache.flink.table.planner.delegation.PlannerBase**

```scala
private val planningConfigurationBuilder: PlanningConfigurationBuilder =
    new PlanningConfigurationBuilder(
      config,
      functionCatalog,
      internalSchema,
      expressionBridge)

private val parser: Parser = new ParserImpl(
    catalogManager,
    new JSupplier[FlinkPlannerImpl] {
      override def get(): FlinkPlannerImpl = getFlinkPlanner
    },
    new JSupplier[CalciteParser] {
      // 依据flink自定义的工厂类(FlinkSqlParserImpl)生成parser
      override def get(): CalciteParser = planningConfigurationBuilder.createCalciteParser()
    }
)

override def getParser: Parser = parser

```

**org.apache.flink.table.planner.ParserImpl**

```java
@Override
public List<Operation> parse(String statement) {
    CalciteParser parser = calciteParserSupplier.get();
    FlinkPlannerImpl planner = validatorSupplier.get();
    // 生成SqlNode
    SqlNode parsed = parser.parse(statement);
    // 校验 并 转换为 Operation
    Operation operation = SqlToOperationConverter.convert(planner, catalogManager, parsed)
        .orElseThrow(() -> new TableException(
            "Unsupported SQL query! parse() only accepts SQL queries of type " +
                "SELECT, UNION, INTERSECT, EXCEPT, VALUES, ORDER_BY or INSERT;" +
                "and SQL DDLs of type " +
                "CREATE TABLE"));
    return Collections.singletonList(operation);
}
```
**org.apache.flink.table.sqlexec.SqlToOperationConverter**

```java
public static Optional<Operation> convert(
        FlinkPlannerImpl flinkPlanner,
        CatalogManager catalogManager,
        SqlNode sqlNode) {
    // validate the query
    final SqlNode validated = flinkPlanner.validate(sqlNode);
    SqlToOperationConverter converter = new SqlToOperationConverter(flinkPlanner, catalogManager);
    if (validated instanceof SqlCreateTable) {
        return Optional.of(converter.convertCreateTable((SqlCreateTable) validated));
    } else if
    ...
    else if (validated.getKind().belongsTo(SqlKind.QUERY)) {
        // select 语句转换
        return Optional.of(converter.convertSqlQuery(validated));
    } else {
        return Optional.empty();
    }
}
private Operation convertSqlQuery(SqlNode node) {
    return toQueryOperation(flinkPlanner, node);
}
private PlannerQueryOperation toQueryOperation(FlinkPlannerImpl planner, SqlNode validated) {
    // 将 sqlNode转换为 relNode树, 并返回RelRoot
    RelRoot relational = planner.rel(validated);
    return new PlannerQueryOperation(relational.rel);
}
```
### **SQL 转换及优化**

```scala
abstract class PlannerBase(
    executor: Executor,
    config: TableConfig,
    val functionCatalog: FunctionCatalog,
    val catalogManager: CatalogManager,
    isStreamingMode: Boolean)
  extends Planner {

      override def translate(
        modifyOperations: util.List[ModifyOperation]): util.List[Transformation[_]] = {
          if (modifyOperations.isEmpty) {
            return List.empty[Transformation[_]]
          }
          // prepare the execEnv before translating
          getExecEnv.configure(
            getTableConfig.getConfiguration,
            Thread.currentThread().getContextClassLoader)
          overrideEnvParallelism()
          // 将DML类型的Operation转化为RelNode
          val relNodes = modifyOperations.map(translateToRel)
          // 优化RelNode
          val optimizedRelNodes = optimize(relNodes)
          // 转化为ExecNode
          val execNodes = translateToExecNodePlan(optimizedRelNodes)
          // 转化为Transformation
          translateToPlan(execNodes)
     }
  }

```

Operation(ModifyOperation)转化为`RelNode`，是通过`QueryOperationConverter`（访问者）和`QueryOperation`组成的访问者模式转化为RelNode。
在得到`RelNode`后，就进入`Calcite`对`RelNode`的优化流程

**CommonSubGraphBasedOptimizer** 基于公共子图的优化

```scala
abstract class CommonSubGraphBasedOptimizer extends Optimizer {
  override def optimize(roots: Seq[RelNode]): Seq[RelNode] = {
    // 以 RelNodeBlock 为单位进行优化，在子类中实现，StreamCommonSubGraphBasedOptimizer，BatchCommonSubGraphBasedOptimizer
    val sinkBlocks = doOptimize(roots)
    // 获得优化后的逻辑计划
    val optimizedPlan = sinkBlocks.map { block =>
      val plan = block.getOptimizedPlan
      require(plan != null)
      plan
    }
    // 将 RelNodeBlock 使用的中间表展开
    expandIntermediateTableScan(optimizedPlan)
  }
}
```

```scala
class StreamCommonSubGraphBasedOptimizer(planner: StreamPlanner)
  extends CommonSubGraphBasedOptimizer {

  override protected def doOptimize(roots: Seq[RelNode]): Seq[RelNodeBlock] = {
    val config = planner.getTableConfig
    // build RelNodeBlock plan
    val sinkBlocks = RelNodeBlockPlanBuilder.buildRelNodeBlockPlan(roots, config)
    ...
    // 递归优化RelNodeBlock
    sinkBlocks.foreach(b => optimizeBlock(b, isSinkBlock = true))
    sinkBlocks
  }
}

```

### 参考：
> https://blog.jrwang.me/2019/flink-source-code-sql-overview/