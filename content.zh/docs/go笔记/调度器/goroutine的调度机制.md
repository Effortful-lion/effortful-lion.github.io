---
title: "Goroutine的调度机制"
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

# Goroutine 的调度机制(概览)

## 什么是协作式调度？

调度触发完全依赖用户代码显式设置调度点，执行单元仅在调度点主动让出 CPU 控制权，调度器才能完成切换。
- 典型示例：Python 中手动调用 yield 关键字，显式告知解释器 “当前协程可被调度出去”；Lua 协程需调用 coroutine.yield() 主动交出执行权。
- 核心特征：无用户手动设置的调度点时，执行单元会持续占用 CPU，调度器无法干预。

## 什么是抢占式调度？

调度器（通常是操作系统内核）无需用户代码配合，可强制中断执行单元并剥夺 CPU 使用权，触发时机由调度器自主决定。
- 典型示例：操作系统对进程 / 线程的调度，即使代码中无任何让出逻辑，内核也会按时间片（如 10ms）强制切换，保证资源公平分配。
- 核心特征：执行单元无法独占 CPU，调度时机与用户代码无关。

## Goroutine的混合调度

**主 协作式 + 辅 抢占式**

![](/assets/goroutine的调度机制-1768814569635.png)

**为什么说它是“抢占式”？**
- 用户不需要手动写 yield 或调度点，完全感知不到调度的发生。
- 自动调度，由 Runtime 决定
- 事件/时间 抢占条件

# 调度详解 + goroutine的生命周期变化

## Goroutine 创建 -> 执行

### G创建

除了 main 函数这个特例之外，所有用户通过 go func(){...} 操作启动的 goroutine，都会以 g 的形式进入到 gmp 架构当中.

**启动一个 go:**
```go
func handle() {
        // 异步启动 goroutine
	go func(){
		// do something ...
	}()
}
```

**底层代码：**
```go
// 创建一个新的 g，本将其投递入队列. 入参 fn 为用户指定的函数.
// 当前执行方还是某个普通 g
func newproc(fn *funcval) {
	// 获取当前正在执行的普通 g 及其程序计数器（program counter）
	gp := getg()
	pc := getcallerpc()
	// 执行 systemstack 时，会临时切换至 g0，并在完成其中闭包函数调用后，切换回到原本的普通 g 
	systemstack(func() {
		// 此时执行方为 g0
		// 构造一个新的 g 实例
		newg := newproc1(fn, gp, pc)
		
		// 获取当前 p 
		_p_ := getg().m.p.ptr()
		
        /*
            newg的高效调度：
		        - 高优位 —— runnext（有sysmon情况下）
		        - 本地队列
		        - 全局队列
        */
		runqput(_p_, newg, true)
		
		// 如果存在因过度空闲而被 block 的 p 和 m，则需要对其进行唤醒
		/*
		P 和 M 会因 “无可运行的 G” 进入阻塞（block）状态：
		当某个 P 的本地队列、全局队列都无 G，且工作窃取也没找到 G 时，P 会进入空闲阻塞状态，绑定的 M 也会休眠；
		但当新 G 被创建（如 go func()）时，若此时有空闲阻塞的 P/M，直接唤醒它们比创建新 M 成本更低，能提升调度效率。
		 */
		if mainStarted {
			wakep()
		}
	})
        // 切换回到原本的普通 g 继续执行
        // ...
}
```

1. newg的高效调度：
- 高优位 —— runnext（有sysmon情况下）
- 本地队列
- 全局队列
2. 本地队列：
- 环形队列 256 有队头队尾索引
- FIFO 队头获取 队尾追加
3. 全局队列
- 本地队列已满 - 调用runqputslow - 慢路径（将当前待处理 G + 本地队列中一半的 G 批量转移到全局队列）
- 全局队列同样遵循 FIFO，但调度优先级低于 P 的 runnext（高优槽位） 和本地队列。
```go
//runqput 尝试将 g 放入本地可运行队列。
// 若 next 为 false，runqput 会将 g 添加到可运行队列的尾部。
// 若 next 为 true，runqput 会将 g 放入 pp.runnext 插槽。
// 若运行队列已满，runnext 会将 g 放入全局队列。
// 仅由所属 P 执行。
func runqput(pp *p, gp *g, next bool) {
	if !haveSysmon && next {
    // 一个runnext协程与当前协程共享相同的时间片（从runqget继承时间）。 
	//为了防止一对频繁切换的协程导致其他协程饿死，我们依靠sysmon来抢占“长时间运行的协程”。
	//也就是说，任何共享相同时间片的协程集合都是如此。
    //
    //如果没有sysmon，我们必须完全避免使用runnext，否则会有饿死的风险。next = false
	}
	if randomizeScheduler && next && randn(2) == 0 {
		next = false
	}
	
	if next {
		retryNext:oldnext := pp.runnext
	    if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
	    	goto retryNext
	    }
	    if oldnext == 0 {
	    	return
	    }
	    // Kick the old runnext out to the regular run queue.
	    gp = oldnext.ptr()
	}
	
	retry:h := atomic.LoadAcq(&pp.runqhead) 
	// load-acquire, synchronize with consumers
	t := pp.runqtail
	if t-h < uint32(len(pp.runq)) {
		pp.runq[t%uint32(len(pp.runq))].set(gp)
	    atomic.StoreRel(&pp.runqtail, t+1) 
		// store-release, makes the item available for consumption
	    return
	}
	if runqputslow(pp, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```
