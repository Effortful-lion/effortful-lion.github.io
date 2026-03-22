---
title: "Benchmark"
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

# Benchmark

## 什么是基准测试

简单来说，基准测试 是一种定量的性能测试方法，核心目的是精准测量代码（函数 / 方法 / 程序）的执行效率、资源消耗（如内存、CPU）等指标，为代码优化、不同实现方案对比提供客观的数据依据。

可以方便快捷地在测试一个函数方法在串行或并行环境下的基准表现。指定一个时间（默认是1秒），看测试对象在达到或超过时间上限时，最多能被执行多少次和在此期间测试对象内存分配情况。

- 它不是只跑一次函数看耗时，而是在设定的时间内（比如默认 1 秒）尽可能高频地重复执行目标函数；
- 最终输出 “每秒执行次数（Ops/s）”“总执行次数”“每次执行的内存分配（堆内存、分配次数）” 等核心指标；
- 对比串行 / 并行环境下的这些指标，就能清晰看出函数在不同执行模式下的性能差异。

助记：
- 不同执行模式
- 限定执行时间
- 尽可能的循环执行
- 统计执行次数、内存分配

## testing框架内置基准测试

benchmark - (b *testing.B)

常用API：
```go
// 停止统计时间（可以手动调用，精准停止）
b.StopTimer()
// 开始统计时间
b.StartTimer()
// 重置统计时间（可以手动调用，精准开始）
b.ResetTimer()

// 本身是生成一个子基准测试（name）；结合 表驱动法 实现 批量进行基准测试
/*
表驱动：
   	// 定义测试表：key=子基准名称，value=测试参数n
   	testCases := map[string]int{
   		"n=100":    100,
   		"n=1000":   1000,
   		"n=10000":  10000,
   		"n=100000": 100000, // 新增参数只需加一行
   	}
    for ...{
        b.Run()
    }
 */
b.Run(name string, f func(b *B))

// 默认将函数分配到所有数量为 runtime.GOMAXPROCS(0) * p（p==1） 的协程上（可以自定义实现 分配+goroutine数量设置）
b.RunParallel(body func(*PB))

// b.ReportAllocs()这个API比较简单，就是打上标记，在benchmark执行完毕后，输出信息会包括B/op和allocs/op这两项信息。
// B/op: 每次分配字节数
// allocs/op：每次分配次数
b.ReportAllocs()

// 增大 goroutine 的个数(注意，b.SetParallelism()的调用一定要放在b.RunParallel()之前。)
// 最终goroutine个数 = 形参p的值 * runtime.GOMAXPROCS(0)
b.SetParallelism(p int)

// 本次benchmark每秒大约用了多少MB的内存
b.SetBytes(n int64)

// BenchmarkResult:结构体包含所有原始数据（总次数、总耗时、内存分配等）；
// 直接执行基准测试并返回原始结果的函数
// 基于这些原始数据，定制任何维度的输出（比如 “执行 b.N 次耗时多少”“1 秒实际能跑多少次”“每次执行的内存分配详情”）。
testing.Benchmark(f func(b *B)) BenchmarkResult
```

## 用法

### 串行
```go
func BenchmarkFoo(b *testing.B) {
  for i:=0; i<b.N; i++ {
    dosomething()
  }
}
```
测试dosomething()在达到1秒或超过1秒时，总共执行多少次。b.N的值就是最大次数。

