
[morestack](morestack.md), [schedule](schedule.md)

# 必要的asm 

## 生成汇编的三种方法

1. 正向
``` $ GOOS=linux GOARCH=amd64 go tool compile -S x.go ``` 
or 
``` go build -gcflags -S x.go```

2. go 逆向
3. objdump

逆向出来的是基本一致的，只是用的汇编语言不同，2相对3的好处是可以看到对应的代码行号，便于学习。正向区别比较不同：

1. 会用到 pseudo-register。
2. 少很多优化
3. 会包含一些	directives, 用于指导后续工作，换句话说，很多工作没有体现在1的汇编代码中
	1. 用于指导GC等，如：FUNCDATA and PCDATA。
	2. 其他？



## plan9汇编中的pseudo-register

* FP: Frame pointer: arguments and locals.
* PC: Program counter: jumps and branches.
* SB: Static base pointer: global symbols.
* SP: Stack pointer: top of stack.


## frame和stack，FP和SP

 1. stack：从高地址向低地址扩展
 2. frame：简单理解，传统上SP认为会不停变，而FP在一个函数内部不变，从而方便的定位stack上的地址。amd64中rbp被设计成做这个的。
 3. 实际中，因为一个函数的frame的大小是可以事先确定的，函数一开始就把SP变到足够大，并不变，这样就不需要FP了，还空出一个通用寄存器。


# golang 中普通的函数调用

##  Stack frame layout
```
// +------------------+
// | args from caller |
// +------------------+ <- frame->argp
// |  return address  |
// +------------------+
// |  caller's BP (*) | (*) if framepointer_enabled && varp < sp
// +------------------+ <- frame->varp
// |     locals       |
// +------------------+
// |  args to callee  |
// +------------------+ <- frame->sp

```

注意：

1. caller's BP 通常没有
2. return address 应该是 CALL 指令压栈的， 
3. args to callee 是算在caller的的frame中的
4. func(a1, a2) (r1, r2) 会按照逆序 (r2, r1, a2, a1) “压栈“, 这样是合理的：内存地址是 a1 < a2 < a3 <a4。 (r1, r2) 只是留出位置
5. 调用 func([]byte){...}，会传三个arg

这些结论来自下面的例子：

### 简单例子

```
func foo(a int) float64 {
        return math.Max(float64(a), 10)
}

func foo2(a int) float64 {
        return 2* float64(a)
}
```

因为  math.Max需要三个word的空间，所以 locals从 0x0涨到0x18

