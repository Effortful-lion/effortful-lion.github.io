---
title: "Goroutine和协程的区别"
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

# Goroutine 和 协程的区别

## 协程

### 什么是协程？

协程（Coroutine）是一种用户态的轻量级执行单元，核心特征可以总结为：
- 由程序员 / 运行时调度，而非操作系统内核调度
- 拥有自己的栈，但栈可以动态伸缩
- 切换成本极低（仅需保存 / 恢复少量上下文）
- 支持暂停（yield）和恢复（resume）

### 协程 vs 线程

![](/assets/goroutine和协程的区别-1768809109427.png)

### 协程的通用分类

- 对称协程：任意协程可以相互切换，调度权由程序员手动控制（如 Lua 协程）；
- 非对称协程：协程只能在 “主协程” 和 “子协程” 之间切换（如 Python 的 generator）；
- 带调度器（scheduler）的协程：由运行时自动调度的协程（如 goroutine、Java 的 Virtual Thread）。

## goroutine

### 协程共性

- ✅ 用户态调度：goroutine 由 Go 调度器（GPM 模型）管理，而非操作系统内核；
- ✅ 轻量级：初始栈仅 2KB，可动态扩缩容（最大达 1GB），创建 / 切换成本极低；
- ✅ 低切换成本：切换时仅保存 PC/SP/BP 等少量上下文（约 200ns），远低于线程（1000-1500ns）；
- ✅ 高并发：一台机器可轻松创建数万个 goroutine，远超线程数量上限。

### 差异

goroutine 是增强版的 协程，是 golang 本土化的实现

![](/assets/goroutine和协程的区别-1768809430599.png)
- 自动抢占式调度：传统协程如果一直不主动让出调度权，会独占线程；而 goroutine 运行超过 10ms（或执行系统调用）时，Runtime 会主动抢占，把调度权交给其他 goroutine；
- 多核并行：Go 调度器会把 goroutine 映射到多个操作系统线程（M）上，绑定到不同 CPU 核心（P），充分利用多核算力；
- 透明的阻塞处理：当 goroutine 执行系统调用（如 IO）阻塞时，Runtime 会把绑定的 P 剥离给其他空闲的 M，避免线程资源浪费。