### 并行
```go
func BenchmarkFoo(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			dosomething()
		}
	})
}
```
1. 并行goroutine数量默认为 runtime.GOMAXPROCS(0)
2. 该方法会创建 P（==GOMAXPROCS(0)） 个goroutine，再把b.N打散到每个goroutine上执行。所以并行用法就比较适合IO型的测试对象。
3. 想增大goroutine的个数，那就使用b.SetParallelism(p int)[注意，b.SetParallelism()的调用一定要放在b.RunParallel()之前。]
![](/assets/benchmark-1769391741669.png)
4. 框架提供的一个 “默认并行实现”，它接管了 b.N 的分配逻辑，但开发者完全可以摆脱这个封装，自己控制 goroutine 数量、b.N 的打散策略，甚至定制更贴合业务场景的并行基准测试逻辑。
```go
// 默认RunParallel实现（对比用）
func BenchmarkDefaultParallel(b *testing.B) {
	b.SetParallelism(2) // 2*GOMAXPROCS个goroutine
	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			mockIOOperation()
		}
	})
}
```
```go
// 自定义并行基准测试：固定10个goroutine，均分b.N
func BenchmarkCustomParallel_EqualSplit(b *testing.B) {
	// 1. 自定义goroutine数量
	customGoroutineNum := 10
	// 2. 计算每个goroutine需要执行的次数（均分b.N）
	perGoroutineN := b.N / customGoroutineNum
	// 处理余数：避免总次数少算（比如b.N=1000003，10个goroutine，前3个多执行1次）
	remainder := b.N % customGoroutineNum

	// 3. 重置计时器（跳过初始化逻辑，只统计核心执行时间）
	b.ResetTimer()

	// 4. 启动goroutine并分配执行次数
	var wg sync.WaitGroup
	wg.Add(customGoroutineNum)

	for i := 0; i < customGoroutineNum; i++ {
		// 每个goroutine的执行次数（处理余数）
		n := perGoroutineN
		if i < remainder {
			n += 1
		}

		go func(taskNum int) {
			defer wg.Done()
			// 执行指定次数的测试函数
			for j := 0; j < taskNum; j++ {
				mockIOOperation()
			}
		}(n)
	}

	// 5. 等待所有goroutine完成
	wg.Wait()
	// 停止计时器
	b.StopTimer()
}
```
```go
// 自定义并行基准测试：加权分配b.N（非均分）
func BenchmarkCustomParallel_Weighted(b *testing.B) {
	// 1. 自定义goroutine数量和权重
	goroutines := []struct {
		weight int // 权重（比如总和为10）
		tasks  int // 最终分配的任务数
	}{
		{weight: 3}, // 30%
		{weight: 3}, // 30%
		{weight: 1}, // 10%
		{weight: 1}, // 10%
		{weight: 1}, // 10%
		{weight: 1}, // 10%
	}
	totalWeight := 0
	for _, g := range goroutines {
		totalWeight += g.weight
	}

	// 2. 根据权重分配b.N（核心：自定义b.N的打散策略）
	remaining := b.N
	for i := range goroutines {
		if i == len(goroutines)-1 {
			// 最后一个goroutine承接剩余所有任务，避免精度丢失
			goroutines[i].tasks = remaining
		} else {
			goroutines[i].tasks = (b.N * goroutines[i].weight) / totalWeight
			remaining -= goroutines[i].tasks
		}
	}

	// 3. 重置计时器
	b.ResetTimer()

	// 4. 启动goroutine执行加权任务
	var wg sync.WaitGroup
	wg.Add(len(goroutines))

	for _, g := range goroutines {
		tasks := g.tasks
		go func(taskNum int) {
			defer wg.Done()
			for j := 0; j < taskNum; j++ {
				mockIOOperation()
			}
		}(tasks)
	}

	wg.Wait()
	b.StopTimer()
}
```
5. b.N的自动校准机制：
![](/assets/benchmark-1769392610476.png)
```go
// 终止条件：三个条件同时不满足时，停止递增b.N
// 条件：竞态检查（有竞态问题） + 时间>=基准时间 + 次数(b.N) > 1e9 终止，否则继续倍增
if !b.failed && b.duration < d && n < 1e9 {
// 继续递增b.N(1 2 4 8...)，重新运行测试
}
```
![](/assets/benchmark-1769392889353.png)

## 公共部分

剩下的API用法就不分串行还是并行了，用在哪种环境下都可以。

