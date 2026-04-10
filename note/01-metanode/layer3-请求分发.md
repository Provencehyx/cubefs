# MetaNode 七层解析 · Layer 3：请求分发

> 入口文件：[metanode/manager.go](../../metanode/manager.go) · [metanode/manager_op.go](../../metanode/manager_op.go)
> 核心问题：**一个 `*Packet` 是怎么根据 `Opcode` 找到对应的 handler、再根据 `PartitionID` 找到本地 MetaPartition 的？**

---

## 1. 这一层在系统中的位置

```
       Layer 2 (server.go)
              │
              │  m.handlePacket(conn, p, remoteAddr)
              ▼
       ┌────────────────────────────────────────────┐
       │  metadataManager.HandleMetadataOperation   │   ← Layer 3 入口
       │  ┌──────────────────────────────────────┐  │
       │  │ 1. 打日志 / 起 metric                  │  │
       │  │ 2. switch p.Opcode → opXxx(...)       │  │
       │  └──────────────────┬───────────────────┘  │
       │                     ▼                       │
       │       opXxx(conn, p, remoteAddr)           │   ← 标准 handler 模板
       │       ┌────────────────────────────────┐   │
       │       │ a. json.Unmarshal(p.Data,&req)  │   │
       │       │ b. mp, _ := getPartition(id)    │   │  ← 路由到具体 mp
       │       │ c. serveProxy(conn, mp, p)      │   │  ← 非 leader 转发给 leader
       │       │ d. checkMultiVersionStatus(mp)  │   │
       │       │ e. mp.CreateInode(req, p, ...)  │   │  ← Layer 4 边界
       │       │ f. respondToClientWithVer(...)  │   │
       │       └────────────────────────────────┘   │
       └────────────────────────────────────────────┘
                           │
                           ▼
                  Layer 4 (partition.go / partition_fsm.go)
```

**记住边界**：
- **Layer 3 入口** = `HandleMetadataOperation` 的第一行
- **Layer 3 出口** = `mp.CreateInode(...)`（或同类的 mp 方法）的那一行——再往里就是 Layer 4 的事了

---

## 2. `metadataManager` 数据结构