```
"".foo t=1 size=80 value=0 args=0x10 locals=0x18
        0x0000 00000 (b.go:5)   TEXT    "".foo(SB), $24-16
        0x0000 00000 (b.go:5)   MOVQ    (TLS), CX
        0x0009 00009 (b.go:5)   CMPQ    SP, 16(CX)
        0x000d 00013 (b.go:5)   JLS     73
        0x000f 00015 (b.go:5)   SUBQ    $24, SP
        0x0013 00019 (b.go:5)   FUNCDATA        $0, gclocals·23e8278e2b69a3a75fa59b23c49ed6ad(SB)
        0x0013 00019 (b.go:5)   FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0013 00019 (b.go:9)   MOVQ    "".a+32(FP), BX
        0x0018 00024 (b.go:5)   XORPS   X0, X0
        0x001b 00027 (b.go:9)   CVTSQ2SD        BX, X0
        0x0020 00032 (b.go:9)   MOVSD   X0, (SP)
        0x0025 00037 (b.go:9)   MOVSD   $f64.4024000000000000(SB), X0
        0x002d 00045 (b.go:9)   MOVSD   X0, 8(SP)
        0x0033 00051 (b.go:9)   PCDATA  $0, $0
        0x0033 00051 (b.go:9)   CALL    math.Max(SB)
        0x0038 00056 (b.go:9)   MOVSD   16(SP), X0
        0x003e 00062 (b.go:9)   MOVSD   X0, "".~r1+40(FP)
        0x0044 00068 (b.go:9)   ADDQ    $24, SP
        0x0048 00072 (b.go:9)   RET
        0x0049 00073 (b.go:5)   CALL    runtime.morestack_noctxt(SB)
        0x004e 00078 (b.go:5)   JMP     0
        0x0000 64 48 8b 0c 25 00 00 00 00 48 3b 61 10 76 3a 48  dH..%....H;a.v:H
        0x0010 83 ec 18 48 8b 5c 24 20 0f 57 c0 f2 48 0f 2a c3  ...H.\$ .W..H.*.
        0x0020 f2 0f 11 04 24 f2 0f 10 05 00 00 00 00 f2 0f 11  ....$...........
        0x0030 44 24 08 e8 00 00 00 00 f2 0f 10 44 24 10 f2 0f  D$.........D$...
        0x0040 11 44 24 28 48 83 c4 18 c3 e8 00 00 00 00 eb b0  .D$(H...........
        rel 5+4 t=13 +0
        rel 41+4 t=11 $f64.4024000000000000+0
        rel 52+4 t=5 math.Max+0
        rel 74+4 t=5 runtime.morestack_noctxt+0
        
"".foo2 t=1 size=48 value=0 args=0x10 locals=0x0
        0x0000 00000 (b.go:12)  TEXT    "".foo2(SB), $0-16
        0x0000 00000 (b.go:12)  NOP
        0x0000 00000 (b.go:12)  NOP
        0x0000 00000 (b.go:12)  FUNCDATA        $0, gclocals·23e8278e2b69a3a75fa59b23c49ed6ad(SB)
        0x0000 00000 (b.go:12)  FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x0000 00000 (b.go:16)  MOVQ    "".a+8(FP), BX
        0x0005 00005 (b.go:12)  XORPS   X0, X0
        0x0008 00008 (b.go:16)  CVTSQ2SD        BX, X1
        0x000d 00013 (b.go:16)  MOVAPD  X1, X0
        0x0011 00017 (b.go:16)  MOVSD   $f64.4000000000000000(SB), X1
        0x0019 00025 (b.go:16)  MULSD   X1, X0
        0x001d 00029 (b.go:16)  MOVSD   X0, "".~r1+16(FP)
        0x0023 00035 (b.go:16)  RET
        0x0000 48 8b 5c 24 08 0f 57 c0 f2 48 0f 2a cb 66 0f 28  H.\$..W..H.*.f.(
        0x0010 c1 f2 0f 10 0d 00 00 00 00 f2 0f 59 c1 f2 0f 11  ...........Y....
        0x0020 44 24 10 c3                                      D$..
        rel 21+4 t=11 $f64.4000000000000000+0
```
###  复杂例子

caller: **Read()** in syscall/syscall_unix.go

```
159 func Read(fd int, p []byte) (n int, err error) {
160     n, err = read(fd, p)
161     if raceenabled {
162         if n > 0 {
163             raceWriteRange(unsafe.Pointer(&p[0]), n)
164         }
165         if err == nil {
166             raceAcquire(unsafe.Pointer(&ioSync))
167         }                                                                                                                     
168     }
169     return
170 }   
```

callee: **read()** in syscall/zsyscall_linux_amd64.go


```
 776 func read(fd int, p []byte) (n int, err error) {
 777     var _p0 unsafe.Pointer
 778     if len(p) > 0 {
 779         _p0 = unsafe.Pointer(&p[0])
 780     } else {
 781         _p0 = unsafe.Pointer(&_zero)
 782     }                                                                                                                        
 783     r0, _, e1 := Syscall(SYS_READ, uintptr(fd), uintptr(_p0), uintptr(len(p)))
 784     n = int(r0)
 785     if e1 != 0 {
 786         err = errnoErr(e1)
 787     }
 788     return
 789 }   
```

围绕  n, err = read(fd, p)这一句：

* caller：
  1. fd和 p 写到 0(SP) 到 18(SP) # 并空出 20(SP) 28(SP) 做返回值 n 和 err
  2. CALL syscall.read(SB) # 返回地址压栈，隐含 ```SUBQ 8, SP```