### G运行

**对一个 m 来说，其运行周期就是处在 g0 与 g 之间轮换交替的过程中**
```go
type m struct {
	// 用于寻找并调度普通 g 的特殊 g，与每个 m 一一对应
	g0      *g     
	// ...
	// m 上正在运行的普通 g
	curg          *g       
	// ...
}
```
**在 m 运行中，能够通过几个桩方法实现 g0 与 g 之间执行权的切换:**

g -> g0：mcall()、systemstack()
g0 -> g：gogo()

```go
// 从 g 切换至 g0 执行. 只允许在 g 中调用
func mcall(fn func(*g))

// 在普通 g 中调用时，会切换至 g0 压栈执行 fn，执行完成后切回到 g
func systemstack(fn func())

// 从 g0 切换至 g 执行. gobuf 包含 g 运行上下文信息(并不是直接调用的)
func gogo(buf *gobuf)
```

**g0 -> g:**
```go
//schedule：调用 findRunnable 方法，获取到可执行的 g
//execute：更新 g 的上下文信息，调用 gogo 方法，将 m 的执行权由 g0 切换到 g

// 调度器主循环，执行方为 g0（每个 M 绑定的管家 Goroutine）
// 核心逻辑：循环查找可运行的 G → 切换执行权至该 G
func schedule() {
    // 获取当前执行的 g0
    _g_ := getg()
    
    top: // 调度循环入口，通过 goto 实现循环
    // 获取当前 M 绑定的 P
    pp := _g_.m.p.ptr()

	/*
		核心：查找可运行的 Goroutine，优先级如下：
		1. 从当前 P 的本地队列（lrq）取 G
		2. 从全局队列（grq）取 G
		3. 检查 netpoll（网络 IO 就绪的 G）
		4. 从其他 P 的本地队列窃取 G（work stealing）
		5. 若无可用 G，将 P/M 阻塞并加入空闲队列，直到有新 G 被创建
	*/
	gp, inheritTime, tryWakeP := findRunnable() // 阻塞直到找到可运行的 G

	// 执行找到的 G，将执行权从 g0 切换至目标 G
	execute(gp, inheritTime)

	// 执行完后通过 goto top 重新进入调度循环（reenter）
	goto top
}

// execute 执行指定的 Goroutine，当前执行方仍为 g0，通过 gogo 切换至目标 G
// gp: 待执行的用户 Goroutine
// inheritTime: 是否继承上一个 G 的剩余时间片（runnext 中的 G 会继承）
func execute(gp *g, inheritTime bool) {
    // 获取当前执行的 g0
    _g_ := getg()

	/* 建立 M 与 G 的绑定关系 */
	// 1. 将 M 的当前执行 G（curg）指向目标 G
	_g_.m.curg = gp
	// 2. 将目标 G 的绑定 M 指向当前 M
	gp.m = _g_.m

	/* 更新 G 的状态：从可运行（Grunnable）转为运行中（Grunning） */
	casgstatus(gp, _Grunnable, _Grunning)

	/* 设置栈保护边界，防止栈溢出 */
	gp.stackguard0 = gp.stack.lo + _StackGuard

	/*
		核心：执行 gogo 汇编函数，完成执行权切换
		- 保存 g0 的上下文（寄存器、程序计数器等）
		- 加载目标 G（gp）的上下文
		- M 的执行权正式从 g0 切换至 gp，开始执行用户代码
	*/
	gogo(&gp.sched)
}
```

**findRunnable 方法声明于 runtime/proc.go 中，其核心步骤包括：**

