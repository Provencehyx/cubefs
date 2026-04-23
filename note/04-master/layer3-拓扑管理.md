# Layer 3: 拓扑管理

## 1. 拓扑结构概览

Master 通过分层拓扑结构管理集群节点，实现故障域隔离。

```
┌─────────────────────────────────────────────────────────────┐
│                      Topology                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                     Zone 1                           │   │
│   │   ┌─────────────┐  ┌─────────────┐                  │   │
│   │   │  NodeSet 1  │  │  NodeSet 2  │                  │   │
│   │   │ ┌─────────┐ │  │ ┌─────────┐ │                  │   │
│   │   │ │DataNode │ │  │ │DataNode │ │                  │   │
│   │   │ │MetaNode │ │  │ │MetaNode │ │                  │   │
│   │   │ └─────────┘ │  │ └─────────┘ │                  │   │
│   │   └─────────────┘  └─────────────┘                  │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                     Zone 2                           │   │
│   │   ┌─────────────┐  ┌─────────────┐                  │   │
│   │   │  NodeSet 3  │  │  NodeSet 4  │                  │   │
│   │   └─────────────┘  └─────────────┘                  │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. topology 结构

```go
// topology.go:42
type topology struct {
    zoneMap            *sync.Map         // Zone 名称 -> Zone 映射
    zones              []*Zone           // Zone 列表
    domainExcludeZones []string          // 排除的域
    
    metaTopology       rsManager         // MetaNode 拓扑
    dataTopology       rsManager         // DataNode 拓扑
    
    zoneLock           sync.RWMutex
}

// rsManager 节点管理器
type rsManager struct {
    nodeType         NodeType          // 节点类型
    nodes            *sync.Map         // 节点映射
    zoneIndexForNode int               // Zone 索引
}
```

## 3. Zone 结构

Zone 代表一个可用区（机房/机架）。

```go
// topology.go:1518
type Zone struct {
    name                string            // Zone 名称
    status              int               // 状态
    dataNodes           *sync.Map         // DataNode 映射
    metaNodes           *sync.Map         // MetaNode 映射
    nodeSetMap          map[uint64]*nodeSet  // NodeSet 映射
    nsLock              sync.RWMutex
    
    // QoS 限制
    QosIopsRLimit       uint64
    QosIopsWLimit       uint64
    QosFlowRLimit       uint64
    QosFlowWLimit       uint64
    
    // 节点集选择器
    dataNodesetSelector NodesetSelector
    metaNodesetSelector NodesetSelector
    
    dataMediaType       uint32            // 介质类型（SSD/HDD）
}
```

### Zone 状态

| 状态 | 值 | 说明 |
|------|-----|------|
| normalZone | 0 | 正常 |
| unavailableZone | 1 | 不可用 |

## 4. NodeSet 结构

NodeSet 是节点的逻辑分组，用于故障隔离。

```go
// topology.go:975
type nodeSet struct {
    ID                uint64           // NodeSet ID
    Capacity          int              // 容量（最大节点数）
    zoneName          string           // 所属 Zone
    metaNodes         *sync.Map        // MetaNode 映射
    dataNodes         *sync.Map        // DataNode 映射
    
    // 节点选择器
    dataNodeSelector  NodeSelector     // DataNode 选择器
    metaNodeSelector  NodeSelector     // MetaNode 选择器
    
    // 下线相关
    decommissionDataPartitionList *DecommissionDataPartitionList
    decommissionParallelLimit     int32
    manualDecommissionDiskList    *DecommissionDiskList
    autoDecommissionDiskList      *DecommissionDiskList
}
```

### NodeSet 初始化

```go
// topology.go:1010
func newNodeSet(c *Cluster, id uint64, cap int, zoneName string) *nodeSet {
    ns := &nodeSet{
        ID:                   id,
        Capacity:             cap,           // 默认 18
        zoneName:             zoneName,
        metaNodes:            new(sync.Map),
        dataNodes:            new(sync.Map),
        dataNodeSelector:     NewNodeSelector(DefaultNodeSelectorName, DataNodeType),
        metaNodeSelector:     NewNodeSelector(DefaultNodeSelectorName, MetaNodeType),
    }
    return ns
}
```

## 5. NodeSetGroup 结构

NodeSetGroup 用于跨 Zone 的故障域管理。

```go
// topology.go:226
type nodeSetGroup struct {
    ID            uint64           // 组 ID
    domainId      uint64           // 域 ID
    nodeSets      []*nodeSet       // NodeSet 列表
    nodeSetsIds   []uint64         // NodeSet ID 列表
    status        uint8            // 状态
    nsgInnerIndex int              // 内部索引
}
```

## 6. DomainManager 结构

DomainManager 管理故障域。

```go
// topology.go:261
type DomainManager struct {
    c                     *Cluster
    init                  bool
    domainNodeSetGrpVec   []*DomainNodeSetGrpManager  // 域管理器列表
    domainId2IndexMap     map[uint64]int              // 域ID -> 索引
    ZoneName2DomainIdMap  map[string]uint64           // Zone -> 域ID
    excludeZoneListDomain map[string]int              // 排除的 Zone
    dataRatioLimit        float64                     // 数据使用率限制
}
```

## 7. 拓扑层级关系

```
Cluster
    │
    └── topology
            │
            ├── Zone 1 ─────────────┐
            │       │               │
            │       ├── NodeSet 1   │
            │       │     ├── DataNode A
            │       │     ├── DataNode B
            │       │     ├── MetaNode A
            │       │     └── MetaNode B
            │       │               │
            │       └── NodeSet 2   │
            │             ├── DataNode C
            │             └── MetaNode C
            │                       │
            └── Zone 2 ────────────┘
                    │
                    └── NodeSet 3
                          ├── DataNode D
                          └── MetaNode D
