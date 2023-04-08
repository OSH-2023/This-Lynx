# 调研报告
**注：相关调研内容要用与主题相关的语言叙述，不要大段copy，介绍背景知识时若与主题相关性低请使用引用，不要硬凑字数**
## 选题:基于Rust语言对Apache Spark性能瓶颈的优化

## 小组成员

闫泽轩 李牧龙 罗浩铭 汤皓宇 徐航宇

## 项目背景

<!-- 给出该领域面临的问题，并给出该领域的设计思路，给出领域中已有的轮子  -->
<!-- 领域：我们这里就是指spark性能瓶颈优化（解决的问题是spark性能瓶颈，用的方法是rust）-->
<!-- 面临问题：spark性能瓶颈    设计思路与已有轮子：调研清楚spark的更新历程-->
1. Rust语言介绍(by xhy)
3. 分布式文件系统调研(by lml)

### 分布式计算框架发展概述

<!-- 偏批处理mapreduce，spark等 -->
## MapReduce
![MapReduceImg](../investigation/src/MapReduceOverview.png)
- 介绍:
  - MapReduce是一种编程模型和一种产生及处理大数据的实现方式。他的关键在于两个函数（由用户编写）：Map和Reduce
- Types:
  - map  (k1,v1)        ----> list(k2,v2)
  - reduce(k2,list(v2)) --->list(v2)
- Example(伪代码):
  
```C++
  map(String key, String value):
    // key: document name
    // value: document contents
    for each word w in value:
    EmitIntermediate(w, "1");


  reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
    result += ParseInt(v);
    Emit(AsString(result));
```
- 执行过程:

1. MapReduce库先将输入文件分成M份（一般每份64MB，也可用可选参数控制），然后他在集群上启动多份程序
2. 有一份特殊的程序拷贝--master，剩下的都是worker并且被master分配任务.共有M个map任务和R份Reduce任务去分配。master选择空闲的worker去分配map task或者reduce task
3. 获得map任务的worker从对应的输入文件中读取内容。他从中解析出键值对并且将每一对传给Map函数。这些中间键值对被缓存在存储器中
4. 周期性的，分区R份后，这些缓冲的中间键值文件的**位置**被传输给master,master负责把这些信息发给reduce worker
5. 当reduce worker接受到来自master的中间键值文件的位置后，它就使用RPC从map worker的本地磁盘中读取缓存数据，并且根据key进行排序。这样所有出现的相同的key就能被合并到一个组。这个排序是必要的因为通常不同的键会被映射到同一个reduce task中。如果中间键值数据过于庞大的话，则应该使用外部排序
6. reduce worker不断的在排序好的中间键值数据上进行迭代，对于遇到的特定的中间键值对，就将键和值集合传入reduce函数中，函数的输出结果就将加载到最终的输出文件中去（对于这个reduce部分）
7. 当所有的map和reduce人物都被完成后，master就唤醒用户程序，这时MapReduce的调用完成，继续返回到用户的代码中

- 容错性
**Worker Failure**
master会周期性的测试每一个worker，以此来判定是否执行失败，如果map失败就重新安排worker执行。这是因为由于ma过程的结果存储在本地，如果失败就无法取得结果。但是已经完成的Reduce工作不需要回滚，因为其结果存储在全局文件系统中。
并且如果map失败，比如A失败后任务被B重新执行，那么还未读取A的reduce task就会切换到来自于B的数据输出。
**Master Failure**
由于Master只有一个节点，因此失败的可能性很低，如果失败就重新运行整个MapReduce


#### Spark及其发展历史[^2]
Apache Spark起源于2009年在加州大学伯克利分校AMPlab的Spark研究项目，Matei Zahari等人于次年发表了其论文，名为“Spark: Cluster Computing with Working Sets”。

