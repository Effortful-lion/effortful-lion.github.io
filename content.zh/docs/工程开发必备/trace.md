---
title: "Trace"
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

# Trace 工具

## 介绍

在真实的程序中还包含许多的隐藏动作，例如：

- Goroutine 在执行时会做哪些操作？
- Goroutine 执行/阻塞了多长时间？
- Syscall 在什么时候被阻止？在哪里被阻止的？
- 谁又锁/解锁了 Goroutine ？
- GC 是怎么影响到 Goroutine 的执行的？

这些东西用 pprof 是很难分析出来的，但如果你又想知道上述的答案的话，你可以用本章节的主角 go tool trace 来打开新世界的大门。

感觉就是查看 goroutine的运行情况 / GMP 调度情况

## 用法

**基础：**
```go
func main(){
	/*
	输出到文件（更常用），比如 os.Create("trace.out")
	f, err := os.Create("trace.out")
	if err != nil {
	    panic(err)
	}
	if err := trace.Start(f); err != nil {
	    panic(err)
	}
	 */
	
	// 启动 trace 追踪，将 trace 数据输出到标准错误（os.Stderr）终端
    trace.Start(os.Stderr)
	// 延迟停止 trace，确保程序退出前完整写入所有 trace 数据
    defer trace.Stop()
	...
}
```

**生成并使用：**
程序运行后会生成 trace.out 文件
```go
go tool trace trace.out
```
如果提示 failed to execute dot: exec: "dot": executable file not found in $PATH：
是缺少可视化依赖（graphviz），安装即可：
```shell
# Mac
brew install graphviz
# Linux
apt install graphviz
# Windows
下载 graphviz 并配置到环境变量
```

命令执行后会自动打开一个本地链接（比如 http://127.0.0.1:6060/trace）

## 指标

1. 调度延迟概况（Scheduler latency profile）[通过 Graph 看到整体的调用开销情况]
2. Goroutine 分析（Goroutine analysis）[整个运行过程中，每个函数块有多少个有 Goroutine 在跑。]
3. 查看跟踪
![](/assets/trace-1769256228792.png)

- 时间线：显示执行的时间单元，根据时间维度的不同可以调整区间，具体可执行 shift + ? 查看帮助手册。
- 堆：显示执行期间的内存分配和释放情况。
- 协程：显示在执行期间的每个 Goroutine 运行阶段有多少个协程在运行，其包含 GC 等待（GCWaiting）、可运行（Runnable）、运行中（Running）这三种状态。
- OS 线程：显示在执行期间有多少个线程在运行，其包含正在调用 Syscall（InSyscall）、运行中（Running）这两种状态。
- 虚拟处理器：每个虚拟处理器显示一行，虚拟处理器的数量一般默认为系统内核数。
- 协程和事件：显示在每个虚拟处理器上有什么 Goroutine 正在运行，而连线行为代表事件关联。

![](/assets/trace-1769256251840.png)
点击具体的 Goroutine 行为后可以看到其相关联的详细信息，文字解释如下：

- Start：开始时间
- Wall Duration：持续时间
- Self Time：执行时间
- Start Stack Trace：开始时的堆栈信息
- End Stack Trace：结束时的堆栈信息
- Incoming flow：输入流
- Outgoing flow：输出流
- Preceding events：之前的事件
- Following events：之后的事件
- All connected：所有连接的事件

4. 查看事件

我们可以通过点击 View Options-Flow events、Following events 等方式，查看我们应用运行中的事件流情况。如下：
![](/assets/trace-1769256363179.png)

通过分析图上的事件流，我们可得知：

这程序从 G1 runtime.main 开始运行。
在运行时创建了 2 个 Goroutine：
先是创建 G18 runtime/trace.Start.func1。
再是创建 G19 main.main.func1。

同时我们可以通过其 Goroutine Name 去了解它的调用类型。如下：
![](/assets/trace-1769256387277.png)
runtime/trace.Start.func1 就是程序中在 main.main 调用了 runtime/trace.Start 方法。
紧接着该方法又利用协程创建了一个闭包func1 去进行调用。
在这里我们结合开头的代码去看的话，很明显就是 ch 的输入输出的过程了。

5. 网络阻塞概况(Network blocking profile)
6. 系统调用阻塞概况(Syscall blocking profile)

## 指标查看

**可视化界面：**
![](/assets/trace-1769255433208.png)
- View trace：查看跟踪
- Goroutine analysis：Goroutine 分析
- Network blocking profile：网络阻塞概况
- Synchronization blocking profile：同步阻塞概况
- Syscall blocking profile：系统调用阻塞概况
- Scheduler latency profile：调度延迟概况
- User defined tasks：用户自定义任务
- User defined regions：用户自定义区域
- Minimum mutator utilization：最低 Mutator 利用率（业务代码（非 GC 代码）—— 业务执行效率）

## 总结

通过本文我们习得了 go tool trace ，它能够跟踪捕获各种执行中的事件，例如：

- Goroutine 的创建/阻塞/解除阻塞。
- Syscall 的进入/退出/阻止，GC 事件。
- Heap 的大小改变。
- Processor 启动/停止等等。

Go 的两大杀器 pprof + trace 组合，此乃排查好搭档，谁用谁清楚