### Start、Stop、ReSet
```go
func (b *B) StartTimer() {
	if !b.timerOn {
		// 记录当前时间为开始时间 和 内存分配情况
		b.timerOn = true
	}
}
func (b *B) StopTimer() {
	if b.timerOn {
		// 累计记录执行的时间（当前时间 - 记录的开始时间）
    // 累计记录内存分配次数和分配字节数
		b.timerOn = false
	}
}
func (b *B) ResetTimer() {
	if b.timerOn {
		// 记录当前时间为开始时间 和 内存分配情况
	}
	// 清空所有的累计变量
}
```

### SetBytes(n int)
这次benchmark每秒大概（每次精准）大约用了多少MB的内存。
```go
for i := 0; i < b.N; i++ {
  dAtA, err := github_com_gogo_protobuf_proto.Marshal(pops[i%10000])
  if err != nil {
    panic(err)
  }
  total += len(dAtA)
}
b.SetBytes(int64(total / b.N))
```

### ReportAllocs()
b.ReportAllocs()这个API比较简单，就是打上标记，在benchmark执行完毕后，输出信息会包括B/op和allocs/op这两项信息。

### Benchmark()

默认benchmark时间(benchtime)上限是1秒，可以通过-test.benchtime来改变：
```go
var benchTime = flag.Duration("test.benchtime", 1*time.Second, "run each benchmark for duration `d`")
```

![](/assets/benchmark-1769394521048.png)

基于这些原始数据，定制任何维度的输出（比如 “执行 b.N 次耗时多少”“1 秒实际能跑多少次”“每次执行的内存分配详情”）:

## 全场景自定义示例

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"strings"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

// ======================== 第一步：待测试的业务函数 ========================
// 模拟IO密集型函数（带内存分配）
func mockIOWithAlloc(n int) string {
	// 随机休眠0-1ms模拟IO等待
	time.Sleep(time.Duration(rand.Intn(1000)) * time.Nanosecond)
	// 字符串拼接（触发内存分配）
	s := ""
	for i := 0; i < n; i++ {
		s += string(rune(i))
	}
	return s
}

// 优化版IO函数（减少内存分配）
func mockIOOptimized(n int) string {
	time.Sleep(time.Duration(rand.Intn(1000)) * time.Nanosecond)
	var sb strings.Builder
	sb.Grow(n) // 预分配内存
	for i := 0; i < n; i++ {
		sb.WriteRune(rune(i))
	}
	return sb.String()
}

// ======================== 第二步：自定义基准测试核心逻辑 ========================
// 自定义并行执行逻辑：接管b.N，自定义goroutine数量和任务分配策略
// 参数说明：
// - b: 基准测试对象
// - goroutineNum: 自定义goroutine数量
// - fn: 待测试函数
// - param: 测试函数参数
func customParallelBenchmark(b *testing.B, goroutineNum int, fn func(int) string, param int) {
	// 1. 自定义设置：开启内存分配统计
	b.ReportAllocs()

	// 2. 自定义：跳过初始化耗时（关键，避免统计无关逻辑）
	b.StopTimer()
	// 初始化随机数（避免首次执行耗时异常）
	rand.Seed(time.Now().UnixNano())
	// 计算每个goroutine分配的b.N（加权分配：前2个goroutine承担60%任务）
	totalTasks := b.N
	perGoroutineTasks := make([]int, goroutineNum)
	// 前2个goroutine各分配30%
	if goroutineNum >= 2 {
		perGoroutineTasks[0] = totalTasks * 30 / 100
		perGoroutineTasks[1] = totalTasks * 30 / 100
	}
	// 剩余任务均分给其他goroutine
	remaining := totalTasks - perGoroutineTasks[0] - perGoroutineTasks[1]
	avgRemaining := remaining / (goroutineNum - 2)
	remainder := remaining % (goroutineNum - 2)
	for i := 2; i < goroutineNum; i++ {
		perGoroutineTasks[i] = avgRemaining
		if i-2 < remainder {
			perGoroutineTasks[i] += 1 // 处理余数
		}
	}
	b.StartTimer()

	// 3. 自定义：启动goroutine执行任务（手动管理waitgroup）
	var wg sync.WaitGroup
	wg.Add(goroutineNum)
	// 统计goroutine执行的实际任务数（验证分配策略）
	var actualTasks int64 = 0

	for i := 0; i < goroutineNum; i++ {
		tasks := perGoroutineTasks[i]
		go func(taskNum int) {
			defer wg.Done()
			for j := 0; j < taskNum; j++ {
				fn(param) // 执行测试函数
				atomic.AddInt64(&actualTasks, 1)
			}
		}(tasks)
	}

	// 等待所有goroutine完成
	wg.Wait()

	// 4. 自定义：验证任务执行完整性（可选）
	if atomic.LoadInt64(&actualTasks) != int64(totalTasks) {
		b.Fatalf("任务执行不完整：预期%d次，实际%d次", totalTasks, actualTasks)
	}
}

