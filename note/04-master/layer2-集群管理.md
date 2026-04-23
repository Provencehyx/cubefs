# Layer 2: 集群管理

## 1. Cluster 结构概览

Cluster 是 Master 的核心结构，管理整个集群的状态。

```go
// cluster.go:141
type Cluster struct {
    Name            string              // 集群名称
    CreateTime      int64               // 创建时间
    clusterUuid     string              // 集群 UUID
    leaderInfo      *LeaderInfo         // Leader 信息
    cfg             *clusterConfig      // 配置
    fsm             *MetadataFsm        // Raft 状态机
    partition       raftstore.Partition // Raft 分区
    
    // 嵌入的子结构
    ClusterVolSubItem       // 卷管理
    ClusterTopoSubItem      // 拓扑管理
    ClusterDecommission     // 下线管理
    
    // 状态控制
    stopc           chan bool           // 停止通道
    stopFlag        int32               // 停止标志
    wg              sync.WaitGroup
    metaReady       bool                // 元数据是否就绪
    
    // 其他管理器
    followerReadManager *followerReadManager  // Follower 读管理
    lcMgr           *lifecycleManager        // 生命周期管理
    snapshotMgr     *snapshotDelManager      // 快照管理
    apiLimiter      *ApiLimiter              // API 限流
}
```

## 2. 子结构详解

### 2.1 ClusterVolSubItem (卷管理)

```go
// cluster.go:53
type ClusterVolSubItem struct {
    vols                map[string]*Vol        // 卷映射表
    delayDeleteVolsInfo []*delayDeleteVolInfo  // 延迟删除的卷
    volMutex            sync.RWMutex           // 卷读写锁
    createVolMutex      sync.RWMutex           // 创建卷锁
    deleteVolMutex      sync.RWMutex           // 删除卷锁
}
```

### 2.2 ClusterTopoSubItem (拓扑管理)

```go
// cluster.go:62
type ClusterTopoSubItem struct {
    dataNodes        sync.Map              // DataNode 映射
    metaNodes        sync.Map              // MetaNode 映射
    lcNodes          sync.Map              // LC Node 映射
    
    idAlloc          *IDAllocator          // ID 分配器
    t                *topology             // 拓扑结构
    
    dataNodeStatInfo *nodeStatInfo         // DataNode 统计
    metaNodeStatInfo *nodeStatInfo         // MetaNode 统计
    zoneStatInfos    map[string]*proto.ZoneStat  // Zone 统计
    
    FaultDomain      bool                  // 是否启用故障域
    domainManager    *DomainManager        // 域管理器
}
```

### 2.3 ClusterDecommission (下线管理)

```go
// cluster.go:106
type ClusterDecommission struct {
    BadDataPartitionIds   *sync.Map         // 坏的数据分区
    BadMetaPartitionIds   *sync.Map         // 坏的元数据分区
    DecommissionDisks     sync.Map          // 下线中的磁盘
    DecommissionLimit     uint64            // 下线并发限制
    DecommissionDiskLimit uint32            // 磁盘下线限制
    
    ForbidMpDecommission       bool         // 禁止 MP 下线
    EnableAutoDecommissionDisk atomicutil.Bool  // 自动下线磁盘
}
```

## 3. Cluster 初始化

```go
// cluster.go:433
func newCluster(name string, leaderInfo *LeaderInfo, fsm *MetadataFsm, 
                partition raftstore.Partition, cfg *clusterConfig, 
                server *Server) (c *Cluster) {
    c = new(Cluster)
    c.Name = name
    c.leaderInfo = leaderInfo
    c.vols = make(map[string]*Vol)
    c.stopc = make(chan bool)
    c.cfg = cfg
    
    // 初始化拓扑
    c.t = newTopology()
    
    // 初始化统计
    c.BadDataPartitionIds = new(sync.Map)
    c.BadMetaPartitionIds = new(sync.Map)
    c.dataNodeStatInfo = new(nodeStatInfo)
    c.metaNodeStatInfo = new(nodeStatInfo)
    
    // 初始化 Raft
    c.fsm = fsm
    c.partition = partition
    
    // 初始化 ID 分配器
    c.idAlloc = newIDAllocator(c.fsm.store, c.partition)
    
    // 初始化管理器
    c.domainManager = newDomainManager(c)
    c.followerReadManager = newFollowerReadManager(c)
    c.lcMgr = newLifecycleManager()
    c.snapshotMgr = newSnapshotManager()
    c.apiLimiter = newApiLimiter()
    
    return
}
```

## 4. 定时任务调度

Master 启动后会启动一系列定时任务来维护集群状态。

```go
// cluster.go:492
func (c *Cluster) scheduleTask() {
    c.scheduleToCheckDelayDeleteVols()      // 检查延迟删除的卷
    c.scheduleToCheckDataPartitions()       // 检查数据分区
    c.scheduleToLoadDataPartitions()        // 加载数据分区
    c.scheduleToCheckReleaseDataPartitions() // 检查释放数据分区
    c.scheduleToCheckHeartbeat()            // 检查心跳
    c.scheduleToCheckMetaPartitions()       // 检查元数据分区
    c.scheduleToUpdateStatInfo()            // 更新统计信息
    c.scheduleToManageDp()                  // 管理数据分区
    c.scheduleToCheckVolStatus()            // 检查卷状态
    c.scheduleToCheckVolQos()               // 检查卷 QoS
    c.scheduleToCheckDiskRecoveryProgress() // 检查磁盘恢复进度
    c.scheduleToCheckMetaPartitionRecoveryProgress() // 检查 MP 恢复
    c.scheduleToLoadMetaPartitions()        // 加载元数据分区
    c.scheduleToReduceReplicaNum()          // 减少副本数
    c.scheduleToCheckNodeSetGrpManagerStatus() // 检查 NodeSet 状态
    c.scheduleToCheckFollowerReadCache()    // 检查 Follower 读缓存
    c.scheduleToCheckVolStorageSize()       // 检查卷存储大小
    c.scheduleToCleanupLcTask()             // 清理生命周期任务
}
```

