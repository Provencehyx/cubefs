# MetaNode 七层解析 · Layer 4：MetaPartition 与 Raft

> 入口文件：[metanode/raft_server.go](../../metanode/raft_server.go) · [metanode/partition.go](../../metanode/partition.go) · [metanode/partition_fsm.go](../../metanode/partition_fsm.go)
> 核心问题：**`mp.CreateInode(...)` 这一行是怎么把"创建一个 inode"这件事经过 raft 复制到三副本、最终修改内存 BTree 的？**

> ⚠️ 这是 MetaNode 最硬核的一层。请耐心读完，后面 Layer 5/6/7 全都建立在这一层之上。

---

## 1. 这一层在系统中的位置

```
   Layer 3 (manager_op.go)
        │
        │  mp.CreateInode(req, p, remoteAddr)   ← 这就是 Layer 3 出口
        ▼
┌────────────────────────────────────────────────────────┐
│              metaPartition.CreateInode                  │
│   ① 校验 storageClass + 分配 inoID                        │
│   ② 构造 *Inode 并 Marshal                               │
│   ③ mp.submit(opFSMCreateInode, val)  ← ★ 进入 raft     │
│   │      ▼                                              │
│   │  mp.raftPartition.Submit(cmd)                       │
│   │      ▼                                              │
│   │  ┌───────────────────────────────────────────┐     │
│   │  │   tiglabs/raft（多 raft 共享引擎）         │     │
│   │  │   leader 写 log → 复制到 follower → commit │     │
│   │  └─────────────┬─────────────────────────────┘     │
│   │                ▼                                    │
│   │   ★ 在所有副本上回调 mp.Apply(cmd, index)            │
│   │                ▼                                    │
│   │   switch msg.Op { case opFSMCreateInode:            │
│   │       mp.fsmCreateInode(ino) }   ◄── 真正改 BTree   │
│   ④ 把 fsm 返回的 status 包成 packet 回客户端              │
└────────────────────────────────────────────────────────┘
                    │
                    ▼
              Layer 5 (inode 子系统)
              Layer 6 (dentry 子系统)
              Layer 7 (持久化)
```

**记住一句话**：**所有写都走 `submit → raft → Apply → fsmXxx`，所有读直接走内存 BTree**。这就是 MetaNode 的核心心智模型。

---

## 2. 三个核心抽象：MetaNode / RaftStore / MetaPartition

CubeFS 的 raft 走的是 **multi-raft（多组共享引擎）** 模式。三个层级：

```
MetaNode 进程
   ├── 1 个 raftStore （共享 raft 引擎，跑在固定的 hb/replica 端口上）
   │      │
   │      ├── raft group #101 ──► metaPartition #101
   │      ├── raft group #102 ──► metaPartition #102
   │      └── raft group #N   ──► metaPartition #N
   │
   └── metadataManager
          └── partitions map[uint64]MetaPartition  ◄── Layer 3 见过
```

**为什么这么设计**：

