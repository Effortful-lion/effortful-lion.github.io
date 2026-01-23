前面一节主要了解了我们基于 raft 实现的分布式 KV 的大致架构和代码框架，从这一节开始就需要开始具体的代码逻辑了，我们首先需要实现的是客户端的逻辑。

上一节中提到了客户端的结构体 Clerk：

```Rust
type Clerk struct {
   servers []*labrpc.ClientEnd
   // You will have to modify this struct.
}
```

可以看到其中维护了 servers 列表，表示的是后端分布式 KV 服务的所有节点信息，我们可以通过这个信息去向指定的节点发送数据读写的请求。

前面提到了三种类型的请求，主要是：

- Get
- Put
- Append

每个 kv 服务的节点都是一个 raft 的 peer，客户端发送请求到 kv 服务的 Leader 节点，然后 Leader 节点会存储请求日志在本地，然后将日志通过 raft 发送给其他的节点进行状态同步。所以 raft 日志其实存储的是一连串客户端请求，然后 server 节点会按照顺序执行请求，并将结果存储到状态机中。

这里 Clerk 发送请求的时候，由于事先并不知道哪个节点是 Leader，所以只能轮询重试，直到得到了正确的响应。然后我们可以保存一下 Leader 节点的 id，下一次发送请求的时候，就直接从这个节点开始发起请求，省去了轮询寻找 Leader 节点的开销。

Get 方法的处理大致如下：

```Go
func (ck *Clerk) Get(key string) string {
   args := GetArgs{Key: key}
   for {
      var reply GetReply
      ok := ck.servers[ck.leaderId].Call("KVServer.Get", &args, &reply)
      if !ok || reply.Err == ErrWrongLeader || reply.Err == ErrTimeout {
         // 节点id加一，继续重试
         ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
         continue
      }
      // 请求成功，返回 value
      return reply.Value
   }
}
```

Put 和 Append 的逻辑由于比较类似，所以将其作为一个 RPC 请求，只是加了一个名为 Op 的参数加以区分。

```Go
func (ck *Clerk) PutAppend(key string, value string, op string) {
   args := PutAppendArgs{
      Key:   key,
      Value: value,
      Op:    op,
   }

   for {
      var reply PutAppendReply
      ok := ck.servers[ck.leaderId].Call("KVServer.PutAppend", &args, &reply)
      if !ok || reply.Err == ErrWrongLeader || reply.Err == ErrTimeout {
         // 节点id加一，继续重试
         ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
         continue
      }
      // 请求成功
      return
   }
}
```

**参考资料**

https://pdos.csail.mit.edu/6.824/labs/lab-kvraft.html