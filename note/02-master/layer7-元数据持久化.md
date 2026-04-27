# Layer 7: 元数据持久化

## 1. 持久化架构

Master 通过 Raft + RocksDB 实现集群元数据的强一致性持久化。

```
┌─────────────────────────────────────────────────────────────┐
│                      Master 持久化架构                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   业务操作 (创建卷/分区/节点注册等)                           │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                  RaftCmd                             │  │
│   │    ┌──────┬──────┬──────────────────────────────┐   │  │
│   │    │  Op  │  K   │           V                  │   │  │
│   │    │ 操作 │  键  │          值                  │   │  │
│   │    └──────┴──────┴──────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────┘  │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                    Raft                              │  │
│   │         (Leader 复制到 Follower)                     │  │
│   └─────────────────────────────────────────────────────┘  │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │               MetadataFsm.Apply()                    │  │
│   │                 (状态机应用)                          │  │
│   └─────────────────────────────────────────────────────┘  │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                   RocksDB                            │  │
│   │                 (持久化存储)                          │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. MetadataFsm 结构

MetadataFsm 实现 Raft 状态机接口。

```go
// metadata_fsm.go:45
type MetadataFsm struct {
    store               *raftstore.RocksDBStore  // RocksDB 存储
    rs                  *raft.RaftServer         // Raft 服务器
    applied             uint64                   // 已应用的日志索引
    retainLogs          uint64                   // 保留的日志数量
    
    leaderChangeHandler raftLeaderChangeHandler  // Leader 变更处理
    peerChangeHandler   raftPeerChangeHandler    // Peer 变更处理
    snapshotHandler     raftApplySnapshotHandler // 快照处理
    UserAppCmdHandler   raftUserCmdApplyHandler  // 用户命令处理
    
    onSnapshot          bool                     // 是否正在快照
    raftLk              sync.Mutex
}
```

## 3. RaftCmd 结构

Raft 命令封装要持久化的数据。

```go
type RaftCmd struct {
    Op uint32    // 操作类型
    K  string    // 键
    V  []byte    // 值 (序列化后的数据)
}

// 序列化
func (m *RaftCmd) Marshal() ([]byte, error)

// 反序列化
func (m *RaftCmd) Unmarshal(data []byte) error
```

## 4. 操作类型

```go
// metadata_fsm_op.go
const (
    // 集群操作
    opSyncPutCluster = 0x01
    
    // 卷操作
    opSyncAddVol     = 0x02
    opSyncUpdateVol  = 0x03
    opSyncDeleteVol  = 0x04
    
    // 数据分区操作
    opSyncAddDataPartition    = 0x05
    opSyncUpdateDataPartition = 0x06
    opSyncDeleteDataPartition = 0x07
    
    // 元数据分区操作
    opSyncAddMetaPartition    = 0x08
    opSyncUpdateMetaPartition = 0x09
    opSyncDeleteMetaPartition = 0x0A
    
    // DataNode 操作
    opSyncAddDataNode    = 0x0B
    opSyncDeleteDataNode = 0x0C
    opSyncUpdateDataNode = 0x0D
    
    // MetaNode 操作
    opSyncAddMetaNode    = 0x0E
    opSyncDeleteMetaNode = 0x0F
    opSyncUpdateMetaNode = 0x10
    
    // NodeSet 操作
    opSyncAddNodeSet    = 0x11
    opSyncUpdateNodeSet = 0x12
    
    // 用户操作
    opSyncAddUserInfo    = 0x13
    opSyncDeleteUserInfo = 0x14
    opSyncUpdateUserInfo = 0x15
    
    // 批量操作
    opSyncBatchPut = 0x20
)
```

## 5. Value 结构

各类元数据序列化为 Value 结构存储。

### 5.1 clusterValue

```go
// metadata_fsm_op.go:37
type clusterValue struct {
    Name                string
    CreateTime          int64
    Threshold           float32
    DisableAutoAllocate bool
    FaultDomain         bool
    MaxDpCntLimit       uint64
    MaxMpCntLimit       uint64
    // ... 更多配置
}
```

### 5.2 volValue

```go
type volValue struct {
    ID              uint64
    Name            string
    ReplicaNum      uint8
    DpReplicaNum    uint8
    Status          uint8
    DataPartitionSize uint64
    Capacity        uint64
    Owner           string
    ZoneName        string
    // ... 更多字段
}
```

### 5.3 dataPartitionValue

```go
type dataPartitionValue struct {
    PartitionID      uint64
    ReplicaNum       uint8
    Hosts            []string
    Peers            []proto.Peer
    Status           int8
    VolID            uint64
    VolName          string
    PartitionType    int
    // ... 更多字段
}
```

### 5.4 metaPartitionValue

```go
type metaPartitionValue struct {
    PartitionID uint64
    Start       uint64
    End         uint64
    Peers       []proto.Peer
    Hosts       []string
    VolID       uint64
    VolName     string
    ReplicaNum  uint8
}
```

### 5.5 dataNodeValue / metaNodeValue

```go
type dataNodeValue struct {
    ID        uint64
    NodeSetID uint64
    Addr      string
    ZoneName  string
}