### 定时任务表

| 任务 | 间隔 | 说明 |
|------|------|------|
| checkDataPartitions | 60s | 检查 DP 状态、副本、Leader |
| checkMetaPartitions | 60s | 检查 MP 状态、副本 |
| checkHeartbeat | 60s | 检查节点心跳 |
| updateStatInfo | 2min | 更新集群统计信息 |
| checkVolStatus | 5s | 检查卷状态 |
| loadDataPartitions | 5min | 加载 DP 信息 |

## 5. 心跳检查流程

```
scheduleToCheckHeartbeat()
       │
       ├── checkDataNodeHeartbeat()
       │       │
       │       ├── 遍历所有 DataNode
       │       │
       │       ├── 检查 ReportTime 是否超时
       │       │
       │       └── 超时 → 标记 isActive = false
       │
       └── checkMetaNodeHeartbeat()
               │
               ├── 遍历所有 MetaNode
               │
               ├── 检查 ReportTime 是否超时
               │
               └── 超时 → 标记 IsActive = false
```

## 6. 集群统计信息

```go
// cluster_stat.go
type nodeStatInfo struct {
    TotalGB     uint64    // 总空间 (GB)
    UsedGB      uint64    // 已用空间 (GB)
    IncreasedGB int64     // 增量 (GB)
    UsedRatio   string    // 使用率
}
```

### 统计信息更新

```
scheduleToUpdateStatInfo()
       │
       ├── 统计 DataNode 空间
       │       ├── 累加 Total
       │       └── 累加 Used
       │
       ├── 统计 MetaNode 空间
       │       ├── 累加 Total
       │       └── 累加 Used
       │
       ├── 统计各 Zone 信息
       │
       └── 更新 dataNodeStatInfo / metaNodeStatInfo
```

## 7. ID 分配器

Master 负责分配全局唯一的 ID。

```go
// id_allocator.go:30
type IDAllocator struct {
    dataPartitionID uint64    // 数据分区 ID
    metaPartitionID uint64    // 元数据分区 ID
    commonID        uint64    // 通用 ID（节点、NodeSet）
    quotaID         uint32    // 配额 ID
    
    store           *raftstore_db.RocksDBStore
    partition       raftstore.Partition
}
```

### ID 分配流程

```
allocateDataPartitionID()
       │
       ├── dpIDLock.Lock()
       │
       ├── 当前 ID + 1
       │
       ├── 通过 Raft 持久化到 RocksDB
       │       └── c.syncPutCluster()
       │
       └── 返回新 ID
```

## 8. 集群元数据持久化

所有集群状态变更都通过 Raft 持久化。

```go
// 同步写入集群配置
func (c *Cluster) syncPutCluster() error

// 同步添加 DataNode
func (c *Cluster) syncAddDataNode(dataNode *DataNode) error

// 同步添加 MetaNode
func (c *Cluster) syncAddMetaNode(metaNode *MetaNode) error

// 同步更新卷
func (c *Cluster) syncUpdateVol(vol *Vol) error

// 同步更新分区
func (c *Cluster) syncUpdateDataPartition(dp *DataPartition) error
```

## 9. 任务管理

Master 通过 AdminTask 向节点下发任务。

```go
// cluster_task.go

// 添加 DataNode 任务
func (c *Cluster) addDataNodeTasks(tasks []*proto.AdminTask) {
    for _, t := range tasks {
        if node, err := c.dataNode(t.OperatorAddr); err == nil {
            node.TaskManager.AddTask(t)
        }
    }
}

// 添加 MetaNode 任务
func (c *Cluster) addMetaNodeTasks(tasks []*proto.AdminTask) {
    for _, t := range tasks {
        if node, err := c.metaNode(t.OperatorAddr); err == nil {
            node.Sender.AddTask(t)
        }
    }
}
```

## 10. Cluster 核心方法

| 方法 | 说明 |
|------|------|
| `dataNode(addr)` | 获取 DataNode |
| `metaNode(addr)` | 获取 MetaNode |
| `getVol(name)` | 获取卷 |
| `createVol()` | 创建卷 |
| `deleteVol()` | 删除卷 |
| `createDataPartition()` | 创建数据分区 |
| `createMetaPartition()` | 创建元数据分区 |
| `getAllDataNodes()` | 获取所有 DataNode |
| `getAllMetaNodes()` | 获取所有 MetaNode |

## 11. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| Cluster 结构 | cluster.go:141 |
| ClusterVolSubItem | cluster.go:53 |
| ClusterTopoSubItem | cluster.go:62 |
| ClusterDecommission | cluster.go:106 |
| newCluster | cluster.go:433 |
| scheduleTask | cluster.go:492 |
| IDAllocator | id_allocator.go:30 |

---

## 下一步
- Layer 3: 拓扑管理

*创建时间：2026-04-12*
