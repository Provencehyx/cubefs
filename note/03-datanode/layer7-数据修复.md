# Layer 7: 数据修复

## 1. 修复机制概述

DataNode 通过定期比较副本之间的 Extent 信息来发现和修复数据不一致。

```
┌─────────────────────────────────────────────────────────────┐
│                      数据修复流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Leader                 Follower1            Follower2     │
│     │                       │                     │         │
│     │   1. 收集 Extent 信息  │                     │         │
│     │◄──────────────────────│                     │         │
│     │◄────────────────────────────────────────────│         │
│     │                                                       │
│     │   2. 比较各副本，找出最大/最新版本                      │
│     │                                                       │
│     │   3. 生成修复任务                                      │
│     │                                                       │
│     │   4. 通知需要修复的节点                                 │
│     │────────────────────▶│                     │           │
│     │──────────────────────────────────────────▶│           │
│     │                                                       │
│     │   5. 执行修复（从源节点拉取数据）                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 修复任务结构

```go
// data_partition_repair.go:44
type RepairExtentInfo struct {
    storage.ExtentInfo
    Source string    // 数据源地址（拥有完整数据的节点）
}

// data_partition_repair.go:57
type DataPartitionRepairTask struct {
    TaskType                       uint8              // Normal / Tiny
    addr                           string             // 目标节点地址
    extents                        map[uint64]*RepairExtentInfo
    ExtentsToBeCreated             []*RepairExtentInfo  // 需要创建的 Extent
    ExtentsToBeRepaired            []*RepairExtentInfo  // 需要修复的 Extent
    LeaderTinyDeleteRecordFileSize int64
    LeaderAddr                     string
}
```

## 3. 两种修复类型

### 3.1 Normal Extent 修复

```
Normal Extent 修复流程:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  1. Leader 收集所有 Follower 的 Extent 信息                 │
│                                                            │
│  2. 对每个 Extent，比较所有副本找出最大 Size                 │
│                                                            │
│  3. 本地 Extent Size < 最大 Size → 加入修复列表              │
│                                                            │
│  4. 从拥有完整数据的节点拉取缺失部分                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 3.2 Tiny Extent 修复

```
Tiny Extent 修复流程:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  1. 创建分区时，所有 Tiny Extent 加入待修复列表              │
│                                                            │
│  2. Leader 定期收集各 Follower 的 Tiny Extent 信息          │
│                                                            │
│  3. 比较各副本，找出最大 Size                               │
│                                                            │
│  4. 本地 Size < 最大 Size → 修复                            │
│                                                            │
│  5. 同步 Tiny Extent 删除记录                               │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## 4. 修复主流程

```go
// data_partition_repair.go:102
func (dp *DataPartition) repair(extentType uint8) {
    // 1. 获取损坏的 Tiny Extent 列表
    tinyExtents := dp.brokenTinyExtents()
    
    // 2. 构建修复任务（收集各副本信息）
    repairTasks := make([]*DataPartitionRepairTask, len(replica))
    dp.buildDataPartitionRepairTask(repairTasks, extentType, tinyExtents, replica)
    
    // 3. 比较所有副本，确定哪些需要修复
    availableTinyExtents, brokenTinyExtents := dp.prepareRepairTasks(repairTasks)
    
    // 4. 通知各副本执行修复
    dp.NotifyExtentRepair(repairTasks)
    
    // 5. Leader 执行自己的修复任务
    dp.DoRepair(repairTasks)
    
    // 6. 更新 Extent 状态
    dp.sendAllTinyExtentsToC(extentType, availableTinyExtents, brokenTinyExtents)
}
```

## 5. 构建修复任务

```go
// data_partition_repair.go:166
func (dp *DataPartition) buildDataPartitionRepairTask(...) {
    // 1. 获取本地 Extent 信息
    extents, leaderTinyDeleteRecordFileSize, _ := dp.getLocalExtentInfo(extentType, tinyExtents)
    
    // 2. 创建 Leader 的修复任务
    repairTasks[0] = NewDataPartitionRepairTask(extents, 
        leaderTinyDeleteRecordFileSize, 
        dp.dataNode.localServerAddr,
        dp.dataNode.localServerAddr, 
        extentType)
    
    // 3. 从各 Follower 获取 Extent 信息并创建任务
    for _, follower := range followers {
        extentFiles := dp.getRemoteExtentInfo(follower, extentType, tinyExtents)
        repairTasks[index] = NewDataPartitionRepairTask(extentFiles, ...)
    }
}
```

## 6. 比较与准备修复

```
比较逻辑:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Extent 1001:                                               │
│    Replica1: Size = 100MB, CRC = 0xABC                     │
│    Replica2: Size = 100MB, CRC = 0xABC                     │
│    Replica3: Size = 80MB,  CRC = 0xDEF  ← 需要修复          │
│                                                             │
│  最大 Size = 100MB                                          │
│  Replica3 需要从 Replica1 或 Replica2 补齐 20MB             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 7. 执行修复

