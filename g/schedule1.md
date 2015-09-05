[TOC]

#schedule: 分配和抢占

## schedule()：

P的工作就是：不停的找G，跑之


1. _g_ := getg()
2. 需要trace 一下unblock事件？
3. 跑一下 GCWorker?（前提 gcBlackenEnabled）
4.  globrunqget (schedule每执行61次，跑一下这个)
5. runqget (local)
6. findrunnable() // blocks until work is available
	1. local
	2. global
	3. netpoll(false) // non-blocking
	4. runqsteal //random steal from other P's， 4 x \#P 次
	5. 如果在GC mark phase，参与到并发的mark过程中去
	6. releasep() //  Disassociate p and the current m.
	7. pidleput(_p_) // Put p to on _Pidle list.
	8. 再查一遍
	9. stopm() //Stops execution of the current m until new work is available.
		1. mput()
		2. notesleep
7. execute(gp, inheritTime)


## sysmon

```
	forcegcperiod := int64(2 * 60 * 1e9)
	scavengelimit := int64(5 * 60 * 1e9)
	while True:
		usleep(delay)
		if not polled for more than 10ms:
			gp := netpoll(false) // non-blocking ,返回一个list！
			injectglist(gp) // 全部加入globalrunq，如果有idle的P，startm
		retake(now) // retake P's blocked in syscalls 
		 			// preempt long running G's
		if 超过 forcegcperiod 没有gc:
			injectglist(forcegc.g)
		if lastscavenge+scavengelimit/2 < now：
			mHeap_Scavenge(int32(nscavenge), uint64(now), uint64(scavengelimit)
			
```



* 注意 mHeap_Scavenge只在此处被自动触发
	* 除非user 调用debug.FreeOSMemory())
* 但是malloc有时好像会是否少量span。


#### 概括一下：


1. sysmon run every / 20 us
2. try retake （Gsyscall） / 20(us)
1. netpoll every / 10ms (nonblocking)
2. gc / 2分钟
1. free os mem / 5分钟


### forcegchelper 

从开始一直在跑，虽然名字中有“force”，但是普通的并发 gc(gcBackgroundMode)

```
func init() {
	go forcegchelper()
}
func forcegchelper() {
	startGC(gcBackgroundMode, true)
}
```

##  preemption

retake() 只在sysmon 被调用

```
for each _p_:
	s := _p_.status
	if s == _Psyscall:
		if it's there for more than 1 sysmon tick (at least 20us):
	 		handoffp(_p_)
	elif s == _Prunning:
		if 跑了超过 10 ms：
			preemptone(_p_)
				gp.preempt = true
				gp.stackguard0 = stackPreempt

```
注释云：

1. best-effort
2. The actual preemption will happen at some point in the future

这里的时机是 newstack (// Called from runtime·morestack when more stack is needed.):

```
preempt := atomicloaduintptr(&gp.stackguard0) == stackPreempt
if preempt:
	gopreempt_m(gp) // never return
		goschedImpl
			dropg
			globrunqput(gp)
			schedule()
```

	// We are interested in preempting user Go code, not runtime code.
	// If we're holding locks, mallocing, or preemption is disabled, don't
	// preempt.


