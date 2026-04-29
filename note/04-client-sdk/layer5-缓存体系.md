# Layer 5: Client 缓存体系

> 核心问题：**Client 有哪些缓存？如何协作减少 RPC 开销？**

---

## 1. 缓存分层架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Level 0: 内核层 - Linux Page Cache                                          │
│  ─────────────────────────────────────────────────────────────────────────  │
│  位置: 内核内存                                                               │
│  控制: FUSE KeepCache 选项 (keepCache=true 启用)                              │
│  作用: 缓存文件内容，避免重复读取到用户态                                       │
│  失效: 文件修改、cache invalidate                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Level 1: FUSE 层 - Node Cache (nodeCache)                                   │
│  ─────────────────────────────────────────────────────────────────────────  │
│  位置: client/fs/super.go:65                                                 │
│  结构: map[uint64]fs.Node  (ino → Dir/File 对象)                             │
│  作用: 复用已创建的 Dir/File 对象，避免重复创建                                 │
│  失效: Forget 回调时删除                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Level 2: 元数据层 - InodeCache + Dcache                                     │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  InodeCache (ic)                           client/fs/icache.go:37      │ │
│  │  结构: map[uint64]*list.Element  (ino → InodeInfo)                     │ │
│  │  淘汰: LRU + 过期时间 (icacheTimeout, 默认 120s)                        │ │
│  │  作用: 缓存 InodeGet RPC 结果 (size, mode, mtime 等)                    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Dcache (dc)                               client/fs/dcachev2.go:37    │ │
│  │  结构: map[string]*list.Element  (parentIno+name → DentryInfo)         │ │
│  │  淘汰: LRU + 过期时间                                                   │ │
│  │  作用: 缓存 Lookup RPC 结果 (name → child ino)                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  DentryCache (per-Dir)                     client/fs/dcache.go:23      │ │
│  │  结构: map[string]uint64  (name → ino)                                 │ │
│  │  位置: 每个 Dir 对象内部                                                │ │
│  │  作用: ReadDir 结果缓存，减少重复 ReadDir RPC                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Level 3: 数据层 - ExtentCache + Streamer                                    │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  ExtentCache (per-Streamer)         sdk/data/stream/extent_cache.go:65 │ │
│  │  结构: BTree (按 FileOffset 排序的 ExtentKey 列表)                      │ │
│  │  位置: 每个 Streamer 内部                                               │ │
│  │  作用: 缓存文件的 extent 布局，知道数据在哪些 DataNode                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Streamers (per-inode)              sdk/data/stream/stream_reader.go   │ │
│  │  结构: map[uint64]*Streamer  (在 ExtentClient 中)                       │ │
│  │  作用: 管理每个文件的读写状态、脏数据列表                                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Level 4: 本地磁盘层 - Block Cache (可选)                                    │
│  ─────────────────────────────────────────────────────────────────────────  │
│  位置: client/blockcache/bcache/                                             │
│  配置: enableBcache=true, bcacheDir=/path/to/cache                           │
│  作用: 将热点数据缓存到本地 SSD，减少网络 I/O                                 │
│  适用: 冷热分层场景，加速冷卷读取                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 各缓存详解

### 2.1 InodeCache

```go
// client/fs/icache.go:37
type InodeCache struct {
    sync.RWMutex
    cache        map[uint64]*list.Element  // ino → InodeInfo
    lruList      *list.List                // LRU 淘汰链表
    expiration   time.Duration             // 过期时间 (默认 120s)
    maxElements  int                       // 最大条目数 (默认 10w)
}
```

**存储内容**：
```go
// proto/fs_proto.go
type InodeInfo struct {
    Inode      uint64  // inode 号
    Mode       uint32  // 文件类型和权限
    Nlink      uint32  // 硬链接数
    Size       uint64  // 文件大小
    Uid        uint32
    Gid        uint32
    ModifyTime time.Time
    AccessTime time.Time
    CreateTime time.Time
}
```

**核心方法**：

| 方法 | 作用 |
|------|------|
| `Put(info)` | 加入缓存，触发 LRU 淘汰 |
| `Get(ino)` | 获取缓存，检查过期 |
| `Delete(ino)` | 删除指定 inode |
| `backgroundEviction()` | 后台每 2 分钟清理过期项 |

**使用场景**：
```go
// client/fs/dir.go - Lookup 时
func (d *Dir) Lookup(ctx, req, resp) (fs.Node, error) {
    // 1. 先查 InodeCache
    if info := d.super.ic.Get(ino); info != nil {
        return d.super.newNode(info), nil  // 命中，不需要 RPC
    }
    
    // 2. 未命中，发起 InodeGet RPC
    info, err := d.super.mw.InodeGet_ll(ino)
    
    // 3. 结果放入缓存
    d.super.ic.Put(info)
}
```

