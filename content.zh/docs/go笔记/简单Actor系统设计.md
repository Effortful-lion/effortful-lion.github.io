---
title: "简单Actor系统设计"
date: 2026-01-11T12:44:36+08:00
# bookComments: false
# bookSearchExclude: false
# bookPostThumbnail: thumbnail.*
---

## 1. 框架设计目标

设计一个简单、组件化的Actor框架，包含以下核心特性：

- **简洁的Actor模型**：遵循经典Actor模型，支持消息传递、封装状态和行为
- **组件化设计**：将框架拆分为独立、可复用的组件
- **轻量级实现**：最小化依赖，易于理解和扩展
- **高性能**：支持高效的消息处理和并发

## 2. 核心组件设计

### 2.1 Actor接口

```go
// Actor 定义Actor的核心接口
type Actor interface {
    // Receive 处理接收到的消息
    Receive(ctx Context)
}
```

### 2.2 Context接口

```go
// Context 提供Actor执行上下文
type Context interface {
    // Self 返回当前Actor的PID
    Self() *PID
    
    // Sender 返回发送当前消息的Actor的PID
    Sender() *PID
    
    // Message 返回当前处理的消息
    Message() interface{}
    
    // Send 发送消息给指定PID的Actor
    Send(pid *PID, message interface{})
    
    // Spawn 创建新的子Actor
    Spawn(props *Props) *PID
    
    // Stop 停止指定PID的Actor
    Stop(pid *PID)
}
```

### 2.3 PID (Process ID)

```go
// PID 表示Actor的唯一标识符
type PID struct {
    // Address 表示Actor所在的地址
    Address string
    
    // ID 表示Actor的唯一标识
    ID string
}
```

### 2.4 Props

```go
// Props 包含创建Actor的配置信息
type Props struct {
    // Producer 用于创建Actor实例的函数
    Producer func() Actor
    
    // MailboxProducer 用于创建邮箱的函数
    MailboxProducer func() Mailbox
    
    // Dispatcher 用于调度Actor的调度器
    Dispatcher Dispatcher
}
```

### 2.5 Mailbox

```go
// Mailbox 用于存储和处理消息
type Mailbox interface {
    // PostUserMessage 发送用户消息
    PostUserMessage(message interface{})
    
    // PostSystemMessage 发送系统消息
    PostSystemMessage(message interface{})
    
    // RegisterHandlers 注册消息处理器
    RegisterHandlers(invoker MessageInvoker, dispatcher Dispatcher)
    
    // Start 启动邮箱
    Start()
}
```

### 2.6 Dispatcher

```go
// Dispatcher 用于调度Actor的消息处理
type Dispatcher interface {
    // Schedule 调度任务执行
    Schedule(task func())
}
```

### 2.7 ActorSystem

```go
// ActorSystem 是Actor框架的运行时环境
type ActorSystem struct {
    // Root 根上下文，用于创建顶级Actor
    Root *RootContext
    
    // ProcessRegistry 用于注册和查找Actor
    ProcessRegistry *ProcessRegistry
    
    // Config 系统配置
    Config *Config
}
```

## 3. 组件化架构

框架采用组件化设计，各组件之间通过接口进行通信，实现松耦合：

```
+----------------+     +----------------+     +----------------+
|                |     |                |     |                |
|  ActorSystem   |     |   Process      |     |    Mailbox     |
|                |     |   Registry     |     |                |
+-------+--------+     +-------+--------+     +--------+-------+
        |                      |                       |
        v                      v                       v
+-------+--------+     +-------+--------+     +--------+-------+
|                |     |                |     |                |
|    Context     |     |     Actor      |     |   Dispatcher   |
|                |     |                |     |                |
+----------------+     +----------------+     +----------------+
```

## 4. 聊天系统设计

### 4.1 系统架构

聊天系统基于上述Actor框架实现，包含以下核心组件：

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  UserActor       |<--->|  RoomActor       |<--->|  RoomManager     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```

### 4.2 消息类型设计

#### 用户相关消息

```go
// UserJoin 用户加入系统
type UserJoin struct {
    Username string
}

// UserLeave 用户离开系统
type UserLeave struct {
    Username string
}

// UserMessage 用户消息
type UserMessage struct {
    Username string
    Content  string
    Time     time.Time
}
```

#### 房间相关消息

```go
// CreateRoom 创建房间
type CreateRoom struct {
    RoomName string
}

// JoinRoom 加入房间
type JoinRoom struct {
    Username string
    RoomName string
    UserPID  *PID
}

// LeaveRoom 离开房间
type LeaveRoom struct {
    Username string
    RoomName string
}

// RoomMessage 房间消息
type RoomMessage struct {
    Username string
    RoomName string
    Content  string
    Time     time.Time
}

// ListRooms 列出房间
type ListRooms struct {}

// RoomList 房间列表响应
type RoomList struct {
    Rooms []string
}
```

#### 响应消息

```go
// Response 通用响应消息
type Response struct {
    Success bool
    Message string
}
```

### 4.3 Actor设计

#### UserActor

```go
// UserActor 处理用户相关的消息
type UserActor struct {
    username string
    rooms    map[string]*PID // 房间名称到房间PID的映射
}

