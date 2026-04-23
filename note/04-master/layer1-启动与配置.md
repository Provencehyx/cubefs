# Layer 1: Master 启动与设计思想

## 1. Master 的定位

### 1.1 为什么需要 Master？

分布式存储系统需要一个**中央协调者**来解决以下问题：

| 问题 | 没有 Master 会怎样 | Master 如何解决 |
|------|-------------------|----------------|
| 节点发现 | 节点不知道彼此存在，无法形成集群 | 节点向 Master 注册，获取集群视图 |
| 元数据一致性 | 谁持有哪些数据？多个节点可能给出不同答案 | Master 是元数据的 Single Source of Truth |
| 资源分配 | 新数据写到哪？可能集中写到少数节点 | Master 全局视角，均衡分配 |
| 故障检测 | 节点挂了谁知道？恢复由谁发起？ | Master 心跳检测，主动发起恢复 |

### 1.2 Master 的设计权衡

```
                    CAP 权衡
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
   强一致性 (CP)                高可用性 (AP)
   Master 选择这条路             部分系统选择这条路
         │
         ▼
   代价：Leader 单点处理
   收益：元数据绝对一致，简化系统设计
```

**CubeFS Master 的选择**：CP 路线，通过 Raft 保证强一致性。

为什么可以接受？
- 元数据操作（创建卷、分区）不是高频操作
- 读操作可以 Follower 分担（FollowerRead）
- 实际数据读写不经过 Master，只需元数据定位

---

## 2. 启动流程设计

### 2.1 启动顺序的考量

```
Start()
   │
   ├── 1. RocksDB 先行
   │       为什么：Raft 状态机需要存储，必须先就绪
   │
   ├── 2. Raft Server 启动
   │       为什么：集群选举必须先完成，才知道谁是 Leader
   │       └── 初始化 FSM（状态机）
   │       └── 恢复已持久化的元数据
   │
   ├── 3. Cluster 初始化
   │       为什么：依赖 Raft 的 partition 来提交变更
   │       └── 此时 metaReady = false，不对外服务
   │
   ├── 4. 后台任务启动
   │       为什么：需要 Cluster 就绪才能执行检查
   │       └── 只有 Leader 执行核心任务
   │
   └── 5. HTTP 服务最后启动
           为什么：所有依赖就绪后才能接受请求
           └── 此时 metaReady 可能仍为 false
```

**关键洞察**：`metaReady` 标志位

```go
// http_server.go:129
if m.partition.IsRaftLeader() || isFollowerRead {
    if m.metaReady || isFollowerRead {   // ← Leader 必须等 metaReady
        next.ServeHTTP(w, r)
        return
    }
    http.Error(w, m.leaderInfo.addr, http.StatusBadRequest)
    return
}
```

这是一个**启动保护**：即使选为 Leader，也要等 FSM 恢复完成才对外服务。

### 2.2 为什么 Raft Group ID 固定为 1？

```go
// server.go:56
const GroupID = 1  // 固定值
```

**原因**：Master 只需要一个 Raft Group。

对比 MetaNode：每个 MetaPartition 是一个独立的 Raft Group，需要动态分配 ID。

Master 管理的是**整个集群的元数据**，不需要分片，一个 Group 足够。

---

## 3. 关键设计决策

### 3.1 Follower 如何处理请求？

当请求到达 Follower 时：

```
请求到达 Follower
       │
       ├── 是 FollowerRead 类请求？
       │       ├── 是 → 本地处理（如 ClientDataPartitions）
       │       └── 否 → 转发给 Leader
       │
       └── 转发机制：ReverseProxy
               └── m.reverseProxy.ServeHTTP(w, r)
```

**为什么用 ReverseProxy 而不是返回 Leader 地址让客户端重试？**

1. 对客户端透明，减少客户端复杂度
2. 避免客户端缓存过期的 Leader 地址
3. Master 节点数量少（通常 3 个），内部转发开销可接受

### 3.2 FollowerRead 的设计

