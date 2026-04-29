# Layer 2: 磁盘与空间管理

## 核心数据结构

### SpaceManager

```go
// datanode/space_manager.go:48
type SpaceManager struct {
    clusterID      string                    // 集群 ID
    disks          map[string]*Disk          // 路径 → 磁盘对象
    partitions     map[uint64]*DataPartition // 分区ID → 分区对象
    raftStore      raftstore.RaftStore       // Raft 存储
    nodeID         uint64                    // 节点 ID
    
    diskMutex      sync.RWMutex              // 磁盘 map 锁
    partitionMutex sync.RWMutex              // 分区 map 锁
    
    stats          *Stats                    // 统计信息
    diskList       []string                  // 磁盘路径列表
    diskUtils      map[string]*atomicutil.Float64  // 磁盘使用率
    rand           *rand.Rand                // 随机数生成器 (Straw 算法)
}
```

### Disk

```go
// datanode/disk.go:73
type Disk struct {
    sync.RWMutex
    Path            string           // 挂载路径，如 /data1
    
    // 错误统计
    ReadErrCnt      uint64           // 读错误计数
    WriteErrCnt     uint64           // 写错误计数
    MaxErrCnt       int              // 最大允许错误数
    
    // 空间管理
    Total           uint64           // 磁盘总容量
    Used            uint64           // 已使用空间
    Available       uint64           // 可用空间
    Unallocated     uint64           // 未分配空间
    Allocated       uint64           // 已分配给分区的空间
    ReservedSpace   uint64           // 预留空间 (防止写满)
    DiskRdonlySpace uint64           // 只读阈值
    
    // 状态
    Status          int              // 磁盘状态 (ReadWrite/ReadOnly/Unavailable)
    isLost          bool             // 是否丢失
    RejectWrite     bool             // 是否拒绝写入
    
    // 关联对象
    partitionMap    map[uint64]*DataPartition  // 该磁盘上的分区
    space           *SpaceManager              // 所属 SpaceManager
    
    // QoS 限流器 (5 种)
    limitRead       *util.IoLimiter  // 同步读限流
    limitWrite      *util.IoLimiter  // 同步写限流
    limitAsyncRead  *util.IoLimiter  // 异步读限流 (修复/预读)
    limitAsyncWrite *util.IoLimiter  // 异步写限流 (后台复制)
    limitDelete     *util.IoLimiter  // 删除限流
}
```

**数据结构关系**：

```
SpaceManager (1个)
    │
    ├── disks map
    │     ├── "/data1" → Disk
    │     │               └── partitionMap
    │     │                     ├── 1001 → DataPartition
    │     │                     └── 1002 → DataPartition
    │     └── "/data2" → Disk
    │                     └── partitionMap
    │                           └── 1003 → DataPartition
    │
    └── partitions map (全局索引)
          ├── 1001 → DataPartition
          ├── 1002 → DataPartition
          └── 1003 → DataPartition
```

---

## 核心问题

1. **SpaceManager 如何选择磁盘创建新分区？**
2. **磁盘故障如何检测和处理？**
3. **为什么需要 QoS 限流？**

---

## 1. 磁盘选择算法

当 Master 要求创建新分区时，SpaceManager 需要选择一个磁盘。

### Straw 算法

```go
// space_manager.go:573
func (manager *SpaceManager) selectDisk(decommissionedDisks []string) (d *Disk) {
    maxStraw := float64(0)
    for _, disk := range manager.disks {
        // 跳过不可写、下线中、丢失的磁盘
        if disk.Status != proto.ReadWrite || disk.isLost {
            continue
        }
        
        // 核心算法：Straw 加权随机
        straw := float64(rand.Intn(65536))
        straw = math.Log(straw/65536) / (float64(disk.Available) / util.GB)
        
        if straw > maxStraw {
            maxStraw = straw
            d = disk
        }
    }
    return d
}
```

**为什么用这个算法？**

这是 CRUSH (Controlled Replication Under Scalable Hashing) 的 Straw 算法变体：
- 每个磁盘抽一根"稻草" (straw)
- 稻草长度 = 随机值 × 权重函数
- 权重与**可用空间成正比**
- 选中稻草最长的磁盘

**效果**：
- 可用空间大的磁盘被选中概率高
- 但不是确定性的，有随机性
- 自然实现负载均衡

```
Disk1: 可用 500GB  → 被选中概率 ~50%
Disk2: 可用 300GB  → 被选中概率 ~30%
Disk3: 可用 200GB  → 被选中概率 ~20%
```

---

## 2. 磁盘故障检测

### 检测机制

