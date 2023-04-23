# SparkStreaming
## 简介：
Spark Streaming是Spark API的一个扩展，它使得Spark可以支持可扩展、高吞吐量、容错的实时数据流处理。数据的来源可以是多种多样的（如Kafka, Kinesis和TCP sockets），数据可以使用通过map，reduce，join和window等高级函数描述的算法处理。结果可以输出到文件系统、数据库等。

### 架构：
Spark Streaming采用微批次的方式实现，即将流式计算当作一系列小规模的批处理来执行。
Spark Streaming提供了表示连续数据流的，高度抽象的Dstream（discretized stream），Dstream可以由数据源产生的输入数据流创建，也可以由其它Dstream使用map，reduce等操作创建。Dstream本质上是RDD序列。

### 运行过程：
1. 初始化Streaming Context对象。在该对象的启动过程中实例化DStreamGraph和JobScheduler。DStreamGraph中包含DStream和它们之间的依赖关系；JobScheduler中包含ReceiverTracker和JobGenerator实例。
2. ReceiverTracker启动时会通知ReceiverSupervisor启动，ReceiverSupervisor会启动流数据接收器Receiver。Receiver会不断接收实时流数据，交给ReceiverSupervisor存储为blocks，存储完毕后ReceiverSupervisor会将元数据发给ReceiverTracker，ReceiverTracker再将数据转发给ReceiverBlockTracker，由它管理元信息。
3. JobGenerator中维护一个定时器，它在批处理时间到来时会生成作业，作业执行过程如下：
   (1)通知ReceiverTracker将接收到的数据进行提交，在提交时采用synchronized关键字进行处理，保证每条数据被划入一个且只被划入一个批次中。
   (2)要求DStreamGraph根据DStream依赖关系生成作业序列Seq[Job]。
   (3)从第一步中ReceiverTracker获取本批次数据的元数据。
   (4)把批处理时间time、作业序列Seq[Job]和本批次数据的元数据包装为JobSet，调用JobScheduler.submitJobSet(JobSet)提交给JobScheduler，JobScheduler将把这些作业发送给Spark核心进行处理，由于该执行为异步，因此本步执行速度将非常快。
   (5)只要提交结束（不管作业是否被执行），SparkStreaming对整个系统做一个检查点（Checkpoint）。
4. Spark核心处理作业队数据，处理完毕输出到外部系统。

## 源码阅读：
Streaming部分的源码全部在streaming文件夹下，核心部分位于/streaming/src/main/scala/org/apache/spark/streaming。由于streaming只是spark的一个扩展模块，本身不承担计算任务，因此代码量较小。

## StreamingContext
StreamingContext类是Spark Streaming的起始点，流式计算的启动和停止都通过它来完成（调用`context.start()` 和 `context.stop()`）。
其形式如下：
```scala
class StreamingContext private[streaming] (
_sc: SparkContext,
_cp: Checkpoint,
_batchDur: Duration
) extends Logging {
   //省略
}
```
其中SparkContext和Duration是必需的，Checkpoint则可以为null。

### 一些重要的变量：
**SparkContext：**
```scala
private[streaming] val sc: SparkContext = {
 if (_sc != null) {
   _sc
 } else if (isCheckpointPresent) {
   SparkContext.getOrCreate(_cp.createSparkConf())
 } else {
   throw new SparkException("Cannot create StreamingContext without a SparkContext")
 }
}
```
SparkContext是Spark的上下文。它可以由直接传参/传入参数（如SparkConf）构造得到，也可以从检查点取得或创建。

**DStreamGraph：**
```scala
private[streaming] val graph: DStreamGraph = {
 if (isCheckpointPresent) {
   _cp.graph.setContext(this)
   _cp.graph.restoreCheckpointData()
   _cp.graph
 } else {
   require(_batchDur != null, "Batch duration for StreamingContext cannot be null")
   val newGraph = new DStreamGraph()
   newGraph.setBatchDuration(_batchDur)
   newGraph
 }
}
```
DStreamGraph用来管理DStream和它们之间的依赖。它可由检查点取得，也可传入Duration创建
（这里为什么新建的graph没有绑定context？）

**JobScheduler：**
```scala
private[streaming] val scheduler = new JobScheduler(this)
```
JobScheduler用于调度在Spark上运行的任务。创建时需要绑定StreamingContext。