```go
// http_server.go:71
func (m *Server) isFollowerRead(r *http.Request) (followerRead bool) {
    // 某些路径允许 Follower 直接返回缓存的视图
    if r.URL.Path == proto.ClientDataPartitions && !m.partition.IsRaftLeader() {
        if volName, err := parseAndExtractName(r); err == nil {
            if followerRead = m.cluster.followerReadManager.IsVolViewReady(volName); followerRead {
                return  // Follower 可以处理
            }
        }
    }
}
```

**为什么需要 FollowerRead？**

`ClientDataPartitions` 是客户端最频繁的请求——获取数据分区列表来定位数据。如果全部压到 Leader：
- Leader 成为瓶颈
- 集群扩大后请求量线性增长

**如何保证 Follower 数据不过期？**

```go
// cluster.go:204
type followerReadManager struct {
    volDataPartitionsView     map[string][]byte  // 缓存的分区视图
    lastUpdateTick            map[string]time.Time  // 上次更新时间
    needCheck                 bool
}
```

Follower 定期从 Leader 同步视图，有 TTL 机制。客户端能接受短暂的不一致（秒级）。

### 3.3 后台任务的 Leader 检查

```go
// cluster.go:536
func (c *Cluster) scheduleToUpdateStatInfo() {
    c.runTask(&cTask{
        tickTime: 2 * time.Minute,
        function: func() (fin bool) {
            if c.partition != nil && c.partition.IsRaftLeader() {  // ← 关键检查
                c.updateStatInfo()
            }
            return
        },
    })
}
```

**为什么每个任务都要检查 IsRaftLeader？**

- 后台任务会修改状态或发送指令
- 只有 Leader 有权执行写操作
- Leader 切换后，旧 Leader 的任务自动失效

**潜在问题**：如果 Leader 频繁切换，任务可能被中断。解决方案是任务设计为**幂等**。

---

## 4. 值得注意的点

### 4.1 启动恢复的顺序依赖

```
RocksDB → Raft FSM → Cluster → HTTP

每一步都依赖前一步：
- FSM.restore() 需要 RocksDB 读取已有数据
- Cluster 需要 FSM 来提交新变更
- HTTP 需要 Cluster 处理业务逻辑
```

### 4.2 单点 Leader 的代价

所有写操作必须经过 Leader，这意味着：
- Leader 故障时写操作短暂不可用（选举期间，通常秒级）
- Leader 是性能瓶颈（但 Master 操作本就不是高频）

### 4.3 配置的关键参数

| 参数 | 默认值 | 影响 |
|------|--------|------|
| `tickInterval` | 500ms | Raft 心跳基础周期 |
| `electionTick` | 5 | 选举超时 = 5 × tickInterval = 2.5s |
| `retainLogs` | 20000 | 保留的 Raft 日志条数，影响 Follower 落后太多时的恢复方式 |
| `nodeSetCapacity` | 18 | 每个 NodeSet 最多容纳的节点数，影响故障域 |

### 4.4 metaReady 的状态变化

```
启动时：metaReady = false
  ↓
FSM 恢复完成后：metaReady = true  （在 handleLeaderChange 或恢复完成时设置）
  ↓
此时才真正对外服务
```

这个标志位防止了"我是 Leader 但数据还没加载完"的窗口期。

---

## 5. Server 核心结构

```go
// server.go:103
type Server struct {
    // 身份信息
    id              uint64
    clusterName     string
    ip, port        string
    
    // 存储层
    rocksDBStore    *raftstore_db.RocksDBStore  // 持久化
    
    // Raft 层
    raftStore       raftstore.RaftStore   // Raft 服务
    fsm             *MetadataFsm          // 状态机
    partition       raftstore.Partition   // Raft Group
    
    // 业务层
    cluster         *Cluster              // 集群管理核心
    user            *User                 // 用户管理
    
    // HTTP 层
    reverseProxy    *httputil.ReverseProxy  // Follower 转发用
    apiServer       *http.Server
    metaReady       bool                    // 是否可以对外服务
}
```

**层次关系**：

```
HTTP 请求
    ↓
Server (路由 + Leader 判断)
    ↓
Cluster (业务逻辑)
    ↓
MetadataFsm (Raft 提交)
    ↓
RocksDB (持久化)
```

---

## 下一步

→ [Layer 2: 集群管理](layer2-集群管理.md) - Cluster 的核心职责和后台任务

---
*更新时间: 2026-04-23*
