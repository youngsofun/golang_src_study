
# timer

rutime/time.go

要点：

1. 每个定时器对应一个timer结构，实际是绑定一个时间和一个回调函数。特别注意 period（见注释）。
2. 所有timer都放在一个全局的heap中，timerproc每次根据堆顶的时间notesleep
3. 加入新的timer如果需要，会提前notewakup timerproc
4. notesleep也是要占资源的，所以timerproc是add时按需启动，没有timer了还要goparkunlock


```
type timer struct {
	i int // heap index
	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(now, arg) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr
}

var timers struct {
	lock         mutex
	gp           *g
	created      bool
	sleeping     bool
	rescheduling bool
	waitnote     note
	t            []*timer
}
```

```

func timerproc() {
	timers.gp = getg()
	for {
		for {
			t = timers.t[0]
			delta = t.when - now
			if t.period > 0:
				t.when += t.period * (1 + -delta/t.period)
				siftdownTimer(0)
			else:
				f := t.f
				arg := t.arg
				seq := t.seq
				f(arg, seq)
		}
		if delta < 0 {
			// No timers left - put goroutine to sleep.
			timers.rescheduling = true
			goparkunlock(&timers.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
			continue
		}
		noteclear(&timers.waitnote)
		unlock(&timers.lock)
		notetsleepg(&timers.waitnote, delta)
	}
}
```

```
func addtimerLocked(t *timer) {
	t.i = len(timers.t)
	timers.t = append(timers.t, t)
	siftupTimer(t.i)
	if t.i == 0 {
		// siftup moved to top: new earliest deadline.
		if timers.sleeping {
			timers.sleeping = false
			notewakeup(&timers.waitnote)
		}
		if timers.rescheduling {
			timers.rescheduling = false
			goready(timers.gp, 0)
		}
	}
	if !timers.created {
		timers.created = true
		go timerproc()
	}
}

```




## 例子：userg 的sleep

```
func timeSleep(ns int64) {
	t := new(timer)
	t.when = nanotime() + ns
	t.f = goroutineReady
	t.arg = getg()
	lock(&timers.lock)
	addtimerLocked(t)
	goparkunlock(&timers.lock, "sleep", traceEvGoSleep, 2)
}

func goroutineReady(arg interface{}, seq uintptr) {
	goready(arg.(*g), 0)
}
```

