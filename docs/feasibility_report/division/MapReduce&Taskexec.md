# 算子功能及任务提交/执行
## MapPartitionsRDD
类签名:
``` java
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev) 
```
参数:
* @param prev the parent RDD.
* @param f The function used to map a tuple of (TaskContext, partition index, input iterator) to
*          an output iterator.
* @param preservesPartitioning Whether the input function preserves the partitioner, which should
*                              be `false` unless `prev` is a pair RDD and the input function
*                              doesn't modify the keys.
* @param isFromBarrier Indicates whether this RDD is transformed from an RDDBarrier, a stage
*                      containing at least one RDDBarrier shall be turned into a barrier stage.
* @param isOrderSensitive whether or not the function is order-sensitive. If it's order
*                         sensitive, it may return totally different result when the input order
*                         is changed. Mostly stateful functions are order-sensitive.


方法:
1. `getPartitions`->Array[Partition]
2. `compute(Partition,TaskContext)`->Iterator[U]
3. `clearDependencies`->Unit

## AppendOnlyMap

类型签名:
`class AppendOnlyMap[K, V](initialCapacity: Int = 64) extends Iterable[(K, V)] with Serializable `

功能介绍:
本质上是一种简单的哈希表，对于append-only的情况进行了优化，也就是说keys不会被移除，但是每一种Key的value可能发生改变。
这个实现使用了平方探测法，哈希表的大小是2^n，保证对于每一个key都能浏览所有的空间。
hash的函数使用了Murmur3_32函数（外部库）
空间上界:`375809638 (0.7 * 2 ^ 29)` elements.

成员变量功能:
- LOAD_FACTOR: 负载因子，常量值=0.7
- initialCapacity: 初始容量值64
- capacity: 容量，初始时=initialCapacity
- curSize: 记录当前已经放入data的key与聚合值的数量
- data: 数组，初始大小为2*capacity,data数组的实际大小之所以是capacity的2倍是因为key和聚合值各占一位
- growThreshhold:data数组容量增加的阈值$growThreshold=LOAD_FACTOR*capacity$
- mask: 计算数据存放位置的掩码值，表达式为capacity-1
- k: 要放入data的key
- pos: k将要放入data的索引值
- curKey: data[2*pos]位置的当前key
- newValue: key的聚合值

## ExternalSorter
参数:
- aggregator 可选的聚合器，带有combine function
- partitioner 可选的划分，partition ID用于排序,然后是key
- ordering 在partition内部使用的排序顺序
重要成员:
1. blockManager
2. spills
主要方法:
1. `spillMemoryIteratorToDisk(WriteablePartitionedIterator[K,C])->SpilledFile`将溢出的内存里的迭代器对应的内容放到临时磁盘中
2. `insertAll(Iterator[Product2[K,V]])->Unit`利用自定义的AppendOnlyMap将records进行更新（缓存聚合）
3. 
``` java
    mergeWithAggregation(
      iterators: Seq[Iterator[Product2[K, C]]],
      mergeCombiners: (C, C) => C,
      comparator: Comparator[K],
      totalOrder: Boolean)
      ->Iterator[Product2[K, C]]
```
    将一系列(K,C)迭代器按照key进行聚合，假定每一个迭代器都已经按照key使用给定的比较器排序好了。

## RDD dependency
基类`abstract class Dependency[T] extends Serializable`

1. `abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T]`
/**
* Get the parent partitions for a child partition.
* @param partitionId a partition of the child RDD
* @return the partitions of the parent RDD that the child partition depends upon
*/

依赖关系的基础类型，子RDD的每一个partition只依赖一小部分数量的父RDD，窄依赖允许流水线/管道式的执行.

2. `class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag]`
/**
 * @param _rdd the parent RDD
 * @param partitioner partitioner used to partition the shuffle output
 * @param serializer [[org.apache.spark.serializer.Serializer Serializer]] to use. If not set
 *                   explicitly then the default serializer, as specified by `spark.serializer`
 *                   config option, will be used.
 * @param keyOrdering key ordering for RDD's shuffles
 * @param aggregator map/reduce-side aggregator for RDD's shuffle
 * @param mapSideCombine whether to perform partial aggregation (also known as map-side combine)
 * @param shuffleWriterProcessor the processor to control the write behavior in ShuffleMapTask
 */
代表在shuffle stage的输出上的依赖，由于shuffle是临时的，这个RDD也是临时的因为我们不需要在executor上挂载这个依赖。
3. `class OneToOneDependency[T](rdd: RDD[T]) extends NarrowDependency[T](rdd)`

代表一对一的依赖，是一种在父RDD与子RDD的partition之间的关系
4. `class RangeDependency[T](rdd: RDD[T], inStart: Int, outStart: Int, length: Int) extends NarrowDependency[T](rdd)`
/**
 * @param rdd the parent RDD
 * @param inStart the start of the range in the parent RDD
 * @param outStart the start of the range in the child RDD
 * @param length the length of the range
 */
代表一对一的依赖，但是是在一系列的partitions在父RDD和子RDD之间