# Layer 2: Cluster 设计与职责

## 1. Cluster 的核心职责

Cluster 是 Master 的业务核心，但它的设计体现了**关注点分离**思想：

```
┌─────────────────────────────────────────────────────────────┐
│                        Cluster                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Cluster 本身不直接实现复杂逻辑，而是：                      │
│                                                             │
│   1. 组合各个子模块（卷、拓扑、下线）                         │
│   2. 协调模块间交互                                          │
│   3. 调度后台任务                                            │
│   4. 作为对外 API 的入口                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 为什么用组合而不是继承？

```go
// cluster.go:141
type Cluster struct {
    ClusterVolSubItem       // 卷管理 - 嵌入
    ClusterTopoSubItem      // 拓扑管理 - 嵌入
    ClusterDecommission     // 下线管理 - 嵌入
    // ...
}
```

**Go 的组合设计**：
- 每个 SubItem 关注自己的领域
- Cluster 可以直接调用子模块方法（`c.vols`、`c.dataNodes`）
- 避免单个文件过于庞大（cluster.go 已经 5000+ 行）

---

## 2. 后台任务调度系统

### 2.1 为什么需要这么多后台任务？

```go
// cluster.go:492
func (c *Cluster) scheduleTask() {
    c.scheduleToCheckHeartbeat()            // 心跳检测
    c.scheduleToCheckDataPartitions()       // DP 健康检查
    c.scheduleToCheckMetaPartitions()       // MP 健康检查
    c.scheduleToUpdateStatInfo()            // 统计更新
    c.scheduleToCheckDiskRecoveryProgress() // 恢复进度
    // ... 20+ 个任务
}
```

**分布式系统的现实**：节点会挂、磁盘会坏、网络会抖动。Master 必须**持续监控**并**主动修复**。

### 2.2 任务调度模式

```go
// cluster.go 中的通用模式
func (c *Cluster) scheduleToXxx() {
    c.runTask(&cTask{
        tickTime: 2 * time.Minute,      // 执行间隔
        name:     "scheduleToXxx",
        function: func() (fin bool) {
            if c.partition != nil && c.partition.IsRaftLeader() {  // ★ Leader 检查
                c.doXxx()
            }
            return  // fin=false 表示继续循环
        },
    })
}
```

**关键设计点**：

| 设计 | 原因 |
|------|------|
| 每个任务独立 goroutine | 任务间隔离，一个卡住不影响其他 |
| IsRaftLeader 检查 | 只有 Leader 执行写操作，避免冲突 |
| fin 返回值 | true 停止任务，false 继续循环 |
| tickTime 不同 | 根据任务紧急程度调整（心跳 60s，统计 2min） |

### 2.3 任务分类

```
┌─────────────────────────────────────────────────────────────┐
│                      后台任务分类                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  【健康检查类】                                              │
│   ├── checkHeartbeat      - 节点存活检测                    │
│   ├── checkDataPartitions - DP 副本完整性                   │
│   └── checkMetaPartitions - MP 副本完整性                   │
│                                                             │
│  【恢复修复类】                                              │
│   ├── checkDiskRecoveryProgress - 磁盘恢复进度              │
│   └── checkMetaPartitionRecoveryProgress - MP 恢复          │
│                                                             │
│  【状态维护类】                                              │
│   ├── updateStatInfo      - 更新集群统计                    │
│   ├── checkVolStatus      - 卷状态检查                      │
│   └── checkFollowerReadCache - Follower 缓存更新            │
│                                                             │
│  【自动化运维类】                                            │
│   ├── checkDecommissionDataNode - 自动下线                  │
│   ├── checkDecommissionDisk     - 坏盘下线                  │
│   └── scheduleToBadDisk         - 坏盘检测                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 心跳与故障检测

### 3.1 心跳设计思想

```
节点 → Master（定期上报心跳）
  │
  │  心跳内容：
  │  - 我还活着
  │  - 我的资源使用情况（CPU、内存、磁盘）
  │  - 我管理的分区状态
  │
Master（被动接收 + 主动检查）
  │
  └─ checkHeartbeat 定时任务
       └─ 对比 lastReportTime，超时则标记节点下线
```

**为什么是节点主动上报而不是 Master 主动探测？**

| 方案 | 优点 | 缺点 |
|------|------|------|
| 节点主动上报 | Master 无需维护探测连接，节点数增加不增加 Master 负担 | 节点挂了要等超时才发现 |
| Master 主动探测 | 发现故障更快 | O(N) 探测请求，Master 压力大 |

CubeFS 选择**节点主动上报**，超时阈值通常设为 2-3 个心跳周期。

### 3.2 故障检测的保守策略

```go
// 伪代码逻辑
func checkDataNodeHeartbeat() {
    for _, node := range allDataNodes {
        if time.Since(node.ReportTime) > DataNodeTimeOut {
            node.isActive = false
            // 但不会立即删除节点或迁移数据！
        }
    }
}
```

**为什么不立即迁移？**

- 可能是网络抖动，节点马上恢复
- 立即迁移会产生大量数据复制，加重集群负担
- 有独立的恢复任务来判断是否需要迁移

---

## 4. ID 分配器设计

### 4.1 全局唯一 ID 的挑战

分布式系统中生成唯一 ID 的常见方案：

| 方案 | 优点 | 缺点 |
|------|------|------|
| UUID | 无需协调 | 128 位太长，无序 |
| Snowflake | 有序，64 位 | 需要机器 ID 分配 |
| **中心分配** | 简单，绝对唯一 | 依赖中心节点 |

CubeFS 选择**中心分配**，因为：
- Master 本身就是中心化的
- 分区/节点创建不是高频操作
- ID 需要持久化到 Raft，天然适合 Master 管理

