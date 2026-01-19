---
title: "什么是 Runtime"
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

# Runtime

## 什么是 Runtime

这是 用户程序、Runtime、OS 三者关系的图：

Go 程序的执行由两层组成：Go Program，Runtime，即用户程序和运行时。它们之间通过函数调用来实现内存管理、channel 通信、goroutines 创建等功能。用户程序进行的系统调用都会被 Runtime 拦截，以此来帮助它进行调度以及垃圾回收相关的工作。

![](/assets/runtime-1768799609158.png)

### Runtime 作用

Runtime 实际可以根据功能划分为两大结构：

#### 对外功能

这部分是 Runtime 暴露给 Go Program 的接口，也是用户代码直接打交道的部分，目的是让开发者无需关心底层细节。

核心功能：
- Goroutine 管理：go 关键字创建协程、runtime.Gosched() 主动让出 CPU 等。
- 内存分配：make、new 以及 runtime.MemStats 内存状态查询。
- 并发原语：channel 通信、sync 包的锁和原子操作等。
- 系统调用封装：os、net 等标准库背后的 Runtime 系统调用拦截与封装。

设计目的：提供简洁、安全的高层 API，让开发者专注业务逻辑，不用管线程、调度等底层细节。

#### 对内功能

这部分是 Runtime 内部的实现，对用户代码完全透明，负责高效管理系统资源，是调度器、垃圾回收等核心能力的载体。

核心组件：
- 调度器（Scheduler）：G/P/M 模型的实现，负责把 Goroutine 映射到操作系统线程和 CPU 核心上执行。
- 垃圾回收器（GC）：三色标记、并发扫描等内存回收与管理逻辑。
- 内存分配器：基于 tcmalloc 实现的高效内存分配，比如 mcache、mcentral、mheap 三级缓存。
- 系统调用拦截器：检测 Goroutine 发起的系统调用，在阻塞时触发调度器抢占，避免线程资源浪费。

设计目的：在底层实现高并发、高吞吐的执行效率，同时隐藏复杂的硬件和操作系统细节。