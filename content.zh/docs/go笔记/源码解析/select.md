---
title: "Select"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: true
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Select 的实现原理

## 原理类似

```text
很多 C 语言或者 Unix 开发者听到 select 想到的都是系统调用，而谈到 I/O 模型时最终大都会提到基于 select、poll 和 epoll 等函数构建的 IO 多路复用模型，我们在这一节中即将介绍的 Go 语言中的 select 关键字其实就与 C 语言中的 select 有比较相似的功能。
这一节会介绍 Go 语言中的 select 的实现原理，包括 select 的结构和常见问题、编译期间的多种优化以及运行时的执行过程。
```

```text
C 语言中的 select 关键字可以同时监听多个文件描述符的可读或者可写的状态，在文件描述符发生状态改变之前，select 会一直阻塞当前的线程，Go 语言中的 select 关键字与 C 语言中的有些类似，只是它能够让一个 Goroutine 同时等待多个 Channel 达到准备状态。
```

```text
select 是一种与 switch 非常相似的控制结构，与 switch 不同的是，select 中虽然也有多个 case，但是这些 case 中的表达式都必须与 Channel 的操作有关，也就是 Channel 的读写操作
```

## 结构

![](/assets/select-1769240712319.png)

## 特点

1. 非阻塞读写：default（程序不阻塞执行default） + select选择读写成功的进行执行（选择成功 true 失败 false）
```go
func main() {
	ch := make(chan int)
	select {
	case i := <-ch:
		println(i)
	case ch <- 3:
	    println("发送成功")
	default:
		println("非阻塞执行")
	}
}
```

2. 多个channel就绪，随机选择一个执行：
```go

```

# Go 语言 select 语句的随机选择算法

## 核心算法：Fisher-Yates 洗牌算法的变体

Go 语言 `select` 语句的随机选择机制基于 **Fisher-Yates 洗牌算法**的变体，结合快速随机数生成实现。


## 算法基本原理

### 1. 随机数生成与范围映射
- **基础随机数**：使用 `cheaprand()` 生成 32 位随机数
- **范围映射**：通过 `cheaprandn(n)` 函数将随机数映射到 `[0, n-1]` 范围，避免昂贵的除法运算
    - 原理：利用 64 位乘法的高位分布特性（`(x * n) >> 32`），参考 Lemire 算法

### 2. 洗牌过程
通过原地交换构建随机轮询顺序（`pollorder` 数组）：
1. 初始时 `norder = 0`（已处理 case 数为 0）
2. 遍历每个有效的 case（非 nil channel）
3. 生成 `[0, norder]` 范围内的随机位置 `j`
4. 将当前已处理的最后一个 case（位置 `norder`）移动到位置 `j`
5. 将当前 case 放入位置 `norder`
6. 递增 `norder`，继续处理下一个 case

### 3. 随机选择执行
- **轮询阶段**：按照随机化的 `pollorder` 顺序遍历所有 case
- **选择机制**：遇到第一个可执行的 case 立即执行（不再检查后续 case）
- **阻塞唤醒**：若需阻塞，被唤醒时处理唤醒的 case（由 channel 操作决定，非随机）


### 实现步骤（基于 `selectgo` 函数）

#### 1. 构建随机轮询顺序（pollorder）
```go
norder := 0
for i := range scases {
    cas := &scases[i]
    if cas.c == nil {
        continue // 跳过无 channel 的 case
    }
    // 生成 [0, norder] 范围内的随机位置
    j := cheaprandn(uint32(norder + 1))
    // 交换元素，构建随机顺序
    pollorder[norder] = pollorder[j]
    pollorder[j] = uint16(i)
    norder++
}
pollorder = pollorder[:norder] // 截取有效长度
```

#### 2. 轮询选择可执行 case
```go
// 按随机顺序轮询，选择第一个可执行的 case
for _, casei := range pollorder {
    casi = int(casei)
    cas = &scases[casi]
    c = cas.c
    // 检查是否可执行（发送/接收/缓冲区状态）
    // 若可执行，直接处理并返回
}
```


### 关键技术点

1. **Fisher-Yates 洗牌**：确保每个 case 有平等的机会被置于轮询顺序的任意位置
2. **Lemire 快速范围映射**：通过 `(x * n) >> 32` 避免除法运算，提升性能
3. **顺序轮询**：一旦轮询顺序确定，按顺序检查 case，确保随机性与效率的平衡


