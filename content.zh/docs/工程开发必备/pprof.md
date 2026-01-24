---
title: "Pprof"
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

# go pprof 性能分析工具

1. 怎么用？
2. 怎么实现？

## 作用和指标
golang 的最常用的性能分析工具：

**指标：**
```text
1. profile：探测各函数对 cpu 的占用情况
2. heap：探测内存分配情况
3. block：探测阻塞情况 （包括 mutex、chan、cond 等）
4. mutex：探测互斥锁占用情况
5. goroutine：探测协程使用情况
```

## 怎么用

**例子：**
```go
package main

import (
    "log"
    "net/http"
    // 启用 pprof 性能分析
    _ "net/http/pprof"
    "os"
    "runtime"
    "time"

    "github.com/wolfogre/go-pprof-practice/animal"
)

func main() {
    // ...

	// 只使用一个cpu逻辑内核 现象更明显
    runtime.GOMAXPROCS(1)
    // 启用 mutex 性能分析
    runtime.SetMutexProfileFraction(1)
    // 启用 block 性能分析
    runtime.SetBlockProfileRate(1)

    go func() {
        // 启动 http server. 对应 pprof 的一系列 handler 也会挂载在该端口下
        if err := http.ListenAndServe(":6060", nil); err != nil {
            log.Fatal(err)
        }
        os.Exit(0)
    }()

    // 运行各项动物的活动
    for {
        for _, v := range animal.AllAnimals {
            v.Live()
        }
        time.Sleep(time.Second)
    }
}
// 这里的 Live 中有个性能炸弹：for空循环 1000000 次 cpu 立刻打满
```

那么需要注意：
1. 打开开关，用于设置想要分析的方面（cpu、goroutine、锁、内存、阻塞情况）
2. 为了可视化的查看结果，配合 http/pprof 包，启动 http 服务（一般就是异步服务go func）
3. 注意设置时的参数，通常和收集频率、内存大小相关，要综合性能要求和收集目的

## CPU占用

> cpu 分析是在一段时间内进行打点采样，通过查看采样点在各个函数栈中的分布比例，以此来反映各函数对 cpu 的占用情况.

### cpu占用的指标

1. flat：函数本身占用cpu/执行时长
2. flat%：flat 的百分比占比
3. sum%：本函数 + 父函数 的执行时长占比
4. cum：父函数falt +子函数falt（函数及其嵌套占用总时长）
5. cum%：cum的百分比占比

那么我们其实一般只需要看百分比，很少用到数字：
```text
- 快速找直接耗 CPU 的函数：看 flat%（核心是 “函数自身” 的占比）；
- 找调用链整体耗 CPU 的入口：看 cum%（核心是 “函数 + 子函数” 的总占比）；
- 量化优化效果 / 精细控制：看 flat/cum 的绝对时长（而非百分比）；
- sum% 仅用于快速看累计占比，实际优化中很少作为核心参考。
```

### cpu占用的指标查看

#### 命令行交互式

点击页面上的 profile 后，默认会在停留 30S 后下载一个 cpu profile 文件.
通过交互式指令打开文件后，查看 cpu 使用情况：
![](/assets/pprof-1769249455263.png)

#### 可视化拓扑图

还可以通过图形化界面来展示 cpu profile 文件中的内容：
```text
go tool pprof -http=:8082  {YOUR PROFILE PATH}
```
![](/assets/pprof-1769249502511.png)

#### 火焰图

![](/assets/pprof-1769249522911.png)

### 大概讲一讲示例

1. 首先定位到 Tiger.Eat 是性能炸弹
2. 通过指标图我们看到还有一个函数，但是代码中其实没体现，这是触发了goroutine的调度机制了（monitor协程，专用goroutine，不受gmp调度：基于信号的超时抢占，插入抢占代码，修改程序计数器pc，执行后让渡cpu）
```go
// 此时执行方是即将要被抢占的 g，这段代码是被临时插入的逻辑
func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    // mcall 切换至 g0，然后完成 g 的让渡
    mcall(gopreempt_m)
    gp.asyncSafePoint = false
}
```

## heap 分析

> 这是对内存管理/分配的情况进行记录：聚焦 “谁分配、分配多少、当前使用多少”，提供分配全量数据；

### heap 指标和作用

