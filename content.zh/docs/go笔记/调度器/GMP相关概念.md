---
title: "调度器其他相关概念"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# scheduler

## 什么是 scheduler

**官方：** 调度器的核心职责是将待运行的 goroutine 分配到工作线程上执行。

他是 runtime 的内置核心实现之一：G/P/M 模型的实现，负责把 Goroutine 映射到操作系统线程和 CPU 核心上执行。

Runtime 维护所有的 goroutines，并通过 scheduler 来进行调度。Goroutines 和 threads 是独立的，但是 goroutines 要依赖 threads 才能执行。

Go 程序执行的高效和 scheduler 的调度是分不开的。

## Scheduler 底层原理

**官方概念说明：**
- G - 协程（goroutine）。
- M - 工作线程（worker thread），也称为机器（machine）。是 golang 中对操作系统线程的抽象。
- P - 处理器（processor），是执行 Go 代码所需的核心资源。
- sched - 全局调度器，绑定全局队列

另：
- M 必须关联一个 P 才能执行 Go 代码；但 M 在阻塞状态下或执行系统调用时，可以不关联任何 P。

# GPM

## GPM 源码

**g:**

```go
type g struct {
	// goroutine 使用的栈
	stack       stack
	// 用于栈的扩张和收缩检查，抢占标志
	stackguard0 uintptr
	stackguard1 uintptr

	// 当前与 g 绑定的 m
	m              *m
	// goroutine 的运行现场
	sched          gobuf
	// wakeup 时传入的参数
	param          unsafe.Pointer
	// g 被阻塞之后的近似时间
	waitsince      int64
	// g 被阻塞的原因
	waitreason     string
	// 指向全局队列里下一个 g
	schedlink      guintptr
	// 抢占调度标志。这个为 true 时，stackguard0 = stackpreempt
	preempt        bool
	// 如果调用了 LockOsThread，那么这个 g 会绑定到某个 m 上
	lockedm        *m
	// 创建该 goroutine 的语句的指令地址
	gopc           uintptr
	// goroutine 函数的指令地址
	startpc        uintptr
	// time.Sleep 缓存的定时器
	timer          *timer
	
	// ... 还有
}
```
```go
// 描述栈的数据结构，栈的范围：[lo, hi)
type stack struct {
    // 栈顶，低地址
	lo uintptr
	// 栈低，高地址
	hi uintptr
}
```
```go
type gobuf struct {
	// 存储 rsp 寄存器的值
	sp   uintptr
	// 存储 rip 寄存器的值
	pc   uintptr
	// 指向 goroutine
	g    guintptr
	ctxt unsafe.Pointer // this has to be a pointer so that gc scans it
	// 保存系统调用的返回值
	ret  sys.Uintreg
	lr   uintptr
	bp   uintptr // for GOEXPERIMENT=framepointer
}
```

**m：**
1. 代表一个工作线程，或者说系统线程(系统线程的抽象，用户级)
2. 保存了 M 自身使用的栈信息、当前正在 M 上执行的 G 信息、与之绑定的 P 信息……
```go
type m struct {
	// 记录工作线程（也就是内核线程）使用的栈信息。在执行调度代码时需要使用
	// 执行用户 goroutine 代码时，使用用户 goroutine 自己的栈，因此调度时会发生栈的切换
	g0      *g
	// 通过 tls 结构体实现 m 与工作线程的绑定
	// 这里是线程本地存储
	tls           [6]uintptr
	// 指向正在运行的 goroutine 对象
	curg          *g
	// 当前工作线程绑定的 p
	p             puintptr
	// 该字段不等于空字符串的话，要保持 curg 始终在这个 m 上运行
	preemptoff    string
	// 为 true 时表示当前 m 处于自旋状态，正在从其他线程偷工作
	spinning      bool
	// m 正阻塞在 note 上
	blocked       bool
	// 正在执行 cgo 调用
	incgo         bool
	// cgo 调用总计数
	ncgocall      uint64
	// 没有 goroutine 需要运行时，工作线程睡眠在这个 park 成员上，
	// 其它线程通过这个 park 唤醒该工作线程
	park          note
	// 记录所有工作线程的链表
	alllink       *m
	// 工作线程 id
	thread        uintptr
}
```

