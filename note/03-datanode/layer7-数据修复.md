# Layer 7: 数据修复

## 核心问题

1. **副本不一致如何检测？**
2. **如何选择修复源？**
3. **修复过程中新写入怎么处理？**

---

## 1. 为什么需要数据修复？

### 不一致的来源

```
┌─────────────────────────────────────────────────────────────────┐
│                      可能导致不一致的场景                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 网络分区                                                     │
│     Client → Leader → Follower1 (成功)                          │
│                    → Follower2 (网络断开，失败)                  │
│                                                                  │
│  2. 节点重启                                                     │
│     写入进行中，Follower 重启                                    │
│     重启后数据比其他副本少                                       │
│                                                                  │
│  3. 磁盘故障                                                     │
│     磁盘坏块导致数据损坏                                         │
│     CRC 校验失败                                                 │
│                                                                  │
│  4. 软件 bug                                                     │
│     极端情况下写入不完整                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 修复的目标

让所有副本的数据最终一致，采用**"最大版本获胜"**策略。

---

## 2. 检测机制

### Leader 主动检测

```
┌───────────────────────────────────────────────────────────────┐
│                     定期检测流程 (每 60 秒)                     │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Leader                                                        │
│    │                                                           │
│    │ 1. 收集本地 Extent 信息                                    │
│    │    [{ID:1001, Size:100MB, CRC:0xABC}, ...]               │
│    │                                                           │
│    │ 2. 请求各 Follower 的 Extent 信息                         │
│    │────────────────────▶ Follower1: [{ID:1001, Size:100MB}]  │
│    │────────────────────▶ Follower2: [{ID:1001, Size:80MB}]   │
│    │                                   ↑                       │
│    │ 3. 比较发现 Follower2 的 1001 只有 80MB                   │
│    │                                                           │
│    │ 4. 生成修复任务: Follower2 需要补 20MB                     │
│    │                                                           │
└────┴───────────────────────────────────────────────────────────┘
```

### 比较规则

| 情况 | 处理方式 |
|------|----------|
| Size 相同，CRC 相同 | 正常，无需修复 |
| Size 不同 | 以最大 Size 为准 |
| Size 相同，CRC 不同 | 按块比较，修复不一致的块 |
| Extent 不存在 | 从有该 Extent 的副本拷贝 |

---

## 3. 修复源选择

### 策略：优先 Leader，就近选择

```
Extent 1001 需要修复:
  Leader:    Size = 100MB  ← 优先选择
  Follower1: Size = 100MB  
  Follower2: Size = 80MB   ← 需要修复

原因:
  - Leader 通常有最新数据
  - 链式复制先写 Leader
  - 减少网络跳转
```

### 修复数据结构

```go
type RepairExtentInfo struct {
    storage.ExtentInfo        // Extent 基本信息
    Source string             // 数据源地址
}

type DataPartitionRepairTask struct {
    addr                string                      // 目标节点
    ExtentsToBeCreated  []*RepairExtentInfo        // 缺失的 Extent
    ExtentsToBeRepaired []*RepairExtentInfo        // 需要补齐的 Extent
}
```

---

## 4. 修复流程

### 流式修复

```
需要修复: Follower2 的 Extent 1001 (80MB → 100MB)
         │
         ▼
┌───────────────────────────────────────────────────────────────┐
│ Follower2                                                     │
│         │                                                     │
│         │ 1. 连接 Leader                                      │
│         │                                                     │
│         │ 2. 发送修复读请求                                    │
│         │    OpExtentRepairRead(extent=1001, offset=80MB)     │
│         │────────────────────────────────────────────▶ Leader │
│         │                                                     │
│         │ 3. Leader 流式返回 80MB-100MB 的数据                 │
│         │◀────────────────────────────────────────────        │
│         │    [64KB block] [64KB block] ... [EOF]              │
│         │                                                     │
│         │ 4. 写入本地 ExtentStore                              │
│         │    每个 block 校验 CRC 后写入                        │
│         │                                                     │
└─────────┴─────────────────────────────────────────────────────┘
```

### 为什么是流式？

```
Extent 可能很大 (128MB)
  - 一次性读取: 内存压力大
  - 流式读取: 每次 64KB，边读边写，内存可控
