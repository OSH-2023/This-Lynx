# 有关分布式系统概况的调研
# 提要
这里主要总结了三大类分布式系统的特征、研究方向与基本算法。

# 详细笔记

# 分布式存储系统
**四个子方向：**
### 结构化存储
强调：
1. 结构化的数据（如关系模型）
2. 强一致性
3. 随机访问
传统的结构化存储常用于单机，可拓展性不佳。如，MySQL
### 非结构化存储
强调：可拓展性（->吞吐率好）
不支持随机访问，只支持追加(append)操作（->难以面对低延时、实时性强的应用）
不支持关系模型
如，分布式文件系统（GFS/HDFS：实现了自动容错、错误恢复）
### 半结构化存储
综合可拓展性与随机访问能力
如，NoSQL
底层数据结构采用LSM-Tree/B-Tree/B+Tree
### In-memory存储
将数据存储在内存中，从而获得读写的高性能
用于处理高并发、低延时的任务
算法及技术：
Paxos,CAP,ConsistentHsh,Timing,2PC,3PC...

# 分布式计算系统
分布式计算与并行计算的区别：
并行计算：数据大小不变，用更多的机器，计算速度更快(high performance)
分布式计算：用更多的机器，处理更大的数据(scalability)
分布式计算的核心：容错
应用：机器学习
**5个类别：**
### 传统基于msg的系统
MPI系统框架：
提供消息传递接口(send,recv接口)、实现资源管理和分配，以及调度的功能
另一个常用接口(AllReduce)，可以实现各机器的数据同步。
优点：简单高效。（高效：因为底层消息传递使用了tree aggregation）
缺点：不支持容错（最近有新的可支持容错的类allreduce接口）

### MapReduce-like系统
也叫dataflow系统，以MapReduce(Hadoop)和Spark为代表。
特点是将计算抽象成high-level operator，如map,reduce,filter这样的算子，然后将算子组合成DAG，然后由后端的调度引擎进行并行化调度。
具有完备的容错机制，可以扩展到超大规模的集群上运行。
计算过程必须在map和reduce函数中描述；输入和输出数据都是一个个records；任务间不能通信，唯一通信机会是map与reduce之间的shuffuling phase。
不支持大规模机器学习应用，且节点同步效率低。
### 图计算系统
将计算过程抽象为图，然后在不同节点分布式执行，适用于PageRank等任务。
常采用BSP、GPS、GAS等计算框架。
如深度学习中的多层结构等问题不能很好地求解。
### 基于状态(state)的系统
例：distbelief,Parameter Server架构
将机器学习的模型存储上升为主要组件，采用异步机制提升处理能力。
### Streaming系统
为流式数据提供服务，如Flink

# 分布式管理系统
待补充

# 参考文献
[1]https://www.zhihu.com/question/23645117/answer/124708083?utm_campaign=&utm_medium=social&utm_oi=1069723187641839616&utm_psn=1618662362256424960&utm_source=qq