* callee：
  1. SUBQ $0x38, SP # 0(SP) to 18(SP) 变成 40(SP) to 58(SP)
  2. ADDQ $0x38, SP 
  3. RET # ADDQ $0x8, SP
  	
  	
  	
callee 的这个0x38怎么来的？

1. local var：这里都被优化掉了
2. callee的参数“最长”的子callee： 这里是CALL syscall.Syscall(SB),  参数和返回值占用了 0(SP) to 30(SP)。 38(SP)是返回地址，所以完全没有浪费。
	* 注: 还调用了CALL syscall.errnoErr(SB)，只有一参数一返回值，参数用的是0(SP)


他们的反汇编：


caller：

```
TEXT syscall.Read(SB) /usr/local/go/src/syscall/syscall_unix.go
	# Read(fd int, p []byte) (n int, err error)
        syscall_unix.go:159     0x473700        64488b0c25f8ffffff      FS MOVQ FS:0xfffffff8, CX
        syscall_unix.go:159     0x473709        483b6110                CMPQ 0x10(CX), SP
        syscall_unix.go:159     0x47370d        7661                    JBE 0x473770
        syscall_unix.go:159     0x47370f        4883ec38                SUBQ $0x38, SP
        syscall_unix.go:159     0x473713        31db                    XORL BX, BX
        syscall_unix.go:159     0x473715        31db                    XORL BX, BX
        syscall_unix.go:159     0x473717        48895c2468              MOVQ BX, 0x68(SP)
        syscall_unix.go:159     0x47371c        48895c2470              MOVQ BX, 0x70(SP)
     # n, err = read(fd, p) ，传人4个参数 0(SP) 到 18(SP)
        syscall_unix.go:160     0x473721        488b5c2440              MOVQ 0x40(SP), BX
        syscall_unix.go:160     0x473726        48891c24                MOVQ BX, 0(SP)
        syscall_unix.go:160     0x47372a        488b5c2448              MOVQ 0x48(SP), BX
        syscall_unix.go:160     0x47372f        48895c2408              MOVQ BX, 0x8(SP)
        syscall_unix.go:160     0x473734        488b5c2450              MOVQ 0x50(SP), BX
        syscall_unix.go:160     0x473739        48895c2410              MOVQ BX, 0x10(SP)
        syscall_unix.go:160     0x47373e        488b5c2458              MOVQ 0x58(SP), BX
        syscall_unix.go:160     0x473743        48895c2418              MOVQ BX, 0x18(SP)
        syscall_unix.go:160     0x473748        e8c3050000              CALL syscall.read(SB)
        syscall_unix.go:160     0x47374d        488b5c2420              MOVQ 0x20(SP), BX
        syscall_unix.go:160     0x473752        48895c2460              MOVQ BX, 0x60(SP)
        syscall_unix.go:160     0x473757        488b5c2428              MOVQ 0x28(SP), BX
        syscall_unix.go:160     0x47375c        48895c2468              MOVQ BX, 0x68(SP)
        syscall_unix.go:160     0x473761        488b5c2430              MOVQ 0x30(SP), BX
        syscall_unix.go:160     0x473766        48895c2470              MOVQ BX, 0x70(SP)
     # return
        syscall_unix.go:169     0x47376b        4883c438                ADDQ $0x38, SP
        syscall_unix.go:169     0x47376f        c3                      RET
 
        syscall_unix.go:159     0x473770        e8cbf6fdff              CALL runtime.morestack_noctxt(SB)
        syscall_unix.go:159     0x473775        eb89                    JMP syscall.Read(SB)
        syscall_unix.go:159     0x473777        0000                    ADDL AL, 0(AX)
        syscall_unix.go:159     0x473779        0000                    ADDL AL, 0(AX)
        syscall_unix.go:159     0x47377b        0000                    ADDL AL, 0(AX)
        syscall_unix.go:159     0x47377d        0000                    ADDL AL, 0(AX)
        syscall_unix.go:159     0x47377f        00                      ?

```

callee：