- **防饿死检查：**每经历 61 次调度后，需要先处理一次全局队列 grq（globrunqget——加锁），避免产生饥饿；
- **本地队列优先：**尝试从本地队列 lrq 中获取 g（runqget——CAS 无锁）
- **全局队列兜底：**尝试从全局队列 grq 获取 g（globrunqget——加锁）
- **IO就绪G：**尝试获取 io 就绪的 g（netpoll——非阻塞模式）
- **工作窃取：**尝试从其他 p 的 lrq 窃取 g（stealwork）
- **double check：**double check 一次 grq（globrunqget——加锁）
- **资源释放：**若没找到 g，将 p 置为 idle 状态，添加到 schedt pidle 队列（动态缩容）
- **IO事件不丢失：**强制保留至少一个 M 以阻塞模式运行 netpoll，专门监听网络 IO 就绪事件，确保因 IO 阻塞的 G 能被及时唤醒
- **资源释放：**若 m 仍无事可做，则将其添加到 schedt midle 队列（动态缩容）
- **资源释放：**暂停 m（回收资源）

```go
// findRunnable 查找可执行的 Goroutine，返回时必定已找到有效 G
// 返回值说明：
//   gp: 找到的可运行 G
//   inheritTime: 是否继承上一个 G 的剩余时间片（runnext 中的 G 会继承）
//   tryWakeP: 是否需要唤醒空闲的 P
func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
    // 获取当前 M 绑定的 g0（执行方始终是 g0）
    _g_ := getg()
    
    top: // 循环入口，未找到 G 则通过 goto 重试
    // 获取当前 M 绑定的 P
    _p_ := _g_.m.p.ptr()

	/* 1. 每61次调度检查全局队列（防止全局队列G饿死） */
	if _p_.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp = globrunqget(_p_, 1) // 从全局队列取1个G
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

	/* 2. 优先从当前 P 的本地队列（lrq）取 G（最高优先级） */
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime, false
	}

	/* 3. 从全局队列（grq）取 G（兜底） */
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp = globrunqget(_p_, 0) // 批量取 G（默认取 GOMAXPROCS 个）
		unlock(&sched.lock)
		if gp != nil {
			return gp, false, false
		}
	}

	/* 4. 检查网络 IO 就绪的 G（netpoll） */
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		// 非阻塞式检查 IO 就绪事件
		if list := netpoll(0); !list.empty() {
			gp = list.pop()          // 取首个就绪 G
			injectglist(&list)       // 剩余 G 注入全局队列
			casgstatus(gp, _Gwaiting, _Grunnable) // 更新 G 状态为可运行
			return gp, false, false
		}
	}

	/* 5. 工作窃取（stealWork）：从其他 P 的本地队列偷 G */
	gp, inheritTime, _, _, _ := stealWork(nil) // 简化入参，保留核心逻辑
	if gp != nil {
		return gp, inheritTime, false
	}

	/* 6. 检查 GC 并发标记任务（空闲时参与 GC，避免浪费 P 资源） */
	// 省略 GC 相关逻辑（核心是无 G 时优先协助 GC，而非直接释放 P）

	/* 7. 二次检查全局队列（加锁保证一致性） */
	lock(&sched.lock)
	if sched.runqsize != 0 {
		gp = globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false, false
	}

	/* 8. 无可用 G：释放 P 到空闲队列，M 进入休眠准备 */
	// 解绑 M 与 P
	releasep()
	// 将 P 加入全局空闲 P 队列（pidle）
	pidleput(_p_, nil)
	unlock(&sched.lock)

	/* 9. 阻塞式 netpoll：等待 IO 就绪事件（保证 IO 不丢失） */
	if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || atomic.Load64(&sched.pollUntil) != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		atomic.Store64(&sched.pollUntil, 0) // 重置 poll 超时时间
		// 阻塞式 netpoll，直到有 IO 就绪事件
		list := netpoll(-1)
		atomic.Store64(&sched.lastpoll, uint64(nil)) // 恢复 lastpoll 标记

		lock(&sched.lock)
		// 从空闲 P 队列获取一个 P
		_p_, _ = pidleget(nil)
		unlock(&sched.lock)

		if _p_ == nil {
			// 无空闲 P：将 IO 就绪的 G 全部注入全局队列
			injectglist(&list)
		} else {
			// 绑定 M 与 P
			acquirep(_p_)
			if !list.empty() {
				// 取首个 IO 就绪 G 直接调度，剩余注入全局队列
				gp = list.pop()
				injectglist(&list)
				casgstatus(gp, _Gwaiting, _Grunnable)
				return gp, false, false
			}
			// 无就绪 G，回到循环顶部重新查找
			goto top
		}
	}

	/* 10. 无任何可运行 G：将 M 加入空闲队列（midle）并休眠 */
	stopm() // 阻塞 M，直到有新 G 创建时被唤醒
	goto top // 被唤醒后重新进入查找循环
}
```

