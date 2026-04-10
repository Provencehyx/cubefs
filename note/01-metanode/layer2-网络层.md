# MetaNode 七层解析 · Layer 2：网络层

> 入口文件：[metanode/server.go](../../metanode/server.go)
> 协议帧解析：[proto/packet.go](../../proto/packet.go)（`ReadFromConnWithVer`）
> 核心问题：**字节流是怎么变成一个 `*Packet` 对象，然后被交给业务层的？**

---

## 1. 网络层在系统里的位置

```
            ┌──────────────────────────────────────────┐
            │           Layer 1：启动与配置             │
            │ doStart() → m.startServer()/SmuxServer() │
            └─────────────────────┬────────────────────┘
                                  │
   ┌──────────────────────────────┴──────────────────────────────┐
   │                                                              │
   ▼                                                              ▼
┌────────────────────────┐                          ┌─────────────────────────┐
│ 普通 TCP 监听            │                          │ smux 多路复用监听         │
│ port = listen          │                          │ port = listen+smuxShift │
│ 一连接一 goroutine、串行 │                          │ 一连接 N 个 stream、并发  │
└──────────┬─────────────┘                          └────────────┬────────────┘
           │                                                     │
           │  serveConn(conn)                                    │  serveSmuxConn → serveSmuxStream
           │                                                     │
           └────────────────────┬───────────────┬────────────────┘
                                ▼               ▼
                       ┌─────────────────────────────────┐
                       │   Packet.ReadFromConnWithVer    │  ← 协议帧解析
                       └────────────────┬────────────────┘
                                        ▼
                       ┌─────────────────────────────────┐
                       │  m.handlePacket(conn, p, addr)  │  ← Layer 2 与 Layer 3 的边界
                       └────────────────┬────────────────┘
                                        ▼
                            metadataManager.HandleMetadataOperation(...)   ◄── 进入 Layer 3
```

**记住一个关键边界**：`handlePacket` 是 Layer 2 的最后一行代码——它把 `*Packet` 交给 `metadataManager`，**之后的事情就和"连接"无关了**。

---

## 2. 普通 TCP 入口：`startServer`

