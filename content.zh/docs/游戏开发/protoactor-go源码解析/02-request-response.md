---
title: "02-request-response"
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

# actor-actor 如何基于 RequestFuture 处理异步请求和响应

## 第一次运行
![](/assets/02-request-response-1768527287104.png)
下面的输出对比hello.md，发现：这是我们响应回来的结果

## 和异步无响应的特殊点

### 异步 - 请求 - 响应
```go
result, _ := rootContext.RequestFuture(pid, Hello{Who: "Roger"}, 30*time.Second).Result() // await result
```
创建一个 future：
```go
type future struct {
	actorSystem *ActorSystem
	pid         *PID
	cond        *sync.Cond
	// protected by cond
	done        bool
	result      interface{}
	err         error
	t           *time.Timer
	pipes       []*PID
	completions []func(res interface{}, err error)
}
```

### RequestFuture 如何响应结果的
1. result, _ := future.Result()
2. 等待赋值，赋值成功后，返回结果
```go
func (f *future) wait() {
	f.cond.L.Lock()
	for !f.done {
		// 继续等待赋值
		f.cond.Wait()
	}
	// 赋值成功
	f.cond.L.Unlock()
}

// Result waits for the future to resolve.
func (f *future) Result() (interface{}, error) {
	f.wait()

	return f.result, f.err
}
```
3. 赋值过程：
```go
// futureProcess实现Process接口的SendUserMessage方法（这段不在NewFuture里！）
// 当Actor调用context.Respond()时，响应消息会发送到future的PID，触发这个方法
func (ref *futureProcess) SendUserMessage(pid *PID, message interface{}) {
    defer ref.instrument()
    
    _, msg, _ := UnwrapEnvelope(message)

    // ========== 【正常响应的赋值：核心位置】 ==========
    // 响应是“死信”（消息发送失败，比如Actor不存在）
	if _, ok := msg.(*DeadLetterResponse); ok {
        ref.result = nil
        ref.err = ErrDeadLetter
    } else {
		// 正常响应
        ref.result = msg
    }
    
    ref.Stop(pid)
}
```

### 封装为 envelop - sendUserMessage
```go
func (rc *RootContext) RequestFuture(pid *PID, message interface{}, timeout time.Duration) Future {
	future := NewFuture(rc.actorSystem, timeout)
	env := &MessageEnvelope{
		Header:  nil,
		Message: message,
		Sender:  future.PID(),
	}
	rc.sendUserMessage(pid, env)

	return future
}
```

### future 对象
```go
// Future defines the public interface for future responses.
type Future interface {
	// PID to the backing actor for the Future result.
	PID() *PID
	// PipeTo forwards the result or error of the future to the specified PIDs.
	PipeTo(pids ...*PID)
	// Result waits for the future to resolve and returns the result or error.
	Result() (interface{}, error)
	// Wait blocks until the future resolves and returns the error, if any.
	Wait() error
}
```

## 总结 future 赋值流程：

**简单说：**
它没有用 Go 原生的chan来接收异步响应，而是复用 Actor 模型的 “PID 通信 + 状态存储” 机制：通过 Actor 间的消息投递，把响应结果保存到future的result/err属性中，再通过sync.Cond实现阻塞等待，而非chan的阻塞接收。

**代码逻辑：**
异步请求 → 创建future Actor（带PID） → 目标Actor处理后通过PID发送响应 → future Actor接收响应并赋值到result → 唤醒阻塞协程返回结果

**阻塞等待值过程：**
future 的 “阻塞 - 唤醒” 全链路，sync.Cond是唯一核心：
阻塞阶段：
调用Result() → 进入wait() → 加锁检查done=false → 调用cond.Wait() → 释放锁、协程挂起（让出 CPU，不阻塞线程）；
唤醒阶段：
Actor 响应消息 → SendUserMessage赋值result → 调用Stop() → 加锁置done=true → 调用cond.Signal() → 唤醒挂起的协程 → 协程重新加锁、检查done=true → 退出等待。

## 证明异步请求 - 响应：
修改源代码：
```go
package main

import (
	"fmt"
	"time"

	console "github.com/asynkron/goconsole"
	"github.com/asynkron/protoactor-go/actor"
)

type Hello struct{ Who string }

func Receive(ctx actor.Context) {
	switch msg := ctx.Message().(type) {
	case Hello:
		// Actor模拟耗时处理（异步体现：Actor独立执行）
		time.Sleep(2 * time.Second)
		ctx.Respond("Hello " + msg.Who)
	}
}

func main() {
	system := actor.NewActorSystem()
	props := actor.PropsFromFunc(Receive)
	pid := system.Root.Spawn(props)

	// 1. 发起请求：非阻塞，立即返回future
	fmt.Println("Step 1：发起请求，当前时间：", time.Now().Format("04:05"))
	fut := system.Root.RequestFuture(pid, Hello{Who: "Roger"}, 30*time.Second)

	// 2. 发起请求后可执行任意逻辑（异步核心：不阻塞）
	fmt.Println("Step 2：请求已发送，继续执行其他代码，当前时间：", time.Now().Format("04:05"))
	time.Sleep(1 * time.Second) // 模拟其他耗时操作
	fmt.Println("Step 3：其他代码执行完毕，准备等结果，当前时间：", time.Now().Format("04:05"))

	// 3. 仅这里阻塞：等待Actor处理完并返回结果
	result, _ := fut.Result()
	fmt.Println("Step 4：拿到结果，当前时间：", time.Now().Format("04:05"), "结果：", result)

	_, _ = console.ReadLine()
}
```