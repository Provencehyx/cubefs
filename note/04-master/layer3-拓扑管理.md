# Layer 3: 拓扑与故障域设计

## 1. 为什么需要拓扑管理？

### 1.1 分布式存储的核心问题

**数据放在哪？** 这个看似简单的问题背后是复杂的权衡：

| 考量 | 要求 | 冲突点 |
|------|------|--------|
| 可靠性 | 副本分散在不同故障域 | 跨机架/机房复制延迟高 |
| 性能 | 副本尽量靠近 | 太近则一损俱损 |
| 均衡 | 负载均匀分布 | 新节点和老节点容量不同 |

### 1.2 CubeFS 的三层拓扑

```
┌─────────────────────────────────────────────────────────────┐
│                       Cluster (集群)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                    Zone (可用区)                      │  │
│   │         对应物理概念：机房 / 可用区 / 机架组           │  │
│   │                                                       │  │
│   │   ┌───────────────────────────────────────────────┐  │  │
│   │   │               NodeSet (节点组)                 │  │  │
│   │   │          逻辑分组，限制副本分布范围             │  │  │
│   │   │                                               │  │  │
│   │   │   ┌─────┐  ┌─────┐  ┌─────┐     容量上限      │  │  │
│   │   │   │Node │  │Node │  │Node │ ... (默认18)     │  │  │
│   │   │   └─────┘  └─────┘  └─────┘                   │  │  │
│   │   └───────────────────────────────────────────────┘  │  │
│   │                                                       │  │
│   │   ┌───────────────────────────────────────────────┐  │  │
│   │   │               NodeSet 2                        │  │  │
│   │   │              ...                               │  │  │
│   │   └───────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                    Zone 2                            │  │
│   │                    ...                               │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Zone 的设计

### 2.1 Zone 是什么？

```go
// topology.go:1518
type Zone struct {
    name            string
    status          int              // 可用/不可用
    dataNodes       *sync.Map        // 本 Zone 的 DataNode
    metaNodes       *sync.Map        // 本 Zone 的 MetaNode
    nodeSetMap      map[uint64]*nodeSet  // 本 Zone 的 NodeSet
    
    // Selector：选择策略
    dataNodesetSelector     NodesetSelector
    metaNodesetSelector     NodesetSelector
}
```

**Zone 对应的物理概念**：
- 小规模部署：一个机架
- 中规模部署：一个机房
- 大规模部署：一个可用区（AZ）

### 2.2 跨 Zone 副本分布

**3 副本的理想分布**：

```
场景 A：3 个 Zone
  Zone1: Replica1
  Zone2: Replica2  
  Zone3: Replica3
  → 任意一个 Zone 故障，数据仍可用

场景 B：2 个 Zone
  Zone1: Replica1, Replica2
  Zone2: Replica3
  → Zone2 故障可恢复，Zone1 故障丢数据！

场景 C：1 个 Zone
  Zone1: Replica1, Replica2, Replica3
  → Zone 故障 = 全部丢失，但延迟最低
```

**CubeFS 的策略**：优先跨 Zone 分布，Zone 不足时允许同 Zone（但不同 NodeSet）。

---

## 3. NodeSet 的设计（核心创新）

### 3.1 为什么需要 NodeSet？

**问题场景**：100 个节点的集群，3 副本。

如果没有 NodeSet：
```
节点1 挂了 → 它的数据分散在其他 99 个节点
           → 恢复时需要从 99 个节点读取
           → 恢复完成后数据更分散
           → 下次故障影响范围更大（连锁反应）
```

有了 NodeSet（假设容量 18）：
```
节点1 挂了 → 它的数据只在同 NodeSet 的 17 个节点中
           → 恢复只涉及这 17 个节点
           → 影响范围可控
