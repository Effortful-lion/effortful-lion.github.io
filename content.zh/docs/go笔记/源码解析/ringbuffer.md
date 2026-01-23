---
title: "Channel"
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

# ringBuffer

# 扩展：

编写一个高性能的无锁RingBuffer，对比与 Channel 的差异

回答：
# Channel vs RingBuffer 核心区别对比表
| 对比维度         | Channel（Go 通道）| RingBuffer（环形缓冲区）|
|------------------|---------------------------------------------|---------------------------------------------|
| 核心定位         | Go 原生**同步原语 + 数据传输管道**          | 通用**内存高效型数据结构**（循环队列）      |
| 设计目标         | 解决 goroutine 间安全通信与同步             | 固定容量下高效读写，复用内存                |
| 同步能力         | 内置阻塞/非阻塞语义，天然支持 goroutine 同步 | 无任何同步机制，多协程访问需手动加锁        |
| 内存管理         | 运行时托管，不可手动复用                    | 固定数组，内存完全可控，无额外分配          |
| 读写性能         | 有内置同步开销，性能适中                    | 纯内存操作，性能极高（加锁后略降）|
| 关闭机制         | 支持 close()，关闭后读取返回零值+ok=false    | 无关闭机制，需手动维护状态                  |
| 数据覆盖         | 不支持，满则阻塞/报错                       | 可轻松实现覆盖写（满时覆盖最旧数据）|
| 遍历方式         | 支持 for range 阻塞遍历                     | 需手动遍历 head 到 tail 有效元素            |
| 语言依赖         | 仅 Go 语言原生支持                          | 跨语言通用（Go/Java/C++ 等）|
| 容量扩展         | 创建后容量固定，不可扩展                    | 固定数组大小，扩展需重新创建                |
| 核心优势         | 语法简洁，贴合 Go 并发模型                  | 内存高效，性能极致，支持自定义逻辑          |
| 典型场景         | Go 协程通信、简单生产者-消费者、同步控制    | 高频数据采集、固定内存场景、覆盖写需求      |

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

![](/assets/channel源码解析-1769159365345.png)

> waitq：环形缓冲区的“调度扩展”

![](/assets/channel源码解析-1769159378495.png)

> 性能优化

![](/assets/channel源码解析-1769159397874.png)

> 总结区别

![](/assets/channel源码解析-1769159462246.png)

**编写：**
```go
package ringbuffer

import (
	"fmt"
	"math"
	"strings"
)

// 线程不安全（单线程）的 ringbuffer
// 核心规则：
// 1. 固定大小，满了覆盖最旧数据
// 2. head 指向最旧数据，tail 指向下一个待插入位置
// 3. count 记录当前元素个数
type RingBuffer struct {
	buf   []int
	head  int // 最旧数据的位置
	tail  int // 下一个待插入的位置
	count int // 当前元素个数
	cap   int // 固定容量（语义更清晰）
}

// NewRingBuffer 创建指定容量的ringbuffer
func NewRingBuffer(size int) *RingBuffer {
	if size <= 0 {
		panic("size must be greater than 0") // 防御性检查
	}
	return &RingBuffer{
		buf:   make([]int, size),
		head:  0,
		tail:  0,
		count: 0,
		cap:   size,
	}
}

// Count 返回当前元素个数
func (rb *RingBuffer) Count() int {
	return rb.count
}

// Size 返回ringbuffer的固定容量
func (rb *RingBuffer) Size() int {
	return rb.cap
}

// Push 写入数据，满了覆盖最旧数据
func (rb *RingBuffer) Push(v int) {
	// 写入新数据到tail位置
	rb.buf[rb.tail] = v
	// 满了则移动head（覆盖最旧数据），否则count+1
	if rb.count == rb.cap {
		rb.head = (rb.head + 1) % rb.cap // 覆盖旧数据，head后移
	} else {
		rb.count++ // 没满，计数+1
	}
	// tail永远后移（环形）
	rb.tail = (rb.tail + 1) % rb.cap
}

// Pop 弹出最旧数据，空则返回math.MinInt32
func (rb *RingBuffer) Pop() int {
	if rb.count == 0 {
		return math.MinInt32
	}
	v := rb.buf[rb.head]
	rb.buf[rb.head] = 0 // 可选：清空原位置（便于调试）
	rb.head = (rb.head + 1) % rb.cap
	rb.count--
	return v
}

// String 格式化输出ringbuffer内容（[v1, v2, v3]）
func (rb *RingBuffer) String() string {
	if rb.count == 0 {
		return "[]"
	}
	sb := strings.Builder{}
	sb.WriteString("[")

	count := 0
	for i := rb.head; count < rb.count; i = (i + 1) % rb.cap {
		v := rb.buf[i]
		if count == rb.count-1 {
			sb.WriteString(fmt.Sprintf("%d", v))
		} else {
			sb.WriteString(fmt.Sprintf("%d, ", v))
		}
		count++
	}

	sb.WriteString("]")
	return sb.String()
}
```

