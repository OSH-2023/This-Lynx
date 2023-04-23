# 可行性报告

## 小组成员
闫泽轩 李牧龙 罗浩铭 汤皓宇 徐航宇

## 引言

## 理论依据
### 源码基础上对spark性能瓶颈的rust重写
基于spark源码的改进是其中一种方案。在源代码上进行修改，特别是用rust重写其中的性能瓶颈模块，特别是内存密集型、CPU密集型的部分，同时保留源码的非关键部分，特别是spark中原生的丰富API，由此达到以小范围的修改达到较为显著的性能提升的效果。
在Spark的Scala源码与rust代码间的交互是这一方案中需要特别关注的点，主要是由于scala基于JVM，与非JVM语言的Rust之间，有较大的交互困难。我们将使用Scala的JNI(Java Native Interface)与Rust进行交互，我们这里使用jni crate，来实现两种语言间的交互。

### 基于不完善的rust版spark开源项目vega的实现
由于rust语言的诸多优点，用rust重写spark是一个非常有诱惑力的方案。此前，就已经有一个较为粗浅的基于rust的spark项目：vega（[Github仓库](https://github.com/rajasekarv/vega)）。这一项目完全使用rust从零写起，构建完成了一个较为简单的spark内核。不过，这一项目已经有两三年没有维护，项目里还有不少算法没有实现，特别是Spark后来的诸多优化更新，这些都是我们的改进空间。


## 技术依据
### JNI交互
Scala是在JVM上运行的语言，和Java比较相似，二者可以无缝衔接。在与其他语言交互时，主要有JNI(Java Native Interface), JNA(Java Native Access), OpenJDK project Panama三种方式。其中最常用的即为JNI接口。借由JNI，Scala可以与Java代码无缝衔接，而Java可以与C也通过JNI来交互。而Rust可通过二进制接口的方式与其他语言进行交互，特别是可以通过Rust的extern语法，十分方便地与C语言代码交互，按照C的方式调用JNI。这一套机制的组合之下，Scala和Rust的各类交互得到了保障。
同时，正如我们通常不会直接在Rust中通过二进制接口调用C的标准库函数，而是使用libc crate一样，直接使用JNI对C的接口会使得编程较为繁琐且不够安全，代码中的大量unsafe块使得程序稳定性大大下降，所以，我们将选择对JNI进行了安全的封装的接口：**jni[^1] crate**。
#### Rust调用Scala
**数据交互**
两种语言在进行交互时，必须使用两边共有的数据类型。
对于基础的参数类型，可以直接用`jni::sys::*`模块提供的系列类型来声明，对照表如下：
| Scala 类型 | Native 类型 | 类型描述         |
| ---------- | ----------- | ---------------- |
| boolean    | jboolean    | unsigned 8 bits  |
| byte       | jbyte       | signed 8 bits    |
| char       | jchar       | unsigned 16 bits |
| short      | jshort      | signed 16 bits   |
| int        | jint        | signed 32 bits   |
| long       | jlong       | signed 64 bits   |
| float      | jfloat      | 32 bits          |
| double     | jdouble     | 64 bits          |
| void       | void        | not applicable   |

对于复合类型，如对象等，则可以统一用`jni::objects::JObject`类型声明。该类型封装了由JVM返回的对象指针，并为该指针赋予了生命周期，以保证在Rust代码中的安全性。
**方法交互**
由于语言间对对象及其特性的实现不同，很难直接调用对方语言中的函数或方法。于是通常需要使用server-client模型，将执行函数或方法的任务交给sever语言，即：client传递所需的数据参数，并由server执行计算任务，并将最终结果返回给client。
基于这种模型的设计，jni提供了调用scala中函数、对象方法以及获取对象数据域的方法。它们定义于`jni::JNIEnv`中，如接受对象、方法名和方法的签名与参数的`jni::JNIEnv::call_method`，接受对象、成员名、类型的`jni::JNIEnv::get_field`等
此外，jni额外实现了一个`jni::objects::JString`接口，用以方便地实现字符串的传输。
#### Scala调用Rust
Rust可以通过`pub unsafe extern "C" fn{}`来创建导出函数，或通过jni封装的函数`JNIEnv::register_native_methods`动态注册native方法。
导出函数会通过函数名为Scala的对应类提供一个native的静态方法。
动态注册会在JNI_Onload这个导出函数里执行，jvm加载jni动态库时会执行这个函数，从而加载注册的函数。
在Rust中定义这些函数时，同样需要遵循上面的那些交互方法和规范。
### Cap'n Proto[^1]
Cap'n Proto是一种速度极快的数据交换格式，以及能力强大的RPC系统.
![capnp](./src/Capnp.png)
#### 优势:
1. 递增读取:可以在整个Cap'n Proto 信息完全传递之前进行处理，这是因为外部对象完全出现在内部对象以前。
2. 随机访问:你可以仅读取一条信息的一个字段而无需将整个对象翻译。
3. MMAP:通过memory-mapping读取一个巨型文件，OS甚至不会读取你未访问的部分。
4. 跨语言通讯:避免了从脚本语言调用C++代码的痛苦。使用Cap'n Proto可以让多种语言轻松地在同一个内存中的数据结构上进行操作。
5. 通过共享内存可以让在同一台机器上多线程共同访问。不需要通过内核来产生管道来传输数据。
6. Arena Allocation:Cap'n Proto对象始终以"arena"或者"region"风格进行分配，因此更快并且提升了缓存局部性。
7. 微小生成代码:Protobuf为每个消息类型产生专一的解析和序列化代码，这种代码会变得庞大无比。
### 优化RDD依赖关系

### 调度

在Spark里，与调度相关的程序位于`spark-3.2.3/core/src/main/scala/org/apache/spark/scheduler/`目录下。

#### DAG调度的过程

我们首先给出一个宏观的说法，其中的不同的名称会在后文进行解释。总的来说，调度由`DAGScheduler`控制，其通过RDD算子构建`DAG`，再基于RDD算子之间的依赖来切分所涉算子，最终得到一些`Stage`对象。每个`Stage`再基于`Partitioner`生成多个`Task`，每个`Stage`中的`Task`集合包装成一个`TaskSet`，生成一个`TaskSetManager`。这些`TaskSetManager`与其他的`Pool`被嵌套地放在`Pool`中，进行宏观的任务调度。[^spark]

![submitJob](./src/submitjob.png)

具体来说，`DAGScheduler`会为每个`Job`计算一个有向无环图，追踪哪些RDD和`Stage`输出可以被实现，找到运行最短的方式。之后，将`Stage`打包成`Taskset`提交给`TaskScheduler`。一个`TaskSet`中只包含可以在某个节点上独立运行的`Task`。

`Stage`根据RDD图上的shuffle边界分割而成。像`map()`和`filter()`这样的有着“窄”依赖的RDD操作，会被分到单个`Stage`中，但有着shuffle依赖的操作就需要被分到不同`Stage`上面。最终得到的每个`Stage`只与其他`Stage`有着依赖，而其内部的`Task`间不存在依赖，保证可以时运行。对这些任务进行的分割操作发生在`RDD.compute()`函数上。

除了产生`Stage`间的有向无环图，`DAGScheduler`还根据当前缓存状态决定在哪里运行哪个任务。之后其把任务交给低一级的`TaskScheduler`。在输出文件丢失时，它还要做错误处理，即重新提交之前的`Stage`。在`Stage`内部的错误会被交给`TaskScheduler`处理，在取消整个`Stage`之前会多次重试每一个Task。

为了从失败中恢复，相同的`Stage`可能需要运行多次.如果`TaskScheduler` 报告一个Task失败，因为来自前一个`Stage`的map输出文件已丢失，需要由`DAGScheduler`重新提交丢失的`Stage`. 这些通过CompletionEvent 伴随 FetchFailed, 或者ExecutorLost 事件被检测到. The `DAGScheduler` 会等待一定时间来看是否有其余的节点或 `Task` 也失败，然后为计算错误的`Task`所在的`Stage`重新提交。作为这一过程的一部分，我们还需要为旧的`Stage`重新创建`Stage`对象，这些`Stage`可能已经被清空。需要注意保证不同的任务正确地处在对应的`Stage`中。

此外，还有缓存追踪机制以及清空机制。`DAGScheduler`判断哪个RDD被缓存，记忆哪些shuffle map 的`Stage`已经产生输出文件以避免重新计算或重复运行shuffle的map阶段。当相关的`Task`结束时，`DAGScheduler`会清空`Stage`，以避免内存泄露。

与上有关的函数包括`sumbitJob`,`submitMapStage`,`submitStage`,`submitMissingTasks`,`submitWaitingChildStage`等。

#### 调度算法

在`SchedulingAlgorithm.scala`中描述，只有两种继承算法`FIFOSchedulingAlgorithm`和`FairSchedulingAlgorithm`，其中的方法`comparator`，返回`boolean`值，用于在调度时进行排序。

以下以`comparator(A,B)`为例.
- FIFO方法，先比较优先级，小的优先，如果同级，则比较提交时间(通过`StageID`判断，小的代表提交时间早)，早的优先。可以判断，`priority`越小，优先级越高。
- Fair方法，如果`A`满足`runningTasks`比`minShare`小，而`B`不满足，则先处理`A`，反之亦然。如果都满足，则比较`runningTasks/minShare`的比值，低的优先。如果都不满足，则比较`runningTasks/weight`的比值，低的优先。当这些比值相同时，比较`name`。总的来说，即资源使用率低的优先。

#### 调度任务类

`Job`类是提交给调度器的最高层的工作对象。当用户启动一个操作，比如`count()`时，一个`Job`会通过`submitJob`被提交。每个`Job`中间层的数据需要多个`Stage`的执行来得到。

`Stage`类是`Task`的集合，这些`Task`计算`Job`中间层的结果。每个`Task`在相同RDD的分割下计算相同的函数，因此有着相同的shuffle依赖。`Stage`在shuffle边界上被分开。有两种`Stage`：用于最终执行的`ResultStage`，直接计算一个Spark操作(如`count()`, `save()`)；以及中间层的`ShuffleMapStage`，其结果用于下一个`Stage`的输入。当多个任务共用相同的RDD时，`Stage`常被在这些任务间共享。因此`Stage`间遵循拓扑排序的来依次执行。每个`Stage`有一个成员`firstJobId`，标识首个提交该`Stage`的`Job`。当使用FIFO调度时，这允许先计算或在失败时更快恢复早期`Job`的`Stage`。如果失败，单个`Stage`可以被多次重新执行。在这种情况下，`Stage`对象会跟踪多个`StageInfo`对象，传递给监听器或web UI。最新的`StageInfo`对象可以通过`latestInfo`访问。

`Task`类，任务本身，包含了任务的一些信息，如`TaskId`，`index`，`attemptNumber`等。其由`executor`执行。每个`Task`会在单个机器上运行。

`TaskSet`类，任务集，是`Task`的集合。

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

因此，我们有如下的过程：

```mermaid
graph LR
Job --1 to N --> Stage --1 to N--> Task --N to 1 --> TaskSet --1 to 1 --> TaskSetManager
```

#### 调度部分可以改进的内容

添加调度算法，如调研报告内提到的利用协程方法实现随时将当前在进行处理的批数据暂停，切换到需要低延迟的流数据上去，在处理完流数据之后，再切换回批数据，在保证了流数据的低延迟的同时兼顾批数据的处理。

#### 向调度部分添加内容时的注意事项

`DAGScheduler`源码中的注释提示如下：
 - 当有关的job结束之后，所有的数据结构应该被清空，以避免在长时间运行程序中的状态无限增加。
 - 添加新数据结构时，更新`DAGSchedulerSuite.assertDataStructuresEmpty`函数，这有助于找到内存泄露。


[^schedulable]: Apache. Job Scheduling. Apache Spark Documents. [EB/OL]. [2023-04-20]. https://spark.apache.org/docs/latest/job-scheduling.html

[^100]:https://zhuanlan.zhihu.com/p/163067566

[^spark]: IWBS. Spark. CSDN. [EB/OL]. [2023-04-20]. https://blog.csdn.net/asd491310/category_7797537.html

## 创新点


## 概要设计报告


## 进度管理









