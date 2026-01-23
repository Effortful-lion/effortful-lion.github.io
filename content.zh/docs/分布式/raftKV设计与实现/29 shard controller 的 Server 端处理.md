上一节我们主要处理了 shardkv 中的 shardctrler 的客户端逻辑，这一节我们处理一下 shardctrler 的服务端的逻辑。

其实 Server 这边的处理和前一个部分我们实现的分布式 KV 的 Server 非常类似，逻辑基本上是差不多的。

我们依然需要维护状态机、通知 channel、去重的哈希表，在前面的概述中提到了，由于 shardctrler 是存储的一些配置信息，并不会存储用户数据，所以数据相对来说是比较少的，因此我们可以不用去实现分布式 KV 中的 snapshot 机制。

这里我们接收客户端的四种请求 Query、Join、Leave、Move，然后将其通过 raft 模块进行各个节点之间的状态同步。

然后我们在后台的 apply 线程中处理 raft 已经 commit 的数据，主要是将操作应用到状态机中。

```Go
func (sc *ShardCtrler) applyLogToStateMachine(op Op) *OpRelpy {
   var err Err
   var config Config
   switch op.OpType {
   case OpJoin:
      err = sc.stateMachine.Join(op.Servers)
   case OpLeave:
      err = sc.stateMachine.Leave(op.GIDs)
   case OpMove:
      err = sc.stateMachine.Move(op.Shard, op.GID)
   case OpQuery:
      config, err = sc.stateMachine.Query(op.Num)
   }
   return &OpRelpy{ControllerConfig: config, Err: err}
}
```