func (u *UserActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *UserJoin:
        u.username = msg.Username
        u.rooms = make(map[string]*PID)
        fmt.Printf("用户 %s 加入系统\n", u.username)

    case *UserMessage:
        fmt.Printf("[%s] %s: %s\n", msg.Time.Format("15:04:05"), msg.Username, msg.Content)

    case *Response:
        fmt.Printf("响应: %t - %s\n", msg.Success, msg.Message)

    // 处理其他消息...
    }
}
```

#### RoomActor

```go
// RoomActor 处理房间相关的消息
type RoomActor struct {
    name     string
    users    map[string]*PID // 用户名到用户PID的映射
    messages []UserMessage   // 聊天记录
}

func (r *RoomActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *JoinRoom:
        if _, exists := r.users[msg.Username]; exists {
            ctx.Send(msg.UserPID, &Response{Success: false, Message: fmt.Sprintf("用户 %s 已在房间 %s 中", msg.Username, r.name)})
            return
        }
        r.users[msg.Username] = msg.UserPID
        // 通知房间内所有用户
        for _, userPID := range r.users {
            userPID.SendUserMessage(ctx.ActorSystem(), &UserMessage{
                Username: "系统",
                Content:  fmt.Sprintf("用户 %s 加入了房间", msg.Username),
                Time:     time.Now(),
            })
        }

    case *RoomMessage:
        // 保存消息
        userMsg := UserMessage{
            Username: msg.Username,
            Content:  msg.Content,
            Time:     msg.Time,
        }
        r.messages = append(r.messages, userMsg)
        // 广播消息给房间内所有用户
        for _, userPID := range r.users {
            userPID.SendUserMessage(ctx.ActorSystem(), &userMsg)
        }

    // 处理其他消息...
    }
}
```

#### RoomManager

```go
// RoomManager 管理所有房间
type RoomManager struct {
    rooms map[string]*PID // 房间名称到房间PID的映射
}

func (rm *RoomManager) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *CreateRoom:
        if _, exists := rm.rooms[msg.RoomName]; exists {
            ctx.Respond(&Response{Success: false, Message: fmt.Sprintf("房间 %s 已存在", msg.RoomName)})
            return
        }
        // 创建新房间Actor
        roomProps := PropsFromProducer(func() Actor {
            return &RoomActor{name: msg.RoomName}
        })
        roomPID := ctx.Spawn(roomProps)
        rm.rooms[msg.RoomName] = roomPID
        ctx.Respond(&Response{Success: true, Message: fmt.Sprintf("房间 %s 创建成功", msg.RoomName)})

    case *JoinRoom:
        if roomPID, exists := rm.rooms[msg.RoomName]; exists {
            // 转发加入请求到对应的房间Actor
            ctx.Forward(roomPID)
        } else {
            ctx.Send(msg.UserPID, &Response{Success: false, Message: fmt.Sprintf("房间 %s 不存在", msg.RoomName)})
        }

    // 处理其他消息...
    }
}
```

### 4.4 系统流程

1. **初始化**：创建Actor系统和房间管理器
2. **用户加入**：创建用户Actor并加入系统
3. **房间管理**：创建、加入、离开房间
4. **消息传递**：在房间内发送和接收消息

```go
func main() {
    // 创建Actor系统
    system := NewActorSystem()

    // 创建房间管理器
    roomManagerProps := PropsFromProducer(func() Actor {
        return &RoomManager{}
    })
    roomManagerPID := system.Root.Spawn(roomManagerProps)

    // 创建用户
    userProps := PropsFromProducer(func() Actor {
        return &UserActor{}
    })
    userPID := system.Root.Spawn(userProps)

    // 用户加入系统
    system.Root.Send(userPID, &UserJoin{Username: "Alice"})

    // 创建房间
    response, _ := system.Root.RequestFuture(roomManagerPID, &CreateRoom{RoomName: "General"}, 5*time.Second).Result()
    if resp, ok := response.(*Response); ok {
        fmt.Printf("创建房间响应: %t - %s\n", resp.Success, resp.Message)
    }

    // 加入房间
    system.Root.Send(roomManagerPID, &JoinRoom{
        Username: "Alice",
        RoomName: "General",
        UserPID:  userPID,
    })

    // 发送消息
    system.Root.Send(roomManagerPID, &RoomMessage{
        Username: "Alice",
        RoomName: "General",
        Content:  "大家好！",
        Time:     time.Now(),
    })

    // 等待用户输入
    console.ReadLine()
}
```

## 5. 扩展性设计

框架设计考虑了扩展性，支持以下扩展方式：

1. **中间件**：可以在消息处理前后添加中间件，用于日志、监控、安全等
2. **自定义调度器**：支持替换默认的调度器实现
3. **自定义邮箱**：支持不同类型的邮箱实现，如无界邮箱、有界邮箱等
4. **远程通信**：可以扩展为支持远程Actor通信

## 6. 性能考虑

1. **异步消息处理**：所有消息处理都是异步的，避免阻塞
2. **高效的邮箱实现**：使用锁-free队列提高性能
3. **合理的调度策略**：根据系统负载动态调整调度策略
4. **最小化内存分配**：减少不必要的内存分配和垃圾回收

## 7. 总结

本设计提供了一个简单、组件化的Actor框架，以及基于此框架的聊天系统实现。框架遵循经典Actor模型，支持消息传递、封装状态和行为，具有良好的扩展性和性能。聊天系统实现了用户管理、房间管理和消息传递等核心功能，可以作为基于Actor模型构建分布式系统的参考。

oldStr:


我将先检查目录结构，然后创建设计文档。


toolName: view_folder

status: failed

d:\protoactor-go\examples\chat-system

我将先创建目录，然后创建设计文档。我将使用show_diff工具来创建设计文档。


toolName: show_diff

status: success


filePath: d:\protoactor-go\examples\chat-system\设计.md

changes:

newStr: # 核心组件化Actor框架设计与聊天系统实现

## 1. 框架设计目标

设计一个简单、组件化的Actor框架，包含以下核心特性：

- **简洁的Actor模型**：遵循经典Actor模型，支持消息传递、封装状态和行为
- **组件化设计**：将框架拆分为独立、可复用的组件
- **轻量级实现**：最小化依赖，易于理解和扩展
- **高性能**：支持高效的消息处理和并发

## 2. 核心组件设计

### 2.1 Actor接口

```go
// Actor 定义Actor的核心接口
type Actor interface {
    // Receive 处理接收到的消息
    Receive(ctx Context)
}
```

### 2.2 Context接口

```go
// Context 提供Actor执行上下文
type Context interface {
    // Self 返回当前Actor的PID
    Self() *PID
    
    // Sender 返回发送当前消息的Actor的PID
    Sender() *PID
    
    // Message 返回当前处理的消息
    Message() interface{}
    
    // Send 发送消息给指定PID的Actor
    Send(pid *PID, message interface{})
    
    // Spawn 创建新的子Actor
    Spawn(props *Props) *PID
    
    // Stop 停止指定PID的Actor
    Stop(pid *PID)
}
```

### 2.3 PID (Process ID)

```go
// PID 表示Actor的唯一标识符
type PID struct {
    // Address 表示Actor所在的地址
    Address string
    
    // ID 表示Actor的唯一标识
    ID string
}
```

### 2.4 Props

```go
// Props 包含创建Actor的配置信息
type Props struct {
    // Producer 用于创建Actor实例的函数
    Producer func() Actor
    
    // MailboxProducer 用于创建邮箱的函数
    MailboxProducer func() Mailbox
    
    // Dispatcher 用于调度Actor的调度器
    Dispatcher Dispatcher
}
```

### 2.5 Mailbox

```go
// Mailbox 用于存储和处理消息
type Mailbox interface {
    // PostUserMessage 发送用户消息
    PostUserMessage(message interface{})
    
    // PostSystemMessage 发送系统消息
    PostSystemMessage(message interface{})
    
    // RegisterHandlers 注册消息处理器
    RegisterHandlers(invoker MessageInvoker, dispatcher Dispatcher)
    
    // Start 启动邮箱
    Start()
}
```

### 2.6 Dispatcher

```go
// Dispatcher 用于调度Actor的消息处理
type Dispatcher interface {
    // Schedule 调度任务执行
    Schedule(task func())
}
```

### 2.7 ActorSystem

```go
// ActorSystem 是Actor框架的运行时环境
type ActorSystem struct {
    // Root 根上下文，用于创建顶级Actor
    Root *RootContext
    
    // ProcessRegistry 用于注册和查找Actor
    ProcessRegistry *ProcessRegistry
    
    // Config 系统配置
    Config *Config
}
```

## 3. 组件化架构

框架采用组件化设计，各组件之间通过接口进行通信，实现松耦合：

```
+----------------+     +----------------+     +----------------+
|                |     |                |     |                |
|  ActorSystem   |     |   Process      |     |    Mailbox     |
|                |     |   Registry     |     |                |
+-------+--------+     +-------+--------+     +--------+-------+
        |                      |                       |
        v                      v                       v
