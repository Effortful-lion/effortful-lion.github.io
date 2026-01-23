前面一节我们处理了单个 Group 的逻辑，其实比较简单，和我们前面在 Lab3 实现的分布式 KV 是基本类似的。

今天这一节来处理下配置变更的问题。

shardkv 需要定时从 shardctrler 这边拉取最新的配置，然后根据配置来确定哪些 shard 应该是需要进行迁移的。

上一节我们已经写了一个简单的拉取配置的后台任务，但是按照 lab 的提示，我们每次只能够拉取一个配置，并且按照顺序处理，这样做的目的主要是为了避免覆盖还未完成的配置变更任务。

```Go
// 获取当前配置
func (kv *ShardKV) fetchConfigTask() {
   for !kv.killed() {
      kv.mu.Lock()
      newConfig := kv.mck.Query(kv.currentConfig.Num + 1)
      kv.mu.Unlock()

      // 传入 raft 模块进行同步
      kv.ConfigCommand(RaftCommand{ConfigChange, newConfig}, &OpReply{})
      time.Sleep(FetchConfigInterval)
   }
}
```

拉取完毕配置之后，我们需要构造一个对应的命令，然后传到 raft 模块进行同步。

这里需要做一点小的改造，因为我们之前传入到 raft 的都是客户端的操作，这里我们需要加上配置变更的操作。并且在 apply 协程中进行反解析。

```Go
var opReply *OpReply
raftCommand := message.Command.(RaftCommand)
if raftCommand.CmdType == ClientOpeartion {
   // 取出用户的操作信息
   op := raftCommand.Data.(Op)
   if op.OpType != OpGet && kv.requestDuplicated(op.ClientId, op.SeqId) {
      opReply = kv.duplicateTable[op.ClientId].Reply
   } else {
      // 将操作应用状态机中
      shardId := key2shard(op.Key)
      opReply = kv.applyToStateMachine(op, shardId)
      if op.OpType != OpGet {
         kv.duplicateTable[op.ClientId] = LastOperationInfo{
            SeqId: op.SeqId,
            Reply: opReply,
         }
      }
   }
} else {
   opReply = kv.handleConfigChangeMessage(raftCommand)
}
```

根据最新状态的 Config 信息，我们能够判断出当前 Group 中负责哪些 shard，也能够判断出某个 shard 转移到当前 shard 中。