[server.go:31](../../metanode/server.go#L31)

```go
func (m *MetaNode) startServer() (err error) {
    m.httpStopC = make(chan uint8)              // ① 停机信号

    addr := fmt.Sprintf(":%s", m.listen)
    if m.bindIp { addr = m.localAddr + ":" + m.listen }   // ② bindIp 决定是否绑特定 IP

    ln, err := net.Listen("tcp", addr)          // ③ 普通 TCP listen
    go func(stopC chan uint8) {
        defer ln.Close()
        for {
            conn, _ := ln.Accept()
            select { case <-stopC: return; default: }
            go m.serveConn(conn, stopC)         // ④ 一连接一 goroutine
        }
    }(m.httpStopC)
}
```

四个细节值得记：

1. **`httpStopC`**：用 `chan uint8` 当信号量，`stopServer` 调 `close(stopC)` 时所有 `serveConn` 协程下次循环会退出。变量名叫 `httpStopC` 但其实是 TCP 服务的停机通道——历史遗留命名。
2. **`bindIp` 决定监听 0.0.0.0 还是具体 IP**。在多网卡或安全要求严格的环境会用到。
3. **Accept 后再 select stopC**：保证停机后 Accept 出来的连接也会被丢弃。
4. **没有 panic recover**：监听协程一旦崩了整个 listener 就死了，靠 process 级别监控来兜底。

### `serveConn`：一条连接的生命周期

[server.go:75](../../metanode/server.go#L75)

```go
func (m *MetaNode) serveConn(conn net.Conn, stopC chan uint8) {
    defer func() { conn.Close(); m.RemoveConnection() }()
    m.AddConnection()                    // ① 计数 +1

    c := conn.(*net.TCPConn)
    c.SetKeepAlive(true)                 // ② TCP keepalive，防长连接静默断
    c.SetNoDelay(true)                   // ③ 关 Nagle，元数据小包延迟优先

    for {                                 // ④ 同一连接顺序读多个 packet
        select { case <-stopC: return; default: }
        p := &Packet{}
        if err := p.ReadFromConnWithVer(conn, proto.NoReadDeadlineTime); err != nil {
            return                        // ⑤ 读出错就关连接
        }
        if err := m.handlePacket(conn, p, remoteAddr); err != nil {
            // ⑥ 业务错误：根据错误类型决定 warn 还是 error
        }
    }
}
```

**几个非常重要的设计决定**：

- **同一条 TCP 连接的请求是串行处理的**：for 循环里"读一个包 → 处理 → 再读下一个"。这是普通 TCP 路径**有 HOL 阻塞**的根因，也正是 smux 路径存在的理由。
- **`NoReadDeadlineTime`**：`SetReadDeadline(time.Time{})` 即**无读超时**。客户端可以保持长连接静默挂着。靠 keepalive 探活。
- **错误日志分级很讲究**（[server.go:104-110](../../metanode/server.go#L104-L110)）：
  - `over quota` / `inode ID out of range` / `unknown meta partition` / `OpNotExistErr`
    → 这些是**预期的业务错误**，打 `warn`
  - 其它的打 `error`
  - 这种分级对 oncall 非常友好——告警只盯 error 级。**新人写代码时也要这么处理日志噪音**。
- **`OpWriteOpOfProtoVerForbidden` 直接 return**：[server.go:101](../../metanode/server.go#L101)。这是兼容旧协议的逻辑——如果当前 vol 禁止 ProtoVer0 的写，遇到这种请求直接断连接，避免一直被刷。

---

## 3. SMux 入口：`startSmuxServer`

[server.go:123](../../metanode/server.go#L123)

```go
addr := util.ShiftAddrPort(":17210", smuxPortShift)   // ① 端口偏移
ln, err := net.Listen("tcp", addr)
go func(stopC chan uint8) {
    for {
        conn, _ := ln.Accept()
        go m.serveSmuxConn(conn, stopC)               // ② 每条底层 TCP 一个 session
    }
}(m.smuxStopC)
```

**和普通 TCP 的差异**只在两点：
1. **端口偏移**：`util.ShiftAddrPort(ip:port, shift)` → `ip:(port+shift)`。所以 MetaNode 占两个端口。
2. **多了一层 session/stream 的拆分**。

### `serveSmuxConn`：把一条 TCP 拆成 session，再拆 stream

[server.go:171](../../metanode/server.go#L171)

```go
func (m *MetaNode) serveSmuxConn(conn net.Conn, stopC chan uint8) {
    m.AddConnection()                            // ① 注意：按 TCP 计数，不按 stream
    sess, _ := smux.Server(conn, smuxPoolCfg.Config)
    defer sess.Close()

    for {
        stream, err := sess.AcceptStream()
        if err != nil {
            if util.FilterSmuxAcceptError(err) != nil {
                log.LogErrorf(...)
            } else {
                log.LogInfof(...)                // ② 客户端正常关闭走 info
            }
            break
        }
        go m.serveSmuxStream(stream, remoteAddr, stopC)   // ③ stream 独立 goroutine
    }
}
```

### `serveSmuxStream`：一个 stream 的生命周期

[server.go:211](../../metanode/server.go#L211)

```go
for {
    select { case <-stopC: return; default: }
    p := &Packet{}
    if err := p.ReadFromConnWithVer(stream, proto.NoReadDeadlineTime); err != nil { return }
    if err := m.handlePacket(stream, p, remoteAddr); err != nil { ... }
}
```

注意 `stream` 实现了 `net.Conn` 接口，所以 `ReadFromConnWithVer` 和 `handlePacket` 可以**完全复用** TCP 的代码——这是 smux 库的精妙之处。

### 三个值得记的对比点

| | 普通 TCP | smux |
|---|---|---|
| 并发单位 | 一个连接一个 goroutine | 一个 stream 一个 goroutine |
| HOL 阻塞 | ❌ 同连接慢请求阻塞后续 | ✅ stream 互不影响 |
| 连接计数 | 每个客户端 ≈ 一条连接 | 一条底层 TCP 服务多 stream，**计数偏低** |
| 客户端连接数 | 高并发场景下爆炸 | 一条 TCP 复用 |
| 错误日志 | 简单 | 需要 `FilterSmuxAcceptError` 区分正常/异常关闭 |

---

## 4. 协议帧：`Packet.ReadFromConnWithVer`

[proto/packet.go:1177](../../proto/packet.go#L1177)。这是 CubeFS 自定义的二进制协议，**所有内部组件都用这个**（不只是 MetaNode）。

读取顺序（按字节流出现的顺序）：

```
┌─────────────────────────────────────────────┐
│ ① PacketHeader (固定 util.PacketHeaderSize) │ ──► UnmarshalHeader 拿到 magic/op/size/...
├─────────────────────────────────────────────┤
│ ② 协议扩展字段 (变长)                         │ ──► TryReadExtraFieldsFromConn
├─────────────────────────────────────────────┤
│ ③ 版本列表（如果 IsVersionList）              │ ──► 多版本快照场景才有
├─────────────────────────────────────────────┤
│ ④ Arg (长度 = ArgLen)                        │ ──► 业务参数（比如 dp 地址列表）
├─────────────────────────────────────────────┤
│ ⑤ Data (长度 = Size)                         │ ──► 真正的请求负载
└─────────────────────────────────────────────┘
```

**几个设计细节**：

- **header buffer 复用**：`Buffers.Get(util.PacketHeaderSize)` 从对象池里拿 header buffer，读完 `defer Buffers.Put(header)` 还回去。**高频小分配的优化**——记住这个池，后面 Layer 5/6 还会见到。
- **超时控制**：`timeoutSec == NoReadDeadlineTime` 时设置零值 deadline 即"无超时"。MetaNode 服务端走的就是这个分支。
- **协议版本兼容**：函数名带 "WithVer"，里面会处理 `IsVersionList`、`UnmarshalVersionSlice` 等多版本场景——这是 CubeFS 引入"快照/多版本"后加的字段，新人看到 `VerSeq` 就知道是多版本相关。
- **大块数据走 buffer pool**：[packet.go:1251](../../proto/packet.go#L1251) 当写操作 size == BlockSize 时，从 `Buffers` 池里取，否则普通 `make`。**4KB/8KB 等典型块大小走池，减少 GC 压力**。
- **协议错误的影响范围**：解析失败直接 return → 上层 `serveConn`/`serveSmuxStream` 会关连接。**协议错误 = 连接级故障**。

### `Packet` 在 MetaNode 的本地包装

[metanode/packet.go:26](../../metanode/packet.go#L26)

```go
type Packet struct {
    proto.Packet
}
```

只是嵌入 `proto.Packet`，外加几个本地工厂方法（`NewPacketToDeleteExtent` 等）和 `AdminOp()` 判断。**所以 MetaNode 的 Packet 和 DataNode、Master 用的是同一个底层结构**——这是协议层统一的好处。

---

## 5. 连接计数与监控

[metanode.go:594-601](../../metanode/metanode.go#L594-L601)

```go
func (m *MetaNode) AddConnection()    { atomic.AddInt64(&m.connectionCnt, 1) }
func (m *MetaNode) RemoveConnection() { atomic.AddInt64(&m.connectionCnt, -1) }
```

**陷阱预警**：
- 普通 TCP 路径：`connectionCnt` ≈ 客户端数
- smux 路径：`connectionCnt` = 底层 TCP 数 ≪ stream 数（≈ 真实并发数）

**监控时不能简单把 `connectionCnt` 当并发量看**。

---

## 6. 停机：`stopServer` / `stopSmuxServer`

```go
func (m *MetaNode) stopServer()     { close(m.httpStopC) }
func (m *MetaNode) stopSmuxServer() { smuxPool.Close(); close(m.smuxStopC) }
```

注意 `stopSmuxServer` **多了一步 `smuxPool.Close()`**——这是关闭**作为 smux 客户端的连接池**。MetaNode 不仅是 smux 服务端（接客户端连），也会**作为 smux 客户端**主动连接其它对端（比如 [manager_proxy.go](../../metanode/manager_proxy.go) 把请求转发给其它 mp 的 leader）。

**记住**：MetaNode 节点之间走 smux 互联是常态。

---

## 7. 本层关键问题自测

读完 Layer 2 应该能回答：

1. ✅ MetaNode 同时开两个监听端口的目的是什么？端口偏移是怎么算的？
2. ✅ 为什么普通 TCP 路径"同连接的请求是串行的"？这是 bug 还是设计？
3. ✅ `serveConn` 用了 `NoReadDeadlineTime` 不设读超时，靠什么避免连接泄漏？
4. ✅ `Packet.ReadFromConnWithVer` 解析顺序是怎样的？为什么 header 走 buffer 池？
5. ✅ smux 场景下监控里的"连接数"为什么会偏低？
6. ✅ Layer 2 与 Layer 3 的代码边界在哪一行？

---

## 8. 给 Layer 3 的钩子

下层我们要回答：

- `m.handlePacket` 调到的 `metadataManager.HandleMetadataOperation` 在 [manager_op.go](../../metanode/manager_op.go) 里，它**怎么根据 `p.Opcode` 把请求路由到具体的 handler 函数**？
- 怎么根据 `p.PartitionID` 找到本地的 MetaPartition？
- mp 还没加载完的时候请求进来怎么办？返回什么错误？
- 哪些 op 是 admin op（[metanode/packet.go:124](../../metanode/packet.go#L124) 的 `AdminOp()`）？为什么要单独标？