当时，Hadoop Map-Reduce是集群的主要并行计算引擎，是第一个在数千个节点规模的集群上进行并行数据处理的开源系统。它展现出了集群计算的巨大潜力：在使用MapReduce的每个组织机构中，都可以使用现有数据构建全新的应用程序，并且在尝试其入门用例之后，许多新的团队开始使用该系统。但是它对一些问题的处理也相当低效。例如，典型的机器学习算法可能需要对数据进行10或20次迭代处理，而在MapReduce中，每次迭代都必须编写为单独的MapReduce作业，并且每次启动时都要在集群上从头开始加载数据。

为了解决这个问题，Spark团队首先设计了一组基于函数式编程的API，可以简洁地表达多步计算应用程序。然后，团队在一个新引擎上实现了这个API，该引擎可以跨多个计算步骤执行高效的内存数据共享。团队还开始与校内校外其他用户一起测试该系统。

Spark的第一个版本仅支持批处理应用程序，但很快又出现了另一类重要应用：交互式数据处理和即席（ad-hoc）查询。通过简单地将Scala解释器插入Spark中，该项目可以提供一个高可用的交互式系统，用于在数百台机器上运行查询。AMPlab还迅速建立了Shark，这是一个可以在Spark上运行SQL查询并支持交互式使用的引擎，其于2011年首次发布。

在这些初始版本之后，发布者很快认识到，对Spark最强大的功能将来自于新的软件库，因此该项目开始遵循今天的“标准库”方法。特别地，AMPlab团队中不同的研究组开始开发MLlib（机器学习库）、Spark Streaming（流处理库）和GraphX（图分析库）。他们还确保这些API具有高度的互操作性，使得人们首次可以在同一引擎中编写多种端到端的大数据应用程序。

2013年，该项目已经得到广泛使用，来自伯克利分校以外的30多个组织共100多位贡献者参与其中。AMPlab将Spark作为长期的、非商业的项目贡献给Apache软件基金会。早期的AMPlab团队还成立了一家公司Databricks，以加强该项目，并加入了其他为Spark做出贡献公司和组织。自那时起，Apache Spark社区在2014年发布了Spark 1.0，在2016年发布了Spark 2.0，并继续定期发布，将新功能带入该项目。

最后，Spark可组合API的核心思想也不断完善。Spark的早期版本（1.0之前）很大程度上只定义了API的功能操作（例如对Java对象集合的map和reduce等并行操作）。从1.0开始，该项目添加了Spark SQL，这是一种用于处理结构化数据（具有固定数据格式，不与Java的内存表示形式绑定的表）的新API。此后，该项目继续添加了大量针对结构化数据的新API，包括DataFrames、机器学习管道和Structured Streaming（这是一个高级的、自动优化的流处理API）。


<!-- 偏流处理flink，storm等 -->

### spark框架的瓶颈

<!-- 讲述为什么选择spark -->
1. shuffle是spark及其他分布式计算框架最核心的问题之一，为了提高shuffle的效率，spark也做了很多迭代更新，如将shuffle机制更新为sorted-bashed shuffle

2. 同时，其计算运行在JVM上也对它的性能有较大影响。[^5]因此，所有处理的数据都是以对象Object的形式存在的。对JVM来说，Object都具有两个特点：

    （1）大小。内存膨胀的问题是大数据处理中一个典型的问题，参考“A Bloat-aware Design for Big Data Application”(ISMM2013)。对象形式会引入许多无关的引用、锁结构、描述符等，导致其内存中的大小相比于对象本身所携带的Value要大得多。例如，一个int值只占4个字节，但是装箱成一个Integer对象，远远不止4个字节了。
    （2）生命周期。JVM有自己的垃圾回收机制，根据对象的生命周期来决定是否需要做垃圾回收。任何对象都有自己的生命周期。由于Spark本身支持cache数据到内存，所以JVM中会有cache的Object。再看shuffle，shuffle 