+-------+--------+     +-------+--------+     +--------+-------+
|                |     |                |     |                |
|    Context     |     |     Actor      |     |   Dispatcher   |
|                |     |                |     |                |
+----------------+     +----------------+     +----------------+
```

## 4. 聊天系统设计

### 4.1 系统架构

聊天系统基于上述Actor框架实现，包含以下核心组件：

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  UserActor       |<--->|  RoomActor       |<--->|  RoomManager     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```

### 4.2 消息类型设计

#### 用户相关消息

```go
// UserJoin 用户加入系统
type UserJoin struct {
    Username string
}

// UserLeave 用户离开系统
type UserLeave struct {
    Username string
}

// UserMessage 用户消息
type UserMessage struct {
    Username string
    Content  string
    Time     time.Time
}
```

#### 房间相关消息

```go
// CreateRoom 创建房间
type CreateRoom struct {
    RoomName string
}

// JoinRoom 加入房间
type JoinRoom struct {
    Username string
    RoomName string
    UserPID  *PID
}

// LeaveRoom 离开房间
type LeaveRoom struct {
    Username string
    RoomName string
}

// RoomMessage 房间消息
type RoomMessage struct {
    Username string
    RoomName string
    Content  string
    Time     time.Time
}

// ListRooms 列出房间
type ListRooms struct {}

// RoomList 房间列表响应
type RoomList struct {
    Rooms []string
}
```

#### 响应消息

```go
// Response 通用响应消息
type Response struct {
    Success bool
    Message string
}
```

### 4.3 Actor设计

#### UserActor

```go
// UserActor 处理用户相关的消息
type UserActor struct {
    username string
    rooms    map[string]*PID // 房间名称到房间PID的映射
}

func (u *UserActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *UserJoin:
        u.username = msg.Username
        u.rooms = make(map[string]*PID)
        fmt.Printf("用户 %s 加入系统\n", u.username)

    case *UserMessage:
        fmt.Printf("[%s] %s: %s\n", msg.Time.Format("15:04:05"), msg.Username, msg.Content)

    case *Response:
        fmt.Printf("响应: %t - %s\n", msg.Success, msg.Message)

    // 处理其他消息...
    }
}
```

