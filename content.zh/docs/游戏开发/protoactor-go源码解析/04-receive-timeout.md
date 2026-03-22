---
title: "04-receive-timeout"
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

# 超时消息设置和自定义非重置计时器消息

## 代码

```go
package main

import (
	"fmt"
	"log"
	"time"

	console "github.com/asynkron/goconsole"
	"github.com/asynkron/protoactor-go/actor"
)

type NoInfluence string

func (NoInfluence) NotInfluenceReceiveTimeout() {}

func main() {
	log.Println("Receive timeout test")

	system := actor.NewActorSystem()
	c := 0

	rootContext := system.Root
	props := actor.PropsFromFunc(func(context actor.Context) {
		switch msg := context.Message().(type) {
		case *actor.Started:
			context.SetReceiveTimeout(1 * time.Second)

		case *actor.ReceiveTimeout:
			c++
			log.Printf("ReceiveTimeout: %d", c)

		case string:
			log.Printf("received '%s'", msg)
			if msg == "cancel" {
				fmt.Println("Cancelling")
				context.CancelReceiveTimeout()
			}

		case NoInfluence:
			log.Println("received a no-influence message")

		}
	})

	pid := rootContext.Spawn(props)
	//for i := 0; i < 7; i++ {
	//	rootContext.Send(pid, "hello")
	//	time.Sleep(500 * time.Millisecond)
	//}

	log.Println("hit [return] to send no-influence messages")
	_, _ = console.ReadLine()

	for i := 0; i < 6; i++ {
		rootContext.Send(pid, NoInfluence("hello"))
		time.Sleep(500 * time.Millisecond)
	}

	log.Println("hit [return] to send a message to cancel the timeout")
	_, _ = console.ReadLine()
	rootContext.Send(pid, "cancel")

	log.Println("hit [return] to finish")
	_, _ = console.ReadLine()

	rootContext.Stop(pid)
}

```

## context.SetReceiveTimeout

- 设置超时消息，actor得实现处理超时消息的逻辑
- 取消设置 context.CancelReceiveTimeout()

## 不能重置计时器的消息接口

```go
// NotInfluenceReceiveTimeout messages will not reset the ReceiveTimeout timer of an actor that receives the message
type NotInfluenceReceiveTimeout interface {
	NotInfluenceReceiveTimeout()
}
```
实现后，我们的消息就无法重置计时器。相当于实现了“一次性定时器”

## 生产环境用法：

### 一、核心结论
你的理解**完全正确**，这正是 `NoInfluence` 消息（实现 `NotInfluenceReceiveTimeout()` 接口）在生产环境中的核心价值——**让“非核心/无需严格控时”的消息不干扰“核心/需严格控时”消息的超时逻辑**。

简单说：
- 对**需要严格控时的核心消息**（如订单支付指令、交易确认、关键业务响应）：用普通消息类型，让它们触发超时计时器重置，保证“只要核心消息正常流转，超时就不触发”；
- 对**无需控时的非核心消息**（如心跳、日志、监控、统计、通知类消息）：标记为 `NoInfluence` 消息，让它们不重置超时计时器，避免这类“无关消息”“续命”超时逻辑，保证核心业务的超时判断精准。

### 二、生产场景落地示例（更易理解）
以「订单支付 Actor」为例，拆解两种消息的设计逻辑：

#### 1. 需严格控时的核心消息（普通消息，重置超时）
```go
// 核心业务消息：订单支付指令（必须严格控时，超时未处理则触发告警/回滚）
type OrderPayRequest struct {
    OrderID string
    Amount  float64
}

// 无 NotInfluenceReceiveTimeout() 接口 → 普通消息
// 收到该消息时，重置超时计时器（证明业务还在正常处理）
```
- 超时逻辑：Actor 启动时设置 `SetReceiveTimeout(30 * time.Second)`（30秒无支付指令则触发超时）；
- 效果：只要每隔<30秒收到 `OrderPayRequest` 消息，超时就不会触发；若超过30秒未收到（说明支付流程卡住），则触发超时逻辑（如记录告警、回滚未完成订单）。

#### 2. 无需控时的非核心消息（NoInfluence 消息，不重置超时）
```go
// 非核心消息：心跳检测（仅确认 Actor 存活，无需影响支付超时）
type ActorHeartbeat string

// 实现特殊接口 → NoInfluence 消息
func (ActorHeartbeat) NotInfluenceReceiveTimeout() {}

// 非核心消息：订单日志（仅记录操作，不影响支付超时）
type OrderLog struct {
    OrderID string
    Content string
}

func (OrderLog) NotInfluenceReceiveTimeout() {}
```
- 设计逻辑：
  ① 每5秒给 Actor 发 `ActorHeartbeat` 心跳消息，哪怕1分钟发12次，也**不会重置**支付超时的30秒计时器；
  ② Actor 处理订单日志 `OrderLog` 时，同样不重置超时；
- 效果：只有“支付指令”这类核心消息能“续命”超时，心跳/日志不会干扰——若30秒内只有心跳、无支付指令，仍会触发超时，保证支付流程的超时判断精准。

### 三、生产落地的关键原则（避坑）
#### 1. 明确区分“两类消息”的标准
| 消息类型         | 特征                                  | 是否标记为 NoInfluence |
|------------------|---------------------------------------|------------------------|
| 核心控时消息     | 影响业务核心逻辑、需保证时效性、超时需处理 | ❌ 否（普通消息）|
| 非核心消息       | 仅监控/通知/日志/心跳、不影响核心逻辑 | ✅ 是（NoInfluence）|

#### 2. 避免的典型误区
- ❌ 误区1：把所有消息都设为普通消息 → 心跳/日志频繁重置超时，导致核心业务的超时逻辑“失效”（比如支付流程卡住，但心跳一直在“续命”，超时永远不触发）；
- ❌ 误区2：把核心消息设为 NoInfluence → 核心消息无法重置超时，哪怕业务正常，也会误触发超时（比如支付指令正常接收，但超时仍按固定间隔触发，导致误告警/回滚）；
- ✅ 正确做法：只给“非核心、无需控时”的消息加 `NoInfluence` 标记，核心消息保持普通类型。

#### 3. 补充：超时逻辑的兜底设计
生产中除了消息类型区分，还需配合：
- `CancelReceiveTimeout()`：核心业务完成后（如订单支付成功），主动取消超时，避免后续无意义的超时触发；
- 超时触发后的处理：超时后不仅打日志，还要做实际兜底（如通知运维、回滚中间状态、重试核心操作）；
- 超时阈值配置：根据业务场景调整（如支付场景30秒、普通查询场景5秒），而非全量用同一个值。

### 四、总结（核心要点）
1. 生产中「严格控时的核心消息」用普通类型，保证它们能重置超时，精准控制业务超时逻辑；
2. 「非核心、无需控时的消息」标记为 `NoInfluence`，避免干扰核心消息的超时判断；
3. 这个设计是 Proto.Actor 解决“超时逻辑被无关消息干扰”的最佳实践，核心是**让超时只对核心业务生效**，非核心消息不影响控时精度。

本质上，这是“把超时逻辑的控制权聚焦到核心业务”，避免海量非核心消息淹没核心业务的超时判断，是生产环境中保证超时逻辑精准性的关键手段。
