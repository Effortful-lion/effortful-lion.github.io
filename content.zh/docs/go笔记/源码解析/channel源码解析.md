---
title: "Channel源码解析"
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

# channel源码解析

## 数据结构

channel 在底层其实是一个 hchan 的结构体：在编译时期会将 make(chan int) 语句转换成 makechan 函数调用
```go
// runtime/chan.go
type hchan struct {
    lock mutex  // lock 用来保护 hchan 上所有的字段
    
    // 缓冲区实际是一个循环队列（ringbuffer的数据结构定义）
    buf unsafe.Pointer  // 指向缓冲区的指针
    dataqsiz uint   // 缓冲区循环队列的大小（注意：运行时通过判断 c.dataqsiz == 0 来快速区分当前是同步模式还是异步模式）
    sendx uint      // 写索引-sendx 它是下一次向通道发送数据（写入缓冲区）时使用的索引。
    recvx uint      // 读索引-recvx 它是下一次从通道接收数据（从缓冲区读取）时使用的索引。
    qcount uint // 当前 hchan 缓存的元素数量
    closed uint32   // hchan 是否关闭
    
    elemsize uint16 // hchan 的元素大小
    elemtype *_type // hchan 的元素类型
    
    recvq waitq     // 等待读的 goroutine 队列
    sendq waitq     // 等待写的 goroutine 队列
}
```
具体 ringbuffer 可以看channel部分，可以说 有缓冲的channel = ringbuffer + waitQueue：

核心定义：
```go
    // 缓冲区实际是一个循环队列（ringbuffer的数据结构定义）
    buf unsafe.Pointer  // 指向缓冲区的指针
    dataqsiz uint   // 缓冲区循环队列的大小
    sendx uint      // 写索引-sendx 它是下一次向通道发送数据（写入缓冲区）时使用的索引。
    recvx uint      // 读索引-recvx 它是下一次从通道接收数据（从缓冲区读取）时使用的索引。
    qcount uint // 当前 hchan 缓存的元素数量
    closed uint32   // hchan 是否关闭
```

并发安全：
```go
lock mutex  // lock 用来保护 hchan 上所有的字段
```

运行时多态：对应泛型（还有不同，后面可以查看不同）
```go
elemsize uint16 // hchan 的元素大小
elemtype *_type // hchan 的元素类型
```

等待队列：(扩展：增加了环形缓冲区的调度扩展)
```go
    recvq waitq     // 等待读的 goroutine 队列
    sendq waitq     // 等待写的 goroutine 队列
```

> 关于channel和Ringbuffer的不同思考：

1. 泛型 和 运行时类型

2. waitq：环形缓冲区的“调度扩展”

3. 性能优化

> 泛型 和 运行时类型