```

### 3.2 NodeSet 结构

```go
// topology.go:975
type nodeSet struct {
    ID        uint64
    Capacity  int           // 容量上限，默认 18
    zoneName  string        // 所属 Zone
    
    dataNodes *sync.Map     // DataNode 集合
    metaNodes *sync.Map     // MetaNode 集合
    
    // 节点选择器
    dataNodeSelector NodeSelector
    metaNodeSelector NodeSelector
}
```

### 3.3 NodeSet 容量的权衡

```
容量太小（如 6）：
  - 副本分布受限，可能找不到足够节点
  - NodeSet 数量多，管理复杂

容量太大（如 100）：
  - 故障恢复涉及节点多
  - 接近"无 NodeSet"的问题

默认 18 的考量：
  - 3 副本时，18 节点可容纳大量分区
  - 单节点故障只影响 1/18 ≈ 5.5% 的数据
  - 恢复时 17 个节点并行，速度快
```

### 3.4 副本选择流程

```
创建 3 副本的 DataPartition
        │
        ▼
选择 Zone（优先跨 Zone）
        │
        ▼
┌───────┴───────┐
│  在每个 Zone  │
│  选择 NodeSet │
└───────┬───────┘
        │
        ▼
┌───────────────────────────────────────────────┐
│  在 NodeSet 内选择 Node                        │
│                                               │
│  getAvailDataNodeHosts(excludeHosts, 1)       │
│      │                                        │
│      ├── 过滤：已有副本的节点                  │
│      ├── 过滤：不可用节点                      │
│      ├── 过滤：磁盘空间不足                    │
│      └── 选择：根据策略（轮询/负载均衡/...）   │
└───────────────────────────────────────────────┘
```

---

## 4. 故障域（FaultDomain）

### 4.1 什么是故障域？

**故障域**：共享同一故障点的一组资源。

```
┌─────────────────────────────────────────────────────────────┐
│                      故障域层次                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   机房断电 ─────────────────────────────────────────────┐   │
│                                                         │   │
│   机架交换机故障 ───────────────────────────────┐       │   │
│                                                 │       │   │
│   服务器故障 ───────────────────────────┐       │       │   │
│                                         │       │       │   │
│   磁盘故障 ─────────────────────┐       │       │       │   │
│                                 │       │       │       │   │
│   影响范围:                     小      中      大     巨大  │
│                                 │       │       │       │   │
│   发生频率:                     高      中      低      极低 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 CubeFS 的故障域策略

```go
// cluster.go
type ClusterTopoSubItem struct {
    FaultDomain   bool           // 是否启用故障域
    domainManager *DomainManager // 域管理器
}
```

**启用故障域时**：
- 副本必须跨 Zone 分布
- 每个 Zone 最多一个副本
- 创建分区时严格检查

**不启用时**：
- 允许同 Zone 不同 NodeSet
- 灵活性高，但可靠性降低

---

## 5. 节点选择策略

### 5.1 NodeSelector 接口

```go
// node_selector.go
type NodeSelector interface {
    Select(ns *nodeSet, excludeHosts []string, replicaNum int) ([]string, []proto.Peer, error)
}
```

### 5.2 内置策略

| 策略 | 适用场景 | 特点 |
|------|----------|------|
| **RoundRobin** | 通用 | 轮询选择，简单均衡 |
| **CarryWeight** | 容量差异大 | 按剩余空间加权 |
| **AvailableSpaceFirst** | 快速填充 | 优先选择空间大的 |
| **Straw** | 一致性哈希场景 | 类似 CRUSH 算法 |

### 5.3 选择过程

```go
// node_selector.go:597
func (ns *nodeSet) getAvailDataNodeHosts(excludeHosts []string, replicaNum int) (hosts []string, peers []proto.Peer, err error) {
    ns.nodeSelectLock.Lock()
    defer ns.nodeSelectLock.Unlock()
    
    // 调用配置的选择器
    return ns.dataNodeSelector.Select(ns, excludeHosts, replicaNum)
}
```

---

## 6. 关键设计决策

### 6.1 为什么节点注册时指定 Zone？

