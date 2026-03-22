---
title: "06-actor生命周期"
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

# Actor 的生命周期

## 生命周期消息

之前也提到过，每一个生命周期，框架都会自动发送一个生命周期的消息。

那么，我们需要在 Actor 的 Receive 中去定义具体的处理逻辑。

## 创建 和 运行

> Actor 创建的时候，会初始化哪些东西？
```go
defaultSpawner = func(actorSystem *ActorSystem, id string, props *Props, parentContext SpawnerContext) (*PID, error) {
	// 初始化 context	
	ctx := newActorContext(actorSystem, props, parentContext.Self())
	// 初始化 mailbox
	mb := props.produceMailbox()
	// 初始化调度器
	dp := props.getDispatcher()
	// 创建一个 ActorProcess 这个可能我们理解的 实现了 Process 接口的 Actor 的动作代理
	proc := NewActorProcess(mb)
	// 在 actorSystem 下注册 id 和 ActorProcess（ActorRef） 的映射
	// 在 基础代码 里面，pid.ref() --> Process接口（ActorProcess）
	pid, absent := actorSystem.ProcessRegistry.Add(proc, id)
	if !absent {
		// 如果存在这个 id 的 actor，就返回报错
		return pid, ErrNameExists
	}
	// 当前 actor 的 pid
	ctx.self = pid
	// 在创建之前的 类似 "PreStart()概念" 自定义的 在正式创建前的钩子函数 全部遍历执行
	initialize(props, ctx)
	// 注册/绑定 消息执行器（转发到处理器的东西）、调度器
	mb.RegisterHandlers(ctx, dp)
	// 发送系统消息： Start
	mb.PostSystemMessage(startedMessage)
	// 正式启动
	mb.Start()
	return pid, nil
}
```

## 重试

.......

## 停止

> 停止是销毁嘛？

不是！！！Stop 是触发 Actor 终止的 “指令”，销毁是 Stop 流程执行完毕后，框架清理 Actor 所有资源的 “最终动作”。

![](/assets/06-actor生命周期-1768570294225.png)


## 代码示例

```go
package main

import (
	"fmt"
	"log/slog"

	console "github.com/asynkron/goconsole"
	"github.com/asynkron/protoactor-go/actor"
)

type (
	hello      struct{ Who string }
	helloActor struct{}
)

func (state *helloActor) Receive(context actor.Context) {
	switch msg := context.Message().(type) {
	// 这里的生命周期的消息全都是由这个框架发出的
	case *actor.Started:
		context.Logger().Info("Started, initialize actor here")
	case *actor.Stopping:
		context.Logger().Info("Stopping, actor is about shut down")
	case *actor.Stopped:
		context.Logger().Info("Stopped, actor and its children are stopped")
	case *actor.Restarting:
		context.Logger().Info("Restarting, actor is about restart")
		// 这是 外部消息 打印
	case *hello:
		context.Logger().Info("Hello", slog.String("Who", msg.Who))
		panic("children： err")
	}
}

type start struct {
	msg string
}

type ParentActor struct{}

func (p ParentActor) Receive(context actor.Context) {
	switch msg := context.Message().(type) {
	// 这里的生命周期的消息全都是由这个框架发出的
	case *actor.Started:
		context.Logger().Info("Started, initialize parent actor here")
		// 这是 外部消息 打印
	case *start:
		context.Logger().Info("Hello", slog.String("Who", msg.msg))
		// 创建并启动子 actor
		props := actor.PropsFromProducer(func() actor.Actor {
			return &helloActor{}
		})
		pid := context.Spawn(props)
		context.Send(pid, &hello{Who: "Parent"})
	}
}

func main() {
	// 创建 actor
	system := actor.NewActorSystem()

	// 补充：父 Actor ，使得 helloActor 接收 restart 的消息

	props := actor.PropsFromProducer(func() actor.Actor {
		return &ParentActor{}
	})
	// 自定义错误处理方式
	decider := func(reason interface{}) actor.Directive {
		fmt.Println("handling failure for child")
		return actor.RestartDirective
	}
	// 1 1
	supervisionStrategy := actor.NewOneForOneStrategy(1, 10, decider)
	propOperation := actor.WithSupervisor(supervisionStrategy)
	props.Configure(propOperation)
	ppid := system.Root.Spawn(props)
	system.Root.Send(ppid, &start{msg: "Parent Coming!"})

	//props = actor.PropsFromProducer(func() actor.Actor { return &helloActor{} })
	//pid := system.Root.Spawn(props)

	// send
	// system.Root.Send(pid, &hello{Who: "Roger"})

	// why wait?
	// Stop is a system message and is not processed through the user message mailbox
	// thus, it will be handled _before_ any user message
	// we only do this to show the correct order of events in the console
	// 这里的意思是：消息处理顺序是 先处理系统消息、再处理用户消息。这里为了上面先执行，等待1s后再发送stop消息
	// time.Sleep(1 * time.Second)
	// system.Root.Stop(pid)

	_, _ = console.ReadLine()
}

```