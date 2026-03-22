---
title: "05-supervision"
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

# Actor 监督机制的实现方式及错误处理的核心逻辑

## 第一次运行
![](/assets/05-supervision-1768558204837.png)

## Actor 监督机制

### Supervision

```go
// 错误处理方式 - 错误处理策略
type DeciderFunc func(reason interface{}) Directive

// SupervisorStrategy is an interface that decides how to handle failing child actors
// 监督策略 - 需要实现 - 具体的错误处理方法
type SupervisorStrategy interface {
	HandleFailure(actorSystem *ActorSystem, supervisor Supervisor, child *PID, rs *RestartStatistics, reason interface{}, message interface{})
}

// Supervisor is an interface that is used by the SupervisorStrategy to manage child actor lifecycle
// 监督行为 - 管理行为
type Supervisor interface {
	Children() []*PID
	EscalateFailure(reason interface{}, message interface{})
	RestartChildren(pids ...*PID)
	StopChildren(pids ...*PID)
	ResumeChildren(pids ...*PID)
}
```

### 具体捕获

> panic
```go
    // panic 出现在 Receive()
    func (state *childActor) Receive(context actor.Context)
    // 自然会在 mailbox 的 run 函数中的 recovery 捕获
	defer func() {
		if r := recover(); r != nil {
			m.invoker.EscalateFailure(r, msg)
		}
	}()
```

> 上报监督者 + 日志记录

```go
func (ctx *actorContext) EscalateFailure(reason interface{}, message interface{}) {
	// 1. 基础日志：记录 Actor 故障恢复的基础信息（self=当前Actor PID，reason=故障原因）
	ctx.Logger().Info("[ACTOR] Recovering", slog.Any("self", ctx.self), slog.Any("reason", reason))
	
	// 2. 开发者模式日志：若开启开发者监督日志，打印堆栈+详细错误（便于调试）
	if ctx.actorSystem.Config.DeveloperSupervisionLogging {
		fmt.Printf("debug.Stack(): %s\n", debug.Stack()) // 打印故障堆栈
		fmt.Println("[Supervision] Actor:", ctx.self, " failed with message:", message, " exception:", reason)
		ctx.Logger().Error("[Supervision]", slog.Any("actor", ctx.self), slog.Any("message", message), slog.Any("exception", reason))
	}

	// 3. 监控指标：若开启Metrics，累加Actor故障计数（用于监控告警）
	if ctx.actorSystem.Config.MetricsEnabled {
		metricsSystem, ok := ctx.actorSystem.Extensions.Get(extensionID).(*Metrics)
		if ok && metricsSystem.Enabled() {
			_ctx := context.Background()
			if instruments := metricsSystem.metrics.Get(metrics.InternalActorMetrics); instruments != nil {
				// 故障计数+1，附加Actor标签（便于按Actor维度统计）
				instruments.ActorFailureCount.Add(_ctx, 1, metric.WithAttributes(metricsSystem.CommonLabels(ctx)...))
			}
		}
	}

	// 4. 封装故障信息：将故障原因、当前Actor PID、重启统计、故障消息封装为Failure系统消息
	failure := &Failure{Reason: reason, Who: ctx.self, RestartStats: ctx.ensureExtras().restartStats(), Message: message}

	// 5. 暂停Mailbox：发送suspendMailboxMessage，暂停当前Actor的消息处理（避免故障中继续处理消息）
	ctx.self.sendSystemMessage(ctx.actorSystem, suspendMailboxMessage)

	// 6. 故障上报：分两种情况处理
	if ctx.parent == nil {
		// 6.1 无父Actor（根Actor）：由ActorSystem直接处理（通常打印日志+终止Actor）
		ctx.handleRootFailure(failure)
	} else {
		// 6.2 有父Actor：将Failure消息发送给父Actor（监督者），触发父Actor的监督策略
		ctx.parent.sendSystemMessage(ctx.actorSystem, failure)
	}
}
```

## 监督机制中的错误处理逻辑

### 具体处理 

> 上报父 Actor + 处理

