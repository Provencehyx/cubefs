# Layer 6: Raft 一致性

## 1. DataNode 中 Raft 的角色

与 MetaNode 不同，DataNode 的 Raft 主要用于**控制面**操作，而非数据写入。

```
┌─────────────────────────────────────────────────────────────┐
│                    DataPartition                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────┐   ┌─────────────────────────┐ │
│   │      数据面 (Data)       │   │    控制面 (Control)     │ │
│   ├─────────────────────────┤   ├─────────────────────────┤ │
│   │                         │   │                         │ │
│   │   链式复制              │   │      Raft               │ │
│   │   - Extent 写入         │   │   - 随机写同步           │ │
│   │   - Extent 读取         │   │   - 成员变更             │ │
│   │   - 顺序写入            │   │   - Leader 选举         │ │
│   │                         │   │   - 元数据同步           │ │
│   └─────────────────────────┘   └─────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. dataPartitionCfg 配置

```go
// partition_raft.go:42
type dataPartitionCfg struct {
    VolName       string              // 卷名
    ClusterID     string              // 集群 ID
    PartitionID   uint64              // 分区 ID
    PartitionSize int                 // 分区大小
    PartitionType int                 // 分区类型
    Peers         []proto.Peer        // Raft 成员
    Hosts         []string            // 主机列表
    NodeID        uint64              // 节点 ID
    RaftStore     raftstore.RaftStore // Raft 存储
    ReplicaNum    int                 // 副本数
}
```

## 3. 启动 Raft

```go
// partition_raft.go:85
func (dp *DataPartition) StartRaft(isLoad bool) (err error) {
    // 1. 获取 Raft 端口
    heartbeatPort, replicaPort, err := dp.raftPort()
    
    // 2. 构建 Peer 地址列表
    var peers []raftstore.PeerAddress
    for _, peer := range dp.config.Peers {
        rp := raftstore.PeerAddress{
            Peer:          raftproto.Peer{ID: peer.ID},
            Address:       addr,
            HeartbeatPort: heartbeatPort,
            ReplicaPort:   replicaPort,
        }
        peers = append(peers, rp)
    }
    
    // 3. 创建 Raft 分区配置
    pc := &raftstore.PartitionConfig{
        ID:      dp.partitionID,
        Applied: dp.appliedID,
        Peers:   peers,
        SM:      dp,          // DataPartition 实现状态机接口
        WalPath: dp.path,
    }
    
    // 4. 创建 Raft 分区
    dp.raftPartition, err = dp.config.RaftStore.CreatePartition(pc)
    
    return
}
```

## 4. 状态机接口实现

DataPartition 实现 `raft.StateMachine` 接口：

```go
// partition_raftfsm.go

// Apply 应用 Raft 日志
func (dp *DataPartition) Apply(command []byte, index uint64) (resp interface{}, err error)

// ApplyMemberChange 应用成员变更
func (dp *DataPartition) ApplyMemberChange(confChange *raftproto.ConfChange, index uint64) (resp interface{}, err error)

// Snapshot 创建快照
func (dp *DataPartition) Snapshot() (raftproto.Snapshot, error)

// ApplySnapshot 应用快照
func (dp *DataPartition) ApplySnapshot(peers []raftproto.Peer, iter raftproto.SnapIterator) error

// HandleFatalEvent 处理致命事件
func (dp *DataPartition) HandleFatalEvent(err *raft.FatalError)

