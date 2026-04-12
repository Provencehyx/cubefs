# MetaNode 七层解析 · Layer 5：Inode 子系统

> 入口文件：[metanode/inode.go](../../metanode/inode.go) · [metanode/partition_fsmop_inode.go](../../metanode/partition_fsmop_inode.go) · [metanode/partition_op_inode.go](../../metanode/partition_op_inode.go)
> 核心问题：**在 `Apply` 被 raft 回调之后，`fsmXxx` 到底怎么把一个 `*Inode` 塞进 / 改动 / 删出那棵内存 BTree？inoID 怎么分配才不会撞车？被 unlink 的 inode 去哪儿了？**

> 💡 Layer 4 留下的钩子：`mp.fsmCreateInode / fsmUnlinkInode / fsmExtentsTruncate` 这些"最终改 BTree 的人"。本层就是把这些人一个个拉出来看。

---

## 1. 这一层在系统中的位置

```
   Layer 4 (partition_fsm.go Apply)
        │
        │  switch msg.Op {
        │    case opFSMCreateInode: mp.fsmCreateInode(ino)
        │    case opFSMUnlinkInode: mp.fsmUnlinkInode(ino, uniqID)
        │    case opFSMAppendExtentsWithCheck: ...
        │    case opFSMExtentTruncate: ...
        │    case opFSMEvictInode: ...
        │    case opFSMInternalDeleteInode: ...
        │  }
        ▼
┌──────────────────────────────────────────────┐
│                Layer 5 本层                   │
│                                              │
│   mp.inodeTree  (*BTree of *Inode)           │
│        ▲                                    │
│        │  ReplaceOrInsert / CopyGet / Delete │
│        │                                    │
│   fsmCreateInode / fsmUnlinkInode / ...      │
│   fsmAppendExtentsWithCheck / fsmTruncate    │
│   fsmEvictInode / internalDeleteInode        │
│                                              │
│   mp.freeList  (待 GC 队列)                   │
│        ▲                                    │
│        │  Push / Pop                         │
│        │                                    │
│   startFreeList 后台线程 ──► 真正释放 extent  │
└──────────────────────────────────────────────┘
        │
        ▼
   Layer 6 (dentry 子系统)
```

**记住一句话**：**Layer 5 负责 inode 生命周期的每一个离散事件**（创建 / 链接 / 断链 / 截断 / 驱逐 / 真删），而这些事件最终都落到**同一棵 `inodeTree` BTree** 上。

---

## 2. `Inode` 结构体：字段分组速记

