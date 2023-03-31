# MAGE: Nearly Zero-Cost Virtual Memory for Secure Computation

MAGE: Memory-Aware Garbling Engine

garbled circuit: 加密电路

## 什么是Secure Computation

SC主要是在加密数据上进行计算的。SC的一大障碍是加密本身导致数据大小的成倍增加，这导致进行运算时内存的超荷。

## MAGE的原理

因为安全的需要，SC的方法本身就以下性质——内存访问模式与输入数据是无关的。如下是一个非常简单的小例子

```C
uint a, b, c, branch, cond;
// 内存访问与输入数据有关
if (cond) 
    a = b;
else
    a = c;
// 内存访问与输入数据无关
branch = 0 - !!cond
a = ((b ^ c) & branch) ^ c;
```

利用这一性质，可以提前计算皓内存访问模式，然后生成内存管理方案。这种方法称为*内存编程*，是一种综合的方法，可以对SC提供高效的虚拟内存抽象。其比操作系统本身的虚拟内存方案可以快上一个数量级，也在很多情况下可以像运行在一台*没有物理内存限制*的机器上那么快。

OS page replacement policy

不能通过内存访问模式泄露关于数据的任何信息

使得可以提前计算 相比传统的OS分页模式更有效

Belady's theoretically optimal paging algorithm

主动更换分页 预读取，async eviction

DSL

# maphea

已有的是profile-guided optimization (PGO) frameworks
mainly focus on optimizing the layout of binaries; they overlook rich information provided by the PMU about data ccess behaviors over the memory hierarchy.

MaPHeA guides and applies the optimized allocation of dynamically allocated heap objects with very low profiling overhead and without additional user intervention to improve application performance

# workload-aware

1. Heterogeneity of Hadoop cluster where nodes have different processing speed.
2. Hadoop cluster shared between Interactive and Batch type jobs.
3. The phenomena where application requests certain Data Blocks more often then others.

1. In first phase,historical data access information is collected from the log file of Namenode to determine the frequently accessed set of data blocks,Task arrival rate in a cluster and to individual nodes requesting frequent data blocks and Task processing rate of nodes.
2. In Second phase,percentage of frequent data blocks allocation to nodes and classification of frequent data blocks based on their requesting jobs are determined.
3. In third phase,the data placement algorithm distributes the frequent data blocks.