消息转发处理
```go
// MessageInvoker is the interface used by a mailbox to forward messages for processing
// MessageInvoker 消息转发接口 转发后被处理 
type MessageInvoker interface {
	InvokeSystemMessage(interface{})
	InvokeUserMessage(interface{})
	EscalateFailure(reason interface{}, message interface{})
}
```

> 具体步骤
```text
1. 执行panic("Children start Panic!!!") → 子 Actor 抛出未捕获异常，触发框架的故障处理逻辑。
2. 框架触发故障上报（EscalateFailure）, 子 Actor 的上下文调用EscalateFailure：封装故障信息（原因、子 Actor PID、触发消息），暂停子 Actor 的 Mailbox，将Failure系统消息上报给父 Actor（监督者）；(若开启开发者日志，会打印故障堆栈、Actor PID、触发消息等信息。)
3. 父 Actor 的监督策略（OneForOneStrategy）接收到Failure消息 → 调用自定义decider函数；
4. decider打印 handling failure for child，返回StopDirective（停止子 Actor）；
5. 框架触发子 Actor 的*actor.Stopping系统消息 → 子 Actor 打印 Stopping, actor is about to shut down；
6. 子 Actor 停止后，框架触发*actor.Stopped系统消息 → 子 Actor 打印 Stopped, actor and its children are stopped。
```

### demo 解析 

这是我们的官方 example ：

另外：
1. actor的生命周期会伴随发送消息
```
*actor.Started 消息的触发时机：
   当调用 system.Root.Spawn(props) 启动 Actor 时，框架内部流程：
    1. 初始化 Actor 上下文、Mailbox、PID 等核心资源；
    2. 框架直接调用 Actor 的 Receive 方法，传入携带 *actor.Started 的上下文（无网络 / 队列传输，是本地方法调用）；
    3. 该消息是 “框架通知”，而非 Actor 通过 Send/Request 给自己发的消息（无 Mailbox 入队过程）。
```

```go
package main

import (
	"fmt"

	console "github.com/asynkron/goconsole"
	"github.com/asynkron/protoactor-go/actor"
)

type (
	hello       struct{ Who string }
	parentActor struct{}
)

func (state *parentActor) Receive(context actor.Context) {
	switch msg := context.Message().(type) {
	// 收到消息后，创建并启动子 Actor ，向子 Actor 发送 msg
	case *actor.Started:
		fmt.Println("Starting, initialize Parent actor here")
	case *hello:
		props := actor.PropsFromProducer(newChildActor)
		child := context.Spawn(props)
		context.Send(child, msg)
	}
}

func newParentActor() actor.Actor {
	return &parentActor{}
}

type childActor struct{}

func (state *childActor) Receive(context actor.Context) {
	switch msg := context.Message().(type) {
	case *actor.Started:
		fmt.Println("Starting, initialize actor here")
	case *actor.Stopping:
		fmt.Println("Stopping, actor is about to shut down")
	case *actor.Stopped:
		fmt.Println("Stopped, actor and its children are stopped")
	case *actor.Restarting:
		fmt.Println("Restarting, actor is about to restart")
	case *hello:
		// 收到 Parent Actor 的消息，开始 panic
		fmt.Printf("Hello %v\n", msg.Who)
		panic("Children start Panic!!!")
	}
}

func newChildActor() actor.Actor {
	return &childActor{}
}

func main() {
	// system - actor system started
	system := actor.NewActorSystem()
	// decider 选定 stop actor 错误处理方式
	decider := func(reason interface{}) actor.Directive {
		fmt.Println("handling failure for child")
		return actor.StopDirective
	}
	// 基于错误处理方式创建监督策略: 影响范围 - 单个actor / 重试次数 / 限期内 / 策略
	supervisorStrategy := actor.NewOneForOneStrategy(10, 1000, decider)
	// 系统级的 context
	rootContext := system.Root
	// 基于监督策略，返回 PropsOption 配置项
	PropsOption := actor.WithSupervisor(supervisorStrategy)
	// Parent Actor 的 Prop
	props := actor.PropsFromProducer(newParentActor, PropsOption)

	// 创建 / 启动 Parent Actor
	pid := rootContext.Spawn(props)
	// 系统发布消息：Roger
	rootContext.Send(pid, &hello{Who: "Roger"})

	// 控制台日志
	_, _ = console.ReadLine()
}
```