3. write和shuffle 
read阶段需要用buffer保存所有处理的中间结果（ExternalSorter），然后再写入磁盘，因此Shuffle buffer中也包含了非常多的Object。无论是Cache的Object还是Shuffle Buffer中的Object，它们的生命周期都比较长。当对象数量增加时，有限的内存空间就会因为这些长生命周期的大对象显得非常有压力，最直接的影响就是频繁的触发JVM的垃圾回收机制，Full GC本身就会导致大量开销，频繁的触发Full GC会导致Spark性能急剧下降。这是所有自动内存管理机制都会面临的一个问题，提高了开发效率却面临着大数据处理时的低效内存管理。


3. 利用协程的特点改进spark调度算法，默认情况下，spark使用的是FIFO即先进先出算法，这样如果先到的是批量的数据，自然会阻塞之后到来的数据，造成等待。而一些常用的调度算法如FAIR公平调度算法，有可能降低总的处理时间，但因为其把所有的任务当成一样的，没有考虑到流数据需要的低延迟，可能导致延迟相比最优解要高。而引入协程之后，可以实现随时将当前在进行处理的批数据暂停，切换到需要低延迟的流数据上去，在处理完流数据之后，再切换回来。这样，保证了流数据的低延迟，同时兼顾了批数据的处理。既使总的处理时间最低，又使延迟最低。当然，这只是最简单的表述，实际过程中，需要考虑到有可能有的批数据被多次延后，可以设定阈值，保证其被多次延后时保证可以持续不被打扰。[^11]
[^11]:Panagiotis Garefalakis, Konstantinos Karanasos, and Peter Pietzuch. 2019. Neptune: Scheduling Suspendable Tasks for Unified Stream/Batch Applications.In ACM Symposium on Cloud Computing (SoCC ’19), November 20–23, 2019, Santa Cruz, CA, USA. ACM, New York, NY, USA, 13 pages. https://doi.org/10.1145/3357223.3362724


## 立项依据

<!-- 给出思路，给出需求分析，并说明该思路切合需求 -->
### Spark与MapReduce对比

|         |MapReduce  | Spark |
|---      |-----|-----|
|提出时间  |2004 by Google  |  2011 by UCB|
|数据存储方式|磁盘介质|内存缓存|
|任务级别并行度|多进程模型|多线程模型|
|流数据支持|不支持|部分支持|
|算子|map&reduce|MR的超集，更加丰富|
|容错机制|丢弃，重新执行|checkpointing|
|速度|并行计算，速度一般|是MR的100倍|

