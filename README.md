


# go1.5 源代码分析

## 前言

动机两方面：

1. 优化go程序免不了要了解一些实现细节，能看到的资源太零散（大牛可能觉得太简单？）。
2. go代码并不多，go.15 基本go实现，好读。但是和任何的应用代码相比，其各个方面细节都更加的分散，需要综合。一次读明白，记录清楚，一劳永逸。

一些系统底层，理解不够深入，希望不影响整体理解：

* 系统和汇编层面理解有限，这里只关注 linux_amd64。
* 编译器相关, 结合代码 + 注释 + 反汇编，尽力而为。

很久没写这么多字，可读性差些，完善中。

整理主要出于自学目的，看代码如破案，不断发现和分析线索，独有一番乐趣。有人愿意看当然欢迎，道行和精力都有限，疏漏和不足请看官请不吝赐教。


## 大纲

内容 完成度(%)：

注： 80% 可能就会暂时停下，先求贯通整体。

0. [包依赖关系](deps.md)
1. goroutine
	1. [asm, stack and syscall](g/stack.md)  80%
	2. [stack增长](g/morestack.md) 50%
	3. [goroutine](g/goroutine.md)  80% 
	4. [runtime自己的同步](g/rtlock.md) 80%
	5. [schedule: overview](g/schedule.md) 80 %
	5. [schedule: timer](g/timer.md) 80
	5. [schedule: sync, chan](g/sync.md) 50%
	5. [schedule: netpoll()](g/netpoll.md) 80% 
	5. [schedule: schedule(),sysmon()](g/schedule1.md) 80%
	6. [startup](g/startup.md) 10%
	7. [panic/defer]
	8. [signal]
2. mallocgc 
	1. [escape: stack or heap?](mem/escape.md) 70%
	2. [静态看布局]
	3. [malloc/free]
	4. [gc]
	5. [memprof]
3. 结构  
	4. map
	6. refect
4. cgo
5. 编译器和工具链


## 待整理：

[link](link.md)

## 附：

1. [读到的源文件说明](runtime_files.md)

