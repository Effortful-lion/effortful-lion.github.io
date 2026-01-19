---
title: "Goroutine和线程的区别"
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

# Goroutine 和 线程 的区别

下面我们将从三个方面去区别二者的区别：内存消耗、创建与销毀、切换

---

## 内存消耗

### Goroutine

1. 创建一个 goroutine 的消耗为 2 KB 栈内存
2. goroutine 的栈空间是可以自动扩容的

### 线程

1. 创建一个线程的消耗为：1MB 栈内存
2. 另外还需要一个被称为 “a guard page” 的区域用于和其他 thread 的栈空间进行隔离。

### 总结

那么，对于 go 来说，可以基于 goroutine 轻松的开n个协程去处理大并发的 http Server

但是，像 Java 这种 基于线程作为并发原语的语言 就需要开n个线程去处理，那么很快会触发OOM

---

## 创建和销毁

### Goroutine

1. 用户级（Go runtime 负责管理）
2. 开销小

### 线程

1. 内核级（操作系统 管理）
2. 开销大

### 总结

当遇到开销大的时候，一般采用线程池、协程池。

---

## 切换

goroutines 切换成本比 threads 要小得多。

另：
CPU一个纳秒平均可以执行 12-18 条指令

### Goroutine

1. 保存  PC (Program Counter)、SP (Stack Pointer)、BP
2. 切换速度： 200ns，相当于 2400-3600 条指令。

### 线程

1. 以下是这段文字的精准中文翻译，适配技术文档的表达习惯：

> 16个通用寄存器、程序计数器（PC）、栈指针寄存器（SP）、段寄存器、16个XMM寄存器、浮点协处理器状态、16个AVX寄存器，以及所有的模型专用寄存器（MSR）等。

**补充说明（便于理解技术术语）**
    
    1. **通用寄存器**：程序执行过程中用于临时存储数据、地址的核心寄存器，支持多种运算操作。
    2.  **程序计数器（PC）**：在x86-64架构中对应`RIP`寄存器，用于存储下一条待执行指令的内存地址。
    3.  **栈指针寄存器（SP）**：对应`RSP`寄存器，指向当前栈空间的栈顶位置，管理函数调用、局部变量存储。
    4.  **模型专用寄存器（MSR）**：CPU厂商为特定处理器型号设计的专用寄存器，用于配置硬件特性、监控系统状态。

2. 切换速度： 1000-1500 ns，相当于 12000-18000 条指令。



