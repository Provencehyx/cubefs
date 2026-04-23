# Layer 5: 日志管理

## 1. raftLog 结构

raftLog 负责管理 Raft 日志，包括内存中的 unstable 和持久化的 storage。

```go
// raft_log.go:31
type raftLog struct {
    unstable  unstable          // 内存中未持久化的日志
    storage   storage.Storage   // 持久化存储
    committed uint64            // 已提交索引
    applied   uint64            // 已应用索引
}
```

## 2. 日志结构图

```
┌─────────────────────────────────────────────────────────────┐
│                      raftLog 结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  索引:    1    2    3    4    5    6    7    8    9        │
│         ┌────┬────┬────┬────┬────┬────┬────┬────┬────┐     │
│  Entry: │ E1 │ E2 │ E3 │ E4 │ E5 │ E6 │ E7 │ E8 │ E9 │     │
│         └────┴────┴────┴────┴────┴────┴────┴────┴────┘     │
│                                                             │
│         │◄──── storage ─────►│◄──── unstable ────►│        │
│         │    (已持久化)       │    (内存中)         │        │
│                                                             │
│                     ▲              ▲                        │
│                     │              │                        │
│                  applied       committed                    │
│                    (3)           (6)                        │
│                                                             │
│  applied:   已应用到状态机                                   │
│  committed: 已被多数节点确认                                 │
│  unstable:  Leader 追加但未持久化                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. Entry 结构

```go
// proto/proto.go
type Entry struct {
    Type  EntryType   // Normal / ConfChange
    Term  uint64      // 任期
    Index uint64      // 索引
    Data  []byte      // 数据
}

type EntryType int
const (
    EntryNormal     EntryType = 0   // 普通日志
    EntryConfChange EntryType = 1   // 配置变更
)
```

## 4. unstable 结构

```go
// raft_log_unstable.go
type unstable struct {
    snapshot *proto.SnapshotMeta  // 快照元数据
    entries  []*proto.Entry       // 未持久化日志
    offset   uint64               // 第一条 entry 的索引
}
```

## 5. 创建 raftLog

```go
// raft_log.go:37
func newRaftLog(storage storage.Storage) (*raftLog, error) {
    log := &raftLog{
        storage: storage,
    }
    
    // 从 storage 获取索引范围
    firstIndex, _ := storage.FirstIndex()
    lastIndex, _ := storage.LastIndex()

    // 初始化 unstable
    log.unstable.offset = lastIndex + 1
    log.unstable.entries = make([]*proto.Entry, 0, 256)
    
    // 初始化已提交/已应用
    log.committed = firstIndex - 1
    log.applied = firstIndex - 1
    
    return log, nil
}
```

## 6. 追加日志

```go
// raft_log.go:167
func (l *raftLog) append(ents ...*proto.Entry) uint64 {
    if len(ents) == 0 {
        return l.lastIndex()
    }
    
    // 检查不能覆盖已提交的
    if after := ents[0].Index - 1; after < l.committed {
        panic("after is out of range [committed]")
    }
    
    // 追加到 unstable
    l.unstable.truncateAndAppend(ents)
    
    return l.lastIndex()
}
```

## 7. maybeAppend (Follower 处理追加)

```go
// raft_log.go:147
func (l *raftLog) maybeAppend(index, logTerm, committed uint64, ents ...*proto.Entry) (lastnewi uint64, ok bool) {
    // 1. 检查 prevLogIndex 和 prevLogTerm 是否匹配
    if l.matchTerm(index, logTerm) {
        lastnewi = index + uint64(len(ents))
        
        // 2. 找冲突点
        ci := l.findConflict(ents)
        
        switch {
        case ci == 0:
            // 无冲突
        case ci <= l.committed:
            // 冲突在已提交区域，panic
            panic("entry conflict with committed entry")
        default:
            // 从冲突点开始追加
            l.append(ents[ci-(index+1):]...)
        }
        
        // 3. 更新 committed
        l.commitTo(util.Min(committed, lastnewi))
        return lastnewi, true
    }
    return 0, false
}
```

## 8. 提交日志

```go
// raft_log.go:209
func (l *raftLog) maybeCommit(maxIndex, term uint64) bool {
    // 只有当前 term 的日志才能通过计数提交
    if maxIndex > l.committed && l.term(maxIndex) == term {
        l.commitTo(maxIndex)
        return true
    }
    return false
}

// raft_log.go:217
func (l *raftLog) commitTo(tocommit uint64) {
    if l.committed < tocommit {
        if l.lastIndex() < tocommit {
            panic("tocommit is out of range")
        }
        l.committed = tocommit
    }
}
```

## 9. 获取待应用日志

```go
// raft_log.go:187
func (l *raftLog) nextEnts(maxSize uint64) (ents []*proto.Entry) {
    // 从 applied+1 到 committed+1
    off := util.Max(l.applied+1, l.firstIndex())
    hi := l.committed + 1
    
    if hi > off {
        ents, _ = l.slice(off, hi, maxSize)
        return ents
    }
    return nil
}

