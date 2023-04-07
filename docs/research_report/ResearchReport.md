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
2. Spark计算模型框架(by yzx)
3. 分布式文件系统调研(by lml)

### 分布式计算框架发展概述

<!-- 偏批处理mapreduce，spark等 -->



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
shuffle是spark及其他分布式计算框架最核心的问题之一，为了提高shuffle的效率，spark也做了很多迭代更新，如将shuffle机制更新为sorted-bashed shuffle

同时，其计算运行在JVM上也对它的性能有较大影响。[^5]
因此，所有处理的数据都是以对象Object的形式存在的。对JVM来说，Object都具有两个特点：（1）大小。内存膨胀的问题是大数据处理中一个典型的问题，参考“A Bloat-aware Design for Big Data Application”(ISMM2013)。对象形式会引入许多无关的引用、锁结构、描述符等，导致其内存中的大小相比于对象本身所携带的Value要大得多。例如，一个int值只占4个字节，但是装箱成一个Integer对象，远远不止4个字节了。（2）生命周期。JVM有自己的垃圾回收机制，根据对象的生命周期来决定是否需要做垃圾回收。任何对象都有自己的生命周期。由于Spark本身支持cache数据到内存，所以JVM中会有cache的Object。再看shuffle，shuffle 
write和shuffle 
read阶段需要用buffer保存所有处理的中间结果（ExternalSorter），然后再写入磁盘，因此Shuffle 
buffer中也包含了非常多的Object。无论是Cache的Object还是Shuffle Buffer中的Object，它们的生命周期都比较长。当对象数量增加时，有限的内存空间就会因为这些长生命周期的大对象显得非常有压力，最直接的影响就是频繁的触发JVM的垃圾回收机制，Full GC本身就会导致大量开销，频繁的触发Full GC会导致Spark性能急剧下降。这是所有自动内存管理机制都会面临的一个问题，提高了开发效率却面临着大数据处理时的低效内存管理。



### rust语言的优势

<!-- rust语言的优势 -->



## 立项依据

<!-- 给出思路，给出需求分析，并说明该思路切合需求 -->
1. Spark与MapReduce对比(by yzx)
2. Rust与Scala语言比较（建议用表格,by xhy&lml）

## 前瞻性/重要性分析

<!-- 立足于趋势和现实回答本项目的价值 -->

rust有诸多优点，且逐渐流行，用rust优化spark性能瓶颈符合这一发展趋势
spark得到重要且广泛的应用，提高其性能将惠及海量用户

## 相关工作

<!-- 对学术界和工业界的经典工作的前沿进展分类梳理，进一步提供各种轮子的信息 -->




## 参考资料

[^1]:Zaharia, Matei, et al. “Spark: Cluster Computing With Working Sets.” IEEE International Conference on Cloud Computing Technology and Science, June 2010, p. 10. www2.eecs.berkeley.edu/Pubs/TechRpts/2010/EECS-2010-53.pdf.
[^2]:Chambers, Bill, and Matei Zaharia. Spark: The Definitive Guide. 2018.




[^5]:张雄. “常见的Spark的性能瓶颈有哪些？.” 知乎, https://www.zhihu.com/question/28023548/answer/138249813.
[^6]:一块小蛋糕. “超全spark性能优化总结.” 知乎专栏, https://zhuanlan.zhihu.com/p/108454557.