---

### 2.2 Dcache (全局 Dentry 缓存)

```go
// client/fs/dcachev2.go:37
type Dcache struct {
    sync.RWMutex
    cache       map[string]*list.Element  // "parentIno/name" → DentryInfo
    lruList     *list.List
    expiration  time.Duration
    maxElements int
}
```

**与 InodeCache 的区别**：

| 维度 | InodeCache | Dcache |
|------|-----------|--------|
| Key | inode 号 | parentIno + name |
| Value | InodeInfo (属性) | DentryInfo (ino + 类型) |
| 减少 RPC | InodeGet | Lookup |
| 场景 | stat 文件属性 | ls 目录、路径解析 |

---

### 2.3 DentryCache (目录级缓存)

```go
// client/fs/dcache.go:23
type DentryCache struct {
    sync.Mutex
    cache        map[string]uint64  // name → ino
    expiration   time.Time          // 整个缓存过期时间
}
```

**特点**：
- 每个 Dir 对象内部一个
- 整体过期（不是单条过期）
- ReadDir 后填充，后续 Lookup 可直接使用

```go
// client/fs/dir.go - ReadDirAll 后
func (d *Dir) ReadDirAll(ctx) ([]fuse.Dirent, error) {
    children, err := d.super.mw.ReadDir_ll(d.info.Inode)
    
    // 填充目录级 DentryCache
    for _, child := range children {
        d.dcache.Put(child.Name, child.Inode)
    }
}

// 后续 Lookup 时
func (d *Dir) Lookup(ctx, req, resp) {
    // 先查目录级缓存
    if ino, ok := d.dcache.Get(req.Name); ok {
        // 命中！不需要 Lookup RPC
    }
}
```

---

### 2.4 ExtentCache

```go
// sdk/data/stream/extent_cache.go:65
type ExtentCache struct {
    sync.RWMutex
    inode   uint64
    gen     uint64           // 代数，用于检测变更
    size    uint64           // 文件大小
    root    *btree.BTree    // ExtentKey 按 FileOffset 排序
}
```

**存储内容**：
```go
// proto/extent_key.go
type ExtentKey struct {
    FileOffset   uint64  // 文件内偏移
    PartitionId  uint64  // 数据分区 ID
    ExtentId     uint64  // Extent ID
    ExtentOffset uint64  // Extent 内偏移
    Size         uint32  // 大小
}
```

**作用**：
```
文件内容: |----块1----|----块2----|----块3----|
          0         128KB      256KB      384KB

ExtentCache 存储:
  FileOffset=0     → DP=100, ExtentId=1, DataNode=[dn1,dn2,dn3]
  FileOffset=128KB → DP=100, ExtentId=2, DataNode=[dn1,dn2,dn3]
  FileOffset=256KB → DP=101, ExtentId=5, DataNode=[dn4,dn5,dn6]

读取 offset=200KB 时:
  1. 在 BTree 中找到 FileOffset ≤ 200KB 的最大项 → 块2
  2. 计算 ExtentOffset = 200KB - 128KB = 72KB
  3. 直接向 dn1/dn2/dn3 读取数据
```

---

### 2.5 nodeCache

```go
// client/fs/super.go:65
type Super struct {
    nodeCache map[uint64]fs.Node  // ino → Dir/File 对象
    fslock    sync.Mutex
}
```

**作用**：
- 复用已创建的 Dir/File Go 对象
- 避免对同一 inode 重复创建对象
- Forget 回调时清理

```go
// client/fs/super.go:395
func (s *Super) newNode(info *proto.InodeInfo) (fs.Node, error) {
    s.fslock.Lock()
    defer s.fslock.Unlock()
    
    // 先查 nodeCache
    if node, ok := s.nodeCache[ino]; ok {
        return node, nil
    }
    
    // 创建新对象并缓存
    if proto.IsDir(info.Mode) {
        node = NewDir(s, info)
    } else {
        node = NewFile(s, info)
    }
    s.nodeCache[ino] = node
    return node, nil
}
```

---

### 2.6 Block Cache (bcache)

```go
// client/fs/super.go:91
type Super struct {
    bc *bcache.BcacheClient  // 本地块缓存客户端
}
```

**配置**：
```json
{
  "enableBcache": true,
  "bcacheDir": "/mnt/ssd/cubefs_cache"
}
```

**适用场景**：
- 冷卷（BlobStore）数据加速
- 计算节点本地有 SSD
- 热点数据重复读取

