## 调度

调度相关的程序位于`spark-3.2.3/core/src/main/scala/org/apache/spark/scheduler/`目录下。

### DAG调度的过程

我们首先给出一个宏观的说法，其中的不同的名称会在后文进行解释。总的来说，调度由`DAGScheduler`控制，其通过RDD算子构建`DAG`，再基于RDD算子之间的依赖来切分所涉算子，最终得到一些`Stage`对象。每个`Stage`再基于`Partitioner`生成多个`Task`，每个`Stage`中的`Task`集合包装成一个`TaskSet`，生成一个`TaskSetManager`。这些`TaskSetManager`与其他的`Pool`被嵌套地放在`Pool`中，进行宏观的任务调度。[^spark]

具体来说，`DAGScheduler`会为每个`Job`计算一个有向无环图，追踪哪些RDD和`Stage`输出可以被实现，找到运行最短的方式。之后，将`Stage`打包成`Taskset`提交给`TaskScheduler`。一个`TaskSet`中只包含可以在某个节点上独立运行的`Task`。

`Stage`根据RDD图上的shuffle边界分割而成。像`map()`和`filter()`这样的有着“窄”依赖的RDD操作，会被分到单个`Stage`中，但有着shuffle依赖的操作就需要被分到不同`Stage`上面。最终得到的每个`Stage`只与其他`Stage`有着依赖，而其内部的`Task`间不存在依赖，保证可以时运行。对这些任务进行的分割操作发生在`RDD.compute()`函数上。

除了产生`Stage`间的有向无环图，`DAGScheduler`还根据当前缓存状态决定在哪里运行哪个任务。之后其把任务交给低一级的`TaskScheduler`。在输出文件丢失时，它还要做错误处理，即重新提交之前的`Stage`。在`Stage`内部的错误会被交给`TaskScheduler`处理，在取消整个`Stage`之前会多次重试每一个Task。

为了从失败中恢复，相同的`Stage`可能需要运行多次.如果`TaskScheduler` 报告一个Task失败，因为来自前一个`Stage`的map输出文件已丢失，需要由`DAGScheduler`重新提交丢失的`Stage`. 这些通过CompletionEvent 伴随 FetchFailed, 或者ExecutorLost 事件被检测到. The `DAGScheduler` 会等待一定时间来看是否有其余的节点或 `Task` 也失败，然后为计算错误的`Task`所在的`Stage`重新提交。作为这一过程的一部分，我们还需要为旧的`Stage`重新创建`Stage`对象，这些`Stage`可能已经被清空。需要注意保证不同的任务正确地处在对应的`Stage`中。

此外，还有缓存追踪机制以及清空机制。`DAGScheduler`判断哪个RDD被缓存，记忆哪些shuffle map 的`Stage`已经产生输出文件以避免重新计算或重复运行shuffle的map阶段。当相关的`Task`结束时，`DAGScheduler`会清空`Stage`，以避免内存泄露。

`sumbitJob`

`runJob`

`submitStage`如果当前`Stage`没有

### 调度算法

在`SchedulingAlgorithm.scala`中描述，只有两种继承算法`FIFOSchedulingAlgorithm`和`FairSchedulingAlgorithm`，其中的方法`comparator`，返回`boolean`值，用于在调度时进行排序。

以下以`comparator(A,B)`为例.
- FIFO方法，先比较优先级，小的优先，如果同级，则比较提交时间(通过`StageID`判断，小的代表提交时间早)，早的优先。可以判断，`priority`越小，优先级越高。
- Fair方法，如果`A`满足`runningTasks`比`minShare`小，而`B`不满足，则先处理`A`，反之亦然。如果都满足，则比较`runningTasks/minShare`的比值，低的优先。如果都不满足，则比较`runningTasks/weight`的比值，低的优先。当这些比值相同时，比较`name`。总的来说，即资源使用率低的优先。

### 调度任务类


`Job`类是提交给调度器的最高层的工作对象。当用户启动一个操作，比如`count()`时，一个`Job`会通过`submitJob`被提交。每个`Job`中间层的数据需要多个`Stage`的执行来得到。

`Stage`类是`Task`的集合，用于计算`Job`中间层的结果。每个`Task`在相同RDD的分割下计算相同的函数。`Stage`在shuffle边界上被分开。有两种`Stage`：用于最终执行的`ResultStage`，以及为一次shuffle写map输出文件的`ShuffleMapStage`。当多个任务共用相同的RDD时，`Stage`常被在这些任务间共享。

`Task`类，任务本身，包含了任务的一些信息，如`TaskId`，`index`，`attemptNumber`等。其由`executor`执行。每个`Task`会在单个机器上运行。

`TaskSet`类，任务集，是`Task`的集合，存放真正需要执行的任务。

`ShuffleMapTask`和`ResultTask`分别继承了`Task`类，对应之前的`ShuffleMapStage`和`ResultStage`。
- `ShuffleMapTask`是shuffle map任务，其`partitionId`是该任务所在的分区，`mapId`是该任务的map id，`mapIndex`是该任务的map index，`mapStatus`是该任务的map状态。
- `ResultTask`是结果任务，其`partitionId`是该任务所在的分区，`resultId`是该任务的结果id，`resultIndex`是该任务的结果index，`resultStatus`是该任务的结果状态。

`Schedulable`类，其中一些成员变量比较重要，列举如下：[^schedulable]
- `weight`，用于控制不同该类之间的权重，初始为1，而如果一个类有双倍的权重，则会获得双倍的资源。当设置很高的权重时，无论是否有活动，都会优先获得资源。
- `minShare`，用于控制最少分配的资源(比如CPU核数)，公平调度器倾向于满足所有活动类的最小要求。默认为0。
- `runningTasks`，当前在运行的任务数量。

`Pool`和`TaskSetManager`分别继承了`Schedulable`类。
- `TaskSetManager`是任务集管理器，其`Tasks`存放了任务
- `Pool`是调度池，其中存在`schedulableQueue`存放调度队列，这是一个嵌套结构，即调度池里可能同时存在子调度池和任务集管理器。举例如下图![Pool](./src/pool.png)
调度池中的一些方法均是递归式的操作，如果是`Pool`类，则继续递归，如果是`TaskSetManager`则调用对应的操作方法

## 向源码添加内容时的注意事项

`DAGScheduler`源码中的注释提示如下：
 - 当有关的job结束之后，所有的数据结构应该被清空，以避免在长时间运行程序中的状态无限增加。
 - 添加新数据结构时，更新`DAGSchedulerSuite.assertDataStructuresEmpty`函数，这有助于找到内存泄露。


[^schedulable]: Apache. Job Scheduling. Apache Spark Documents. [EB/OL]. [2023-04-20]. https://spark.apache.org/docs/latest/job-scheduling.html

[^100]:https://zhuanlan.zhihu.com/p/163067566

[^spark]: IWBS. Spark. CSDN. [EB/OL]. [2023-04-20]. https://blog.csdn.net/asd491310/category_7797537.html