- 一个 MetaNode 可能负责几十~上百个 mp，每个 mp 是独立的 raft group
- 如果每个 mp 起一对 hb/replica TCP 端口 → 端口爆炸 + 长连接爆炸
- 因此**所有 mp 复用同一个 raft 引擎**：raftStore 用一对端口，靠内部的 groupID（= PartitionId）路由
- 这就是 [raft_server.go:29](../../metanode/raft_server.go#L29) 在 Layer 1 启动时只起 **一个** raftStore 的原因

### `startRaftServer` 干了什么

[raft_server.go:29](../../metanode/raft_server.go#L29)

```go
func (m *MetaNode) startRaftServer(cfg *config.Config) (err error) {
    os.MkdirAll(m.raftDir, 0o755)                    // ① 准备 raftDir
    raftConf := &raftstore.Config{
        NodeID:            m.nodeId,                  // ② 由 register 拿到
        RaftPath:          m.raftDir,
        IPAddr:            m.localAddr,
        HeartbeatPort:     heartbeatPort,             // ③ 固定的两个端口
        ReplicaPort:       replicaPort,
        TickInterval:      m.tickInterval,
        RecvBufSize:       m.raftRecvBufSize,
        NumOfLogsToRetain: m.raftRetainLogs,
    }
    m.raftStore, err = raftstore.NewRaftStore(raftConf, cfg)
}
```

注意几个细节：
- **`NodeID` 是节点级唯一**（不是 mp 级），一个 MetaNode 一个值
- **`NumOfLogsToRetain`** 决定每个 raft group 在做完 snapshot 后保留多少条 log（用于 follower 追赶）。Layer 1 [parseConfig](../../metanode/metanode.go#L286) 解析的 `cfgRetainLogs` 走到这里
- **`raftPath`** 是所有 mp 的 raft log 共享根目录，每个 mp 在下面有自己的子目录
- 这是个**非常薄的封装**——真正的 raft 实现在 `depends/tiglabs/raft`，CubeFS 只是包了一层 facade

---

## 3. `metaPartition` 数据结构

[partition.go:484](../../metanode/partition.go#L484)

```go
type metaPartition struct {
    config        *MetaPartitionConfig    // 配置（含 PartitionId/Start/End/Peers/...）
    size          uint64
    applyID       uint64                   // ★ 已 apply 的 raft 日志 index
    storedApplyId uint64                   // ★ 已落盘到 snapshot 的 applyID

    dentryTree    *BTree                   // ★ Layer 6：dentry 内存树
    inodeTree     *BTree                   // ★ Layer 5：inode 内存树
    extendTree    *BTree                   // xattr
    multipartTree *BTree                   // S3 multipart

    txProcessor   *TransactionProcessor    // 分布式事务
    raftPartition raftstore.Partition      // ★ 这个 mp 对应的 raft group 句柄

    stopC         chan bool
    storeChan     chan *storeMsg           // ★ Layer 7：触发持久化的 channel
    state         uint32                   // standby/start/running/shutdown
    delInodeFp    *os.File                 // 已删除 inode 的日志文件
    freeList      *freeList                // 待 GC 的 inode

    extDelCh      chan []proto.ExtentKey   // 待删除 extent 的批量队列
    vol           *Vol                     // 所属卷的视图（来自 master）
    manager       *metadataManager         // 反向引用

    uniqChecker   *uniqChecker             // 写幂等去重
    verSeq        uint64                   // 多版本快照序号
    multiVersionList *proto.VolVersionInfoList
    nonIdempotent sync.Mutex               // ★ Apply 串行化锁
    ...
}
```

**4 个最关键字段必须记住**：

| 字段 | 作用 | 谁会改它 |
|---|---|---|
| `applyID` | 已 apply 到 fsm 的最大 raft index | `Apply` / `ApplySnapshot` 末尾的 `uploadApplyID` |
| `inodeTree` / `dentryTree` | **整个 mp 的核心内存状态** | 只在 `fsmXxx` 函数里改 |
| `raftPartition` | mp 对应的 raft 句柄 | `startRaft` 创建，`stopRaft` 关闭 |
| `nonIdempotent` | Apply 串行锁 | 写到 BTree 时持有 |

> 💡 **这就是 MetaNode 的全部秘密**：内存里维护几棵 BTree，所有写改动通过 raft 同步到所有副本，每个副本独立调用 `fsmXxx` 在自己的 BTree 上做相同的修改 → 三副本最终一致。

---

## 4. mp 生命周期：从 New 到 Running

```
NewMetaPartition()         ── 仅构造对象（4 棵空 BTree）
        │
        ▼
metaPartition.Start(isCreate)
        │
        ▼
metaPartition.onStart(isCreate)
   1. versionInit()             ── 拉多版本快照列表（如启用）
   2. mp.load(isCreate)         ── ★ Layer 7：从磁盘 snapshot + raft log 恢复
   3. startScheduleTask()       ── 后台定期 store 任务
   4. forceUpdateVolumeView()   ── 从 master 拉 vol view（重试 200 次）
   5. blob client 初始化（混合云）
   6. startFreeList()           ── 启动 GC 线程
   7. startCheckerEvict()       ── 启动 inode 驱逐检查
   8. ★ startRaft(isCreate)    ── 把 mp 自己作为 fsm 注册到 raftStore
   9. updateSize()              ── 后台周期统计 size
```

### `startRaft`：把 mp 注入 raftStore

[partition.go:904](../../metanode/partition.go#L904)

```go
func (mp *metaPartition) startRaft(isCreate bool) (err error) {
    heartbeatPort, replicaPort, _ := mp.getRaftPort()
    var peers []raftstore.PeerAddress
    for _, peer := range mp.config.Peers {
        peers = append(peers, raftstore.PeerAddress{
            Peer:          raftproto.Peer{ID: peer.ID},
            Address:       strings.Split(peer.Addr, ":")[0],
            HeartbeatPort: heartbeatPort,
            ReplicaPort:   replicaPort,
        })
    }

    pc := &raftstore.PartitionConfig{
        ID:       mp.config.PartitionId,   // ★ raft group ID = mp.PartitionId
        Applied:  mp.applyID,              // ★ 从恢复后的 applyID 开始
        Peers:    peers,
        SM:       mp,                      // ★★ 把 mp 自己作为 state machine 注册
        IsCreate: isCreate,
    }
    mp.raftPartition, err = mp.config.RaftStore.CreatePartition(pc)
    ...
}
```

**3 个最关键的事实**：

1. **`SM: mp`**：mp 自己实现了 raft 的 state machine 接口，被注册进去。所以 raft 在某个 index 提交后会**直接调 `mp.Apply(cmd, index)`**——不需要中间转发。
2. **`Applied: mp.applyID`**：从 `mp.load` 恢复出来的 applyID 开始，告诉 raft "我的状态机已经吃到这一条了，从这之后给我喂"。**这是 snapshot + log 协调一致的关键**。
3. **`ID = PartitionId`**：raft group ID 就是 PartitionId。raftStore 内部用这个字段路由消息。

> 💡 注意 `raftPartitionCanUsingDifferentPort` 这个开关——支持不同 mp 用不同的 raft 端口，是后期为兼容混合部署加的扩展。新人看到这种字段不要慌，**默认走 false 分支即可**。

---

## 5. fsm 接口：mp 实现了什么

`mp` 作为 raft state machine 必须实现 5 个回调（接口定义在 `raftstore.Partition`），都在 [partition_fsm.go](../../metanode/partition_fsm.go) 里：

| 接口 | 何时被调 | 实现位置 | 作用 |
|---|---|---|---|
| **`Apply(cmd, index)`** | leader 提案 commit 后，**每个副本**都调一次 | [partition_fsm.go:38](../../metanode/partition_fsm.go#L38) | **改 BTree** ★ |
| **`ApplyMemberChange(cc, index)`** | 配置变更（add/remove peer）commit 后 | [partition_fsm.go:631](../../metanode/partition_fsm.go#L631) | 改 mp.config.Peers 并 persist |
| **`Snapshot()`** | leader 给 follower 发快照时 | [partition_fsm.go:676](../../metanode/partition_fsm.go#L676) | 返回 `newMetaItemIterator(mp)` 把 BTree 序列化 |
| **`ApplySnapshot(peers, iter)`** | follower 收到 leader 快照时 | [partition_fsm.go:681](../../metanode/partition_fsm.go#L681) | 把整个 BTree 整体替换 |
| **`HandleLeaderChange(leader)`** | raft 选主切换 | [partition_fsm.go:941](../../metanode/partition_fsm.go#L941) | 启停 store tick、初始化根 inode |
| **`HandleFatalEvent(err)`** | raft 引擎报致命错误 | [partition_fsm.go:933](../../metanode/partition_fsm.go#L933) | 直接 panic |

**这 6 个函数 = mp 与 raft 引擎的全部接口**。Layer 7 我们会看 `Snapshot/ApplySnapshot` 的细节，本层先把 `Apply` 和 `submit` 串通。

---

## 6. ★ 一次写请求的完整生命周期（必须刻进脑子）

以 `CreateInode` 为例，串完从 packet 到 BTree：

### Step ① 客户端 → leader 节点 → 分发到 mp
（Layer 2 + Layer 3 已经讲完，跳过）

### Step ② mp.CreateInode：构造 inode 并 submit

[partition_op_inode.go:159](../../metanode/partition_op_inode.go#L159)

```go
func (mp *metaPartition) CreateInode(req *CreateInoReq, p *Packet, remoteAddr string) (err error) {
    // a. 校验 storageClass、分配新 inoID
    inoID, err := mp.nextInodeID()
    ino := NewInode(inoID, req.Mode)
    ino.Uid = req.Uid
    ino.Gid = req.Gid
    ino.setVer(mp.verSeq)
    ino.LinkTarget = req.Target
    ino.StorageClass = requiredStorageClass

    // b. 序列化为字节流
    val, _ := ino.Marshal()

    // c. ★ 提交 raft 提案
    resp, err := mp.submit(opFSMCreateInode, val)
    if err != nil {
        p.PacketErrorWithBody(proto.OpAgain, []byte(err.Error()))
        return
    }

    // d. 提案 commit 后，resp 是 fsmCreateInode 的返回值
    if resp.(uint8) == proto.OpOk { ...构造响应... }
    p.PacketErrorWithBody(status, reply)
}
```

**两个关键细节**：

- `mp.nextInodeID()` 是**本地分配**的，每个 mp 有一段 inode ID 区间（`config.Start` ~ `config.End`），用 `Cursor` 递增。**不需要走 raft，因为 leader 独占分配**。
- `submit` 是**同步阻塞**的——调用方会一直等 raft 提案 commit 后才返回。

### Step ③ submit：包装成 raft 提案

[partition_fsm.go:995](../../metanode/partition_fsm.go#L995)

```go
func (mp *metaPartition) submit(op uint32, data []byte) (resp interface{}, err error) {
    snap := NewMetaItem(0, nil, nil)
    snap.Op = op                       // 比如 opFSMCreateInode
    snap.V = data                      // 序列化后的 inode
    cmd, _ := snap.MarshalJson()       // 整个 MetaItem 转 JSON

    resp, err = mp.raftPartition.Submit(cmd)   // ★ 调 raft 引擎
    return
}
```

**`MetaItem` 是 raft 命令的统一信封**。所有 fsm op 都先包成 `{Op, K, V}` 再走 raft：

```go
type MetaItem struct {
    Op uint32   // 命令类型，比如 opFSMCreateInode/opFSMCreateDentry
    K  []byte
    V  []byte   // 业务负载，比如序列化后的 *Inode
}
```

→ **这就是为什么 Apply 里第一行要 `msg.UnmarshalJson(command)`**：把 raft 喂回来的字节流先解成 `MetaItem`，再按 Op 分发。

### Step ④ raft 引擎复制日志（黑盒）

`mp.raftPartition.Submit(cmd)` 在 leader 上：

1. 把 cmd 写本地 raft log
2. 通过 replicaPort 发到所有 follower
3. 等多数派 ack
4. 标记为 committed
5. **逐个 commit 的 entry 调 `mp.Apply(entry.Data, entry.Index)`**

`Submit` 是同步的——直到 commit（在 leader 上意味着 Apply 也已经被调过）才返回。

### Step ⑤ Apply：在所有副本上分发到 fsmXxx

[partition_fsm.go:38](../../metanode/partition_fsm.go#L38)

```go
func (mp *metaPartition) Apply(command []byte, index uint64) (resp interface{}, err error) {
    msg := &MetaItem{}
    defer func() {
        // 异常 panic / 成功 uploadApplyID
        if r := recover(); r != nil { panic(...) }
        if err == nil { mp.uploadApplyID(index) }   // ★ 更新 applyID
    }()

    msg.UnmarshalJson(command)                   // 解信封

    mp.nonIdempotent.Lock()                      // ★ Apply 全程串行
    defer mp.nonIdempotent.Unlock()

    switch msg.Op {
    case opFSMCreateInode:
        ino := NewInode(0, 0)
        ino.Unmarshal(msg.V)
        if mp.config.Cursor < ino.Inode {
            mp.config.Cursor = ino.Inode         // ★ 同步 Cursor
        }
        resp = mp.fsmCreateInode(ino)            // ◄── 真正改 inodeTree
    case opFSMUnlinkInode:    ...
    case opFSMCreateDentry:   ...
    case opFSMDeleteDentry:   ...
    case opFSMSetXAttr:       ...
    case opFSMTxCommit:       ...
    ... 几十种 op
    }
}
```

**4 个绝对要记住的设计点**：

1. **`nonIdempotent` 锁让 Apply 全程串行**——这是**整个 mp 的"全局写锁"**。所有读路径不持有这把锁（直接读 BTree 自带的 RLock），所以读 >> 写并发度极高，但写是单线程的。
2. **`uploadApplyID(index)` 必须在 Apply 成功后才调**：确保 applyID 单调递增、和 BTree 状态对齐。这是 **snapshot/log 一致性的命脉**。
3. **Apply 在每个副本上独立执行**：leader、follower 都会跑同一段代码、同一份 inode 数据 → 三副本 BTree 状态一致。这就是 raft 的 SMR（state machine replication）模型。
4. **Apply panic 会真的 panic 整个 MetaNode 进程**——见 [partition_fsm.go:45](../../metanode/partition_fsm.go#L45)。设计意图是"宁可崩，不能让副本状态分叉"。

### Step ⑥ fsmCreateInode：真正改 BTree

[partition_fsmop_inode.go:78](../../metanode/partition_fsmop_inode.go#L78)（细节留给 Layer 5）

简化版逻辑：
```go
func (mp *metaPartition) fsmCreateInode(ino *Inode) uint8 {
    if mp.inodeTree.Has(ino) { return proto.OpExistErr }
    mp.inodeTree.ReplaceOrInsert(ino, true)
    return proto.OpOk
}
```

**关键**：fsmXxx 函数**永远不能失败（不返 err）**，只返回业务 status。原因是 raft Apply 的语义是"已经 committed 的命令必须能落地"——如果失败 raft 会一直重试，副本就 stuck 了。

### Step ⑦ 返回客户端
回到 `CreateInode` 的尾部，把 `fsmCreateInode` 的返回值 `OpOk` 包成响应 → Layer 3 `respondToClientWithVer` → Layer 2 写回 conn。

---

## 7. `HandleLeaderChange`：选主切换的副作用

[partition_fsm.go:941](../../metanode/partition_fsm.go#L941) 是个**非常关键的钩子**，列三个核心动作：

```go
func (mp *metaPartition) HandleLeaderChange(leader uint64) {
    if mp.config.NodeId == leader {
        // ① 自检：能不能本地连上自己的 listen 端口
        conn, err := net.DialTimeout("tcp", localIp+":"+serverPort, time.Second)
        if err != nil {
            // 本地端口都连不上 → 主动让贤
            go mp.raftPartition.TryToLeader(mp.config.PartitionId)
            return
        }
        conn.Close()

        // ② 启动周期性 store tick（开始定期落 snapshot）
        mp.storeChan <- &storeMsg{command: startStoreTick}

        // ③ 如果是新 mp 的第一次选主，初始化根 inode
        if mp.config.Start == 0 && mp.config.Cursor == 0 {
            id, _ := mp.nextInodeID()
            ino := NewInode(id, proto.Mode(os.ModePerm|os.ModeDir))
            go mp.initInode(ino)
        }
    } else {
        // 变成 follower → 停止 store tick
        mp.storeChan <- &storeMsg{command: stopStoreTick}
    }
}
```

**3 个深含义**：

1. **`只有 leader 才周期性 store snapshot`**：follower 跟着 leader 的 log 走就行，不需要自己存 snapshot。这显著降低磁盘压力。
2. **本地端口自检 + 主动让贤**：极端情况下（端口被占、网络异常）leader 没法对外服务，干脆把 leader 让出去，避免黑洞。
3. **根 inode 在第一次成为 leader 时才创建**：因为根 inode 创建本身也是写操作，必须有 leader 才能 raft 提交。这就是为什么新 mp 的 inode 树**第一次有数据是在第一次选主之后**。

---

## 8. `submit` 失败 vs 成功的语义边界

这是新人很容易踩的坑：

| `submit` 返回情况 | 含义 | 调用方应做什么 |
|---|---|---|
| `err == nil, resp == OpOk` | raft commit 成功 + fsm 应用成功 | 正常返回客户端 |
| `err == nil, resp == OpExistErr` | raft commit 成功 + fsm 业务拒绝 | 把 status 透传给客户端 |
| `err != nil` | **raft 层失败**（比如不是 leader、channel 满、超时） | 回 `OpAgain`，让客户端重试 |

[partition_op_inode.go:206](../../metanode/partition_op_inode.go#L206)：
```go
resp, err = mp.submit(opFSMCreateInode, val)
if err != nil {
    p.PacketErrorWithBody(proto.OpAgain, ...)   // ← 注意是 OpAgain 不是 OpErr
    return err
}
```

**`OpAgain` 是给客户端的"快重试"信号**——客户端会立刻重连/换 leader 重试，不算业务错误。**所有 mp.submit 调用都是这种范式，下一层 inode/dentry 写函数照搬就行**。

---

## 9. 本层关键问题自测

读完 Layer 4 应该能回答：

1. ✅ multi-raft 是什么意思？为什么所有 mp 共享一对 hb/replica 端口？
2. ✅ `raftStore` 和 `mp.raftPartition` 是什么关系？谁包含谁？
3. ✅ mp 启动时为什么 raft 要排在 `mp.load` 之后？反过来会怎样？
4. ✅ Apply 函数为什么要持有 `nonIdempotent` 锁？读路径要不要持有？
5. ✅ `applyID` 和 `storedApplyId` 的区别？哪个更新得更快？
6. ✅ `MetaItem` 是什么？为什么 raft 命令要包一层信封？
7. ✅ leader 上的 `submit` 调用什么时候返回？同步还是异步？
8. ✅ `fsmCreateInode` 为什么不返回 error？
9. ✅ 选主切换时 leader 端做了哪 3 件事？为什么 follower 要停 store tick？
10. ✅ `submit` 返回 `err != nil` 时为什么回 `OpAgain` 而不是 `OpErr`？

---

## 10. 给后续 Layer 的钩子

- **Layer 5 (Inode)**：`fsmCreateInode` / `fsmUnlinkInode` / `fsmExtentTruncate` 这些 `fsmXxx` 函数到底怎么改 inodeTree 的？inode 结构体有哪些字段？分配 inoID 的 cursor 怎么不冲突？
- **Layer 6 (Dentry)**：`fsmCreateDentry` 怎么处理 `(ParentId, Name)` 索引？rename 走的是分布式事务还是单 mp？
- **Layer 7 (持久化)**：`mp.load` 具体怎么从 `partition_<id>/snapshot/` 目录恢复 4 棵 BTree？raft log replay 在哪一步？`startScheduleTask`/`storeChan` 是怎么周期性把 BTree dump 成新 snapshot 的？`Snapshot()` / `ApplySnapshot()` 的字节流格式是什么？
