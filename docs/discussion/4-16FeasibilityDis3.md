# 4-16 可行性报告讨论

可行性分析，包括理论依据、技术依据、创新点和概要设计报告

## 对于scala与rust调用的问题

已知rust可以调用C语言，但是否可以在rust中调用scala的函数？

搜集了一些资料，需要之后进行整理。

初步讨论需要实现`scala<rust<scala<rust<scala`的调用。

## 其余讨论

分两步，用rust改写部分瓶颈相关的代码，然后将其与scala进行调用。

## benchmark

可以考虑的方向：文件读写等