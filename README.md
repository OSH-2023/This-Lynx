# This-Lynx

OSH-2023 team project repo, our chinese team name is "芝士猞猁".

```
 ________ __       __          __                                  
|        \  \     |  \        |  \                                 
 \▓▓▓▓▓▓▓▓ ▓▓____  \▓▓ _______| ▓▓      __    __ _______  __    __ 
   | ▓▓  | ▓▓    \|  \/       \ ▓▓     |  \  |  \       \|  \  /  \
   | ▓▓  | ▓▓▓▓▓▓▓\ ▓▓  ▓▓▓▓▓▓▓ ▓▓     | ▓▓  | ▓▓ ▓▓▓▓▓▓▓\\▓▓\/  ▓▓
   | ▓▓  | ▓▓  | ▓▓ ▓▓\▓▓    \| ▓▓     | ▓▓  | ▓▓ ▓▓  | ▓▓ >▓▓  ▓▓ 
   | ▓▓  | ▓▓  | ▓▓ ▓▓_\▓▓▓▓▓▓\ ▓▓_____| ▓▓__/ ▓▓ ▓▓  | ▓▓/  ▓▓▓▓\ 
   | ▓▓  | ▓▓  | ▓▓ ▓▓       ▓▓ ▓▓     \\▓▓    ▓▓ ▓▓  | ▓▓  ▓▓ \▓▓\
    \▓▓   \▓▓   \▓▓\▓▓\▓▓▓▓▓▓▓ \▓▓▓▓▓▓▓▓_\▓▓▓▓▓▓▓\▓▓   \▓▓\▓▓   \▓▓
                                       |  \__| ▓▓                  
                                        \▓▓    ▓▓                  
                                         \▓▓▓▓▓▓                   
```

## Topic

基于Rust语言对Apache Spark性能瓶颈的优化

## Members

- [闫泽轩](https://github.com/yuriYanZeXuan)
- [李牧龙](https://github.com/NanqiOP)
- [罗浩铭](https://github.com/4332001876)
- [汤皓宇](https://github.com/himalalps)
- [徐航宇](https://github.com/XhyDds)

## Overview

本项目基于印度开发者rajasekarv开发的vega项目，使用Rust语言（nightly）重构分布式计算框架Spark，实现其核心功能并进行性能上的优化。
原作者的项目已停止更新2年，项目中存在大量未完成部分，且已无法在当前版本的Rust nightly下通过编译。本小组对该项目进行了维护，修正了部分代码，提升了容错性和运行效率，增加了与Hdfs、Prometheus、Grafana等的接口。目前vega已可以完成基本的分布式计算操作，并在许多任务上实现了相对Spark的较大性能提升。

## Discussion Progress

| 日期  |          主题          |                                  记录                                  |     备注     |
| :---: | :--------------------: | :--------------------------------------------------------------------: | :----------: |
| 3-11  |        首次讨论        |              [3-11First](./docs/discussion/3-11First.md)               |              |
| 3-18  |      调研结果讨论      |                [3-18Pre](./docs/discussion/3-18Pre.md)                 |              |
| 3-26  |        正式立项        |        [3-26FormalTopic](./docs/discussion/3-26FormalTopic.md)         |              |
|  4-2  |      选题最终方案      |          [4-2TopicFinal](./docs/discussion/4-2TopicFinal.md)           | 老师参会讨论 |
|  4-9  |     可行性方案分工     | [4-9FeasibilityDivision](./docs/discussion/4-9FeasibilityDivision.md)  |              |
| 4-13  |     可行性报告讨论     | [4-13FeasibilityDiscussion](./docs/discussion/4-13FeasibilityDis2.md)  |              |
| 4-16  |     可行性报告讨论     | [4-16FeasibilityDiscussion2](./docs/discussion/4-16FeasibilityDis3.md) |              |
| 4-22  |     可行性报告讨论     | [4-22FeasibilityDiscussion3](./docs/discussion/4-22FeasibilityDis4.md) |              |
|  5-5  |        中期汇报        |        [5-5MidtermDiscussion](./docs/discussion/5-5Midterm.md)         |              |
|  5-7  |        尝试vega        |             [5-7TryVega](./docs/discussion/5-7TryVega.md)              |              |
| 5-11  |      定位vega分工      |         [5-11LocateVega](./docs/discussion/5-11LocateVega.md)          |              |
| 5-14  |        讨论尝试        |         [5-14VegaDis](./docs/discussion/5-14VegaDiscussion.md)         |              |
| 5-18  | 阅读代码，尝试双机连接 |                                                                        |              |
| 5-21  | 阅读代码，尝试多机连接 |                                                                        |              |
| 5-28  |        阅读代码        |                                                                        |              |
|  6-2  |     讨论Lab4的分工     |                [6-2Lab4](./docs/discussion/6-2Lab4.md)                 |              |
|  6-8  |   完成Lab4的剩余部分   |                [6-8Lab4](./docs/discussion/6-8Lab4.md)                 |              |
| 6-11  |    讨论接下来的工作    |                [6-11Job](./docs/discussion/6-11Job.md)                 |              |
| 7-4   |  考试周后继续完善项目  |                   [7-4](./docs/discussion/7-4.md)                       |              |
| 7-5   |      继续完善项目      |                   [7-5](./docs/discussion/7-5.md)                       |              |
| 7-6   |   基本完成大部分工作   |                   [7-6](./docs/discussion/7-6.md)                       |              |

## Research Progress

小组在调研阶段广泛查询各种资料，多方收集信息，确定了包括优化Spark在内的多个选题。在与老师充分沟通过后，考虑小组成员的爱好，最终确定了用Rust语言优化Spark的选题。

关于项目的具体实现，也有2个方向：一是在Spark的基础上用Rust重构部分代码；二是基于项目vega进行改进（事实上，vega正是我们选题的灵感来源）。在调研阶段，这两个方向都困难重重。将Rust加入Spark中需要面临Scala语言和Rust互相调用的难题，由于Scala是运行在JVM上的，要和Rust相互调用显然非常困难。而vega已多年未更新，甚至无法通过编译，在调研初期小组成员Rust运用尚不熟练的情况下，无法知晓该项目的实际运行效率与完成情况。最终本小组在讨论后选择了后者。

## Basis of Project

Spark采用在JVM上运行的Scala语言编写。虽然带有运行时环境具有方便内存管理等优势，但同时也会拖累其性能。Rust作为一门系统编程语言，具有极高的运行效率（接近C++）的同时还可以保证内存安全性。非常适合用来优化Spark。

另外，由于是在vega的基础上进行优化，而vega在调度、容错和接口等方面均存在明显不足，可以通过增添组件和合理修改原有的代码对其进行改进。