**p**
为 M 的执行提供“上下文”，保存 M 执行 G 时的一些资源，例如本地可运行 G 队列，memeory cache 等。

一个 M 只有绑定 P 才能执行 goroutine，当 M 被阻塞时，整个 P 会被传递给其他 M ，或者说整个 P 被接管。
```go
type p struct {
	// 在 allp 中的索引
	id          int32
	// 每次调用 schedule 时会加一
	schedtick   uint32
	// 每次系统调用时加一
	syscalltick uint32
	// 用于 sysmon 线程记录被监控 p 的系统调用时间和运行时间
	sysmontick  sysmontick
	// 指向绑定的 m，如果 p 是 idle 的话，那这个指针是 nil
	m           muintptr
    // 内存管理部分：本地内存缓存（mcache 为每个 P 缓存了小对象内存块（span），让 P 在分配小对象时优先从本地缓存获取，无需加锁访问全局堆。）
	mcache      *mcache
	// 本地可运行的队列，不用通过锁即可访问
	runqhead uint32 // 队列头
	runqtail uint32 // 队列尾
	// 使用数组实现的循环队列
	runq     [256]guintptr
	
	// runnext 非空时，代表的是一个 runnable 状态的 G，
	// 这个 G 被 当前 G 修改为 ready 状态，相比 runq 中的 G 有更高的优先级。
	// 如果当前 G 还有剩余的可用时间，那么就应该运行这个 G
	// 运行之后，该 G 会继承当前 G 的剩余时间
	runnext guintptr

	// 空闲的 g
	gfree    *g
	gfreecnt int32
}
```

**sched**

```go
// 保存调度器的信息
type schedt struct {
	// accessed atomically. keep at top to ensure alignment on 32-bit systems.
	// 需以原子访问访问。
	// 保持在 struct 顶部，以使其在 32 位系统上可以对齐
	goidgen  uint64
	lastpoll uint64

	lock mutex

	// 由空闲的工作线程组成的链表
	midle        muintptr // idle m's waiting for work
	// 空闲的工作线程数量
	nmidle       int32    // number of idle m's waiting for work
	// 空闲的且被 lock 的 m 计数
	nmidlelocked int32    // number of locked m's waiting for work
	// 已经创建的工作线程数量
	mcount       int32    // number of m's that have been created
	// 表示最多所能创建的工作线程数量
	maxmcount    int32    // maximum number of m's allowed (or die)

	// goroutine 的数量，自动更新
	ngsys uint32 // number of system goroutines; updated atomically

	// 由空闲的 p 结构体对象组成的链表
	pidle      puintptr // idle p's
	// 空闲的 p 结构体对象的数量
	npidle     uint32
	nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

	// Global runnable queue.
	// 全局可运行的 G队列
	runqhead guintptr // 队列头
	runqtail guintptr // 队列尾
	runqsize int32 // 元素数量

	// Global cache of dead G's.
	// dead G 的全局缓存
	// 已退出的 goroutine 对象，缓存下来
	// 避免每次创建 goroutine 时都重新分配内存
	gflock       mutex
	gfreeStack   *g
	gfreeNoStack *g
	// 空闲 g 的数量
	ngfree       int32

	// Central cache of sudog structs.
	// sudog 结构的集中缓存
	sudoglock  mutex
	sudogcache *sudog

	// Central pool of available defer structs of different sizes.
	// 不同大小的可用的 defer struct 的集中缓存池
	deferlock mutex
	deferpool [5]*_defer

	gcwaiting  uint32 // gc is waiting to run
	stopwait   int32
	stopnote   note
	sysmonwait uint32
	sysmonnote note

	// safepointFn should be called on each P at the next GC
	// safepoint if p.runSafePointFn is set.
	safePointFn   func(*p)
	safePointWait int32
	safePointNote note

	profilehz int32 // cpu profiling rate

	// 上次修改 gomaxprocs 的纳秒时间
	procresizetime int64 // nanotime() of last change to gomaxprocs
	totaltime      int64 // ∫gomaxprocs dt up to procresizetime
}
```

