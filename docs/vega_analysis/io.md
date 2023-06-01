通过广度优先搜索的方式总结了一下io方面需要实现的类（即vega还没有的）
有些类应该是从Hadoop那边继承过来的，所以描述中只针对map-reduce任务。

## HadoopRDD
HadoopRDD类是用于从Hadoop中读取数据的RDD，继承RDD类。在vega中的实现预计将采用简化版本，即只支持HDFS（或者应该改名叫HDFSRDD？）

它需要的变量有：
```scala
sc: SparkContext,
broadcastedConf: Broadcast[SerializableConfiguration],
initLocalJobConfFuncOpt: Option[JobConf => Unit],
inputFormatClass: Class[_ <: InputFormat[K, V]],
keyClass: Class[K],
valueClass: Class[V],
minPartitions: Int
```

依赖的其他类有：
HadoopPartition，HadoopMapPartitionsWithSplitRDD（在HadoopRDD文件中实现）
SparkHadoopUtil
RecordReader

## HadoopPartition
HadoopPartition是用来给Hadoop中文件分区的类，继承Partition。依赖InputSplit类。
一个HadoopPartition对象就是一个分区。

## Partition
是一个trait，里面包含该分区的index等。

## InputSplit
是一个java实现的接口，用于将输入数据划分为多个逻辑片段，从而并行地在MapReduce程序中处理。
一个InputSplit对象就是一个片段（应该吧）

## Broadcast
一个抽象类。一个Broadcast对象可以将一个（通常是比较大的）只读数据集广播到所有机器上，让所有机器都缓存这一变量，而不是在执行任务时再通过网络复制。

## JobConf
一个java类，继承Configuration。用来描述一个map-reduce job

## Configuration
gpt说是一个java的类？

## InputFormat
用来描述一个map-reduce job的输入格式。
这个类在Spark中有很多子类，定义了一些常用的输入格式

## SparkHadoopUtil
一些Spark和Hadoop交互的实用小方法。在Spark中有一系列这样的util，不同的是SparkHadoopUtil不在util路径下。

## RecordReader
用来把数据弄成键值对，一般和InputFormat成对出现。