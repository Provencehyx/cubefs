# Layer 7: WAL 存储

## 1. Storage 接口

```go
// storage/storage.go:25
type Storage interface {
    // 初始状态
    InitialState() (proto.HardState, error)
    
    // 获取日志条目 [lo, hi)
    Entries(lo, hi uint64, maxSize uint64) ([]*proto.Entry, bool, error)
    
    // 获取 term
    Term(i uint64) (term uint64, isCompact bool, err error)
    
    // 索引范围
    FirstIndex() (uint64, error)
    LastIndex() (uint64, error)
    
    // 存储日志
    StoreEntries(entries []*proto.Entry) error
    
    // 存储硬状态
    StoreHardState(st proto.HardState) error
    
    // 截断日志
    Truncate(index uint64) error
    
    // 应用快照
    ApplySnapshot(meta proto.SnapshotMeta) error
    
    // 关闭
    Close()
}
```

## 2. WAL Storage 结构

```go
// storage/wal/storage.go:28
type Storage struct {
    c *Config
    
    // 日志存储
    ls         *logEntryStorage
    truncIndex uint64           // 截断索引
    truncTerm  uint64           // 截断 term
    
    // 硬状态
    hardState  proto.HardState
    metafile   *metaFile
    prevCommit uint64
    
    closed bool
}
```

## 3. 文件布局

```
┌─────────────────────────────────────────────────────────────┐
│                      WAL 目录结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /raft/partition_100/                                       │
│  ├── meta                    # 元数据文件                    │
│  │   └── HardState + TruncateMeta                          │
│  ├── 0000000000000001.log    # 日志文件 1                   │
│  ├── 0000000000001000.log    # 日志文件 2                   │
│  └── 0000000000002000.log    # 日志文件 3 (当前写入)         │
│                                                             │
│  文件名 = 起始 Index (16位十六进制)                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. HardState

```go
// proto/proto.go
type HardState struct {
    Term   uint64   // 当前任期
    Vote   uint64   // 投票给谁
    Commit uint64   // 已提交索引
}
```

## 5. logEntryStorage 结构

```go
// storage/wal/log_storage.go:29
type logEntryStorage struct {
    s *Storage
    
    dir         string
    filesize    int             // 单文件大小限制
    rotateTime  int64           // 轮转时间
    logfiles    []logFileName   // 所有日志文件
    last        *logEntryFile   // 当前写入文件
    nextFileSeq uint64          // 下一个文件序号
    
    cache *logFileCache         // 文件缓存
}
```

## 6. 创建 Storage

```go
// storage/wal/storage.go:44
func NewStorage(dir string, c *Config) (*Storage, error) {
    // 1. 初始化目录
    initDir(dir)

    // 2. 加载元数据（HardState + TruncateMeta）
    mf, hardState, meta, _ := openMetaFile(dir)

    s := &Storage{
        c:          c.dup(),
        truncIndex: meta.truncIndex,
        truncTerm:  meta.truncTerm,
        hardState:  hardState,
        metafile:   mf,
        prevCommit: hardState.Commit,
    }

    // 3. 加载日志文件
    ls, _ := openLogStorage(dir, s)
    s.ls = ls

    return s, nil
}
```

## 7. 存储日志

```go
// 存储日志条目
func (s *Storage) StoreEntries(entries []*proto.Entry) error {
    if len(entries) == 0 {
        return nil
    }
    return s.ls.saveEntries(entries)
}

// logEntryStorage.saveEntries
func (ls *logEntryStorage) saveEntries(entries []*proto.Entry) error {
    for _, entry := range entries {
        // 检查是否需要轮转文件
        if ls.shouldRotate() {
            ls.rotate(entry.Index)
        }
        
        // 写入当前文件
        ls.last.Append(entry)
    }
    
    // Sync
    return ls.last.Sync()
}
```

## 8. 读取日志

```go
// 读取日志条目 [lo, hi)
func (s *Storage) Entries(lo, hi uint64, maxSize uint64) ([]*proto.Entry, bool, error) {
    // 检查是否已压缩
    if lo <= s.truncIndex {
        return nil, true, nil
    }
    
    return s.ls.entries(lo, hi, maxSize)
}