### 4.2 ID 分配的持久化

```go
// id_allocator.go
func (alloc *IDAllocator) allocateDataPartitionID() (uint64, error) {
    alloc.dpIDLock.Lock()
    defer alloc.dpIDLock.Unlock()
    
    alloc.dataPartitionID++           // 递增
    
    // ★ 关键：必须 Raft 持久化后才返回
    if err := alloc.syncToRaft(); err != nil {
        alloc.dataPartitionID--        // 回滚
        return 0, err
    }
    
    return alloc.dataPartitionID, nil
}
```

**为什么必须先持久化？**

如果先返回 ID，再异步持久化：
1. 返回 ID=100 给调用方
2. 持久化前 Master 挂了
3. 重启后 ID 从 99 开始
4. 下次分配又是 100，ID 冲突！

---

## 5. Follower 读优化

### 5.1 问题背景

Client 最频繁的请求：`GetDataPartitions`（获取卷的分区列表来定位数据）。

如果全部走 Leader：
- Leader 成为瓶颈
- 3 节点 Master 集群，2/3 的能力被浪费

### 5.2 解决方案

```go
// cluster.go:204
type followerReadManager struct {
    volDataPartitionsView     map[string][]byte    // 缓存的分区视图
    volDataPartitionsCompress map[string][]byte    // 压缩版本
    lastUpdateTick            map[string]time.Time // 更新时间
    status                    map[string]bool      // 是否可用
}
```

**工作流程**：

```
Leader 定期将分区视图推送给 Follower
    │
    ▼
Follower 缓存视图
    │
    ▼
Client 请求到 Follower
    │
    ├── 缓存有效？
    │       ├── 是 → 直接返回缓存
    │       └── 否 → 返回 Leader 地址，让 Client 重试
```

### 5.3 一致性权衡

Follower 返回的数据可能**轻微过期**（秒级）。为什么可以接受？

- 分区列表变化不频繁（创建/扩容/迁移才变）
- Client 有本地缓存，不会每次都请求
- 即使拿到过期列表，最多导致一次重试（分区不存在时会刷新）

---

## 6. 子结构设计分析

### 6.1 ClusterVolSubItem（卷管理）

```go
type ClusterVolSubItem struct {
    vols                map[string]*Vol
    delayDeleteVolsInfo []*delayDeleteVolInfo  // ★ 延迟删除
    volMutex            sync.RWMutex
    createVolMutex      sync.RWMutex           // ★ 创建专用锁
    deleteVolMutex      sync.RWMutex           // ★ 删除专用锁
}
```

**为什么有三把锁？**

- `volMutex`：读写 vols map
- `createVolMutex`：防止并发创建同名卷
- `deleteVolMutex`：防止删除过程中的并发问题

**为什么要延迟删除？**

删除卷是危险操作，延迟删除提供**后悔窗口**：
1. 用户删除卷 → 加入 delayDeleteVolsInfo
2. 等待一段时间（可配置）
3. 后台任务真正执行删除

### 6.2 ClusterDecommission（下线管理）

```go
type ClusterDecommission struct {
    DecommissionLimit     uint64    // 并发限制
    DecommissionDiskLimit uint32    // 磁盘下线并发
    
    EnableAutoDecommissionDisk atomicutil.Bool  // 自动下线开关
}
```

**为什么需要并发限制？**

下线一个节点意味着迁移其上所有数据。如果同时下线多个：
- 网络带宽被打满
- 其他节点磁盘 IO 被打满
- 正常业务受影响

限制并发数是**保护集群稳定性**的关键。

---

## 7. 值得注意的设计

### 7.1 stopc 的优雅停机

```go
type Cluster struct {
    stopc    chan bool
    stopFlag int32
    wg       sync.WaitGroup
}

// 启动任务时
c.wg.Add(1)
go func() {
    defer c.wg.Done()
    for {
        select {
        case <-c.stopc:
            return  // 收到停止信号，退出
        case <-time.After(tickTime):
            // 执行任务
        }
    }
}()

// 停机时
close(c.stopc)  // 通知所有任务
c.wg.Wait()     // 等待全部退出
```

### 7.2 metaReady 的保护

```go
// 只有 Leader 且 metaReady 才真正处理业务
if c.partition.IsRaftLeader() && c.metaReady {
    // 处理请求
}
```

这防止了**启动瞬间**的不一致状态：
- 刚选为 Leader
- 但 FSM 还在恢复数据
- 此时处理请求会基于不完整的数据

### 7.3 任务的幂等性

后台任务可能因为 Leader 切换而中断。设计时必须考虑：
- 任务执行到一半，另一个 Master 接手
- 新 Leader 的任务可能重复执行部分逻辑

**解决方案**：任务设计为幂等——执行多次效果等同于执行一次。

---

## 8. Cluster 核心方法索引

| 类别 | 方法 | 位置 | 说明 |
|------|------|------|------|
| 初始化 | `newCluster` | cluster.go:433 | 创建 Cluster |
| 任务调度 | `scheduleTask` | cluster.go:492 | 启动所有后台任务 |
| 心跳检查 | `checkDataNodeHeartbeat` | cluster.go | 检查 DN 心跳 |
| 心跳检查 | `checkMetaNodeHeartbeat` | cluster.go | 检查 MN 心跳 |
| 统计更新 | `updateStatInfo` | cluster_stat.go | 更新集群统计 |
| ID 分配 | `allocateDataPartitionID` | id_allocator.go | 分配 DP ID |

---

## 下一步

→ [Layer 3: 拓扑管理](layer3-拓扑管理.md) - Zone/NodeSet 的故障域设计

---
*更新时间: 2026-04-23*
