# Layer 4: Extent 存储引擎

## 1. Extent 概念

Extent 是 CubeFS 存储数据的基本单位，类似于文件系统的块（block），但更大。

```
┌─────────────────────────────────────────────────────────────┐
│                        文件数据                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Block 1  │  Block 2  │  Block 3  │  ...  │ Block N │   │
│   └─────────────────────────────────────────────────────┘   │
│                            │                                 │
│                            ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           Extent (最大 128MB)                        │   │
│   │   ┌─────────┬─────────┬─────────┬───────────────┐   │   │
│   │   │  64KB   │  64KB   │  64KB   │     ...       │   │   │
│   │   │ Block   │ Block   │ Block   │               │   │   │
│   │   └─────────┴─────────┴─────────┴───────────────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 2. Extent 类型

| 类型 | ID 范围 | 最大大小 | 用途 |
|------|---------|----------|------|
| Tiny Extent | 1 - 64 | 128MB | 小文件（多文件共享） |
| Normal Extent | >= 1024 | 128MB | 普通文件 |

```go
// extent_store.go:51-54
const (
    TinyExtentCount   = 64
    TinyExtentStartID = 1
    MinExtentID       = 1024       // Normal Extent 起始 ID
    MaxExtentCount    = 20000      // 每个分区最多 20000 个 Extent
)
```

## 3. ExtentStore 结构

ExtentStore 是 Extent 的存储管理器。

```go
// extent_store.go:125
type ExtentStore struct {
    dataPath               string                    // 数据目录
    baseExtentID           uint64                    // 下一个可用 Extent ID
    extentInfoMap          map[uint64]*ExtentInfo    // Extent 信息映射
    eiMutex                sync.RWMutex
    
    cache                  *ExtentCache              // Extent 缓存
    storeSize              int                       // 存储大小
    partitionID            uint64                    // 所属分区 ID
    
    // 文件句柄
    metadataFp             *os.File                  // EXTENT_META
    verifyExtentFp         *os.File                  // EXTENT_CRC
    tinyExtentDeleteFp     *os.File                  // TINYEXTENT_DELETE
    normalExtentDeleteFp   *os.File                  // NORMALEXTENT_DELETE
    
    // Tiny Extent 管理
    availableTinyExtentC   chan uint64               // 可用 Tiny Extent 通道
    brokenTinyExtentC      chan uint64               // 损坏 Tiny Extent 通道
    
    closed                 int32
}
```

## 4. ExtentInfo 结构

```go
// extent.go:99
type ExtentInfo struct {
    FileID              uint64    // Extent ID
    Size                uint64    // 大小
    Crc                 uint32    // CRC 校验
    ModifyTime          int64     // 修改时间
    IsDeleted           bool      // 是否已删除
    SnapshotDataOff     uint64    // 快照数据偏移
    ApplyID             uint64    // Raft 应用 ID
}
```

## 5. 存储文件结构

```
datapartition_1001_3/
├── EXTENT_META              # Extent 元信息（下一个可用 ID）
├── EXTENT_CRC               # 所有 Extent 的 CRC 校验值
├── TINYEXTENT_DELETE        # Tiny Extent 删除记录
├── NORMALEXTENT_DELETE      # Normal Extent 删除记录
│
├── 1                        # Tiny Extent 1
├── 2                        # Tiny Extent 2
├── ...
├── 64                       # Tiny Extent 64
│
├── 1024                     # Normal Extent 1024
├── 1025                     # Normal Extent 1025
└── ...
```

## 6. 写入参数

```go
// extent.go:84
type WriteParam struct {
    ExtentID    uint64    // 目标 Extent ID
    Offset      int64     // 写入偏移
    Size        int64     // 写入大小
    Data        []byte    // 数据
    Crc         uint32    // CRC 校验
    WriteType   int       // 写入类型
    IsSync      bool      // 是否同步写
    IsHole      bool      // 是否为空洞
    IsRepair    bool      // 是否为修复写
}
```

### 写入类型

| 类型 | 值 | 说明 |
|------|-----|------|
| AppendWriteType | 1 | 追加写 |
| RandomWriteType | 2 | 随机写 |
| AppendRandomWriteType | 4 | 追加+随机写 |

## 7. 核心操作流程

### 7.1 创建 Extent

```
ExtentStore.Create(extentID)
       │
       ├── 检查 ID 有效性
       │
       ├── 创建文件: os.OpenFile(extentID, O_CREATE|O_RDWR|O_EXCL)
       │
       ├── 更新 extentInfoMap
       │
       └── 返回 Extent 句柄
```

### 7.2 写入数据

```
ExtentStore.Write(WriteParam)
       │
       ├── 获取/打开 Extent 文件
       │
       ├── 定位偏移: file.Seek(offset)
       │
       ├── 写入数据: file.Write(data)
       │
       ├── 计算 CRC
       │
       ├── 更新 ExtentInfo
       │
       └── 可选: file.Sync()
```

### 7.3 读取数据

```
ExtentStore.Read(extentID, offset, size)
       │
       ├── 从 cache 获取 Extent
       │       │
       │       └── 未命中则打开文件
       │
       ├── 定位偏移
       │
       └── 读取数据
```

## 8. Tiny Extent 特殊处理

Tiny Extent 用于存储小文件，多个小文件可以共享同一个 Extent。

```
┌─────────────────────────────────────────────────────────┐
│                   Tiny Extent (ID=1)                     │
├─────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │ File A   │  │ File B   │  │ File C   │  │  空闲  │  │
│  │ 4KB      │  │ 12KB     │  │ 8KB      │  │        │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  │
│  offset=0      offset=4KB   offset=16KB                 │
└─────────────────────────────────────────────────────────┘

删除 File B:
- 不真正删除数据
- 使用 punch hole (fallocate) 释放空间
- 记录到 TINYEXTENT_DELETE
```

### Tiny Extent 删除记录

```go
// 每条删除记录 24 字节
type TinyDeleteRecord struct {
    ExtentID   uint64    // 8 bytes
    Offset     uint64    // 8 bytes  
    Size       uint64    // 8 bytes
}
```

## 9. CRC 校验

每个 Extent 按块（64KB）计算 CRC：

```
┌──────────────────────────────────────────────────┐
│                  Extent 文件                      │
├──────────────────────────────────────────────────┤
│  Block 0  │  Block 1  │  Block 2  │  ...        │
│  64KB     │  64KB     │  64KB     │             │
│  CRC0     │  CRC1     │  CRC2     │             │
└──────────────────────────────────────────────────┘

EXTENT_CRC 文件:
┌──────────────────────────────────────────────────┐
│ ExtentID │ BlockCRC0 │ BlockCRC1 │ BlockCRC2 │...│
└──────────────────────────────────────────────────┘
```

## 10. 关键常量

```go
// extent.go & extent_store.go
const (
    ExtentMaxSize          = 128 * 1024 * 1024   // 128MB
    BlockSize              = 64 * 1024           // 64KB (util.BlockSize)
    TinyExtentCount        = 64
    MaxExtentCount         = 20000
    UpdateCrcInterval      = 600                 // 10 分钟
    RepairInterval         = 60                  // 1 分钟
)
```

## 11. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| ExtentStore 结构 | storage/extent_store.go:125 |
| ExtentInfo 结构 | storage/extent.go:99 |
| WriteParam 结构 | storage/extent.go:84 |
| 创建 ExtentStore | storage/extent_store.go:165 |
| Extent 常量 | storage/extent_store.go:45 |

---

## 下一步
- Layer 5: 副本复制协议

*创建时间：2026-04-12*