### 总结

Go 语言 `select` 的随机选择算法通过 **Fisher-Yates 洗牌算法**构建随机轮询顺序，结合 **Lemire 快速范围映射**生成随机位置，最终实现公平、高效的 case 随机选择。该设计既保证了每个 case 的公平性（避免饥饿），又通过性能优化确保了高频调用场景下的效率。

## 编译期间的优化（代码改写）

select 语句在编译期间会被转换成 OSELECT 节点，每一个 OSELECT 节点都会持有一系列的 OCASE 节点，如果 OCASE 节点的都是空的，就意味着这是一个 default 节点:

上图展示的其实就是 select 在编译期间的结构，每一个 OCASE 既包含了执行条件也包含了满足条件后执行的代码，我们在这一节中就会介绍 select 语句在编译期间进行的优化和转换。
编译器在中间代码生成期间会根据 select 中 case 和 default 编写的不同对控制语句进行优化，这一过程其实都发生在 walkselectcases 函数中，我们在这里会分四种情况分别介绍优化的过程和结果：

     1. select 中不存在任何的 case；
     2. select 中只存在一个 case；
     3. select 中存在两个 case，其中一个 case 是 default 语句；
     4. 通用的 select 条件；

我们会按照这四种不同的情况拆分 walkselectcases 函数并分别介绍不同场景下优化的结果。

### 直接阻塞
```go
select {}
```
处理：直接阻塞
```go
func walkselectcases(cases *Nodes) []*Node {
	n := cases.Len()

	// 调用 block
	if n == 0 {
		return []*Node{mkcall("block", nil, nil)}
	}
	// ...
}

// 阻塞
func block() {
	// 让出 cpu （waitReasonSelectNoCases：没有 case 即 select{}）
    gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1)
}
```

### 独立情况
当 select 里只有一个 case 时，Go 会做优化，把它直接改成等价的 if 逻辑，不再走复杂的 select 调度流程。
```go
//-------------------------只有一个读 case------------------------
select {
case v, ok := <-ch:
    // ...
}

// 优化：
if ch == nil {
    block() // 永久阻塞
}
v, ok := <-ch
// ...

//-------------------------只有一个写 case------------------------
select {
case ch <- x:
// ...
}

if ch == nil {
    block() // 永久阻塞
}
ch <- x
// ...
```

### 非阻塞情况

> 一句话：双 case（含 default）的 select，本质就是 “非阻塞尝试一次操作，成功就执行，失败就走 default”，和直接用 if/else 调用非阻塞收发函数效果一样。

**非阻塞发送：**

```go
select {
case ch <- i:
    // 发送成功
default:
    // 发送失败
}
```

优化为：
```go
if selectnbsend(ch, i) { // 非阻塞尝试发送
    // 发送成功
} else {
    // 发送失败（走 default）
}
```

- selectnbsend 内部调用 chansend(ch, i, false)，false 表示不阻塞；
- 能发就发（缓冲区未满 / 有接收者），不能发就立即返回 false，执行 default。

**非阻塞接收：**

```go
select {
case v := <-ch:
    // 接收成功
default:
    // 接收失败
}
```

优化为：
```go
if selectnbrecv(&v, ch) { // 非阻塞尝试接收
    // 接收成功
} else {
    // 接收失败（走 default）
}
```
- 若用 case v, ok := <-ch，则用 selectnbrecv2(&v, &ok, ch)，会把 ok 也返回；
- selectnbrecv/selectnbrecv2 内部调用 chanrecv(ch, &v, false)，false 表示不阻塞；
- 能收就收（缓冲区有数据 / 有发送者 / 已关闭），不能收就立即返回 false，执行 default。

### 通用情况

一句话：多 case 的 select，就是先给每个 case 做信息卡片，再用 selectgo 随机选一个可执行的，最后按索引执行对应代码。

1. 准备阶段
   - 把每个 case 都转成一个叫 scase 的结构体，里面存着： 是读还是写
   - 对应的 channel
   - 要发 / 收的数据地址
   - 就像给每个 case 做个 “信息卡片”，方便后续统一处理。
---
2. 选择阶段
   调用运行时函数 selectgo，它会： 遍历所有 scase 卡片，检查哪些 case 能无阻塞执行
   有多个可执行的 → 随机选一个 一个都没有 → 有 default → 选 default;
   没 default → 阻塞，直到有 case 可执行;

   selectgo 会返回两个关键信息：
   chosen：被选中的 case 索引（0、1、2…）
   revOK：如果是读 case，返回 ok 标志（判断 channel 是否关闭）
