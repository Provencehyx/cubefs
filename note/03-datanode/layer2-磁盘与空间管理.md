# Layer 2: 磁盘与空间管理

## 1. SpaceManager 结构

SpaceManager 是 DataNode 的核心组件，负责管理所有磁盘和数据分区。

```go
// space_manager.go:48
type SpaceManager struct {
    clusterID      string
    disks          map[string]*Disk          // 磁盘映射 (路径 -> Disk)
    partitions     map[uint64]*DataPartition // 分区映射 (ID -> Partition)
    raftStore      raftstore.RaftStore       // Raft 存储
    nodeID         uint64
    
    diskMutex      sync.RWMutex              // 磁盘锁
    partitionMutex sync.RWMutex              // 分区锁
    
    stats          *Stats                    // 统计信息
    diskList       []string                  // 磁盘路径列表
    dataNode       *DataNode                 // 反向引用
    diskUtils      map[string]*atomicutil.Float64  // 磁盘利用率
}
```

## 2. Disk 结构

```go
// disk.go:73
type Disk struct {
    sync.RWMutex
    Path          string                    // 磁盘路径
    
    // 错误统计
    ReadErrCnt    uint64                    // 读错误次数
    WriteErrCnt   uint64                    // 写错误次数
    MaxErrCnt     int                       // 最大容错次数
    
    // 空间信息
    Total         uint64                    // 总空间
    Used          uint64                    // 已用空间
    Available     uint64                    // 可用空间
    Unallocated   uint64                    // 未分配空间
    Allocated     uint64                    // 已分配空间
    ReservedSpace uint64                    // 预留空间
    
    // 状态
    Status        int                       // 状态 (READONLY/Unavailable)
    isLost        bool                      // 是否丢失
    RejectWrite   bool                      // 拒绝写入
    decommission  bool                      // 下线中
    
    // 分区管理
    partitionMap  map[uint64]*DataPartition // 该磁盘上的分区
    space         *SpaceManager             // 反向引用
    
    // QoS 限流器
    limitRead       *util.IoLimiter         // 读限流
    limitWrite      *util.IoLimiter         // 写限流
    limitAsyncRead  *util.IoLimiter         // 异步读限流
    limitAsyncWrite *util.IoLimiter         // 异步写限流
    limitDelete     *util.IoLimiter         // 删除限流
}
```

## 3. 层级关系

```
┌─────────────────────────────────────────────────────────────┐
│                       DataNode                              │
│                          │                                  │
│                          ▼                                  │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                   SpaceManager                         │ │
│  │                                                        │ │
│  │   disks: map[string]*Disk                             │ │
│  │   ┌────────┬────────┬────────┐                        │ │
│  │   │ Disk 1 │ Disk 2 │ Disk 3 │  ...                   │ │
│  │   └───┬────┴───┬────┴───┬────┘                        │ │
│  │       │        │        │                              │ │
│  │   partitions: map[uint64]*DataPartition               │ │
│  │   ┌────┬────┬────┬────┬────┬────┬────┐               │ │
│  │   │DP1│DP2 │DP3 │DP4 │DP5 │DP6 │... │               │ │
│  │   └────┴────┴────┴────┴────┴────┴────┘               │ │
│  │                                                        │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

Disk 与 DataPartition 的关系：
┌────────────────────────────────────────┐
│              Disk 1                     │
│   Path: /data1                          │
│   partitionMap:                         │
│     ├── DP 1001                         │
│     ├── DP 1002                         │
│     └── DP 1003                         │
└────────────────────────────────────────┘
```

## 4. 磁盘目录结构

```
/data1/                              # 磁盘挂载点
├── .diskStatus                      # 磁盘状态文件（检测磁盘是否在线）
├── datapartition_1001_3/            # 分区目录 (ID_副本数)
│   ├── 1                            # Extent 文件
│   ├── 2
│   ├── ...
│   ├── EXTENT_CRC                   # CRC 校验文件
│   ├── EXTENT_META                  # 元数据文件
│   └── wal_1001/                    # Raft WAL 日志
├── datapartition_1002_3/
└── expired_datapartition_xxx/       # 过期分区（待删除）
```

## 5. 磁盘状态管理

### 5.1 状态类型

| 状态 | 说明 |
|------|------|
| `ReadWrite` | 正常读写 |
| `ReadOnly` | 只读（空间不足） |
| `Unavailable` | 不可用（错误过多） |
| `Lost` | 磁盘丢失 |
| `Decommission` | 下线中 |

### 5.2 磁盘丢失检测

```go
// space_manager.go:227
func (manager *SpaceManager) checkAllDisksLost() {
    for _, disk := range manager.disks {
        path := path.Join(disk.Path, DiskStatusFile)
        if _, err := os.Stat(path); err != nil {
            // 磁盘丢失处理
            manager.processLostDisk(disk.Path)
        }
    }
}
```

通过检测 `.diskStatus` 文件是否存在来判断磁盘是否在线。

## 6. 空间计算

```go
// disk.go
func (d *Disk) computeUsage() (err error) {
    // 获取文件系统信息
    fs := syscall.Statfs_t{}
    syscall.Statfs(d.Path, &fs)
    
    d.Total = fs.Blocks * uint64(fs.Bsize)
    d.Available = fs.Bavail * uint64(fs.Bsize)
    d.Used = d.Total - d.Available
    
    // 计算未分配空间
    d.Unallocated = d.Total - d.ReservedSpace - d.Allocated
}
```

**空间关系：**
```
Total = Used + Available
Allocated = 所有分区实际占用
Unallocated = Total - ReservedSpace - Allocated
```

## 7. QoS 限流

每个磁盘有独立的 IO 限流器：

```go
// disk.go:177
d.limitRead = util.NewIOLimiter(diskReadFlow, diskReadIocc)
d.limitWrite = util.NewIOLimiter(diskWriteFlow, diskWriteIocc)
d.limitAsyncRead = util.NewIOLimiter(diskAsyncReadFlow, diskAsyncReadIocc)
d.limitAsyncWrite = util.NewIOLimiter(diskAsyncWriteFlow, diskAsyncWriteIocc)
d.limitDelete = util.NewIOLimiter(diskDeleteFlow, diskDeleteIocc)
```

| 限流维度 | 说明 |
|----------|------|
| IOPS | 每秒 IO 操作数 |
| Flow | 每秒流量 (bytes) |
| IOCC | IO 并发数 |

## 8. 核心流程

### 8.1 加载磁盘

```
SpaceManager.LoadDisk()
       │
       ▼
  NewDisk(path)
       │
       ├── computeUsage()        计算空间
       ├── updateSpaceInfo()     更新空间信息
       ├── 初始化限流器
       └── startScheduleToUpdateSpaceInfo()  启动定时更新
       │
       ▼
  disk.RestorePartition()
       │
       └── 遍历 datapartition_* 目录
           │
           └── LoadDataPartition()  加载每个分区
```

### 8.2 创建分区

```
Master 请求创建分区
       │
       ▼
SpaceManager.CreatePartition()
       │
       ├── 选择磁盘 (负载均衡)
       ├── 创建目录 datapartition_ID_ReplicaNum
       ├── NewDataPartition()
       └── 注册到 SpaceManager
```

## 9. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| SpaceManager 结构 | space_manager.go:48 |
| Disk 结构 | disk.go:73 |
| 创建 Disk | disk.go:129 |
| 磁盘丢失检测 | space_manager.go:216 |
| 空间计算 | disk.go (computeUsage) |

---

## 下一步
- Layer 3: DataPartition 核心结构与操作

*创建时间：2026-04-12*