![](//assets/channel源码解析-1769159365345.png)

> waitq：环形缓冲区的“调度扩展”

![](//assets/channel源码解析-1769159378495.png)

> **性能优化（重要）**

![](//assets/channel源码解析-1769159397874.png)

> 总结区别

![](//assets/channel源码解析-1769159462246.png)

**注意：**
对于无缓冲 channel ，它在逻辑上不存储任何元素，发送和接收必须同时发生（同步握手），因此 Ring Buffer 相关的字段在逻辑上都是冗余的。即：
```go
// 只有：
type hchan struct {
    lock mutex  // lock 用来保护 hchan 上所有的字段
    
    closed uint32   // hchan 是否关闭
    
    // 虽然不进缓冲区，但直接拷贝数据时（Direct Send），运行时依然需要知道元素的大小和类型，以便进行内存移动（memmove）和 GC 类型检查。
    elemsize uint16 // hchan 的元素大小
    elemtype *_type // hchan 的元素类型
    
    recvq waitq     // 等待读的 goroutine 队列
    sendq waitq     // 等待写的 goroutine 队列
}
```

## 待发送队列 & 待接收队列
注意到他们都是 waitq 类型：

```go
// runtime/chan.go
// 双向链表
type waitq struct {
    first *sudog
    last * sudog
}
// 入队
func(q *waitq) enqueue(sgp *sudog){
// ...
} 

// 出队
func (q *waitq) dequeue(sgp *sudog){
// ...
}
```

## sudog 结构
```go

// sudog（伪 goroutine）表示等待列表中的一个 g（goroutine），例如在通道（channel）上
// 执行发送/接收操作时的等待 goroutine。
//
// sudog 的存在是必要的，因为 g 与同步对象之间是多对多的关系：一个 g 可能处于多个等待列表中，
// 因此一个 g 可能对应多个 sudog；同时多个 g 可能等待同一个同步对象，
// 因此一个对象也可能对应多个 sudog。
//
// sudog 从专用的内存池（pool）中分配。请使用 acquireSudog 和
// releaseSudog 函数来分配和释放 sudog 实例。
type sudog struct {
	// 以下字段受当前 sudog 所阻塞的通道的 hchan.lock 保护。
	// shrinkstack（栈收缩）逻辑依赖此字段来处理涉及通道操作的 sudog。

	g *g // 关联的 goroutine 指针

	next *sudog // 等待列表中的下一个 sudog 节点
	prev *sudog // 等待列表中的上一个 sudog 节点

	// 在 channel 会被用来指向待发送者要发送的数据或者待接收者的接收位置
	// 比如：
	/*
	
	// 从 ch 接收数据被阻塞，那么 sudog.elem 会指向 x
	x <- ch

	// 向 ch 发送数据被阻塞，那么 sudog.elem 会指向 y
	ch <- y

	*/
	elem maybeTraceablePtr // 数据元素（可能指向栈内存）

	// 以下字段永远不会被并发访问：
	// - 对于通道场景，waitlink 仅由所属的 g 访问；
	// - 对于信号量（semaphore）场景，所有字段（包括上述字段）
	//   仅在持有 semaRoot 锁时被访问。

	acquiretime int64 // 抢占（等待）开始时间戳
	releasetime int64 // 释放（唤醒）时间戳
	ticket      uint32 // 信号量场景下的排队凭证

	// isSelect 标记当前 g 是否参与 select 操作，
	// 因此需要通过 CAS 操作更新 g.selectDone 来赢取唤醒竞争。
	isSelect bool

	// success 表示通过通道 c 完成的通信是否成功：
	// - 若为 true：goroutine 被唤醒是因为有值通过通道 c 传递完成；
	// - 若为 false：goroutine 被唤醒是因为通道 c 被关闭。
	success bool

	// waiters 表示 semaRoot 等待列表中除表头外的等待者数量，
	// 被限定为 uint16 类型以适配未使用的内存空间。
	// 仅在列表表头节点中此字段有意义。
	// （若想进一步优化，可将高 16 位存储在列表的第二个节点中。）
	waiters uint16

	parent   *sudog             // 信号量树（semaRoot 二叉树）的父节点
	waitlink *sudog             // 关联到 g.waiting 列表或 semaRoot 的链接
	waittail *sudog             // semaRoot 等待列表的尾节点
	c        maybeTraceableChan // 当前 sudog 阻塞的通道
}
```

那么其实 channel 会使用到的部分：
```go
// runtime/runtime2.go
type sudog struct {
    g       *g              //阻塞的 goroutine
    elem    unsafe.Pointer
    c       *hchan          // 阻塞的 channel
```

## makechan 创建 channel
```go
// src/runtime/chan.go
func makechan(t *chantype, size int) *hchan{
    elem := t.elem
    
    // 检查 elem size，align
    
    // 计算出缓冲区的大小，如果是非缓冲 channel 或者元素为 struct{}，那么 mem 就是 0
    mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0{
        panic(plainError("makechan: size out of range"))
    }
    
    var c *hchan
    switch{
    // 非缓冲 channel 或者 缓冲区元素 为 struct{}
    case mem == 0:
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // 如果是非缓冲，则buf并没有用
        // 如果缓冲元素类型为 struct{}, 则只会用到 sendx 和 recvx, 并不会真正拷贝数据到缓冲区
        c.buf = unsafe.Pointer(&c.buf)
        
    // channel 中元素不包含指针
    case elem.ptrdata == 0:
        // 将 hchan 结构和缓冲区的内存一起分配
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        // buf 指向 hchan 后边的地址
        c.buf = add(unsafe.Pointer(c), hchanSize)
        
    // 默认，分别分配 chan 和 buf 的内存    
    default:
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }
    
    // 设置 hchan 的其他字段
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    // 底层循环队列长度
    c.datasiz = uint(size)
    return c
```
**通过 makechan 函数，可以总结出 hchan 结构的特点**

无缓冲或者缓冲的元素类型为 struct{} 时，并不会为缓冲区（hcha.buf）分配内存
缓冲的元素结构中不包含指针时，会将 hchan 和 缓冲区buf 是一块连续的内存

### make 与 makechan
make 函数在编译阶段又是如何转换成 makechan 函数调用的呢

首先编译器会将 make 的调用转换成 OMAKE 类型的节点，然后判断 make 的对象类型，如果是 TCHAN 的话，将节点类型置为 OMAKECHAN，并且检查 make 的第二个参数，也就是缓冲区大小

```go
// src/cmd/compile/internal/gc/typecheck.go
func typecheck1(n *Node, top int) (res *Node) {
    // ...
    switch n.Op{
    case OMAKE:
        switch t.Etype {
        case TCHAN:
            l = nil
            if i < len(args){
                // ... 对缓冲区大小进行检测
                n.Left = l  // 带缓冲区，赋值缓冲区大小
            }else{
                n.Left = nodintconst(0) // 不带缓冲区
            }
            n.Op = OMAKECHAN
        }
    }
}
```

然后OMAKECHAN 节点会在 walkexpr 函数中转换成调用 makechan 或者 makechan64 函数

```go
// src/cmd/compile/internal/gc/walk.go
func walkexpr(n *Node, init *Nodes) *Node {
    switch n.Op {
    case OMAKECHAN:
        size := n.Left
        fnname := "makechan64"
        argtype := types.Types[TINT64]

        if size.Type.IsKind(TIDEAL) || maxintval[size.Type.Etype].Cmp(maxintval[TUINT]) <= 0 {
            fnname = "makechan"
            argtype = types.Types[TINT]
        }
        n = mkcall1(chanfn(fnname, 1, n.Type), n.Type, init, typename(n.Type), conv(size, argtype))
    }
}
```

## 发送数据

向 channel 发送数据的语句会在编译期间转换成 chansend 函数
```go
ch := make(chan int)
ch <- 10
```
发送语句非常简单，但是真正的函数执行会区分很多的情况，做一些小的优化，可以称为特性

### 发送操作的特性(有待检查)

- 向 nil channel 发送数据会被永久阻塞，并且不会被 select 语句选中
- 如果 channel 未关闭，非缓冲并且没有待接收的 goroutine，或者缓冲区已满，那么不会被 select 语句选中
- 向关闭的 channel 发送数据，会 panic ，并且可以被 select 语句选中，意味着 select 语句中可能会 panic
- 如果有待接收者，那么会将发送的数据直接 copy 到待接收者的接收位置，然后唤醒接收者
- 如果有缓冲区，并且缓冲区未满，那么就把发送的数据 copy 到缓冲区中
- 如果 channel 未关闭，缓冲区为空并且没有待接收者，那么直接阻塞当前 goroutine, 等待被唤醒
- 发送者被阻塞后，可以被关闭 channel 操作或者被接收操作唤醒，关闭 channel 导致发送者被唤醒后，会panic
- 当 channel 中有待接收 goroutine，那么 channel 的状态必然是 非缓冲或者缓冲区为空