// 标记已应用
func (l *raftLog) appliedTo(i uint64) {
    if i > l.applied {
        l.applied = i
    }
}
```

## 10. 日志查询

```go
// 获取第一条索引
func (l *raftLog) firstIndex() uint64 {
    index, _ := l.storage.FirstIndex()
    return index
}

// 获取最后一条索引
func (l *raftLog) lastIndex() uint64 {
    // 先查 unstable
    if i, ok := l.unstable.maybeLastIndex(); ok {
        return i
    }
    // 再查 storage
    i, _ := l.storage.LastIndex()
    return i
}

// 获取指定索引的 term
func (l *raftLog) term(i uint64) (uint64, error) {
    // 先查 unstable
    if t, ok := l.unstable.maybeTerm(i); ok {
        return t, nil
    }
    // 再查 storage
    return l.storage.Term(i)
}
```

## 11. 日志持久化流程

```
┌─────────────────────────────────────────────────────────────┐
│                    日志持久化流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Leader 收到提案                                         │
│     └─► raftLog.append() → 追加到 unstable                 │
│                                                             │
│  2. 发送给 Follower                                         │
│     └─► raftFsm.bcastAppend()                              │
│                                                             │
│  3. 收到多数响应                                             │
│     └─► raftLog.maybeCommit() → 更新 committed             │
│                                                             │
│  4. 持久化                                                  │
│     └─► storage.StoreEntries(unstable.entries)             │
│     └─► unstable.stableTo(index)                           │
│                                                             │
│  5. 应用                                                    │
│     └─► raftLog.nextEnts() → 获取待应用日志                 │
│     └─► StateMachine.Apply()                               │
│     └─► raftLog.appliedTo(index)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 12. Storage 接口

```go
// storage/storage.go
type Storage interface {
    // 索引范围
    FirstIndex() (uint64, error)
    LastIndex() (uint64, error)
    
    // 获取 term
    Term(i uint64) (term uint64, isCompact bool, err error)
    
    // 获取日志条目
    Entries(lo, hi uint64, maxSize uint64) ([]*proto.Entry, bool, error)
    
    // 存储日志
    StoreEntries(entries []*proto.Entry) error
    
    // 截断日志
    Truncate(index uint64) error
    
    // 快照
    ApplySnapshot(meta proto.SnapshotMeta) error
    
    // 初始状态
    InitialState() (proto.HardState, error)
    
    // 关闭
    Close()
}
```

## 13. 核心方法

raftLog 的主要方法 (raft_log.go):

### 13.1 索引查询

| 方法 | 位置 | 说明 |
|------|------|------|
| `firstIndex() uint64` | :61 | 第一条日志索引 |
| `lastIndex() uint64` | :71 | 最后一条日志索引 |
| `lastIndexAndTerm() (uint64, uint64)` | :116 | 最后索引和 term |
| `term(i uint64) (uint64, error)` | :84 | 获取指定索引的 term |
| `lastTerm() uint64` | :106 | 最后一条的 term |

### 13.2 日志操作

| 方法 | 位置 | 说明 |
|------|------|------|
| `append(ents ...*proto.Entry) uint64` | :167 | 追加日志 |
| `maybeAppend(index, logTerm, committed uint64, ents ...*proto.Entry) (uint64, bool)` | :147 | Follower 追加 |
| `entries(i uint64, maxsize uint64) ([]*proto.Entry, error)` | :202 | 获取日志条目 |
| `slice(lo, hi uint64, maxSize uint64) ([]*proto.Entry, error)` | :256 | 切片获取 |
| `allEntries() []*proto.Entry` | :330 | 获取所有日志 |
| `unstableEntries() []*proto.Entry` | :180 | 获取未持久化日志 |

### 13.3 提交与应用

| 方法 | 位置 | 说明 |
|------|------|------|
| `maybeCommit(maxIndex, term uint64) bool` | :209 | 尝试提交 |
| `commitTo(tocommit uint64)` | :217 | 提交到指定索引 |
| `nextEnts(maxSize uint64) []*proto.Entry` | :187 | 获取待应用日志 |
| `appliedTo(i uint64)` | :228 | 标记已应用 |
| `stableTo(i, t uint64)` | :240 | 标记已持久化 |

### 13.4 日志匹配

| 方法 | 位置 | 说明 |
|------|------|------|
| `matchTerm(i, term uint64) bool` | :127 | 检查 term 匹配 |
| `findConflict(ents []*proto.Entry) uint64` | :135 | 查找冲突点 |
| `findConflictByTerm(index, term uint64) uint64` | :366 | 按 term 查找冲突 |
| `isUpToDate(lasti, term uint64, fpri, lpri uint16) bool` | :242 | 检查是否足够新 |

### 13.5 快照恢复

| 方法 | 位置 | 说明 |
|------|------|------|
| `restore(index uint64)` | :247 | 从快照恢复 |

---
*创建时间: 2026-04-22*