点击 heap 进入 http://localhost:6060/debug/pprof/heap?debug=1
> 在页面的路径中能看到 debug 参数，如果 debug = 1，则将数据在页面上呈现；如果将 debug 设为 0，则会将数据以二进制文件的形式下载，并支持通过交互式指令或者图形化界面对文件内容进行呈现. block/mutex/goroutine 的机制也与此相同，后续章节中不再赘述

**指标、作用/使用场景：：**

1. 当前活跃的采样数
2. 当前活跃内存大小 = 当前活跃采样数 × 采样粒度
3. 累计采样数
4. 累计分配内存大小 = 累计采样数 × 采样粒度
5. 采样粒度（我们设置的采样内存大小）

整体：
1. 分配：总体数值增大
2. gc：活跃数值减少

eg：
- 累计≈活跃 → 优先查分配后的引用泄漏（GC 收不回）；
- 累计 >> 活跃 → 优先查过度分配（GC 收得快但分配太频繁）；
- 活跃偏高但累计正常 → 优先查 GC 效率（触发太晚 / 回收慢）；

---

1. 第一行：全局采样概览（heap profile: 1: 1291845632 [21: 3371171968] @ heap/1048576）
   这一行是**整个堆分析的元数据**，每个数字 / 符号的含义如下：
    ![](/assets/pprof-1769250598962.png)
2. 第二行：**单条调用栈**的采样数据（1: 1291845632 [1: 1291845632] @ 0x104303b48 0x1043033b8 0x104303cc0 0x10410938c 0x10413ca24）
   这一行是具体调用栈的采样数据，和第一行格式对应，但只针对当前这条调用链：
    ![](/assets/pprof-1769250664422.png)
![](/assets/pprof-1769250752441.png)
![](/assets/pprof-1769250836416.png)


### 指标查看

比如：这里的指标，在页面中获取到有关 heap 的信息：
![](/assets/pprof-1769250219627.png)
![](/assets/pprof-1769250243320.png)
![](/assets/pprof-1769250324029.png)
对应的文件路径：
![](/assets/pprof-1769250343362.png)

## block分析

> 查看某个 goroutine 陷入 waiting 状态（被动阻塞，通常因 gopark 操作触发，比如因加锁、读chan条件不满足而陷入阻塞）的触发次数和持续时长.

![](/assets/pprof-1769251546930.png)

### block指标

1. 每秒钟对应的cpu周期数（cycles/second=1000002977）cycle作为周期（也是后面度量的单位）
2. 阻塞的周期数（3002910915），可以换算为 s，根据 1 中的单位值，做除法（3003307959/1000002977 ≈ 3S）
3. 发生的阻塞次数（3次）

### 指标查看
![](/assets/pprof-1769251675364.png)
![](/assets/pprof-1769251841142.png)
于是我们定位到其中一处引起阻塞的代码是 Cat.Pee，每当函数被调用时会简单粗暴地等待 timer 1S，里面会因读 chan 而陷入阻塞：
```go
func (c *Cat) Pee() {
    log.Println(c.Name(), "pee")

    <-time.After(time.Second)
}
```

## mutex分析

> mutex 分析看的是某个 goroutine 持有锁的时长（mutex.Lock -> mutex.Unlock 之间这段时间），且只有在存在锁竞争关系时才会上报这部分数据.
![](/assets/pprof-1769251920060.png)

### mutex指标

1. 每秒下的 cycle 数
2. 持有锁的 cycle 总数（计算采样次数）
3. 采样次数（采样事件：锁竞争事件触发[出现了goroutine的等待情况]）

### 指标查看

![](/assets/pprof-1769252053210.png)
![](/assets/pprof-1769252069416.png)
- 1000002767——每秒下的 cycle 数
- 4007486874——持有锁的 cycle 总数
- 4——采样了 4 次
- sampling period=1 是 “全量采样”**（我们自己设置的）**，采样次数 = 实际竞争次数；数值越大，采样越稀疏，开销越低但需换算实际次数。
于是定位到占有锁较多的方法是 Wolf.Howl，每次加锁后都睡了一秒：(代码没写全，但是确实有)
```go
func (w *Wolf) Howl() {
    log.Println(w.Name(), "howl")

    m := &sync.Mutex{}
    m.Lock()
    go func() {
        time.Sleep(time.Second)
        m.Unlock()
    }()
    m.Lock()
}
```

## goroutine分析
![](/assets/pprof-1769252403172.png)
![](/assets/pprof-1769252414124.png)
