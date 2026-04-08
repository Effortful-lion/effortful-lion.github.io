---
title: "Context"
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

# Context 源码解析

参考文章：https://zhuanlan.zhihu.com/p/682397085

## Context 图解
![](/assets/Context-1769002117867.png)

**问题：**
![](/assets/Context-1769001818034.png)

## 从示例看通信

### cancelContext

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// 调试用：打印协程信息
func debugPrint(goroutine string, msg string) {
	fmt.Printf("[%s] %s\n", goroutine, msg)
}

func main() {
	// 1. 创建 根 cancel context
	parentCtx, parentCancel := context.WithCancel(context.Background())
	defer parentCancel()

	debugPrint("main", "父 context 创建完成")

	// 2. 创建 子 context
	childCtx, childCancel := context.WithCancel(parentCtx)
	defer childCancel()

	debugPrint("main", "子 context 创建完成")

	// 3. 创建 孙 context
	grandsonCtx, grandsonCancel := context.WithCancel(childCtx)
	defer grandsonCancel()

	debugPrint("main", "孙 context 创建完成")

	// 启动 父 协程
	go worker(parentCtx, "父协程")

	// 启动 子 协程
	go worker(childCtx, "子协程")

	// 启动 孙 协程
	go worker(grandsonCtx, "孙协程")

	// 等待 2 秒，模拟业务执行
	time.Sleep(2 * time.Second)

	debugPrint("main", "======================")
	debugPrint("main", "!!! 调用 父cancel()，开始递归取消全部")
	debugPrint("main", "======================")

	// 4. 调用父 cancel → 递归取消子、孙
	parentCancel()

	// 等待协程退出打印
	time.Sleep(1 * time.Second)
	debugPrint("main", "全部协程已退出")
}

// worker 监听 ctx.Done() 退出信号
func worker(ctx context.Context, name string) {
	debugPrint(name, "启动成功，等待取消信号")

	// 阻塞等待取消信号
	<-ctx.Done()

	// 收到取消信号
	debugPrint(name, "!!! 收到取消信号，退出")
	debugPrint(name, "退出原因: "+ctx.Err().Error())
}
```

```text
[main] 父 context 创建完成
[main] 子 context 创建完成
[main] 孙 context 创建完成
[孙协程] 启动成功，等待取消信号
[父协程] 启动成功，等待取消信号
[子协程] 启动成功，等待取消信号
[main] ======================
[main] !!! 调用 父cancel()，开始递归取消全部
[main] ======================
[孙协程] !!! 收到取消信号，退出
[父协程] !!! 收到取消信号，退出
[孙协程] 退出原因: context canceled
[父协程] 退出原因: context canceled
[子协程] !!! 收到取消信号，退出
[子协程] 退出原因: context canceled
[main] 全部协程已退出
```

**通信链路：**
```text
父cancel()
   ↓ 关闭父 done channel
   ↓ 遍历父 children，取消子
   子cancel()
      ↓ 关闭子 done channel
      ↓ 遍历子 children，取消孙
         孙cancel()
             ↓ 关闭孙 done channel
```

**问题：**
0. context的设计原理
```text
设计原理核心三点：
1. 树形传播
子上下文创建时自动注册到父节点的 children 中，形成上下文树，实现父子绑定、递归取消。
2. 通信机制
依靠 done channel 做广播通知，<-ctx.Done() 阻塞等待，关闭通道即触发所有 goroutine 退出。
3. 职责分离
取消、超时、传值分别由不同实现类负责，通过组合而非继承扩展，结构轻量、高效、无侵入。
```

1. cancel工作原理
```text
一句话：context 利用 Channel + Select 调度，让 Goroutine 主动监听 ctx.Done() 信号。一旦 close(done)，Goroutine 收到信号就自动退出，从而实现安全、优雅地停止 Goroutine。
eg：

外部调用 1. cancel() 2. 然后函数内就会 close(ctx.done)，这个会关闭 done 通道，用于通知关闭
那么就会在这被捕捉到：
select {
case <-ctx.Done(): // 等待关闭信号
    return // 退出 goroutine
default:
    // 正常执行业务
}
直接退出goroutine
```

2. context传递原理
```text
对应的context传递原理是：父context中维护子context的map（子context会注册到父context的map中），递归关闭的本质实际上就是父context遍历map然后依次调用cancel()
那么对应的效果是：
子context退出
父context才退出（这是一个栈式的结果）
```

3. context定时原理
```text
context 定时原理的本质是：内部通过 time.AfterFunc () 函数（实际上是创建了一个timer），设置过期时间和 cancel 回调函数。当时间到达时，自动回调 cancel () 实现取消。
定时器的实现原理就是通过 Go 运行时维护的最小堆（timer heap）统一管理所有 Timer，不需要为每个定时任务创建单独的 goroutine 等待；
时间到达后，runtime 会从最小堆中取出到期的 timer，
然后【新建一个独立 goroutine】执行回调函数，
触发 cancel ()，进而递归取消整个 context 树。
```
**补充：**
```text
1. 一个P有一个系统协程 timerproc
2. 管理全局定时器的最小堆
3. timerproc 会从最小堆中取出对应的 timer 对象，然后新建一个 goroutine 来执行注册的回调函数
```
**作用过程：**
```text
1. WithTimeout → 创建 timerCtx
2. time.AfterFunc → 把定时器丢入 runtime 最小堆
3. 系统协程 timerproc 独自管理堆，不占用业务 goroutine
4. 时间到
5. 从堆里取出到期 timer
6. go t.fn () 新建一个 goroutine 执行回调
7. 回调里调用 cancel()
8. cancel () 关闭 done channel
9. 递归取消所有子 context
10. 所有监听 <-ctx.Done () 的 goroutine 被唤醒退出
```