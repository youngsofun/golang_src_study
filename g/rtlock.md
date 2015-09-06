
# runtime 内同步
## overview

[runtime/lock_futex.go](https://github.com/youngsofun/go/blob/master/src/runtime/lock_futex.go) 中实现了基于os提供的futex提供了两组同步原语，字面很好理解：


func lock(l *mutex) 
func unlock(l *mutex) 

func notewakeup(n *note)
func notesleep(n *note)
func notetsleep(n *note, ns int64) bool  // return false 表示超时

```
type mutex struct {
	key uintptr
}

type note struct {
	key uintptr
}
```


### futex 的原型

```
// This implementation depends on OS-specific implementations of
//
//	runtime·futexsleep(uint32 *addr, uint32 val, int64 ns)
//		Atomically,
//			if(*addr == val) sleep
//		Might be woken up spuriously; that's allowed.
//		Don't sleep longer than ns; ns < 0 means forever.
//
//	runtime·futexwakeup(uint32 *addr, uint32 cnt)
//		If any procs are sleeping on addr, wake up at most cnt.
```

## note

TODO: 不理解这个名字

“sleep and wakeup on one-time events. ”

这组和futex几乎是直接对应的：


func notewakeup(n *note)

	futexwakeup(key32(&n.key), 1)
	
	
func notesleep(n *note) 

	futexsleep(key32(&n.key), 0, -1)

func notetsleep(n *note, ns int64) bool
 	futexsleep(key32(&n.key), 0, ns)
 	
 	
用法见 [代码注释](https://github.com/youngsofun/go/blob/master/src/runtime/runtime2.go#L666)

此外还有个 notesleepg ：“is similar to notetsleep but is called on user g”. 只用在timer/sleep和signals处理当中。


```
func notetsleepg(n *note, ns int64) bool {
	...
	entersyscallblock(0)
	ok := notetsleep_internal(n, ns)
	exitsyscall(0)
	return ok
}
```

注：entersyscallblock 只用在了这一处

##  lock

```
const (
	mutex_unlocked = 0
	mutex_locked   = 1
	mutex_sleeping = 2

	active_spin     = 4
	active_spin_cnt = 30
	passive_spin    = 1
)

// Possible lock states are mutex_unlocked, mutex_locked and mutex_sleeping.
// mutex_sleeping means that there is presumably at least one sleeping thread.
// Note that there can be spinning threads during all states - they do not
// affect mutex's state
```
先看相对简单的 unlock:

1. 有人等才futexwakeup
2. m.lock不为0时，是无法被抢占的 （注意这时谈的lock不是sync.Muxtex，所以没那么严重）

```
func unlock(l *mutex) {
	v := xchg(key32(&l.key), mutex_unlocked)
	if v == mutex_sleeping {
		futexwakeup(key32(&l.key), 1)
	}

	gp := getg()
	gp.m.locks--
	if gp.m.locks == 0 && gp.preempt { // restore the preemption request in case we've cleared it in newstack
		gp.stackguard0 = stackPreempt
	}
}
```

再看 lock，注意：

1. 开始，spinlock，sleep前多次尝试原子操作
2. futexsleep时如果发生变化，会continue 
3. 比较有趣的是spinlock, （即使单P还有个passive_spin），之前没研究过，感觉时间不短，最不理解的是：只是一个g可能阻塞，为什么要急于让出cpu？[TODO]


params：

	active_spin     = 4
	active_spin_cnt = 30
	passive_spin    = 1

code：

```
	gp.m.locks++
	v := xchg(key32(&l.key), mutex_locked)
	if v == mutex_unlocked {
		return
	}
	wait := v
	
	// On uniprocessors, no point spinning.
	// On multiprocessors, spin for ACTIVE_SPIN attempts.
	spin := 0
	if ncpu > 1 {
		spin = active_spin
	}
	for {
		// Try for lock, spinning.
		for i := 0; i < spin; i++ {
			for l.key == mutex_unlocked {
				if cas(key32(&l.key), mutex_unlocked, wait) {
					return
				}
			}
			procyield(active_spin_cnt)
		}

		// Try for lock, rescheduling.
		for i := 0; i < passive_spin; i++ {
			for l.key == mutex_unlocked {
				if cas(key32(&l.key), mutex_unlocked, wait) {
					return
				}
			}
			osyield()
		}

		// Sleep.
		v = xchg(key32(&l.key), mutex_sleeping)
		if v == mutex_unlocked {
			return
		}
		wait = mutex_sleeping
		futexsleep(key32(&l.key), mutex_sleeping, -1)
	}
```

###  procyield
```
TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```

[explain](http://stackoverflow.com/a/4725741) :

> PAUSE notifies CPU that this is spinlock wait loop so memory and cache accesses may be optimized. Also PAUSE may actually stop CPU for some time while NOP runs as fast as possible.




### osyield

```
TEXT runtime·osyield(SB),NOSPLIT,$0
	MOVL	$24, AX
	SYSCALL
	RET
```

man :

> A process can relinquish the processor voluntarily without blocking by calling sched_yield(). The process will then be moved to the end of the queue for its static priority and a new process gets to run.


[Linux Kernel Development](http://www.informit.com/articles/article.aspx?p=101760&seqNum=5):

> inux通过sched_yield()系统调用，提供了一种让进程显式地将处理器时间让给其他等待执行进程的机制。它是通过将进程从活动队列中（因为进程正在执行，所以它肯定位于此队列当中）移到过期队列中实现的。由此产生的效果不仅抢占了该进程并将其放入优先级队列的最后面，还将其放入过期队列中—这样能确保在一段时间内它都不会再被执行了。






