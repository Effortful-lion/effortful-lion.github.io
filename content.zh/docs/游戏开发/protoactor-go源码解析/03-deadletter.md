---
title: "03-deadletter"
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

# 基于Actor模型的系统中死信处理流程

## 出现
死信通常出现在不能建立网络连接或者远端actor此时不存在的时候。

## 作用
适用于错误处理和调试场景

## 概念

1. EventStream 这是 actorSystem 的全局事件流：所有框架核心事件（如死信、Actor 终止、重启等）
2. 所以死信会被发布到这里，如果需要收集，只需要订阅就好了

## 流程

1. 触发：消息发往无效 PID → 框架判定为死信；
2. 分发：框架封装死信事件，发布到全局 EventStream，同时完成可选的监控计数；
3. 消费：框架默认打印限流日志，业务层可订阅事件实现自定义收集 / 统计 / 告警（多订阅者共享事件，互不影响）。

> 补充：关键特性
死信是 “一次性事件”：框架不持久化，未订阅则丢失；
性能注意：高并发下需异步转发死信事件，避免阻塞 EventStream 的同步分发；
扩展能力：业务层可基于死信收集节点，实现生产级的死信持久化、重试、告警闭环。


## 死信订阅

这个代码为了验证：全局事件流会将死信拷贝发送给所有订阅者
```go
package main

import (
	"context"
	"flag"
	"log"
	"sync/atomic"
	"time"

	"github.com/asynkron/protoactor-go/actor"
	"golang.org/x/time/rate"
)

// 1. 定义我们业务消息
type hello struct {
	Who string
}

// 2. 定义死信收集Actor的处理逻辑
type deadLetterCollector struct{}

func (state *deadLetterCollector) Receive(ctx actor.Context) {
	switch msg := ctx.Message().(type) {
	case *actor.DeadLetterEvent:
		log.Printf("【死信收集器】检测到死信！目标: %v | 消息类型: %T | 内容: %+v",
			msg.PID, msg.Message, msg.Message)
	case *actor.Started:
		log.Println("【死信收集器】已启动，开始监控全局事件流...")
	}
}

func main() {
	// ========== 参数解析 ==========
	irate := flag.Int("rate", 5, "每秒发送的消息数 (设低一点方便观察)")
	flag.Parse()

	// ========== 1. 初始化 ActorSystem ==========
	// 我们把系统的死信日志阈值设为 0，让所有死信都打印出来
	cfg := actor.Configure(actor.WithDeadLetterThrottleCount(0))
	system := actor.NewActorSystemWithConfig(cfg)

	// ========== 2. 启动死信收集 Actor ==========
	collectorProps := actor.PropsFromProducer(func() actor.Actor { return &deadLetterCollector{} })
	collectorPid := system.Root.Spawn(collectorProps)

	// ========== 3. 订阅全局事件流 (EventStream) ==========
	// 这是核心：框架把死信发布到 EventStream，我们订阅它并转发给我们的 collectorActor
	system.EventStream.Subscribe(func(msg interface{}) {
		if deadLetter, ok := msg.(*actor.DeadLetterEvent); ok {
			system.Root.Send(collectorPid, deadLetter)
		}
	})

	// ========== 4. 构造一个会产生死信的场景 ==========
	// 场景：给一个已经停止的 Actor 发消息
	log.Println("--- 场景 A：给已停止的 Actor 发消息 ---")
	tempProps := actor.PropsFromFunc(func(ctx actor.Context) {})
	tempPid := system.Root.Spawn(tempProps)
	system.Root.Stop(tempPid) // 杀掉它

	// 发送消息，这会产生死信
	system.Root.Send(tempPid, &hello{Who: "Ghost User"})

	// ========== 5. 持续发送给不存在的 PID ==========
	log.Println("--- 场景 B：持续发送给无效 PID (unknown) ---")
	invalidPid := system.NewLocalPID("unknown")
	limiter := rate.NewLimiter(rate.Limit(*irate), 1)

	// 运行 3 秒
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	run := int32(1)
	go func() {
		<-ctx.Done()
		atomic.StoreInt32(&run, 0)
	}()

	for atomic.LoadInt32(&run) == 1 {
		system.Root.Send(invalidPid, &hello{Who: "DeadLetter Test"})
		limiter.Wait(context.Background())
	}

	log.Println("测试结束，正在关闭系统...")
	system.Root.Stop(collectorPid)
	log.Println("Done.")
}
```