type metaNodeValue struct {
    ID        uint64
    NodeSetID uint64
    Addr      string
    ZoneName  string
}
```

## 6. Apply 流程

```go
// metadata_fsm.go:109
func (mf *MetadataFsm) Apply(command []byte, index uint64) (resp interface{}, err error) {
    mf.raftLk.Lock()
    defer mf.raftLk.Unlock()
    
    // 1. 反序列化命令
    cmd := new(RaftCmd)
    cmd.Unmarshal(command)
    
    // 2. 准备写入数据
    cmdMap := make(map[string][]byte)
    cmdMap[cmd.K] = cmd.V
    cmdMap[applied] = []byte(strconv.FormatUint(index, 10))
    
    // 3. 根据操作类型处理
    switch cmd.Op {
    case opSyncDeleteDataNode, opSyncDeleteMetaNode, ...:
        // 删除操作
        mf.delKeyAndPutIndex(cmd.K, cmdMap)
    default:
        // 写入操作
        mf.store.BatchPut(cmdMap, true)
    }
    
    // 4. 更新已应用索引
    mf.applied = index
    
    // 5. 定期截断日志
    if mf.applied > 0 && (mf.applied % mf.retainLogs) == 0 {
        mf.rs.Truncate(GroupID, mf.applied)
    }
    
    return
}
```

## 7. 同步写入方法

```go
// 同步写入集群配置
func (c *Cluster) syncPutCluster() error {
    cv := newClusterValue(c)
    return c.submit(opSyncPutCluster, clusterPrefix, cv)
}

// 同步添加卷
func (c *Cluster) syncAddVol(vol *Vol) error {
    vv := newVolValue(vol)
    return c.submit(opSyncAddVol, volPrefix+vol.Name, vv)
}

// 同步添加数据分区
func (c *Cluster) syncAddDataPartition(dp *DataPartition) error {
    dpv := newDataPartitionValue(dp)
    return c.submit(opSyncAddDataPartition, dataPartitionPrefix+dpv.PartitionID, dpv)
}

// 通用提交方法
func (c *Cluster) submit(op uint32, key string, value interface{}) error {
    cmd := &RaftCmd{
        Op: op,
        K:  key,
        V:  json.Marshal(value),
    }
    _, err := c.partition.Submit(cmd.Marshal())
    return err
}
```

## 8. 数据恢复

Master 启动时从 RocksDB 恢复集群状态。

```go
// 恢复流程
func (c *Cluster) loadClusterValue() error {
    // 从 RocksDB 读取集群配置
    value, _ := c.fsm.store.Get(clusterPrefix)
    cv := &clusterValue{}
    json.Unmarshal(value, cv)
    c.applyClusterValue(cv)
}

