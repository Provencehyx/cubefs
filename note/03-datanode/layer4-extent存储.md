# Layer 4: Extent 存储引擎

## 核心问题

1. **TinyExtent 为什么只有 64 个？**
2. **小文件删除为什么用 punch hole？**
3. **为什么 NormalExtent ID 从 1024 开始？**

---

## 1. Extent 双轨设计

### 问题：小文件存储效率

```
场景：100 万个 1KB 小文件

方案 A：每个文件一个 Extent 文件
  → 100 万个磁盘文件
  → inode 爆炸
  → 目录遍历慢
  → 元数据开销 >> 数据本身

方案 B：多个小文件共享一个 Extent
  → 64 个 Extent 文件
  → 追加写入
  → 高效
```

### 设计决策

| Extent 类型 | ID 范围 | 场景 | 设计原因 |
|-------------|---------|------|----------|
| TinyExtent | 1-64 | 小文件 (<1MB) | 共享 Extent，减少文件数 |
| NormalExtent | ≥1024 | 大文件 | 独占 Extent，便于管理 |
| (预留) | 65-1023 | 未来扩展 | 预留空间 |

---

## 2. TinyExtent 详解

### 为什么是 64 个？

```
每个 TinyExtent 最大 128MB
64 个 TinyExtent = 8GB 小文件空间

每个分区 120GB：
  - 假设 20% 是小文件 → 24GB
  - 需要 24GB / 128MB ≈ 192 个 TinyExtent
  
为什么选 64？
  - 实测 64 个够用（大部分分区小文件比例不高）
  - 太多 TinyExtent 增加管理复杂度
  - 空间不够时可以分配更多分区
```

### 追加写入模型

```
TinyExtent (ID=1)
┌─────────────────────────────────────────────────────────┐
│ [file_a: 4KB] [file_b: 12KB] [file_c: 8KB] [空闲...]   │
│ offset=0      offset=4096    offset=16384              │
└─────────────────────────────────────────────────────────┘

写入 file_d (2KB):
  1. 找到 file_c 末尾位置
  2. 追加写入
  3. 返回 (extentID=1, offset=24576, size=2048)
```

### 小文件删除：Punch Hole

**问题**：删除中间的文件，怎么回收空间？

```
删除 file_b 后的 Extent:
┌─────────────────────────────────────────────────────────┐
│ [file_a: 4KB] [   空洞   ] [file_c: 8KB] [空闲...]     │
│ offset=0      offset=4096   offset=16384               │
└─────────────────────────────────────────────────────────┘
```

**解决：Linux fallocate + FALLOC_FL_PUNCH_HOLE**

```go
// storage/extent.go:787
err = fallocate(int(e.file.Fd()), 
    util.FallocFLPunchHole|util.FallocFLKeepSize, 
    offset, size)
```

**效果**：
- 文件大小不变（KEEP_SIZE）
- 该区域变为"空洞"
- 实际不占用磁盘空间（稀疏文件）
- 读取空洞返回全零

**为什么不用其他方案？**

| 方案 | 问题 |
|------|------|
| 物理移动数据 | 太慢，需要移动后面所有数据 |
| 标记删除不回收 | 空间永远不释放 |
| 定期整理压缩 | 复杂，影响性能 |
| Punch hole | 立即释放空间，O(1) 复杂度 |

---

## 3. NormalExtent 详解

### 为什么 ID 从 1024 开始？

```
ID 空间划分:
┌───────┬────────────┬─────────────────┐
│ 1-64  │  65-1023   │   1024+         │
│ Tiny  │   预留     │   Normal        │
└───────┴────────────┴─────────────────┘
```

**原因**：
- 简单的 ID 划分规则
- `if (id >= 1024) → Normal; else if (id <= 64) → Tiny`
- 预留空间给未来（比如 MediumExtent）

### NormalExtent 生命周期

```
创建: 文件写入需要新 Extent
  1. 分配 ID (baseExtentID++)
  2. 创建磁盘文件 (文件名就是 ID)
  3. 写入数据
  
删除: 文件删除
  1. 记录到 NORMALEXTENT_DELETE
  2. 延迟 4 小时删除物理文件 (防止误删)
  3. 定期清理
```

