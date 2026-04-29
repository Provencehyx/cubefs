# Client FUSE 五层解析 · Layer 2：Super 与缓存

> 入口文件：[client/fs/super.go](../../client/fs/super.go) · [client/fs/icache.go](../../client/fs/icache.go) · [client/fs/dcache.go](../../client/fs/dcache.go)
> 核心问题：**Super 是怎么把 MetaWrapper、ExtentClient、各种缓存组装在一起，成为 FUSE 文件系统的核心的？**

> 💡 Layer 1 留下的钩子：`cfs.NewSuper(opt)` 做了什么？MetaWrapper 和 ExtentClient 是怎么初始化的？

---

## 1. Super 结构体：字段分组速记

[super.go:50-104](../../client/fs/super.go#L50-L104)

```go
type Super struct {
    // —— 集群标识 ——
    cluster     string
    volname     string
    masters     string
    mountPoint  string
    subDir      string
    owner       string

    // —— 核心组件（最重要！） ——
    mw          *meta.MetaWrapper      // 元数据客户端 → MetaNode
    ec          *stream.ExtentClient   // 数据客户端 → DataNode

    // —— 缓存层 ——
    ic          *InodeCache            // inode 信息缓存
    dc          *Dcache                // dentry 缓存（新版 v2）
    nodeCache   map[uint64]fs.Node     // ino → Dir/File 节点映射
    fslock      sync.Mutex

    // —— 孤儿 inode ——
    orphan      *OrphanInodeList       // 待清理的已删除 inode

    // —— 配置开关 ——
    enSyncWrite     bool               // 同步写
    keepCache       bool               // FUSE KeepCache
    disableDcache   bool               // 禁用 dentry 缓存
    fsyncOnClose    bool               // close 时 fsync
    enableXattr     bool               // xattr 支持
    rootIno         uint64             // 根 inode（可能是子目录）

    // —— 冷卷/BlobStore 相关 ——
    volType         int                // 热卷/冷卷
    ebsEndpoint     string             // BlobStore 地址
    EbsBlockSize    int                // EC 块大小
    ebsc            *blobstore.BlobStoreClient
    bc              *bcache.BcacheClient  // 块缓存客户端

    // —— 状态控制 ——
    state       fs.FSStatType          // Resume/Suspend/Restore/Shutdown
    sockaddr    string
    suspendCh   chan interface{}
}
```

**4 大组件**：

| 组件 | 类型 | 作用 | 连接目标 |
|------|------|------|----------|
| `mw` | `*meta.MetaWrapper` | 元数据操作（Create/Lookup/Delete/Setattr...） | MetaNode |
| `ec` | `*stream.ExtentClient` | 数据读写（Read/Write/Flush） | DataNode |
| `ebsc` | `*blobstore.BlobStoreClient` | 冷卷数据读写 | BlobStore |
| `bc` | `*bcache.BcacheClient` | 本地块缓存 | 本地磁盘 |

**3 层缓存**：

| 缓存 | 类型 | 作用 |
|------|------|------|
| `ic` | `*InodeCache` | 缓存 `proto.InodeInfo`，减少 `InodeGet` RPC |
| `dc` | `*Dcache` | 缓存 `(parentIno, name) → childIno`，减少 `Lookup` RPC |
| `nodeCache` | `map[uint64]fs.Node` | 缓存已创建的 Dir/File 对象 |

---

## 2. NewSuper 初始化流程

[super.go:120-350](../../client/fs/super.go#L120-L350)

```go
func NewSuper(opt *proto.MountOptions) (s *Super, err error) {
    s = new(Super)
    
    // ① 创建 MetaWrapper
    metaConfig := &meta.MetaConfig{
        Volume:          opt.Volname,
        Owner:           opt.Owner,
        Masters:         strings.Split(opt.Master, meta.HostsSeparator),
        Authenticate:    opt.Authenticate,
        MetaSendTimeout: opt.MetaSendTimeout,
        SubDir:          opt.SubDir,
    }
    s.mw, err = meta.NewMetaWrapper(metaConfig)
    
    // ② 初始化缓存
    inodeExpiration := DefaultInodeExpiration
    if opt.IcacheTimeout >= 0 {
        inodeExpiration = time.Duration(opt.IcacheTimeout) * time.Second
    }
    s.ic = NewInodeCache(inodeExpiration, int(opt.InodeLruLimit), s.metaCacheAcceleration)
    s.dc = NewDcache(inodeExpiration, MaxInodeCache)
    s.orphan = NewOrphanInodeList()
    s.nodeCache = make(map[uint64]fs.Node)

    // ③ 创建 ExtentClient
    extentConfig := &stream.ExtentConfig{
        Volume:            opt.Volname,
        Masters:           masters,
        FollowerRead:      opt.FollowerRead,     // ★ 从 follower 读
        NearRead:          opt.NearRead,         // ★ 就近读
        MaximallyRead:     opt.MaximallyRead,
        ReadRate:          opt.ReadRate,
        WriteRate:         opt.WriteRate,
        MaxStreamerLimit:  opt.MaxStreamerLimit,
        MetaWrapper:       s.mw,
        OnAppendExtentKey: s.mw.AppendExtentKey, // ★ 回调：写完通知 meta
        OnTruncate:        s.mw.Truncate,
        OnEvictIcache:     s.ic.Delete,          // ★ 回调：驱逐 inode 缓存
        OnLoadBcache:      s.bc.Get,             // ★ 回调：读块缓存
        OnCacheBcache:     s.bc.Put,
        OnEvictBcache:     s.bc.Evict,
    }
    s.ec, err = stream.NewExtentClient(extentConfig)

    // ④ 冷卷：创建 BlobStore 客户端
    if proto.IsVolSupportStorageClass(extentConfig.VolAllowedStorageClass, proto.StorageClass_BlobStore) {
        s.ebsc, err = blobstore.NewEbsClient(access.Config{...})
    }

    // ⑤ 获取根 inode
    s.rootIno, err = s.mw.GetRootIno(opt.SubDir)

    // ⑥ 启动后台任务
    if proto.IsCold(opt.VolType) || ... {
        go s.scheduleFlush()       // 定期 flush 冷卷 writer
    }
    go s.loopSyncMeta()            // 定期同步 inode 缓存
    go s.loopWarmUpMetaPaths()     // 元数据预热
    
    return s, nil
}
```

**6 个关键步骤**：

1. **MetaWrapper**：封装了所有元数据 RPC（sdk/meta 包）
2. **缓存初始化**：InodeCache + Dcache + nodeCache
3. **ExtentClient**：封装了数据读写（sdk/data/stream 包），注意回调函数
4. **BlobStore 客户端**：冷卷/EC 存储需要
5. **根 inode**：如果挂载子目录，`rootIno` 不是 1
6. **后台任务**：缓存同步、冷卷 flush、元数据预热

---

## 3. InodeCache：LRU + 过期驱逐

[icache.go:37-56](../../client/fs/icache.go#L37-L56)

```go
type InodeCache struct {
    sync.RWMutex
    cache        map[uint64]*list.Element  // ino → LRU 节点
    lruList      *list.List                // LRU 链表
    expiration   time.Duration             // 过期时间
    maxElements  int                       // 容量上限
    initExp      time.Duration             // 初始过期时间
    acceleration bool                      // 元数据加速模式
}
```

### 3.1 Put 操作

[icache.go:64-82](../../client/fs/icache.go#L64-L82)

```go
func (ic *InodeCache) Put(info *proto.InodeInfo) {
    ic.Lock()
    old, ok := ic.cache[info.Inode]
    if ok {
        ic.lruList.Remove(old)     // 已存在则先删除
        delete(ic.cache, info.Inode)
    }

    if ic.lruList.Len() >= ic.maxElements {
        ic.evict(true)             // ★ 容量满了，前台驱逐
    }

    inodeSetExpiration(info, ic.expiration)  // 设置过期时间
    element := ic.lruList.PushFront(info)    // 插入 LRU 头部
    ic.cache[info.Inode] = element
    ic.Unlock()
}
```

### 3.2 Get 操作

[icache.go:85-110](../../client/fs/icache.go#L85-L110)

```go
func (ic *InodeCache) Get(ino uint64) *proto.InodeInfo {
    ic.RLock()
    element, ok := ic.cache[ino]
    if !ok {
        ic.RUnlock()
        return nil
    }

    info := element.Value.(*proto.InodeInfo)
    if inodeExpired(info) && DisableMetaCache && !ic.acceleration {
        ic.RUnlock()
        return nil  // ★ 过期了，返回 nil 强制走 RPC
    }
    ic.RUnlock()

    if ic.acceleration && info != nil {
        // 加速模式：刷新过期时间（类似 LRU 访问更新）
        info.SetExpiration(time.Now().Add(ic.expiration).UnixNano())
    }
    return info
}
```

### 3.3 驱逐策略

[icache.go:129-176](../../client/fs/icache.go#L129-L176)

```go
func (ic *InodeCache) evict(foreground bool) {
    // 前台驱逐：至少驱逐 MinInodeCacheEvictNum=10 个
    for i := 0; i < MinInodeCacheEvictNum; i++ {
        element := ic.lruList.Back()  // 从 LRU 尾部取
        if element == nil { return }
        
        info := element.Value.(*proto.InodeInfo)
        // 后台驱逐：只驱逐过期的；前台驱逐：不管过期直接驱逐
        if !foreground && !inodeExpired(info) {
            return
        }
        ic.lruList.Remove(element)
        delete(ic.cache, info.Inode)
    }
    
    // 后台驱逐：继续驱逐所有过期的，最多 MaxInodeCacheEvictNum=200000 个
    if foreground { return }
    for i := 0; i < MaxInodeCacheEvictNum; i++ {
        element := ic.lruList.Back()
        if element == nil || !inodeExpired(info) { break }
        // ...
    }
}
```

**两种驱逐**：

| 类型 | 触发时机 | 驱逐条件 | 驱逐数量 |
|------|----------|----------|----------|
| 前台 | Put 时容量满 | 从 LRU 尾部直接驱逐 | 10 个 |
| 后台 | 每 2 分钟定时 | 只驱逐已过期的 | 最多 20 万 |

---

## 4. InodeGet：缓存+RPC 的入口

[inode.go:33-120](../../client/fs/inode.go#L33-L120)

```go
func (s *Super) InodeGet(ino uint64) (info *proto.InodeInfo, err error) {
    // ① 先查缓存
    info = s.ic.Get(ino)
    if info != nil {
        return info, nil
    }

    // ② 缓存未命中，走 RPC
    if s.metaCacheAcceleration {
        info, err = s.mw.InodeGetExt_ll(ino)  // 带 extent 的版本
    } else {
        info, err = s.mw.InodeGet_ll(ino)
    }
    if err != nil {
        return nil, ParseError(err)
    }

    // ③ 写入缓存
    s.ic.Put(info)

    // ④ 更新 nodeCache 中的节点
    s.fslock.Lock()
    node, isFind := s.nodeCache[ino]
    s.fslock.Unlock()
    if isFind {
        if dir, ok := node.(*Dir); ok {
            dir.info = info
        } else {
            file := node.(*File)
            // 检查存储类是否变化（迁移场景）
            if info.StorageClass != file.info.StorageClass {
                if proto.IsStorageClassBlobStore(info.StorageClass) {
                    // ★ 迁移到冷卷：创建 BlobStore reader/writer
                    file.fReader = blobstore.NewReader(clientConf)
                    file.fWriter = blobstore.NewWriter(clientConf)
                }
            }
            file.info = info
        }
    }

    // ⑤ 热卷：刷新 extent 缓存
    if !proto.IsStorageClassBlobStore(info.StorageClass) && !info.HasExtents() {
        s.ec.RefreshExtentsCache(ino)
    }
    return info, nil
}
```

**5 个关键点**：

1. **缓存优先**：先查 InodeCache
2. **RPC 获取**：缓存未命中才走 MetaWrapper
3. **写回缓存**：RPC 结果存入 InodeCache
4. **更新节点**：同步更新 nodeCache 中的 Dir/File 对象
5. **迁移处理**：存储类变化时（热→冷）重建 reader/writer

---

## 5. Root 方法：FUSE 入口

[super.go:379-386](../../client/fs/super.go#L379-L386)

```go
func (s *Super) Root() (fs.Node, error) {
    inode, err := s.InodeGet(s.rootIno)
    if err != nil {
        return nil, err
    }
    root := NewDir(s, inode, inode.Inode, "")
    return root, nil
}
```

**关键**：`s.rootIno` 可能不是 1（挂载子目录时）。FUSE 内核第一个调用的就是 Root()。

---

## 6. Statfs：文件系统统计

[super.go:412-424](../../client/fs/super.go#L412-L424)

```go
func (s *Super) Statfs(ctx context.Context, req *fuse.StatfsRequest, resp *fuse.StatfsResponse) error {
    total, used, inodeCount := s.mw.Statfs()  // ★ 调 MetaWrapper
    resp.Blocks = total / uint64(DefaultBlksize)
    resp.Bfree = (total - used) / uint64(DefaultBlksize)
    resp.Bavail = resp.Bfree
    resp.Bsize = DefaultBlksize   // 65536
    resp.Namelen = DefaultMaxNameLen  // 256
    resp.Files = inodeCount
    resp.Ffree = defaultMaxMetaPartitionInodeID - inodeCount
    return nil
}
```

这就是 `df` 命令看到的数据来源。

---

## 7. 后台任务：loopSyncMeta

[super.go:700-725](../../client/fs/super.go#L700-L725)

```go
func (s *Super) loopSyncMeta() {
    ticker := time.NewTicker(time.Second * 10)
    for {
        select {
        case <-ticker.C:
            if !s.stopWarmMeta {
                s.ic.ChangeExpiration(RemoteMetaCacheDuration)  // 48 小时
            } else {
                s.ic.RecoverExpiration()
            }
            if s.bcacheDir != "" || !s.stopWarmMeta {
                go s.syncMeta()  // ★ 同步 inode 缓存
            }
        case <-s.closeC:
            return
        }
    }
}

func (s *Super) syncMeta() {
    // 遍历 InodeCache，批量查询 MetaNode
    // 比较 ModifyTime / Generation，发现变化就驱逐缓存
    infos := s.mw.BatchInodeGet(inodes)
    for _, newInfo := range infos {
        oldInfo := s.ic.Get(newInfo.Inode)
        if !oldInfo.ModifyTime.Equal(newInfo.ModifyTime) || newInfo.Generation != oldGen {
            s.ic.Delete(newInfo.Inode)
            s.ec.ForceRefreshExtentsCache(newInfo.Inode)
            // 驱逐块缓存
            for _, extent := range extents {
                s.bc.Evict(cacheKey)
            }
        }
    }
}
```

**作用**：定期检查 InodeCache 中的 inode 是否还有效，发现变化就驱逐。这是保证缓存一致性的机制。

---

## 8. 本层关键问题自测

1. ✅ `mw` (MetaWrapper) 和 `ec` (ExtentClient) 各负责什么？
2. ✅ InodeCache 的两种驱逐策略有什么区别？
3. ✅ `InodeGet` 为什么要检查存储类是否变化？
4. ✅ 挂载子目录时 `rootIno` 是什么？
5. ✅ `loopSyncMeta` 为什么需要定期运行？
6. ✅ nodeCache 的 key 是什么？value 是什么类型？
7. ✅ ExtentClient 的 `OnAppendExtentKey` 回调是干什么的？

---

## 9. 给后续 Layer 的钩子

- **Layer 3 (Dir)**：`NewDir` 创建的 Dir 对象实现了哪些 FUSE 接口？Lookup、ReadDir、Create 内部怎么调用 `s.mw` 的方法？
- **Layer 4 (File)**：Read/Write 怎么区分热卷和冷卷？`fReader` / `fWriter` 什么时候创建？
- **存疑点**：
  - `orphan` 列表什么时候填充？什么时候清理？
  - `scheduleFlush` 为什么只对冷卷启用？
