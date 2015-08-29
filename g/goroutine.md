
next [schedule](schedule.md)

0. [G, P, M 概述](#gpm)
1. [基础](#jichu)
    1. [getg](#getg)
    2. [mutex](#mutex)
2. [M](#M)
    1. 创建 [newm](#newm)
    2. newm 被调用情况
    3. startm， stopm
    4. m的结束
3. [P](#P)
    1. 创建 [procresize](#procresize)
4. [G](#G)
 	2. G 的创建[newproc](#newproc)

# [G, P, M 概述](id:gpm)

go 1.5 中
结构定义在 [runtime2.go](https://github.com/youngsofun/go/blob/master/src/runtime/runtime2.go#L211) 中,
调度相关在 [proc1.go](https://github.com/youngsofun/go/blob/master/src/runtime/proc1.go)

简单的说：

1. M 是系统线程
2. P是G的容器， \#P = GOMAXPROCS
3. G 被**分组**到某个P中，P被**分配**到M上，P队列头的G在M上执行

稍微复杂点：

1. M上可能没有P（比如syscall）, P也可能闲置(freelist中)
2. G可能换P，P可能换M，当然尽量不换，G可以被lock在M上。
3. G(runnable)可以能在全局队列中，
4. M有个 g0: goroutine with scheduling stack

# [基础](id:jichu)
# [getg](id:getg)

除了全局 g0 和 m0, 多数时候是通过getg()获得当前线程的g，然后借助他再拿到p和m。

* runtime/stubs.go

```
// getg returns the pointer to the current g.
// The compiler rewrites calls to this function into instructions
// that fetch the g directly (from TLS or from the dedicated register).
func getg() *g
```


* [cmd/compile/internal/amd64/ggen.go](https://github.com/youngsofun/go/blob/master/src/cmd/compile/internal/amd64/ggen.go#L729)
	
```
// res = runtime.getg()
func getg(res *gc.Node) {...} // 这里 gc 是go compiler
```

* [cmd/compile/internal/arm64/galign.go](https://github.com/youngsofun/go/blob/master/src/cmd/compile/internal/amd64/galign.go#L101)

```
	gc.Thearch.Getg = getg
```
		

## [mutex](id:mutex)



* lock_sema.go
	* // +build darwin nacl netbsd openbsd plan9 solaris windows
* lock_futex.go
	* // +build dragonfly freebsd linux


# [M](id:M)

[完整定义](https://github.com/youngsofun/go/blob/master/src/runtime/runtime2.go#L275)


```
	g0      *g     // goroutine with scheduling stack
	curg          *g       // current running goroutine
	park          note // 停车场
```

##[newm](id:newm)

```
func newm(fn func(), _p_ *p)      
     newosproc(mp, unsafe.Pointer(mp.g0.stack.hi))
     // newosproc(mp *m, stk unsafe.Pointer)  // os1_linux.go
          clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
          // TEXT runtime·clone(SB),NOSPLIT,$0   // Copy mp, gp, fn off parent stack for use by child
          
```

newm(fn func(), _p_ *p)  特殊，和系统调用fork/clone一样，一跑到他就是"花开两枝，各表一头"。
注意clone后的线程跑的是 mstart()：

```
	minit()
	...
	
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}
	if _g_.m.helpgc != 0 {
		_g_.m.helpgc = 0
		stopm()
	} else if _g_.m != &m0 {
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	schedule()
```

也就是说mstart先跑一下m.mstartfn, 然后m.nextp的schedule，他们是在newm中被设置的：

```
newm()：
 	 allocm:
 		 mp.mstartfn = fn 
 	 mp.nextp.set(_p_)
 	 newosproc ...
```

## newm时机

上文可见

* 如果 newm时 p == nil， 就可以理解为一般的fork，跑fn , 如sysmon和mhelpgc。
	* 问题来了：  他们不需要P就能跑吗? 应该没问题，因为他们跑不到schedule：
		* 对于helpgc，会直接stopm()
		* 而sysmon是一个无限循环
* 否则 就是先跑fn，除了nil就是mspinning， 然后跑p 

### newm 被调用的3处：

1. runtime.main() 
	2. newm(sysmon, nil) 	
2. startTheWorldWithSema() 会为每个P启动M，P如果没有带着M，就创建
	3. newm(nil, p)	
	4. newm(mhelpgc, nil)
2. startm() Schedules some M to run the p (creates an M if necessary).
	3. fn = mspinning； newm(fn, _p_) 
	
   * 其被调用链
   	 * handoffp  // Hands off P from syscall or locked M.    
   	 * wakep
       * ready // g ready to run
       * newproc1  // Create a new g 
       
###  startm，stopm

* 注意一开始执行的是mstart， 另外还有个startm，stopm，

startm: 是Schedules some M to run the p (creates an M if necessary)

```
startm(_p_ *p, spinning bool) ：  
	if mp = mget(); 
		mp.nextp.set(_p_)
		notewakeup(&mp.park)
	else:
	   newm(fn, _p_)	 
```
			
			
stopm: Stops execution of the current m until new work is available

```
	mput(_g_.m) // 放到 idle list
	notesleep(&_g_.m.park)
```

			
### m 的结束

没有mstop。clone后"线程"执行结束就是结束了 ( P跑完一个（或一会儿）G 会找下一个，如果找不到，就futex_sleep, 所以最多GOMAXPROCS个物理线程因此 sleep)，但m结构体会放到 idle list 中，永远不会被销毁
	

          
# P 

P 最简单， 重要filed：
	
```
	runq     [256]*g
	mcache      *mcache
// 和 gc 相关
	gcBgMarkWorker   *g
	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork
```

## [创建](id：procresize)

schedinit 以及 调用GOMAXPROCS()时，由procresize批量产生。
	


	
## G


```
	stack       stack   // offset known to runtime/cgo
	
	_panic       *_panic // innermost panic 
	_defer         *_defer // innermost defer
	
	m              *m      // current m
	
	sched          gobuf
	syscallsp      uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc      uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
	
	param          unsafe.Pointer // passed parameter on wakeup
	waiting        *sudog // sudog structures this g is waiting on (that have a valid elem ptr)
		
	waitreason     string // if status==Gwaiting
	schedlink      guintptr
	preempt        bool   // preemption signal, duplicates stackguard0 = stackpreempt
	
	preemptscan    bool   // preempted g does scan for gc
	gcscandone     bool   // g has scanned stack; protected by _Gscan bit in status
	gcscanvalid    bool   // false at start of gc cycle, true if G has not run since last scan
	
	throwsplit     bool   // must not split stack




```

## [G 的创建](id:newproc)

```
// 。。。 The compiler turns a go statement into a call to this.。。。

func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), ptrSize)
	pc := getcallerpc(unsafe.Pointer(&siz))
	systemstack(func() {
		newproc1(fn, (*uint8)(argp), siz, 0, pc)
	})
}
```

## G分类
* user G
* system G 
	* 常驻
		* m->g0
		* m->gsignal 一个还是 per_p?
* gc... timer 怎么算？
	
	