```go
// DoRepair 执行修复
func (dp *DataPartition) DoRepair(repairTasks []*DataPartitionRepairTask) {
    for _, task := range repairTasks {
        // 处理需要创建的 Extent
        for _, extent := range task.ExtentsToBeCreated {
            dp.streamRepairExtent(extent)
        }
        
        // 处理需要修复的 Extent
        for _, extent := range task.ExtentsToBeRepaired {
            dp.streamRepairExtent(extent)
        }
    }
}

// streamRepairExtent 从源节点流式拉取数据
func (dp *DataPartition) streamRepairExtent(extentInfo *RepairExtentInfo) {
    // 1. 连接源节点
    conn := gConnPool.GetConnect(extentInfo.Source)
    
    // 2. 发送修复读请求
    packet := repl.NewExtentRepairReadPacket(...)
    packet.WriteToConn(conn)
    
    // 3. 接收数据并写入本地
    for {
        reply := repl.NewPacket()
        reply.ReadFromConn(conn)
        
        // 写入本地 ExtentStore
        dp.extentStore.Write(...)
    }
}
```

## 8. 修复触发条件

| 触发条件 | 说明 |
|----------|------|
| 定时检查 | 周期性检查各副本一致性 |
| 新分区创建 | 创建分区后初始化 Tiny Extent |
| 副本恢复 | 副本重新上线后同步数据 |
| Master 指令 | Master 检测到不一致后触发 |

## 9. 修复相关常量

```go
// extent_store.go
const (
    RepairInterval = 60              // 修复检查间隔（秒）
    UpdateCrcInterval = 600          // CRC 更新间隔（秒）
)

// partition.go
const (
    DpCheckBaseInterval = 7200       // 分区检查基础间隔
    DpCheckRandomRange = 1800        // 随机范围
)
```

## 10. 修复流程图

```
┌──────────────────────────────────────────────────────────────┐
│                    完整修复流程                               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┐                                                 │
│  │ 定时器   │                                                 │
│  └────┬────┘                                                 │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            repair(extentType)                        │    │
│  └─────────────────────────────────────────────────────┘    │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │     buildDataPartitionRepairTask()                   │    │
│  │     - 收集本地 Extent 信息                           │    │
│  │     - 收集各 Follower Extent 信息                    │    │
│  └─────────────────────────────────────────────────────┘    │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │     prepareRepairTasks()                             │    │
│  │     - 比较各副本                                     │    │
│  │     - 确定需要创建/修复的 Extent                     │    │
│  └─────────────────────────────────────────────────────┘    │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │     NotifyExtentRepair()                             │    │
│  │     - 通知各 Follower 执行修复                       │    │
│  └─────────────────────────────────────────────────────┘    │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │     DoRepair()                                       │    │
│  │     - 执行 Leader 自己的修复任务                     │    │
│  │     - streamRepairExtent() 拉取数据                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 11. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| RepairExtentInfo | data_partition_repair.go:44 |
| DataPartitionRepairTask | data_partition_repair.go:57 |
| repair 主函数 | data_partition_repair.go:102 |
| 构建修复任务 | data_partition_repair.go:166 |
| 流式修复 | data_partition_repair.go (streamRepairExtent) |

---

## 总结

DataNode 笔记完成，共 7 层：

1. **启动与配置** - DataNode 结构与启动流程
2. **磁盘与空间管理** - SpaceManager 和 Disk
3. **DataPartition** - 数据分区核心结构
4. **Extent 存储** - 底层存储引擎
5. **副本复制** - 链式复制协议
6. **Raft 一致性** - 控制面一致性
7. **数据修复** - 副本修复机制

*创建时间：2026-04-12*
