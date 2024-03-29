## MapReduce的改进方向之一(Spark)
MapReduce是面向磁盘的。因此，受限于磁盘读写性能的约束，MapReduce在处理迭代计算、实时计算、交互式数据查询等方面并不高效。但是，这些计算却在图计算、数据挖掘和机器学习等相关应用领域中非常常见。

而Spark是面向内存的。这使得Spark能够为多个不同数据源的数据提供近乎实时的处理性能，适用于需要多次操作特定数据集的应用场景。

在相同的实验环境下处理相同的数据，若在内存中运行，那么Spark要比MapReduce快100倍。其它方面，例如处理迭代运算、计算数据分析类报表、排序等，Spark都比MapReduce快很多。

Map 步骤是在不同机器上独立且同步运行的，它的主要目的是将数据转换为 key-value 的形式；而 Reduce 步骤是做聚合运算，它也是在不同机器上独立且同步运行的。Map 和 Reduce 中间夹杂着一步数据移动，也就是 shuffle，这步操作会涉及数量巨大的网络传输（network I/O），需要耗费大量的时间。 由于 MapReduce 的框架限制，一个 MapReduce 任务只能包含一次 Map 和一次 Reduce，计算完成之后，MapReduce 会将运算结果写回到磁盘中（更准确地说是分布式存储系统）供下次计算使用。如果所做的运算涉及大量循环，比如估计模型参数的梯度下降或随机梯度下降算法就需要多次循环使用训练数据，那么整个计算过程会不断重复地往磁盘里读写中间结果。这样的读写数据会引起大量的网络传输以及磁盘读写，极其耗时，而且它们都是没什么实际价值的废操作。因为上一次循环的结果会立马被下一次使用，完全没必要将其写入磁盘。整个算法的瓶颈是不必要的数据读写，而Spark 主要改进的就是这一点。具体地，Spark 延续了MapReduce 的设计思路：对数据的计算也分为Map 和Reduce 两类。但不同的是，一个Spark 任务并不止包含一个Map 和一个Reduce，而是由一系列的Map、Reduce构成。这样，计算的中间结果可以高效地转给下一个计算步骤，提高算法性能。虽然Spark 的改进看似很小，但实验结果显示，它的算法性能相比MapReduce 提高了10～100 倍。
raft: https://raft.github.io/

vega(fast spark): https://github.com/rajasekarv/vega

HDFS: 
