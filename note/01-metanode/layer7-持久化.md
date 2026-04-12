# MetaNode 七层解析 · Layer 7：持久化（snapshot + raft log）

> 入口文件：[metanode/partition_store.go](../../metanode/partition_store.go) · [metanode/partition_store_ticket.go](../../metanode/partition_store_ticket.go) · [metanode/partition.go](../../metanode/partition.go)（`load` / `store` / `LoadSnapshot`）· [metanode/partition_fsm.go](../../metanode/partition_fsm.go)（`Snapshot` / `ApplySnapshot`）
> 核心问题：**几棵内存 BTree 是怎么周期性落成磁盘快照的？重启时怎么从磁盘 + raft log replay 恢复成"一分钱不差"的状态？两台 MetaNode 之间同步 snapshot（新节点加入 / 落后太多的 follower）又是另一条路径，两者怎么共存？**

> 💡 Layer 4~6 的共同钩子：**内存 BTree 怎么才能不丢？** 这是元数据系统最硬的一层，所有正确性最终都收敛于此。

---

## 1. 这一层在系统中的位置

MetaNode 有**两套**持久化，各司其职：

```
         ┌──────────────── 本机持久化 ────────────────┐
         │                                           │
 ① 本地 snapshot：周期性把 BTree dump 到                │
    <rootDir>/snapshot/ 下的若干文件                   │
    （inode / dentry / extend / multipart / ...）    │
         │                                           │
 ② 本地 raft log：raft 引擎自己维护                    │
    <raftDir>/wal-<partitionID>/                     │
         │                                           │
         │  重启时：load(snapshot) + replay(log)     │
         │         → 内存 BTree                       │
         └───────────────────────────────────────────┘

         ┌──────────── 跨节点 snapshot 同步 ───────────┐
         │                                           │
 ③ raft 协议的 snapshot 传输                           │
    leader: Snapshot()   → newMetaItemIterator(mp)   │
    follower: ApplySnapshot(iter)                    │
                                                     │
    这条路径是"按条目流式传输"，和本地文件格式不一样！     │
         └───────────────────────────────────────────┘
```

**记住一句话**：**"本地 snapshot（文件）" 与 "raft snapshot（字节流）" 是两种不同的序列化**。本地 snapshot 是给本机 `load()` 读的，raft snapshot 是给 follower `ApplySnapshot()` 读的；两者共存是因为它们有不同的优化点：
- 本地 snapshot **按类型分文件**、可并行 load、带 crc 独立校验
- raft snapshot 是**按条目的统一信封**（`MetaItem`）流，跨网络逐条 Next/Unmarshal

---

## 2. 本地 snapshot：目录布局

```
<rootDir>/
├── meta                       ← mp 静态配置（PartitionId / Start / End / Peers）JSON
├── snapshot/                  ← 当前快照
│   ├── inode                  ← inodeTree 序列化
│   ├── dentry                 ← dentryTree 序列化
│   ├── extend                 ← xattr 树
│   ├── multipart              ← S3 multipart 树
│   ├── tx_info                ← 进行中事务
│   ├── tx_rb_inode            ← inode rollback 记录
│   ├── tx_rb_dentry           ← dentry rollback 记录
│   ├── apply                  ← applyID
│   ├── transactionID          ← txID 分配器
│   ├── uniqID                 ← uniqID 分配器
│   ├── uniqChecker            ← 写幂等窗口
│   ├── multiVer               ← 多版本快照 list
│   └── .sign                  ← 各 crc 的文本形式（空格分隔）
├── .snapshot/                 ← 写入中的 tmp（最后 rename 成 snapshot/）
└── .snapshot_backup/          ← 旧 snapshot 的备份（rename 中间态）
```

