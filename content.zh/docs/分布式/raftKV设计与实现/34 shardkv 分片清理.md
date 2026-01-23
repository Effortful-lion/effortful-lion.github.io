前面一节我们主要处理了 shardkv 分片迁移的主要流程，配置变更时更改 shard 的状态，并且启动一个后台线程，定期获取 shard 的状态，执行实际的 shard 迁移。

Shard 有四种状态：

- Normal
- MoveIn
- MoveOut
- GC

在前面的实现中，如果一个 shard 已经从 Group 迁移出去了，这个 shard 还会在这个 Group 中存在，并且数据也会继续保留。

但实际上，因为 shard 已经完全迁移到了另一个 Group 中，所以这个 shard 在原 Group 中已经可以不用继续保留了，我们可以将其删除掉。

在 lab4 的 challenge 1 中，要求我们及时清理 Group 中已经无效的 shard，这样能够及时释放空间。

在上一节 shard 迁移的流程中，如果一个 shard 已经完成了迁移，我们会将其置为 GC 状态，所以我们可以启动一个后台线程，定时获取需要执行 GC 的 shard。

拿到这些 shard 之后，我们需要做两件事情，一是给旧的 Group 发送消息，删除对应的 shard；二是给当前 Group 的 shard 发送消息，将其状态从 GC 更改为 Normal。

以下是一个 Shard 清理的大致流程示例：

```Go
G1 1     3      5
              MoveOut
G2 2     6
G3 4     7      5
                GC

后台线程拉取所有状态为 GC 的 shard

发送 RPC 消息：

之前的 G1 收到消息：将 shard 删除
现在的 G3 收到消息：将 shard 5 的状态设置为 Normal，即变更为正常状态
```