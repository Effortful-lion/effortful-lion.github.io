前面一节我们主要对 shardctrler 的服务端进行了处理，主要沿用了之前分布式 KV 的一部分代码，并且由于 shard ctrler 数据量较少，我们也不需要 snapshot 的逻辑。

当 raft 模块状态同步完成之后，节点会发送已经 commit 的日志，我们就会在后台常驻的 apply 线程中进行处理，主要是将用户的操作持久化到状态机中，这一节就来看看状态机中具体的操作逻辑是什么样的。

前面其实提到了我们有四个客户端的方法，分别是 `Query、Join、Leave、Move`，实际上状态机的处理，就是对这几种方法进行处理，将处理完成之后的配置存储起来，供外部调用，接下来就依次看看这几个方法的大致处理逻辑。

## Query

`Query` 的逻辑比较简单，是通过配置编号 num 进行查询，我们会在状态机中维护一个配置数组，num 其实就是数组的下标，所以能够直接获取到下标对应的配置。

```Go
type CtrlerStateMachine struct {
   Configs []Config
}
```

## Join

`Join` 主要是添加一个 Group 到集群中，我们需要处理添加完成之后的负载均衡问题。

```Go
// Join 加入新的复制组，需要处理加入后的负载均衡问题
func (csm *CtrlerStateMachine) Join(groups map[int][]string) Err {
}
```

前面我们已经知道了 Config 的具体内容，主要包含配置编号 Num、shard 到 gid 的映射、gid 及其对应的节点信息。

```Go
// Config A configuration -- an assignment of shards to groups.
// Please don't change this.
type Config struct {
   Num    int              // config number
   Shards [NShards]int     // shard -> gid
   Groups map[int][]string // gid -> servers[]
}
```

当新添加 Group 的时候，我们这里的参数是 Groups，表示有可能是多个 Group 加入进来。

首先我们需要遍历传递进来的 Group，将其加入到 Config 的 `Groups` 中，这里就需要注意我们前面提到过的问题，那就是加入之后，shard 应该怎么处理。

假设这样一种情况，我们有 10 个 shard 和 3 个 Group，其在 Config 中的对应关系如下：

```Go
Config.Shards 数组:

0 1 2 3 4 5 6 7 8 9 -- shard id
1 1 1 2 2 2 2 3 3 3 -- group id
```

现在有了一个新的 Group ID 为 4 加入进来，我们就需要重新处理 shard 到 Group 的关系，因为不能让新加入的 Group 处于空闲状态，而应该分担一部分 shard，让整个集群重新达到平衡。

```Go
Config.Shards 数组:

0 1 2 3 4 5 6 7 8 9            -- shard id
1 1 1 2 2 2 2 3 3 3 4          -- group id
```

我们的做法简单来说是从拥有最多 shard 的 Group 中取出一个 shard，将其分配给最少 shard 的那个 Group，如果最多和最少的 shard 的差值小于等于 1，那么说明就已经达到了平衡，否则的话就按照同样的方法一直重复移动 shard。

首先我们将 gid 到 shard 做一个简单的映射，主要是从 Config 的 shards 数组中获取：

```Go
// shard gid
//   0    1
//   1    1
//   2    2
//   3    2
//   4    1

//  gid     shard
//   1    [0, 1, 4]
//   2     [2, 3]
```

这样我们就得到了每个 gid 对应负责的 shard id。

然后进行前面所说的移动，这里是一个简单的示例：

```Go
//  gid     shard
//   1    [0, 1, 4, 8]
//   2    [2, 3, 7]
//   3    [5, 6, 9]
//   4    []

-- 第一次遍历，shard 最多的是 gid 1，最少的是新加入的 gid 4，所以移动一个到 gid 4
//  gid     shard
//   1    [1, 4, 8]
//   2    [2, 3, 7]
//   3    [5, 6, 9]
//   4    [0]

-- 第二次遍历，shard 最多的是 gid 1，最少的是 gid 4，所以移动一个到 gid 4
//  gid     shard
//   1    [4, 8]
//   2    [2, 3, 7]
//   3    [5, 6, 9]
//   4    [0, 1]

-- 第三次遍历，shard 最多的是 gid 2，最少的是 gid 1，其差值等于 1，所以结束移动，集群达到平衡
```

这样移动完成之后，需要将 gid->shard id 的映射关系，重新写入到 Config 的 shards 数组中，然后存储起来。

## Leave

`Leave`方法是将一个或多个 Group 从集群中删除，和 Join 一样，在删除掉集群中的 Group 之后，其负责的 shard 应该转移到其他的 Group 中，重新让集群达到均衡。

处理的逻辑和 Join 类似，首先我们将 gid 进行遍历，并将其从 Config 的 `Groups` 中删除，并且记录这些被删除的 gid 所对应的 shard，然后将这些 shard 分配给拥有最少 shard 的 Group。

这里举一个简单的例子。

```Go
//  gid     shard
//   1    [4, 8]
//   2    [2, 3, 7]
//   3    [5, 6, 9]
//   4    [0, 1] 

-- gid 4 离开集群之后，shard 0 和 1 就需要重新分配

-- 第一次遍历，找到拥有 shard 数最少的 gid 1 并分配
//  gid     shard
//   1    [4, 8, 0]
//   2    [2, 3, 7]
//   3    [5, 6, 9]

-- 第二次遍历，找到拥有 shard 数最少的 gid 1 并分配
//  gid     shard
//   1    [4, 8, 0, 1]
//   2    [2, 3, 7]
//   3    [5, 6, 9]
```

这样集群就重新达到了平衡状态，然后和 Join 一样，重新构造 Config 中的 shards 对应关系，并将其存储起来。

## Move

`Move` 方法的参数是一个 shard id 和 gid，表示将这个 shard 移动的指定的 gid 之上，在这里我们的处理比较简单，因为 Config 中的 shards 关系都是明确的，只需要将 shard id 的 gid 重置为传进来的新的 gid 即可。

以上部分处理完成之后，我们在状态机中的逻辑就完成了，这里可能很多人有一个疑问，那就是 Group 的 Join、Leave 只是将配置变更了，具体移动 shard 的操作应该是在哪里完成呢？

实际上这个是第二部分 shardkv 需要做的事情，shardkv 当中拿到配置之后，如果发现了不同，则需要处理 shard 在不同的 Group 之间的转移。

所以这一部分我们下一节才开始讲述，到这一节整个 shard controller 的功能便完成了，我们实现了一个高可用的分布式 KV，用于存储 shardkv 的配置信息，并且提供了几个简单的接口来处理和查询配置。