前面一小节我们主要学习了基于 multi raft 的 shardkv 的大致架构和代码框架，从这一节开始就开始具体的实现编码了。

我们首先来处理一下 shard controller，shardctrler 也是一个分布式 KV 服务，这和我们前一个部分实现的分布式 KV 类似，只是其存储的是 shardkv 的一些配置信息。

Shardctrler 存储的配置信息，实际上是多个带编号的配置组合而成的数组，每个配置都有一个唯一的编号（数组下标），配置中存储的是 shard id 到 Group id 的映射关系，以及 Group id 对应的 KVServer。

每次只要有了新的配置产生，就会向数组中新增一个元素。KV 客户端或者服务端可以向 shardctrler 获取最新的或者旧的配置信息。

```Go
type ShardCtrler struct {
   mu      sync.Mutex
   me      int
   rf      *raft.Raft
   applyCh chan raft.ApplyMsg

   // Your data here.

   configs []Config // indexed by config num
}

// A configuration -- an assignment of shards to groups.
// Please don't change this.
type Config struct {
   Num    int              // config number
   Shards [NShards]int     // shard -> gid
   Groups map[int][]string // gid -> servers[]
}
```

在 shardctrler 的客户端中，提供的方法主要是对配置进行获取，以及配置变更的处理。目前有四个方法：Query、Leave、Join、Move。

`Join` 方法是添加新的 Group，它的参数是一个 map，存储了 Replica Group 的唯一标识 GID 到服务节点名字列表的映射关系。

```Go
func (ck *Clerk) Join(servers map[int][]string) {
   args := &JoinArgs{}
   // Your code here.
   args.Servers = servers

   for {
      // try each known server.
      for _, srv := range ck.servers {
         var reply JoinReply
         ok := srv.Call("ShardCtrler.Join", args, &reply)
         if ok && reply.WrongLeader == false {
            return
         }
      }
      time.Sleep(100 * time.Millisecond)
   }
}
```

`Leave` 方法的参数是一组集群中的 Group ID，表示这些 Group 退出了分布式集群。

```Go
func (ck *Clerk) Leave(gids []int) {
   args := &LeaveArgs{}
   // Your code here.
   args.GIDs = gids

   for {
      // try each known server.
      for _, srv := range ck.servers {
         var reply LeaveReply
         ok := srv.Call("ShardCtrler.Leave", args, &reply)
         if ok && reply.WrongLeader == false {
            return
         }
      }
      time.Sleep(100 * time.Millisecond)
   }
}
```

`Move` 方法的参数是一个 shard 编号和一个 Group ID。主要是用于将一个 shard 移动到指定的 Group 中。

```Go
func (ck *Clerk) Move(shard int, gid int) {
   args := &MoveArgs{}
   // Your code here.
   args.Shard = shard
   args.GID = gid

   for {
      // try each known server.
      for _, srv := range ck.servers {
         var reply MoveReply
         ok := srv.Call("ShardCtrler.Move", args, &reply)
         if ok && reply.WrongLeader == false {
            return
         }
      }
      time.Sleep(100 * time.Millisecond)
   }
}
```

`Query` 方法的参数是一个配置编号，shardctrler 依赖于这个带编号的配置，如果编号是 -1，或者大于已知的最大的编号，那么应该返回最近的一个配置。

```Go
func (ck *Clerk) Query(num int) Config {
   args := &QueryArgs{}
   // Your code here.
   args.Num = num
   for {
      // try each known server.
      for _, srv := range ck.servers {
         var reply QueryReply
         ok := srv.Call("ShardCtrler.Query", args, &reply)
         if ok && reply.WrongLeader == false {
            return reply.Config
         }
      }
      time.Sleep(100 * time.Millisecond)
   }
}
```

熟悉了这几个方法之后，我们应该怎么处理呢？

实际上和前面的分布式 KV 部分的逻辑比较类似，我们在向客户端发送请求的时候，一是需要注意如果发生了一些错误，例如得到了 ErrWrongLeader 或 ErrTimeout 错误，说明当前节点并不是 Leader 或者发生了其他的错误，我们就需要选择另一个节点重试请求。

二是我们仍然需要像之前一样处理重复请求，保证线性一致性，处理的方法和之前一样，给每个客户端的请求都赋予一个维护的标识符，然后在 server 端进行去重。