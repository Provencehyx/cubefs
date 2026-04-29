# Client FUSE 五层解析 · Layer 3：目录操作

> 入口文件：[client/fs/dir.go](../../client/fs/dir.go)
> 核心问题：**Dir 节点是怎么实现 FUSE 的目录操作接口，和 MetaNode 交互完成 Lookup、ReadDir、Create、Remove 等操作的？**

> 💡 Layer 2 留下的钩子：`NewDir` 创建的 Dir 对象实现了哪些 FUSE 接口？

---

## 1. Dir 结构体

[dir.go:102-112](../../client/fs/dir.go#L102-L112)

```go
type Dir struct {
    super     *Super              // 指向 Super，访问 mw/ec/ic
    info      *proto.InodeInfo    // 目录的 inode 信息
    dcache    *DentryCache        // 本目录的 dentry 缓存
    dctx      *DirContexts        // ReadDir 的分页上下文
    parentIno uint64              // 父目录 inode
    name      string              // 目录名
    missCount uint32              // Lookup 缓存未命中计数
    lastTime  int64               // 上次触发 ReadDirAll 的时间
}
```

### Dir 实现的 FUSE 接口

[dir.go:115-132](../../client/fs/dir.go#L115-L132)

```go
var (
    _ fs.Node                = (*Dir)(nil)
    _ fs.NodeCreater         = (*Dir)(nil)   // Create 文件
    _ fs.NodeForgetter       = (*Dir)(nil)   // Forget
    _ fs.NodeMkdirer         = (*Dir)(nil)   // Mkdir
    _ fs.NodeMknoder         = (*Dir)(nil)   // Mknod
    _ fs.NodeRemover         = (*Dir)(nil)   // Remove
    _ fs.NodeFsyncer         = (*Dir)(nil)   // Fsync
    _ fs.NodeRequestLookuper = (*Dir)(nil)   // Lookup
    _ fs.HandleReadDirAller  = (*Dir)(nil)   // ReadDirAll
    _ fs.NodeRenamer         = (*Dir)(nil)   // Rename
    _ fs.NodeSetattrer       = (*Dir)(nil)   // Setattr
    _ fs.NodeSymlinker       = (*Dir)(nil)   // Symlink
    _ fs.NodeGetxattrer      = (*Dir)(nil)   // Getxattr
    _ fs.NodeListxattrer     = (*Dir)(nil)   // Listxattr
    _ fs.NodeSetxattrer      = (*Dir)(nil)   // Setxattr
    _ fs.NodeRemovexattrer   = (*Dir)(nil)   // Removexattr
)
```

---

## 2. Lookup：路径解析的核心

[dir.go:336-493](../../client/fs/dir.go#L336-L493)

```go
func (d *Dir) Lookup(ctx context.Context, req *fuse.LookupRequest, resp *fuse.LookupResponse) (fs.Node, error) {
    var ino uint64
    
    // ① 先查 dentry 缓存
    if d.needDentrycache() {
        dcacheKey := d.buildDcacheKey(d.info.Inode, req.Name)
        dentryInfo := d.super.dc.Get(dcacheKey)
        if dentryInfo != nil {
            ino = dentryInfo.Inode  // ★ 缓存命中
        }
    }
    
    // ② 缓存未命中，走 MetaNode RPC
    if ino == 0 {
        cino, ok := d.dcache.Get(req.Name)
        if !ok {
            cino, _, err = d.super.mw.Lookup_ll(d.info.Inode, req.Name)  // ★ RPC
            if err != nil {
                return nil, ParseError(err)
            }
        }
        ino = cino
    }
    
    // ③ 获取 inode 信息
    info, err := d.super.InodeGet(ino)  // 可能走缓存
    if err != nil {
        // 返回 dummy 节点，避免 FUSE 重试
        dummyInodeInfo := &proto.InodeInfo{Inode: ino}
        dummyChild := NewFile(d.super, dummyInodeInfo, DefaultFlag, d.info.Inode, req.Name)
        return dummyChild, nil
    }
    
    // ④ 创建或更新 nodeCache
    mode := proto.OsMode(info.Mode)
    d.super.fslock.Lock()
    child, ok := d.super.nodeCache[ino]
    if !ok {
        if mode.IsDir() {
            child = NewDir(d.super, info, d.info.Inode, req.Name)
        } else {
            child = NewFile(d.super, info, DefaultFlag, d.info.Inode, req.Name)
        }
        d.super.nodeCache[ino] = child
    }
    d.super.fslock.Unlock()
    
    // ⑤ 触发预热（缓存未命中次数过多时）
    if missCache && d.super.metaCacheAcceleration {
        if atomic.AddUint32(&d.missCount, 1) > 5 {
            go d.ReadDirAll(context.Background())  // ★ 后台预热
        }
    }
    
    resp.EntryValid = LookupValidDuration  // 告诉内核缓存多久
    return child, nil
}
```

**5 个关键步骤**：

1. **dentry 缓存**：先查 `d.dcache` 或 `d.super.dc`
2. **Lookup RPC**：缓存未命中走 `mw.Lookup_ll(parentIno, name)`
3. **InodeGet**：拿到 ino 后获取 inode 详细信息
4. **构建节点**：Dir 或 File，存入 nodeCache
5. **预热触发**：Lookup 未命中过多时后台 ReadDirAll

> ⚠️ **关键点**：Lookup 返回的 `EntryValid` 告诉内核"这个结果可以缓存多久"，减少后续 Lookup 调用。

---

## 3. ReadDirAll：读取目录内容

[dir.go:634-721](../../client/fs/dir.go#L634-L721)

```go
func (d *Dir) ReadDirAll(ctx context.Context) ([]fuse.Dirent, error) {
    // ① 分批读取所有 dentry
    noMore := false
    from := ""
    var children []proto.Dentry
    for !noMore {
        batches, err := d.super.mw.ReadDirLimit_ll(d.info.Inode, from, DefaultReaddirLimit)
        if err != nil { return nil, ParseError(err) }
        
        batchNr := uint64(len(batches))
        if batchNr < DefaultReaddirLimit {
            noMore = true
        }
        if from != "" {
            batches = batches[1:]  // 跳过上次的最后一个
        }
        children = append(children, batches...)
        from = batches[len(batches)-1].Name  // 下次从这里开始
    }
    
    // ② 收集 inode 列表
    inodes := make([]uint64, 0, len(children))
    dirents := make([]fuse.Dirent, 0, len(children))
    dcache := NewDentryCache(d.super.metaCacheAcceleration)
    
    for _, child := range children {
        dentry := fuse.Dirent{
            Inode: child.Inode,
            Type:  ParseType(child.Type),
            Name:  child.Name,
        }
        inodes = append(inodes, child.Inode)
        dirents = append(dirents, dentry)
        dcache.Put(child.Name, child.Inode)  // ★ 填充 dentry 缓存
    }
    
    // ③ 批量获取 inode 信息
    var infos []*proto.InodeInfo
    if d.super.metaCacheAcceleration {
        infos = d.super.mw.BatchInodeGetExtents(inodes)  // 带 extent
    } else {
        infos = d.super.mw.BatchInodeGet(inodes)
    }
    
    // ④ 更新 inode 缓存
    for _, info := range infos {
        d.super.ic.Put(info)
    }
    
    d.dcache = dcache  // ★ 保存 dentry 缓存
    return dirents, nil
}
```

**4 个关键步骤**：

1. **分页读取**：`ReadDirLimit_ll` 每次最多读 1024 个
2. **构建 dirent**：转换成 FUSE 格式
3. **批量 InodeGet**：一次 RPC 获取所有 inode 信息
4. **填充缓存**：dentry 缓存 + inode 缓存

> 💡 **性能优化**：`BatchInodeGet` 一次获取多个 inode，比逐个 `InodeGet` 高效得多。

---

## 4. Create：创建文件

[dir.go:183-231](../../client/fs/dir.go#L183-L231)

```go
func (d *Dir) Create(ctx context.Context, req *fuse.CreateRequest, resp *fuse.CreateResponse) (fs.Node, fs.Handle, error) {
    fullPath := path.Join(d.getCwd(), req.Name)
    
    // ① 调用 MetaNode 创建
    info, err := d.super.mw.Create_ll(
        d.info.Inode,           // 父目录 ino
        req.Name,               // 文件名
        proto.Mode(req.Mode.Perm()),  // 权限
        req.Uid, req.Gid,       // 属主
        nil,                    // target（符号链接才用）
        fullPath,               // 完整路径（用于配额）
        false,                  // 是否 overwrite
    )
    if err != nil {
        return nil, nil, ParseError(err)
    }
    
    // ② 写入缓存
    d.super.ic.Put(info)
    
    // ③ 创建 File 节点
    child := NewFile(d.super, info, uint32(req.Flags&DefaultFlag), d.info.Inode, req.Name)
    
    // ④ 打开流
    openForWrite := req.Flags&0x0f != syscall.O_RDONLY
    isCache := proto.IsCold(d.super.volType) || proto.IsStorageClassBlobStore(info.StorageClass)
    d.super.ec.OpenStream(info.Inode, openForWrite, isCache, fullPath)
    
    // ⑤ 存入 nodeCache
    d.super.fslock.Lock()
    d.super.nodeCache[info.Inode] = child
    d.super.fslock.Unlock()
    
    // ⑥ 使父目录缓存失效
    d.super.ic.Delete(d.info.Inode)  // ★ 父目录的 nlink/mtime 变了
    
    resp.EntryValid = LookupValidDuration
    return child, child, nil  // 返回 Node 和 Handle
}
```

**关键点**：
- `Create_ll` 是 MetaWrapper 的方法，最终发 RPC 到 MetaNode
- 创建完成后立即 `OpenStream`，准备好数据通道
- 父目录 inode 缓存失效（nlink/mtime 变化）

---

## 5. Remove：删除文件/目录

[dir.go:289-329](../../client/fs/dir.go#L289-L329)

```go
func (d *Dir) Remove(ctx context.Context, req *fuse.RemoveRequest) error {
    // ① 清理 dentry 缓存
    d.dcache.Delete(req.Name)
    dcacheKey := d.buildDcacheKey(d.info.Inode, req.Name)
    d.super.dc.Delete(dcacheKey)
    
    fullPath := path.Join(d.getCwd(), req.Name)
    
    // ② 调用 MetaNode 删除
    info, err := d.super.mw.Delete_ll(
        d.info.Inode,  // 父目录 ino
        req.Name,      // 文件名
        req.Dir,       // 是否是目录
        fullPath,      // 完整路径（用于回收站）
    )
    if err != nil {
        return ParseError(err)
    }
    
    // ③ 使父目录缓存失效
    d.super.ic.Delete(d.info.Inode)
    
    // ④ 处理孤儿 inode
    if info != nil && info.Nlink == 0 && !proto.IsDir(info.Mode) {
        d.super.orphan.Put(info.Inode)  // ★ 加入孤儿列表
    }
    
    return nil
}
```

**孤儿 inode**：当 `Nlink == 0` 但文件还被打开时，inode 不能立即删除。加入 `orphan` 列表，等所有 fd 关闭后再真正删除。

---

## 6. Rename：重命名/移动

[dir.go:723-800](../../client/fs/dir.go#L723-L800)

```go
func (d *Dir) Rename(ctx context.Context, req *fuse.RenameRequest, newDir fs.Node) error {
    dstDir, ok := newDir.(*Dir)
    if !ok {
        return fuse.ENOTSUP  // 目标不是目录
    }
    
    // ① 清理 dentry 缓存
    d.dcache.Delete(req.OldName)
    
    // ② 配额检查
    if d.super.mw.EnableQuota {
        if !d.canRenameByQuota(dstDir, req.OldName) {
            return fuse.EPERM  // ★ 不能跨配额边界移动
        }
    }
    
    srcPath := path.Join(d.getCwd(), req.OldName)
    dstPath := path.Join(dstDir.getCwd(), req.NewName)
    
    // ③ 调用 MetaNode
    err = d.super.mw.Rename_ll(
        d.info.Inode, req.OldName,
        dstDir.info.Inode, req.NewName,
        srcPath, dstPath,
        true,  // 是否检查权限
    )
    if err != nil {
        return ParseError(err)
    }
    
    // ④ 更新 nodeCache 中节点的 parent 和 name
    d.super.fslock.Lock()
    node, ok := d.super.nodeCache[srcInode]
    if ok {
        if dir, ok := node.(*Dir); ok {
            dir.name = req.NewName
            dir.parentIno = dstDir.info.Inode
        } else {
            file := node.(*File)
            file.name = req.NewName
            file.parentIno = dstDir.info.Inode
        }
    }
    d.super.fslock.Unlock()
    
    // ⑤ 使缓存失效
    d.super.ic.Delete(d.info.Inode)
    d.super.ic.Delete(dstDir.info.Inode)
    
    return nil
}
```

**关键点**：
- 配额检查：不能把文件移出配额目录
- 更新 nodeCache：节点的 `parentIno` 和 `name` 要同步更新
- 两个目录的 inode 缓存都要失效

---

## 7. Mkdir / Symlink / Link

| 操作 | MetaWrapper 方法 | 特殊处理 |
|------|------------------|----------|
| Mkdir | `mw.Create_ll(..., os.ModeDir\|perm, ...)` | mode 带 `ModeDir` |
| Symlink | `mw.Create_ll(..., os.ModeSymlink, target)` | target 是链接目标 |
| Link | `mw.Link(parentIno, name, oldIno, fullPath)` | 硬链接，只能链接文件 |

---

## 8. DentryCache：目录项缓存

[dcache.go](../../client/fs/dcache.go)

```go
type DentryCache struct {
    sync.RWMutex
    cache map[string]uint64  // name → ino
}

func (dc *DentryCache) Put(name string, ino uint64) {
    dc.Lock()
    dc.cache[name] = ino
    dc.Unlock()
}

func (dc *DentryCache) Get(name string) (uint64, bool) {
    dc.RLock()
    ino, ok := dc.cache[name]
    dc.RUnlock()
    return ino, ok
}
```

**和 InodeCache 的区别**：
- InodeCache：`ino → InodeInfo`（inode 的详细信息）
- DentryCache：`(parentIno, name) → childIno`（目录项映射）

---

## 9. 本层关键问题自测

1. ✅ Lookup 的 5 个步骤分别做什么？
2. ✅ 为什么 Lookup 缓存未命中次数过多时要触发 ReadDirAll？
3. ✅ ReadDirAll 为什么用 `BatchInodeGet` 而不是逐个 `InodeGet`？
4. ✅ Create 之后为什么要 `d.super.ic.Delete(d.info.Inode)`？
5. ✅ Remove 时 `Nlink == 0` 的文件为什么不能立即删除？
6. ✅ Rename 时为什么要检查配额？
7. ✅ DentryCache 和 InodeCache 各存什么？

---

## 10. 给后续 Layer 的钩子

- **Layer 4 (File)**：`NewFile` 创建的 File 对象实现了哪些接口？Open/Read/Write/Flush/Release 的流程是什么？冷卷和热卷的 Read/Write 有什么不同？
- **存疑点**：
  - `orphan` 列表什么时候真正清理？谁负责调用 `mw.Evict`？
  - `dctx` (DirContexts) 在分页 ReadDir 中是怎么工作的？