### Spark和其他主流流处理框架对比
![image](
https://images2015.cnblogs.com/blog/1004194/201707/1004194-20170705141432487-820336058.png)

### 容错性

以下是流处理框架容错性处理方案：

#### Apache Storm：
Storm使用上游数据备份和消息确认的机制来保障消息在失败之后会重新处理。消息确认原理：每个操作都会把前一次的操作处理消息的确认信息返回。这保障了没有数据丢失，但数据结果会有重复，这就是at-least once传输机制。

#### Storm
采用对每个源数据记录仅仅要求几个字节存储空间来跟踪确认消息。纯数据记录消息确认架构，尽管性能不错，但不能保证exactly once消息传输机制，所有应用开发者需要处理重复数据。Storm存在低吞吐量和流控问题，因为消息确认机制在反压下经常误认为失败。

#### Spark Streaming：
Spark Streaming实现微批处理，容错机制的实现跟Storm不一样的方法。微批处理的想法相当简单。Spark在集群各worker节点上处理micro-batches。每个micro-batches一旦失败，重新计算就行。因为micro-batches本身的不可变性，并且每个micro-batches也会持久化，所以exactly once传输机制很容易实现。

#### Samza：
Samza的实现方法跟前面两种流处理框架完全不一样。Samza利用消息系统Kafka的持久化和偏移量。Samza监控任务的偏移量，当任务处理完消息，相应的偏移量被移除。消息的偏移量会被checkpoint到持久化存储中，并在失败时恢复。

#### Apache Flink：
Flink的容错机制是基于分布式快照实现的，这些快照会保存流处理作业的状态。Flink仍然是原生流处理框架，它与Spark Streaming在概念上就完全不同。Flink也提供exactly once消息传输机制。


### RDD运行流程

RDD在Spark中运行大概分为以下三步：
1. 创建RDD对象
2. DAGScheduler模块介入运算，计算RDD之间的依赖关系，RDD之间的依赖关系就形成了DAG
3. 每一个Job被分为多个Stage。划分Stage的一个主要依据是当前计算因子的输入是否是确定的，如果是则将其分在同一个Stage，避免多个Stage之间的消息传递开销

![image](https://images2015.cnblogs.com/blog/1004194/201608/1004194-20160830112210980-1493673048.png)

- 创建 RDD  上面的例子除去最后一个 collect 是个动作，不会创建 RDD 之外，前面四个转换都会创建出新的 RDD 。因此第一步就是创建好所有 RDD( 内部的五项信息 )？
- 创建执行计划 Spark 会尽可能地管道化，并基于是否要重新组织数据来划分 阶段 (stage) ，例如本例中的 groupBy() 转换就会将整个执行计划划分成两阶段执行。最终会产生一个 DAG(directed acyclic graph ，有向无环图 ) 作为逻辑执行计划

- 调度任务  将各阶段划分成不同的 任务 (task) ，每个任务都是数据和计算的合体。在进行下一阶段前，当前阶段的所有任务都要执行完成。因为下一阶段的第一个转换一定是重新组织数据的，所以必须等当前阶段所有结果数据都计算出来了才能继续

### Rust相较于其他语言的优势（建议用表格,by xhy&lml）


## 前瞻性/重要性分析

<!-- 立足于趋势和现实回答本项目的价值 -->

rust有诸多优点，且逐渐流行，用rust优化spark性能瓶颈符合这一发展趋势
spark得到重要且广泛的应用，提高其性能将惠及海量用户

## 相关工作

<!-- 对学术界和工业界的经典工作的前沿进展分类梳理，进一步提供各种轮子的信息 -->

1. SnappyData[^10]是一个支持流处理分析的统一引擎，其在Apache Spark内部融合了一个混合数据库。为了实现低延迟，他绕过了Spark的调度器，实现为统一的分析工作负载提供高吞吐量、低延迟和高并发性。
2. coroutine 协程，这是一种类似python语言中的generator的概念。本质类似一个状态机，定义下这样的协程之后，每次使用，都进行`yield`一次，得到之后的状态。
3.  Zookeeper, ZooKeeper主要服务于分布式系统，可以用ZooKeeper来做：统一配置管理、统一命名服务、分布式锁、集群管理。

## 参考资料

[^1]:Zaharia, Matei, et al. “Spark: Cluster Computing With Working Sets.” IEEE International Conference on Cloud Computing Technology and Science, June 2010, p. 10. www2.eecs.berkeley.edu/Pubs/TechRpts/2010/EECS-2010-53.pdf.
[^2]:Chambers, Bill, and Matei Zaharia. Spark: The Definitive Guide. 2018.




[^5]:张雄. “常见的Spark的性能瓶颈有哪些？.” 知乎, https://www.zhihu.com/question/28023548/answer/138249813.
[^6]:一块小蛋糕. “超全spark性能优化总结.” 知乎专栏, https://zhuanlan.zhihu.com/p/108454557.


[^10]:Barzan Mozafari, Jags Ramnarayan, Sudhir Menon, Yogesh Mahajan, Soubhik Chakraborty, Hemant Bhanawat, and Kishor Bachhav. SnappyData: A Unified Cluster for Streaming, Transactions and Interactice Analytics. CIDR, 2017. https://github.com/TIBCOSoftware/snappydata




