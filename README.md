[TOC]


# go1.5 源代码分析

动机两方面：

1. 优化go程序免不了要了解一些实现细节，能看到的资源太零散（大牛可能觉得太简单？）。
2. go代码并不多，go.15 基本go实现，好读。但是和任何的应用代码相比，其各个方面细节都更加的分散，需要综合。一次读明白，记录清楚，一劳永逸。

个人水平有些限，有些方面学校时了解过，基本都忘了，即使当时实际经验很少，理解不够深入，希望不影响整体理解：

* 系统和汇编层面理解有限，这里只关注 linux_amd64。
* 编译器相关, 代码 + 注释 + 反汇编，尽力而为。



内容 完成度(%)：

0. [包依赖关系](deps.md)
1. goroutine
	1. [asm, stack and syscall](g/stack.md)  80%
	2. [stack增长](g/morestack.md) 50%
	2. [goroutine](g/goroutine.md)  80% 
	3. [schedule: overview](g/schedule.md) 80 %
	4. [schedule: sync, chan, timer](g/sync.md) 60%
	5. [schedule: netpoll()](g/netpoll.md) 80% 
	6. [schedule: schedule(),sysmon()](g/schedule1.md) 80%
	4. [startup](g/startup.md) 10%
2. mallocgc  comming
3. 结构  
	4. map
	6. refect
4. cgo
5. 工具 (prof 等)
6. 有用的点



待整理：
	[link](link.md)

附：

1. [读到的源文件说明](runtime_files.md)