在程序运行过程中，schedt 对象只有一份实体，它维护了调度器的所有信息。

在 proc.go 和 runtime2.go 文件中，有一些很重要全局的变量，我们先列出来：
```go
// 所有 g 的长度
allglen     uintptr

// 保存所有的 g
allgs    []*g

// 保存所有的 m
allm        *m

// 保存所有的 p，_MaxGomaxprocs = 1024
allp        [_MaxGomaxprocs + 1]*p

// p 的最大值，默认等于 ncpu
gomaxprocs  int32

// 程序启动时，会调用 osinit 函数获得此值
ncpu        int32

// 调度器结构体对象，记录了调度器的工作状态
sched       schedt

// 代表进程的主线程
m0           m

// m0 的 g0，即 m0.g0 = &g0
g0           g
```
在程序初始化时，这些全局变量都会被初始化为零值：指针被初始化为 nil 指针，切片被初始化为 nil 切片，int 被初始化为 0，结构体的所有成员变量按其类型被初始化为对应的零值。

因此程序刚启动时 allgs，allm 和allp 都不包含任何 g，m 和 p。

不仅是 Go 程序，系统加载可执行文件大概都会经过这几个阶段：

1. 从磁盘上读取可执行文件，加载到内存
2. 创建进程和主线程
3. 为主线程分配栈空间
4. 把由用户在命令行输入的参数拷贝到主线程的栈
5. 把主线程放入操作系统的运行队列等待被调度

## GPM 状态流转

**G**

![](/assets/GMP相关概念-1768825254006.png)
说明一下，上图省略了一些垃圾回收的状态。
- newproc	创建新 Goroutine，将其从 _Gidle 或 _Gdead 状态转为 _Grunnable，并放入调度队列。
- execute	调度器将可运行 Goroutine（_Grunnable）切换到执行状态（_Grunning），完成 G0 到用户 G 的执行权切换。
- goexit	Goroutine 正常执行完毕时调用，将其从 _Grunning 转为 _Gdead，回收栈和资源，可被后续 newproc 复用。
- Gosched	Goroutine 主动让出 CPU，从 _Grunning 回到 _Grunnable，等待下一次调度。
- entersyscall	Goroutine 进入系统调用时触发，状态从 _Grunning 转为 _Gsyscall，Runtime 会解绑当前 P 并分配给其他 M。
- exitsyscall	系统调用结束时触发，尝试重新获取 P： 成功 → 状态从 _Gsyscall 直接回到 _Grunning 失败 → 回到 _Grunnable 等待调度
- exitsyscall0	系统调用结束但未获取到 P 时的分支处理，将 Goroutine 从 _Gsyscall 转为 _Grunnable 放入全局队列。
- park_m	将 Goroutine 从 _Grunning 或 _Gsyscall 转为 _Gwaiting，使其进入休眠状态，等待被唤醒。
- ready	唤醒休眠的 Goroutine，从 _Gwaiting 转为 _Grunnable，重新加入调度队列。

**P**

通常情况下（在程序运行时不调整 P 的个数），P 只会在上图中的四种状态下进行切换。 当程序刚开始运行进行初始化时，所有的 P 都处于 _Pgcstop 状态， 随着 P 的初始化（runtime.procresize），会被置于 _Pidle。

当 M 需要运行时，会 runtime.acquirep 来使 P 变成 Prunning 状态，并通过 runtime.releasep 来释放。

当 G 执行时需要进入系统调用，P 会被设置为 _Psyscall， 如果这个时候被系统监控抢夺（runtime.retake），则 P 会被重新修改为 _Pidle。

如果在程序运行中发生 GC，则 P 会被设置为 _Pgcstop， 并在 runtime.startTheWorld 时重新调整为 _Prunning。