#### RoomActor

```go
// RoomActor 处理房间相关的消息
type RoomActor struct {
    name     string
    users    map[string]*PID // 用户名到用户PID的映射
    messages []UserMessage   // 聊天记录
}

func (r *RoomActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *JoinRoom:
        if _, exists := r.users[msg.Username]; exists {
            ctx.Send(msg.UserPID, &Response{Success: false, Message: fmt.Sprintf("用户 %s 已在房间 %s 中", msg.Username, r.name)})
            return
        }
        r.users[msg.Username] = msg.UserPID
        // 通知房间内所有用户
        for _, userPID := range r.users {
            userPID.SendUserMessage(ctx.ActorSystem(), &UserMessage{
                Username: "系统",
                Content:  fmt.Sprintf("用户 %s 加入了房间", msg.Username),
                Time:     time.Now(),
            })
        }

    case *RoomMessage:
        // 保存消息
        userMsg := UserMessage{
            Username: msg.Username,
            Content:  msg.Content,
            Time:     msg.Time,
        }
        r.messages = append(r.messages, userMsg)
        // 广播消息给房间内所有用户
        for _, userPID := range r.users {
            userPID.SendUserMessage(ctx.ActorSystem(), &userMsg)
        }

    // 处理其他消息...
    }
}
```

#### RoomManager

```go
// RoomManager 管理所有房间
type RoomManager struct {
    rooms map[string]*PID // 房间名称到房间PID的映射
}

func (rm *RoomManager) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *CreateRoom:
        if _, exists := rm.rooms[msg.RoomName]; exists {
            ctx.Respond(&Response{Success: false, Message: fmt.Sprintf("房间 %s 已存在", msg.RoomName)})
            return
        }
        // 创建新房间Actor
        roomProps := PropsFromProducer(func() Actor {
            return &RoomActor{name: msg.RoomName}
        })
        roomPID := ctx.Spawn(roomProps)
        rm.rooms[msg.RoomName] = roomPID
        ctx.Respond(&Response{Success: true, Message: fmt.Sprintf("房间 %s 创建成功", msg.RoomName)})

    case *JoinRoom:
        if roomPID, exists := rm.rooms[msg.RoomName]; exists {
            // 转发加入请求到对应的房间Actor
            ctx.Forward(roomPID)
        } else {
            ctx.Send(msg.UserPID, &Response{Success: false, Message: fmt.Sprintf("房间 %s 不存在", msg.RoomName)})
        }

    // 处理其他消息...
    }
}
```

### 4.4 系统流程

1. **初始化**：创建Actor系统和房间管理器
2. **用户加入**：创建用户Actor并加入系统
3. **房间管理**：创建、加入、离开房间
4. **消息传递**：在房间内发送和接收消息

```go
func main() {
    // 创建Actor系统
    system := NewActorSystem()

    // 创建房间管理器
    roomManagerProps := PropsFromProducer(func() Actor {
        return &RoomManager{}
    })
    roomManagerPID := system.Root.Spawn(roomManagerProps)

    // 创建用户
    userProps := PropsFromProducer(func() Actor {
        return &UserActor{}
    })
    userPID := system.Root.Spawn(userProps)

    // 用户加入系统
    system.Root.Send(userPID, &UserJoin{Username: "Alice"})

    // 创建房间
    response, _ := system.Root.RequestFuture(roomManagerPID, &CreateRoom{RoomName: "General"}, 5*time.Second).Result()
    if resp, ok := response.(*Response); ok {
        fmt.Printf("创建房间响应: %t - %s\n", resp.Success, resp.Message)
    }

    // 加入房间
    system.Root.Send(roomManagerPID, &JoinRoom{
        Username: "Alice",
        RoomName: "General",
        UserPID:  userPID,
    })

    // 发送消息
    system.Root.Send(roomManagerPID, &RoomMessage{
        Username: "Alice",
        RoomName: "General",
        Content:  "大家好！",
        Time:     time.Now(),
    })

    // 等待用户输入
    console.ReadLine()
}
```

## 5. 扩展性设计

框架设计考虑了扩展性，支持以下扩展方式：

1. **中间件**：可以在消息处理前后添加中间件，用于日志、监控、安全等
2. **自定义调度器**：支持替换默认的调度器实现
3. **自定义邮箱**：支持不同类型的邮箱实现，如无界邮箱、有界邮箱等
4. **远程通信**：可以扩展为支持远程Actor通信

## 6. 性能考虑

1. **异步消息处理**：所有消息处理都是异步的，避免阻塞
2. **高效的邮箱实现**：使用锁-free队列提高性能
3. **合理的调度策略**：根据系统负载动态调整调度策略
4. **最小化内存分配**：减少不必要的内存分配和垃圾回收

## 7. 总结

本设计提供了一个简单、组件化的Actor框架，以及基于此框架的聊天系统实现。框架遵循经典Actor模型，支持消息传递、封装状态和行为，具有良好的扩展性和性能。聊天系统实现了用户管理、房间管理和消息传递等核心功能，可以作为基于Actor模型构建分布式系统的参考。

oldStr:


我将创建一个更简洁的设计文档，确保它能被正确生成。


toolName: show_diff

status: success


filePath: d:\protoactor-go\设计.md

changes:

newStr: # 核心组件化Actor框架与聊天系统设计

## 1. 框架核心组件

### Actor接口
```go
type Actor interface {
    Receive(ctx Context)
}
```

