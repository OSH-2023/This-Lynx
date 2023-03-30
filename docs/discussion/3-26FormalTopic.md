# 3-26 正式立项讨论

## 在分布式系统下一些可行方向

1. 基于2004年google提出的Mapreduce上的hadoop和spark
2. paxos算法到raft协议
3. ray 和 ceph
4. [使用rust改进spark框架](https://medium.com/@rajasekar3eg/fastspark-a-new-fast-native-implementation-of-spark-from-scratch-368373a29a5c)
5. GFS 对象存储

## 确定方向

~~使用已有的分布式文件，任务调度等系统，基于mapreduce对分布式计算框架进行改进。~~（已被否）
### rust重写Spark
- 创新性:原生Spark使用Scala实现，Scala 是一种支持函数式编程的高级编程语言，但是rust作为一种系统编程语言，相较于Scala具有内存安全性的特点，同时运行速度可以与C语言相比，很适合编写高性能，安全的系统及代码。
- 可行性:清华大学操作系统课程用rust实现了教学操作系统rCore,在2022级x-realism小组中实现了rust重写内核的工作，加之rust有活跃的社区以及完整的文档，语言本身可以快速入门。另外Spark作为Apache下的开源顶级项目，可以轻松的查阅源码实现。
- 重要性/前瞻性:目前Rust社区处于快速发展迭代的黄金时期，用其作为工具实现分布式计算系统框架具有乐观的发展前景。
- 工业/学术前沿:
  http%3A//spark.apachecn.org/paper/zh/spark-rdd.html
  
  Spark解决了已有框架,如MapReduce对于迭代式算法场景和交互性数据挖掘场景的处理性能差，RDDs的概念提供了高效的容错方式并且很好的支持了流式数据处理的问题。

## 调研报告的分工

已知调研报告需要包括：项目背景、立项依据、前瞻性/重要性分析、相关工作

[ ] 文件系统 - 李牧龙

[ ] 任务调度 - 汤皓宇

[ ] mapreduce原理+RPC - 闫泽轩

[ ] 一致性算法 - 罗浩铭

[ ] 不同分布式计算框架的区别 - 徐航宇

下周4月2日开会