![](/assets/GMP相关概念-1768825260447.png)
- acquirep	M 绑定空闲的 P（_Pidle），将其状态转为 _Prunning，开始执行 Go 代码。
- releasep	M 解绑当前绑定的 P，将其状态从 _Prunning 转为 _Pidle，放入空闲 P 队列。
- retake	sysmon 线程检测到 P 绑定的 M 进入系统调用或长时间运行时，强制回收 P： 从 _Psyscall / _Prunning 转为 _Pidle， 保证 P 资源不被长时间占用。
- entersyscall	Goroutine 进入系统调用时，P 状态从 _Prunning 转为 _Psyscall，等待系统调用结束或被 retake 回收。
- exitsyscall	系统调用结束时，若重新获取到 P，状态从 _Psyscall 回到 _Prunning；否则转为 _Pidle。
- proresize	动态调整 P 的数量（如修改 GOMAXPROCS），将 P 从 _Pgstop 转为 _Pidle 或反之。
- startTheWorld	GC 结束后恢复所有 P，将其从 _Pgstop 唤醒并转为 _Pidle/_Prunning，恢复调度。
- GC（红色事件）	触发垃圾回收时，Runtime 会暂停所有执行中的 P，将其状态转为 _Pgstop，直到 GC 结束。

**M**

![](/assets/GMP相关概念-1768825268595.png)
M 只有自旋和非自旋两种状态。自旋的时候，会努力找工作；找不到的时候会进入非自旋状态，之后会休眠，直到有工作需要处理时，被其他工作线程唤醒，又进入自旋状态。

## GPM 工作流程

### GPM 和 操作系统
![](/assets/GMP相关概念-1768816380798.png)

### GPM 工作流图
![](/assets/GMP相关概念-1768816936367.png)

另外：还有 G0 的角色(上面画的太复杂，抽离出来)

![](/assets/GMP相关概念-1768816284776.png)

# M:N 模型

![](/assets/scheduler-1768815327767.png)

- Runtime 会在程序启动的时候，创建 M 个线程（CPU 执行调度的单位），之后创建的 N 个 goroutine 都会依附在这 M 个线程上执行。这就是 M:N 模型：
- 在同一时刻，一个线程上只能跑一个 goroutine。（当 goroutine 发生阻塞（例如上篇文章提到的向一个 channel 发送数据，被阻塞）时，runtime 会把当前 goroutine 调度走，让其他 goroutine 来执行。目的就是不让一个线程闲着，榨干 CPU 的每一滴油水。）这也就是GO Scheduler的作用：充分发挥多核优势

# GMP的生态工具

将 goroutine 统一为整个语言层面的并发粒度，并遵循着 gmp 的秩序进行运作。在此基础之上，紧密围绕着 gmp 理念打造设计的一系列工具、模块，比如：

## 内存管理

![](/assets/GMP相关概念-1768817242735.png)

golang 的内存管理模块主要继承自 TCMalloc（Thread-Caching-Malloc）的设计思路，其中由契合 gmp 模型做了因地制宜的适配改造，为每个 p 准备了一份私有的高速缓存——mcache，能够无锁化地完成一部分 p 本地的内存分配操作.

## 并发工具

在 golang 中的并发工具（例如锁 mutex、通道 channel 等）均契合 gmp 作了适配改造，保证在执行阻塞操作时，会将阻塞粒度限制在 g（goroutine）而非 m（thread）的粒度，使得阻塞与唤醒操作都属于用户态行为，无需内核的介入，同时一个 g 的阻塞也完全不会影响 m 下其他 g 的运行.

## io多路复用

在设计 io 模型时，golang 采用了 linux 系统提供的 epoll 多路复用技术，然而为了因为 epoll_wait 操作而引起 m（thread）粒度的阻塞，golang 专门设计一套 netpoll 机制，使用用户态的 gopark 指令实现阻塞操作，使用非阻塞 epoll_wait 结合用户态的 goready 指令实现唤醒操作，从而将 io 行为也控制在 g 粒度，很好地契合了 gmp 调度体系.