```
TEXT syscall.read(SB) /usr/local/go/src/syscall/zsyscall_linux_amd64.go
        zsyscall_linux_amd64.go:776     0x473d10        64488b0c25f8ffffff      FS MOVQ FS:0xfffffff8, CX
        zsyscall_linux_amd64.go:776     0x473d19        483b6110                CMPQ 0x10(CX), SP
        zsyscall_linux_amd64.go:776     0x473d1d        0f8696000000            JBE 0x473db9
        zsyscall_linux_amd64.go:776     0x473d23        4883ec38                SUBQ $0x38, SP
        zsyscall_linux_amd64.go:776     0x473d27        488b4c2450              MOVQ 0x50(SP), CX # 这里是 p.len
        zsyscall_linux_amd64.go:776     0x473d2c        31db                    XORL BX, BX
        zsyscall_linux_amd64.go:776     0x473d2e        31db                    XORL BX, BX
        zsyscall_linux_amd64.go:776     0x473d30        48895c2468              MOVQ BX, 0x68(SP)
        zsyscall_linux_amd64.go:776     0x473d35        48895c2470              MOVQ BX, 0x70(SP)
    # if len(p) > 0 
        zsyscall_linux_amd64.go:778     0x473d3a        4883f900                CMPQ $0x0, CX
        zsyscall_linux_amd64.go:778     0x473d3e        7e6d                    JLE 0x473dad
        zsyscall_linux_amd64.go:779     0x473d40        488b5c2448              MOVQ 0x48(SP), BX # 这是 p.array
        zsyscall_linux_amd64.go:779     0x473d45        4883f900                CMPQ $0x0, CX
        zsyscall_linux_amd64.go:779     0x473d49        765b                    JBE 0x473da6
        zsyscall_linux_amd64.go:779     0x473d4b        4889d8                  MOVQ BX, AX
            
        zsyscall_linux_amd64.go:783     0x473d4e        48c7042400000000        MOVQ $0x0, 0(SP) # 系统调用号
        zsyscall_linux_amd64.go:783     0x473d56        488b5c2440              MOVQ 0x40(SP), BX # fd 
        zsyscall_linux_amd64.go:783     0x473d5b        48895c2408              MOVQ BX, 0x8(SP)  
        zsyscall_linux_amd64.go:783     0x473d60        4889442410              MOVQ AX, 0x10(SP) # p 
        zsyscall_linux_amd64.go:783     0x473d65        48894c2418              MOVQ CX, 0x18(SP) # len  
        zsyscall_linux_amd64.go:783     0x473d6a        e821110000              CALL syscall.Syscall(SB)  
        zsyscall_linux_amd64.go:783     0x473d6f        488b4c2420              MOVQ 0x20(SP), CX # n
        zsyscall_linux_amd64.go:783     0x473d74        488b442430              MOVQ 0x30(SP), AX # e1
        
        zsyscall_linux_amd64.go:784     0x473d79        48894c2460              MOVQ CX, 0x60(SP)
        zsyscall_linux_amd64.go:785     0x473d7e        4883f800                CMPQ $0x0, AX
        zsyscall_linux_amd64.go:785     0x473d82        741d                    JE 0x473da1
        zsyscall_linux_amd64.go:786     0x473d84        48890424                MOVQ AX, 0(SP)
        zsyscall_linux_amd64.go:786     0x473d88        e853f7ffff              CALL syscall.errnoErr(SB)
        zsyscall_linux_amd64.go:786     0x473d8d        488b5c2408              MOVQ 0x8(SP), BX
        zsyscall_linux_amd64.go:786     0x473d92        48895c2468              MOVQ BX, 0x68(SP)
        zsyscall_linux_amd64.go:786     0x473d97        488b5c2410              MOVQ 0x10(SP), BX
        zsyscall_linux_amd64.go:786     0x473d9c        48895c2470              MOVQ BX, 0x70(SP)
        zsyscall_linux_amd64.go:788     0x473da1        4883c438                ADDQ $0x38, SP
        zsyscall_linux_amd64.go:788     0x473da5        c3                      RET
        zsyscall_linux_amd64.go:779     0x473da6        e80510fbff              CALL runtime.panicindex(SB)
        zsyscall_linux_amd64.go:779     0x473dab        0f0b                    UD2
        zsyscall_linux_amd64.go:781     0x473dad        488d1dac411400          LEAQ 0x1441ac(IP), BX
        zsyscall_linux_amd64.go:781     0x473db4        4889d8                  MOVQ BX, AX
        zsyscall_linux_amd64.go:783     0x473db7        eb95                    JMP 0x473d4e
        zsyscall_linux_amd64.go:776     0x473db9        e882f0fdff              CALL runtime.morestack_noctxt(SB)
        zsyscall_linux_amd64.go:776     0x473dbe        e94dffffff              JMP syscall.read(SB)
```