[inode.go:78](../../metanode/inode.go#L78)

```go
type Inode struct {
    sync.RWMutex                       // ★ 细粒度锁：inode 内部并发

    // —— POSIX 核心字段 ——
    Inode           uint64             // inode id（BTree key）
    Size            uint64             // 文件大小（字节）
    Generation      uint64             // 写代（每次写 +1，给 client 判缓存）
    CreateTime      int64
    AccessTime      int64
    ModifyTime      int64
    Reserved        uint64             // 预留
    LeaseExpireTime uint64             // 租约到期时间戳

    Type         uint32                // 文件/目录/符号链接
    Uid, Gid     uint32
    NLink        uint32                // 硬链接数
    Flag         int32                 // DeleteMarkFlag / InodeDelTop / ...
    StorageClass uint32                // ★ Replica / BlobStore / 混合云
    ClientID     uint32

    LinkTarget   []byte                // 符号链接目标

    // —— 多版本快照 ——
    multiSnap *InodeMultiSnap          // 快照链（descending by verSeq）

    // —— Extent 容器（混合云重构后） ——
    HybridCloudExtents          *SortedHybridCloudExtents          // 主 extent
    HybridCloudExtentsMigration *SortedHybridCloudExtentsMigration // 迁移中的 extent
}
```

**4 个字段分组**：

| 分组 | 字段 | 谁在用 |
|---|---|---|
| POSIX 标识 | Inode / Type / Uid / Gid / NLink | 所有元数据操作 |
| 大小与时间 | Size / Generation / C/A/M Time | 写路径、client 缓存 |
| 存储类 | StorageClass / HybridCloudExtents* | 副本 vs EC vs 混合云 |
| 删除标记 | Flag / LeaseExpireTime | unlink / evict / GC |

### 2.1 为什么 inode 自己带 `sync.RWMutex`？

Layer 4 告诉你 **Apply 全局串行**（`nonIdempotent` 锁），理论上不需要 inode 粒度的锁，但：

1. **读路径不持全局锁**：`getInode` 直接访问 BTree，返回的 `*Inode` 可能被**后续 Apply 并发修改字段**（比如 `AppendExtents`）。inode 自己的 `RWMutex` 保护 extent 列表的读/写一致。
2. **inode 内部字段修改跨多步**（比如 `AppendExtentWithCheck` 同时改 `Size` / `Generation` / extent 列表），需要原子性。

> ⚠️ **新人经典误区**：以为"有 Apply 锁就够了"。实际上"写路径 × 读路径"是两把锁的协作。

### 2.2 Marshal 格式（给 Layer 7 埋钩子）

[inode.go](../../metanode/inode.go) 头部的注释画得很清楚：

```
key:   +-------+-------+
       | item  | Inode |
       +-------+-------+
       | bytes |   8   |

value: +------+------+-----+----+----+----+--------+------------------+
       | Type | Size | Gen | CT | AT | MT | ExtLen | MarshaledExtents |
       +------+------+-----+----+----+----+--------+------------------+
       |  4   |  8   |  8  | 8  | 8  | 8  |   4    |      ExtLen      |
```

→ snapshot 就是把 BTree 里每个 inode 按 `{KeyLen, Key, ValLen, Val}` 串起来写文件。Layer 7 细讲。

---

## 3. inoID 分配：`nextInodeID` 与 `Cursor`

每个 mp 在 Master 创建时就拿到一段独立的 `[Start, End]` 区间，这段区间只归这一个 mp 用，永不交叉。分配流程：

[partition.go:1365](../../metanode/partition.go#L1365)

```go
func (mp *metaPartition) nextInodeID() (uint64, error) {
    for {
        cur := atomic.LoadUint64(&mp.config.Cursor)
        end := mp.config.End
        if cur >= end {
            return 0, ErrInodeIDOutOfRange        // ★ 区间耗尽
        }
        newId := cur + 1
        if atomic.CompareAndSwapUint64(&mp.config.Cursor, cur, newId) {
            return newId, nil                     // ★ CAS 抢占
        }
    }
}
```

**3 个关键事实**：

1. **分配在 leader 上完成，不走 raft**：`CreateInode` 流程里先 `nextInodeID` 拿到 id，再把 `*Inode` 序列化 submit 到 raft。
2. **CAS 循环保证多 goroutine 并发安全**——Layer 4 讲过 Apply 是串行的，但**写请求到达 `CreateInode` 的入口是并发的**，`nextInodeID` 发生在 submit **之前**。
3. **follower 怎么跟上 Cursor？** 看 Apply 函数：
   ```go
   case opFSMCreateInode:
       if mp.config.Cursor < ino.Inode {
           mp.config.Cursor = ino.Inode     // ★ 同步 Cursor
       }
       resp = mp.fsmCreateInode(ino)
   ```
   → follower 通过 raft 日志里 inode 的 `Inode` 字段**被动推进 Cursor**，这样选主切换后新 leader 也能接着分配。

> 💡 **为什么不把 Cursor 做成 raft 命令**？因为那会让**每次 CreateInode 额外绕一圈 raft**。现在的设计是"惰性同步"：leader 本地 CAS，follower 通过 Apply 追平，选主切换的瞬间新 leader 的 Cursor 已经 ≥ 最大 inode id，分配语义不破坏。

---

## 4. 6 类 fsm 写操作逐个过

`Apply` 里会 switch 到的 `fsmXxx` 函数有几十个，但抓住 6 个最典型的即可：

### 4.1 `fsmCreateInode`：最简单的样板

[partition_fsmop_inode.go:78](../../metanode/partition_fsmop_inode.go#L78)

```go
func (mp *metaPartition) fsmCreateInode(ino *Inode) (status uint8) {
    if status = mp.uidManager.addUidSpace(ino.Uid, ino.Inode, nil); status != proto.OpOk {
        return
    }
    status = proto.OpOk
    if _, ok := mp.inodeTree.ReplaceOrInsert(ino, false); !ok {
        status = proto.OpExistErr                    // ★ 第二个参数 false = "不要覆盖"
    }
    return
}
```

核心就两步：
1. **配额检查**：把这个 uid 下的 inode 计到配额里
2. **插入 BTree**：`ReplaceOrInsert(ino, false)` 的 false 参数 = "如果已存在不要覆盖，而是返回失败"

> 💡 这 6 行就是整个写流程的"着陆点"——**Layer 4 整套 raft 机制存在的意义，就是为了让这 6 行在三个副本上各跑一次并达到相同状态**。

### 4.2 `fsmCreateLinkInode`：硬链接

[partition_fsmop_inode.go:128](../../metanode/partition_fsmop_inode.go#L128)

```go
item := mp.inodeTree.CopyGet(ino)          // ① 找到已存在的 inode
if item == nil || i.ShouldDelete() { ... return OpNotExistErr }
if !mp.uniqChecker.legalIn(uniqID) { return }  // ② 幂等检查
i.IncNLink(ino.getVer())                   // ③ NLink++
```

**`uniqChecker` 是什么？** 写幂等去重器——client 重试时带同一个 `uniqID`，mp 已经做过就直接跳过。
**为什么用 `CopyGet` 而不是 `Get`？** 因为后续要修改 inode 字段，需要拿到可写版本（BTree 内部会做 CoW）。

### 4.3 `fsmUnlinkInode`：断链（不是真删）

[partition_fsmop_inode.go:331](../../metanode/partition_fsmop_inode.go#L331)

简化逻辑：

```go
inode := mp.inodeTree.CopyGet(ino)
if inode == nil || inode.ShouldDelete() { return OpNotExistErr }

ext2Del, doMore, status := inode.unlinkTopLayer(...)  // ① NLink-- 并决定是否真删

if inode.IsEmptyDirAndNoSnapshot() && NLink < 2 {
    mp.inodeTree.Delete(inode)                         // ② 空目录直接删
} else if inode.IsTempFile() && NLink == 0 {
    mp.freeList.Push(inode.Inode)                      // ★★ 文件先进 freeList
}

if len(ext2Del) > 0 {
    mp.extDelCh <- ext2Del                             // ★ extent 走异步删除 channel
}
```

**3 个关键事实**：

1. **文件 unlink 后并不立即从 BTree 删除**，而是 `Push(freeList)` → 背景 goroutine 真删。
   **为什么？** 因为 **extent 数据在 DataNode 上**，client 可能还持有 fd，需要一个延迟窗口：
   - client close 后重引用 → inode 还活着 → OK
   - client 直接崩 → 租约过期后 freeList 异步清理
2. **目录 unlink 是同步删除**（`IsEmptyDirAndNoSnapshot`），因为目录没有外部引用。
3. **`extDelCh` 与 `freeList` 是两条独立的延迟删除链路**：
   - `freeList` 是 **inode 层**的 GC（整个 inode）
   - `extDelCh` 是 **extent 层**的 GC（truncate / append 覆盖产生的废数据块）

### 4.4 `fsmAppendExtentsWithCheck`：写文件的核心

[partition_fsmop_inode.go:495](../../metanode/partition_fsmop_inode.go#L495)

这是**写路径最热的函数**，一次 `write(fd, buf, n)` 在 client 端被切成 extent，最终走到这里。简化骨架：

```go
fsmIno := mp.inodeTree.CopyGet(ino).(*Inode)
if fsmIno.ShouldDelete() { return OpNotExistErr }

eks := ino.HybridCloudExtents.sortedEks.(*SortedExtents).CopyExtents()
// 只处理 eks[0]，eks[1:] 进 discardExtentKey
discardExtentKey := eks[1:]

appendExtParam := &AppendExtParam{ ek: eks[0], discardExtents: ..., ... }
delExtents, status = fsmIno.AppendExtentWithCheck(appendExtParam)    // ① 真正改 inode 的 extent 列表

fsmIno.DecSplitExts(mpId, delExtents)     // ② 引用计数减
mp.extDelCh <- delExtents                  // ③ 旧 extent 异步回收
```

**`AppendExtentWithCheck` 内部**（[inode.go](../../metanode/inode.go)）会：
1. 在 `HybridCloudExtents.sortedEks` 里找重叠区间
2. 覆盖/合并 → 产生一组"被替换掉"的旧 extent
3. 返回 `delExtents` 给 fsm 丢进 `extDelCh`

**`status == OpConflictExtentsErr` 的特殊处理**：
```go
if status == proto.OpConflictExtentsErr {
    mp.extDelCh <- eks[:1]                 // 新写入的 ek 本身也要删（避免 DataNode 垃圾残留）
}
```
这是为了**在冲突时清理 client 已经写到 DataNode 上的孤儿 extent**。

### 4.5 `fsmExtentsTruncate`：截断文件

[partition_fsmop_inode.go:650](../../metanode/partition_fsmop_inode.go#L650)

```go
i := mp.inodeTree.Get(ino).(*Inode)
if !proto.IsStorageClassReplica(i.StorageClass) { return OpArgMismatchErr }  // ① EC 存储不能 truncate
if proto.IsDir(i.Type) { return OpArgMismatchErr }                           // ② 目录也不能

i.Lock()                                                                      // ★ inode 细粒度锁
defer i.Unlock()
delExtents := i.ExtentsTruncate(ino.Size, ino.ModifyTime, insertSplitKey)
i.DecSplitExts(mp.config.PartitionId, delExtents)
mp.extDelCh <- delExtents
```

**为什么要 `i.Lock()` 而不只靠 Apply 的全局锁？**
因为 `ExtentsTruncate` 内部会改 `i.Size` + 遍历 extent 列表 + 生成 `delExtents`——这段期间可能有**读路径**（`getInode`）并发访问 inode 字段，只有 inode 自带的锁能保护它。

### 4.6 `fsmEvictInode` & `internalDeleteInode`：两步 GC

```
fsmUnlinkInode (NLink-- → NLink==0)
        │
        ▼
   mp.freeList.Push(inodeID)         ← inode 还在 BTree 里
        │
        │  startFreeList 后台线程定期扫
        ▼
   删除 DataNode 上的 extent
        │
        ▼
   发起 opFSMInternalDeleteInode  ← 再过一次 raft
        │
        ▼
   internalDeleteInode:
       mp.inodeTree.Delete(ino)     ← 这次才真从 BTree 拿掉
       mp.freeList.Remove(ino)
       mp.extendTree.Delete(...)    ← xattr 一并清理
```

**`fsmEvictInode` 是另一条路径**：client 主动调用 `Evict`（比如 unmount 时），把 `Flag |= DeleteMarkFlag` 并进 freeList，和自然 unlink 是两种入口，最终归并到同一个 GC 循环。

> 💡 **为什么 `internalDeleteInode` 也要走 raft**？因为它本质是一次 BTree 修改，**副本之间必须达成一致**——不能让 leader 自己先删了而 follower 还留着。

---

## 5. freeList：异步 GC 的"暂存区"

[free_list.go](../../metanode/free_list.go)

```go
type freeList struct {
    sync.Mutex
    list   []uint64
    index  map[uint64]int   // 去重
}
```

**数据结构很朴素**：一个 slice + 一个 map（用来 O(1) 去重 / Remove）。

**谁在消费它？** [partition.go](../../metanode/partition.go) 里的 `startFreeList` 启动一个后台 goroutine：
1. 每隔一段时间 `Pop` 一批
2. 对每个 inode：发送 `ExtentsDelete` 请求到 DataNode 真删 extent
3. 成功后提交 `opFSMInternalDeleteInode` 走 raft 从 BTree 抠掉

**两个不变式**：
- `freeList` 里的 inode 一定在 `inodeTree` 里（`ShouldDelete() == true`）
- `inodeTree` 里 `ShouldDelete() == true` 的 inode 一定在 `freeList` 里（否则会变成孤儿）

> ⚠️ **Layer 7 预告**：snapshot 持久化时要**跳过** `ShouldDelete == true` 的 inode 吗？**不跳**——因为 snapshot 必须包含完整 BTree 状态，否则 replay 后 freeList 会丢。具体细节到 Layer 7 讲。

---

## 6. 读路径：为什么不走 raft

只读操作（`getInode` / `hasInode` / `listInode`）**直接读 BTree，不经过 submit**：

```go
func (mp *metaPartition) getInode(ino *Inode, listAll bool) (resp *InodeResponse) {
    i := mp.getInodeByVer(ino)
    if i == nil { return OpNotExistErr }
    resp.Msg = i.Copy().(*Inode)      // ★ 返回副本，不是 tree 里的原对象
    resp.Msg.AccessTime = timeutil.GetCurrentTimeUnix()
    return
}
```

**3 个设计点**：

1. **返回 `i.Copy()` 而非指针**——避免外层 goroutine 持有 tree 里的活对象，被后续 Apply 并发修改。
2. **AccessTime 只在返回值里改，不改 BTree**——老注释 `FIXME: not protected by lock yet` 说明这个字段暂时是近似值，不走 raft 同步。
3. **读发生在哪个副本？** 默认在 leader（通过 Layer 3 的 leader 校验），但 CubeFS 支持"stale read"让 follower 也能读（应对读热点），代价是可能读到略旧的数据。

---

## 7. 本层关键问题自测

1. ✅ `Inode.Inode` 字段（id）和 `Inode` 类型名重名，BTree 比较用的是哪个？
2. ✅ `nextInodeID` 为什么不走 raft？follower 怎么保持 Cursor 一致？
3. ✅ `fsmCreateInode` 的 `ReplaceOrInsert(ino, false)` 第二个参数如果传 true 会怎样？
4. ✅ 为什么文件 unlink 是"先进 freeList 后真删"而目录可以直接删？
5. ✅ `extDelCh` 和 `freeList` 各自负责什么？它们是一套还是两套 GC？
6. ✅ `fsmAppendExtentsWithCheck` 在 `OpConflictExtentsErr` 时为什么要把 `eks[:1]` 也扔进 `extDelCh`？
7. ✅ 为什么 `Inode` 自带 `sync.RWMutex`？Apply 的 `nonIdempotent` 锁不是够了吗？
8. ✅ `internalDeleteInode` 为什么也要走 raft？
9. ✅ 读路径为什么返回 `i.Copy()` 而不是原指针？
10. ✅ snapshot 会包含 `ShouldDelete == true` 的 inode 吗？为什么？

---

## 8. 给后续 Layer 的钩子

- **Layer 6 (Dentry)**：`dentryTree` 的 key 是什么？`(ParentId, Name)` 怎么映射？rename 是单 mp 内就能做，还是要跨 mp 分布式事务？rename 和 unlink 会不会在 inode 子系统留下副作用（比如 NLink 变化）？
- **Layer 7 (持久化)**：snapshot 怎么把 `inodeTree` 序列化？`freeList` 和 `extDelCh` 里"在途"的数据怎么恢复？`mp.load` 时 inode 的 extent 列表（`HybridCloudExtents`）怎么反序列化回来？
- **存疑点**（带着问 Layer 7）：
  - `uniqChecker` 是不是也要持久化？重启后幂等窗口怎么办？
  - `Cursor` 字段存在 `config` 里，那它的持久化走 snapshot 还是单独写？
  - `multiSnap` / `HybridCloudExtentsMigration` 这些"高级字段"在 snapshot 里的向后兼容是怎么处理的？
