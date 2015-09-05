
[TOC]

# 同步操作


## sudog

runtime/runtime2.go

```
type sudog struct {
	g           *g
	selectdone  *uint32
	next        *sudog
	prev        *sudog
	elem        unsafe.Pointer // data element
	releasetime int64
	nrelease    int32  // -1 for acquire
	waitlink    *sudog // g.waiting list
}
```

1. 它把一个g和一个elem绑在一起，表示这个g等在elem上。
2. next和prev用来构成一个双向链表：他们等在同一个elem上。链表头是runtime内的容器（如semtable）
3. waitlink用来构成一个单向链表，用在select中，他们g相同，elem不同，这个链表头是g.waiting。




## sync.Mutex

mutex 由一个状态和一个信号量组成

```
type Mutex struct {
	state int32
	sema  uint32
}
```

m.Lock -> runtime_Semacquire(&m.sema)

注：这里还用了spinlock, 先略过


### 信号量机制

信号量从使用者角度就是一个uint32的计数器

两处用途：

0. sync.Mutex 
1. runtime中有个全局的worldsema 用来stop the word。


实现：


```
type semaRoot struct {
	lock  mutex
	head  *sudog
	tail  *sudog
	nwait uint32 // Number of waiters. Read w/o the lock.
}

const semTabSize = 251

var semtable [semTabSize]struct {
	root semaRoot
	pad  [_CacheLineSize - unsafe.Sizeof(semaRoot{})]byte
}
```

semtable 整个是个最简单的hash表（固定桶的溢出链表）。

操作简化（不考虑优化）

```

// Called from runtime.
func semacquire(addr *uint32, profile bool)  {
	root := semroot(addr)
	for {
		lock(&root.lock)
		xadd(&root.nwait, 1)
		root.queue(addr, s)
		goparkunlock(&root.lock, "semacquire", traceEvGoBlockSync, 4)
		if cansemacquire(addr) {
			break
		}
	}
}

func semrelease(addr *uint32) {
	root := semroot(addr)
	xadd(addr, 1)
	s := root.head
	for ; s != nil; s = s.next {
		if s.elem == unsafe.Pointer(addr) {
			xadd(&root.nwait, -1)
			root.dequeue(s)
			break
		}
	if s != nil {
		goready(s.g, 4)
	}
```

注意解锁要顺序遍历list，只有251个这样的队列，所以如果很多G，每个都要长时间加锁做事情，队列就会比较长，并且操作每个队列都是要加锁的。


对外: 

```
//go:linkname net_runtime_Semacquire net.runtime_Semacquire
func net_runtime_Semacquire(addr *uint32) {
	semacquire(addr, true)
}
```


## sync.Cond ：syncSema

```
type syncSema struct {
	lock mutex
	head *sudog
	tail *sudog
}
```

## chan


### select