文件名常量集中在 [partition_store.go:37](../../metanode/partition_store.go#L37)。

### 2.1 原子 swap：`.snapshot` → `snapshot` → `.snapshot_backup`

[partition.go:1264 store()](../../metanode/partition.go#L1264) 的核心片段：

```go
// ① 写到 tmpDir = ".snapshot"
os.MkdirAll(tmpDir, 0o775)
for _, f := range storeFuncs {
    crc, err := f(tmpDir, sm)
    crcBuffer.WriteString(fmt.Sprintf("%d", crc))
}
fileutil.WriteFileWithSync(path.Join(tmpDir, SnapshotSign), crcBuffer.Bytes(), ...)

// ② rename dance
// - 如果已有 snapshot/，先改名为 .snapshot_backup/
os.Rename(snapshotDir, backupDir)
// - 再把 .snapshot/ 改名为 snapshot/
if err = os.Rename(tmpDir, snapshotDir); err != nil {
    _ = os.Rename(backupDir, snapshotDir)    // ★ 失败回滚
    return
}
// - 删掉备份
os.RemoveAll(backupDir)

mp.storedApplyId = sm.applyIndex              // ★ 成功后才更新
```

**3 个保证**：

1. **`Rename` 是文件系统原子操作**——要么看到完整的旧 snapshot，要么看到完整的新 snapshot，不会有"半写状态"。
2. **写 tmp 成功前，旧 snapshot 始终是完整的**——任何时刻机器掉电都能用旧 snapshot + raft log 恢复。
3. **`storedApplyId` 只在第三步之后更新**——如果 rename 之前崩溃，下次启动时 `storedApplyId` 仍是旧值，raft log 会从旧的 applyID replay，一条不丢。

> 💡 这是经典的 **"write-rename-cleanup"** 原子写模式。新人可以把这 25 行代码背下来——几乎任何需要"原子更新一组文件"的场景都能复用。

### 2.2 `storeInode`：inode 文件的格式

[partition_store.go:1169](../../metanode/partition_store.go#L1169)

```go
sm.inodeTree.Ascend(func(i BtreeItem) bool {
    ino := i.(*Inode)
    ino.MarshalV2(buf)
    data := buf.Bytes()

    binary.BigEndian.PutUint32(lenBuf, uint32(len(data)))
    fp.Write(lenBuf); sign.Write(lenBuf)
    fp.Write(data);   sign.Write(data)
    return true
})
crc = sign.Sum32()
```

**格式**（与 [inode.go](../../metanode/inode.go) 顶部注释一致）：

```
+----+------+----+------+----+------+-----+
| L1 | Ino1 | L2 | Ino2 | L3 | Ino3 | ... |
+----+------+----+------+----+------+-----+
  |    |
  |    └─ MarshalV2 后的 inode 字节流（含 Inode/Size/Gen/C/A/M Time/...）
  └────── 4 字节 BigEndian 长度
```

**为什么加 length 前缀**？因为 inode value 是变长的（混合云 extent 列表长度不定），`reader` 需要先知道长度才能准确 `ReadFull`。load 侧（[partition_store.go:160 loadInode](../../metanode/partition_store.go#L160)）的对称逻辑就是先读 4 字节长度、再读 length 字节。

**`Ascend` 保证按 key 有序**——所以 snapshot 文件本身也是有序的，便于 diff / 调试。

**crc32 边写边算**：最后一起写进 `.sign` 文件。load 时再校验。**crc 错 = 硬错，直接退出进程**（见 `ErrSnapshotCrcMismatch`）。

### 2.3 `storeDentry` 同理

[partition_store.go:1249](../../metanode/partition_store.go#L1249) 和 inode 几乎一模一样：遍历 `sm.dentryTree.Ascend`、`Dentry.MarshalV2` → 长度前缀 + 数据 + crc。

> 💡 **一致性关键**：`storeInode` 和 `storeDentry` 吃的都是 `sm.inodeTree` / `sm.dentryTree`——**同一个 `storeMsg` 里的两棵树必须来自同一时刻的视图**。这个"同一时刻"是怎么拿到的？看 2.4。

### 2.4 BTree 的快照视图：copy-on-write

注意 `storeFunc` 的签名吃的是 `sm.inodeTree` 而不是 `mp.inodeTree`——`storeMsg` 里存的是**调用 `GetTree()` 拿到的 tree 引用**。CubeFS 的 BTree（`util/btree`）支持 CoW：只要有人拿着旧版引用，新的写会 CoW 一条修改路径，旧版引用看到的依然是老快照。

所以 dump 过程中**不阻塞 Apply**——dump goroutine 读的是 dump 时刻的冻结视图，主流程继续在新视图上改 BTree。这就是 CubeFS 元数据"不停机快照"的核心技巧。

---

## 3. 周期性 dump 的触发链路

[partition_store_ticket.go:47 startSchedule](../../metanode/partition_store_ticket.go#L47) 是**整个 dump 调度的中枢**。看骨架：

```go
func (mp *metaPartition) startSchedule(curIndex uint64) {
    timer := time.NewTimer(...)                    // 周期性 tick
    timerCursor := time.NewTimer(intervalToSyncCursor)

    go func() {
        for {
            select {
            case msg := <-mp.storeChan:
                switch msg.command {
                case startStoreTick:  timer.Reset(intervalToPersistData)   // leader 才触发
                case stopStoreTick:   timer.Stop()                          // follower 不 dump
                case opFSMStoreTick:  msgs = append(msgs, msg)              // ★ 待 dump
                }
            case <-timer.C:
                // ★ 1. 定时到点：leader 往自己提一个 raft 提案
                mp.submit(opFSMStoreTick, nil)
            case <-timerCursor.C:
                // ★ 2. cursor 变化了：也要同步到 follower
                mp.submit(opFSMSyncCursor, Buf)
                mp.submit(opFSMSyncTxID, Buf)
            }
        }
    }()
}
```

**整个 dump 的事件链**：

```
 leader 上的 timer 到点
     │
     ▼
 mp.submit(opFSMStoreTick, nil)            ← ① 走 raft
     │
     ▼
 每个副本的 Apply 里：
   case opFSMStoreTick:
     sm := mp.getStoreMsg(...)
     mp.storeChan <- sm                    ← ② 打包当前 BTree 视图
     │
     ▼
 startSchedule 的 goroutine 捞到 msgs
     │
     ▼
 dumpFunc(maxMsg):
   mp.store(msg)                           ← ③ 真正落盘
   mp.raftPartition.Truncate(curIndex)     ← ④ ★ 成功后截断 raft log
     │
     ▼
 一次 dump 完成
```

**4 个关键设计点**：

1. **dump 本身也要过 raft**：`opFSMStoreTick` 不是内存操作，而是一条真 raft 提案。**为什么？** 因为所有副本必须在**同一个 applyID 的 BTree 视图**上 dump，才能保证三份 snapshot 语义一致（便于互相接管）。
2. **dump 成功 → 立即 `Truncate(curIndex)`**：告诉 raft 可以丢掉 `<= curIndex` 的 log。这是 CubeFS 用"snapshot 代替日志"控制 raft log 大小的核心——否则 log 会无限增长。
3. **只有 leader 定时 tick**：`HandleLeaderChange` 切 leader 时才发 `startStoreTick`（Layer 4 讲过）。follower 只被动接 raft 同步。
4. **合并 msgs 只取 max applyID**：如果 dump goroutine 忙不过来攒了一批，挑 applyID 最大的那条做，中间的直接丢——反正"更新的 snapshot 覆盖更旧的"。

### 3.1 Cursor 的单独同步：为什么不能让 snapshot 自然承载

`timerCursor` 通道独立 submit `opFSMSyncCursor`，**周期短于 snapshot 周期**。原因：

- `nextInodeID` 是 **leader 本地 CAS**（Layer 5 讲过），follower 通过 `opFSMCreateInode` 被动推进 Cursor。
- 但如果 leader **分配了 inode id 却没 submit**（比如写请求失败在 submit 前面），那个 id 会永久跳过；follower 也永远追不上。
- → 定期 `opFSMSyncCursor` 作为"兜底心跳"，把 leader 最新的 Cursor 强推给所有副本和 snapshot。

---

## 4. 重启恢复：`mp.load` → `LoadSnapshot` → raft replay

[partition.go:1220 load](../../metanode/partition.go#L1220)：

```go
func (mp *metaPartition) load(isCreate bool) (err error) {
    mp.loadMetadata()                    // ① 读 meta 文件（PartitionId/Start/End/Peers）
    if isCreate { mp.storeSnapshotFiles(); return }

    snapshotPath := path.Join(mp.config.RootDir, snapshotDir)
    if _, err := os.Stat(snapshotPath); err != nil { return nil }  // 没 snapshot 就空启动
    return mp.LoadSnapshot(snapshotPath)
}
```

### 4.1 `LoadSnapshot`：并行加载各棵树

[partition.go:1122](../../metanode/partition.go#L1122)

```go
crcs, _ := mp.parseCrcFromFile()                 // 读 .sign
loadFuncs := []func(...) error{
    mp.loadInode, mp.loadDentry, nil, mp.loadMultipart,
    mp.loadTxInfo, mp.loadTxRbInode, mp.loadTxRbDentry,
    mp.loadUniqChecker,
}
// 兼容性：老版本 crc 少，按 crc count 决定加载多少

// ★ 并行！
var wg sync.WaitGroup
for idx, f := range loadFuncs {
    go func() {
        defer wg.Done()
        errs[i] = f(snapshotPath, crcs[i])
    }()
}
wg.Wait()
```

**3 个关键事实**：

1. **各树独立文件 → 可并行 load**，这是**为什么不把所有东西塞一个文件**的原因。对于大 mp（千万级 inode），load 是启动瓶颈，并行能显著缩短。
2. **`loadFuncs[2] = nil`（loadExtend）**：注释写得很清楚——quota 信息依赖 inode 已经 load 完，**所以 extend 不能并行**，得单独在 load 外面等 inode 完成后再 load。这是一个隐藏的依赖。
3. **crc 数量决定加载集合**：`CRC_COUNT_BASIC` / `CRC_COUNT_TX_STUFF` / `CRC_COUNT_UINQ_STUFF` / `CRC_COUNT_MULTI_VER`——这是**版本前向兼容**的处理方式：老版本 snapshot 没有 tx / uniq 字段，就少读几个文件。

### 4.2 `loadInode`：对称反序列化

[partition_store.go:160](../../metanode/partition_store.go#L160) 简化骨架：

```go
for {
    io.ReadFull(reader, inoBuf[:4])               // 4 字节长度
    length := binary.BigEndian.Uint32(inoBuf)
    limitReader.N = int64(length)
    ino := NewInode(0, 0)
    ino.UnmarshalFromReader(limitReader)          // ★ 直接 unmarshal

    crcCheck.Write(...)                           // 同步算 crc
    mp.inodeTree.ReplaceOrInsert(ino, true)       // 插入内存树
    mp.checkAndInsertFreeList(ino)                // ★★ 标记删除的 inode 重新进 freeList
}
if crcCheck.Sum32() != crc { return ErrSnapshotCrcMismatch }
```

**Layer 5 留的钩子答案**：**`ShouldDelete == true` 的 inode 不会被跳过，而是原样写进 snapshot + load 时再放回 freeList**——这样重启后后台 GC 继续之前的工作。

### 4.3 恢复流程的完整顺序

```
启动 → mp.load(false)
        ├─ loadMetadata()   ← 恢复 Cursor=Start 等静态配置
        ├─ LoadSnapshot()
        │    ├─ loadInode  ─┐
        │    ├─ loadDentry ─┤ 并行
        │    ├─ loadMultipart ...
        │    └─ loadApplyID   ← ★ 恢复 mp.applyID = 上次 store 时的 index
        │
        ▼
      onStart 后续：
        ├─ startRaft(isCreate=false)
        │    └─ pc.Applied = mp.applyID   ← ★ 告诉 raft"我已经吃到这里"
        │
        ▼
      raft 引擎自动从 (applyID+1) 开始 replay log
        └─ 对每条 log 回调 mp.Apply(cmd, idx)
                → fsmXxx 再走一次
                → 更新 inode/dentry tree
                → uploadApplyID 单调增
```

**正确性的基石**：
- **`pc.Applied = mp.applyID`**：让 raft 从正确的位置 replay，避免重复应用已在 snapshot 里的命令。
- **`mp.applyID` 来自 `loadApplyID`**：即落盘时 `sm.applyIndex` 对应的值。
- **三者必须 apply-order 单调一致**：store 写入时的 applyID、storedApplyId、load 时的 applyID——任何一处错位都会导致 **BTree 状态与 applyID 漂移**，引发副本状态分叉。

> ⚠️ **Layer 4 埋的钩子答案**：`applyID` 和 `storedApplyId` 的区别——前者是"内存里的 BTree 应用到了哪条"，后者是"磁盘 snapshot 应用到了哪条"。`applyID >= storedApplyId` 永远成立，差值就是 raft log 需要 replay 的量。

---

## 5. raft snapshot 传输：`Snapshot` 与 `ApplySnapshot`

当 follower 落后太多（leader 已经 truncate 掉它需要的 log），raft 引擎会触发**快照同步**。

### 5.1 `Snapshot()`：leader 侧生成迭代器

[partition_fsm.go:676](../../metanode/partition_fsm.go#L676)

```go
func (mp *metaPartition) Snapshot() (snap raftproto.Snapshot, err error) {
    snap, err = newMetaItemIterator(mp)
    return
}
```

就一行——返回一个 **`MetaItemIterator`**，它实现了 `Next() ([]byte, error)`，在 raft 层被**按需拉取**、编码成网络包发给 follower。

**`newMetaItemIterator` 内部**会按顺序生成：
1. `opFSMSnapFormatVersion` 头（格式版本号）
2. `opFSMApplyId` 条目
3. `opFSMCursor` / `opFSMTxId` / `opFSMUniqIDSnap` / `opFSMVerListSnapShot`
4. 遍历 inodeTree → 每个 inode 打包成 `opFSMCreateInode` 条目
5. 遍历 dentryTree → `opFSMCreateDentry`
6. extendTree / multipartTree / txTree / ...
7. 一些特殊文件（如 `extentDelete` 日志）走 `opExtentFileSnapshot`

**每个条目都是一个 `MetaItem{Op, K, V}`**——和 raft log 的格式完全一样！所以 follower 可以**复用解析代码**。

### 5.2 `ApplySnapshot()`：follower 侧重建

[partition_fsm.go:681](../../metanode/partition_fsm.go#L681)

核心是一个 `for iter.Next()` 大循环，`switch snap.Op`：

```go
for {
    data, err := iter.Next()
    snap.UnmarshalBinary(data)
    switch snap.Op {
    case opFSMApplyId:       appIndexID = ...
    case opFSMCursor:        cursor = ...
    case opFSMCreateInode:
        ino := NewInode(0, 0)
        ino.UnmarshalKey(snap.K); ino.UnmarshalValue(snap.V)
        inodeTree.ReplaceOrInsert(ino, true)   // ★ 插到临时 tree
    case opFSMCreateDentry:
        dentryTree.ReplaceOrInsert(...)
    case opFSMSetXAttr:       extendTree.ReplaceOrInsert(...)
    case opFSMCreateMultipart: ...
    case opFSMTxSnapshot:      ...
    ...
    }
}
```

**3 个关键机制**：

1. **先插到临时 tree，最后一把 swap**：
   ```go
   defer func() {
       if err == io.EOF {
           mp.applyID = appIndexID
           mp.inodeTree = inodeTree       // ★ 一次性替换
           mp.dentryTree = dentryTree
           ...
       }
   }()
   ```
   这避免"半替换状态"——替换失败时原 tree 还在。
2. **替换完成后往 `storeChan` 发一条 `opFSMStoreTick`**：立刻落盘一份本地 snapshot，然后 `blockUntilStoreSnapshot` 阻塞到 `storedApplyId >= appIndexID`。**为什么？** 防止 ApplySnapshot 刚完成 follower 就收到新的 log apply，而本地 snapshot 还是旧的 → 崩溃后恢复会对不上。
3. **`leaderSnapFormatVer` 前向兼容**：如果 leader 发来的格式号比自己高，未知 Op **跳过而非报错**（向前兼容升级），但低于自己就硬报错（老 leader 给不出新格式需要的数据）。

### 5.3 本地 snapshot vs raft snapshot：对比表

| 维度 | 本地 snapshot（`store` / `LoadSnapshot`） | raft snapshot（`Snapshot` / `ApplySnapshot`） |
|---|---|---|
| 触发者 | 定时 tick（leader 周期性） | raft 引擎（follower 落后太多） |
| 输出形式 | 本地目录下多个 文件 + crc | 字节流，按 `MetaItem` 逐条 |
| 读者 | 本机进程自己 load | 远端 follower |
| 按类型分文件 | ✅（可并行 load） | ❌（单一条目流） |
| 增量 vs 全量 | 全量 | 全量 |
| 是否含 applyID | 单独文件 `apply` | 以 `opFSMApplyId` 条目嵌入 |
| 写完后的 cleanup | truncate raft log 到 storeApplyId | 本地再做一次 snapshot（对齐） |

**一个重要的共享点**：**Apply 函数里处理 `opFSMCreateInode` 这些 op 的代码，既用于 raft log replay，也用于 raft snapshot 应用**——所以逻辑永远一致。这就是为什么 `ApplySnapshot` 里每个 case 的处理和 `Apply` 里的处理**几乎对称**。

---

## 6. 关键不变式总结

这是一张表——**每一行都是一个副本语义正确性的前置条件**，任何一行破了都会导致三副本状态分叉。

| 不变式 | 谁维护 |
|---|---|
| `storedApplyId <= applyID` | `store` 成功后才 `storedApplyId = sm.applyIndex` |
| snapshot 里 `apply` 文件的值 = snapshot 本身对应的 applyID | `storeApplyID(tmpDir, sm)` 与其他 `storeXxx` 用同一个 `sm` |
| raft log 只有 `<= storedApplyId` 的部分可被 truncate | `dumpFunc` 中 `raftPartition.Truncate(curIndex)` 在 `store` 成功后才调 |
| 启动时 `pc.Applied = loaded applyID` | `startRaft` 里 `Applied: mp.applyID` |
| 同一份 `storeMsg` 内的 inodeTree / dentryTree / ... 是同一时刻视图 | Apply 里一次性拷贝 BTree 引用放进 `sm` |
| 快照格式跨版本向前兼容 | `SnapFormatVersion` + 未知 op 的 skip 分支 |
| `ShouldDelete == true` 的 inode 必须进 snapshot + load 后回 freeList | `storeInode` 不过滤 + `loadInode` 里 `checkAndInsertFreeList` |
| `Cursor` 单调不减（选主切换也不能回退） | `opFSMCreateInode` Apply 里 `if Cursor < ino.Inode { Cursor = ino.Inode }` + `opFSMSyncCursor` 周期同步 |

---

## 7. 本层关键问题自测

1. ✅ 为什么本地 snapshot 要 `.snapshot → snapshot → .snapshot_backup` 三步 rename？中间任何一步崩溃会怎样？
2. ✅ `store` 时 BTree 是怎么做到"边 dump 边服务写请求"的？
3. ✅ 为什么 `opFSMStoreTick` 要走 raft 而不是 leader 本地直接 dump？
4. ✅ `storedApplyId` 和 `applyID` 的关系？谁先更新？谁用于 raft log truncate？
5. ✅ `LoadSnapshot` 为什么把各个 load 函数丢 goroutine 并行？`loadExtend` 为什么被排除？
6. ✅ 重启时 raft replay 是从哪个 index 开始的？怎么保证不和 snapshot 的内容重复？
7. ✅ `ShouldDelete` 的 inode 为什么要进 snapshot？不进会怎样？
8. ✅ `Snapshot()` 返回的 iterator 和本地 `storeInode` 写的文件格式，哪个更"紧凑"？
9. ✅ `ApplySnapshot` 里为什么先写临时 tree 最后 swap？直接往 `mp.inodeTree` 插会怎样？
10. ✅ `ApplySnapshot` 结尾为什么要 `blockUntilStoreSnapshot`？省掉这个 block 会怎样？
11. ✅ `Cursor` 为什么除了周期 `opFSMSyncCursor` 还要嵌入每条 `opFSMCreateInode`？两者可以只留一个吗？
12. ✅ 老版本 snapshot（crc 数量少）怎么被新版 MetaNode 识别？新版 snapshot 能被老版 MetaNode 读吗？

---

## 8. MetaNode 七层完结 · 回顾框架

到这里 MetaNode 七层已经完整串通：

```
Layer 1 启动与配置    ← server.go / raft_server.go / metadataManager
Layer 2 网络层        ← TCP listener / conn pool
Layer 3 请求分发      ← wrap_*.go / manager_op.go
Layer 4 MP 与 Raft    ← partition.go / partition_fsm.go （submit / Apply）
Layer 5 Inode 子系统  ← inode.go / partition_fsmop_inode.go  (fsmCreateInode...)
Layer 6 Dentry 子系统 ← dentry.go / partition_fsmop_dentry.go (fsmCreateDentry...)
Layer 7 持久化        ← partition_store.go / partition_store_ticket.go / partition_fsm.go (store/load/Snapshot)
```

**最重要的收获是这套"七层阅读框架"本身**——它是后续读 DataNode / Master / BlobStore 时的**第一把钥匙**。每个新模块都可以套进去问：
1. 启动流程是什么？配置怎么解析？
2. 网络层监听什么端口？协议是什么？
3. 请求怎么分发到具体处理函数？
4. 写路径走什么一致性协议（raft？主备？）？状态机的接口是什么？
5. 核心数据结构（inode 的等价物）是怎么组织的？
6. 关联数据结构（dentry 的等价物）呢？
7. 持久化怎么做？snapshot 怎么恢复？跨节点同步走什么路径？

**下一步**：回到 `note/学习计划.md`，按阶段 2 开始读 Master，带着这套问题。

---

## 9. 遗留存疑点（Layer 4~7 累积）

这些问题不影响继续学习 Master，但值得回头再研究：

1. **快照与 extent 删除日志的交互**：`opExtentFileSnapshot` 是把本地文件直接塞进 raft snapshot——这个机制怎么保证幂等？重复 ApplySnapshot 会不会重复删除？
2. **`uniqChecker` 的时间窗口**：重启后幂等窗口是直接清零，还是从 snapshot 恢复？如果清零，会不会在"重启瞬间"出现同一请求被重复执行？
3. **`multiVersionList`（多版本快照）** 跨机器传输是走 raft snapshot 还是单独通道？
4. **跨 mp 分布式事务的持久化**：`txInfo` / `txRbInode` / `txRbDentry` 的 rollback 记录在 snapshot 里。如果 leader 崩溃在 Phase2 之间，新 leader 从 snapshot 恢复后怎么判断 tx 应该 commit 还是 abort？
5. **`loadExtend` 的串行依赖**：有没有办法在并行架构里消除这个特殊情况？或者它暗示了 xattr / quota 数据模型上的某个设计妥协？
6. **snapshot 与 raft log truncate 的时机**：如果 leader 发生频繁选主切换，follower 可能一直跟不上，最后触发 raft snapshot 同步——这种抖动场景下性能怎么保证？