---

## 3. 缓存协作流程

### 3.1 读请求命中顺序

```
cat /mnt/cubefs/foo.txt
    │
    ▼
① Page Cache 命中? ─Yes→ 直接返回 (内核处理)
    │No
    ▼
② nodeCache 有 File? ─No→ 创建 File，加入 nodeCache
    │Yes
    ▼
③ InodeCache 有 InodeInfo? ─No→ InodeGet RPC，加入缓存
    │Yes
    ▼
④ ExtentCache 有 extent 布局? ─No→ GetExtents RPC，填充缓存
    │Yes
    ▼
⑤ Block Cache 有数据块? ─Yes→ 从本地 SSD 读取
    │No
    ▼
⑥ 从 DataNode 读取数据
```

### 3.2 Lookup 请求

```
ls /mnt/cubefs/dir/
    │
    ▼
① 目录的 DentryCache 有效? ─Yes→ 直接返回缓存内容
    │No
    ▼
② 全局 Dcache 有 (parentIno, name)? ─Yes→ 返回 ino
    │No
    ▼
③ Lookup RPC 到 MetaNode
    │
    ▼
④ 结果填入 Dcache + DentryCache
```

### 3.3 写请求缓存更新

```
echo "hello" > /mnt/cubefs/foo.txt
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ① Write 请求进入                              client/fs/file.go:490       │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  InodeCache 立即失效:                                                        │
│    defer func() {                                                            │
│        f.super.ic.Delete(ino)  // size, mtime 会变，必须失效                 │
│    }()                                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ② 数据写入 Streamer                    sdk/data/stream/stream_writer.go   │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  1. 数据先写入 ExtentHandler (内存缓冲)                                       │
│  2. ExtentHandler 加入 dirtylist (脏数据列表)                                 │
│  3. ExtentCache 临时追加未提交的 ExtentKey                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ③ Flush 到 DataNode                   sdk/data/stream/stream_writer.go    │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  1. dirtylist 中的 ExtentHandler 逐个 flush                                  │
│  2. 数据写入 DataNode (3 副本)                                               │
│  3. AppendExtentKey RPC → MetaNode (持久化 extent 信息)                      │
│  4. ExtentCache 更新 (标记为已提交)                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ④ Block Cache 失效 (如果启用)           sdk/data/stream/stream_writer.go   │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  if s.client.bcacheEnable {                                                  │
│      s.client.evictBcache(cacheKey)  // 旧数据块失效                         │
│  }                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 目录操作缓存更新

```
mkdir /mnt/cubefs/newdir
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Create/Mkdir/Remove/Rename                        client/fs/dir.go        │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  1. 父目录 InodeCache 失效 (mtime 变化)                                      │
│  2. 父目录 DentryCache 失效 (子项变化)                                       │
│  3. Dcache 删除/更新相关条目                                                 │
│  4. nodeCache 删除被删除的节点                                               │
└─────────────────────────────────────────────────────────────────────────────┘

rm /mnt/cubefs/foo.txt
    │
    ▼
  d.super.ic.Delete(parentIno)     // 父目录 InodeCache 失效
  d.dcache.Delete(name)            // 目录级缓存删除
  d.super.dc.Delete(parentIno+name) // 全局 Dcache 删除
  d.super.fslock.Lock()
  delete(d.super.nodeCache, childIno)  // nodeCache 删除
```

---

## 4. 读写缓存行为对比

### 4.1 各缓存读写行为

| 缓存 | 读取时 | 写入时 | 目录操作时 |
|------|--------|--------|-----------|
| Page Cache | 查询/填充 | 内核自动更新 | - |
| nodeCache | 查询/创建 | 不变 | 删除被删节点 |
| InodeCache | 查询/填充 | **立即删除** | 父目录失效 |
| Dcache | 查询/填充 | 不变 | 删除/更新条目 |
| DentryCache | 查询/填充 | 不变 | 整体失效 |
| ExtentCache | 查询/填充 | **追加新项** | - |
| Block Cache | 查询/填充 | **失效旧块** | - |

### 4.2 关键代码位置

```go
// ========== 读取时填充缓存 ==========

// InodeCache 填充 - client/fs/dir.go Lookup
info, err := d.super.mw.InodeGet_ll(ino)
d.super.ic.Put(info)  // 加入缓存

// Dcache 填充 - client/fs/dir.go Lookup
d.super.dc.Put(&proto.DentryInfo{Name: name, Inode: ino, ...})

// ExtentCache 填充 - sdk/data/stream/extent_cache.go
func (cache *ExtentCache) RefreshForce(...) {
    gen, size, extents, err := getExtents(inode, ...)
    cache.update(gen, size, force, extents)
}

