# Layer 5: 卷管理

## 1. Vol 结构

Vol（卷）是 CubeFS 的核心逻辑单元，包含一组元数据分区和数据分区。

```go
// vol.go:142
type Vol struct {
    ID            uint64                // 卷 ID
    Name          string                // 卷名称
    Owner         string                // 所有者
    Status        uint8                 // 状态
    VolType       int                   // 卷类型
    zoneName      string                // Zone 名称
    user          *User                 // 用户
    createTime    int64                 // 创建时间
    description   string                // 描述
    
    // 副本配置
    dpReplicaNum      uint8             // DP 副本数
    mpReplicaNum      uint8             // MP 副本数
    dataPartitionSize uint64            // DP 大小 (byte)
    Capacity          uint64            // 容量 (GB)
    
    // 分区管理
    MetaPartitions    map[uint64]*MetaPartition  // MP 映射
    dataPartitions    *DataPartitionMap          // DP 管理器
    
    // 读取配置
    FollowerRead      bool              // Follower 读
    MetaFollowerRead  bool              // Meta Follower 读
    DirectRead        bool              // 直接读
    MaximallyRead     bool              // 最大化读
    
    // 嵌入的子结构
    TopoSubItem                         // 拓扑配置
    CacheSubItem                        // 缓存配置
    TxSubItem                           // 事务配置
    AuthenticSubItem                    // 认证配置
    VolDeletionSubItem                  // 删除配置
    
    // 其他管理器
    qosManager        *QosCtrlManager   // QoS 管理
    quotaManager      *MasterQuotaManager  // 配额管理
    VersionMgr        *VolVersionManager   // 版本管理
}
```

## 2. 卷状态

```go
const (
    VolStatusNormal       = 0   // 正常
    VolStatusMarkDelete   = 1   // 标记删除
)
```

## 3. 卷类型

```go
const (
    VolumeTypeNormal = 0   // 普通卷
    VolumeTypeCold   = 1   // 冷数据卷 (纠删码)
    VolumeTypeHot    = 2   // 热数据卷
)
```

## 4. 创建卷流程

```
创建卷请求
       │
       ├── 1. 参数校验
       │       ├── 卷名称格式
       │       ├── 副本数
       │       └── 容量
       │
       ├── 2. 检查卷是否已存在
       │
       ├── 3. 分配卷 ID
       │       └── c.idAlloc.allocateCommonID()
       │
       ├── 4. 创建 Vol 结构
       │       └── newVol(volValue)
       │
       ├── 5. 创建初始分区
       │       ├── 创建初始 MetaPartition
       │       │       └── vol.createMetaPartition()
       │       │
       │       └── 创建初始 DataPartition
       │               └── vol.createDataPartition()
       │
       ├── 6. 持久化卷信息
       │       └── c.syncAddVol(vol)
       │
       └── 7. 加入集群卷列表
                └── c.putVol(vol)
```

## 5. 卷初始化

```go
// vol.go:206
func newVol(vv volValue) (vol *Vol) {
    vol = &Vol{
        ID:             vv.ID, 
        Name:           vv.Name, 
        MetaPartitions: make(map[uint64]*MetaPartition),
    }
    
    vol.dataPartitions = newDataPartitionMap(vv.Name)
    vol.VersionMgr = newVersionMgr(vol)
    vol.dpReplicaNum = vv.DpReplicaNum
    vol.mpReplicaNum = vv.ReplicaNum
    vol.Owner = vv.Owner
    vol.dataPartitionSize = vv.DataPartitionSize
    vol.Capacity = vv.Capacity
    vol.FollowerRead = vv.FollowerRead
    vol.zoneName = vv.ZoneName
    
    // 初始化 QoS 管理器
    vol.initQosManager(limitQosVal)
    
    // 初始化配额管理器
    vol.quotaManager = &MasterQuotaManager{...}
    
    return
}
```

## 6. DataPartitionMap 结构

管理卷的所有数据分区。

```go
// data_partition_map.go
type DataPartitionMap struct {
    volName            string
    readWriteDataPartitions   []*DataPartition  // 可读写 DP
    partitionMap       map[uint64]*DataPartition // DP 映射
    lastLoadedIndex    uint64
    sync.RWMutex
}
```

## 7. 创建数据分区

