# Layer 4: 节点管理

## 1. DataNode 结构

Master 中的 DataNode 结构用于管理数据节点的元信息。

```go
// data_node.go:36
type DataNode struct {
    Total              uint64            // 总空间
    Used               uint64            // 已用空间
    AvailableSpace     uint64            // 可用空间
    ID                 uint64            // 节点 ID
    ZoneName           string            // Zone 名称
    Addr               string            // 地址
    HeartbeatPort      string            // 心跳端口
    ReplicaPort        string            // 复制端口
    
    ReportTime         time.Time         // 上报时间
    StartTime          int64             // 启动时间
    LastUpdateTime     time.Time         // 最后更新时间
    isActive           bool              // 是否活跃
    
    UsageRatio         float64           // 使用率
    SelectedTimes      uint64            // 被选中次数
    TaskManager        *AdminTaskManager // 任务管理器
    
    DataPartitionReports []*proto.DataPartitionReport  // DP 上报
    DataPartitionCount   uint32                        // DP 数量
    TotalPartitionSize   uint64                        // 分区总大小
    
    NodeSetID          uint64            // 所属 NodeSet
    BadDisks           []string          // 坏盘列表
    DiskStats          []proto.DiskStat  // 磁盘统计
    
    // 下线相关
    ToBeOffline        bool              // 待下线
    RdOnly             bool              // 只读
    DecommissionStatus uint32            // 下线状态
    DecommissionDstAddr string           // 下线目标地址
    
    // QoS
    QosIopsRLimit      uint64
    QosIopsWLimit      uint64
    QosFlowRLimit      uint64
    QosFlowWLimit      uint64
    
    MediaType          uint32            // 介质类型
}
```

## 2. MetaNode 结构

```go
// meta_node.go:29
type MetaNode struct {
    ID                 uint64            // 节点 ID
    Addr               string            // 地址
    DomainAddr         string            // 域地址
    IsActive           bool              // 是否活跃
    Sender             *AdminTaskManager // 任务发送器
    ZoneName           string            // Zone 名称
    
    MaxMemAvailWeight  uint64            // 最大可用内存权重
    Total              uint64            // 总内存
    Used               uint64            // 已用内存
    Ratio              float64           // 使用率
    NodeMemTotal       uint64            // 节点总内存
    NodeMemUsed        uint64            // 节点已用内存
    
    SelectCount        uint64            // 被选中次数
    Threshold          float32           // 阈值
    ReportTime         time.Time         // 上报时间
    
    metaPartitionInfos []*proto.MetaPartitionReport  // MP 上报
    MetaPartitionCount int               // MP 数量
    
    NodeSetID          uint64            // 所属 NodeSet
    ToBeOffline        bool              // 待下线
    RdOnly             bool              // 只读
    
    HeartbeatPort      string            // 心跳端口
    ReplicaPort        string            // 复制端口
}
```

## 3. 节点创建

### 3.1 创建 DataNode

```go
// data_node.go:96
func newDataNode(addr, raftHeartbeatPort, raftReplicaPort, zoneName, 
                 clusterID string, mediaType uint32) (dataNode *DataNode) {
    if zoneName == "" {
        zoneName = DefaultZoneName
    }
    
    dataNode = new(DataNode)
    dataNode.Total = 1
    dataNode.Addr = addr
    dataNode.HeartbeatPort = raftHeartbeatPort
    dataNode.ReplicaPort = raftReplicaPort
    dataNode.ZoneName = zoneName
    dataNode.LastUpdateTime = time.Now().Add(-time.Minute)
    dataNode.TaskManager = newAdminTaskManager(dataNode.Addr, clusterID)
    dataNode.DecommissionStatus = DecommissionInitial
    dataNode.MediaType = mediaType
    
    return
}
```

### 3.2 创建 MetaNode

```go
// meta_node.go:60
func newMetaNode(addr, heartbeatPort, replicaPort, zoneName, 
                 clusterID string) (node *MetaNode) {
    node = &MetaNode{
        Addr:          addr,
        HeartbeatPort: heartbeatPort,
        ReplicaPort:   replicaPort,
        ZoneName:      zoneName,
        Sender:        newAdminTaskManager(addr, clusterID),
    }
    return
}
```

## 4. 节点注册流程

```
节点启动后发送注册请求
       │
       ├── Master 收到注册请求
       │
       ├── 检查节点是否已存在
       │       │
       │       ├── 已存在 → 更新信息
       │       │
       │       └── 不存在 → 创建新节点
       │                   │
       │                   ├── 分配节点 ID
       │                   │
       │                   ├── 创建节点结构
       │                   │
       │                   ├── 加入 Zone/NodeSet
       │                   │
       │                   └── 通过 Raft 持久化
       │
       └── 返回注册结果
```

## 5. 心跳处理

### 5.1 DataNode 心跳

```go
// 收到 DataNode 心跳后更新信息
func (c *Cluster) handleDataNodeHeartbeat(nodeAddr string, 
                                          nodeReportInfo *proto.DataNodeHeartbeatResponse) {
    // 1. 获取 DataNode
    dataNode, err := c.dataNode(nodeAddr)
    
    // 2. 更新基础信息
    dataNode.Total = nodeReportInfo.Total
    dataNode.Used = nodeReportInfo.Used
    dataNode.AvailableSpace = nodeReportInfo.Available
    dataNode.ReportTime = time.Now()
    dataNode.isActive = true
    
    // 3. 更新磁盘信息
    dataNode.DiskStats = nodeReportInfo.DiskStats
    dataNode.BadDisks = nodeReportInfo.BadDisks
    
    // 4. 更新分区信息
    dataNode.DataPartitionReports = nodeReportInfo.DataPartitionReports
    dataNode.DataPartitionCount = uint32(len(nodeReportInfo.DataPartitionReports))
}
```