// HandleLeaderChange 处理 Leader 变更
func (dp *DataPartition) HandleLeaderChange(leader uint64)
```

## 5. Apply 流程

```go
// partition_raftfsm.go:37
func (dp *DataPartition) Apply(command []byte, index uint64) (resp interface{}, err error) {
    // 1. 解析版本号
    buff := bytes.NewBuffer(command)
    binary.Read(buff, binary.BigEndian, &version)
    
    // 2. 根据版本处理
    if version != BinaryMarshalMagicVersion {
        // 处理版本操作或修复状态操作
        opItem, _ := UnmarshalRaftCmd(command)
        if opItem.Op == proto.OpVersionOp {
            dp.fsmVersionOp(opItem)
        } else if opItem.Op == proto.OpSetRepairingStatus {
            dp.fsmSetRepairingStatusOp(opItem)
        }
        return
    }
    
    // 3. 应用随机写
    if index > dp.metaAppliedID {
        resp, err = dp.ApplyRandomWrite(command, index)
    }
    
    return
}
```

## 6. 成员变更

```go
// partition_raftfsm.go:70
func (dp *DataPartition) ApplyMemberChange(confChange *raftproto.ConfChange, index uint64) (resp interface{}, err error) {
    
    switch confChange.Type {
    case raftproto.ConfAddNode:
        // 添加节点
        req := &proto.AddDataPartitionRaftMemberRequest{}
        json.Unmarshal(confChange.Context, req)
        dp.addRaftNode(req, index)
        
    case raftproto.ConfRemoveNode:
        // 移除节点
        req := &proto.RemoveDataPartitionRaftMemberRequest{}
        json.Unmarshal(confChange.Context, req)
        dp.removeRaftNode(req, index)
        
    case raftproto.ConfUpdateNode:
        // 暂不支持
    }
    
    // 持久化元数据
    dp.PersistMetadata()
    
    return
}
```

## 7. 随机写与 Raft

随机写需要通过 Raft 保证一致性：

```
客户端随机写请求
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│                     Leader DataNode                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. 接收随机写请求                                        │
│         │                                                │
│         ▼                                                │
│  2. 封装为 Raft Command                                   │
│         │                                                │
│         ▼                                                │
│  3. raftPartition.Submit(command)                        │
│         │                                                │
│         ▼                                                │
│  4. Raft 复制到 Followers                                 │
│         │                                                │
│         ▼                                                │
│  5. Apply() 被调用                                        │
│         │                                                │
│         ▼                                                │
│  6. ApplyRandomWrite() 执行实际写入                       │
│         │                                                │
│         ▼                                                │
│  7. 返回响应                                              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## 8. 顺序写 vs 随机写

| 写入类型 | 一致性机制 | 说明 |
|----------|-----------|------|
| 顺序写 (Append) | 链式复制 | 性能优先，数据追加 |
| 随机写 (Random) | Raft | 一致性优先，覆盖写 |

```
顺序写:
  Client → DN1 → DN2 → DN3  (链式复制)
  
随机写:
  Client → Leader (Raft) → Followers  (Raft 复制)
```

## 9. ApplyID 管理

```go
// DataPartition 中的 ApplyID 字段
type DataPartition struct {
    appliedID      uint64    // 已应用的 Raft 日志 ID
    metaAppliedID  uint64    // 持久化时的 Apply ID
    minAppliedID   uint64
    maxAppliedID   uint64
}

// 更新 ApplyID
func (dp *DataPartition) uploadApplyID(index uint64) {
    atomic.StoreUint64(&dp.appliedID, index)
}

// 持久化 ApplyID
dp.PersistApplyIdChan <- PersistApplyIdRequest{}
```

## 10. DataNode vs MetaNode Raft 对比

| 维度 | DataNode | MetaNode |
|------|----------|----------|
| 主要用途 | 随机写、成员变更 | 所有元数据操作 |
| 数据写入 | 链式复制（主） + Raft（辅） | 纯 Raft |
| 快照内容 | Extent 信息 | inode/dentry 全量 |
| Apply 频率 | 较低（仅随机写） | 很高（所有写操作） |

## 11. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| dataPartitionCfg | partition_raft.go:42 |
| StartRaft | partition_raft.go:85 |
| Apply | partition_raftfsm.go:37 |
| ApplyMemberChange | partition_raftfsm.go:70 |
| ApplyRandomWrite | partition_op_by_raft.go |

---

## 下一步
- Layer 7: 数据修复

*创建时间：2026-04-12*
