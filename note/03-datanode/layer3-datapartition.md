# Layer 3: DataPartition 核心

## 1. DataPartition 结构

DataPartition 是数据分区的核心结构，管理一组 Extent 文件的存储和副本复制。

```go
// partition.go:102
type DataPartition struct {
    // 基本信息
    clusterID       string
    volumeID        string
    partitionID     uint64
    partitionStatus int
    partitionSize   int              // 分区大小 (默认 120GB)
    partitionType   int              // Normal / Tiny
    replicaNum      int              // 副本数
    
    // 副本信息
    replicas        []string         // 副本地址列表
    isLeader        bool             // 是否 Leader
    isRaftLeader    bool             // Raft Leader
    
    // 存储相关
    disk            *Disk            // 所属磁盘
    path            string           // 分区目录路径
    extentStore     *storage.ExtentStore  // Extent 存储引擎
    
    // Raft 相关
    raftPartition   raftstore.Partition   // Raft 分区
    appliedID       uint64           // 已应用的 Raft 日志 ID
    lastTruncateID  uint64           // 上次截断的日志 ID
    
    // 控制通道
    stopC           chan bool
    stopRaftC       chan uint64
    storeC          chan uint64
    
    // 状态
    raftStatus      int32            // Raft 运行状态
    used            int              // 已用空间
    leaderSize      int              // Leader 上的大小
}
```

## 2. 元数据结构

```go
// partition.go:68
type DataPartitionMetadata struct {
    VolumeID                string              // 所属卷
    PartitionID             uint64              // 分区 ID
    PartitionSize           int                 // 分区大小
    PartitionType           int                 // 分区类型
    CreateTime              string              // 创建时间
    Peers                   []proto.Peer        // Raft 成员
    Hosts                   []string            // 副本主机列表
    ReplicaNum              int                 // 副本数
    LastTruncateID          uint64              // 最后截断 ID
    ApplyID                 uint64              // 应用 ID
    DataPartitionCreateType int                 // 创建类型
}
```

## 3. 分区目录结构

```
datapartition_1001_3/              # 分区目录 (ID_副本数)
├── META                           # 元数据 JSON 文件
├── APPLY                          # 已应用的 Raft ID
├── .dpStatus                      # 分区状态
├── EXTENT_CRC                     # Extent CRC 校验
├── EXTENT_META                    # Extent 元信息
├── TINYEXTENT_DELETE              # Tiny Extent 删除记录
├── NORMALEXTENT_DELETE            # Normal Extent 删除记录
├── wal_1001/                      # Raft WAL 日志
│   └── ...
├── 1                              # Extent 文件 (ID 1-64 为 TinyExtent)
├── 2
├── ...
├── 1024                           # Normal Extent (ID >= 1024)
├── 1025
└── ...
```

## 4. DataPartition 与 MetaPartition 对比

| 维度 | DataPartition | MetaPartition |
|------|---------------|---------------|
| 存储内容 | 文件数据 (Extent) | 元数据 (inode/dentry) |
| 存储方式 | 多个文件 (每个 Extent 一个) | 内存 B-Tree + 快照 |
| 分区大小 | 120GB (默认) | 较小 |
| Extent 类型 | Tiny (1-64) + Normal (1024+) | 无 |
| 副本同步 | Raft + 主从复制 | 纯 Raft |
| 数据修复 | 按 Extent 修复 | 按分区快照修复 |

## 5. 分区类型

### 5.1 Normal Partition
- 存储普通大小文件
- 使用 Normal Extent (ID >= 1024)
- 每个 Extent 最大 128MB

### 5.2 Tiny Partition (已废弃，保留兼容)
- 存储小文件
- 使用 Tiny Extent (ID 1-64)
- 多个小文件共享一个 Extent

## 6. 生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                   DataPartition 生命周期                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │  Create  │────▶│  Normal  │────▶│  Delete  │            │
│  └──────────┘     └──────────┘     └──────────┘            │
│       │                │                                    │
│       │                ▼                                    │
│       │          ┌──────────┐                               │
│       │          │ ReadOnly │  (空间满/下线)                 │
│       │          └──────────┘                               │
│       │                │                                    │
│       ▼                ▼                                    │
│  ┌──────────────────────────┐                               │
│  │       Recovering         │  (副本修复中)                  │
│  └──────────────────────────┘                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 7. 核心操作

### 7.1 创建分区

```go
// partition.go:199
func CreateDataPartition(dpCfg *dataPartitionCfg, disk *Disk, 
    request *proto.CreateDataPartitionRequest) (dp *DataPartition, err error) {
    
    // 1. 创建分区实例
    dp, err = newDataPartition(dpCfg, disk, true)
    
    // 2. 注册到 SpaceManager
    disk.space.partitions[dp.partitionID] = dp
    
    // 3. 加载 Extent 头信息
    dp.ForceLoadHeader()
    
    // 4. 启动 Raft
    err = dp.StartRaft(false)
    
    // 5. 持久化元数据
    err = dp.PersistMetadata()
    
    return
}
```

### 7.2 加载分区

```go
// partition.go:281
func LoadDataPartition(partitionDir string, disk *Disk) (dp *DataPartition, err error) {
    // 1. 读取 META 文件
    metaFileData, err = ioutil.ReadFile(path.Join(partitionDir, "META"))
    
    // 2. 解析元数据
    meta := &DataPartitionMetadata{}
    json.Unmarshal(metaFileData, meta)
    
    // 3. 创建分区实例
    dp, err = newDataPartition(dpCfg, disk, false)
    
    // 4. 启动 Raft
    dp.StartRaft(true)
    
    return
}
```

### 7.3 写入数据

```
Client 写请求
       │
       ▼
DataPartition.Write()
       │
       ├── 1. 检查是否 Leader
       │
       ├── 2. 分配/获取 Extent
       │        │
       │        ▼
       │   ExtentStore.Write()
       │
       ├── 3. 提交 Raft 日志
       │        │
       │        ▼
       │   raftPartition.Submit()
       │
       └── 4. 同步到副本 (Raft)
```

## 8. 分区状态

| 状态 | 值 | 说明 |
|------|-----|------|
| Unavailable | -1 | 不可用 |
| ReadOnly | 1 | 只读 |
| ReadWrite | 2 | 可读写 |
| Recovering | 3 | 恢复中 |

## 9. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| DataPartition 结构 | partition.go:102 |
| 元数据结构 | partition.go:68 |
| 创建分区 | partition.go:199 |
| 加载分区 | partition.go:281 |
| 启动 Raft | partition_raft.go |
| Raft 状态机 | partition_raftfsm.go |

---

## 下一步
- Layer 4: Extent 存储引擎详解

*创建时间：2026-04-12*