**测试：**
```go
package ringbuffer

import (
	"fmt"
	"math"
	"sync"
	"testing"
)

func Test_RB(t *testing.T) {
	rb := NewRingBuffer(5)
	nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
	for _, num := range nums {
		rb.Push(num)
		fmt.Printf("buffer：%s\n", rb.String())
	}
}

// TestRingBuffer_New 测试创建ringbuffer的边界情况
func TestRingBuffer_New(t *testing.T) {
	// 测试1：创建容量为0的ringbuffer，应panic
	defer func() {
		if r := recover(); r == nil {
			t.Error("创建容量为0的ringbuffer未触发panic")
		}
	}()
	NewRingBuffer(0)

	// 测试2：创建正常容量的ringbuffer
	rb := NewRingBuffer(3)
	if rb.Size() != 3 || rb.Count() != 0 {
		t.Errorf("创建容量为3的ringbuffer失败，期望Size=3/Count=0，实际Size=%d/Count=%d", rb.Size(), rb.Count())
	}
}

// TestRingBuffer_Push 测试写入数据（未覆盖/覆盖场景）
func TestRingBuffer_Push(t *testing.T) {
	rb := NewRingBuffer(3)

	// 场景1：写入未超容量（3个数据）
	rb.Push(1)
	rb.Push(2)
	rb.Push(3)
	if rb.Count() != 3 || rb.String() != "[1, 2, 3]" {
		t.Errorf("写入3个数据失败，期望Count=3/内容[1,2,3]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}

	// 场景2：写入超容量（覆盖最旧数据）
	rb.Push(4)
	if rb.Count() != 3 || rb.String() != "[2, 3, 4]" {
		t.Errorf("覆盖写入失败，期望Count=3/内容[2,3,4]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}

	// 场景3：继续覆盖写入
	rb.Push(5)
	if rb.Count() != 3 || rb.String() != "[3, 4, 5]" {
		t.Errorf("多次覆盖写入失败，期望Count=3/内容[3,4,5]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}
}

// TestRingBuffer_Pop 测试弹出数据（正常弹出/空队列弹出）
func TestRingBuffer_Pop(t *testing.T) {
	rb := NewRingBuffer(3)

	// 场景1：空队列弹出
	v := rb.Pop()
	if v != math.MinInt32 {
		t.Errorf("空队列弹出失败，期望返回%d，实际返回%d", math.MinInt32, v)
	}

	// 场景2：正常弹出（未空）
	rb.Push(1)
	rb.Push(2)
	rb.Push(3)
	v = rb.Pop()
	if v != 1 || rb.Count() != 2 || rb.String() != "[2, 3]" {
		t.Errorf("弹出第一个数据失败，期望值=1/Count=2/内容[2,3]，实际值=%d/Count=%d/内容%s", v, rb.Count(), rb.String())
	}

	// 场景3：弹出至空
	rb.Pop() // 弹出2
	rb.Pop() // 弹出3
	if rb.Count() != 0 || rb.String() != "[]" {
		t.Errorf("弹出至空失败，期望Count=0/内容[]，实际Count=%d/内容%s", rb.Count(), rb.String())
	}

	// 场景4：空队列再次弹出
	v = rb.Pop()
	if v != math.MinInt32 {
		t.Errorf("空队列弹出失败，期望返回%d，实际返回%d", math.MinInt32, v)
	}
}

// TestRingBuffer_Mix 测试混合读写场景（最接近实际使用）
func TestRingBuffer_Mix(t *testing.T) {
	rb := NewRingBuffer(3)

	// 步骤1：写入2个数据
	rb.Push(10)
	rb.Push(20)
	if rb.Count() != 2 || rb.String() != "[10, 20]" {
		t.Error("步骤1失败：", rb.String())
	}

	// 步骤2：弹出1个数据
	rb.Pop()
	if rb.Count() != 1 || rb.String() != "[20]" {
		t.Error("步骤2失败：", rb.String())
	}

	// 步骤3：写入3个数据（覆盖）
	rb.Push(30)
	rb.Push(40)
	rb.Push(50)
	if rb.Count() != 3 || rb.String() != "[30, 40, 50]" {
		t.Error("步骤3失败：", rb.String())
	}

	// 步骤4：弹出2个数据
	rb.Pop()
	rb.Pop()
	if rb.Count() != 1 || rb.String() != "[50]" {
		t.Error("步骤4失败：", rb.String())
	}

	// 步骤5：写入2个数据（未覆盖）
	rb.Push(60)
	rb.Push(70)
	if rb.Count() != 3 || rb.String() != "[50, 60, 70]" {
		t.Error("步骤5失败：", rb.String())
	}
}

// 这个需要稍加改造，加个读写锁就好了
func TestSafeRingBuffer_ConcurrentPushPop(t *testing.T) {
	// 跳过短测试（如需运行，注释此行）
	// t.Skip("高并发压测，按需运行")

	srb := NewSafeRingBuffer(100)
	var wg sync.WaitGroup
	total := 10000 // 总读写次数

	// 1. 启动10个写入goroutine
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < total/10; j++ {
				srb.Push(j)
			}
		}()
	}

	// 2. 启动10个读取goroutine
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			count := 0
			for count < total/10 {
				if ok := srb.Pop(); ok != 0 {
					count++
				}
			}
			t.Log("输出count:", count)
		}()
	}

	// 3. 等待所有goroutine完成
	wg.Wait()

	// 4. 验证无panic、无死锁（核心是程序能正常结束）
	t.Log("高并发读写测试完成，无异常")
}
```