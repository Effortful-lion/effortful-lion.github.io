---
title: "01 Hello"
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

# Actor 怎么接收并处理消息

## 第一次运行程序
![img.png](/assets/img.png)

## actor注册
![](static/assets/01-hello-1768532836010.png)

## RequestId 作用 和 Process作用
![img_1.png](/assets/img_1.png)

## 主要代码：
```go
func main() {
	// 创建 actorSystem
	system := actor.NewActorSystem()
	// 创建 producer 用于返回一个 actor 实例
	producer := func() actor.Actor { return &helloActor{} }
	// 创建配置项：配置工厂函数（producer）
	props := actor.PropsFromProducer(producer)

	// RootContext: 用于和 system 交互的入口（代理 system）
	// 根据 props 创建 PID
	// PID : Process + ID + Address(网络地址) + RequestId()
	// PID 是 protobuf 定义生成的：Protobuf 提供了一种标准化的序列化方式，使得 PID（Process Identifier）可以在网络上传输。
	/*
		交互接口：这是actor通过PID
		type Process interface {
			SendUserMessage(pid *PID, message interface{})
			SendSystemMessage(pid *PID, message interface{})
			Stop(pid *PID)
		}
	*/
	pid := system.Root.Spawn(props)
	// Send 异步，无阻塞无通知
	// Request 同步，阻塞有通知
	// 但是：
	system.Root.Send(pid, &hello{Who: "Roger"})
	/*
		底层调用：
		func (rc *RootContext) Send(pid, msg) {}
		func (rc *RootContext) sendUserMessage(pid *PID, message interface{}) {
			if rc.senderMiddleware != nil {
				// Request based middleware
				rc.senderMiddleware(rc, pid, WrapEnvelope(message))
			} else {
				// tell based middleware
				pid.sendUserMessage(rc.actorSystem, message)
			}
		}
	*/

	_, _ = console.ReadLine()
}
```

除了注释部分，补充：

> system.Root.Send(pid, &hello{Who: "Roger"})
```text
1. Send() [同级还有 Request]
2. sendUserMessage(),这里其实是根据是否有中间件划分的。没有就直接调用pid.sendUserMessage(rc.actorSystem, message)
3. 底层调用：
func (pid *PID) sendUserMessage(actorSystem *ActorSystem, message interface{}) {
	pid.ref(actorSystem).SendUserMessage(pid, message)
}
这其实还是 PID.Process.SendUserMessage
通过 ref() -> Process(有就是pid.p，没有就从 actorSystem.ProcessRegistryValue中找)
4. 最后就是调用 mailbox 的 SendUserMessage 方法把 msg 放到 mailbox 中
// SendUserMessage posts a user message to the actor's mailbox.
func (ref *ActorProcess) SendUserMessage(_ *PID, message interface{}) {
	ref.mailbox.PostUserMessage(message)
}
5. m.userMailbox.Push(message)
```

## 发送 用户消息 类型
![](/assets/01-hello-1768479027931.png)

1. batch message (批量 envelope )
2. envelope message (normal messages + header)
3. normal messages (简单消息、非批量 envelope)

```go
    // 发送消息到 mailbox
	m.userMailbox.Push(message)
    // 原子增加 数目
	atomic.AddInt32(&m.userMessages, 1)
	// 触发 process
    m.schedule()
```

## Process 
```go
func (m *defaultMailbox) processMessages() {
process:
	// 不断从队列中取出消息并交给 Actor 处理，直到队列变空或者达到本次运行的时间片上限。
	m.run()

	// set mailbox to idle
	atomic.StoreInt32(&m.schedulerStatus, idle)
	sys := atomic.LoadInt32(&m.sysMessages)
	user := atomic.LoadInt32(&m.userMessages)
	// check if there are still messages to process (sent after the message loop ended)
	if sys > 0 || (atomic.LoadInt32(&m.suspended) == 0 && user > 0) {
		// try setting the mailbox back to running
		if atomic.CompareAndSwapInt32(&m.schedulerStatus, idle, running) {
			//	fmt.Printf("looping %v %v %v\n", sys, user, m.suspended)
			goto process
		}
	}

	if user == 0 && (atomic.LoadInt32(&m.suspended) == 0) {
		for _, ms := range m.middlewares {
			ms.MailboxEmpty()
		}
	}
}
```
![](/assets/01-hello-1768479613712.png)
![](/assets/01-hello-1768479669673.png)
![](/assets/01-hello-1768480008821.png)
1. CAS 保证了只有一个 Goroutine 在跑
2. 如果两个协程同时试图把 Mailbox 从 idle 唤醒，并且没有cas，会出现数据竞态
```go
func (m *defaultMailbox) run() {
	var msg interface{}

	// 收到 panic
	// 将错误“升级”给该 Actor 的监督者（Supervisor）。由监督者决定是重启、停止还是恢复这个 Actor。
	defer func() {
		if r := recover(); r != nil {
			m.invoker.EscalateFailure(r, msg)
		}
	}()

	i, t := 0, m.dispatcher.Throughput()
	for {
		if i > t {
			i = 0
			runtime.Gosched()
		}

		i++

		// keep processing system messages until queue is empty
		if msg = m.systemMailbox.Pop(); msg != nil {
			atomic.AddInt32(&m.sysMessages, -1)
			switch msg.(type) {
			case *SuspendMailbox:
				atomic.StoreInt32(&m.suspended, 1)
			case *ResumeMailbox:
				atomic.StoreInt32(&m.suspended, 0)
			default:
				m.invoker.InvokeSystemMessage(msg)
			}
			for _, ms := range m.middlewares {
				ms.MessageReceived(msg)
			}
			continue
		}

		// 挂起机制，防止actor崩溃期间继续处理业务
		// didn't process a system message, so break until we are resumed
		if atomic.LoadInt32(&m.suspended) == 1 {
			return
		}

		if msg = m.userMailbox.Pop(); msg != nil {
			atomic.AddInt32(&m.userMessages, -1)
			m.invoker.InvokeUserMessage(msg)
			for _, ms := range m.middlewares {
				ms.MessageReceived(msg)
			}
		} else {
			return
		}
	}
}
```
这里不就是先处理系统消息，直到处理完，再处理用户消息

## 整理一下普通消息的异步 发送、接收、处理链路
下面所有路径都是函数，粒度有大有小：
```go
1. system.Root.Send(pid, &hello{Who: "Roger"}) —— 这是 main，发送消息
2. RootContext.sendUserMessage(pid, message)
3. RootContext.sendUserMessage(pid *PID, message interface{}) 
4. pid.sendUserMessage(rc.actorSystem, message)
5. pid.ref(actorSystem).SendUserMessage(pid, message)
6. ActorProcess.SendUserMessage
7. ActorProcess.mailbox.PostUserMessage(message)
8. mailbox.userMailbox.Push(message)  [计数+1：atomic.AddInt32(&m.userMessages, 1)]
9. mailbox.schedule() // 触发 run
10. mailbox.processMessages // -- run
11. mailbox.invoker.InvokeUserMessage(msg)
12. ctx.defaultReceive()
13. ctx.actor.Receive(ctx) // -- 这里的 ctx.actor = helloActor 
在 defaultSpawner 看到的 newActorContext，框架已经把你定义的 &helloActor{} 结构体实例赋值给了这个 ctx.actor 字段。
```
简单一点：
![](/assets/01-hello-1768484737245.png)