### Context接口
```go
type Context interface {
    Self() *PID
    Sender() *PID
    Message() interface{}
    Send(pid *PID, message interface{})
    Spawn(props *Props) *PID
    Stop(pid *PID)
}
```

### PID
```go
type PID struct {
    Address string
    ID      string
}
```

### Props
```go
type Props struct {
    Producer func() Actor
}
```

### ActorSystem
```go
type ActorSystem struct {
    Root *RootContext
}
```

## 2. 聊天系统设计

### 2.1 消息类型
```go
// 用户消息
type UserMessage struct {
    Username string
    Content  string
}

// 房间消息
type RoomMessage struct {
    Username string
    RoomName string
    Content  string
}
```

### 2.2 UserActor
```go
type UserActor struct {
    username string
}

func (u *UserActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *UserMessage:
        fmt.Printf("%s: %s\n", msg.Username, msg.Content)
    }
}
```

### 2.3 RoomActor
```go
type RoomActor struct {
    name  string
    users map[string]*PID
}

func (r *RoomActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *RoomMessage:
        for _, user := range r.users {
            user.Tell(&UserMessage{
                Username: msg.Username,
                Content:  msg.Content,
            })
        }
    }
}
```

### 2.4 RoomManager
```go
type RoomManager struct {
    rooms map[string]*PID
}

func (rm *RoomManager) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *RoomMessage:
        if room, exists := rm.rooms[msg.RoomName]; exists {
            room.Tell(msg)
        }
    }
}
```

## 3. 系统流程

1. 创建ActorSystem
2. 创建RoomManager
3. 创建UserActor
4. 用户发送消息到特定房间
5. 房间广播消息给所有用户

## 4. 扩展性

- 支持中间件
- 支持自定义调度器
- 支持远程通信


oldStr:


我将创建一个完整的设计文档，包含核心Actor框架设计和聊天系统实现方案。


toolName: show_diff

status: success


filePath: d:\protoactor-go\设计.md

changes:

newStr: # 核心组件化Actor框架设计与聊天系统实现

## 1. 框架设计概述

### 1.1 设计目标

设计一个简单、组件化的Actor框架，具备以下特性：
- 遵循经典Actor模型，支持消息传递、状态封装
- 组件化架构，各模块独立可扩展
- 轻量级实现，易于理解和使用
- 高性能消息处理

### 1.2 核心概念

- **Actor**：封装状态和行为的计算单元，通过消息与外界通信
- **消息**：Actor间通信的载体，不可变对象
- **PID**：Actor的唯一标识符
- **Context**：Actor的执行上下文，提供消息处理的环境
- **ActorSystem**：Actor的运行时环境

## 2. 核心组件设计

### 2.1 Actor接口

```go
// Actor 定义Actor的核心接口
type Actor interface {
    // Receive 处理接收到的消息
    Receive(ctx Context)
}
```

### 2.2 Context接口

```go
// Context 提供Actor的执行上下文
type Context interface {
    // Self 返回当前Actor的PID
    Self() *PID
    
    // Sender 返回发送当前消息的Actor的PID
    Sender() *PID
    
    // Message 返回当前处理的消息
    Message() interface{}
    
    // Send 发送消息给指定Actor
    Send(pid *PID, message interface{})
    
    // Spawn 创建新的子Actor
    Spawn(props *Props) *PID
    
    // Stop 停止指定Actor
    Stop(pid *PID)
}
```

### 2.3 PID (Process ID)

```go
// PID 表示Actor的唯一标识符
type PID struct {
    // Address Actor所在的地址
    Address string
    
    // ID Actor的唯一标识
    ID string
}
```

### 2.4 Props

```go
// Props 包含创建Actor的配置信息
type Props struct {
    // Producer 用于创建Actor实例的函数
    Producer func() Actor
}
```

### 2.5 ActorSystem

```go
// ActorSystem 是Actor框架的运行时环境
type ActorSystem struct {
    // Root 根上下文，用于创建顶级Actor
    Root *RootContext
    
    // ProcessRegistry 用于注册和查找Actor
    ProcessRegistry *ProcessRegistry
}
```

## 3. 聊天系统设计

### 3.1 系统架构

聊天系统基于Actor框架实现，包含以下核心组件：

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  UserActor       |<--->|  RoomActor       |<--->|  RoomManager     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```

### 3.2 消息类型设计

#### 用户相关消息

```go
// UserJoin 用户加入系统
type UserJoin struct {
    Username string
}

// UserLeave 用户离开系统
type UserLeave struct {
    Username string
}

// UserMessage 用户接收的消息
type UserMessage struct {
    Username string
    Content  string
    Time     time.Time
}
```

#### 房间相关消息

```go
// CreateRoom 创建房间
type CreateRoom struct {
    RoomName string
}

// JoinRoom 加入房间
type JoinRoom struct {
    Username string
    RoomName string
    UserPID  *PID
}

// LeaveRoom 离开房间
type LeaveRoom struct {
    Username string
    RoomName string
}

// RoomMessage 发送到房间的消息
type RoomMessage struct {
    Username string
    RoomName string
    Content  string
    Time     time.Time
}

// ListRooms 请求房间列表
type ListRooms struct {}

// RoomList 房间列表响应
type RoomList struct {
    Rooms []string
}
```

### 3.3 Actor实现

#### UserActor

```go
// UserActor 处理用户相关的消息
type UserActor struct {
    username string
    rooms    map[string]*PID // 已加入的房间
}