---
3. 执行阶段
   用 for 循环 + 一堆 if 语句，判断 chosen 是几，就执行对应的 case 代码：
```go
   if chosen == 0 {
   // 执行第0个case
   }
   if chosen == 1 {
   // 执行第1个case
   }
   if chosen == 2 {
   // 执行第2个case
   }
```

### 运行时

selectgo 是会在运行期间运行的函数，这个函数的主要作用就是从 select 控制结构中的多个 case 中选择一个需要执行的 case，随后的多个 if 条件语句就会根据 selectgo 的返回值执行相应的语句。

### 初始化（selectgo）
```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}

	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// sort the cases by Hchan address to get the locking order.
	// ...
	
	sellock(scases, lockorder)

	// ...
}
```

```text
Channel 的轮询顺序是通过 fastrandn 随机生成的，这其实就导致了如果多个 Channel 同时『响应』select 会随机选择其中的一个执行;
而另一个 lockOrder 就是根据 Channel 的地址确定的，根据相同的顺序锁定 Channel 能够避免死锁的发生，最后调用的 sellock 就会按照之前生成的顺序锁定所有的 Channel。
```

详细的程序：
```go
// selectgo 实现 select 语句的核心逻辑。
//
// cas0 指向 [ncases]scase 类型的数组，order0 指向 [2*ncases]uint16 类型的数组（ncases 不得超过 65536）。
// 两者均位于 goroutine 栈上（不受 selectgo 逃逸分析影响）。
//
// 竞态检测构建模式下，pc0 指向 [ncases]uintptr 类型的数组（同样在栈上）；其他模式下置为 nil。
//
// selectgo 返回选中的 scase 索引（与 select{recv,send,default} 调用的序号一致）。
// 若选中的 scase 是接收操作，还会返回是否成功接收到值。
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
	gp := getg() // 获取当前 goroutine
	if debugSelect {
		print("select: cas0=", cas0, "\n")
	}

	// 注：为控制栈大小，scase 数量上限为 65536
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	ncases := nsends + nrecvs       // 总 case 数 = 发送 case 数 + 接收 case 数
	scases := cas1[:ncases:ncases]  // 所有 case 的数组切片
	pollorder := order1[:ncases:ncases]  // 轮询顺序（随机化，用于检测可执行 case）
	lockorder := order1[ncases:][:ncases:ncases] // 加锁顺序（按 channel 地址排序，避免死锁）

	// 竞态检测相关：初始化 case 程序计数器切片
	var pcs []uintptr
	if raceenabled && pc0 != nil {
		pc1 := (*[1 << 16]uintptr)(unsafe.Pointer(pc0))
		pcs = pc1[:ncases:ncases]
	}
	// 获取指定 case 的程序计数器
	casePC := func(casi int) uintptr {
		if pcs == nil {
			return 0
		}
		return pcs[casi]
	}

	// 阻塞性能分析相关：记录起始时间
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// 编译器已对 0/1 个 case + default 的场景做优化，此处处理通用多 case 场景
	// 即使少数 case 的 channel 为 nil，通用逻辑也能正确处理

	// ========== 阶段1：生成随机轮询顺序 ==========
	norder := 0
	allSynctest := true // 是否所有 channel 都是 synctest 类型
	for i := range scases {
		cas := &scases[i]

		// 跳过 channel 为 nil 的 case（无意义，允许 GC 回收 elem）
		if cas.c == nil {
			cas.elem = nil
			continue
		}

		// 检查 synctest channel 合法性
		if cas.c.bubble != nil {
			if getg().bubble != cas.c.bubble {
				fatal("select on synctest channel from outside bubble")
			}
		} else {
			allSynctest = false
		}

		// 触发 channel 定时器（若有）
		if cas.c.timer != nil {
			cas.c.timer.maybeRunChan(cas.c)
		}

		// 随机打乱轮询顺序（保证公平性，避免固定顺序导致的饥饿）
		j := cheaprandn(uint32(norder + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	// 裁剪为有效 case 长度（剔除 nil channel）
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]

	// 设置 goroutine 等待原因（区分普通 select / synctest select）
	waitReason := waitReasonSelect
	if gp.bubble != nil && allSynctest {
		waitReason = waitReasonSynctestSelect
	}

	// ========== 阶段2：生成加锁顺序（按 channel 地址排序，避免死锁） ==========
	// 堆排序保证 O(n log n) 时间复杂度，栈空间恒定
	for i := range lockorder {
		j := i
		c := scases[pollorder[i]].c
		// 堆化：父节点 < 当前节点则交换
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	// 堆排序：调整堆结构
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			// 选择子节点中较大的那个
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}

	// 调试：校验加锁顺序是否正确
	if debugSelect {
		for i := 0; i+1 < len(lockorder); i++ {
			if scases[lockorder[i]].c.sortkey() > scases[lockorder[i+1]].c.sortkey() {
				print("i=", i, " x=", lockorder[i], " y=", lockorder[i+1], "\n")
				throw("select: broken sort")
			}
		}
	}

	// ========== 阶段3：加锁所有涉及的 channel ==========
	sellock(scases, lockorder)

	var (
		sg     *sudog   // 等待的 goroutine 结构体
		c      *hchan   // 当前遍历的 channel
		k      *scase   // 当前 case
		sglist *sudog   // goroutine 等待列表
		sgnext *sudog   // 等待列表下一个元素
		qp     unsafe.Pointer // 缓冲区指针
		nextp  **sudog  // 等待列表指针
	)

	// ========== 阶段4：第一轮检测 - 寻找已就绪的 case ==========
	var casi int               // 选中的 case 索引
	var cas *scase             // 选中的 case 结构体
	var caseSuccess bool       // case 是否执行成功
	var caseReleaseTime int64 = -1 // case 释放时间（性能分析）
	var recvOK bool            // 接收操作是否成功
	for _, casei := range pollorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c

		// 处理接收 case（索引 >= 发送 case 数）
		if casi >= nsends {
			sg = c.sendq.dequeue() // 从发送队列取出等待的 goroutine
			if sg != nil {
				goto recv // 有等待的发送者，跳转到接收逻辑
			}
			if c.qcount > 0 {
				goto bufrecv // 缓冲区有数据，跳转到缓冲区接收逻辑
			}
			if c.closed != 0 {
				goto rclose // channel 已关闭，跳转到关闭接收逻辑
			}
		} else {
			// 处理发送 case（索引 < 发送 case 数）
			if raceenabled {
				racereadpc(c.raceaddr(), casePC(casi), chansendpc)
			}
			if c.closed != 0 {
				goto sclose // channel 已关闭，发送会 panic
			}
			sg = c.recvq.dequeue() // 从接收队列取出等待的 goroutine
			if sg != nil {
				goto send // 有等待的接收者，跳转到发送逻辑
			}
			if c.qcount < c.dataqsiz {
				goto bufsend // 缓冲区未满，跳转到缓冲区发送逻辑
			}
		}
	}

	// 非阻塞模式（有 default）：无就绪 case 则直接返回
	if !block {
		selunlock(scases, lockorder)
		casi = -1
		goto retc
	}

	// ========== 阶段5：第二轮处理 - 所有 case 入队等待 ==========
	if gp.waiting != nil {
		throw("gp.waiting != nil") // 异常：goroutine 等待列表非空
	}
	nextp = &gp.waiting
	// 遍历加锁顺序，将当前 goroutine 加入所有 channel 的等待队列
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog() // 分配 sudog 结构体
		sg.g = gp
		sg.isSelect = true   // 标记为 select 等待
		sg.elem.set(cas.elem) // 设置数据指针
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c.set(c) // 绑定 channel

		// 构建 goroutine 等待链表（按加锁顺序）
		*nextp = sg
		nextp = &sg.waitlink

		// 将 sudog 加入 channel 的发送/接收队列
		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}

		// 触发 channel 定时器（若有）
		if c.timer != nil {
			blockTimerChan(c)
		}
	}

	// ========== 阶段6：阻塞等待被唤醒 ==========
	gp.param = nil
	gp.parkingOnChan.Store(true) // 标记：goroutine 即将阻塞在 channel 上
	// 挂起 goroutine，等待被唤醒
	gopark(selparkcommit, nil, waitReason, traceBlockSelect, 1)
	gp.activeStackChans = false

	// 被唤醒后重新加锁
	sellock(scases, lockorder)

	// 清理等待状态
	gp.selectDone.Store(0)
	sg = (*sudog)(gp.param) // 获取唤醒当前 goroutine 的 sudog
	gp.param = nil

	// ========== 阶段7：第三轮处理 - 清理未就绪的 case ==========
	casi = -1
	cas = nil
	caseSuccess = false
	sglist = gp.waiting
	// 清空所有 sudog 的 elem 和 channel 指针（避免 GC 泄漏）
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem.set(nil)
		sg1.c.set(nil)
	}
	gp.waiting = nil

	// 遍历加锁顺序，清理未选中 case 的等待队列
	for _, casei := range lockorder {
		k = &scases[casei]
		if k.c.timer != nil {
			unblockTimerChan(k.c)
		}
		if sg == sglist {
			// 当前 sglist 是唤醒当前 goroutine 的 case，记录信息
			casi = int(casei)
			cas = k
			caseSuccess = sglist.success
			if sglist.releasetime > 0 {
				caseReleaseTime = sglist.releasetime
			}
		} else {
			// 从 channel 队列中移除未选中的 sudog
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		// 释放 sudog 资源
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	// 异常：唤醒后未找到对应的 case
	if cas == nil {
		throw("selectgo: bad wakeup")
	}
	c = cas.c

	// 调试日志
	if debugSelect {
		print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " send=", casi < nsends, "\n")
	}

	// 标记接收操作是否成功
	if casi < nsends {
		if !caseSuccess {
			goto sclose // 发送 case 失败（channel 已关闭）
		}
	} else {
		recvOK = caseSuccess
	}

	// 竞态检测/内存检测相关
	if raceenabled {
		if casi < nsends {
			raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
		} else if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
	}
	if msanenabled {
		if casi < nsends {
			msanread(cas.elem, c.elemtype.Size_)
		} else if cas.elem != nil {
			msanwrite(cas.elem, c.elemtype.Size_)
		}
	}
	if asanenabled {
		if casi < nsends {
			asanread(cas.elem, c.elemtype.Size_)
		} else if cas.elem != nil {
			asanwrite(cas.elem, c.elemtype.Size_)
		}
	}

	// 解锁并返回结果
	selunlock(scases, lockorder)
	goto retc

// ========== 分支逻辑：从缓冲区接收数据 ==========
bufrecv:
	// 竞态/内存检测
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
		racenotify(c, c.recvx, nil)
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.Size_)
	}
	if asanenabled && cas.elem != nil {
		asanwrite(cas.elem, c.elemtype.Size_)
	}
	recvOK = true
	// 从缓冲区读取数据
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp) // 拷贝数据到 elem
	}
	typedmemclr(c.elemtype, qp) // 清空缓冲区对应位置
	// 更新缓冲区读指针
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount-- // 缓冲区数据数减1
	selunlock(scases, lockorder)
	goto retc

// ========== 分支逻辑：向缓冲区发送数据 ==========
bufsend:
	// 竞态/内存检测
	if raceenabled {
		racenotify(c, c.sendx, nil)
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.Size_)
	}
	if asanenabled {
		asanread(cas.elem, c.elemtype.Size_)
	}
	// 向缓冲区写入数据
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	// 更新缓冲区写指针
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++ // 缓冲区数据数加1
	selunlock(scases, lockorder)
	goto retc

// ========== 分支逻辑：从等待的发送者接收数据 ==========
recv:
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc

// ========== 分支逻辑：从已关闭的 channel 接收数据 ==========
rclose:
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem) // 清空 elem（返回零值）
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc

// ========== 分支逻辑：向等待的接收者发送数据 ==========
send:
	// 竞态/内存检测
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.Size_)
	}
	if asanenabled {
		asanread(cas.elem, c.elemtype.Size_)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc

// ========== 分支逻辑：返回结果 ==========
retc:
	// 性能分析：记录阻塞时长
	if caseReleaseTime > 0 {
		blockevent(caseReleaseTime-t0, 1)
	}
	return casi, recvOK

// ========== 分支逻辑：向已关闭的 channel 发送数据（panic） ==========
sclose:
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```
![](/assets/select-1769245211852.png)
![](/assets/select-1769245225898.png)

