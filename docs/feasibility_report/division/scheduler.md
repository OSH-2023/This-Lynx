# 调度

调度相关的程序位于`spark-3.2.3/core/src/main/scala/org/apache/spark/scheduler/`目录下

## 调度任务类

`Schedulable`类，其中一些成员变量比较重要，列举如下：[^1]
- `weight`，用于控制不同该类之间的权重，初始为1，而如果一个类有双倍的权重，则会获得双倍的资源。当设置很高的权重时，无论是否有活动，都会优先获得资源。
- `minShare`，用于控制最少分配的资源(比如CPU核数)，公平调度器倾向于满足所有活动类的最小要求。默认为0。
- `runningTasks`，当前在运行的任务数量。

## 调度算法

在`SchedulingAlgorithm.scala`中描述，只有两种继承算法`FIFOSchedulingAlgorithm`和`FairSchedulingAlgorithm`，其中的方法`comparator`，返回`boolean`值，用于在调度时进行排序。

以下以`comparator(A,B)`为例.
- FIFO方法，先比较优先级，小的优先，如果同级，则比较提交时间(通过`stageID`判断，小的代表提交时间早)，早的优先。可以判断，`priority`越小，优先级越高。
- Fair方法，如果`A`满足`runningTasks`比`minShare`小，而`B`不满足，则先处理`A`，反之亦然。如果都满足，则比较`runningTasks/minShare`的比值，低的优先。如果都不满足，则比较`runningTasks/weight`的比值，低的优先。当这些比值相同时，比较`name`。总的来说，即资源使用率低的优先。

## DAG调度

源码注释：

DAG调度是实现阶段

The high-level scheduling layer that implements stage-oriented scheduling. It computes a DAG of stages for each job, keeps track of which RDDs and stage outputs are materialized, and finds a minimal schedule to run the job. It then submits stages as TaskSets to an underlying TaskScheduler implementation that runs them on the cluster. A TaskSet contains fully independent tasks that can run right away based on the data that's already on the cluster (e.g. map output files from previous stages), though it may fail if this data becomes unavailable.

Spark stages are created by breaking the RDD graph at shuffle boundaries. RDD operations with "narrow" dependencies, like `map()` and `filter()`, are pipelined together into one set of tasks in each stage, but operations with shuffle dependencies require multiple stages (one to write a set of map output files, and another to read those files after a barrier). In the end, every stage will have only shuffle dependencies on other stages, and may compute multiple operations inside it. The actual pipelining of these operations happens in the `RDD.compute()` functions of
various RDDs

In addition to coming up with a DAG of stages, the DAGScheduler also determines the preferred locations to run each task on, based on the current cache status, and passes these to the low-level TaskScheduler. Furthermore, it handles failures due to shuffle output files being lost, in which case old stages may need to be resubmitted. Failure *within* a stage that are not caused by shuffle file loss are handled by the `TaskScheduler`, which will retry each task a small number of times before cancelling the whole stage.

When looking through this code, there are several key concepts:
 - Jobs (represented by [[ActiveJob]]) are the top-level work items submitted to the scheduler. For example, when the user calls an action, like `count()`, a job will be submitted through submitJob. Each Job may require the execution of multiple stages to build intermediate data.
 - Stages ([[Stage]]) are sets of tasks that compute intermediate results in jobs, where each task computes the same function on partitions of the same RDD. Stages are separated at shuffle boundaries, which introduce a barrier (where we must wait for the previous stage to finish to fetch outputs). There are two types of stages: [[ResultStage]], for the final stage that executes an action, and [[ShuffleMapStage]], which writes map output files for a shuffle. Stages are often shared across multiple jobs, if these jobs reuse the same RDDs.
 - Tasks are individual units of work, each sent to one machine.
 - Cache tracking: the `DAGScheduler` figures out which RDDs are cached to avoid recomputing them and likewise remembers which shuffle map stages have already produced output files to avoid redoing the map side of a shuffle.
 - Preferred locations: the DAGScheduler also computes where to run each task in a stage based on the preferred locations of its underlying RDDs, or the location of cached or shuffle data.
 - Cleanup: all data structures are cleared when the running jobs that depend on them finish, to prevent memory leaks in a long-running application.

To recover from failures, the same stage might need to run multiple times, which are called "attempts". If the TaskScheduler reports that a task failed because a map output file from a previous stage was lost, the DAGScheduler resubmits that lost stage. This is detected through a CompletionEvent with FetchFailed, or an ExecutorLost event. The DAGScheduler will wait a small amount of time to see whether other nodes or tasks fail, then resubmit TaskSets for any lost stage(s) that compute the missing tasks. As part of this process, we might also have to create Stage objects for old (finished) stages where we previously cleaned up the Stage object. Since tasks from the old attempt of a stage could still be running, care must be taken to map any events received in the correct Stage object.

Here's a checklist to use when making or reviewing changes to this class:
 - All data structures should be cleared when the jobs involving them end to avoid indefinite accumulation of state in long-running programs.
 - When adding a new data structure, update `DAGSchedulerSuite.assertDataStructuresEmpty` to include the new structure. This will help to catch memory leaks.


[^1]:https://spark.apache.org/docs/latest/job-scheduling.html

[^100]:https://zhuanlan.zhihu.com/p/163067566