// logEntryStorage.entries
func (ls *logEntryStorage) entries(lo, hi uint64, maxSize uint64) ([]*proto.Entry, bool, error) {
    var entries []*proto.Entry
    var size uint64
    
    for i := lo; i < hi; i++ {
        // 找到包含该索引的文件
        file := ls.findFile(i)
        
        // 读取 entry
        entry, _ := file.Get(i)
        entries = append(entries, entry)
        
        size += uint64(entry.Size())
        if size >= maxSize {
            break
        }
    }
    
    return entries, false, nil
}
```

## 9. 截断日志

```go
// 截断到指定索引（保留 index 及之前的）
func (s *Storage) Truncate(index uint64) error {
    // 1. 截断日志文件
    s.ls.truncateTo(index)
    
    // 2. 更新元数据
    s.truncIndex = index
    s.truncTerm, _ = s.Term(index)
    
    // 3. 保存元数据
    s.metafile.SaveTruncateMeta(truncateMeta{
        truncIndex: s.truncIndex,
        truncTerm:  s.truncTerm,
    })
    
    return s.metafile.Sync()
}
```

## 10. 存储硬状态

```go
func (s *Storage) StoreHardState(st proto.HardState) error {
    // 只在有变化时才保存
    if st.Term != s.hardState.Term || st.Vote != s.hardState.Vote {
        s.hardState = st
        s.metafile.SaveHardState(st)
        return s.metafile.Sync()
    }
    
    // Commit 变化时也要 sync
    if st.Commit != s.prevCommit {
        s.prevCommit = st.Commit
        s.hardState.Commit = st.Commit
        return s.metafile.Sync()
    }
    
    return nil
}
```

## 11. 日志文件轮转

```go
// 检查是否需要轮转
func (ls *logEntryStorage) shouldRotate() bool {
    // 文件大小超限
    if ls.last.Size() >= ls.filesize {
        return true
    }
    // 时间间隔
    if now - ls.rotateTime > rotateInterval {
        return true
    }
    return false
}

// 轮转
func (ls *logEntryStorage) rotate(nextIndex uint64) error {
    // 1. Sync 并关闭当前文件
    ls.last.Sync()
    ls.last.Close()
    
    // 2. 创建新文件
    newFile, _ := ls.createNew(nextIndex)
    ls.last = newFile
    ls.logfiles = append(ls.logfiles, newFile.Name())
    
    return nil
}
```

## 12. 文件缓存

```go
// 使用 LRU 缓存打开的日志文件
type logFileCache struct {
    capacity int
    files    map[logFileName]*logEntryFile
    lru      *list.List
    opener   func(name logFileName) (*logEntryFile, error)
}

func (c *logFileCache) Get(name logFileName) (*logEntryFile, error) {
    // 命中缓存
    if f, ok := c.files[name]; ok {
        c.moveToFront(name)
        return f, nil
    }
    
    // 未命中，打开文件
    f, _ := c.opener(name)
    c.add(name, f)
    
    return f, nil
}
```

## 13. 数据持久化流程

```
┌─────────────────────────────────────────────────────────────┐
│                    持久化流程                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Leader 追加日志到 unstable                              │
│                                                             │
│  2. 复制到 Follower 并收到响应                              │
│                                                             │
│  3. 更新 committed                                          │
│                                                             │
│  4. 持久化 (handleReady)                                    │
│     ├─ storage.StoreEntries(unstable.entries)              │
│     │     └─ 写入日志文件，fsync                            │
│     ├─ storage.StoreHardState(hardState)                   │
│     │     └─ 写入元数据文件，fsync                          │
│     └─ unstable.stableTo(index)                            │
│           └─ 清空已持久化的 unstable                        │
│                                                             │
│  5. 应用到状态机                                             │
│     └─ StateMachine.Apply()                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 14. 配置参数

```go
type Config struct {
    FileSize          int    // 单文件大小 (默认 64MB)
    FileCacheCapacity int    // 文件缓存数量 (默认 2)
    Sync              bool   // 是否每次写入都 sync
}
```

## 15. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| Storage 接口 | storage/storage.go:25 |
| WAL Storage | storage/wal/storage.go:28 |
| NewStorage | storage/wal/storage.go:44 |
| logEntryStorage | storage/wal/log_storage.go:29 |
| logEntryFile | storage/wal/log_file.go |
| metaFile | storage/wal/meta.go |

---
*创建时间：2026-04-22*