**StreamingSource：**
```scala
private val streamingSource = new StreamingSource(this)
```
暂时不知道干什么用的

### 重要的方法：
**设置检查点：**
```scala
def checkpoint(directory: String): Unit = {
 if (directory != null) {
   val path = new Path(directory)
   val fs = path.getFileSystem(sparkContext.hadoopConfiguration)
   fs.mkdirs(path)
   val fullPath = fs.getFileStatus(path).getPath().toString
   sc.setCheckpointDir(fullPath)
   checkpointDir = fullPath
 } else {
   checkpointDir = null
 }
}
```
初始化检查点的路径。

**启动stream的处理：**
```scala
def start(): Unit = synchronized {
 state match {
   case INITIALIZED =>
     startSite.set(DStream.getCreationSite())
     StreamingContext.ACTIVATION_LOCK.synchronized {
       StreamingContext.assertNoOtherContextIsActive()
       try {
         validate()

         registerProgressListener()

         // Start the streaming scheduler in a new thread, so that thread local properties
         // like call sites and job groups can be reset without affecting those of the
         // current thread.
         ThreadUtils.runInNewThread("streaming-start") {
           sparkContext.setCallSite(startSite.get)
           sparkContext.clearJobGroup()
           sparkContext.setLocalProperty(SparkContext.SPARK_JOB_INTERRUPT_ON_CANCEL, "false")
           savedProperties.set(Utils.cloneProperties(sparkContext.localProperties.get()))
           scheduler.start()
         }
         state = StreamingContextState.ACTIVE
         scheduler.listenerBus.post(
           StreamingListenerStreamingStarted(System.currentTimeMillis()))
       } catch {
         case NonFatal(e) =>
           logError("Error starting the context, marking it as stopped", e)
           scheduler.stop(false)
           state = StreamingContextState.STOPPED
           throw e
       }
       StreamingContext.setActiveContext(this)
     }
     logDebug("Adding shutdown hook") // force eager creation of logger
     shutdownHookRef = ShutdownHookManager.addShutdownHook(
       StreamingContext.SHUTDOWN_HOOK_PRIORITY)(() => stopOnShutdown())
     // Registering Streaming Metrics at the start of the StreamingContext
     assert(env.metricsSystem != null)
     env.metricsSystem.registerSource(streamingSource)
     uiTab.foreach(_.attach())
     logInfo("StreamingContext started")
   case ACTIVE =>
     logWarning("StreamingContext has already been started")
   case STOPPED =>
     throw new IllegalStateException("StreamingContext has already been stopped")
 }
}
```
只有已被初始化，且未被启动或停止的StreamingContext才能被启动。
start()方法中的关键部分在于：
1. 设置startSite：
`startSite.set(DStream.getCreationSite())`
2. 注册ProgressListener，用于监听所有Streaming Job的进度：
`registerProgressListener()`
3. **启动JobScheduler：**
```scala
ThreadUtils.runInNewThread("streaming-start") {
           sparkContext.setCallSite(startSite.get)
           sparkContext.clearJobGroup()
           sparkContext.setLocalProperty(SparkContext.SPARK_JOB_INTERRUPT_ON_CANCEL, "false")
           savedProperties.set(Utils.cloneProperties(sparkContext.localProperties.get()))
           scheduler.start()
         }
```