# 系统调用

上面只是用cpu寄存器，系统调用则这要和linux配合。




入口在这里 
syscall/asm_linux_amd64.s

```
func Syscall(trap int64, a1, a2, a3 int64) (r1, r2, err int64)
	CALL	runtime·entersyscall(SB)
	...
	SYSCALL
	...
	CALL	runtime·exitsyscall(SB)

func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2, err uintptr)
func RawSyscall6。。。
```

看最常用的：
runtime·entersyscall 和 runtime·exitsyscall相关参见 [调度](schedule.md) 一节

1. 调用 syscall.Syscall和普通go函数没区别，传参方式也没区别。
2. Syscall 函数根据 linux 内核 syscall 对 amd64 寄存器的规则将堆栈上的数据存入寄存器，调用syscall
3. linux 系统调用 如果出错返回一负数。 CMPQ 和 JLS指令检查ax中的返回值如果是负数，就把错误码放到 err中，否则把AX，DX分别放到 r1, r2， err = 0。 对于linux amd64，只用AX传返回值，r2 应该实际不会用到。


linux 内核 syscall 对 amd64 寄存器的规则：

1. eax 是系统调用号和返回值
2. 参数 1到6依次放到 rdi, rsi, rdx, r10, r8, r9, 更多参数放到stack中（逆序）
 
![x64_frame_nonleaf](../imgs/x64_frame_nonleaf.png)


```
TEXT	·Syscall(SB),NOSPLIT,$0-56
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	$0, R10
	MOVQ	$0, R8
	MOVQ	$0, R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
	
```

下面是反汇编生成的， 可顺便对比一下: 

```
TEXT syscall.Syscall(SB) /usr/local/go/src/syscall/asm_linux_amd64.s
        asm_linux_amd64.s:18    0x474e90        e8dba0fbff              CALL runtime.entersyscall(SB)
        // 为syscall准备参数
        asm_linux_amd64.s:19    0x474e95        488b7c2410              MOVQ 0x10(SP), DI
        asm_linux_amd64.s:20    0x474e9a        488b742418              MOVQ 0x18(SP), SI
        asm_linux_amd64.s:21    0x474e9f        488b542420              MOVQ 0x20(SP), DX
        asm_linux_amd64.s:22    0x474ea4        4531d2                  XORL R10, R10
        asm_linux_amd64.s:23    0x474ea7        4531c0                  XORL R8, R8
        asm_linux_amd64.s:24    0x474eaa        4531c9                  XORL R9, R9
        asm_linux_amd64.s:25    0x474ead        488b442408              MOVQ 0x8(SP), AX
        
        asm_linux_amd64.s:26    0x474eb2        0f05                    SYSCALL
        
        asm_linux_amd64.s:27    0x474eb4        483d01f0ffff            CMPQ $-0xfff, AX
        asm_linux_amd64.s:28    0x474eba        7620                    JBE 0x474edc
        asm_linux_amd64.s:29    0x474ebc        48c7442428ffffffff      MOVQ $-0x1, 0x28(SP)
        asm_linux_amd64.s:30    0x474ec5        48c744243000000000      MOVQ $0x0, 0x30(SP)
        asm_linux_amd64.s:31    0x474ece        48f7d8                  NEGQ AX
        asm_linux_amd64.s:32    0x474ed1        4889442438              MOVQ AX, 0x38(SP)
        asm_linux_amd64.s:33    0x474ed6        e875a5fbff              CALL runtime.exitsyscall(SB)
        asm_linux_amd64.s:34    0x474edb        c3                      RET
        asm_linux_amd64.s:36    0x474edc        4889442428              MOVQ AX, 0x28(SP)
        asm_linux_amd64.s:37    0x474ee1        4889542430              MOVQ DX, 0x30(SP)
        asm_linux_amd64.s:38    0x474ee6        48c744243800000000      MOVQ $0x0, 0x38(SP)
        asm_linux_amd64.s:39    0x474eef        e85ca5fbff              CALL runtime.exitsyscall(SB)
        asm_linux_amd64.s:40    0x474ef4        c3                      RET
        asm_linux_amd64.s:40    0x474ef5        0000                    ADDL AL, 0(AX)
        asm_linux_amd64.s:40    0x474ef7        0000                    ADDL AL, 0(AX)
        asm_linux_amd64.s:40    0x474ef9        0000                    ADDL AL, 0(AX)
        asm_linux_amd64.s:40    0x474efb        0000                    ADDL AL, 0(AX)
        asm_linux_amd64.s:40    0x474efd        0000                    ADDL AL, 0(AX)
        asm_linux_amd64.s:40    0x474eff        00                      ?
```
        