```

## 8. 节点注册流程

当 DataNode/MetaNode 注册时：

```
节点注册请求
       │
       ├── 1. 获取/创建 Zone
       │       └── topology.getZone(zoneName)
       │           └── 不存在则创建 newZone()
       │
       ├── 2. 获取/创建 NodeSet
       │       └── zone.getNodeSet()
       │           └── 不存在则创建 newNodeSet()
       │
       ├── 3. 将节点加入 NodeSet
       │       └── nodeSet.putDataNode() / putMetaNode()
       │
       └── 4. 更新拓扑缓存
                └── topology.putDataNodeToCache()
```

## 9. 节点选择策略

### 9.1 选择 DataNode

```go
// 从 NodeSet 中选择可用的 DataNode
func (ns *nodeSet) getAvailDataNodeHosts(excludeHosts []string, 
                                          replicaNum int) (hosts []string, 
                                                           peers []proto.Peer, 
                                                           err error) {
    // 1. 过滤出可写节点
    // 2. 使用 NodeSelector 选择节点
    // 3. 排除已使用的节点
    // 4. 返回选中的节点列表
}
```

### 9.2 选择 MetaNode

```go
// 从 NodeSet 中选择可用的 MetaNode
func (ns *nodeSet) getAvailMetaNodeHosts(excludeHosts []string, 
                                          replicaNum int) (hosts []string, 
                                                           peers []proto.Peer, 
                                                           err error)
```

## 10. 故障域策略

故障域确保副本分布在不同的 Zone/NodeSet 中。

```
3 副本分布策略:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   策略 1: 3 Zone 模式                                        │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│   │  Zone 1  │  │  Zone 2  │  │  Zone 3  │                 │
│   │ Replica1 │  │ Replica2 │  │ Replica3 │                 │
│   └──────────┘  └──────────┘  └──────────┘                 │
│                                                             │
│   策略 2: 2+1 模式                                           │
│   ┌──────────┐  ┌──────────┐                               │
│   │  Zone 1  │  │  Zone 2  │                               │
│   │ Replica1 │  │ Replica2 │                               │
│   │          │  │ Replica3 │                               │
│   └──────────┘  └──────────┘                               │
│                                                             │
│   策略 3: 单 Zone 模式                                       │
│   ┌──────────────────────────────────────────────┐         │
│   │                    Zone 1                     │         │
│   │  ┌────────┐  ┌────────┐  ┌────────┐         │         │
│   │  │NodeSet1│  │NodeSet2│  │NodeSet3│         │         │
│   │  │Replica1│  │Replica2│  │Replica3│         │         │
│   │  └────────┘  └────────┘  └────────┘         │         │
│   └──────────────────────────────────────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 11. NodeSet 配置

```go
const (
    defaultNodeSetCapacity = 18   // 默认 NodeSet 容量
    defaultNodeSetGrpBatchCnt = 3 // 默认每批创建的 NodeSetGroup 数量
    defaultFaultDomainZoneCnt = 3 // 默认故障域 Zone 数量
    defaultReplicaNum = 3         // 默认副本数
)
```

## 12. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| topology 结构 | topology.go:42 |
| Zone 结构 | topology.go:1518 |
| nodeSet 结构 | topology.go:975 |
| newNodeSet | topology.go:1010 |
| nodeSetGroup | topology.go:226 |
| DomainManager | topology.go:261 |
| putDataNode | topology.go:123 |
| putMetaNode | topology.go:150 |

---

## 下一步
- Layer 4: 节点管理

*创建时间：2026-04-12*