```

---

## 5. 修复过程中的新写入

### 问题

```
T1: 开始修复 Extent 1001 (0-100MB)
T2: 新写入到达 Extent 1001 (追加 100-110MB)
T3: 修复完成

如果不处理:
  修复只拷贝了 0-100MB
  新写入的 100-110MB 可能丢失
```

### 解决方案

```
1. 修复期间分区不变为 ReadOnly
   - 新写入正常进行
   - 链式复制会同步到所有副本

2. 修复结束后再次检测
   - 发现还有不一致，再次修复
   - 直到所有副本一致

3. 最终一致性保证
   - 多轮修复后收敛
   - 通常 1-2 轮即可
```

---

## 6. TinyExtent 的特殊处理

### 问题

TinyExtent 有删除记录 (punch hole)，修复时需要同步删除记录。

```
Leader TinyExtent 1:
  [file_a] [hole] [file_c] [file_d]
            ↑
        已删除的 file_b

Follower TinyExtent 1:
  [file_a] [file_b] [file_c]  ← 还没有删除记录
```

### 解决

```
修复流程:
  1. 同步 TINYEXTENT_DELETE 文件
  2. Follower 读取删除记录
  3. 对相应位置执行 punch hole
  4. 然后补齐数据
```

---

## 7. 修复触发时机

| 触发条件 | 间隔 | 说明 |
|----------|------|------|
| 定时检查 | 60 秒 | 常规一致性检查 |
| 分区创建后 | 立即 | 初始化 TinyExtent |
| 节点重启后 | 立即 | 快速恢复一致性 |
| Master 指令 | 按需 | 管理员手动触发 |

### 检查间隔随机化

```go
const (
    DpCheckBaseInterval = 7200   // 2 小时
    DpCheckRandomRange  = 1800   // 随机 ±30 分钟
)

实际间隔 = 7200 + rand(-1800, 1800)
         = 90 分钟 ~ 150 分钟
```

**为什么随机化？**
- 避免所有分区同时检查
- 减少网络和 CPU 峰值

---

## 8. 修复限流

### 问题

```
场景: 节点重启后有 100 个分区需要修复

不限流:
  - 100 个分区同时修复
  - 磁盘 IO 打满
  - 正常读写延迟爆炸
```

### 解决

```go
// 限制并发修复的 Extent 数量
const MaxExtentRepairReadLimit = 1

// 每个磁盘一个限流 channel
disk.extentRepairReadLimit = make(chan struct{}, MaxExtentRepairReadLimit)

// 修复前获取令牌
disk.extentRepairReadLimit <- struct{}{}
defer func() { <-disk.extentRepairReadLimit }()
```

---

## 9. 关键代码

| 功能 | 位置 | 要点 |
|------|------|------|
| 修复入口 | data_partition_repair.go:102 | `repair()` |
| 构建任务 | data_partition_repair.go:166 | `buildDataPartitionRepairTask()` |
| 流式修复 | data_partition_repair.go | `streamRepairExtent()` |
| 修复限流 | disk.go | `extentRepairReadLimit` |
| 检查间隔 | partition.go:62-65 | `DpCheckBaseInterval` |

---

## 10. DataNode 学习总结

```
┌─────────────────────────────────────────────────────────────────┐
│                      DataNode 架构全景                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: 启动配置                                               │
│    DataNode 启动 → 注册 Master → 加载磁盘 → 加载分区              │
│                                                                  │
│  Layer 2: 空间管理                                               │
│    SpaceManager 管理多磁盘，Straw 算法选择磁盘                    │
│                                                                  │
│  Layer 3: 数据分区                                               │
│    DataPartition (120GB) 是管理单元，每个分区独立 Raft           │
│                                                                  │
│  Layer 4: Extent 存储                                            │
│    TinyExtent (小文件共享) + NormalExtent (大文件独占)           │
│    Punch hole 实现小文件删除                                     │
│                                                                  │
│  Layer 5: 链式复制                                               │
│    追加写: Leader → F1 → F2 (链式复制，高吞吐)                   │
│                                                                  │
│  Layer 6: Raft 一致性                                            │
│    随机写 + 成员变更走 Raft (强一致)                              │
│                                                                  │
│  Layer 7: 数据修复                                               │
│    定期检测 → 比较 → 流式修复 (最大版本获胜)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*更新时间：2026-04-27*