```
节点启动配置：
  zoneName: "zone1"   ← 节点自己声明属于哪个 Zone

为什么不让 Master 自动分配？
  - 物理位置只有运维知道
  - 机房、机架信息无法自动发现
  - 错误的 Zone 分配会导致假的故障隔离
```

### 6.2 NodeSet 是自动创建的

```go
// 节点注册时
func (zone *Zone) putDataNode(dataNode *DataNode) {
    // 找一个有空位的 NodeSet
    ns := zone.findAvailNodeSet()
    if ns == nil {
        // 没有空位，创建新 NodeSet
        ns = zone.createNodeSet()
    }
    ns.putDataNode(dataNode)
}
```

**自动创建的好处**：运维不需要手动管理 NodeSet，只需正确设置 Zone。

### 6.3 NodeSet 满了怎么办？

```
NodeSet1 (18/18 满)
    │
    ▼
创建 NodeSet2，新节点加入这里
    │
    ▼
问题：旧分区的副本还在 NodeSet1
      新分区可能跨 NodeSet1 和 NodeSet2
```

**解决**：CubeFS 允许分区的副本分布在同 Zone 的不同 NodeSet（如果 Zone 内 NodeSet 足够多）。

---

## 7. 值得注意的点

### 7.1 Zone 名称的重要性

```
错误示例：
  节点 A (实际在机房1) → zoneName: "default"
  节点 B (实际在机房2) → zoneName: "default"
  
结果：Master 认为它们在同一 Zone，可能把 3 副本都放这里
     机房1 断电 → 数据丢失！
```

**Zone 名称必须反映真实物理拓扑。**

### 7.2 NodeSet 容量调整

```go
// 可以动态调整（但要谨慎）
defaultNodeSetCapacity = 18

调大：允许更多节点加入现有 NodeSet
调小：不影响已有 NodeSet，只影响新建的
```

### 7.3 跨 NodeSet 迁移的开销

```
原 NodeSet 内恢复：
  └─ 数据在本 NodeSet 17 个节点间复制
  └─ 网络流量局限于 NodeSet

跨 NodeSet 迁移：
  └─ 数据需要跨 NodeSet 复制
  └─ 可能涉及更多节点
  └─ 网络流量更大
```

---

## 8. 拓扑结构示例

```
Cluster: "production"
│
├── Zone: "beijing-zone1"
│   ├── NodeSet: 1 (capacity: 18)
│   │   ├── DataNode: 192.168.1.1
│   │   ├── DataNode: 192.168.1.2
│   │   └── ... (共 18 个)
│   │
│   └── NodeSet: 2 (capacity: 18)
│       ├── DataNode: 192.168.1.19
│       └── ...
│
├── Zone: "beijing-zone2"
│   └── NodeSet: 3 (capacity: 18)
│       ├── DataNode: 192.168.2.1
│       └── ...
│
└── Zone: "shanghai-zone1"
    └── NodeSet: 4 (capacity: 18)
        ├── DataNode: 10.0.1.1
        └── ...
```

**3 副本分布示例**：
- Replica1 → beijing-zone1 / NodeSet1 / 192.168.1.1
- Replica2 → beijing-zone2 / NodeSet3 / 192.168.2.5
- Replica3 → shanghai-zone1 / NodeSet4 / 10.0.1.3

---

## 9. 核心代码索引

| 组件 | 位置 | 说明 |
|------|------|------|
| `topology` | topology.go:42 | 拓扑管理器 |
| `Zone` | topology.go:1518 | 可用区 |
| `nodeSet` | topology.go:975 | 节点组 |
| `NodeSelector` | node_selector.go | 节点选择接口 |
| `getAvailDataNodeHosts` | node_selector.go:597 | 选择可用 DataNode |
| `putDataNode` | topology.go:122 | 添加节点到拓扑 |

---

## 下一步

→ [Layer 4: 节点管理](layer4-节点管理.md) - DataNode/MetaNode 的注册与心跳

---
*更新时间: 2026-04-23*
