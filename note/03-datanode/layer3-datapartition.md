# Layer 3: DataPartition 核心

## 核心数据结构

```go
// datanode/partition.go:102
type DataPartition struct {
    // 基本信息
    clusterID       string           // 集群 ID
    volumeID        string           // 卷 ID
    partitionID     uint64           // 分区 ID
    partitionStatus int              // 状态 (ReadWrite/ReadOnly/...)
    partitionSize   int              // 分区大小 (默认 120GB)
    partitionType   int              // 分区类型
    replicaNum      int              // 副本数
    
    // 副本信息
    replicas        []string         // 副本地址列表
    replicasLock    sync.RWMutex
    isLeader        bool             // 链式复制 Leader
    isRaftLeader    bool             // Raft Leader
    
    // 关联对象
    disk            *Disk            // 所属磁盘
    dataNode        *DataNode        // 所属节点
    path            string           // 分区目录路径
    
    // 存储引擎
    extentStore     *storage.ExtentStore   // Extent 存储管理
    raftPartition   raftstore.Partition    // Raft 分区
    
    // Raft 状态
    appliedID       uint64           // 已应用的 Raft 日志 ID
    lastTruncateID  uint64           // 最后截断的 ID
    metaAppliedID   uint64           // 持久化的 ApplyID
    minAppliedID    uint64           // 最小 ApplyID
    maxAppliedID    uint64           // 最大 ApplyID
    
    // 控制信号
    stopRaftC       chan uint64      // 停止 Raft 信号
    storeC          chan uint64      // 存储信号
    stopC           chan bool        // 停止信号
    
    // 快照与版本
    snapshot        []*proto.File    // 快照信息
    verSeq          uint64           // 版本序列号
    
    // 修复相关
    decommissionRepairProgress float64  // 下线修复进度
    stopRecover     bool             // 停止修复
    recoverErrCnt   uint64           // 修复错误计数
}
```

**关键字段说明**：

| 字段 | 作用 |
|------|------|
| `isLeader` | 链式复制的 Leader，Client 写入首先到达此节点 |
| `isRaftLeader` | Raft 协议的 Leader，处理随机写和成员变更 |
| `extentStore` | 底层存储引擎，管理 TinyExtent 和 NormalExtent |
| `raftPartition` | Raft 组，用于随机写一致性 |
| `appliedID` | 已应用的 Raft 日志位点，重启恢复时使用 |

**两个 Leader 的区别**：

```
通常 isLeader == isRaftLeader (同一节点)

isLeader (链式复制):
  - 控制数据写入顺序
  - 数据: Client → Leader → Follower1 → Follower2

isRaftLeader (Raft 协议):
  - 控制成员变更
  - 控制随机写顺序
  - 日志: Leader ─┬─→ Follower1
                  └─→ Follower2
```

---

## 核心问题

1. **为什么分区大小是 120GB？**
2. **分区元数据如何持久化？**
3. **isLeader 和 isRaftLeader 有什么区别？**

---

## 1. 分区大小设计

```go
// util/unit.go:35
DefaultDataPartitionSize = 120 * GB
```

### 为什么是 120GB？

| 分区大小 | 优点 | 缺点 |
|----------|------|------|
| 太小 (10GB) | 故障恢复快 | 分区数量爆炸，管理开销大 |
| 太大 (1TB) | 分区数量少 | 故障恢复慢，一个坏分区影响大 |
| 120GB | 平衡点 | — |

**计算**：
- 假设网络带宽 1Gbps ≈ 100MB/s
- 修复 120GB 需要 ~20 分钟
- 可接受的恢复时间窗口

### 分区数量示例

```
10TB 卷，3 副本，分区大小 120GB:
  分区数 = 10TB / 120GB ≈ 85 个
  副本总数 = 85 × 3 = 255 个
  
分布在 10 个 DataNode 上：
  每节点 ≈ 25 个分区副本，可管理
```

---

## 2. 两个 Leader 的区别

```go
type DataPartition struct {
    isLeader     bool   // 复制协议 Leader
    isRaftLeader bool   // Raft 协议 Leader
}
```

**为什么有两个？**

```
┌──────────────────────────────────────────────────────────────┐
│                    一个分区的两套协议                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  链式复制 (isLeader)              Raft (isRaftLeader)         │
│  ─────────────────────           ────────────────────         │
│  用途: 数据写入                   用途: 成员变更、随机写        │
│  Client 写 → Leader              成员变更 → Raft Leader       │
│    → Follower1 → Follower2       随机覆盖写 → Raft Leader     │
│                                                               │
│  通常两个角色在同一节点，但理论上可以分离                        │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**实际情况**：
- 两个 Leader 通常是同一个节点
- 但代码分开跟踪，为了灵活性

---

## 3. 元数据持久化

### META 文件内容

```json
{
    "VolumeID": "vol1",
    "PartitionID": 1001,
    "PartitionSize": 128849018880,
    "Peers": [
        {"ID": 1, "Addr": "192.168.1.1:17310"},
        {"ID": 2, "Addr": "192.168.1.2:17310"},
        {"ID": 3, "Addr": "192.168.1.3:17310"}
    ],
    "Hosts": ["192.168.1.1:17310", "192.168.1.2:17310", "192.168.1.3:17310"],
    "ReplicaNum": 3,
    "ApplyID": 12345,
    "LastTruncateID": 10000
}
```

### 为什么需要持久化 ApplyID？

```
场景：DataNode 重启

