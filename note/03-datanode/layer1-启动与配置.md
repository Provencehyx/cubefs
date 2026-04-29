# Layer 1: DataNode 启动与配置

## 核心数据结构

```go
// datanode/server.go:165
type DataNode struct {
    space           *SpaceManager    // 空间管理器（管理所有磁盘和分区）
    port            string           // 服务端口
    zoneName        string           // 可用区名称
    clusterID       string           // 集群 ID
    localServerAddr string           // 本机地址
    nodeID          uint64           // 节点 ID（Master 分配）
    
    // Raft 相关
    raftDir         string           // Raft 日志目录
    raftHeartbeat   string           // Raft 心跳地址
    raftReplica     string           // Raft 复制地址
    raftStore       raftstore.RaftStore  // Raft 存储
    
    // 网络相关
    tcpListener     net.Listener     // TCP 监听器
    smuxListener    net.Listener     // SMUX 多路复用监听器
    smuxConnPool    *util.SmuxConnectPool  // SMUX 连接池
    
    stopC           chan bool        // 停止信号
}
```

**字段说明**：

| 字段 | 作用 | 初始化时机 |
|------|------|-----------|
| `space` | 管理所有磁盘和分区 | `newSpaceManager()` |
| `nodeID` | 唯一标识，Raft 需要 | `register()` 向 Master 注册后获得 |
| `raftStore` | 管理多个 Raft 分组 | `startRaftServer()` |
| `smuxConnPool` | SMUX 连接复用 | `startSmuxService()` |

---

## 核心问题

1. **启动顺序为什么是这样的？**
2. **为什么要先注册再加载分区？**
3. **DataNode 和 MetaNode 启动有什么区别？**

---

## 1. 启动顺序设计

```
doStart() 启动流程                         设计意图
─────────────────────────────────────────────────────────────────
parseConfig()                             解析配置
       │
       ▼
register() → Master                       【关键】先注册获取 nodeID
       │                                  ↳ 后续 Raft 需要 nodeID
       ▼
startRaftServer()                         启动 Raft（但还没有分区）
       │
       ▼
newSpaceManager()                         创建空间管理器（还未加载）
       │
       ▼
startTCPService()                         开始监听（准备接收请求）
       │
       ▼
startSpaceManager()                       【关键】加载磁盘和分区
       │                                  ↳ 每个分区启动 Raft
       ▼
checkLocalPartitionMatchWithMaster()      【关键】校验分区一致性
       │                                  ↳ 防止孤儿分区
       ▼
registerHandler() + scheduleTask()        注册 HTTP + 启动后台任务
```

### 为什么 register() 要在 startSpaceManager() 之前？

```go
// server.go:297
if err = s.register(cfg); err != nil {
    return  // 注册失败，不继续
}
// ...
if err = s.startSpaceManager(cfg); err != nil {
    return
}
```

**原因**：
1. `register()` 向 Master 注册后获得 `nodeID`
2. 每个 DataPartition 的 Raft 需要 `nodeID` 来标识自己
3. 如果先加载分区，Raft 无法正确初始化

### 为什么要 checkLocalPartitionMatchWithMaster()？

这一步防止**孤儿分区**问题：

```
场景：DataNode 重启后发现本地有分区 1001
     但 Master 记录中该分区已迁移到其他节点

如果不校验：
- 本地 1001 启动 Raft
- 和新的副本产生脑裂
- 数据不一致

有校验：
- 发现 Master 中 1001 不属于本节点
- 启动失败，管理员介入处理
```

---

## 2. DataNode vs MetaNode 启动差异

| 阶段 | DataNode | MetaNode | 差异原因 |
|------|----------|----------|----------|
| 存储初始化 | `newSpaceManager()` 管理多磁盘 | `startMetaManager()` 无磁盘概念 | DataNode 要管理物理磁盘 |
| 分区加载 | 从磁盘目录扫描 | 从 RocksDB 加载 | 存储引擎不同 |
| 网络协议 | TCP + SMUX（多路复用） | TCP 为主 | 大数据量需要多路复用 |
| QoS | 支持 IOPS/流量限制 | 无 | 数据流量大需要限流 |

### 为什么 DataNode 需要 SMUX？

**SMUX (Stream Multiplexing)**：在一个 TCP 连接上多路复用多个流

```
没有 SMUX：
Client ──conn1──> DataNode  (写文件1)
Client ──conn2──> DataNode  (写文件2)
Client ──conn3──> DataNode  (写文件3)
每个文件一个连接 → 连接数爆炸

有 SMUX：
Client ═══一个TCP连接═══> DataNode
         │ stream1 │ (写文件1)
         │ stream2 │ (写文件2)
         │ stream3 │ (写文件3)
多个流复用一个连接 → 连接数可控
```

**为什么 MetaNode 不需要？**
- 元数据请求小（几 KB）
- 请求频繁但短命
- 没有大数据流

---

## 3. 配置设计

### 关键配置项

| 配置 | 默认值 | 为什么这样设计 |
|------|--------|----------------|
| `disks` | 无 | 必填，告诉 DataNode 用哪些磁盘存储数据 |
| `raftDir` | 独立目录 | Raft 日志放单独目录，不和数据混在一起，防止磁盘满影响 Raft |
| `diskRdonlySpace` | 5GB | 磁盘剩余空间低于此值变为只读，防止写满 |

### QoS 配置的意义

```yaml
diskQosEnable: true
diskReadIops: 10000
diskWriteIops: 5000
diskReadFlow: 1073741824   # 1GB/s
diskWriteFlow: 536870912   # 512MB/s
```

**为什么需要 QoS？**
- 单个热点文件可能打满磁盘带宽
- 影响同磁盘其他分区
- QoS 确保公平性

---

## 4. 核心组件关系

```
┌──────────────────────────────────────────────────────────────────┐
│                           DataNode                                │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │                      SpaceManager                             ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                       ││
│  │  │  Disk1  │  │  Disk2  │  │  Disk3  │  ← 物理磁盘           ││
│  │  │ /data1  │  │ /data2  │  │ /data3  │                       ││
│  │  └────┬────┘  └────┬────┘  └────┬────┘                       ││
│  │       │            │            │                             ││
│  │       ▼            ▼            ▼                             ││
│  │   DP 1001       DP 1002      DP 1003   ← DataPartition       ││
│  │   DP 1004       DP 1005      DP 1006                         ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                   │
│  ┌────────────┐         ┌────────────┐         ┌────────────┐   │
│  │ RaftStore  │         │ TCPServer  │         │ SmuxServer │   │
│  │ (成员变更)  │         │ (数据请求)  │         │ (多路复用)  │   │
│  └────────────┘         └────────────┘         └────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

**层次关系**：
- `DataNode` 是入口
- `SpaceManager` 管理所有磁盘
- 每个 `Disk` 包含多个 `DataPartition`
- 每个 `DataPartition` 有自己的 `ExtentStore` 和 `RaftPartition`

---

## 5. 关键代码

| 功能 | 位置 | 要点 |
|------|------|------|
| 启动入口 | server.go:264 | `Start()` → `doStart()` |
| 向 Master 注册 | server.go:297 | 获取 nodeID，后续 Raft 需要 |
| 分区校验 | server.go:339 | `checkLocalPartitionMatchWithMaster()` 防止孤儿分区 |
| 关闭顺序 | server.go:376 | 先停 Raft，再停磁盘，最后关连接 |

---

*更新时间：2026-04-27*
