# Layer 2: RaftServer - Multi-Raft 管理器

## 1. 概述

### 1.1 定位

RaftServer 是 tiglabs/raft 的**中央调度器**，负责管理同一进程内的多个 Raft Group（Multi-Raft 架构）。

```
┌─────────────────────────────────────────────────────────────────┐
│                         RaftServer                              │
│                      (1个进程内唯一)                             │
│                                                                 │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐       ┌─────────┐      │
│   │ Raft 1  │  │ Raft 2  │  │ Raft 3  │  ...  │ Raft N  │      │
│   │ (分区1) │  │ (分区2) │  │ (分区3) │       │ (分区N) │      │
│   └─────────┘  └─────────┘  └─────────┘       └─────────┘      │
│                                                                 │
│   共享: Ticker | Transport | HeartbeatChannel                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **生命周期管理** | 创建/删除 Raft Group |
| **统一 Tick 驱动** | 单定时器驱动所有 Raft 状态机 |
| **心跳调度** | 统一发送和处理心跳消息 |
| **资源共享** | 多 Raft Group 共享网络传输层 |
| **故障隔离** | 单个 Group 故障不影响其他 Group |

---

## 2. 核心设计思想

### 2.1 资源共享 vs 状态隔离

```
┌────────────────────────────────────────────────────────────────┐
│                      资源模型                                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  【共享资源】节省开销                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • Ticker (1个)     → 所有 Raft 同步 tick，避免选举风暴   │  │
│  │  • Transport (1个)  → 复用 TCP 连接，减少连接数           │  │
│  │  • HeartbeatChan    → 统一心跳调度，优先处理              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  【独立资源】状态隔离                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  每个 Raft Group 独立拥有:                                │  │
│  │  • raftFsm     → 独立状态机 (term/vote/role)             │  │
│  │  • raftLog     → 独立日志序列                             │  │
│  │  • Storage     → 独立持久化                               │  │
│  │  • Channels    → 独立消息通道 (propc/recvc/applyc)        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 单 Ticker 设计

**问题**: 1000 个 Raft Group = 1000 个独立定时器？

**方案对比**:

| 方案 | 资源开销 | 同步性 | 复杂度 |
|------|----------|--------|--------|
| 每 Raft 独立 Ticker | O(N) goroutine + timer | 不同步，易选举风暴 | 低 |
| **共享单 Ticker** | O(1) | 同步 tick，协调选举 | 略高 |

**采用方案 B**：单 Ticker 遍历调用所有 raft.tick()

### 2.3 心跳独立通道

**问题**: 心跳是 Raft 生命线，混在大消息队列中会被阻塞

```
普通队列: [大日志][大日志][心跳][大日志]...  ← 心跳延迟，触发误选举
              ↓
分离设计: heartc: [心跳][心跳]...  ← 小、快、优先
          recvc:  [大日志]...      ← 排队处理
```

---

## 3. 核心结构

```go
// server.go:31
type RaftServer struct {
    config *Config                    // 全局配置
    ticker *time.Ticker               // ★ 共享定时器 (单一 tick 源)
    heartc chan *proto.Message        // ★ 心跳专用通道 (优先级高)
    stopc  chan struct{}              // 停止信号
    mu     sync.RWMutex               // 保护 rafts map
    rafts  map[uint64]*raft           // ★ Raft Group 集合 (key=GroupID)
}
```

**关键字段说明**:

| 字段 | 作用 |
|------|------|
| `ticker` | 统一时钟源，驱动所有 Raft 状态转换 |
| `heartc` | 心跳消息独立通道，避免被大消息阻塞 |
| `rafts` | GroupID → raft 实例的映射表 |

---

## 4. 生命周期

### 4.1 启动流程

```
NewRaftServer()
    │
    ├─→ 1. 校验配置 config.validate()
    │
    ├─→ 2. 初始化结构
    │       • ticker = NewTicker(TickInterval)
    │       • rafts = make(map[uint64]*raft)
    │       • heartc = make(chan, 512)
    │
    ├─→ 3. 创建网络传输层 (共享)
    │       • transport = NewMultiTransport()
    │
    └─→ 4. 启动主循环 goroutine
            • go rs.run()
```

### 4.2 主循环 (核心)

```go
// server.go:90
func (rs *RaftServer) run() {
    ticks := 0
    for {
        select {
        case <-rs.stopc:
            return  // 停止

        case id := <-fatalStopc:
            delete(rs.rafts, id)  // 移除故障 Raft

        case m := <-rs.heartc:
            // 优先处理心跳
            rs.handleHeartbeat(m) 或 rs.handleHeartbeatResp(m)

        case <-rs.ticker.C:
            // 定时 tick
            ticks++
            if ticks >= HeartbeatTick {
                ticks = 0
                rs.sendHeartbeat()  // Leader 发心跳
            }
            // 驱动所有 Raft 状态机
            for _, raft := range rs.rafts {
                raft.tick()
            }
        }
    }
}
```

**主循环职责图**:

```
              ┌─────────────────────────────────────────┐
              │           RaftServer.run()              │
              │         (单 goroutine 事件循环)          │
              └───────────────────┬─────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
   ┌─────────┐            ┌──────────────┐          ┌─────────────┐
   │ stopc   │            │   heartc     │          │  ticker.C   │
   │ 优雅停止 │            │ 心跳收发     │          │ tick 驱动   │
   └─────────┘            └──────────────┘          └─────────────┘
                                                           │
                                                    ┌──────┴──────┐
                                                    ▼             ▼
                                              sendHeartbeat   raft.tick()
                                              (Leader发心跳)  (状态机推进)
```

