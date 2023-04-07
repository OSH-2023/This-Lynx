# 调研报告

## 小组成员

闫泽轩 闫泽轩 李牧龙 罗浩铭 汤皓宇 徐航宇

## 项目背景

<!-- 一段文字引入 -->


### 分布式计算框架发展概述

<!-- 偏批处理mapreduce，spark等 -->



#### Spark及其发展历史[^2]
Apache Spark起源于2009年在加州大学伯克利分校的Spark研究项目，该项目在随后的一年由UC Berkeley AMPlab的Matei Zaharia、Mosharaf Chowdhury、Michael Franklin、Scott Shenker和Ion Stoica发表了一篇名为“Spark: Cluster Computing with Working Sets”的论文。当时，Hadoop Map-Reduce是集群的主要并行编程引擎，是第一个开源系统，可以处理成千上万个节点的数据并行处理。AMPlab与多个早期的MapReduce用户合作，了解这种新的编程模型的好处和缺点，因此能够综合几个用例的问题列表，并开始设计更通用的计算平台。此外，Zaharia还与UC Berkeley的Hadoop用户合作，了解他们对平台的需求，尤其是那些使用迭代算法进行大规模机器学习并需要对数据进行多次遍历的团队。

通过这些对话，有两件事是清楚的。首先，集群计算具有巨大的潜力：在使用MapReduce的每个组织中，都可以使用现有数据构建全新的应用程序，并且在其初始用例之后，许多新的团队开始使用该系统。但是，第二个问题是，MapReduce引擎使构建大型应用程序变得具有挑战性和低效。例如，典型的机器学习算法可能需要对数据进行10或20次遍历，在MapReduce中，每个遍历都必须编写为单独的MapReduce作业，并且必须在集群上单独启动并从头开始加载数据。

为了解决这个问题，Spark团队首先设计了基于函数式编程的API，可以简洁地表达多步应用程序。然后，团队在一个新引擎上实现了这个API，该引擎可以在计算步骤之间执行高效的内存数据共享。团队还开始与伯克利和外部用户一起测试该系统。

Spark的第一个版本仅支持批处理应用程序，但很快又出现了另一个引人注目的用例：交互式数据科学和即席查询。通过将Scala解释器插入Spark中，该项目可以提供一个高度可用的交互式系统，用于在数百台机器上运行查询。AMPlab还迅速建立了Shark，这是一个可以在Spark上运行SQL查询并使分析师和数据科学家也能够进行交互使用的引擎。Shark于2011年首次发布。

在这些初始版本之后，很快就变得清楚，对Spark最有用的增强功能将是新库，因此该项目开始遵循今天的“标准库”方法。特别是，不同的AMPlab团队启动了MLlib、Spark Streaming和GraphX。他们还确保这些API可以高度互操作，首次在同一引擎中编写端到端的大数据应用程序。

2013年，该项目已经得到广泛使用，来自30多个组织外的100多位贡献者参与其中。AMPlab将Spark作为长期的、独立于供应商的项目贡献给Apache软件基金会。早期的AMPlab团队还成立了一家公司Databricks，以加强该项目，加入了其他公司和组织为Spark做出贡献的社区。自那时以来，Apache Spark社区在2014年发布了Spark 1.0，在2016年发布了Spark 2.0，并继续定期发布，将新功能带入该项目。

最后，Spark的核心思想是可组合的API也经过了时间的精炼。Spark的早期版本（1.0之前）主要是通过函数操作来定义这个API，例如对Java对象集合的并行操作，例如映射和约简。从1.0开始，该项目添加了Spark SQL，这是一个与结构化数据（具有固定数据格式，不与Java的内存表示形式绑定的表）一起工作的新API。Spark SQL通过更详细地理解数据格式和用户在其上运行的代码，实现了跨库和API的强大新优化。随着时间的推移，该项目添加了大量基于这种更强大的结构化基础的新API，包括DataFrames、机器学习管道和Structured Streaming，这是一个高级的、自动优化的流API。在本书中，我们将花费大量时间解释这些下一代API，其中大部分标记为生产就绪。


<!-- 偏流处理flink，storm等 -->

### spark框架的瓶颈

<!-- 讲述为什么选择spark -->

### rust语言的优势

<!-- rust语言的优势 -->



## 立项依据

<!-- 查找相关论文 -->

## 前瞻性/重要性分析

<!-- 做相关分析 -->

## 相关工作

<!-- 需要填一些内容 -->




## 参考资料

[^1]:Zaharia, Matei, et al. “Spark: Cluster Computing With Working Sets.” IEEE International Conference on Cloud Computing Technology and Science, June 2010, p. 10. www2.eecs.berkeley.edu/Pubs/TechRpts/2010/EECS-2010-53.pdf.
[^2]:Chambers, Bill, and Matei Zaharia. Spark: The Definitive Guide. 2018.