func (c *Cluster) loadDataNodes() error {
    // 遍历所有 DataNode 记录
    c.fsm.store.Range(dataNodePrefix, func(key, value []byte) bool {
        dnv := &dataNodeValue{}
        json.Unmarshal(value, dnv)
        c.applyDataNodeValue(dnv)
        return true
    })
}

// 类似地恢复 MetaNode、Vol、DataPartition、MetaPartition 等
```

## 9. 快照

### 9.1 创建快照

```go
// metadata_fsm.go:200
func (mf *MetadataFsm) Snapshot() (proto.Snapshot, error) {
    mf.raftLk.Lock()
    defer mf.raftLk.Unlock()
    
    snapshot := mf.store.RocksDBSnapshot()
    iterator := mf.store.Iterator(snapshot)
    iterator.SeekToFirst()
    
    return &MetadataSnapshot{
        applied:  mf.applied,
        snapshot: snapshot,
        fsm:      mf,
        iterator: iterator,
    }, nil
}
```

### 9.2 应用快照

```go
// metadata_fsm.go:216
func (mf *MetadataFsm) ApplySnapshot(peers []proto.Peer, iterator proto.SnapIterator) error {
    mf.onSnapshot = true
    defer func() { mf.onSnapshot = false }()
    
    // 1. 清空现有数据
    mf.store.Clear()
    
    // 2. 应用快照数据
    for {
        data, err := iterator.Next()
        if err == io.EOF {
            break
        }
        
        cmd := &RaftCmd{}
        json.Unmarshal(data, cmd)
        mf.store.Put(cmd.K, cmd.V, false)
    }
    
    // 3. 刷盘
    mf.store.Flush()
    
    // 4. 触发恢复回调
    mf.snapshotHandler()
    
    return nil
}
```

## 10. Key 前缀

```go
const (
    clusterPrefix       = "#c"      // 集群配置
    volPrefix           = "#vol_"   // 卷
    dataNodePrefix      = "#dn_"    // DataNode
    metaNodePrefix      = "#mn_"    // MetaNode
    dataPartitionPrefix = "#dp_"    // DataPartition
    metaPartitionPrefix = "#mp_"    // MetaPartition
    nodeSetPrefix       = "#ns_"    // NodeSet
    userPrefix          = "#user_"  // 用户
)
```

## 11. 数据流转图

```
业务操作
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                    Cluster 方法                          │
│  (syncAddVol, syncAddDataPartition, ...)                │
└──────────────────────────────────────────────────────────┘
    │
    ├── 1. 构造 Value 结构
    │
    ├── 2. 序列化为 RaftCmd
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                  Raft Submit                             │
│              (提交到 Raft Leader)                        │
└──────────────────────────────────────────────────────────┘
    │
    ├── 3. Raft 复制到 Follower
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                 MetadataFsm.Apply()                      │
│                  (状态机应用)                             │
└──────────────────────────────────────────────────────────┘
    │
    ├── 4. 写入 RocksDB
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│                    RocksDB                               │
│                  (持久化存储)                             │
└──────────────────────────────────────────────────────────┘
```

## 12. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| MetadataFsm 结构 | metadata_fsm.go:45 |
| Apply | metadata_fsm.go:109 |
| Snapshot | metadata_fsm.go:200 |
| ApplySnapshot | metadata_fsm.go:216 |
| clusterValue | metadata_fsm_op.go:37 |
| volValue | metadata_fsm_op.go |
| dataPartitionValue | metadata_fsm_op.go |
| 操作常量 | metadata_fsm_op.go |

---

## 总结

Master 笔记完成，共 7 层：

1. **启动与配置** - Server 结构与启动流程
2. **集群管理** - Cluster 核心结构与定时任务
3. **拓扑管理** - Zone、NodeSet、故障域
4. **节点管理** - DataNode、MetaNode 管理
5. **卷管理** - Vol 创建/删除/扩容
6. **分区管理** - DataPartition、MetaPartition
7. **元数据持久化** - Raft + RocksDB

*创建时间：2026-04-12*
