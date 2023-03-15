## MemLiner: Lining up Tracing and Application for a Far-Memory-Friendly Runtime
- 关键词：远程存储,GC（垃圾回收）,JDK,
- 作者在论文中为了解决当前GC存在的两大问题（资源竞争，低效预取）提出了新的运行时技术(MemLiner)可以结合当前已有的GC技术，将回收效率提高1.5~2倍。
- 需要注意该技术被希望运用在远程的分布式存储系统上以减少远程管理存储的开销，同时实验主要在服务器上运行，硬件要求较高（CPU E5-2640，128GB mem, 1024GB SSD,connected by RDMA over 40Gbps InfiniBand network）,综合考虑该论文属于JVM GC调优的范畴，可实操性不明确。