没有持久化 ApplyID:
  1. 重启后 ApplyID = 0
  2. Raft 日志被 truncate 了
  3. 找不到日志 0-10000
  4. 分区无法恢复！

有持久化 ApplyID:
  1. 重启后读取 ApplyID = 12345
  2. 从 12345 开始重放
  3. 正常恢复
```

### 持久化时机

```go
// 单独的 APPLY 文件，高频更新
func (dp *DataPartition) persistApplyID() {
    // 写入 APPLY 文件
    // 比 META 更频繁，因为 ApplyID 每次 Raft 操作都变化
}
```

**为什么 APPLY 单独存？**
- ApplyID 变化频繁（每次 Raft 操作）
- META 其他字段变化少（成员变更时）
- 分开存避免频繁重写整个 META

---

## 4. 目录结构设计

```
datapartition_1001_3/               # 命名: datapartition_{ID}_{副本数}
├── META                            # 分区元数据 (JSON)
├── APPLY                           # Raft ApplyID (二进制)
├── EXTENT_CRC                      # 所有 Extent 的 CRC
├── EXTENT_META                     # Extent 分配信息
├── TINYEXTENT_DELETE               # TinyExtent 删除记录
├── NORMALEXTENT_DELETE             # NormalExtent 删除记录
├── wal_1001/                       # Raft WAL 日志
│   ├── 00000001.log
│   └── ...
├── 1, 2, ... 64                    # TinyExtent 文件
└── 1024, 1025, ...                 # NormalExtent 文件
```

### 各文件作用

| 文件 | 作用 | 更新频率 |
|------|------|----------|
| `META` | Raft 成员、分区配置 | 低（成员变更时） |
| `APPLY` | Raft apply 位点 | 高（每次 Raft 操作） |
| `EXTENT_CRC` | 数据完整性校验 | 中（写入时） |
| `*_DELETE` | 软删除记录 | 中（删除时） |
| `wal_*/` | Raft 日志 | 高 |

---

## 5. 生命周期

```
┌───────────────────────────────────────────────────────────────┐
│                     DataPartition 生命周期                     │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Master 分配     DataNode 创建      正常服务                    │
│  ─────────────   ─────────────      ──────────                 │
│                                                                │
│  ┌──────────┐    ┌──────────┐       ┌──────────┐              │
│  │ Allocate │───▶│  Create  │──────▶│ ReadWrite│              │
│  └──────────┘    └──────────┘       └────┬─────┘              │
│                                          │                     │
│                            ┌─────────────┼─────────────┐       │
│                            ▼             ▼             ▼       │
│                      ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│                      │ ReadOnly │  │Recovering│  │  Delete  │ │
│                      │(空间满)   │  │(副本修复) │  │(卷删除)  │ │
│                      └──────────┘  └──────────┘  └──────────┘ │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### 状态转换触发条件

| 从 | 到 | 触发条件 |
|----|-----|----------|
| ReadWrite | ReadOnly | 分区空间不足 / 磁盘空间不足 |
| ReadWrite | Recovering | 发现副本数据不一致 |
| Recovering | ReadWrite | 修复完成 |
| 任意 | Delete | 卷被删除 / 管理员删除 |

---

## 6. 创建与加载

### 创建流程

```
Master 请求创建 ─────────────────────────────────────────────────
       │
       ▼
SpaceManager.CreatePartition()
       │
       ├── 1. selectDisk()            选择磁盘 (Straw 算法)
       │
       ├── 2. os.Mkdir(path)          创建目录
       │
       ├── 3. newDataPartition()      创建内存结构
       │
       ├── 4. NewExtentStore()        创建存储引擎
       │
       ├── 5. StartRaft()             启动 Raft (空成员)
       │
       └── 6. PersistMetadata()       写入 META 文件
```

### 加载流程（重启时）

```
DataNode 启动 ─────────────────────────────────────────────────
       │
       ▼
disk.RestorePartition()
       │
       └── 遍历 datapartition_* 目录
           │
           ▼
       LoadDataPartition()
           │
           ├── 1. 读取 META 文件
           │
           ├── 2. 读取 APPLY 文件 (获取 ApplyID)
           │
           ├── 3. newDataPartition()
           │
           ├── 4. NewExtentStore()
           │
           └── 5. StartRaft(isLoad=true)  ← 从 ApplyID 恢复
```

---

## 7. DataPartition vs MetaPartition

| 维度 | DataPartition | MetaPartition |
|------|---------------|---------------|
| 存储内容 | 文件字节数据 | inode/dentry |
| 存储引擎 | 裸文件 (Extent) | RocksDB |
| 分区大小 | 120GB | 不限（按 inode 数量分裂） |
| 复制协议 | 链式复制 + Raft | 纯 Raft |
| Raft 用途 | 成员变更、随机写 | 所有写操作 |
| 修复粒度 | 单个 Extent | 整个快照 |

---

## 8. 关键代码

| 功能 | 位置 | 要点 |
|------|------|------|
| 分区大小定义 | util/unit.go:35 | `DefaultDataPartitionSize = 120 * GB` |
| 创建分区 | partition.go:199 | `CreateDataPartition()` |
| 加载分区 | partition.go | `LoadDataPartition()` |
| 持久化 ApplyID | partition.go | `persistApplyID()` 独立于 META |
| 启动 Raft | partition_raft.go:85 | `StartRaft()` |

---

*更新时间：2026-04-27*