// ========== 写入时失效缓存 ==========

// InodeCache 删除 - client/fs/file.go:518
func (f *File) Write(...) error {
    defer func() {
        f.super.ic.Delete(ino)  // 写完必删
    }()
}

// Block Cache 失效 - sdk/data/stream/stream_writer.go:414
if s.client.bcacheEnable {
    go s.client.evictBcache(cacheKey)
}

// ========== 目录操作时失效缓存 ==========

// Remove 操作 - client/fs/dir.go
func (d *Dir) Remove(...) error {
    d.super.ic.Delete(d.info.Inode)  // 父目录失效
    d.dcache.Delete(req.Name)
    d.super.dc.Delete(d.info.Inode, req.Name)
}
```

### 4.3 为什么写入要删除 InodeCache？

```
写入前:
  InodeCache[ino=100] = {size: 1000, mtime: 10:00:00}

写入 500 字节后:
  实际: size=1500, mtime: 10:00:05
  缓存: size=1000, mtime: 10:00:00  ← 过期数据！

解决方案:
  写入完成后立即 ic.Delete(ino)
  下次读取时重新从 MetaNode 获取最新值
```

### 4.4 ExtentCache 的追加 vs 覆盖

```
追加写 (append):
  原有: [ek1: 0-100KB] [ek2: 100-200KB]
  追加 50KB 后:
  更新: [ek1: 0-100KB] [ek2: 100-200KB] [ek3: 200-250KB]  ← 新增

覆盖写 (overwrite):
  原有: [ek1: 0-100KB] [ek2: 100-200KB]
  覆盖 offset=50KB, size=100KB:
  更新: [ek1: 0-50KB] [ek_new: 50-150KB] [ek2_split: 150-200KB]  ← 分裂
```

---

## 5. 配置调优

### 5.1 相关配置项

| 配置项 | 含义 | 默认值 | 建议 |
|--------|------|--------|------|
| `icacheTimeout` | InodeCache 过期时间 | 120s | 读多写少可增大 |
| `lookupValid` | Lookup 结果有效期 | 5s | 目录稳定可增大 |
| `attrValid` | Attr 结果有效期 | 5s | 文件稳定可增大 |
| `keepCache` | 启用内核 Page Cache | false | 单客户端可开启 |
| `enableBcache` | 启用本地块缓存 | false | 有 SSD 可开启 |
| `disableDcache` | 禁用 Dentry 缓存 | false | 多客户端强一致可开启 |

### 5.2 场景建议

**单客户端独占访问**：
```json
{
  "keepCache": true,
  "icacheTimeout": 300,
  "lookupValid": 30,
  "attrValid": 30
}
```

**多客户端共享访问**：
```json
{
  "keepCache": false,
  "icacheTimeout": 30,
  "lookupValid": 1,
  "attrValid": 1
}
```

**AI 训练场景 (大量小文件读取)**：
```json
{
  "keepCache": true,
  "enableBcache": true,
  "bcacheDir": "/mnt/nvme/cache"
}
```

---

## 6. 缓存一致性

### 6.1 问题场景

```
Client A                          Client B
    │                                 │
    ▼                                 ▼
 读取 foo.txt                     修改 foo.txt
 缓存 size=100                    size → 200
    │                                 │
    ▼                                 ▼
 再次读取                         写入完成
 缓存命中 size=100 ← 错误！
```

### 6.2 解决方案

1. **短过期时间**：icacheTimeout, lookupValid, attrValid
2. **禁用缓存**：keepCache=false, disableDcache=true
3. **主动失效**：文件修改时 invalidate (需要内核支持)

### 6.3 各缓存失效机制

| 缓存 | 失效触发 |
|------|----------|
| Page Cache | close 后失效 (keepCache=false) |
| InodeCache | 过期时间 + 后台清理 |
| Dcache | 过期时间 + 后台清理 |
| DentryCache | 整体过期 |
| ExtentCache | 文件关闭时丢弃 |
| nodeCache | FUSE Forget 回调 |

---

## 7. 监控与调试

### 7.1 查看缓存状态

```bash
# pprof 接口
curl http://127.0.0.1:17420/debug/pprof/

# 查看统计信息
curl http://127.0.0.1:17420/stat
```

### 7.2 日志关键词

```bash
# InodeCache
grep "InodeCache" /var/log/cubefs/client.log

# Dcache
grep "Dcache\|DentryCache" /var/log/cubefs/client.log

# ExtentCache
grep "ExtentCache" /var/log/cubefs/client.log
```