func (u *UserActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *UserJoin:
        u.username = msg.Username
        u.rooms = make(map[string]*PID)
        fmt.Printf("用户 %s 加入系统\n", u.username)

    case *UserMessage:
        fmt.Printf("[%s] %s: %s\n", msg.Time.Format("15:04:05"), msg.Username, msg.Content)

    case *UserLeave:
        fmt.Printf("用户 %s 离开系统\n", u.username)
        u.rooms = nil

    // 系统消息处理
    case *actor.Started:
        fmt.Println("UserActor已启动")
    }
}
```

#### RoomActor

```go
// RoomActor 处理房间相关的消息
type RoomActor struct {
    name     string
    users    map[string]*PID // 房间内的用户
    messages []UserMessage   // 聊天记录
}

func (r *RoomActor) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *JoinRoom:
        if _, exists := r.users[msg.Username]; exists {
            return // 用户已在房间内
        }
        r.users[msg.Username] = msg.UserPID
        fmt.Printf("用户 %s 加入房间 %s\n", msg.Username, r.name)
        
        // 通知房间内所有用户
        for _, userPID := range r.users {
            ctx.Send(userPID, &UserMessage{
                Username: "系统",
                Content:  fmt.Sprintf("用户 %s 加入了房间", msg.Username),
                Time:     time.Now(),
            })
        }

    case *LeaveRoom:
        if _, exists := r.users[msg.Username]; !exists {
            return // 用户不在房间内
        }
        delete(r.users, msg.Username)
        fmt.Printf("用户 %s 离开房间 %s\n", msg.Username, r.name)
        
        // 通知房间内所有用户
        for _, userPID := range r.users {
            ctx.Send(userPID, &UserMessage{
                Username: "系统",
                Content:  fmt.Sprintf("用户 %s 离开了房间", msg.Username),
                Time:     time.Now(),
            })
        }

    case *RoomMessage:
        // 保存消息
        userMsg := UserMessage{
            Username: msg.Username,
            Content:  msg.Content,
            Time:     msg.Time,
        }
        r.messages = append(r.messages, userMsg)
        
        // 广播消息给房间内所有用户
        for _, userPID := range r.users {
            ctx.Send(userPID, &userMsg)
        }

    case *actor.Started:
        r.users = make(map[string]*PID)
        r.messages = make([]UserMessage, 0)
        fmt.Printf("RoomActor %s 已启动\n", r.name)
    }
}
```

#### RoomManager

```go
// RoomManager 管理所有房间
type RoomManager struct {
    rooms map[string]*PID // 房间名称到房间PID的映射
}

