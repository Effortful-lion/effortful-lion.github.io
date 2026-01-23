在 shardkv 的分片迁移和清理大致完成之后，我们还需要修改几个地方，才能够将测试 run 起来。

主要有以下改动：

1. `matchGroup` 方法，需要判断 shard 的状态，如果是 GC 或者 Normal 状态，均可以继续提供服务
2. `StartServer` 方法中，需要注册 labgob 相关的结构体。
3. `makeSnapshot` 和 `restoreFromSnapshot` 中， config 信息也需要进行持久化，并且需要初始化 shard 的状态
4. fetchConfigTask 中，需要加上一个判断，如果任何一个 shard 的状态是非 Normal 的，则说明前一个 shard 迁移的流程还在进行中，我们就跳过拉取新的配置，避免覆盖之前的任务
5. Apply 的时候，客户端操作也需要判断 Group 是否匹配