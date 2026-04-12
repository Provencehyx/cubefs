# MetaNode 七层解析 · Layer 6：Dentry 子系统

> 入口文件：[metanode/dentry.go](../../metanode/dentry.go) · [metanode/partition_fsmop_dentry.go](../../metanode/partition_fsmop_dentry.go) · [metanode/partition_op_dentry.go](../../metanode/partition_op_dentry.go)
> 核心问题：**目录项（"名字 → inode"的绑定关系）怎么存？`(ParentId, Name)` 怎么组织成 BTree？rename 跨 mp 怎么保证原子？parent inode 的 NLink 什么时候该加、什么时候该减？**

> 💡 Layer 5 留下的钩子：`fsmCreateDentry` / `fsmDeleteDentry` / `fsmUpdateDentry` 是怎么改 `dentryTree` 的？它们和 inode 子系统的联动点在哪儿？

---

## 1. 这一层在系统中的位置

```
   Layer 4 (Apply)
        │
        │  opFSMCreateDentry / opFSMDeleteDentry / opFSMUpdateDentry / opFSMTxXxxDentry
        ▼
┌────────────────────────────────────────────────┐
│                Layer 6 本层                     │
│                                                │
│   mp.dentryTree  (*BTree of *Dentry)           │
│        key = (ParentId, Name) 字典序             │
│                                                │
│   fsmCreateDentry  ──► 插入新 dentry             │
│                        + parIno.IncNLink        │
│   fsmDeleteDentry  ──► 删 dentry                │
│                        + parIno.DecNLink        │
│   fsmUpdateDentry  ──► 原子 swap inode 字段      │
│                        （rename 同目录场景）     │
│                                                │
│   fsmTxCreateDentry / fsmTxDeleteDentry ...    │
│        └─► 跨 mp 分布式事务                      │
└────────────────────────────────────────────────┘
```

**记住一句话**：**dentry 不是 inode 的附属字段，而是独立的 BTree**。它和 `inodeTree` 通过 `(ParentId, Inode)` 这两个 id 产生联动——改 dentry 常常要连带改 parent inode 的 `NLink`。

---

## 2. `Dentry` 结构体