func (rm *RoomManager) Receive(ctx Context) {
    switch msg := ctx.Message().(type) {
    case *CreateRoom:
        if _, exists := rm.rooms[msg.RoomName]; exists {
            return // 房间已存在
        }
        
        // 创建新房间Actor
        roomProps := actor.PropsFromProducer(func() actor.Actor {
            return &RoomActor{name: msg.RoomName}
        })
        roomPID := ctx.Spawn(roomProps)
        rm.rooms[msg.RoomName] = roomPID
        fmt.Printf("房间 %s 创建成功\n", msg.RoomName)

    case *JoinRoom:
        if roomPID, exists := rm.rooms[msg.RoomName]; exists {
            // 转发加入请求到房间Actor
            ctx.Forward(roomPID)
        } else {
            fmt.Printf("房间 %s 不存在\n", msg.RoomName)
        }

    case *LeaveRoom:
        if roomPID, exists := rm.rooms[msg.RoomName]; exists {
            // 转发离开请求到房间Actor
            ctx.Forward(roomPID)
        }

    case *RoomMessage:
        if roomPID, exists := rm.rooms[msg.RoomName]; exists {
            // 转发消息到房间Actor
            ctx.Forward(roomPID)
        }

    case *ListRooms:
        rooms := make([]string, 0, len(rm.rooms))
        for roomName := range rm.rooms {
            rooms = append(rooms, roomName)
        }
        ctx.Respond(&RoomList{Rooms: rooms})

    case *actor.Started:
        rm.rooms = make(map[string]*PID)
        fmt.Println("RoomManager已启动")
    }
}
```

### 3.4 系统初始化与流程

```go
func main() {
    // 创建Actor系统
    system := actor.NewActorSystem()

    // 创建房间管理器
    roomManagerProps := actor.PropsFromProducer(func() actor.Actor {
        return &RoomManager{}
    })
    roomManagerPID := system.Root.Spawn(roomManagerProps)

    // 创建用户Alice
    aliceProps := actor.PropsFromProducer(func() actor.Actor {
        return &UserActor{}
    })
    alicePID := system.Root.Spawn(aliceProps)

    // Alice加入系统
    system.Root.Send(alicePID, &UserJoin{Username: "Alice"})

    // 创建房间
    system.Root.Send(roomManagerPID, &CreateRoom{RoomName: "General"})

    // Alice加入房间
    system.Root.Send(roomManagerPID, &JoinRoom{
        Username: "Alice",
        RoomName: "General",
        UserPID:  alicePID,
    })

    // Alice发送消息
    system.Root.Send(roomManagerPID, &RoomMessage{
        Username: "Alice",
        RoomName: "General",
        Content:  "大家好！",
        Time:     time.Now(),
    })

    // 创建用户Bob
    bobProps := actor.PropsFromProducer(func() actor.Actor {
        return &UserActor{}
    })
    bobPID := system.Root.Spawn(bobProps)

    // Bob加入系统和房间
    system.Root.Send(bobPID, &UserJoin{Username: "Bob"})
    system.Root.Send(roomManagerPID, &JoinRoom{
        Username: "Bob",
        RoomName: "General",
        UserPID:  bobPID,
    })

    // Bob发送消息
    system.Root.Send(roomManagerPID, &RoomMessage{
        Username: "Bob",
        RoomName: "General",
        Content:  "你好，Alice！",
        Time:     time.Now(),
    })

    // 等待用户输入
    fmt.Println("按任意键退出...")
    console.ReadLine()

    // 清理资源
    system.Root.Stop(alicePID)
    system.Root.Stop(bobPID)
    system.Root.Stop(roomManagerPID)
}
```

## 4. 扩展性设计

### 4.1 中间件支持

框架支持在消息处理前后添加中间件，用于：
- 日志记录
- 性能监控
- 安全验证

### 4.2 自定义调度器

支持替换默认的消息调度器，以适应不同的性能需求。

### 4.3 远程通信

可扩展为支持跨网络的Actor通信，实现分布式聊天系统。

### 4.4 持久化

可添加消息持久化功能，保存聊天记录和用户状态。

## 5. 性能优化

1. **异步消息处理**：所有消息处理都是异步的，避免阻塞
2. **高效的消息队列**：使用高效的队列实现消息传递
3. **减少内存分配**：通过对象池等技术减少内存分配
4. **合理的并发模型**：根据系统负载调整并发度

## 6. 总结

本设计提供了一个简单、组件化的Actor框架，以及基于此框架的聊天系统实现。框架遵循经典Actor模型，具备良好的扩展性和性能。聊天系统实现了用户管理、房间管理和消息传递等核心功能，可作为基于Actor模型构建分布式系统的参考。


oldStr:


我将提供完整的设计文档内容，用户可以手动创建文件并复制内容。由于工具限制，我无法直接创建文件，但我已经准备好完整的设计文档内容。以下是核心组件化Actor框架和聊天系统的设计文档：

# 核心组件化Actor框架设计与聊天系统实现

## 1. 框架设计目标

设计一个简单、组件化的Actor框架，包含以下核心特性：

- **简洁的Actor模型**：遵循经典Actor模型，支持消息传递、封装状态和行为
- **组件化设计**：将框架拆分为独立、可复用的组件
- **轻量级实现**：最小化依赖，易于理解和扩展
- **高性能**：支持高效的消息处理和并发

## 2. 核心组件设计

### 2.1 Actor接口

```go
// Actor 定义Actor的核心接口
type Actor interface {
// Receive 处理接收到的消息
Receive(ctx Context)
}
```

### 2.2 Context接口

```go
// Context 提供Actor的执行上下文
type Context interface {
// Self 返回当前Actor的PID
Self() *PID

// Sender 返回发送当前消息的Actor的PID
Sender() *PID

// Message 返回当前处理的消息
Message() interface{}

// Send 发送消息给指定Actor
Send(pid *PID, message interface{})

// Spawn 创建新的子Actor
Spawn(props *Props) *PID

// Stop 停止指定Actor
Stop(pid *PID)
}
```

### 2.3 PID (Process ID)

```go
// PID 表示Actor的唯一标识符
type PID struct {
// Address Actor所在的地址
Address string

// ID Actor的唯一标识
ID string
}
```

### 2.4 Props

```go
// Props 包含创建Actor的配置信息
type Props struct {
// Producer 用于创建Actor实例的函数
Producer func() Actor
}
```

### 2.5 ActorSystem

```go
// ActorSystem 是Actor框架的运行时环境
type ActorSystem struct {
// Root 根上下文，用于创建顶级Actor
Root *RootContext

// ProcessRegistry 用于注册和查找Actor
ProcessRegistry *ProcessRegistry
}
```

## 3. 聊天系统设计

### 3.1 系统架构

聊天系统基于Actor框架实现，包含以下核心组件：

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  UserActor       |<--->|  RoomActor       |<--->|  RoomManager     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```

### 3.2 消息类型设计

#### 用户相关消息

```go
// UserJoin 用户加入系统
type UserJoin struct {
Username string
}

// UserLeave 用户离开系统
type UserLeave struct {
Username string
}

// UserMessage 用户接收的消息
type UserMessage struct {
Username string
Content  string
Time     time.Time
}
```

#### 房间相关消息

```go
// CreateRoom 创建房间
type CreateRoom struct {
RoomName string
}

// JoinRoom 加入房间
type JoinRoom struct {
Username string
RoomName string
UserPID  *PID
}

// LeaveRoom 离开房间
type LeaveRoom struct {
Username string
RoomName string
}

// RoomMessage 发送到房间的消息
type RoomMessage struct {
Username string
RoomName string
Content  string
Time     time.Time
}

// ListRooms 请求房间列表
type ListRooms struct {}

// RoomList 房间列表响应
type RoomList struct {
Rooms []string
}
```

### 3.3 Actor实现

#### UserActor

```go
// UserActor 处理用户相关的消息
type UserActor struct {
username string
rooms    map[string]*PID // 已加入的房间
}

func (u *UserActor) Receive(ctx Context) {
switch msg := ctx.Message().(type) {
case *UserJoin:
u.username = msg.Username
u.rooms = make(map[string]*PID)
fmt.Printf("用户 %s 加入系统\n", u.username)

case *UserMessage:
fmt.Printf("[%s] %s: %s\n", msg.Time.Format("15:04:05"), msg.Username, msg.Content)

case *UserLeave:
fmt.Printf("用户 %s 离开系统\n", u.username)
u.rooms = nil
}
}
```

