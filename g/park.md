
# park and ready

进入，退出 waiting状态


## gopark

1. 核心工作是 casgstatus(gp, _Grunning, _Gwaiting)，然后schedule()；但好像没看到对P的队列的操作。
2. park是因为等等某个事件，如果最后发现 `unlockf returns false`, 就说明发生变化，可以比如重试。
3. 每个都有reason，代码中容易找到。


```
// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason string, traceEv byte, traceskip int) {
	mp := acquirem()
	gp := mp.curg
	mp.waitlock = lock
	mp.waitunlockf = *(*unsafe.Pointer)(unsafe.Pointer(&unlockf))
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}
```

比较常用的：

```
// Puts the current goroutine into a waiting state and unlocks the lock.
// The goroutine can be made runnable again by calling goready(gp).
func goparkunlock(lock *mutex, reason string, traceEv byte, traceskip int) {
	gopark(parkunlock_c, unsafe.Pointer(lock), reason, traceEv, traceskip)
}
```

mcall(park_m)：

```
// park continuation on g0.
func park_m(gp *g) {
	_g_ := getg()
	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	if _g_.m.waitunlockf != nil {
		fn := *(*func(*g, unsafe.Pointer) bool)(unsafe.Pointer(&_g_.m.waitunlockf))
		ok := fn(gp, _g_.m.waitlock)
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
		if !ok {
			if trace.enabled {
				traceGoUnpark(gp, 2)
			}
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
	schedule()
}
```

## goready

```
func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip)
	})
}
```