### 5.2 MetaNode 心跳

```go
// 收到 MetaNode 心跳后更新信息
func (c *Cluster) handleMetaNodeHeartbeat(nodeAddr string,
                                          nodeReportInfo *proto.MetaNodeHeartbeatResponse) {
    // 1. 获取 MetaNode
    metaNode, err := c.metaNode(nodeAddr)
    
    // 2. 更新基础信息
    metaNode.Total = nodeReportInfo.Total
    metaNode.Used = nodeReportInfo.Used
    metaNode.ReportTime = time.Now()
    metaNode.IsActive = true
    
    // 3. 更新分区信息
    metaNode.metaPartitionInfos = nodeReportInfo.MetaPartitionReports
    metaNode.MetaPartitionCount = len(nodeReportInfo.MetaPartitionReports)
}
```

## 6. 节点状态检查

### 6.1 活跃性检查

```go
// data_node.go:131
func (dataNode *DataNode) checkLiveness() {
    dataNode.Lock()
    defer dataNode.Unlock()
    
    // 超过 defaultNodeTimeOutSec (60秒) 未上报则标记为不活跃
    if time.Since(dataNode.ReportTime) > time.Second*time.Duration(defaultNodeTimeOutSec) {
        dataNode.isActive = false
    }
}
```

### 6.2 可写性检查

```go
// DataNode 可写条件
func (dataNode *DataNode) IsWriteAble() bool {
    return dataNode.isActive && 
           dataNode.AvailableSpace > minAvailableSpace &&
           !dataNode.RdOnly &&
           !dataNode.ToBeOffline
}

// MetaNode 可写条件
func (metaNode *MetaNode) IsWriteAble() (ok bool) {
    if metaNode.IsActive && 
       metaNode.MaxMemAvailWeight > gConfig.metaNodeReservedMem &&
       !metaNode.reachesThreshold() && 
       metaNode.MetaPartitionCount < defaultMaxMetaPartitionCountOnEachNode &&
       !metaNode.RdOnly {
        ok = true
    }
    return
}
```

## 7. 节点下线

### 7.1 下线状态

```go
const (
    DecommissionInitial     = 0  // 初始状态
    DecommissionRunning     = 1  // 下线中
    DecommissionPause       = 2  // 暂停
    DecommissionSuccess     = 3  // 成功
    DecommissionFail        = 4  // 失败
    DecommissionRetry       = 5  // 重试
)
```

### 7.2 下线流程

```
发起节点下线
       │
       ├── 1. 标记节点 ToBeOffline = true
       │
       ├── 2. 迁移该节点上的分区
       │       │
       │       ├── 遍历所有分区
       │       │
       │       └── 为每个分区添加新副本，删除旧副本
       │
       ├── 3. 等待所有分区迁移完成
       │
       └── 4. 从集群中移除节点
```

## 8. AdminTaskManager

用于向节点下发管理任务。

```go
// admin_task_manager.go
type AdminTaskManager struct {
    clusterID string
    targetAddr string
    TaskMap    sync.Map              // taskID -> AdminTask
    sendChan   chan *proto.AdminTask // 发送通道
    exitCh     chan struct{}
}

// 添加任务
func (m *AdminTaskManager) AddTask(task *proto.AdminTask) {
    m.TaskMap.Store(task.ID, task)
    m.sendChan <- task
}

// 同步发送任务并等待响应
func (m *AdminTaskManager) syncSendAdminTask(task *proto.AdminTask) (response *proto.Packet, error)
```

## 9. 节点查询方法

```go
// 获取 DataNode
func (c *Cluster) dataNode(addr string) (*DataNode, error)

// 获取 MetaNode  
func (c *Cluster) metaNode(addr string) (*MetaNode, error)

// 获取所有 DataNode
func (c *Cluster) getAllDataNodes() []*DataNode

// 获取所有 MetaNode
func (c *Cluster) getAllMetaNodes() []*MetaNode

// 获取 Zone 内的 DataNode
func (zone *Zone) getDataNodes() []*DataNode

// 获取 NodeSet 内的 DataNode
func (ns *nodeSet) getDataNodes() []*DataNode
```

## 10. 节点统计

```go
type nodeStatInfo struct {
    TotalGB     uint64    // 总空间 (GB)
    UsedGB      uint64    // 已用空间 (GB)
    IncreasedGB int64     // 增量 (GB)
    UsedRatio   string    // 使用率
}
```

## 11. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| DataNode 结构 | data_node.go:36 |
| MetaNode 结构 | meta_node.go:29 |
| newDataNode | data_node.go:96 |
| newMetaNode | meta_node.go:60 |
| checkLiveness | data_node.go:131 |
| IsWriteAble (DataNode) | data_node.go |
| IsWriteAble (MetaNode) | meta_node.go:139 |
| AdminTaskManager | admin_task_manager.go |

---

## 下一步
- Layer 5: 卷管理

*创建时间：2026-04-12*