```
┌─────────────────────────────────────────────────────────────┐
│                     故障检测三道防线                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 状态文件检测                                              │
│     每个磁盘有 .diskStatus 文件                               │
│     文件消失 → 磁盘可能挂载失败/被卸载                         │
│                                                              │
│  2. IO 错误计数                                               │
│     读写操作累计错误次数                                       │
│     超过 MaxErrCnt → 标记为 Unavailable                      │
│                                                              │
│  3. 分区级错误                                                │
│     单个分区连续错误                                          │
│     超过阈值 → 分区级故障隔离                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 为什么用文件检测而不是其他方式？

| 方式 | 问题 |
|------|------|
| 定期 `df` 命令 | 挂载点存在但磁盘故障时仍会返回 |
| SMART 监控 | 需要特殊权限，不跨平台 |
| 文件存在性检测 | 简单可靠，磁盘故障时一定失败 |

```go
// 检测逻辑
path := path.Join(disk.Path, ".diskStatus")
if _, err := os.Stat(path); err != nil {
    // 磁盘丢失！
    manager.processLostDisk(disk.Path)
}
```

### 故障处理流程

```
磁盘故障检测
       │
       ├─ .diskStatus 消失
       │       │
       │       ▼
       │  标记 isLost = true
       │       │
       │       ▼
       │  停止该磁盘所有分区的 Raft
       │       │
       │       ▼
       │  上报 Master
       │
       └─ IO 错误超过阈值
               │
               ▼
         Status = Unavailable
               │
               ▼
         拒绝新分区创建
               │
               ▼
         触发数据迁移
```

---

## 3. 空间管理

### 空间概念

```
┌─────────────────────────────────────────────────────────────┐
│                        磁盘 (10TB)                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Total = 10TB                                                │
│  ├── ReservedSpace = 5GB    (预留空间，防止写满)             │
│  ├── Allocated = 7TB        (已分配给分区)                   │
│  │   ├── Used = 5TB         (分区实际使用)                   │
│  │   └── 分区内空闲 = 2TB                                    │
│  └── Unallocated = 3TB-5GB  (可用于新分区)                   │
│                                                              │
│  Available = 5TB            (文件系统实际剩余)                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 为什么需要 ReservedSpace？

**场景**：磁盘写满
- Linux 文件系统满了，很多操作会失败
- 包括 Raft 写日志
- Raft 日志写不了 → 整个分区瘫痪

**解决**：
- 预留 5GB 空间
- 可用空间 < ReservedSpace 时，磁盘变为只读
- 但 Raft 和管理操作仍可进行

---

## 4. QoS 限流设计

### 为什么需要限流？

```
问题场景：
  用户 A: 大文件顺序读，打满磁盘带宽
  用户 B: 小文件随机读，延迟爆炸
  
没有 QoS:
  ┌─────────────────────────────────────┐
  │ 磁盘带宽 500MB/s                     │
  │ 用户A: 490MB/s  ██████████████████  │
  │ 用户B:  10MB/s  █                   │  ← 延迟 10x
  └─────────────────────────────────────┘
  
有 QoS:
  ┌─────────────────────────────────────┐
  │ 每用户限制 200MB/s                   │
  │ 用户A: 200MB/s  ████████            │
  │ 用户B: 200MB/s  ████████            │  ← 公平
  └─────────────────────────────────────┘
```

### 限流维度

| 限流器 | 控制对象 | 为什么单独限流 |
|--------|----------|----------------|
| `limitRead` | 同步读 | 前台请求，需要快速响应 |
| `limitWrite` | 同步写 | 前台请求，需要快速响应 |
| `limitAsyncRead` | 异步读 | 后台修复/预读，可以慢 |
| `limitAsyncWrite` | 异步写 | 后台复制，可以慢 |
| `limitDelete` | 删除操作 | 删除有 IO 开销，需要限流 |

**设计思想**：前台优先，后台让步

```go
// 前台写：使用 limitWrite
func (d *Disk) WriteSync(data []byte) {
    d.limitWrite.Wait()  // 限流等待
    // 实际写入
}

// 后台写（复制/修复）：使用 limitAsyncWrite
func (d *Disk) WriteAsync(data []byte) {
    d.limitAsyncWrite.Wait()  // 更严格的限流
    // 实际写入
}
```

---

## 5. 目录结构设计

```
/data1/                                 # 磁盘挂载点
├── .diskStatus                         # 磁盘在线标记
├── datapartition_1001_3/               # 分区目录
│   │                                   # 命名: datapartition_{ID}_{副本数}
│   ├── META                            # 分区元数据 JSON
│   ├── APPLY                           # Raft apply ID
│   ├── EXTENT_CRC                      # Extent CRC 校验
│   ├── EXTENT_META                     # Extent 元信息
│   ├── wal_1001/                       # Raft WAL
│   ├── 1, 2, ... 64                    # TinyExtent 文件
│   └── 1024, 1025, ...                 # NormalExtent 文件
├── expired_datapartition_1002_3/       # 待删除的过期分区
│                                       # 保留 7 天后删除（安全起见）
└── backup_datapartition_xxx/           # 备份分区（用于恢复）
```

**为什么过期分区要保留 7 天？**
- 误删除保护
- 给运维时间确认
- 7 天后自动清理

---

## 6. 关键代码

| 功能 | 位置 | 要点 |
|------|------|------|
| 磁盘选择 | space_manager.go:573 | Straw 算法，按可用空间加权随机 |
| 磁盘丢失检测 | space_manager.go | 检查 `.diskStatus` 文件存在性 |
| QoS 限流 | disk.go:97-102 | 五种限流器，前台/后台分离 |
| 空间计算 | disk.go `computeUsage()` | 区分 Allocated/Used/Available |

---

*更新时间：2026-04-27*
