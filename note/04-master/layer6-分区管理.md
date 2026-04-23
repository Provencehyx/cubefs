# Layer 6: 分区管理

## 1. DataPartition 结构

DataPartition 是数据存储的基本单位。

```go
// data_partition.go:34
type DataPartition struct {
    PartitionID      uint64             // 分区 ID
    PartitionType    int                // 分区类型
    ReplicaNum       uint8              // 副本数
    Status           int8               // 状态
    isRecover        bool               // 是否恢复中
    
    Replicas         []*DataReplica     // 副本列表
    LeaderReportTime int64              // Leader 上报时间
    Hosts            []string           // 主机列表
    Peers            []proto.Peer       // Raft Peer 列表
    
    total            uint64             // 总空间
    used             uint64             // 已用空间
    
    VolName          string             // 所属卷名
    VolID            uint64             // 所属卷 ID
    createTime       int64              // 创建时间
    modifyTime       int64              // 修改时间
    
    MissingNodes     map[string]int64   // 丢失的节点
    FileInCoreMap    map[string]*FileInCore  // 文件信息
    
    // 下线相关
    DecommissionStatus    uint32        // 下线状态
    DecommissionSrcAddr   string        // 下线源地址
    DecommissionDstAddr   string        // 下线目标地址
    DecommissionRaftForce bool          // 强制 Raft 下线
    
    OfflinePeerID    uint64             // 下线的 Peer ID
    RdOnly           bool               // 只读
    MediaType        uint32             // 介质类型
}
```

## 2. DataReplica 结构

```go
// data_replica.go
type DataReplica struct {
    Addr            string             // 节点地址
    dataNode        *DataNode          // DataNode 引用
    
    ReportTime      int64              // 上报时间
    Total           uint64             // 总空间
    Used            uint64             // 已用空间
    
    Status          int8               // 状态
    DiskPath        string             // 磁盘路径
    IsLeader        bool               // 是否 Leader
    
    FileCount       uint32             // 文件数
    NeedsToCompare  bool               // 需要比较
    
    ReadOnlyReasons uint32             // 只读原因
}
```

## 3. MetaPartition 结构

MetaPartition 是元数据存储的基本单位。

```go
// meta_partition.go:54
type MetaPartition struct {
    PartitionID      uint64             // 分区 ID
    Start            uint64             // 起始 inode ID
    End              uint64             // 结束 inode ID
    MaxInodeID       uint64             // 最大 inode ID
    InodeCount       uint64             // inode 数量
    DentryCount      uint64             // dentry 数量
    
    Replicas         []*MetaReplica     // 副本列表
    LeaderReportTime int64              // Leader 上报时间
    ReplicaNum       uint8              // 副本数
    Status           int8               // 状态
    IsRecover        bool               // 是否恢复中
    
    volID            uint64             // 所属卷 ID
    volName          string             // 所属卷名
    Hosts            []string           // 主机列表
    Peers            []proto.Peer       // Raft Peer 列表
    
    MissNodes        map[string]int64   // 丢失的节点
    OfflinePeerID    uint64             // 下线的 Peer ID
    
    VerSeq           uint64             // 版本序列
}
```

## 4. MetaReplica 结构

```go
// meta_partition.go:30
type MetaReplica struct {
    Addr            string             // 节点地址
    start           uint64             // 起始 inode ID
    end             uint64             // 结束 inode ID
    dataSize        uint64             // 数据大小
    nodeID          uint64             // 节点 ID
    
    MaxInodeID      uint64             // 最大 inode ID
    InodeCount      uint64             // inode 数量
    DentryCount     uint64             // dentry 数量
    
    ReportTime      int64              // 上报时间
    Status          int8               // 状态
    IsLeader        bool               // 是否 Leader
    
    metaNode        *MetaNode          // MetaNode 引用
}
```

## 5. 分区状态

```go
// proto/proto.go
const (
    Unavailable = iota  // 0: 不可用
    ReadOnly            // 1: 只读
    ReadWrite           // 2: 可读写
)
```

## 6. 分区创建流程

### 6.1 创建 DataPartition

```
请求创建 DataPartition
       │
       ├── 1. 分配分区 ID
       │       └── idAlloc.allocateDataPartitionID()
       │
       ├── 2. 选择节点
       │       └── getHostFromNormalZone(TypeDataPartition, ...)
       │               │
       │               ├── 选择 Zone
       │               │
       │               ├── 选择 NodeSet
       │               │
       │               └── 选择可用 DataNode
       │
       ├── 3. 创建分区结构
       │       └── newDataPartition(id, replicaNum, volName, ...)
       │
       ├── 4. 设置副本
       │       ├── dp.Hosts = hosts
       │       └── dp.Peers = peers
       │
       ├── 5. 下发创建任务
       │       └── c.addDataNodeTasks(createTasks)
       │
       └── 6. 持久化
                └── c.syncAddDataPartition(dp)
```