```go
// 创建数据分区
func (vol *Vol) createDataPartition(c *Cluster, preload bool) (dp *DataPartition, err error) {
    // 1. 分配分区 ID
    partitionID, err := c.idAlloc.allocateDataPartitionID()
    
    // 2. 选择节点
    hosts, peers, err := c.getHostFromNormalZone(...)
    
    // 3. 创建分区结构
    dp = newDataPartition(partitionID, vol.dpReplicaNum, vol.Name, vol.ID, ...)
    dp.Hosts = hosts
    dp.Peers = peers
    
    // 4. 向节点发送创建任务
    tasks := dp.createTaskToCreateDataPartition(...)
    c.addDataNodeTasks(tasks)
    
    // 5. 持久化分区信息
    c.syncAddDataPartition(dp)
    
    // 6. 加入卷的分区列表
    vol.dataPartitions.put(dp)
    
    return
}
```

## 8. 创建元数据分区

```go
// 创建元数据分区
func (vol *Vol) createMetaPartition(c *Cluster, start, end uint64) (mp *MetaPartition, err error) {
    // 1. 分配分区 ID
    partitionID, err := c.idAlloc.allocateMetaPartitionID()
    
    // 2. 选择节点
    hosts, peers, err := c.getHostFromNormalZone(...)
    
    // 3. 创建分区结构
    mp = newMetaPartition(partitionID, start, end, vol.mpReplicaNum, vol.Name, vol.ID, ...)
    mp.setHosts(hosts)
    mp.setPeers(peers)
    
    // 4. 向节点发送创建任务
    tasks := mp.buildNewMetaPartitionTasks(...)
    c.addMetaNodeTasks(tasks)
    
    // 5. 持久化分区信息
    c.syncAddMetaPartition(mp)
    
    // 6. 加入卷的分区列表
    vol.addMetaPartition(mp)
    
    return
}
```

## 9. 删除卷流程

```
删除卷请求
       │
       ├── 1. 检查卷是否存在
       │
       ├── 2. 检查删除锁定时间
       │       └── 必须超过 DeleteLockTime
       │
       ├── 3. 标记卷为删除状态
       │       └── vol.Status = VolStatusMarkDelete
       │
       ├── 4. 删除所有分区
       │       ├── 删除所有 DataPartition
       │       │       └── 向节点发送删除任务
       │       │
       │       └── 删除所有 MetaPartition
       │               └── 向节点发送删除任务
       │
       ├── 5. 从集群中移除卷
       │       └── c.deleteVol(volName)
       │
       └── 6. 持久化删除操作
                └── c.syncDeleteVol(vol)
```

## 10. 卷扩容

```go
// 扩容卷（增加容量）
func (vol *Vol) expandCapacity(capacity uint64) {
    vol.Capacity = capacity
    // 更新后会触发自动创建新的 DP
}

// 检查是否需要创建新 DP
func (vol *Vol) checkAutoDataPartitionCreation(c *Cluster) {
    // 如果可写 DP 数量不足，自动创建新的 DP
    if vol.needCreateDataPartition() {
        vol.createDataPartition(c, false)
    }
}
```

## 11. 卷相关配置

```go
// 默认配置
const (
    defaultDataPartitionSize = 120 * 1024 * 1024 * 1024  // 120GB
    defaultMetaPartitionInodeIDStep = 1 << 24            // 16M
    defaultMaxMetaPartitionInodeID = 1 << 63             // 最大 inode ID
)

// QoS 配置
type qosArgs struct {
    qosEnable     bool
    diskQosEnable bool
    iopsRVal      uint64   // 读 IOPS 限制
    iopsWVal      uint64   // 写 IOPS 限制
    flowRVal      uint64   // 读流量限制
    flowWVal      uint64   // 写流量限制
}
```

## 12. 卷视图缓存

为了提高查询效率，Master 缓存卷的分区视图。

```go
// 获取数据分区视图
func (vol *Vol) getDataPartitionsView() (body []byte, err error) {
    if len(vol.viewCache) == 0 {
        vol.updateViewCache(c)
    }
    return vol.viewCache, nil
}

// 更新视图缓存
func (vol *Vol) updateViewCache(c *Cluster) {
    view := &proto.DataPartitionsView{}
    // 遍历所有 DP 构建视图
    for _, dp := range vol.dataPartitions.partitionMap {
        view.DataPartitions = append(view.DataPartitions, dp.toView())
    }
    // 序列化并缓存
    vol.viewCache, _ = json.Marshal(view)
}
```

## 13. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| Vol 结构 | vol.go:142 |
| newVol | vol.go:206 |
| DataPartitionMap | data_partition_map.go |
| createDataPartition | vol.go |
| createMetaPartition | vol.go |
| 视图缓存 | vol.go |

---

## 下一步
- Layer 6: 分区管理

*创建时间：2026-04-12*
