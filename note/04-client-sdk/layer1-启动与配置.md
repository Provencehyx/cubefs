# Client FUSE 五层解析 · Layer 1：启动与配置

> 入口文件：[client/fuse.go](../../client/fuse.go)
> 核心问题：**FUSE 客户端进程是怎么从命令行参数启动，一步步把自己变成一个能挂载 CubeFS 卷的文件系统的？**

---

## 1. 顶层架构：Client 在系统中的位置

```
┌─────────────────────────────────────────────────────────────┐
│                    用户态应用 (ls, cat, cp...)               │
└─────────────────────────────┬───────────────────────────────┘
                              │ VFS 系统调用
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Linux FUSE 内核模块                      │
│                   (/dev/fuse 字符设备)                       │
└─────────────────────────────┬───────────────────────────────┘
                              │ FUSE 协议
                              ▼
┌─────────────────────────────────────────────────────────────┐
│               CubeFS Client (本层)                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Super (fs/super.go)                                 │   │
│  │    ├── MetaWrapper (sdk/meta) ──► MetaNode           │   │
│  │    ├── ExtentClient (sdk/data/stream) ──► DataNode   │   │
│  │    ├── InodeCache / Dcache                           │   │
│  │    └── BlobStoreClient ──► BlobStore (可选)          │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Dir / File (fs/dir.go, fs/file.go)                  │   │
│  │    实现 bazil.org/fuse/fs 的 Node/Handle 接口         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**记住一句话**：**Client 是 FUSE 协议的用户态实现，通过 MetaWrapper 访问 MetaNode 获取元数据，通过 ExtentClient 访问 DataNode 读写数据**。

---

## 2. 命令行入口与守护进程模式

[fuse.go:353-367](../../client/fuse.go#L353-L367)

```go
func main() {
    flag.Parse()

    if *configVersion {
        fmt.Print(proto.DumpVersion(Role))
        os.Exit(0)
    }

    if !*configForeground {
        if err := startDaemon(); err != nil {   // ★ 默认后台模式
            fmt.Printf("Mount failed: %v\n", err)
            os.Exit(1)
        }
        os.Exit(0)
    }
    // 以下是守护进程里的逻辑...
}
```

**命令行参数**：

| 参数 | 含义 | 默认 |
|------|------|------|
| `-c` | 配置文件路径 | 必填 |
| `-v` | 打印版本 | - |
| `-f` | 前台运行（调试用） | 后台 |
| `-r` | 恢复 FUSE 连接 | - |

**守护进程化**：默认不带 `-f` 时，`startDaemon()` 会 fork 一个子进程，父进程立即退出。这样用户 `mount` 命令返回时，挂载已经完成。

---

## 3. 配置加载与验证

启动流程分两步加载配置：

### 3.1 本地配置解析 `parseMountOption`

[fuse.go:933-1079](../../client/fuse.go#L933-L1079)

关键配置项分组：

| 组 | 字段 | 含义 |
|---|---|---|
| **必填** | `mountPoint`, `volName`, `owner`, `masterAddr` | 挂载点、卷名、所有者、Master 地址 |
| **网络** | `profPort`, `locallyProf` | pprof 端口、是否只监听本地 |
| **缓存** | `icacheTimeout`, `lookupValid`, `attrValid` | inode 缓存超时、lookup 有效期、attr 有效期 |
| **读写** | `readRate`, `writeRate`, `enSyncWrite` | 限速、同步写 |
| **BCache** | `bcacheDir`, `enableBcache`, `bcacheFilterFiles` | 块缓存目录、启用、过滤文件 |
| **高级** | `maxStreamerLimit`, `enableAudit`, `requestTimeout` | 流数限制、审计日志、请求超时 |

### 3.2 从 Master 加载配置 `loadConfFromMaster`

[fuse.go:1161-1187](../../client/fuse.go#L1161-L1187)

```go
func loadConfFromMaster(opt *proto.MountOptions) (err error) {
    mc := master.NewMasterClientFromString(opt.Master, false)
    volumeInfo, err := mc.AdminAPI().GetVolumeSimpleInfo(opt.Volname)
    // 从 volumeInfo 填充:
    opt.VolType = volumeInfo.VolType           // 热卷/冷卷
    opt.EbsBlockSize = volumeInfo.ObjBlockSize // EC 块大小
    opt.EnableQuota = volumeInfo.EnableQuota   // 配额开关
    opt.EnableTransaction = ...                // 事务开关
    opt.VolStorageClass = ...                  // 存储类
    
    clusterInfo, _ := mc.AdminAPI().GetClusterInfo()
    opt.EbsEndpoint = clusterInfo.EbsAddr      // BlobStore 地址
    opt.EbsServicePath = clusterInfo.ServicePath
}
```

**为什么要两步？** 本地配置是用户写的，Master 配置是集群下发的。例如 `VolType`（热/冷卷）、`EnableQuota` 这些只有 Master 知道。

---

## 4. mount 函数：挂载核心流程

[fuse.go:751-887](../../client/fuse.go#L751-L887)

```go
func mount(opt *proto.MountOptions) (fsConn *fuse.Conn, super *cfs.Super, err error) {
    // ① 检查挂载点是否已被占用
    for _, mountPoint := range getMountPoints() {
        if mountPoint == opt.MountPoint {
            return nil, nil, errors.NewErrorf("mountpoint:%v has been mounted", ...)
        }
    }

    // ② 创建 Super（核心！）
    super, err = cfs.NewSuper(opt)

    // ③ 注册 HTTP 控制接口
    http.HandleFunc(ControlCommandSetRate, super.SetRate)
    http.HandleFunc(ControlCommandGetRate, super.GetRate)
    http.HandleFunc(ControlCommandSuspend, super.SetSuspend)
    http.HandleFunc(ControlCommandResume, super.SetResume)
    // ...

    // ④ 启动后台配置更新
    go func() {
        t := time.NewTicker(UpdateConfInterval)  // 2 分钟
        for range t.C {
            volumeInfo, _ := mc.AdminAPI().GetVolumeSimpleInfo(opt.Volname)
            super.SetTransaction(...)
            // 检测卷是否被删除
            if volumeInfo.Status == proto.VolStatusMarkDelete {
                os.Exit(1)  // ★ 卷被删除，客户端退出
            }
        }
    }()

    // ⑤ 配置 FUSE 挂载选项
    options := []fuse.MountOption{
        fuse.AllowOther(),              // 允许其他用户访问
        fuse.MaxReadahead(512 * 1024),  // 预读 512KB
        fuse.AsyncRead(),               // 异步读
        fuse.FSName(opt.FileSystemName),
        fuse.RequestTimeout(opt.RequestTimeout),
    }
    if opt.WriteCache { options = append(options, fuse.WritebackCache()) }
    if opt.EnablePosixACL { options = append(options, fuse.PosixACL()) }

    // ⑥ 真正挂载
    fsConn, err = fuse.Mount(opt.MountPoint, opt.NeedRestoreFuse, options...)
    return
}
```

**6 个关键步骤**：

1. **挂载点检查**：防止重复挂载
2. **创建 Super**：初始化 MetaWrapper + ExtentClient（Layer 2 详解）
3. **HTTP 控制接口**：运行时调参（限速、暂停、恢复）
4. **后台配置同步**：每 2 分钟从 Master 拉取卷状态
5. **FUSE 选项**：根据配置启用各种特性
6. **执行挂载**：调用 bazil.org/fuse 库

---

## 5. FUSE 服务主循环

[fuse.go:599-611](../../client/fuse.go#L599-L611)

```go
if err = fs.Serve(fsConn, super, opt); err != nil {
    syslog.Printf("fs Serve returns err(%v)", err)
    os.Exit(1)
}

