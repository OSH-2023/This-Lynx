# 3-26 正式立项讨论

## 在分布式系统下一些可行方向

1. 基于2004年google提出的Mapreduce上的hadoop和spark
2. paxos算法到raft协议
3. ray 和 ceph
4. [使用rust改进spark框架](https://medium.com/@rajasekar3eg/fastspark-a-new-fast-native-implementation-of-spark-from-scratch-368373a29a5c)
5. GFS 对象存储

## 确定方向

使用已有的分布式文件，任务调度等系统，基于mapreduce对分布式计算框架进行改进。

## 调研报告的分工

已知调研报告需要包括：项目背景、立项依据、前瞻性/重要性分析、相关工作

[ ] 文件系统 - 李牧龙

[ ] 任务调度 - 汤皓宇

[ ] mapreduce原理+RPC - 闫泽轩

[ ] 一致性算法 - 罗浩铭

[ ] 不同分布式计算框架的区别 - 徐航宇

下周4月2日开会