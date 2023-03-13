# MAGE: Nearly Zero-Cost Virtual Memory for Secure Computation

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

(未完)