<-fsConn.Ready  // ★ 阻塞等待挂载完成
if fsConn.MountError != nil {
    os.Exit(1)
}
syslog.Printf("exit normally\n")
```

`fs.Serve` 是 bazil.org/fuse 库的入口，它会：
1. 从 `/dev/fuse` 读取内核发来的请求
2. 调用 `Super` 实现的各种 FUSE 操作（Root、Lookup、Read、Write...）
3. 把结果写回内核

**阻塞点**：`<-fsConn.Ready` 会阻塞直到 `umount` 或出错。

---

## 6. 信号处理与优雅退出

[fuse.go:889-931](../../client/fuse.go#L889-L931)

```go
var exitSignals = []os.Signal{
    syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT,
    syscall.SIGSEGV, syscall.SIGFPE, syscall.SIGBUS, ...
}

func registerInterceptedSignal(mnt string, cb func(bool) bool) {
    sigC := make(chan os.Signal, 1)
    signal.Ignore(syscall.SIGURG, syscall.SIGPIPE, syscall.SIGHUP)
    signal.Notify(sigC, exitSignals...)

    go func() {
        for sig := range sigC {
            syslog.Printf("Received signal (%v)", sig)
            if isExitSignal(sig) {
                auditlog.StopAudit()
                log.LogFlush()
                os.Exit(1)
            }
        }
    }()
}
```

**处理策略**：
- **忽略**：SIGURG、SIGPIPE、SIGHUP
- **退出**：SIGINT(Ctrl+C)、SIGTERM、SIGQUIT 等
- 退出前刷新日志、停止审计

---

## 7. 权限检查

[fuse.go:1081-1114](../../client/fuse.go#L1081-L1114)

```go
func checkPermission(opt *proto.MountOptions) (err error) {
    mc := master.NewMasterClientFromString(opt.Master, false)
    localIP, _ := ump.GetLocalIpAddr()
    
    // ① IP ACL 检查
    if info, err := mc.UserAPI().AclOperation(opt.Volname, localIP, util.AclCheckIP); !info.OK {
        return proto.ErrNoAclPermission
    }
    
    // ② AccessKey/SecretKey 检查（如果配置了）
    if opt.AccessKey != "" {
        userInfo, _ := mc.UserAPI().GetAKInfo(opt.AccessKey)
        if userInfo.SecretKey != opt.SecretKey {
            return proto.ErrNoPermission
        }
        policy := userInfo.Policy
        if policy.IsOwn(opt.Volname) { return nil }  // owner 有全部权限
        // 检查 POSIX 读写权限
        if policy.IsAuthorized(opt.Volname, opt.SubDir, proto.POSIXReadAction) &&
           !policy.IsAuthorized(opt.Volname, opt.SubDir, proto.POSIXWriteAction) {
            opt.Rdonly = true  // ★ 只有读权限，强制只读挂载
        }
    }
}
```

**两道检查**：
1. **IP ACL**：这个 IP 能不能挂载这个卷
2. **用户策略**：这个用户在这个卷的这个子目录下有什么权限

---

## 8. 本层关键问题自测

1. ✅ 为什么默认是后台运行？前台运行 `-f` 有什么用？
2. ✅ `parseMountOption` 和 `loadConfFromMaster` 各负责什么？为什么要分两步？
3. ✅ `mount` 函数的 6 个步骤顺序能调换吗？
4. ✅ 后台配置更新循环发现卷被删除会怎样？
5. ✅ `fuse.AllowOther()` 选项的作用是什么？
6. ✅ `fs.Serve` 是阻塞的还是非阻塞的？
7. ✅ 只有读权限时客户端会怎么处理？

---

## 9. 给后续 Layer 的钩子

- **Layer 2 (Super)**：`cfs.NewSuper(opt)` 内部做了什么？MetaWrapper 和 ExtentClient 是怎么初始化的？InodeCache 的过期策略是什么？
- **Layer 3 (Dir)**：目录的 Lookup、ReadDir、Create、Remove 怎么和 MetaNode 交互？
- **Layer 4 (File)**：文件的 Read、Write、Flush、Fsync 怎么和 DataNode 交互？冷卷和热卷的处理有什么不同？