**为什么延迟删除？**
- 副本同步有延迟
- 防止在同步完成前删除
- 给运维留出恢复时间

---

## 4. CRC 校验设计

### 分块 CRC

```
Extent 文件 (128MB)
┌──────────┬──────────┬──────────┬────────┐
│ Block 0  │ Block 1  │ Block 2  │  ...   │  每块 64KB
│   CRC0   │   CRC1   │   CRC2   │        │
└──────────┴──────────┴──────────┴────────┘
     │          │          │
     ▼          ▼          ▼
EXTENT_CRC 文件: [CRC0][CRC1][CRC2]...
```

**为什么按 64KB 分块？**
- 太小：CRC 元数据过多
- 太大：校验粒度太粗，一个错误整块重传
- 64KB：一次 IO 合适的大小

### CRC 校验时机

| 时机 | 动作 |
|------|------|
| 写入时 | 计算 CRC，存入 EXTENT_CRC |
| 定期检查 (10分钟) | 重算 CRC，对比发现损坏 |
| 修复时 | 按块校验，只修复不一致的块 |

---

## 5. 存储文件结构

```
datapartition_1001_3/
├── EXTENT_META              # baseExtentID (下一个可分配的 ID)
├── EXTENT_CRC               # 所有 Extent 的分块 CRC
├── TINYEXTENT_DELETE        # Tiny 删除记录 (extentID, offset, size)
├── NORMALEXTENT_DELETE      # Normal 删除记录 (extentID, deleteTime)
│
├── 1                        # TinyExtent 1
├── 2                        # TinyExtent 2
├── ...
├── 64                       # TinyExtent 64
│
├── 1024                     # NormalExtent 1024
├── 1025                     # NormalExtent 1025
└── ...
```

### 各文件作用

| 文件 | 内容 | 更新频率 |
|------|------|----------|
| `EXTENT_META` | 下一个可分配 ID | 每次创建新 Extent |
| `EXTENT_CRC` | 所有块的 CRC | 每次写入 |
| `TINYEXTENT_DELETE` | (ID, offset, size) | 删除小文件时 |
| `NORMALEXTENT_DELETE` | (ID, deleteTime) | 删除大文件时 |

---

## 6. 写入流程

### 追加写（顺序写）

```
Client: Write(data, size=100KB)
        │
        ▼
DataPartition.handleWrite()
        │
        ├── 1. 选择 Extent
        │      TinyExtent (size < 1MB) 或 NormalExtent
        │
        ├── 2. ExtentStore.Write()
        │      file.Seek(offset) + file.Write(data)
        │
        ├── 3. 计算 CRC，更新 EXTENT_CRC
        │
        └── 4. 链式复制到 Follower（见 Layer 5）
```

### 随机写（覆盖写）

```
Client: Write(data, offset=1000, size=100)
        │
        ▼
DataPartition.handleRandomWrite()
        │
        ├── 1. 提交 Raft 日志 (确保顺序)
        │
        ├── 2. Apply 后: ExtentStore.Write(offset, data)
        │
        └── 3. 更新受影响块的 CRC
```

**为什么随机写需要 Raft？**
- 多副本可能同时收到不同的覆盖写
- 需要 Raft 确定全局顺序
- 否则数据不一致

---

## 7. 关键常量

```go
// storage/extent_store.go
const (
    TinyExtentCount        = 64            // TinyExtent 数量
    TinyExtentStartID      = 1             // TinyExtent 起始 ID
    MinExtentID            = 1024          // NormalExtent 起始 ID
    MaxExtentCount         = 20000         // 每分区最大 Extent 数
    
    NormalExtentDeleteRetainTime = 3600 * 4  // 延迟删除 4 小时
    UpdateCrcInterval            = 600       // CRC 检查间隔 10 分钟
)
```

---

## 8. 关键代码

| 功能 | 位置 | 要点 |
|------|------|------|
| ExtentStore | storage/extent_store.go:125 | Extent 管理器 |
| Punch Hole | storage/extent.go:787 | `fallocate(PUNCH_HOLE)` |
| 延迟删除 | storage/extent_store.go:63 | 4 小时后物理删除 |
| CRC 分块 | storage/persistence_crc.go | 64KB 块校验 |

---

*更新时间：2026-04-27*