### 4.3 创建 Raft Group

```go
// server.go:151
func (rs *RaftServer) CreateRaft(config *RaftConfig) error {
    raft, err := newRaft(rs.config, config)  // 创建实例
    
    rs.mu.Lock()
    if _, exists := rs.rafts[config.ID]; exists {
        return ErrRaftExists  // 已存在
    }
    rs.rafts[config.ID] = raft  // 注册到 map
    rs.mu.Unlock()
}
```

### 4.4 删除 Raft Group

```go
// server.go:184
func (rs *RaftServer) RemoveRaft(id uint64) error {
    rs.mu.Lock()
    raft := rs.rafts[id]
    delete(rs.rafts, id)  // 从 map 移除
    rs.mu.Unlock()
    
    raft.stop()  // 停止实例
}
```

---

## 5. 对外接口

RaftServer 对外暴露的公共方法 (server.go):

### 5.1 生命周期管理

| 方法 | 位置 | 说明 |
|------|------|------|
| `CreateRaft(raftConfig *RaftConfig) error` | :151 | 创建 Raft Group |
| `RemoveRaft(id uint64) error` | :184 | 删除 Raft Group |
| `RemoveRaftForce(raftId uint64, cc *proto.ConfChange)` | :40 | 强制删除 |
| `Stop()` | :126 | 停止 RaftServer |

### 5.2 数据操作

| 方法 | 位置 | 说明 |
|------|------|------|
| `Submit(id uint64, cmd []byte) *Future` | :196 | 提交命令 |
| `ChangeMember(id uint64, changeType proto.ConfChangeType, peer proto.Peer, context []byte) *Future` | :210 | 成员变更 |
| `ReadIndex(id uint64) *Future` | :395 | 线性一致性读 |
| `GetEntries(id uint64, startIndex uint64, maxSize uint64) *Future` | :410 | 获取日志条目 |
| `Truncate(id uint64, index uint64)` | :321 | 截断日志 |
| `TryToLeader(id uint64) *Future` | :307 | 尝试成为 Leader |

### 5.3 状态查询

| 方法 | 位置 | 说明 |
|------|------|------|
| `Status(id uint64) *Status` | :233 | 获取状态 |
| `IsLeader(id uint64) bool` | :264 | 是否 Leader |
| `IsRestoring(id uint64) bool` | :224 | 是否恢复中 |
| `LeaderTerm(id uint64) (leader, term uint64)` | :253 | Leader 和 Term |
| `AppliedIndex(id uint64) uint64` | :274 | 已应用索引 |
| `CommittedIndex(id uint64) uint64` | :285 | 已提交索引 |
| `FirstCommittedIndex(id uint64) uint64` | :296 | 首个已提交索引 |

### 5.4 集群信息

| 方法 | 位置 | 说明 |
|------|------|------|
| `GetPeers(id uint64) []uint64` | :512 | 获取成员列表 |
| `GetUnreachable(id uint64) []uint64` | :332 | 不可达节点 |
| `GetDownReplicas(id uint64) []DownReplica` | :341 | 宕机副本 |
| `GetPendingReplica(id uint64) []uint64` | :371 | 待同步副本 |

### 5.5 Submit 流程示例

```go
// server.go:196
func (rs *RaftServer) Submit(id uint64, cmd []byte) (future *Future) {
    rs.mu.RLock()
    raft, ok := rs.rafts[id]   // 1. 查找 Raft Group
    rs.mu.RUnlock()

    future = newFuture()        // 2. 创建异步结果
    if !ok {
        future.respond(nil, ErrRaftNotExists)
        return
    }
    raft.propose(cmd, future)   // 3. 提交到 raft
    return
}
```

---

## 6. 配置参数

```go
// config.go
type Config struct {
    // 节点标识
    NodeID         uint64         // 本节点 ID
    
    // 时间参数
    TickInterval   time.Duration  // tick 间隔 (如 500ms)
    HeartbeatTick  int            // 心跳 = N 个 tick (如 2)
    ElectionTick   int            // 选举超时 = N 个 tick (如 10)
    
    // 缓冲区
    ReqBufferSize  int            // 请求缓冲区
    AppBufferSize  int            // 应用缓冲区
    
    // 网络
    TransportConfig TransportConfig
}
```

**时间关系**:
```
心跳间隔 = TickInterval × HeartbeatTick
选举超时 = TickInterval × ElectionTick

例: 500ms × 2 = 1s 心跳
    500ms × 10 = 5s 选举超时
```

---

## 7. 代码索引

| 功能 | 位置 |
|------|------|
| RaftServer 结构定义 | server.go:31 |
| NewRaftServer 构造 | server.go:68 |
| run 主循环 | server.go:90 |
| CreateRaft | server.go:151 |
| RemoveRaft | server.go:184 |
| Submit | server.go:196 |
| ChangeMember | server.go:210 |
| sendHeartbeat | server.go:~250 |
| handleHeartbeat | server.go:~270 |

---

## 8. 总结

```
RaftServer 核心要点:

1. 定位: Multi-Raft 中央调度器，管理多个 Raft Group

2. 资源模型:
   • 共享: Ticker + Transport + HeartbeatChannel
   • 隔离: 每个 Group 独立状态/日志/存储

3. 主循环: 单 goroutine + select
   • ticker.C → 驱动所有 raft.tick()
   • heartc   → 优先处理心跳
   • stopc    → 优雅停止

4. 关键设计:
   • 单 Ticker 避免选举风暴
   • 心跳独立通道保证优先级
   • fatalStopc 实现故障隔离
```

---
*创建时间: 2026-04-22*