### 6.2 创建 MetaPartition

```
请求创建 MetaPartition
       │
       ├── 1. 分配分区 ID
       │       └── idAlloc.allocateMetaPartitionID()
       │
       ├── 2. 确定 inode 范围
       │       └── [start, end)
       │
       ├── 3. 选择节点
       │       └── getHostFromNormalZone(TypeMetaPartition, ...)
       │
       ├── 4. 创建分区结构
       │       └── newMetaPartition(id, start, end, replicaNum, ...)
       │
       ├── 5. 下发创建任务
       │       └── c.addMetaNodeTasks(createTasks)
       │
       └── 6. 持久化
                └── c.syncAddMetaPartition(mp)
```

## 7. MetaPartition 分裂

当 MaxInodeID 接近 End 时，需要分裂 MP。

```
分裂流程:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  原 MP:    [Start=1, End=16M, MaxInode=15M]                 │
│                                                             │
│            分裂                                              │
│            ↓                                                │
│                                                             │
│  原 MP:    [Start=1, End=15M+1]    (Range 缩小)             │
│  新 MP:    [Start=15M+1, End=32M]  (新范围)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```go
// 检查是否需要分裂
func (mp *MetaPartition) canSplit(end uint64, step uint64, ignoreNoLeader bool) error {
    // 检查 end 是否合法
    // 检查是否超过 MaxInodeID
    // 检查是否有 Leader
}
```

## 8. 分区检查任务

Master 定期检查分区状态。

### 8.1 检查 DataPartition

```go
// 定期检查 DP 状态
func (c *Cluster) scheduleToCheckDataPartitions() {
    go func() {
        for {
            // 检查每个 DP
            c.checkDataPartitions()
            time.Sleep(time.Second * 60)
        }
    }()
}

func (c *Cluster) checkDataPartitions() {
    vols := c.copyVols()
    for _, vol := range vols {
        for _, dp := range vol.dataPartitions.partitionMap {
            // 检查副本数
            // 检查 Leader
            // 检查可用性
            dp.checkStatus(c)
        }
    }
}
```

### 8.2 检查 MetaPartition

```go
// 定期检查 MP 状态
func (c *Cluster) scheduleToCheckMetaPartitions() {
    go func() {
        for {
            c.checkMetaPartitions()
            time.Sleep(time.Second * 60)
        }
    }()
}
```

## 9. 分区副本管理

### 9.1 添加副本

```go
// 添加 DataPartition 副本
func (c *Cluster) addDataReplica(dp *DataPartition, addr string) error {
    // 1. 获取目标 DataNode
    // 2. 创建 Raft AddMember 任务
    // 3. 下发添加副本任务
    // 4. 更新分区信息
}

// 添加 MetaPartition 副本
func (c *Cluster) addMetaReplica(mp *MetaPartition, addr string) error
```

### 9.2 删除副本

```go
// 删除 DataPartition 副本
func (c *Cluster) deleteDataReplica(dp *DataPartition, addr string, force bool) error {
    // 1. 创建 Raft RemoveMember 任务
    // 2. 下发删除副本任务
    // 3. 更新分区信息
}

// 删除 MetaPartition 副本
func (c *Cluster) deleteMetaReplica(mp *MetaPartition, addr string, ...) error
```

## 10. 分区迁移

分区迁移用于节点下线或负载均衡。

```
迁移流程:
       │
       ├── 1. 选择目标节点
       │
       ├── 2. 添加新副本
       │       └── addDataReplica(dp, newAddr)
       │
       ├── 3. 等待数据同步完成
       │
       └── 4. 删除旧副本
                └── deleteDataReplica(dp, oldAddr)
```

## 11. 分区状态检查

```go
// 检查 DP 状态
func (dp *DataPartition) checkStatus(c *Cluster) {
    // 1. 检查副本数量
    if len(dp.Replicas) < int(dp.ReplicaNum) {
        // 副本不足，需要补充
    }
    
    // 2. 检查 Leader
    if !dp.hasLeader() {
        // 无 Leader，尝试触发选举
    }
    
    // 3. 检查副本状态
    for _, replica := range dp.Replicas {
        if !replica.isActive(timeout) {
            // 副本不活跃
        }
    }
}
```

## 12. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| DataPartition 结构 | data_partition.go:34 |
| DataReplica 结构 | data_replica.go |
| MetaPartition 结构 | meta_partition.go:54 |
| MetaReplica 结构 | meta_partition.go:30 |
| newDataPartition | data_partition.go:95 |
| newMetaPartition | meta_partition.go:100 |
| checkDataPartitions | cluster.go |
| addDataReplica | cluster_task.go |
| deleteDataReplica | cluster_task.go |

---

## 下一步
- Layer 7: 元数据持久化

*创建时间：2026-04-12*