// ======================== 第三步：表驱动+子基准测试（全自定义配置） ========================
// 封装基准测试入口：包含串行/并行、不同参数、不同函数的对比
func benchCustomEntry(b *testing.B) {
	// 自定义配置表：覆盖不同场景
	testCases := []struct {
		name          string        // 子基准名称
		fn            func(int) string // 待测试函数
		param         int           // 函数参数
		goroutineNum  int           // 自定义goroutine数量（0表示串行）
		customTimeout time.Duration // 自定义终止时间（默认1秒，这里演示自定义）
	}{
		{"Serial/Unoptimized", mockIOWithAlloc, 100, 0, 2 * time.Second},  // 串行+未优化+参数100+2秒超时
		{"Serial/Optimized", mockIOOptimized, 100, 0, 2 * time.Second},    // 串行+优化+参数100+2秒超时
		{"Parallel/Unoptimized", mockIOWithAlloc, 100, 16, 2 * time.Second}, // 并行(16goroutine)+未优化+参数100+2秒超时
		{"Parallel/Optimized", mockIOOptimized, 100, 16, 2 * time.Second},   // 并行(16goroutine)+优化+参数100+2秒超时
		{"Parallel/Optimized/200", mockIOOptimized, 200, 20, 2 * time.Second}, // 并行(20goroutine)+优化+参数200+2秒超时
	}

	// 遍历测试表，创建子基准
	for _, tc := range testCases {
		tc := tc // 闭包捕获循环变量
		// 自定义子基准名称
		b.Run(tc.name, func(b *testing.B) {
			// 1. 自定义终止时间（覆盖默认1秒）
			b.SetBenchmarkTime(tc.customTimeout)
			// 2. 自定义GOMAXPROCS（可选，控制P数量）
			runtime.GOMAXPROCS(runtime.NumCPU() * 2) // 设置P为CPU核心数*2

			// 3. 区分串行/并行执行
			if tc.goroutineNum == 0 {
				// 串行执行：自定义b.N循环
				b.ReportAllocs()
				b.ResetTimer()
				for i := 0; i < b.N; i++ {
					tc.fn(tc.param)
				}
			} else {
				// 并行执行：调用自定义并行逻辑
				customParallelBenchmark(b, tc.goroutineNum, tc.fn, tc.param)
			}
		})
	}
}

