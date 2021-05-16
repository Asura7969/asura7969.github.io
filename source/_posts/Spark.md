---
title: Spark
date: 2020-09-23 08:31:57
tags: spark
categories: spark
cover: /img/topimg/202105161056.png
---


# Spark

> https://databricks.com/blog

## 基本概念

### RDD特点
RDD具有容错机制，并且只读不能修改，RDD具有以下几个属性：
* 只读：不能修改，只能通过转换操作生成新的RDD
* 分布式：可以分布在多台机器上进行并行处理
* 弹性：计算过程中内存不够时会和磁盘进行数据交换
* 基于内存：可以全部或部分缓存在内存中，在多次计算间重用

### RDD血缘关系
RDD 的最重要的特性之一就是血缘关系（Lineage )，它描述了一个 RDD 是如何从父 RDD 计算得来的。如果某个 RDD 丢失了，则可以根据血缘关系，从父 RDD 计算得来。

### RDD依赖类型
根据不同的转换操作，RDD血缘关系的依赖分为`宽依赖`和`窄依赖`。`窄依赖`是指父RDD的每个分区都只被子RDD的一个分区使用。`宽依赖`是指父RDD的每个分区都被子RDD的分区所依赖。
map、filter、union 等操作是窄依赖，而 groupByKey、reduceByKey 等操作是宽依赖。
join 操作有两种情况，如果 join 操作中使用的每个 Partition 仅仅和固定个 Partition 进行 join，则该 join 操作是窄依赖，其他情况下的 join 操作是宽依赖。所以可得出一个结论，窄依赖不仅包含一对一的窄依赖，还包含一对固定个数的窄依赖，也就是说，对父 RDD 依赖的 Partition 不会随着 RDD 数据规模的改变而改变。

## Spark启动流程

`org.apache.spark.deploy.SparkSubmit.main`
```scala

// 依据参数信息准备运行环境
private[deploy] def prepareSubmitEnvironment(
      args: SparkSubmitArguments,
      conf: Option[HadoopConfiguration] = None)
      : (Seq[String], Seq[String], SparkConf, String){
        ...

      // 如果是yarn模式，在YarnClusterApplication中new了一个Client对象，并调用了run方法
      // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
      if (isYarnCluster) {
        childMainClass = YARN_CLUSTER_SUBMIT_CLASS # org.apache.spark.deploy.yarn.YarnClusterApplication
        if (args.isPython) {
          childArgs += ("--primary-py-file", args.primaryResource)
          childArgs += ("--class", "org.apache.spark.deploy.PythonRunner")
        } else if (args.isR) {
          val mainFile = new Path(args.primaryResource).getName
          childArgs += ("--primary-r-file", mainFile)
          childArgs += ("--class", "org.apache.spark.deploy.RRunner")
        } else {
          if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
            childArgs += ("--jar", args.primaryResource)
          }
          childArgs += ("--class", args.mainClass)
        }
        if (args.childArgs != null) {
          args.childArgs.foreach { arg => childArgs += ("--arg", arg) }
        }
      }
}
```
`org.apache.spark.deploy.yarn.Client`内部类

`org.apache.spark.deploy.yarn.YarnClusterApplication`
```scala
private[spark] class YarnClusterApplication extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    // SparkSubmit would use yarn cache to distribute files & jars in yarn mode,
    // so remove them from sparkConf here for yarn mode.
    conf.remove(JARS)
    conf.remove(FILES)

    new Client(new ClientArguments(args), conf, null).run()
  }

}
```

`org.apache.spark.deploy.yarn.Client`
```scala

private[spark] class Client(
    val args: ClientArguments,
    val sparkConf: SparkConf,
    val rpcEnv: RpcEnv) extends Logging {

    // Submit an application to the ResourceManager.
    def run(): Unit = {
      this.appId = submitApplication()
      if (!launcherBackend.isConnected() && fireAndForget) {
        val report = getApplicationReport(appId)
        val state = report.getYarnApplicationState
        logInfo(s"Application report for $appId (state: $state)")
        logInfo(formatReportDetails(report))
        if (state == YarnApplicationState.FAILED || state == YarnApplicationState.KILLED) {
          throw new SparkException(s"Application $appId finished with status: $state")
        }
      } else {
        val YarnAppReport(appState, finalState, diags) = monitorApplication(appId)
        if (appState == YarnApplicationState.FAILED || finalState == FinalApplicationStatus.FAILED) {
          diags.foreach { err =>
            logError(s"Application diagnostics message: $err")
          }
          throw new SparkException(s"Application $appId finished with failed status")
        }
        if (appState == YarnApplicationState.KILLED || finalState == FinalApplicationStatus.KILLED) {
          throw new SparkException(s"Application $appId is killed")
        }
        if (finalState == FinalApplicationStatus.UNDEFINED) {
          throw new SparkException(s"The final status of application $appId is undefined")
        }
      }
    }

    def submitApplication(): ApplicationId = {
      ResourceRequestHelper.validateResources(sparkConf)

      var appId: ApplicationId = null
      try {
        // 初始化并启动yarn client
        launcherBackend.connect()
        yarnClient.init(hadoopConf)
        yarnClient.start()

        logInfo("Requesting a new application from cluster with %d NodeManagers"
          .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))

        // 启动AM，并返回资源信息
        val newApp = yarnClient.createApplication()
        val newAppResponse = newApp.getNewApplicationResponse()
        appId = newAppResponse.getApplicationId()

        val appStagingBaseDir = sparkConf.get(STAGING_DIR)
          .map { new Path(_, UserGroupInformation.getCurrentUser.getShortUserName) }
          .getOrElse(FileSystem.get(hadoopConf).getHomeDirectory())
        stagingDirPath = new Path(appStagingBaseDir, getAppStagingDir(appId))

        new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),
          Option(appId.toString)).setCurrentContext()

        // 校验是否有足够的资源运行AM
        verifyClusterResources(newAppResponse)

        // 设置上下文,启动AM
        val containerContext = createContainerLaunchContext(newAppResponse)
        val appContext = createApplicationSubmissionContext(newApp, containerContext)

        // 提交application
        logInfo(s"Submitting application $appId to ResourceManager")
        yarnClient.submitApplication(appContext)
        launcherBackend.setAppId(appId.toString)
        reportLauncherState(SparkAppHandle.State.SUBMITTED)

        appId
      } catch {
        case e: Throwable =>
          if (stagingDirPath != null) {
            cleanupStagingDir()
          }
          throw e
      }
  }
}
```
`SparkContext`初始化