### 循环

![](/assets/select-1769245511073.png)
![](/assets/select-1769245535503.png)
这其实是循环执行的第一次遍历，主要作用就是寻找所有 case 中 Channel 是否有可以立刻被处理的情况，无论是在包含等待的 Goroutine 还是缓冲区中存在数据，只要满足条件就会立刻处理，如果不能立刻找到活跃的 Channel 就会进入循环的下一个过程，按照需要将当前的 Goroutine 加入到所有 Channel 的 sendq 或者 recvq 队列中：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	// ...
	gp = getg()
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		sg.elem = cas.elem
		sg.c = c
		*nextp = sg
		nextp = &sg.waitlink

		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)

	// ...
}
```
这里创建 sudog 并入队的过程其实和 Channel 中直接进行发送和接收时的过程几乎完全相同，只是除了在入队之外，这些 sudog 结构体都会被串成链表附着在当前 Goroutine 上，在入队之后会调用 gopark 函数挂起当前的 Goroutine 等待调度器的唤醒。

![](/assets/select-1769245586450.png)

等到 select 对应的一些 Channel 准备好之后，当前 Goroutine 就会被调度器唤醒，这时就会继续执行 selectgo 函数中剩下的逻辑，也就是从上面 入队的 sudog 结构体中获取数据：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
// ...
gp.selectDone = 0
sg = (*sudog)(gp.param)
gp.param = nil

	casi = -1
	cas = nil
	sglist = gp.waiting
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if sg == sglist {
			casi = int(casei)
			cas = k
		} else {
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	c = cas.c

	if cas.kind == caseRecv {
		recvOK = true
	}

	selunlock(scases, lockorder)
	goto retc
	// ...
}
```
在第三次根据 lockOrder 遍历全部 case 的过程中，我们会先获取 Goroutine 接收到的参数 param，这个参数其实就是被唤醒的 sudog 结构，我们会依次对比所有 case 对应的 sudog 结构找到被唤醒的 case 并释放其他未被使用的 sudog                结构。

