# Client FUSE 五层解析 · Layer 4：文件操作

> 入口文件：[client/fs/file.go](../../client/fs/file.go)
> 核心问题：**File 节点是怎么实现 FUSE 的文件操作接口，完成 Open/Read/Write/Flush/Release 等操作的？热卷和冷卷的处理有什么不同？**

> 💡 Layer 3 留下的钩子：`NewFile` 创建的 File 对象实现了哪些接口？

---

## 1. File 结构体

[file.go:38-48](../../client/fs/file.go#L38-L48)

```go
type File struct {
    super     *Super              // 指向 Super，访问 mw/ec
    info      *proto.InodeInfo    // 文件的 inode 信息
    idle      int32               // 空闲计数（冷卷 flush 用）
    parentIno uint64              // 父目录 inode
    name      string              // 文件名
    sync.RWMutex

    fReader   *blobstore.Reader   // ★ 冷卷读取器
    fWriter   *blobstore.Writer   // ★ 冷卷写入器
    flag      uint32              // 打开标志 (O_RDONLY/O_WRONLY/O_RDWR)
}
```

### File 实现的 FUSE 接口

[file.go:51-67](../../client/fs/file.go#L51-L67)

```go
var (
    _ fs.Node              = (*File)(nil)
    _ fs.Handle            = (*File)(nil)
    _ fs.NodeForgetter     = (*File)(nil)   // Forget
    _ fs.NodeOpener        = (*File)(nil)   // Open
    _ fs.HandleReleaser    = (*File)(nil)   // Release
    _ fs.HandleReader      = (*File)(nil)   // Read
    _ fs.HandleWriter      = (*File)(nil)   // Write
    _ fs.HandleFlusher     = (*File)(nil)   // Flush
    _ fs.NodeFsyncer       = (*File)(nil)   // Fsync
    _ fs.NodeSetattrer     = (*File)(nil)   // Setattr
    _ fs.NodeReadlinker    = (*File)(nil)   // Readlink
    _ fs.NodeGetxattrer    = (*File)(nil)   // Getxattr
    _ fs.NodeListxattrer   = (*File)(nil)   // Listxattr
    _ fs.NodeSetxattrer    = (*File)(nil)   // Setxattr
    _ fs.NodeRemovexattrer = (*File)(nil)   // Removexattr
)
```

---

## 2. NewFile：热卷 vs 冷卷

[file.go:98-146](../../client/fs/file.go#L98-L146)

```go
func NewFile(s *Super, i *proto.InodeInfo, flag uint32, pino uint64, filename string) fs.Node {
    // ★ 冷卷 或 BlobStore 存储类：需要创建 fReader/fWriter
    if proto.IsCold(s.volType) || proto.IsStorageClassBlobStore(i.StorageClass) {
        clientConf := blobstore.ClientConfig{
            VolName:         s.volname,
            VolType:         s.volType,
            Ino:             i.Inode,
            BlockSize:       s.EbsBlockSize,
            Bc:              s.bc,       // 块缓存
            Mw:              s.mw,       // MetaWrapper
            Ec:              s.ec,       // ExtentClient
            Ebsc:            s.ebsc,     // BlobStore 客户端
            EnableBcache:    s.enableBcache,
            WConcurrency:    s.writeThreads,
            ReadConcurrency: s.readThreads,
            FileSize:        i.Size,
            StorageClass:    i.StorageClass,
        }
        
        var fReader *blobstore.Reader
        var fWriter *blobstore.Writer
        switch flag {
        case syscall.O_RDONLY:
            fReader = blobstore.NewReader(clientConf)
        case syscall.O_WRONLY:
            fWriter = blobstore.NewWriter(clientConf)
        case syscall.O_RDWR:
            fReader = blobstore.NewReader(clientConf)
            fWriter = blobstore.NewWriter(clientConf)
        }
        
        return &File{super: s, info: i, fWriter: fWriter, fReader: fReader, ...}
    }
    
    // ★ 热卷：不需要 fReader/fWriter，直接用 ExtentClient
    return &File{super: s, info: i, parentIno: pino, name: filename, flag: flag}
}
```

**两条路径**：

| 存储类型 | 读取 | 写入 |
|----------|------|------|
| 热卷 (Replica) | `ec.Read()` | `ec.Write()` |
| 冷卷 (BlobStore) | `fReader.Read()` | `fWriter.Write()` |

---

## 3. Open：打开文件

[file.go:247-347](../../client/fs/file.go#L247-L347)

```go
func (f *File) Open(ctx context.Context, req *fuse.OpenRequest, resp *fuse.OpenResponse) (handle fs.Handle, err error) {
    ino := f.info.Inode
    
    // ① 决定是否启用块缓存
    var needBCache bool
    if f.super.bcacheDir != "" && !f.filterFilesSuffix(f.super.bcacheFilterFiles) {
        parentPath := f.getParentPath()
        if strings.HasPrefix(parentPath, f.super.bcacheDir) {
            needBCache = true
        }
    }
    
    // ② 打开流
    openForWrite := req.Flags&0x0f != syscall.O_RDONLY
    isCache := proto.IsCold(f.super.volType) || proto.IsStorageClassBlobStore(f.info.StorageClass)
    
    if needBCache {
        f.super.ec.OpenStreamWithCache(ino, needBCache, openForWrite, isCache, fullPath)
    } else {
        f.super.ec.OpenStream(ino, openForWrite, isCache, fullPath)
    }
    
    // ③ 刷新 extent 缓存
    if f.super.metaCacheAcceleration {
        inodeInfo, _ := f.super.InodeGet(ino)
        if inodeInfo.Extents != nil {
            f.super.ec.RefreshExtentsWithCache(inodeInfo)
        } else {
            f.super.ec.RefreshExtentsCache(ino)
        }
    } else {
        f.super.ec.RefreshExtentsCache(ino)
    }
    
    // ④ KeepCache 选项
    if f.super.keepCache && resp != nil {
        resp.Flags |= fuse.OpenKeepCache  // 告诉内核保留页缓存
    }
    
    // ⑤ 冷卷：重新创建 reader/writer
    if proto.IsCold(f.super.volType) || proto.IsStorageClassBlobStore(f.info.StorageClass) {
        fileSize, _ := f.fileSizeVersion2(ino)
        clientConf := blobstore.ClientConfig{...}
        
        f.fWriter.FreeCache()  // 释放旧缓存
        
        switch req.Flags & 0x0f {
        case syscall.O_RDONLY:
            f.fReader = blobstore.NewReader(clientConf)
            f.fWriter = nil
        case syscall.O_WRONLY:
            f.fWriter = blobstore.NewWriter(clientConf)
            f.fReader = nil
        case syscall.O_RDWR:
            f.fReader = blobstore.NewReader(clientConf)
            f.fWriter = blobstore.NewWriter(clientConf)
        }
    }
    
    f.flag = uint32(req.Flags)
    return f, nil  // 返回自己作为 Handle
}
```

**5 个关键步骤**：

1. **块缓存决策**：根据路径和过滤规则
2. **打开流**：`ec.OpenStream` 增加引用计数
3. **刷新 extent**：确保 extent 列表最新
4. **KeepCache**：告诉内核不要清空页缓存
5. **冷卷特殊处理**：每次 Open 重建 reader/writer

---

## 4. Read：读取数据

[file.go:422-487](../../client/fs/file.go#L422-L487)

```go
func (f *File) Read(ctx context.Context, req *fuse.ReadRequest, resp *fuse.ReadResponse) (err error) {
    var size int
    
    // ★ 根据存储类选择读取路径
    if f.shouldAccessReplicaStorageClass() {
        // 热卷：走 ExtentClient
        f.super.ec.GetStreamer(f.info.Inode).SetParentInode(f.parentIno)
        size, err = f.super.ec.Read(
            f.info.Inode,
            resp.Data[fuse.OutHeaderSize:],
            int(req.Offset),
            req.Size,
            f.info.StorageClass,
            false,  // 不是预读
        )
    } else {
        // 冷卷：走 BlobStore Reader
        size, err = f.fReader.Read(ctx, resp.Data[fuse.OutHeaderSize:], int(req.Offset), req.Size)
    }
    
    if err != nil && err != io.EOF {
        f.super.handleError("Read", msg)
        errMetric := exporter.NewCounter("fileReadFailed")
        if !isReadEio(err) {
            errMetric.AddWithLabels(1, map[string]string{exporter.Err: "NOTSUP"})
        } else {
            errMetric.AddWithLabels(1, map[string]string{exporter.Err: "EIO"})
        }
        return ParseError(err)
    }
    
    // 截断响应数据
    if size > 0 {
        resp.Data = resp.Data[:size+fuse.OutHeaderSize]
    } else {
        resp.Data = resp.Data[:fuse.OutHeaderSize]
    }
    
    return nil
}
```

### shouldAccessReplicaStorageClass

[file.go:406-419](../../client/fs/file.go#L406-L419)

```go
func (f *File) shouldAccessReplicaStorageClass() bool {
    if proto.IsValidStorageClass(f.info.StorageClass) {
        if proto.IsStorageClassReplica(f.info.StorageClass) {
            return true  // 明确是副本存储类
        }
    } else {
        // 兼容老版本：没有 StorageClass 字段
        if proto.IsHot(f.super.volType) {
            return true
        }
    }
    return false
}
```

**两条读路径**：

```
热卷 (Replica):  ec.Read() → DataNode → Extent
冷卷 (BlobStore): fReader.Read() → BlobStore → Object
```

---

## 5. Write：写入数据

[file.go:490-612](../../client/fs/file.go#L490-L612)

```go
func (f *File) Write(ctx context.Context, req *fuse.WriteRequest, resp *fuse.WriteResponse) (err error) {
    ino := f.info.Inode
    reqlen := len(req.Data)
    
    // ① 特殊处理：posix_fallocate 兼容
    if proto.IsHot(f.super.volType) || proto.IsStorageClassReplica(f.info.StorageClass) {
        filesize, _ := f.fileSize(ino)
        if req.Offset > int64(filesize) && reqlen == 1 && req.Data[0] == 0 {
            // NFS 的 fallocate 兼容：写 1 字节 0 到文件末尾之后
            err = f.super.ec.Truncate(f.super.mw, f.parentIno, ino, int(req.Offset)+reqlen, fullPath)
            if err == nil {
                resp.Size = reqlen
            }
            return
        }
    }
    
    defer func() {
        f.super.ic.Delete(ino)  // ★ 写后使 inode 缓存失效
    }()
    
    // ② 决定写标志
    var waitForFlush bool
    var flags int
    
    if isDirectIOEnabled(req.FileFlags) || (req.FileFlags&fuse.OpenSync != 0) {
        waitForFlush = true
        if f.super.enSyncWrite {
            flags |= proto.FlagsSyncWrite  // 同步写
        }
        if proto.IsCold(f.super.volType) || proto.IsStorageClassBlobStore(f.info.StorageClass) {
            waitForFlush = false
            flags |= proto.FlagsSyncWrite
        }
    }
    
    if req.FileFlags&fuse.OpenAppend != 0 || proto.IsCold(f.super.volType) {
        flags |= proto.FlagsAppend  // 追加模式
    }
    
    // ③ 配额检查
    checkFunc := func() error {
        if !f.super.mw.EnableQuota { return nil }
        if f.super.ec.UidIsLimited(req.Uid) { return syscall.ENOSPC }
        // 检查配额限制
        var quotaIds []uint32
        for quotaId := range f.info.QuotaInfos {
            quotaIds = append(quotaIds, quotaId)
        }
        if f.super.mw.IsQuotaLimited(quotaIds) {
            return syscall.ENOSPC
        }
        return nil
    }
    
    // ④ 执行写入
    var size int
    if f.shouldAccessReplicaStorageClass() {
        // 热卷
        f.super.ec.GetStreamer(ino).SetParentInode(f.parentIno)
        size, err = f.super.ec.Write(ino, int(req.Offset), req.Data, flags, checkFunc, 
            f.info.StorageClass, false, waitForFlush)
    } else {
        // 冷卷
        atomic.StoreInt32(&f.idle, 0)  // 重置空闲计数
        size, err = f.fWriter.Write(context.Background(), int(req.Offset), req.Data, flags)
    }
    
    if err != nil {
        f.super.handleError("Write", msg)
        if err == syscall.EOPNOTSUPP {
            return fuse.ENOTSUP
        }
        return fuse.EIO
    }
    
    resp.Size = size
    
    // ⑤ 同步写：等待 flush
    if waitForFlush {
        err = f.super.ec.Flush(ino)
        if err != nil {
            return ParseError(err)
        }
    }
    
    return nil
}
```

**5 个关键步骤**：

1. **fallocate 兼容**：NFS 的特殊处理
2. **写标志决策**：DirectIO/Sync/Append
3. **配额检查**：写前检查 UID 和卷配额
4. **执行写入**：热卷 `ec.Write()` / 冷卷 `fWriter.Write()`
5. **同步写等待**：DirectIO 或 O_SYNC 时等 flush 完成

---

## 6. Flush / Fsync：刷写数据

### Flush

[file.go:614-672](../../client/fs/file.go#L614-L672)

```go
func (f *File) Flush(ctx context.Context, req *fuse.FlushRequest) (err error) {
    // 只在 fsyncOnClose 启用时才真正 flush
    if !f.super.fsyncOnClose {
        return fuse.ENOSYS  // 告诉内核不支持
    }
    
    if proto.IsHot(f.super.volType) || proto.IsStorageClassReplica(f.info.StorageClass) {
        err = f.super.ec.Flush(f.info.Inode)
    } else {
        f.Lock()
        err = f.fWriter.Flush(f.info.Inode, context.Background())
        f.Unlock()
    }
    
    if err != nil {
        return ParseError(err)
    }
    
    // 写后使 inode 缓存失效
    if DisableMetaCache && openForWrite {
        f.super.ic.Delete(f.info.Inode)
    }
    
    return nil
}
```

### Fsync

[file.go:675-707](../../client/fs/file.go#L675-L707)

```go
func (f *File) Fsync(ctx context.Context, req *fuse.FsyncRequest) (err error) {
    if proto.IsHot(f.super.volType) || proto.IsStorageClassReplica(f.info.StorageClass) {
        err = f.super.ec.Flush(f.info.Inode)
    } else {
        err = f.fWriter.Flush(f.info.Inode, context.Background())
    }
    
    if err != nil {
        return ParseError(err)
    }
    
    f.super.ic.Delete(f.info.Inode)  // 使缓存失效
    return nil
}
```

**Flush vs Fsync**：
- `Flush`：close() 时调用，`fsyncOnClose` 开启才真正 flush
- `Fsync`：fsync() 系统调用，总是真正 flush

---

## 7. Release：关闭文件

[file.go:350-404](../../client/fs/file.go#L350-L404)

```go
func (f *File) Release(ctx context.Context, req *fuse.ReleaseRequest) (err error) {
    ino := f.info.Inode
    
    defer func() {
        f.fWriter.FreeCache()  // 释放冷卷写缓存
        
        // 引用计数为 0 时清理
        if f.super.ec.RefCnt(ino) == 0 && !f.super.metaCacheAcceleration {
            f.super.fslock.Lock()
            delete(f.super.nodeCache, ino)  // 从 nodeCache 删除
            f.super.fslock.Unlock()
            
            if DisableMetaCache {
                f.super.ic.Delete(ino)  // 从 InodeCache 删除
            }
            
            // 删除父目录的 dcache
            f.super.fslock.Lock()
            node, ok := f.super.nodeCache[f.parentIno]
            if ok {
                parent, ok := node.(*Dir)
                if ok {
                    parent.dcache.Delete(f.name)
                }
            }
            f.super.fslock.Unlock()
        }
    }()
    
    // 关闭流
    err = f.super.ec.CloseStream(ino)  // 减少引用计数
    if err != nil {
        return ParseError(err)
    }
    
    return nil
}
```

**关键点**：
- `ec.CloseStream` 减少引用计数
- 引用计数为 0 时清理各种缓存
- 冷卷释放 `fWriter` 缓存

---

## 8. Forget：驱逐 inode

[file.go:210-244](../../client/fs/file.go#L210-L244)

```go
func (f *File) Forget() {
    ino := f.info.Inode
    
    if DisableMetaCache {
        f.super.ic.Delete(ino)
        f.super.fslock.Lock()
        delete(f.super.nodeCache, ino)
        f.super.fslock.Unlock()
        
        if err := f.super.ec.EvictStream(ino); err != nil {
            log.LogWarnf("Forget: stream not ready to evict, ino(%v) err(%v)", ino, err)
            return
        }
    }
    
    // ★ 清理孤儿 inode
    if !f.super.orphan.Evict(ino) {
        return
    }
    
    fullPath := f.getParentPath() + f.name
    if err := f.super.mw.Evict(ino, fullPath); err != nil {
        log.LogWarnf("Forget Evict: ino(%v) err(%v)", ino, err)
    }
}
```

**孤儿清理**：当 FUSE 内核认为 inode 不再需要时调用 Forget。如果这个 inode 在 orphan 列表中（Nlink==0 但之前还被打开），现在才真正调用 `mw.Evict` 删除。

---

## 9. Setattr：设置属性（包括 truncate）

[file.go:710-777](../../client/fs/file.go#L710-L777)

```go
func (f *File) Setattr(ctx context.Context, req *fuse.SetattrRequest, resp *fuse.SetattrResponse) error {
    ino := f.info.Inode
    
    // ★ truncate 特殊处理
    if req.Valid.Size() && (proto.IsHot(f.super.volType) || proto.IsStorageClassReplica(f.info.StorageClass)) {
        // 热卷才支持 truncate
        if err := f.super.ec.OpenStream(ino, true, false, fullPath); err != nil {
            return ParseError(err)
        }
        defer f.super.ec.CloseStream(ino)
        
        if err := f.super.ec.Flush(ino); err != nil {
            return ParseError(err)
        }
        
        if err := f.super.ec.Truncate(f.super.mw, f.parentIno, ino, int(req.Size), fullPath); err != nil {
            return ParseError(err)
        }
        
        f.super.ic.Delete(ino)
        f.super.ec.RefreshExtentsCache(ino)
    }
    
    // 获取当前 inode 信息
    info, err := f.super.InodeGet(ino)
    if err != nil {
        return ParseError(err)
    }
    
    // 设置其他属性
    if valid := setattr(info, req); valid != 0 {
        err = f.super.mw.Setattr(ino, valid, info.Mode, info.Uid, info.Gid, 
            info.AccessTime.Unix(), info.ModifyTime.Unix())
        if err != nil {
            f.super.ic.Delete(ino)
            return ParseError(err)
        }
    }
    
    fillAttr(info, &resp.Attr)
    return nil
}
```

**truncate 流程**：
1. 打开流
2. 先 flush 再 truncate
3. 刷新 extent 缓存

---

## 10. 冷卷定时 Flush

[super.go:352-376](../../client/fs/super.go#L352-L376)

```go
func (s *Super) scheduleFlush() {
    t := time.NewTicker(2 * time.Second)
    for range t.C {
        ctx := context.Background()
        s.fslock.Lock()
        for ino, node := range s.nodeCache {
            if file, ok := node.(*File); ok {
                // 空闲计数超过阈值就 flush
                if atomic.LoadInt32(&file.idle) >= BlobWriterIdleTimeoutPeriod {  // 10
                    if file.fWriter != nil {
                        atomic.StoreInt32(&file.idle, 0)
                        go file.fWriter.Flush(ino, ctx)
                    }
                } else {
                    atomic.AddInt32(&file.idle, 1)
                }
            }
        }
        s.fslock.Unlock()
    }
}
```

**作用**：冷卷的 Writer 有缓存，如果文件长时间没有写入（idle 超过 10×2=20 秒），后台 flush 把数据刷到 BlobStore。

---

## 11. 本层关键问题自测

1. ✅ 热卷和冷卷的 Read/Write 分别走什么路径？
2. ✅ `fReader` / `fWriter` 什么时候创建？什么时候销毁？
3. ✅ Write 时为什么要做配额检查？
4. ✅ Flush 和 Fsync 有什么区别？`fsyncOnClose` 选项的作用？
5. ✅ Release 时为什么要检查 `ec.RefCnt(ino) == 0`？
6. ✅ Forget 和 orphan 列表的关系是什么？
7. ✅ truncate 为什么只支持热卷？
8. ✅ `scheduleFlush` 的 `idle` 计数机制是什么？

---

## 12. 热卷 vs 冷卷总结

| 维度 | 热卷 (Replica) | 冷卷 (BlobStore) |
|------|----------------|------------------|
| 数据存储 | DataNode (3 副本) | BlobStore (EC) |
| 读取 | `ec.Read()` | `fReader.Read()` |
| 写入 | `ec.Write()` | `fWriter.Write()` |
| truncate | 支持 | 不支持 |
| 后台 flush | 不需要 | `scheduleFlush` 每 2 秒检查 |
| reader/writer | 不需要 | 每次 Open 重建 |

---

## 13. 完整读写流程图

```
   用户态应用
       │
       │ read(fd, buf, size) / write(fd, buf, size)
       ▼
┌──────────────────┐
│  FUSE 内核模块   │
└────────┬─────────┘
         │ FUSE 协议
         ▼
┌──────────────────────────────────────────────────────────────────┐
│                    CubeFS Client                                  │
│                                                                   │
│  ┌────────────┐                                                  │
│  │   File     │                                                  │
│  │ Read/Write │                                                  │
│  └─────┬──────┘                                                  │
│        │                                                          │
│        │ shouldAccessReplicaStorageClass()?                       │
│        │                                                          │
│   热卷 │                           │ 冷卷                         │
│        ▼                           ▼                              │
│  ┌──────────────┐           ┌──────────────┐                     │
│  │ ExtentClient │           │   fReader    │                     │
│  │   ec.Read    │           │   fWriter    │                     │
│  │   ec.Write   │           │ .Read/.Write │                     │
│  └──────┬───────┘           └──────┬───────┘                     │
│         │                          │                              │
│         ▼                          ▼                              │
│    DataNode              BlobStore (EC 存储)                      │
│   (3 副本存储)                                                    │
└──────────────────────────────────────────────────────────────────┘
```
