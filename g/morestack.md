
# 暂时的有用结论

一些细节不明白，搁置一下，主要的一些问题已经可以回答了：

1. 决定是否要更多stack？主要是比较SP和stackGuard, 根据frame的大小有不同。
2. morestack自己也是函数，它的切换和调用普通函数的不同？掉函数本身就要用栈，但这里要换掉stack。m.morebuf 和g.sched怎么用的比较细致，回头再看[TODO]。
3. stack 变大多少？现在的简单的double，最大1G。
4. 新stack的分配？放到mallocgc里一块讨论 
5. copystack的怎么处理指针？会escape的都分配到堆上了， 扫描stack，只被stack内部引用的地方，引用换个偏移量就行（ajustpointers函数）
6. G的状态切换涉及的问题 【TODO】
7. copystack 过程中的抢占，放到 调度里 处理
7. copystack过程中的GC
8. 好像用到一直不懂的 PCDATA， [文献](https://docs.google.com/document/d/1lyPIbmsYbXnpNj57a261hgOYVpNRcgydurVQIyZOz_o/pub)

TODO: 下面待整理
----



# morestack() 前



```
syscall_unix.go:159     0x473700        64488b0c25f8ffffff      FS MOVQ FS:0xfffffff8, CX
syscall_unix.go:159     0x473709        483b6110                CMPQ 0x10(CX), SP
syscall_unix.go:159     0x47370d        7661                    JBE 0x473770
...
syscall_unix.go:159     0x473770        e8cbf6fdff              CALL runtime.morestack_noctxt(SB)
        syscall_unix.go:159     0x473775        eb89                    JMP syscall.Read(SB)
```
[stack](stack.md) 一节中，除了 syscall.Syscall，read 和 Read 都有这段代码。它是检查是否需要申请更多stack。

生成过程： (go代码中的 "//go:nosplit" ->) asm 中的 NOSPLIT -> 最终不加人上述代码


runtime·morestack_noctxt @ [runtime/asm_amd64.s#L334](https://github.com/youngsofun/go/blob/master/src/runtime/asm_amd64.s#L334)

```
// morestack but not preserving ctxt.
TEXT runtime·morestack_noctxt(SB),NOSPLIT,$0
	MOVL	$0, DX
	JMP	runtime·morestack(SB)
```
runtime·morestack_noctxt @ [runtime/asm_amd64.s#L334](https://github.com/youngsofun/go/blob/master/src/runtime/asm_amd64.s#L334)

```
/*
 * support for morestack
 */

// Called during function prolog when more stack is needed.
//
// The traceback routines see morestack on a g0 as being
// the top of a stack (for example, morestack calling newstack
// calling the scheduler calling newm calling gc), so we must
// record an argument size. For that purpose, it has no arguments.
TEXT runtime·morestack(SB),NOSPLIT,$0-0
	// Cannot grow scheduler stack (m->g0).
	get_tls(CX)
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX
	MOVQ	m_g0(BX), SI
	CMPQ	g(CX), SI
	JNE	2(PC)
	INT	$3

	// Cannot grow signal stack (m->gsignal).
	MOVQ	m_gsignal(BX), SI
	CMPQ	g(CX), SI
	JNE	2(PC)
	INT	$3

	// Called from f.
	// Set m->morebuf to f's caller.
	MOVQ	8(SP), AX	// f's caller's PC
	MOVQ	AX, (m_morebuf+gobuf_pc)(BX)
	LEAQ	16(SP), AX	// f's caller's SP
	MOVQ	AX, (m_morebuf+gobuf_sp)(BX)
	get_tls(CX)
	MOVQ	g(CX), SI
	MOVQ	SI, (m_morebuf+gobuf_g)(BX)

	// Set g->sched to context in f.
	MOVQ	0(SP), AX // f's PC
	MOVQ	AX, (g_sched+gobuf_pc)(SI)
	MOVQ	SI, (g_sched+gobuf_g)(SI)
	LEAQ	8(SP), AX // f's SP
	MOVQ	AX, (g_sched+gobuf_sp)(SI)
	MOVQ	DX, (g_sched+gobuf_ctxt)(SI)
	MOVQ	BP, (g_sched+gobuf_bp)(SI)

	// Call newstack on m->g0's stack.
	MOVQ	m_g0(BX), BX
	MOVQ	BX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(BX), SP
	CALL	runtime·newstack(SB)
	MOVQ	$0, 0x1003	// crash if newstack returns
	RET
	
```
用到 g 和 m 的相关field：

```
type m struct {
	g0      *g     // goroutine with scheduling stack
	morebuf gobuf  // gobuf arg to morestack
	gsignal *g     // signal-handling g
}

type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic         *_panic // innermost panic - offset known to liblink
	_defer         *_defer // innermost defer
	m              *m      // current m; offset known to arm liblink
	stackAlloc     uintptr // stack allocation is [stack.lo,stack.lo+stackAlloc)
	sched          gobuf
}

```
g 和 m 中都有一个gobuf 结构

```
type gobuf struct {
	// The offsets of sp, pc, and g are known to (hard-coded in) libmach.
	sp   uintptr
	pc   uintptr
	g    guintptr
	ctxt unsafe.Pointer // this has to be a pointer so that gc scans it
	ret  uintreg
	lr   uintptr
	bp   uintptr // for GOEXPERIMENT=framepointer
}
```

### TLS

```FS MOVQ FS:0xfffffff8, CX``` 是从 ```MOVQ    (TLS), CX``` 译来

TODO: FS 和 TLS (线程本地存储) 有关， [R1](http://stackoverflow.com/questions/6611346/amd64-fs-gs-registers-in-linux)， [R2](http://blog.csdn.net/dog250/article/details/7704898)

```
#ifdef GOARCH_amd64
#define	get_tls(r)	MOVQ TLS, r
#define	g(r)	0(r)(TLS*1)
#endif
```




## 策略

```

	guard = g->stackguard
	frame = function's stack frame size
	argsize = size of function arguments (call + return)

	stack frame size <= StackSmall:
		CMPQ guard, SP
		JHI 3(PC)
		MOVQ m->morearg, $(argsize << 32)
		CALL morestack(SB)

	stack frame size > StackSmall but < StackBig
		LEAQ (frame-StackSmall)(SP), R0
		CMPQ guard, R0
		JHI 3(PC)
		MOVQ m->morearg, $(argsize << 32)
		CALL morestack(SB)

	stack frame size >= StackBig:
		MOVQ m->morearg, $((argsize << 32) | frame)
		CALL morestack(SB)
```


*  _StackSmall = 128 对应 The red zone

### 策略的实现


[cmd/internal/obj/arm64/obj7.go](https://github.com/youngsofun/go/blob/master/src/cmd/internal/obj/arm64/obj7.go#L52)

```
func stacksplit(ctxt *obj.Link, p *obj.Prog, framesize int32) *obj.Prog {
	...
	call.To.Type = obj.TYPE_BRANCH
	morestack := "runtime.morestack"
	switch {
	case ctxt.Cursym.Cfunc != 0:
		morestack = "runtime.morestackc"
	case ctxt.Cursym.Text.From3.Offset&obj.NEEDCTXT == 0:
		morestack = "runtime.morestack_noctxt"
	}
	call.To.Sym = obj.Linklookup(ctxt, morestack, 0)
}
	
func preprocess(ctxt *obj.Link, cursym *obj.LSym) {
	for p := cursym.Text; p != nil; p = p.Link {
		o = int(p.As)
		switch o {
		case obj.ATEXT:
		... // 计算 ctxt.Autosize， 也就是 framesize！
			if !(p.From3.Offset&obj.NOSPLIT != 0) {
				p = stacksplit(ctxt, p, ctxt.Autosize) // emit split check
			}
		...
	}
}

```


# newstack()

@ stack1.go#665


```
// void gogo(Gobuf*)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $0-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		// make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	gobuf_sp(BX), SP	// restore SP
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP
	MOVQ	$0, gobuf_sp(BX)	// clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
	MOVQ	gobuf_pc(BX), BX
	JMP	BX
```

```
func main() {
	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
	if ptrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}
```

```
// Called from runtime·morestack when more stack is needed.
// Allocate larger stack and relocate to new stack.
// Stack growth is multiplicative, for constant amortized cost.
//
// g->atomicstatus will be Grunning or Gscanrunning upon entry.
// If the GC is trying to stop this g then it will set preemptscan to true.
func newstack() {
	thisg := getg()
	gp := thisg.m.curg
	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0
	rewindmorestack(&gp.sched)

	// NOTE: stackguard0 may change underfoot, if another thread
	// is about to try to preempt gp. Read it just once and use that same
	// value now and below.
	preempt := atomicloaduintptr(&gp.stackguard0) == stackPreempt

	// Be conservative about where we preempt.
	// We are interested in preempting user Go code, not runtime code.
	// If we're holding locks, mallocing, or preemption is disabled, don't
	// preempt.
	// This check is very early in newstack so that even the status change
	// from Grunning to Gwaiting and back doesn't happen in this case.
	// That status change by itself can be viewed as a small preemption,
	// because the GC might change Gwaiting to Gscanwaiting, and then
	// this goroutine has to wait for the GC to finish before continuing.
	// If the GC is in some way dependent on this goroutine (for example,
	// it needs a lock held by the goroutine), that small preemption turns
	// into a real deadlock.
	if preempt {
		if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
			// Let the goroutine keep running for now.
			// gp->preempt is set, so it will be preempted next time.
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gwaiting)
	gp.waitreason = "stack growth"

	sp := gp.sched.sp
	if thechar == '6' || thechar == '8' {
		// The call to morestack cost a word.
		sp -= ptrSize
	}
	
	if gp.sched.ctxt != nil {
		// morestack wrote sched.ctxt on its way in here,
		// without a write barrier. Run the write barrier now.
		// It is not possible to be preempted between then
		// and now, so it's okay.
		writebarrierptr_nostore((*uintptr)(unsafe.Pointer(&gp.sched.ctxt)), uintptr(gp.sched.ctxt))
	}

	if preempt {
		if gp.preemptscan {
			for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {
				// Likely to be racing with the GC as
				// it sees a _Gwaiting and does the
				// stack scan. If so, gcworkdone will
				// be set and gcphasework will simply
				// return.
			}
			if !gp.gcscandone {
				scanstack(gp)
				gp.gcscandone = true
			}
			gp.preemptscan = false
			gp.preempt = false
			casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)
			casgstatus(gp, _Gwaiting, _Grunning)
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}

		// Act like goroutine called runtime.Gosched.
		casgstatus(gp, _Gwaiting, _Grunning)
		gopreempt_m(gp) // never return
	}

	// Allocate a bigger segment and move the stack.
	oldsize := int(gp.stackAlloc)
	newsize := oldsize * 2
	if uintptr(newsize) > maxstacksize {
		print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
		throw("stack overflow")
	}

	casgstatus(gp, _Gwaiting, _Gcopystack)

	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
	copystack(gp, uintptr(newsize))
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}
```
# copysstack


```
	adjustframe((*stkframe)&frame))), adjinfo) 
	// adjust other miscellaneous things that have pointers into stacks.
	adjustctxt(gp, &adjinfo)
	adjustdefers(gp, &adjinfo)
	adjustpanics(gp, &adjinfo)
	adjustsudogs(gp, &adjinfo)
	adjuststkbar(gp, &adjinfo)
```
adjustframe
	// bv describes the memory starting at address scanp.
	// Adjust any pointers contained therein.
	adjustpointers

# stackalloc

# stackfree