[manager.go:94](../../metanode/manager.go#L94)

```go
type metadataManager struct {
    nodeId       uint64
    rootDir      string
    raftStore    raftstore.RaftStore
    connPool     *util.ConnectPool          // 给 serveProxy 转发用的连接池

    mu           sync.RWMutex
    partitions   map[uint64]MetaPartition   // ★ Key=PartitionID, Val=mp 实例
    metaNode     *MetaNode

    state        uint32                     // standby/start/running/shutdown
    stopC        chan struct{}

    volUpdating  *sync.Map                  // 卷级多版本状态
    verUpdateChan chan string

    enableGcTimer    bool
    gcRecyclePercent float64
    gcTimer          *util.RecycleTimer

    limitFactor  map[uint32]*rate.Limiter   // 各类操作的 QPS 限速
    cpuUtil      atomicutil.Float64
}
```

**3 个最关键的字段**：

| 字段 | 作用 | 后面用到的地方 |
|---|---|---|
| `partitions` | 本节点持有的所有 MetaPartition | **路由的核心**——所有 op 都靠它定位目标 mp |
| `connPool` | TCP 连接池 | `serveProxy` 把请求转发给 leader 节点 |
| `raftStore` | 共享的 raft 引擎 | mp 启动时把自己作为 fsm 注册进去（Layer 4）|

> 💡 注意 `partitions` 是 `map[uint64]MetaPartition` 而不是 `map[uint64]*metaPartition`——`MetaPartition` 是接口（[manager.go:62](../../metanode/manager.go#L62) 同文件能看到）。**面向接口编程**让 mp 的实现可以被 mock，单测里大量用到。

---

## 3. `HandleMetadataOperation`：超大 switch

[manager.go:228](../../metanode/manager.go#L228)。整个函数分三段。

### 段 1：日志 + 监控

```go
start := time.Now()
if p.AdminOp() {
    log.LogWarnf("HandleMetadataOperation input ...")   // ★ admin 级强制 warn 打印
} else if log.EnableInfo() {
    log.LogInfof(...)                                    // 普通 op 看 info 开关
}

metric := exporter.NewTPCnt(p.GetOpMsg())                // 按 op 名建 metric
labels := m.getPacketLabels(p)
defer func() {
    metric.SetWithLabels(err, labels)                    // 上报 metric
    // 打 out 日志：admin 强制 info，普通看开关
}()
```

**3 个值得记的设计点**：

1. **AdminOp 强制走 warn 级别**——[packet.go:124](../../metanode/packet.go#L124) 列了哪些是 admin op：
   ```go
   func (p *Packet) AdminOp() bool {
       return p.Opcode == proto.OpAddMetaPartitionRaftMember ||
              p.Opcode == proto.OpRemoveMetaPartitionRaftMember ||
              p.Opcode == proto.OpCreateMetaPartition ||
              p.Opcode == proto.OpMetaPartitionTryToLeader ||
              p.Opcode == proto.OpDeleteMetaPartition
   }
   ```
   **这些 op 改的是集群拓扑，强制全量留痕**——出问题时排查必须要有。
2. **metric label 在 `getPacketLabels` 里按 op 区分**：[manager.go:185](../../metanode/manager.go#L185)。心跳和 CreateMP **没有** PartitionID（因为还没建出来），所以 label 留空；其它 op 会查 `getPartition` 拿 vol 名当 label。**这里是个隐式约定：心跳 op 不能查 mp，否则会打 warn 日志噪音**。
3. **`defer` 里做 metric + out 日志**：保证不管成功失败都有"出口"记录，便于按 reqID 串起来。

### 段 2：Opcode 大 switch

[manager.go:254-426](../../metanode/manager.go#L254-L426) 是一个 **170+ 行的大 switch**。这里不抄全表，按业务领域分组列出：

| 分组 | 代表 Op | 对应 handler |
|---|---|---|
| **Inode 管理** | `OpMetaCreateInode` / `LinkInode` / `UnlinkInode` / `InodeGet` / `EvictInode` / `Setattr` | `opCreateInode`、`opMetaLinkInode` ... |
| **Dentry 管理** | `OpMetaCreateDentry` / `DeleteDentry` / `UpdateDentry` / `Lookup` | `opCreateDentry`、`opMetaLookup` ... |
| **目录读取** | `OpMetaReadDir` / `ReadDirOnly` / `ReadDirLimit` | `opReadDir*` |
| **Extent 管理** | `OpMetaExtentsAdd` / `ExtentsList` / `ExtentsDel` / `OpMetaTruncate` | `opMetaExtents*` |
| **XAttr** | `OpMetaSetXAttr` / `GetXAttr` / `RemoveXAttr` / `ListXAttr` | `opMetaXAttr*` |
| **MultiPart**（S3 分片上传）| `OpCreateMultipart` / `ListMultiparts` / `AddMultipartPart` | `opCreate/Append/...Multipart` |
| **事务**（分布式 rename 等）| `OpMetaTxCreateInode` / `OpTxCommit` / `OpTxRollback` | `opTx*` |
| **配额** | `OpMetaBatchSetInodeQuota` / `OpQuotaCreateInode` | `opMeta*Quota`、`opQuota*` |
| **多版本快照** | `OpVersionOperation` | `opMultiVersionOp` |
| **MP 生命周期（admin）** | `OpCreateMetaPartition` / `Delete` / `Update` / `Decommission` / `Load` | `opCreateMetaPartition` ... |
| **Raft 成员变更（admin）** | `OpAddMetaPartitionRaftMember` / `Remove` / `TryToLeader` | `opAddMetaPartitionRaftMember` ... |
| **节点控制** | `OpMetaNodeHeartbeat` / `OpIsRaftStatusOk` | `opMasterHeartbeat`、`opIsRaftStatusOk` |
| **混合云迁移** | `OpMetaUpdateExtentKeyAfterMigration` / `OpDeleteMigrationExtentKey` | `opMeta*Migration` |
| **default** | 未知 op | 直接返回 `unknown Opcode` 错误 |

> 💡 **看一遍这张表 ≈ 看完了 MetaNode 对外暴露的所有能力**。Layer 5/6/7 我们会按这张表里的 4 大块（Inode/Dentry/Extent/Multipart）逐个深入。

### 段 3：错误包装

```go
if err != nil {
    err = errors.NewErrorf("%s [%s] req: %d - %s", remoteAddr, p.GetOpMsg(),
        p.GetReqID(), err.Error())
}
```

**统一在最外层包装 reqID + remoteAddr + opMsg**——下层 handler 不需要重复格式化。Layer 2 在 `serveConn` 里看到的错误日志就是这条出来的。

---

## 4. 标准 handler 模板（重点）

所有 `opXxx` 函数都长得高度相似。以 [opCreateInode](../../metanode/manager_op.go#L300) 为例：

```go
func (m *metadataManager) opCreateInode(conn net.Conn, p *Packet,
    remoteAddr string,
) (err error) {
    // ① 反序列化业务参数
    req := &CreateInoReq{}
    if err = json.Unmarshal(p.Data, req); err != nil {
        p.PacketErrorWithBody(proto.OpErr, ([]byte)(err.Error()))
        m.respondToClientWithVer(conn, p)
        return
    }

    // ② 路由：根据 PartitionID 找到本地 mp
    mp, err := m.getPartition(req.PartitionID)
    if err != nil {
        p.PacketErrorWithBody(proto.OpErr, ([]byte)(err.Error()))
        m.respondToClientWithVer(conn, p)
        return
    }

    // ③ 主从代理：非 leader 把请求转发给 leader
    if !m.serveProxy(conn, mp, p) {
        return
    }

    // ④ 多版本快照检查（vol 在做版本切换时拒绝/排队）
    if err = m.checkMultiVersionStatus(mp, p); err != nil {
        m.respondToClientWithVer(conn, p)
        return
    }

    // ⑤ 调用具体 mp 的业务方法 ── ★ Layer 4 边界
    err = mp.CreateInode(req, p, remoteAddr)

    // ⑥ 更新响应里的多版本序号
    m.updatePackRspSeq(mp, p)

    // ⑦ 把结果写回客户端
    m.respondToClientWithVer(conn, p)
    return
}
```

**这个 7 步模板要刻进脑子里**——后面 Layer 5/6 看到的所有 opXxx 全都是这个套路：

| 步骤 | 干啥 | 失败时 |
|---|---|---|
| ① Unmarshal | JSON 解析业务参数（请求结构体） | 立即回 OpErr |
| ② getPartition | 在 `partitions` map 里查 mp | 查不到回 OpErr |
| ③ serveProxy | 非 leader 通过 connPool 转发到 leader | 已转发 → 直接 return |
| ④ checkMultiVersionStatus | 多版本快照状态门禁 | 业务错误 |
| ⑤ mp.XXX | **真正的元数据操作（进 raft / 改内存树）** | 业务错误 |
| ⑥ updatePackRspSeq | 给响应打上当前 verSeq | - |
| ⑦ respondToClientWithVer | 把 packet 写回 conn | - |

> ⚠️ **3 类常见错误进入这个模板的方式不同**：
> - **协议层错误**（unmarshal 失败、mp 不存在）→ 步骤 ①② 直接回错
> - **路由错误**（不是 leader）→ 步骤 ③ 转发掉，本地直接 return
> - **业务错误**（inode 已存在等）→ 步骤 ⑤ 里设置 `p.ResultCode`，步骤 ⑦ 正常回包

---

## 5. `getPartition`：本层的灵魂

[manager.go:572](../../metanode/manager.go#L572)

```go
func (m *metadataManager) getPartition(id uint64) (mp MetaPartition, err error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    mp, ok := m.partitions[id]
    if ok { return }
    err = errors.NewErrorf("unknown meta partition: %d", id)
    return
}
```

简单到只有 6 行，但要记住几点：

- **读写锁的读路径极快**：所有 op 都走 RLock，只有 mp 增删（CreateMP/DeleteMP/Decommission）走 WLock。**这就是 MetaNode 高并发的基础**。
- **错误信息 `unknown meta partition`** 正是 Layer 2 [server.go:106](../../metanode/server.go#L106) 里被 warn 级别过滤的字符串之一——上下层串起来了。
- **没找到 = 客户端发错节点了**。客户端会从 master 拉新的 `vol view` 重试。这是 CubeFS 容错的常见模式。

---

## 6. `serveProxy`：非 leader 怎么处理写请求？

> 完整实现在 [metanode/manager_proxy.go](../../metanode/manager_proxy.go)，本层只关注**它在分发流程里的位置**，细节留给 Layer 4。

**逻辑**：

```
serveProxy(conn, mp, p) returns shouldHandleLocally bool

if mp.IsLeader() || p.IsReadOperation() {
    return true                         // 本地处理
}
// 否则：
//   1. 从 connPool 拿一条到 leader 的 TCP 连接
//   2. 把 p 整包发过去
//   3. 把 leader 的响应写回原 conn
//   4. return false  ── 告诉 handler "我已经搞定了，你别再管"
```

**3 个隐含约定**：

1. **读操作允许在 follower 执行**（前提是 vol 开启 `followerRead`）。
2. **写操作必须在 leader 执行**——所以 follower 收到写请求会做透明转发。
3. 转发用的是**本节点维护的 connPool**（[manager.go:543](../../metanode/manager.go#L543) `m.connPool = util.NewConnectPool()`），而不是 smux 池——因为 mp 之间是节点级的稳定连接，按 op 复用就够了。

> 💡 这就解释了为什么 Layer 1 stop 时只关 smux pool 不关 connPool——connPool 在 manager.Stop 流程里关。

---

## 7. `checkForbidWriteOpOfProtoVer0`：升级兼容门禁

[manager.go:209](../../metanode/manager.go#L209)

```go
if pktProtoVersion != proto.PacketProtoVersion0 { return nil }   // 新协议直接放行
if m.metaNode.nodeForbidWriteOpOfProtoVer0 { return error }      // 节点级禁
if mpForbidWriteOpOfProtoVer0 { return error }                    // vol 级禁
return nil
```

这是分级存储/混合云改造之后加的**协议版本门禁**：

- 集群升级期间，老客户端（用 ProtoVer0）发来的写请求可能数据格式和新版不兼容
- 通过 master 下发的开关（Layer 1 [register](../../metanode/metanode.go#L518) 拉的 `UpgradeCompatibleSettings`）控制
- 颗粒度：**节点级 / 卷级** 两层
- 命中后的拒绝结果码 `OpWriteOpOfProtoVerForbidden` 正是 Layer 2 [server.go:101](../../metanode/server.go#L101) 那个会**直接断连接**的特殊码

**Layer 1 → Layer 2 → Layer 3 之间的"暗线"由这一处串起来**——理解了它，等于理解了 CubeFS 升级期间的兼容性策略。

---

## 8. 限速：`limitFactor`（点到为止）

[manager.go:116](../../metanode/manager.go#L116) `limitFactor map[uint32]*rate.Limiter` 是按 op 类型分组的 token bucket 限速器，由 `golang.org/x/time/rate` 实现。**部分 handler 在进入业务逻辑前会先 wait limiter**——比如 `readDirIops` 控制 `OpMetaReadDir` 的 QPS。

> 这一块和 Layer 5 的 inode 列表性能强相关，等到 Layer 5 再展开。

---

## 9. 启动期：`onStart` 是怎么把 mp 加载进来的

[manager.go:542](../../metanode/manager.go#L542)

```go
func (m *metadataManager) onStart() (err error) {
    m.connPool = util.NewConnectPool()
    m.initFileStatsConfig()
    err = m.loadPartitions()           // ★ 从磁盘扫 partition_<id>/ 目录恢复 mp
    if err != nil { return }
    m.stopC = make(chan struct{})
    m.startCpuSample()                  // 后台采 CPU
    m.startSnapshotVersionPromote()     // 多版本推进
    m.startUpdateVolumes()              // 周期性从 master 拉 vol view
    m.startGcTimer()                    // 主动 GC
    return
}
```

**`loadPartitions`** 就是把磁盘上的 mp 数据目录（`partition_<id>`）逐个加载、回放 raft log、注入到 `m.partitions` map 的过程——**这是 Layer 4 的主菜**，本层不展开。

但要记住一个事实：**`onStart` 完成之前 `partitions` map 是空的**，此时 TCP 端口已经开了（Layer 1 顺序），所以早进来的请求会在 `getPartition` 里拿到 `unknown meta partition` 错误并被客户端重试。**这就是 Layer 1 那个"端口先开 mp 后加载"的设计能跑通的原因**。

---

## 10. 本层关键问题自测

读完 Layer 3 应该能回答：

1. ✅ 一个写请求从 `HandleMetadataOperation` 进来到调用 `mp.CreateInode` 之间经过哪 7 步？
2. ✅ `getPartition` 用读写锁的哪一边？为什么这是 MetaNode 高并发的基础？
3. ✅ 非 leader 节点收到写请求会怎么处理？走的是哪个连接池？
4. ✅ AdminOp 为什么强制 warn 级日志？哪些 op 是 admin op？
5. ✅ 心跳 op 的 metric label 为什么不查 mp？
6. ✅ `OpWriteOpOfProtoVerForbidden` 这个错误码是从 Layer 1 哪个配置项 → Layer 3 哪个函数 → Layer 2 哪个分支串出来的？
7. ✅ Layer 3 与 Layer 4 的代码边界在哪一行？

---

## 11. 给 Layer 4 的钩子

下层我们要回答：

- `mp.CreateInode(req, p, ...)` 怎么把"修改 inode 树"这件事走 raft 提交？
- `metaPartition` 结构里的 raft 实例和 `m.raftStore` 是什么关系？（multi-raft）
- `loadPartitions` 怎么从磁盘恢复一个 mp 的状态机？snapshot + log replay 顺序是怎样的？
- `Apply / ApplySnapshot / Snapshot` 三个 fsm 接口在 [partition_fsm.go](../../metanode/partition_fsm.go) 里的实现长什么样？
- 一次 `serveProxy` 转发到 leader 后，leader 上的 raft 提案是怎么走完的？