#### RoomActor

```go
// RoomActor 处理房间相关的消息
type RoomActor struct {
name     string
users    map[string]*PID // 房间内的用户
messages []UserMessage   // 聊天记录
}

func (r *RoomActor) Receive(ctx Context) {
switch msg := ctx.Message().(type) {
case *JoinRoom:
if _, exists := r.users[msg.Username]; exists {
return // 用户已在房间内
}
r.users[msg.Username] = msg.UserPID
fmt.Printf("用户 %s 加入房间 %s\n", msg.Username, r.name)

// 通知房间内所有用户
for _, userPID := range r.users {
ctx.Send(userPID, &UserMessage{
Username: "系统",
Content:  fmt.Sprintf("用户 %s 加入了房间", msg.Username),
Time:     time.Now(),
})
}

case *LeaveRoom:
if _, exists := r.users[msg.Username]; !exists {
return // 用户不在房间内
}
delete(r.users, msg.Username)
fmt.Printf("用户 %s 离开房间 %s\n", msg.Username, r.name)

// 通知房间内所有用户
for _, userPID := range r.users {
ctx.Send(userPID, &UserMessage{
Username: "系统",
Content:  fmt.Sprintf("用户 %s 离开了房间", msg.Username),
Time:     time.Now(),
})
}

case *RoomMessage:
// 保存消息
userMsg := UserMessage{
Username: msg.Username,
Content:  msg.Content,
Time:     msg.Time,
}
r.messages = append(r.messages, userMsg)

// 广播消息给房间内所有用户
for _, userPID := range r.users {
ctx.Send(userPID, &userMsg)
}
}
}
```

#### RoomManager

```go
// RoomManager 管理所有房间
type RoomManager struct {
rooms map[string]*PID // 房间名称到房间PID的映射
}

func (rm *RoomManager) Receive(ctx Context) {
switch msg := ctx.Message().(type) {
case *CreateRoom:
if _, exists := rm.rooms[msg.RoomName]; exists {
return // 房间已存在
}

// 创建新房间Actor
roomProps := actor.PropsFromProducer(func() actor.Actor {
return &RoomActor{name: msg.RoomName}
})
roomPID := ctx.Spawn(roomProps)
rm.rooms[msg.RoomName] = roomPID
fmt.Printf("房间 %s 创建成功\n", msg.RoomName)

case *JoinRoom:
if roomPID, exists := rm.rooms[msg.RoomName]; exists {
// 转发加入请求到房间Actor
ctx.Forward(roomPID)
} else {
fmt.Printf("房间 %s 不存在\n", msg.RoomName)
}

case *LeaveRoom:
if roomPID, exists := rm.rooms[msg.RoomName]; exists {
// 转发离开请求到房间Actor
ctx.Forward(roomPID)
}

case *RoomMessage:
if roomPID, exists := rm.rooms[msg.RoomName]; exists {
// 转发消息到房间Actor
ctx.Forward(roomPID)
}

case *ListRooms:
rooms := make([]string, 0, len(rm.rooms))
for roomName := range rm.rooms {
rooms = append(rooms, roomName)
}
ctx.Respond(&RoomList{Rooms: rooms})
}
}
```

### 3.4 系统初始化与流程

```go
func main() {
// 创建Actor系统
system := actor.NewActorSystem()

// 创建房间管理器
roomManagerProps := actor.PropsFromProducer(func() actor.Actor {
return &RoomManager{}
})
roomManagerPID := system.Root.Spawn(roomManagerProps)

// 创建用户Alice
aliceProps := actor.PropsFromProducer(func() actor.Actor {
return &UserActor{}
})
alicePID := system.Root.Spawn(aliceProps)

// Alice加入系统
system.Root.Send(alicePID, &UserJoin{Username: "Alice"})

// 创建房间
system.Root.Send(roomManagerPID, &CreateRoom{RoomName: "General"})

// Alice加入房间
system.Root.Send(roomManagerPID, &JoinRoom{
Username: "Alice",
RoomName: "General",
UserPID:  alicePID,
})

// Alice发送消息
system.Root.Send(roomManagerPID, &RoomMessage{
Username: "Alice",
RoomName: "General",
Content:  "大家好！",
Time:     time.Now(),
})

// 等待用户输入
fmt.Println("按任意键退出...")
console.ReadLine()
}
```

## 4. 扩展性设计

框架设计考虑了扩展性，支持以下扩展方式：

1. **中间件**：可以在消息处理前后添加中间件，用于日志、监控、安全等
2. **自定义调度器**：支持替换默认的调度器实现
3. **自定义邮箱**：支持不同类型的邮箱实现，如无界邮箱、有界邮箱等
4. **远程通信**：可以扩展为支持远程Actor通信

## 5. 性能优化

1. **异步消息处理**：所有消息处理都是异步的，避免阻塞
2. **高效的邮箱实现**：使用锁-free队列提高性能
3. **合理的调度策略**：根据系统负载动态调整调度策略
4. **最小化内存分配**：减少不必要的内存分配和垃圾回收

## 6. 总结

本设计提供了一个简单、组件化的Actor框架，以及基于此框架的聊天系统实现。框架遵循经典Actor模型，支持消息传递、封装状态和行为，具有良好的扩展性和性能。聊天系统实现了用户管理、房间管理和消息传递等核心功能，可以作为基于Actor模型构建分布式系统的参考。