// ======================== 第四步：自定义输出（基于testing.Benchmark()） ========================
func main() {
	// 1. 执行全自定义基准测试，获取原始结果
	fmt.Println("==== 开始全自定义基准测试 ====")
	fmt.Printf("测试环境：CPU核心数=%d，GOMAXPROCS=%d\n", runtime.NumCPU(), runtime.GOMAXPROCS(0))
	fmt.Println("=============================\n")

	// 执行基准测试（返回原始结果）
	benchResult := testing.Benchmark(benchCustomEntry)

	// 2. 自定义输出：解析原始数据，输出多维度指标
	fmt.Println("\n==== 自定义基准测试报告 ====")
	// 遍历所有子基准结果（benchResult是汇总，子基准结果在benchResult.Sub）
	printBenchResult(benchResult, 0)

	// 3. 单独输出关键统计（比如1秒实际执行次数、内存优化对比）
	fmt.Println("\n==== 关键指标对比 ====")
	// 提取并行未优化vs优化的对比
	unoptimizedParallel := findSubBenchResult(benchResult, "Parallel/Unoptimized")
	optimizedParallel := findSubBenchResult(benchResult, "Parallel/Optimized")
	if unoptimizedParallel != nil && optimizedParallel != nil {
		// 计算1秒实际执行次数
		opsUnopt := float64(unoptimizedParallel.N) / unoptimizedParallel.T.Seconds()
		opsOpt := float64(optimizedParallel.N) / optimizedParallel.T.Seconds()
		// 计算内存优化率
		memUnopt := unoptimizedParallel.MemBytes / uint64(unoptimizedParallel.N)
		memOpt := optimizedParallel.MemBytes / uint64(optimizedParallel.N)
		memOptRate := 100 - (float64(memOpt)/float64(memUnopt))*100

		fmt.Printf("并行未优化：1秒执行≈%.0f次，每次分配≈%d B\n", opsUnopt, memUnopt)
		fmt.Printf("并行优化版：1秒执行≈%.0f次，每次分配≈%d B\n", opsOpt, memOpt)
		fmt.Printf("内存优化率：%.2f%%，性能提升：%.2f%%\n", memOptRate, (opsOpt/opsUnopt-1)*100)
	}
}

// ======================== 辅助函数：自定义输出工具 ========================
// 递归打印所有子基准结果
func printBenchResult(br testing.BenchmarkResult, indent int) {
	indentStr := strings.Repeat("  ", indent)
	// 输出当前基准的核心指标
	fmt.Printf("%s名称：%s\n", indentStr, br.Name)
	fmt.Printf("%s总执行次数（b.N）：%d\n", indentStr, br.N)
	fmt.Printf("%s总耗时：%v（%.2f秒）\n", indentStr, br.T, br.T.Seconds())
	fmt.Printf("%s每次执行耗时：%d ns/op\n", indentStr, br.T.Nanoseconds()/int64(br.N))
	fmt.Printf("%s每次执行内存分配：%d B/op\n", indentStr, br.MemBytes/uint64(br.N))
	fmt.Printf("%s每次执行分配次数：%d allocs/op\n", indentStr, br.MemAllocs/uint64(br.N))
	fmt.Printf("%s1秒实际执行次数：≈%.0f次\n", indentStr, float64(br.N)/br.T.Seconds())
	fmt.Println(indentStr + "---------------------")

	// 递归打印子基准
	for _, sub := range br.Sub {
		printBenchResult(sub, indent+1)
	}
}

// 查找指定名称的子基准结果
func findSubBenchResult(br testing.BenchmarkResult, name string) *testing.BenchmarkResult {
	if strings.Contains(br.Name, name) {
		return &br
	}
	for _, sub := range br.Sub {
		found := findSubBenchResult(sub, name)
		if found != nil {
			return found
		}
	}
	return nil
}
```
![](/assets/benchmark-1769394215524.png)
### 总结:
- 这个示例整合了：
- 自定义b.N分配 + goroutine 调度；
- 自定义终止时间 + GOMAXPROCS；
- 表驱动子基准 + 串行 / 并行对比；
- 内存统计 + 自定义多维度输出；
- 业务级指标对比（优化率、执行次数）。

## 常用场景测试用例
```go
package main

import (
	"crypto/md5"
	"encoding/hex"
	"net/http"
	"sync"
	"testing"
	"time"
)

// ======================== 场景1：CPU密集型函数测试（如算法、数据处理） ========================
// 核心特点：无IO等待，纯计算，性能瓶颈在CPU算力
// 核心配置：goroutine数=CPU核心数（避免调度开销），默认1秒超时，开启内存统计

