# Layer 1: 启动与配置

## 1. Server 结构

Master 的核心入口是 `Server` 结构。

```go
// server.go:103
type Server struct {
    id              uint64              // Master 节点 ID
    clusterName     string              // 集群名称
    ip              string              // IP 地址
    port            string              // 监听端口
    logDir          string              // 日志目录
    walDir          string              // Raft WAL 目录
    storeDir        string              // RocksDB 存储目录
    
    config          *clusterConfig      // 集群配置
    cluster         *Cluster            // 集群管理器
    user            *User               // 用户管理
    
    // Raft 相关
    rocksDBStore    *raftstore_db.RocksDBStore  // RocksDB 存储
    raftStore       raftstore.RaftStore         // Raft 存储
    fsm             *MetadataFsm                // Raft 状态机
    partition       raftstore.Partition         // Raft 分区
    
    leaderInfo      *LeaderInfo         // Leader 信息
    reverseProxy    *httputil.ReverseProxy  // 反向代理（转发到 Leader）
    apiServer       *http.Server        // HTTP API 服务器
}
```

## 2. 启动流程

```
Server.Start(cfg)
       │
       ├── 1. 加载配置
       │       ├── m.config = newClusterConfig()
       │       └── m.checkConfig(cfg)
       │
       ├── 2. 初始化 RocksDB
       │       └── raftstore_db.NewRocksDBStoreAndRecovery()
       │
       ├── 3. 创建 Raft Server
       │       ├── m.createRaftServer(cfg)
       │       │       ├── raftstore.NewRaftStore()
       │       │       ├── m.initFsm()
       │       │       └── m.raftStore.CreatePartition()
       │       │
       │       └── 注册状态机处理器
       │               ├── LeaderChangeHandler
       │               ├── PeerChangeHandler
       │               ├── ApplySnapshotHandler
       │               └── RaftUserCmdApplyHandler
       │
       ├── 4. 初始化 Cluster
       │       └── m.initCluster()
       │               └── newCluster()
       │
       ├── 5. 初始化 User
       │       └── m.initUser()
       │
       ├── 6. 启动定时任务
       │       └── m.cluster.scheduleTask()
       │
       ├── 7. 启动 HTTP 服务
       │       └── m.startHTTPService()
       │
       └── 8. 启动监控
                └── metricsService.start()
```

## 3. 核心配置项

```go
// server.go 配置常量
const (
    ClusterName     = "clusterName"     // 集群名
    ID              = "id"              // 节点 ID
    IP              = "ip"              // IP 地址
    Port            = "port"            // 端口
    LogDir          = "logDir"          // 日志目录
    WalDir          = "walDir"          // WAL 目录
    StoreDir        = "storeDir"        // 存储目录
    GroupID         = 1                 // Raft Group ID（固定为 1）
)
```

### 配置示例

```json
{
    "clusterName": "cubefs-cluster",
    "id": "1",
    "ip": "192.168.1.1",
    "port": "17010",
    "peers": "1:192.168.1.1:17010,2:192.168.1.2:17010,3:192.168.1.3:17010",
    "logDir": "/var/log/cubefs/master",
    "walDir": "/var/lib/cubefs/master/raft",
    "storeDir": "/var/lib/cubefs/master/rocksdb",
    "retainLogs": "20000"
}
```

## 4. clusterConfig 配置

```go
// config.go
type clusterConfig struct {
    // 节点配置
    nodeSetCapacity    int      // NodeSet 容量（默认 18）
    
    // Raft 配置
    peers              []raftstore.PeerAddress  // Raft peers
    heartbeatPort      int64    // 心跳端口
    replicaPort        int64    // 复制端口
    
    // 分区配置
    MissingDataPartitionInterval int64   // 分区丢失检测间隔
    DataPartitionTimeOutSec      int64   // 分区超时时间
    MetaPartitionTimeOutSec      int64   // 元数据分区超时
    
    // MetaNode 配置
    metaNodeReservedMem  uint64  // 预留内存
    
    // 其他
    DisableAutoCreate    bool    // 禁用自动创建分区
    FaultDomain          bool    // 启用故障域
}
```

## 5. createRaftServer 详解

```go
// server.go:480
func (m *Server) createRaftServer(cfg *config.Config) (err error) {
    // 1. 配置 Raft
    raftCfg := &raftstore.Config{
        NodeID:            m.id,
        RaftPath:          m.walDir,
        IPAddr:            cfg.GetString(IP),
        NumOfLogsToRetain: m.retainLogs,
        HeartbeatPort:     int(m.config.heartbeatPort),
        ReplicaPort:       int(m.config.replicaPort),
        TickInterval:      m.tickInterval,
        ElectionTick:      m.electionTick,
    }
    
    // 2. 创建 RaftStore
    m.raftStore, err = raftstore.NewRaftStore(raftCfg, cfg)
    
    // 3. 初始化状态机
    m.initFsm()
    
    // 4. 创建 Raft Partition
    partitionCfg := &raftstore.PartitionConfig{
        ID:      GroupID,           // 固定为 1
        Peers:   m.config.peers,
        Applied: m.fsm.applied,
        SM:      m.fsm,             // 状态机
    }
    m.partition, err = m.raftStore.CreatePartition(partitionCfg)
    
    return
}
```

## 6. initFsm 初始化状态机

```go
// server.go:509
func (m *Server) initFsm() {
    m.fsm = newMetadataFsm(m.rocksDBStore, m.retainLogs, m.raftStore.RaftServer())
    
    // 注册各种处理器
    m.fsm.registerLeaderChangeHandler(m.handleLeaderChange)
    m.fsm.registerPeerChangeHandler(m.handlePeerChange)
    m.fsm.registerApplySnapshotHandler(m.handleApplySnapshot)
    m.fsm.registerRaftUserCmdApplyHandler(m.handleRaftUserCmd)
    
    // 恢复状态
    m.fsm.restore()
}
```

## 7. initCluster 初始化集群

```go
// server.go:520
func (m *Server) initCluster() {
    m.cluster = newCluster(
        m.clusterName,
        m.leaderInfo,
        m.fsm,
        m.partition,
        m.config,
        m,
    )
    m.cluster.retainLogs = m.retainLogs
    
    // 加载 API 限流配置
    m.cluster.loadApiLimiterInfo()
}
```

## 8. Master 组件关系图

```
┌─────────────────────────────────────────────────────────────┐
│                         Server                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────┐         ┌─────────────┐                  │
│   │   config    │         │  leaderInfo │                  │
│   │ (clusterCfg)│         │             │                  │
│   └─────────────┘         └─────────────┘                  │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                      Cluster                         │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐            │  │
│   │   │   Vol   │  │Topology │  │IDAlloc  │            │  │
│   │   │  管理    │  │ 管理    │  │  管理    │            │  │
│   │   └─────────┘  └─────────┘  └─────────┘            │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                    Raft Layer                        │  │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐         │  │
│   │   │raftStore │  │   fsm    │  │partition │         │  │
│   │   └──────────┘  └──────────┘  └──────────┘         │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                   RocksDB Store                      │  │
│   │                  (rocksDBStore)                      │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 9. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| Server 结构 | server.go:103 |
| Start | server.go:140 |
| checkConfig | server.go:234 |
| createRaftServer | server.go:480 |
| initFsm | server.go:509 |
| initCluster | server.go:520 |
| clusterConfig | config.go |

---

## 下一步
- Layer 2: 集群管理

*创建时间：2026-04-12*
