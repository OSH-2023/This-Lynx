---
presentation:
  width: 1600
  height: 1200
---

<!-- slide -->

# 基于Rust语言对Apache Spark性能瓶颈的优化

## This-Lynx

> 闫泽轩 李牧龙 罗浩铭 汤皓宇 徐航宇

<!-- slide -->
# 目录
## 1 项目介绍
## 2 立项依据
## 3 具体改进
## 4 测试结果
## 5 项目总结

<!-- slide -->
## 1 项目介绍

<!-- slide vertical=true -->

## 基于Rust版Spark开源项目vega

<!-- slide -->
## 2 背景和立项依据

<!-- slide vertical=true -->

## Rust语言

- ### 安全性
- ### 高性能
- ### 并发性

<!-- slide vertical=true -->

## 分布式计算框架

- ### Apache Hadoop
- ### Apache Spark
- ### Ray
- ### . . .

<!-- slide vertical=true -->

## Spark和RDD

<img src="../midterm_report/src/Spark2.png" style="zoom:150%">

<!-- slide -->
## 3 具体改进

<!-- slide -->
## Shuffle优化

<!-- slide vertical=true -->
## Shuffle介绍
Shuffle：将输入的M个分区内的数据“按一定规则”重新分配到R个分区上。
- 各个节点上相同key的内容写入主节点磁盘文件中
- 相同key的数据将被拉取到同一个分区进行聚合操作

<!-- slide vertical=true -->
## Shuffle介绍
在Spark程序中，Shuffle过程涉及大量的磁盘文件读写操作，以及数据的网络传输操作，因而是性能的最大瓶颈。Shuffle阶段的设计优劣是决定性能好坏的关键因素。

<!-- slide vertical=true -->
## SortShuffleManager
- 对每一对Map和Reduce端的分区配对都产生一条分区记录
- 由于生成的文件数过多，会对文件系统造成压力，且大量小文件的随机读写会带来一定的磁盘开销，故其性能不佳。
而Vega中已将Shuffle记录保存在以DashMap(分布式HashMap)实现的缓存里，这大幅降低了本地I/O开销，但远程开销仍然较大，且DashMap占用空间与性能也会受到索引条数过多的影响。


## SortShuffleManager
Spark自1.1.0版本起默认采用的是更先进的SortShuffle。数据会根据目标的分区Id（即带Shuffle过程的目标RDD中各个分区的Id值）进行排序，然后写入一个单独的Map端输出文件中，而非很多个小文件。输出文件中按reduce端的分区号来索引文件中的不同shuffle部分。这种shuffle方式大幅减小了随机访存的开销与文件系统的压力，不过增加了排序的开销。（Spark起初不采用SortShuffle的原因正是为了避免产生不必要的排序开销）

在我们对Vega中shuffle逻辑的优化中，由于使用了DashMap缓存来保存Shuffle记录，我们无需进行排序，直接按reduce端分区号作为键值写入缓存即可。这既避免了排序的开销，又获得了SortShuffle合并shuffle记录以减少shuffle记录条数的效果。这样，shuffle输出只需以reduce端分区号为键值读出即可。

对shuffle部分，以两千万条shuffle记录的载量（Map端有M个分区，Reduce端有R个分区，`M*R=20000000`）进行单元测试，测试结果如下：
优化前：9.73,10.96,10.32 平均：10.34s
优化后：6.82,5.46,4.87 平均：5.72s
测得运行速度提升了81%，由此说明我们对这一模块的优化是成功的。

<!-- slide -->
## 实现容错

<!-- slide -->
## 加入HDFS文件系统

<!-- slide -->
## 添加实时监控拓展模块

<!-- slide vertical=true -->

## 监控工具

- prometheus
- grafana
- node_exporter

<!-- slide vertical=true -->

## 效果展示

<img src="./src/local_running.png" style="zoom:150%">

<!-- slide -->
## 自动化测试

<!-- slide -->
## 4 测试结果

<!-- slide -->
## 5 项目总结

<!-- slide vertical=true -->
## 组员分工与进度管理

- 闫泽轩（组长）：
- 李牧龙：
- 罗浩铭：
- 汤皓宇：对vega进行Docker部署，添加性能拓展模块，配置docker下的prometheus+grafana+node_exporter来展示vega运行时各机器的CPU使用率和vega的运行情况
- 徐航宇：

<!-- slide vertical=true -->
## 项目意义与前瞻性

- ### Spark本身有很广的应用
- ### 使用Rust进行计算框架撰写有很大好处

<!-- slide -->
## 未来可优化方向

- ### 减少序列化开销
- ### 构建更友好的API