**停止处理：**
```scala
def stop(stopSparkContext: Boolean, stopGracefully: Boolean): Unit = {
 var shutdownHookRefToRemove: AnyRef = null
 if (LiveListenerBus.withinListenerThread.value) {
   throw new SparkException(s"Cannot stop StreamingContext within listener bus thread.")
 }
 synchronized {
   // The state should always be Stopped after calling `stop()`, even if we haven't started yet
   state match {
     case INITIALIZED =>
       logWarning("StreamingContext has not been started yet")
       state = STOPPED
     case STOPPED =>
       logWarning("StreamingContext has already been stopped")
       state = STOPPED
     case ACTIVE =>
       // It's important that we don't set state = STOPPED until the very end of this case,
       // since we need to ensure that we're still able to call `stop()` to recover from
       // a partially-stopped StreamingContext which resulted from this `stop()` call being
       // interrupted. See SPARK-12001 for more details. Because the body of this case can be
       // executed twice in the case of a partial stop, all methods called here need to be
       // idempotent.
       Utils.tryLogNonFatalError {
         scheduler.stop(stopGracefully)
       }
       // Removing the streamingSource to de-register the metrics on stop()
       Utils.tryLogNonFatalError {
         env.metricsSystem.removeSource(streamingSource)
       }
       Utils.tryLogNonFatalError {
         uiTab.foreach(_.detach())
       }
       Utils.tryLogNonFatalError {
         unregisterProgressListener()
       }
       StreamingContext.setActiveContext(null)
       Utils.tryLogNonFatalError {
         waiter.notifyStop()
       }
       if (shutdownHookRef != null) {
         shutdownHookRefToRemove = shutdownHookRef
         shutdownHookRef = null
       }
       logInfo("StreamingContext stopped successfully")
       state = STOPPED
   }
 }
 if (shutdownHookRefToRemove != null) {
   ShutdownHookManager.removeShutdownHook(shutdownHookRefToRemove)
 }
 // Even if we have already stopped, we still need to attempt to stop the SparkContext because
 // a user might stop(stopSparkContext = false) and then call stop(stopSparkContext = true).
 if (stopSparkContext) sc.stop()
}
```
与`start()`不同，还未被启动的StreamingContext也可以被停止。
若StreamingContext处于活动状态，则`stop()`方法将`start()`方法中启动的各种操作依次停止（如注销progressListener，停止scheduler等），并根据传入的参数额外执行停止SparkContext/等待所有接收到的数据处理完成等。
`stop()`在最后才将StreamingContext的状态标为`STOPPED`


## DStream
DStream意为离散化数据流，是用于封装流式数据的数据结构，其内部由一系列连续的RDDs构成，每个RDD代表特定时间间隔内的一批数据。

### 重要变量：
**dependencies：**
```scala
def dependencies: List[DStream[_]]
```
是该DStream依赖的父DStream列表。

**generatedRDDs：**
```scala
private[streaming] var generatedRDDs = new HashMap[Time, RDD[T]]()
```
已经产生的RDD，用HashMap存储。

**zeroTime：**
```scala
private[streaming] var zeroTime: Time = null
```
DStream的时间零点，在初始化时进行设置，用来标识该DStream是否被初始化，以及判断后续传入的时间参数是否合法。



## DStreamGraph
包含DStream和它们之间的依赖关系

### 重要变量：
**inputStreams：**
```scala
private var inputStreams = mutable.ArraySeq.empty[InputDStream[_]]
```
输入数据源的集合

**outputStreams：**
```scala
private var outputStreams = mutable.ArraySeq.empty[DStream[_]]
```
DStream的集合

### 重要方法：
**start()**
```scala
def start(time: Time): Unit = {
 this.synchronized {
   require(zeroTime == null, "DStream graph computation already started")
   zeroTime = time
   startTime = time
   outputStreams.foreach(_.initialize(zeroTime))
   outputStreams.foreach(_.remember(rememberDuration))
   outputStreams.foreach(_.validateAtStart())
   numReceivers = inputStreams.count(_.isInstanceOf[ReceiverInputDStream[_]])
   inputStreamNameAndID = inputStreams.map(is => (is.name, is.id)).toSeq
   new ParVector(inputStreams.toVector).foreach(_.start())
 }
}
```
用于启动DStreamGraph。该方法中设置zeroTime和startTime，初始化各个outputStream，计算Receiver的数量，记录inputStream，最后启动各个inputStream。

**stop()**
```scala
def stop(): Unit = {
 this.synchronized {
   new ParVector(inputStreams.toVector).foreach(_.stop())
 }
}
```
停止每个inputStream。

**generateJobs(time: Time)**
```scala
def generateJobs(time: Time): Seq[Job] = {
 logDebug("Generating jobs for time " + time)
 val jobs = this.synchronized {
   outputStreams.flatMap { outputStream =>
     val jobOption = outputStream.generateJob(time)
     jobOption.foreach(_.setCallSite(outputStream.creationSite))
     jobOption
   }.toSeq
 }
 logDebug("Generated " + jobs.length + " jobs for time " + time)
 jobs
}
```
对outputStreams中的每一个DStream调用generateJob方法，最后返回一个已调度的Job序列