**runqget ：从 高优执行槽位 + 本地队列 获取 g**

```go
//runnext：是一个单个 G 的 “快速槽”，仅能存放 1 个 G；
//本地队列（runq）：是环形数组，最多存放 256 个 G，通过 runqhead/runqtail 维护队列读写。
type p struct {
    // runnext：单个 G 的指针，高优执行槽位（仅存 1 个 G）
    runnext guintptr
    // 本地队列：固定容量 256 的环形数组（存多个 G）
    runq     [256]guintptr
    runqhead uint32         // 本地队列头指针
    runqtail uint32         // 本地队列尾指针
    // ... 其他字段
}
```

- 以 CAS 操作取 runnext 位置的 g，获取成功则返回
- 以 CAS 操作移动 lrq 的头节点索引，然后返回头节点对应 g

```go
// [无锁化]从某个 p 的本地队列 lrq 中获取 g
func runqget(_p_ *p) (gp *g, inheritTime bool) {
	// 首先尝试获取特定席位 runnext 中的 g，使用 cas 操作
	next := _p_.runnext
	if next != 0 && _p_.runnext.cas(next, 0) {
		return next.ptr(), true
	}

        // 尝试基于 cas 操作，获取本地队列头节点中的 g
	for {
                // 获取头节点索引
		h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
		// 获取尾节点索引
                t := _p_.runqtail
                // 头尾节点重合，说明 lrq 为空
		if t == h {
			return nil, false
		}
                // 根据索引从 lrq 中取出头节点对应的 g
		gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
                // 通过 cas 操作更新头节点索引
		if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
			return gp, false
		}
	}
}
```

**globrunqget： 从全局队列中获取g**

```go
// 从全局队列 grq 中获取 g. 调用此方法前必须持有 schedt 中的互斥锁 lock
func globrunqget(_p_ *p, max int32) *g {
        // 断言确保持有锁
	assertLockHeld(&sched.lock)
        // 队列为空，直接返回
	if sched.runqsize == 0 {
		return nil
	}

        // ...
	// 此外还有一些逻辑是根据传入的 max 值尝试获取 grq 中的半数 g 填充到 p 的 lrq 中. 此处不展开
	// ...

        // 从全局队列的队首弹出一个 g
	gp := sched.runq.pop()
	// ...
	return gp
}
```

**netpoll:获取 io 就绪的 g**

```go
func netpoll(delay int64) gList {
	// ...
        // 调用 epoll_wait 获取就绪的 io event
	var events [128]epollevent
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	// ...
	var toRun gList
	for i := int32(0); i < n; i++ {
		ev := &events[i]
                // 将就绪 event 对应 g 追加到的 glist 中
		netpollready(...)
	}
	return toRun
}
```

**stealWork:从其他 p 窃取 g**

```go
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
	// 获取当前 p
        pp := getg().m.p.ptr()
        // ...
        // 外层循环 4 次
	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		// ...
                // 通过随机数以随机起点随机步长选取目标 p 进行窃取
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			// ...
                        // 获取拟窃取的目标 p
			p2 := allp[enum.position()]
                        // 如果目标 p 是当前 p，则跳过
			if pp == p2 {
				continue
			}

			// ...
			// 只要目标 p 不为 idle 状态，则进行窃取
			if !idlepMask.read(enum.position()) {
                                // 窃取目标 p，其中会尝试将目标 p lrq 中半数 g 窃取到当前 p 的 lrq 中
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}
	}

	// 窃取失败 未找到合适的目标
	return nil, false, now, pollUntil, ranTimer
}
```

**pidleput:回收空闲的 p 和 m**

```go
// 将 p 追加到 schedt pidle 队列中
func pidleput(_p_ *p, now int64) int64 {
	assertLockHeld(&sched.lock)
	// ...
        // p 指针指向原本 pidle 队首
	_p_.link = sched.pidle
        // 将 p 设置为 pidle 队首
	sched.pidle.set(_p_)
	atomic.Xadd(&sched.npidle, 1)
	// ...
}

// 将当前 m 添加到 schedt midle 队列并停止 m
func stopm() {
	_g_ := getg()

	// ...
	lock(&sched.lock)
        // 将 m 添加到 schedt.mdile 
	mput(_g_.m)
	unlock(&sched.lock)
        // 停止 m
	mPark()
        // ...
}
```

## Goroutine 从运行到让渡（协作式）

敬请期待......

## Goroutine 从运行被抢占（抢占式）

敬请期待......



