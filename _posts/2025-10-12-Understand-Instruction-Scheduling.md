---
layout: post
title: Understand Instruction Scheduling
tags: ["指令调度"]
mathjax: true
---

# 1 Instruction Scheduling
这篇文档描述指令调度解决的问题以及基本算法和工程实践。
# 2 背景
处理器（CPU）引入pipeline技术，将指令的执行分为多个阶段（stage）, 在一条流水线中并行执行多条指令。
TODO
流水线技术引入了一些问题，
1.	数据依赖 两条相邻的指令无法完美并行执行 
2.	结构依赖 多条指令使用相同的单元在同一时刻
3.	控制依赖 分支语句有可能会导致流水线flush
> 1.2 导致流水线空泡(stall)， 3导致流水线flush

为了提高流水线并行度，必须减少空泡的产生，进而提升吞吐量。

对于1，2.编译器可以将有依赖的指令的顺序错开。

这就是编译器指令调度要做的事情，也称为静态指令调度，因为发生在编译期，而非运行期（动态指令调度，由硬件完成。）
# 3 数据依赖
+ RAR read after read 非依赖
+ RAW read after write 真依赖 先写后读
+ WAR write after read  反依赖 先读后写 
+ WAW write after write 输出依赖 先写后写
TODO
# 4 List Scheduling
> 背景 TranslateUnit=>SCG=>Function=>Loop=>BasicBlock

现代编译器一般都是在BB上调度。

例子
 

TODO

算法
TODO

第一个for:
•  依照 优先级降序 遍历 ready-list。
•  如果有空闲功能单元（FU）可以让这个操作执行，就安排它：
•	从 ready-list 中移除
•	放入 inflight-list
•	标记它在当前 cycle 被调度
•	如果 op 有 anti-dependence 边，并且其目标操作已经满足依赖，就放入 ready-list

第二个for:
当前 cycle 结束后，检查哪些操作已经执行完成：
•	从 inflight-list 中移除
•	检查有没有等待这个操作的下游节点，如果它们所有依赖都满足，就加入 ready-list










总结
指令调度对于in-order处理器非常重要不可缺少，且列表调度算法在in-order CPU中工作的相当不错。

限制：
1）	工作在BB上，不过目前LLVM中对aarch64已经开始探索global schedling实现，还在进程中。
2）	在OoO CPU上，主要依赖动态调度，但是列表调度提供的调度顺序能减轻CPU负担，起不可缺少的辅助作用（必不可少，任意顺序影响硬件性能），一主一辅。


































5 LLVM指令调度
Llvm中有多种调度算法，工作在不同的时机以及不同的数据结构上，侧重不同的任务。

 TODO


这三次指令调度的算法基本都与列表调度形变神似。

核心是列表调度。
三次是在不同的时机做的指令调度，每次都不相同，任务侧重也不同，并非所有架构都会启用全部。

第一次，	侧重从图转为线性序列。
第二次，	RA前，没有寄存器限制，可以自由调度指令。
第三次，	RA后，存在寄存器限制， 可以再次优化。












SchedleDAGSDNodes


 


GODO












ScheduleDAGFast
 





TODO
















机器模型
用编译器内部的一种语言，描述CPU的机器模型，也就是为CPU pipeline建立软件模型。
这些指令以及pipeline的细节会在调度时，选择指令时提供依据。
 

TODO




















# 参考
+ https://www.zhihu.com/collection/787009369
+ Instruction Scheduler in LLVM（LLVM DevMeeting）
+ L18-Instruction-Scheduling-pre-class（CMU编译器课件）