### Syscall6 

是更完整的系统调用，该用的寄存器都用了，但多数用Syscall的三个参数就够了。

```
// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·Syscall6(SB),NOSPLIT,$0-80
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	a4+32(FP), R10
	MOVQ	a5+40(FP), R8
	MOVQ	a6+48(FP), R9
	MOVQ	trap+0(FP), AX	// syscall entry
```

###  RawSyscall 或 RawSyscall6
* 用于不阻塞的系统调用，实现区别就是没有没有 runtime·entersyscall 和 runtime·exitsyscall

```
func socket(domain int, typ int, proto int) (fd int, err error) {
	r0, _, e1 := RawSyscall(SYS_SOCKET, uintptr(domain), uintptr(typ), uintptr(proto))
	fd = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}
```

* syscall/zsyscall_darwin_amd64.go 中的是用perl脚本以 syscall/syscall_linux_amd64.go为输入生成的，依据如下注释，sys和sysnb决定用不用RawXX，再看参数个数决定用不用XX6；另外一些特殊系统调用如 gettimeofday 单独处理。

```
//sys	accept(s int, rsa *RawSockaddrAny, addrlen *_Socklen) (fd int, err error)
//sysnb	socket(domain int, typ int, proto int) (fd int, err error)
```




# 参考

## amd64 ：

0. 官方 [abi](http://x86-64.org/documentation/abi.pdf)， [overview](http://palantir.cs.colby.edu/maxwell/classes/e25/S05/papers/amd64-35-59.pdf)
1. [指令速查](http://www.mathemainzel.info/files/x86asmref.html)
1. [X86-64寄存器和栈帧](http://www.searchtb.com/2013/03/x86-64_register_and_function_frame.html)
2. [stack-frame-layout-on-x86-64](http://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)
3. [Introduction to Compilers](http://www.cs.cornell.edu/courses/cs412/2008sp/lectures/lec20.pdf)

4. [tosee](http://eli.thegreenplace.net/2011/02/04/where-the-top-of-the-stack-is-on-x86/)

### -fomit-frame-pointer
[1](http://www.cnblogs.com/islandscape/p/3444122.html)
[2](http://stackoverflow.com/questions/14666665/trying-to-understand-gcc-option-fomit-frame-pointer)
[3](http://stackoverflow.com/questions/1942801/when-should-i-omit-the-frame-pointer)
	

	
	
## Plan9

[go文档 asm相关]( https://golang.org/doc/asm)

要点：

* rbp(base pointer) 是用作 FP (frame pointer)，会被编译器优化掉（预分配，直接用SP, 通常反汇编看不到）

	
	
## linux

1. [linux系统调用src快速索引](https://filippo.io/linux-syscall-table/)
2. [The Linux kernel](http://www.win.tue.nl/~aeb/linux/lk/lk-4.html)