[dentry.go:53](../../metanode/dentry.go#L53)

```go
type Dentry struct {
    ParentId uint64    // 父目录 inode id
    Inode    uint64    // 本 dentry 指向的 inode id
    Name     string    // 文件/子目录名
    Type     uint32    // 文件类型（给 readdir 用，省一次 inode 查找）

    multiSnap *DentryMultiSnap   // 快照链
}
```

**4 个字段的语义分工**：

| 字段 | 在 key 里还是 value 里 | 作用 |
|---|---|---|
| `ParentId` | key | 聚类：同一目录的 dentry 相邻 |
| `Name` | key | 同目录内唯一性 |
| `Inode` | value | 指向目标 inode（硬链接让多个 dentry 指同一个 inode） |
| `Type` | value | readdir 优化：不用为每个 child 查 inodeTree |

### 2.1 BTree 的 key：`Less` 函数

[dentry.go:698](../../metanode/dentry.go#L698)

```go
func (d *Dentry) Less(than BtreeItem) (less bool) {
    dentry, ok := than.(*Dentry)
    less = ok && ((d.ParentId < dentry.ParentId) ||
                  ((d.ParentId == dentry.ParentId) && (d.Name < dentry.Name)))
    return
}
```

**这一行决定了 dentryTree 的全部物理结构**：
- 主键：`ParentId` 升序 → 同一目录的所有 dentry **物理相邻**
- 次键：`Name` 升序（按字节）→ readdir 天然有序

**最关键的推论**：**readdir 就是 `AscendRange(ParentId, ParentId+1, ...)`**。看 [partition_fsmop_dentry.go:412](../../metanode/partition_fsmop_dentry.go#L412)：

```go
begDentry := &Dentry{ParentId: req.ParentID}
endDentry := &Dentry{ParentId: req.ParentID + 1}
mp.dentryTree.AscendRange(begDentry, endDentry, func(i BtreeItem) bool { ... })
```

→ 因为 key 设计让同目录 dentry 连续，**一次 range 扫描就能列出整个目录**。这是 CubeFS 元数据性能的一个底层红利。

### 2.2 Marshal 格式

[dentry.go](../../metanode/dentry.go) 顶部注释：

```
key:   +----------+------+       value: +-------+------+
       | ParentId | Name |              | Inode | Type |
       +----------+------+              +-------+------+
       |    8     | rest |              |   8   |   4  |
```

**注意 Name 是变长的**——所以 key marshal 后长度不定，BTree 比较时只能靠 `Less` 函数而非定长 memcmp。

---

## 3. `fsmCreateDentry`：插入 + parent link++

[partition_fsmop_dentry.go:78](../../metanode/partition_fsmop_dentry.go#L78)

核心骨架（去掉快照分支）：

```go
func (mp *metaPartition) fsmCreateDentry(dentry *Dentry, forceUpdate bool) (status uint8) {
    // ① 校验 parent inode 存在且是目录
    if !forceUpdate {
        parIno := mp.inodeTree.CopyGet(NewInode(dentry.ParentId, 0)).(*Inode)
        if parIno == nil || parIno.ShouldDelete() { return OpNotExistErr }
        if !proto.IsDir(parIno.Type) { return OpArgMismatchErr }
    }

    // ② 插入 dentry tree
    if item, ok := mp.dentryTree.ReplaceOrInsert(dentry, false); !ok {
        // 已存在，走冲突处理分支（通常返回 OpExistErr）
        ...
        return proto.OpExistErr
    }

    // ③ parent inode NLink++, mtime 更新
    if !forceUpdate {
        parIno.IncNLink(mp.verSeq)
        parIno.SetMtime()
    }
    return
}
```

**3 个关键设计点**：

1. **parent inode 的存在性校验在 insert 之前**：如果 parent 已经 `ShouldDelete`，就不能挂新的 dentry（避免"父不存在的孤儿 dentry"）。
2. **`ReplaceOrInsert(dentry, false)` 第二参数 false**：与 inode 一样，表示"已存在就冲突"，同目录不允许重名。
3. **`IncNLink` 是直接改指针**：`parIno` 是 `CopyGet` 拿到的 BTree 内对象，这里**直接修改字段**。这合法的前提是 Apply 全局串行（`nonIdempotent` 锁）——没有并发写。
4. **`forceUpdate` 分支跳过 parent link 维护**：用于特殊场景（如 rename 的 "remove-old + insert-new"），让调用方手动管理 NLink，避免重复计数。

**联动链**：
```
fsmCreateDentry(d)
   ├─► dentryTree.ReplaceOrInsert(d)
   └─► inodeTree 里的 parIno.NLink++ ← 改的是 Layer 5 的 inode 字段！
```

这是 Layer 5 和 Layer 6 的第一个联动点。

---

## 4. `fsmDeleteDentry`：删除 + parent link--

[partition_fsmop_dentry.go:230](../../metanode/partition_fsmop_dentry.go#L230)

去掉快照 / checkInode 分支后的核心：

```go
// ① 从 dentryTree 删除
item := mp.dentryTree.Delete(denParm)
denFound := item.(*Dentry)

// ② 找到 parent inode，NLink--
mp.inodeTree.CopyFind(NewInode(denParm.ParentId, 0), func(item BtreeItem) {
    if item != nil {
        ino := item.(*Inode)
        if !ino.ShouldDelete() {
            if denParm.getSeqFiled() == 0 {
                ino.DecNLink()           // ★ 非快照路径才减
            }
            ino.SetMtime()
        }
    }
})

resp.Msg = denFound
return
```

**3 个细节**：

1. **`CopyFind` 是带回调的"读改一体"**：相当于 `item = Get(); callback(item)`，避免两次锁开销。
2. **快照路径不减 NLink**：`denParm.getSeqFiled() != 0` 时是在删某个快照版本，不影响实时的 NLink。
3. **删除是同步的**——不像 inode 走 freeList。**为什么？** 因为 dentry 是"纯元数据"，没有外部资源（extent / 磁盘块）需要延迟回收。

**`checkInode` 参数**：如果传 true，删除前会比对 `d.Inode == denParm.Inode`——用于 **rename 期间的 ABA 防护**：避免 A rename 了一半，B 又把同名文件塞回来，rename 回滚时错删了 B。

---

## 5. `fsmUpdateDentry`：同目录 rename 的捷径

[partition_fsmop_dentry.go:369](../../metanode/partition_fsmop_dentry.go#L369)

```go
mp.dentryTree.CopyFind(dentry, func(item BtreeItem) {
    d := item.(*Dentry)
    if dentry.Inode == d.Inode { return }    // 没变化
    ...
    d.Inode, dentry.Inode = dentry.Inode, d.Inode   // ★ swap
    resp.Msg = dentry
})
```

**语义**：**原地把同一个 `(ParentId, Name)` 的 dentry 绑定到新的 inode**，返回值是被覆盖的旧 inode（让调用方去减引用、清理 extent）。

**这个操作存在的意义**：
- 覆盖写场景 `mv a b`（b 已存在且同目录）：不需要删 + 插，直接原子 swap 指向。
- 避免短暂的"dentry 不存在"窗口，让并发读不会看到空洞。

---

## 6. rename：为什么需要分布式事务

**直觉**：`rename(src_parent, src_name → dst_parent, dst_name)` 看起来就是 "delete(src) + create(dst)"。

**问题**：如果 `src_parent` 和 `dst_parent` 属于**同一个 mp**，那确实是同一次 raft Apply 里改两棵 BTree，原子；但如果**跨 mp**（不同 PartitionId）呢？两个 mp 是两个 raft group，**没有共享的 Apply 锁**——需要真正的分布式事务。

CubeFS 的解法是 `txProcessor`（见 [partition.go](../../metanode/partition.go) 里的 `txProcessor *TransactionProcessor`）：

```
Client rename
   │
   ▼
Coordinator (通常是 src_parent 所在 mp)
   │
   ├─► 2PC Phase1: 在 src_mp 注册 TxRollbackDentry(src, TxAdd)
   │                在 dst_mp 注册 TxRollbackDentry(dst, TxDelete)
   │                在 src_ino_mp 注册 TxRollbackInode(ino, TxDelete)
   │
   ├─► 2PC Phase2 (commit):
   │     src_mp:     fsmTxDeleteDentry   ← 真正删 src dentry
   │     dst_mp:     fsmTxCreateDentry   ← 真正插 dst dentry
   │     src_ino_mp: fsmTxCreateInode    ← inode 的 parent 计数变化
   │
   └─► Phase2 (abort): 按 Rollback 记录逆向恢复
```

看 [partition_fsmop_dentry.go:36](../../metanode/partition_fsmop_dentry.go#L36) `fsmTxCreateDentry`：

```go
// ① 幂等检查
if mp.txProcessor.txManager.txInRMDone(txDentry.TxInfo.TxID) { return OpTxInfoNotExistErr }

// ② 注册 rollback 记录
rbDentry := NewTxRollbackDentry(txDentry.Dentry, txDenInfo, TxDelete)
mp.txProcessor.txResource.addTxRollbackDentry(rbDentry)

// ③ 真正创建（和普通 fsmCreateDentry 共用）
return mp.fsmCreateDentry(txDentry.Dentry, false)
```

**tx 版本与非 tx 版本的区别**：多了 step ①（幂等 / 重复拦截）和 step ②（rollback 记录注册），最后 step ③ 还是调**同一个 `fsmCreateDentry`**。这个"薄皮"设计让 tx 和非 tx 能共享底层 BTree 操作，避免逻辑分裂。

**`txInRMDone` 的作用**：避免 2PC Phase2 的 commit 重试（比如 coordinator 网络重试）把同一笔事务执行两次——**幂等是 2PC 的命脉**。

### 6.1 rename 时的 inode 副作用

看 [partition_fsmop_inode.go:320](../../metanode/partition_fsmop_inode.go#L320)（Layer 5 跳过的代码）：

```go
if txIno.TxInfo.TxType == proto.TxTypeRename {
    mp.fsmEvictInode(txIno.Inode)
}
```

**含义**：rename 到 dst 时如果 dst 原本已存在同名文件，那个**旧 inode 会被 evict 进 freeList**——即"覆盖 rename" 语义。这又是一个 Layer 5 和 Layer 6 的联动点。

---

## 7. Layer 5 ↔ Layer 6 联动点汇总

| 操作 | 改 inodeTree | 改 dentryTree | 备注 |
|---|---|---|---|
| `mkdir` | CreateInode(dir) + parent.NLink++ | CreateDentry | parent NLink++ 因为目录有 ".." 反向计数 |
| `create file` | CreateInode(file) | CreateDentry | file 的 NLink 初始为 1 |
| `link()` | CreateLinkInode（NLink++） | CreateDentry | 硬链接：一个 inode 对应多 dentry |
| `unlink file` | UnlinkInode（NLink--） | DeleteDentry（parent.NLink--） | 最后一个 dentry 消失 → inode 进 freeList |
| `rmdir` | UnlinkInode（NLink--） | DeleteDentry（parent.NLink--） | 要求目录内无 dentry |
| `rename` (同 mp) | 可能 EvictInode(覆盖目标) | DeleteDentry + CreateDentry | 同 Apply 原子 |
| `rename` (跨 mp) | 同上 + Tx | 同上 + Tx | 走 `fsmTxXxx` |

> 💡 **NLink 记账的直觉模型**：inode 的 NLink = "有多少 dentry 指着我" + "如果是目录，还要加上 `.`"。每次 dentry 增删都要同步这个计数。

---

## 8. 本层关键问题自测

1. ✅ dentry 的 BTree key 是什么？为什么选这个顺序？
2. ✅ readdir 为什么不需要扫整棵树？靠什么实现 O(目录内 dentry 数)？
3. ✅ `fsmCreateDentry` 里 `forceUpdate == true` 的场景是什么？为什么要跳过 parent NLink 维护？
4. ✅ `fsmDeleteDentry` 的 `checkInode` 参数防的是什么情景？
5. ✅ `fsmUpdateDentry` 相比 "delete + create" 组合的优势是什么？
6. ✅ 同 mp 内的 rename 需要走分布式事务吗？跨 mp 呢？为什么？
7. ✅ `fsmTxCreateDentry` 和 `fsmCreateDentry` 的关系？tx 版本额外做了什么？
8. ✅ 为什么 `txInRMDone` 幂等检查对 2PC 是必需的？
9. ✅ rename 覆盖已存在目标时，目标 inode 会走哪条路径被清理？
10. ✅ dentry 删除为什么是同步的而 inode 删除要走 freeList？

---

## 9. 给后续 Layer 的钩子

- **Layer 7 (持久化)**：`dentryTree` 的 snapshot 格式就是 `(MarshalKey, MarshalValue)` 流——和 `inodeTree` 几乎对称。关键是**两棵树的 snapshot 必须同一时刻的视图**（否则恢复后 dentry 指向的 inode 不存在），这就要求 snapshot 必须在 `nonIdempotent` 锁保护下启动（或用一致性快照）。
- **分布式事务的持久化**：`txProcessor` 的 rollback 记录、tx 状态也需要落盘——**如果 mp 重启时一个 tx 在 Phase1/Phase2 之间，应该怎么恢复？** 这是 Layer 7 会回答的问题。
- **存疑点**：
  - 跨 mp rename 的 coordinator 如果崩了，这个 tx 会无限 pending 吗？`txManager` 是否有超时清理？
  - `DentryMultiSnap` 的快照链表增长上限是多少？会不会把 dentry 撑爆？
  - 目录非常大（百万级子项）时，`AscendRange` 的 readdir 是流式返回还是一次性 dump？