**yarn-cluster**模式下：client会先申请向RM(Yarn Resource Manager)一个Container,来启动AM（ApplicationMaster）进程,而SparkContext运行在AM（ApplicationMaster）进程中;

**yarn-client**模式下:在提交节点上执行SparkContext初始化,由client类（JavaMainApplication）调用。
```scala
  ...

  // Create and start the scheduler
  val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
  _schedulerBackend = sched
  _taskScheduler = ts
  _dagScheduler = new DAGScheduler(this)

  // 启动 _taskScheduler(TaskSchedulerImpl),同时启动_schedulerBackend(CoarseGrainedSchedulerBackend)
  _taskScheduler.start()
  ...

```

1、SparkSubmit
* 封装参数
* 准备部署环境
* 利用反射执行部署类（Client）

2、YarnClient
* 封装参数
* 创建YarnClient
* 封装启动AM

3、ApplicationMaster
* 封装参数
* 启动Driver线程,执行用户程序(runDriver)
* 向RM注册并申请资源
* 获取资源后,启动ExecutorBackend

4、ExecutorBackend
* start方法:向Driver注册
* receive方法:创建Executor对象


![spark on yarn](https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=1495056033,1745168161&fm=15&gp=0.jpg)



## 重要组件
### BlockManager
TODO:

### Spark Listener

* 初始化`SparkContext`的时候初始化`LiveListenerBus`
* 一个`LiveListenerBus`包含多个**queue**（`AsyncEventQueue`）
* 后面会调用 *setupAndStartListenerBus* 方法，启动**queue**,并把**event**放到**queue**中
* `AsyncEventQueue.dispatch`启动后，循环调用*postToAll*方法,把阻塞队列里的**event**发送给**listener**

![listenerBus工作原理.png](http://ww1.sinaimg.cn/large/b3b57085gy1gkzzhgzk2xj21dc0mwguf.jpg)
### SparkContext

```scala
...
// 初始化listenerBus
_listenerBus = new LiveListenerBus(_conf)
...
setupAndStartListenerBus()
...

  private def setupAndStartListenerBus(): Unit = {
    try {
      conf.get(EXTRA_LISTENERS).foreach { classNames =>
        val listeners = Utils.loadExtensions(classOf[SparkListenerInterface], classNames, conf)
        listeners.foreach { listener =>
          listenerBus.addToSharedQueue(listener)
          logInfo(s"Registered listener ${listener.getClass().getName()}")
        }
      }
    } catch {
      case e: Exception =>
        try {
          stop()
        } finally {
          throw new SparkException(s"Exception when registering SparkListener", e)
        }
    }
    // 启动listenerBus
    listenerBus.start(this, _env.metricsSystem)
    _listenerBusStarted = true
  }

```

### LiveListenerBus

```scala
...
private val queues = new CopyOnWriteArrayList[AsyncEventQueue]()
// 缓冲队列
@volatile private[scheduler] var queuedEvents = new mutable.ListBuffer[SparkListenerEvent]()
...

  def start(sc: SparkContext, metricsSystem: MetricsSystem): Unit = synchronized {
    if (!started.compareAndSet(false, true)) {
      throw new IllegalStateException("LiveListenerBus already started.")
    }

    this.sparkContext = sc
    queues.asScala.foreach { q =>
      // 启动
      q.start(sc)
      // 把缓冲队列里的event刷到AsyncEventQueue中
      queuedEvents.foreach(q.post)
    }
    queuedEvents = null
    metricsSystem.registerSource(metrics)
  }
```

### AsyncEventQueue

```scala
  private def dispatch(): Unit = LiveListenerBus.withinListenerThread.withValue(true) {
    var next: SparkListenerEvent = eventQueue.take()
    while (next != POISON_PILL) {
      val ctx = processingTime.time()
      try {
        // 调用父类（ListenerBus）方法由Listener消费
        super.postToAll(next)
      } finally {
        ctx.stop()
      }
      eventCount.decrementAndGet()
      next = eventQueue.take()
    }
    eventCount.decrementAndGet()
  }
```

## Spark 内存
> https://mp.weixin.qq.com/s/xTWAPtgUc5hZMxxKitD-MQ


