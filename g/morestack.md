

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


# morestack()

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
	
	
```
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


