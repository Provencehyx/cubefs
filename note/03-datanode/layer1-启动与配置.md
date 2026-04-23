# Layer 1: DataNode 启动与配置

## 1. DataNode 核心结构

```go
// datanode/server.go:164
type DataNode struct {
    space           *SpaceManager       // 空间管理器（管理磁盘和分区）
    port            string              // 服务端口
    zoneName        string              // 可用区
    clusterID       string              // 集群 ID
    nodeID          uint64              // 节点 ID
    
    // Raft 相关
    raftDir         string              // Raft 日志目录
    raftHeartbeat   string              // Raft 心跳端口
    raftReplica     string              // Raft 复制端口
    raftStore       raftstore.RaftStore // Raft 存储（同 MetaNode）
    
    // 网络层
    tcpListener     net.Listener        // TCP 监听
    smuxListener    net.Listener        // SMUX 多路复用监听
    smuxConnPool    *util.SmuxConnectPool
    
    // 监控与指标
    metrics         *DataNodeMetrics
    cpuUtil         atomicutil.Float64
    
    // QoS 限流
    diskQosEnable   bool
    diskReadIops    int
    diskWriteIops   int
    // ...
}
```

## 2. 启动流程

```
┌──────────────────────────────────────────────────────────────┐
│                    doStart() 启动流程                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. parseConfig()          解析基础配置                       │
│         │                                                    │
│         ▼                                                    │
│  2. parseRaftConfig()      解析 Raft 配置                     │
│         │                                                    │
│         ▼                                                    │
│  3. register()             向 Master 注册                     │
│         │                                                    │
│         ▼                                                    │
│  4. parseSmuxConfig()      解析 SMUX 多路复用配置              │
│         │                                                    │
│         ▼                                                    │
│  5. startRaftServer()      启动 Raft 服务                     │
│         │                                                    │
│         ▼                                                    │
│  6. newSpaceManager()      创建空间管理器                      │
│         │                                                    │
│         ▼                                                    │
│  7. startTCPService()      启动 TCP 服务                      │
│         │                                                    │
│         ▼                                                    │
│  8. startSmuxService()     启动 SMUX 服务                     │
│         │                                                    │
│         ▼                                                    │
│  9. startSpaceManager()    启动空间管理（加载磁盘/分区）        │
│         │                                                    │
│         ▼                                                    │
│  10. checkLocalPartitionMatchWithMaster()  校验分区一致性      │
│         │                                                    │
│         ▼                                                    │
│  11. registerHandler()     注册 HTTP 处理器                   │
│         │                                                    │
│         ▼                                                    │
│  12. scheduleTask()        启动后台定时任务                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 3. 配置项说明

### 3.1 基础配置

| 配置键 | 类型 | 说明 |
|--------|------|------|
| `localIP` | string | 本机 IP |
| `port` | int | TCP 服务端口 |
| `masterAddr` | array | Master 地址列表 |
| `zoneName` | string | 可用区名称 |
| `disks` | array | 磁盘列表 |
| `logDir` | string | 日志目录 |

### 3.2 Raft 配置

| 配置键 | 类型 | 说明 |
|--------|------|------|
| `raftDir` | string | Raft 日志目录 |
| `raftHeartbeat` | string | 心跳端口 |
| `raftReplica` | string | 复制端口 |
| `tickInterval` | int | Tick 间隔 (ms) |
| `raftRecvBufSize` | int | 接收缓冲区大小 |

### 3.3 QoS 配置

| 配置键 | 类型 | 说明 |
|--------|------|------|
| `diskQosEnable` | bool | 启用磁盘 QoS |
| `diskReadIops` | int | 读 IOPS 限制 |
| `diskWriteIops` | int | 写 IOPS 限制 |
| `diskReadFlow` | int | 读流量限制 |
| `diskWriteFlow` | int | 写流量限制 |

## 4. DataNode vs MetaNode 启动对比

```
DataNode                              MetaNode
─────────────────────────────────────────────────────────
parseConfig()                         parseConfig()
         │                                   │
register() → Master                   register() → Master
         │                                   │
startRaftServer()                     startRaftServer()
         │                                   │
newSpaceManager()  ←── 磁盘管理        startMetaManager() ←── 无磁盘
         │                                   │
startTCPService()                     startServer()
         │                                   │
loadDataPartitions() ←── 加载分区      loadMetaPartitions() ←── 加载分区
```

## 5. 核心组件关系

```
┌───────────────────────────────────────────────────────────┐
│                        DataNode                           │
├───────────────────────────────────────────────────────────┤
│                                                           │
│   ┌─────────────┐    ┌──────────────┐    ┌────────────┐  │
│   │ RaftStore   │    │ SpaceManager │    │ TCPServer  │  │
│   │             │    │              │    │            │  │
│   │ (Raft协议)  │    │  ┌────────┐  │    │ (处理请求) │  │
│   │             │    │  │ Disk 1 │  │    │            │  │
│   └─────────────┘    │  │ Disk 2 │  │    └────────────┘  │
│          │           │  │ Disk 3 │  │          │         │
│          │           │  └────────┘  │          │         │
│          │           │      │       │          │         │
│          │           │      ▼       │          │         │
│          │           │ DataPartition│          │         │
│          └──────────►│   1, 2, 3... │◄─────────┘         │
│                      └──────────────┘                    │
│                             │                            │
│                             ▼                            │
│                     ┌──────────────┐                     │
│                     │ ExtentStore  │                     │
│                     │ (底层存储)    │                     │
│                     └──────────────┘                     │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

## 6. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| DataNode 结构体 | server.go:164 |
| 启动入口 | server.go:264 |
| 启动流程 | server.go:280 (doStart) |
| 配置解析 | server.go:405 (parseConfig) |
| 关闭流程 | server.go:376 (doShutdown) |

---

## 下一步
- Layer 2: 深入 SpaceManager 和 Disk 管理

*创建时间：2026-04-12*