// 待测试函数：MD5加密（CPU密集）
func md5Encrypt(data string) string {
	h := md5.New()
	h.Write([]byte(data))
	return hex.EncodeToString(h.Sum(nil))
}

// 基准测试：CPU密集型场景
func BenchmarkCPUIntensive(b *testing.B) {
	// 1. 核心参数设置
	b.ReportAllocs()                    // 开启内存统计（CPU密集型也需关注内存）
	b.SetBenchmarkTime(1 * time.Second) // 默认1秒，足够覆盖CPU密集型测试
	runtime.GOMAXPROCS(runtime.NumCPU())// 设为CPU核心数，最大化算力

	// 2. 预热/初始化（跳过计时）
	b.StopTimer()
	testData := "test-cpu-intensive-123456" // 固定测试数据
	b.StartTimer()

	// 3. 串行执行（CPU密集型无需并行，避免调度开销）
	for i := 0; i < b.N; i++ {
		md5Encrypt(testData)
	}
}

// ======================== 场景2：IO密集型函数测试（如HTTP请求、数据库操作） ========================
// 核心特点：大量IO等待（网络/磁盘），goroutine会频繁阻塞
// 核心配置：goroutine数=CPU核心数*4~10，延长超时时间（避免测试时间过短），开启内存统计

// 待测试函数：模拟HTTP请求（IO密集）
func mockHTTPRequest(url string) (int, error) {
	client := &http.Client{Timeout: 500 * time.Millisecond}
	resp, err := client.Get(url)
	if err != nil {
		return 0, err
	}
	defer resp.Body.Close()
	return resp.StatusCode, nil
}

// 基准测试：IO密集型场景
func BenchmarkIOIntensive(b *testing.B) {
	// 1. 核心参数设置
	b.ReportAllocs()                        // 必须开启，IO密集型常伴随内存分配
	b.SetBenchmarkTime(3 * time.Second)     // 延长超时到3秒，减少统计误差
	b.SetParallelism(8)                     // goroutine数=8*CPU核心数（IO密集型需多goroutine）

	// 2. 预热/初始化（跳过计时）
	b.StopTimer()
	testURL := "https://www.baidu.com"     // 固定测试URL
	b.StartTimer()

	// 3. 并行执行（利用IO等待时间调度其他goroutine）
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			// 忽略错误，专注性能测试
			_, _ = mockHTTPRequest(testURL)
		}
	})
}

// ======================== 场景3：高并发场景测试（如goroutine池、限流逻辑） ========================
// 核心特点：模拟生产环境高并发，goroutine数远大于CPU核心数
// 核心配置：自定义goroutine数，加权分配b.N，验证并发安全性

// 待测试函数：并发计数器（模拟高并发下的共享资源操作）
type Counter struct {
	mu    sync.Mutex
	count int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	c.count++
	c.mu.Unlock()
}

// 基准测试：高并发场景
func BenchmarkHighConcurrency(b *testing.B) {
	// 1. 核心参数设置
	b.ReportAllocs()                        // 开启内存统计
	b.SetBenchmarkTime(2 * time.Second)     // 2秒足够覆盖高并发测试
	customGoroutineNum := runtime.NumCPU() * 10 // 自定义goroutine数=10*CPU核心数

	// 2. 预热/初始化（跳过计时）
	b.StopTimer()
	counter := &Counter{}
	var wg sync.WaitGroup
	wg.Add(customGoroutineNum)
	// 计算每个goroutine分配的b.N（均分）
	perGoroutineN := b.N / customGoroutineNum
	remainder := b.N % customGoroutineNum
	b.StartTimer()

	// 3. 自定义高并发执行（接管b.N分配）
	for i := 0; i < customGoroutineNum; i++ {
		n := perGoroutineN
		if i < remainder {
			n += 1 // 处理余数
		}

		go func(taskNum int) {
			defer wg.Done()
			for j := 0; j < taskNum; j++ {
				counter.Inc() // 高并发操作共享资源
			}
		}(n)
	}

	wg.Wait()

	// 4. 验证并发安全性（可选）
	if counter.count != b.N {
		b.Fatalf("并发计数错误：预期%d，实际%d", b.N, counter.count)
	}
}