由于当前的 select 结构已经挑选了其中的一个 case 进行执行，那么剩下 case 中没有被用到的 sudog 其实就会直接忽略并且释放掉了，为了不影响 Channel 的正常使用，我们还是需要将这些废弃的 sudog 从 Channel 中出队；而除此之外的发生事件导致我们被唤醒的 sudog 结构已经在 Channel
进行收发时就已经出队了，不需要我们再次处理，出队的代码以及相关分析其实都在 Channel 一节中发送和接收的章节。

当我们在循环中发现缓冲区中有元素或者缓冲区未满时就会通过 goto 关键字跳转到以下的两个代码段，这两段代码的执行过程其实都非常简单，都只是向 Channel 中发送或者从缓冲区中直接获取新的数据：

```go
bufrecv:
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

bufsend:
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc
```

这里在缓冲区中进行的操作和直接对 Channel 调用 chansend 和 chanrecv 进行收发的过程差不多，执行结束之后就会直接跳到 retc 字段。

两个直接收发的情况，其实也就是调用 Channel 运行时的两个方法 send 和 recv，这两个方法会直接操作对应的 Channel：

```go
recv:
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	recvOK = true
	goto retc

send:
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	goto retc
```

不过当发送或者接收时，情况就稍微有一点复杂了，从一个关闭 Channel 中接收数据会直接清除 Channel 中的相关内容，而向一个关闭的 Channel 发送数据就会直接 panic 造成程序崩溃：

```go
rclose:
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	goto retc

sclose:
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
```

# 总结

总体来看，Channel 相关的收发操作和上一节 Channel 实现原理中介绍的没有太多出入，只是由于 select 多出了 default 关键字所以会出现非阻塞收发的情况。

另外：
1. select 结构的执行过程与实现原理，首先在编译期间，Go 语言会对 select 语句进行优化，以下是根据 select 中语句的不同选择了不同的优化路径：
![](/assets/select-1769245779096.png)
2. 在编译器已经对 select 语句进行优化之后，Go 语言会在运行时执行编译期间展开的 selectgo 函数，这个函数会按照以下的过程执行：
![](/assets/select-1769245806372.png)
```text
Go 语言中的 select 关键字与 IO 多路复用中的 select、epoll 等函数非常相似，不但 Channel 的收发操作与等待 IO 的读写能找到这种一一对应的关系，这两者的作用也非常相似；
总的来说，select 关键字的实现原理稍显复杂，与 Channel 的关系非常紧密，这里省略了很多 Channel 操作的细节，数据结构一章其实就介绍了 Channel 收发的相关细节。
```