// ======================== 场景4：表驱动多参数对比（如不同数据量/配置） ========================
// 核心特点：测试同一函数在不同参数下的性能
// 核心配置：子基准测试，统一参数模板，批量对比

// 基准测试：表驱动多参数场景
func BenchmarkTableDriven(b *testing.B) {
	// 1. 全局参数设置
	b.ReportAllocs() // 所有子基准都开启内存统计

	// 2. 测试表（不同参数组合）
	testCases := []struct {
		name     string // 子基准名称
		dataSize int    // 测试数据大小
		isParallel bool // 是否并行
	}{
		{"small/serial", 100, false},    // 小数据+串行
		{"small/parallel", 100, true},   // 小数据+并行
		{"large/serial", 10000, false},  // 大数据+串行
		{"large/parallel", 10000, true}, // 大数据+并行
	}

	// 3. 遍历测试表
	for _, tc := range testCases {
		tc := tc // 闭包捕获循环变量
		b.Run(tc.name, func(b *testing.B) {
			// 子基准参数设置
			if tc.isParallel {
				b.SetParallelism(4) // 并行场景设为4*CPU核心数
			}

			// 初始化测试数据
			b.StopTimer()
			testData := make([]byte, tc.dataSize) // 生成指定大小的测试数据
			b.StartTimer()

			// 执行测试
			if tc.isParallel {
				b.RunParallel(func(pb *testing.PB) {
					for pb.Next() {
						_ = md5Encrypt(string(testData)) // 复用CPU密集型函数
					}
				})
			} else {
				for i := 0; i < b.N; i++ {
					_ = md5Encrypt(string(testData))
				}
			}
		})
	}
}

// ======================== 场景5：自定义输出（生产环境性能报表） ========================
// 核心特点：将基准测试结果输出为易读的报表，用于性能对比/上线前验证
func main() {
	// 1. 执行各场景基准测试
	benchCPU := testing.Benchmark(BenchmarkCPUIntensive)
	benchIO := testing.Benchmark(BenchmarkIOIntensive)
	benchConcurrency := testing.Benchmark(BenchmarkHighConcurrency)
	benchTable := testing.Benchmark(BenchmarkTableDriven)

	// 2. 自定义输出报表
	printBenchReport("CPU密集型（MD5加密）", benchCPU)
	printBenchReport("IO密集型（HTTP请求）", benchIO)
	printBenchReport("高并发（计数器）", benchConcurrency)
	printBenchReport("多参数对比（MD5）", benchTable)
}

// 辅助函数：打印易读的性能报表
func printBenchReport(scene string, br testing.BenchmarkResult) {
	// 核心指标计算
	nsPerOp := br.T.Nanoseconds() / int64(br.N)
	opsPerSecond := float64(br.N) / br.T.Seconds()
	bytesPerOp := br.MemBytes / uint64(br.N)
	allocsPerOp := br.MemAllocs / uint64(br.N)

	// 输出报表
	fmt.Printf("==== %s ====\n", scene)
	fmt.Printf("总执行次数：%d次\n", br.N)
	fmt.Printf("总耗时：%.2f秒\n", br.T.Seconds())
	fmt.Printf("单次耗时：%d ns/op\n", nsPerOp)
	fmt.Printf("每秒执行：%.0f次\n", opsPerSecond)
	fmt.Printf("单次内存分配：%d B/op\n", bytesPerOp)
	fmt.Printf("单次分配次数：%d allocs/op\n", allocsPerOp)
	fmt.Println("------------------------\n")
}
```



