# Layer 5: 副本复制协议

## 1. 复制协议概述

CubeFS DataNode 使用**链式复制（Chain Replication）**协议来保证数据副本一致性。

```
┌─────────────────────────────────────────────────────────────┐
│                      链式复制流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client                                                     │
│    │                                                        │
│    │ 1. 写请求                                               │
│    ▼                                                        │
│  ┌──────────┐    2. 转发    ┌──────────┐    3. 转发    ┌──────────┐
│  │ Leader   │ ──────────▶  │ Follower1│ ──────────▶  │ Follower2│
│  │ (DN1)    │              │ (DN2)    │              │ (DN3)    │
│  └──────────┘              └──────────┘              └──────────┘
│       ▲                          │                        │
│       │                          │                        │
│       │    5. 返回成功            │    4. 返回成功         │
│       └──────────────────────────┴────────────────────────┘
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. ReplProtocol 结构

```go
// repl_protocol.go:39
type ReplProtocol struct {
    packetList      *list.List              // 接收的数据包列表
    ackCh           chan struct{}           // 确认通道
    
    toBeProcessedCh chan *Packet            // 待处理数据包通道
    responseCh      chan *Packet            // 响应通道
    
    sourceConn      net.Conn                // 客户端连接
    exitC           chan bool
    
    followerConnects map[string]*FollowerTransport  // Follower 连接
    
    // 处理函数
    prepareFunc     func(p *Packet) error   // 预处理
    operatorFunc    func(p *Packet, c net.Conn) error  // 执行操作
    postFunc        func(p *Packet) error   // 后处理
}
```

## 3. FollowerTransport 结构

管理与单个 Follower 的通信。

```go
// repl_protocol.go:67
type FollowerTransport struct {
    addr     string                // Follower 地址
    conn     net.Conn              // 连接
    sendCh   chan *FollowerPacket  // 发送通道 (缓冲 200)
    recvCh   chan *FollowerPacket  // 接收通道 (缓冲 200)
    exitCh   chan struct{}
}
```

## 4. Packet 结构

```go
// packet.go:74
type Packet struct {
    proto.Packet                    // 继承基础协议包
    followersAddrs  []string        // Follower 地址列表
    followerPackets []*FollowerPacket
    NeedReply       bool            // 是否需要回复
    Object          interface{}     // 关联对象
}

// packet.go:90
type FollowerPacket struct {
    proto.Packet
    respCh chan error               // 响应通道
}
```

## 5. 复制流程详解

### 5.1 写入流程

```
Client 写请求
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│                    Leader DataNode                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. ServerConn 接收请求                                   │
│         │                                                │
│         ▼                                                │
│  2. prepareFunc() 解析 Follower 地址                      │
│         │                                                │
│         ▼                                                │
│  3. toBeProcessedCh ← Packet                             │
│         │                                                │
│         ▼                                                │
│  4. OperatorAndForwardPktGoRoutine                       │
│         │                                                │
│         ├── 本地写入 ExtentStore                          │
│         │                                                │
│         └── 转发到 Followers                              │
│                   │                                      │
│                   ▼                                      │
│         ┌─────────────────────────────────┐              │
│         │    FollowerTransport.sendCh     │              │
│         │         │                       │              │
│         │         ▼                       │              │
│         │  serverWriteToFollower()        │              │
│         │         │                       │              │
│         │         ▼                       │              │
│         │  serverReadFromFollower()       │              │
│         │         │                       │              │
│         │         ▼                       │              │
│         │    等待响应                      │              │
│         └─────────────────────────────────┘              │
│                   │                                      │
│  5. postFunc() 处理响应                                   │
│         │                                                │
│         ▼                                                │
│  6. responseCh → 返回客户端                               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.2 Follower 处理流程

```
Leader 转发请求
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│                   Follower DataNode                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. 接收请求                                              │
│         │                                                │
│         ▼                                                │
│  2. 检查是否还有下游 Follower                              │
│         │                                                │
│         ├── 有 → 继续转发到下游                           │
│         │                                                │
│         └── 无 → 本地写入                                 │
│                   │                                      │
│                   ▼                                      │
│         ExtentStore.Write()                              │
│                   │                                      │
│                   ▼                                      │
│  3. 返回响应给上游                                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## 6. 数据包类型

| 操作码 | 说明 |
|--------|------|
| OpWrite | 写入数据 |
| OpRead | 读取数据 |
| OpExtentRepairRead | 修复读 |
| OpTinyExtentRepairRead | Tiny Extent 修复读 |
| OpMarkDelete | 标记删除 |
| OpBatchDeleteExtent | 批量删除 |

## 7. 错误处理

### 7.1 Follower 失败

```
写入请求 → Leader
              │
              ├── Follower1 成功
              ├── Follower2 失败 ──→ 标记副本异常
              │                       │
              │                       ▼
              │               上报 Master
              │                       │
              │                       ▼
              │               Master 触发修复
              │
              └── 返回部分成功/失败
```

### 7.2 超时设置

```go
// proto/proto.go
const (
    ReadDeadlineTime                 = 5   // 秒
    WriteDeadlineTime                = 5   // 秒
    BatchDeleteExtentReadDeadLineTime = 120 // 秒
)
```

## 8. 与 Raft 的关系

DataNode 同时使用**链式复制**和 **Raft**：

| 机制 | 用途 | 场景 |
|------|------|------|
| 链式复制 | 数据写入 | 写 Extent 数据 |
| Raft | 元数据一致性 | 分区成员变更、Leader 选举 |

```
┌─────────────────────────────────────────────────────────┐
│                   DataPartition                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   数据面 (Data Plane)          控制面 (Control Plane)   │
│   ┌─────────────────┐         ┌─────────────────┐      │
│   │  链式复制        │         │     Raft        │      │
│   │                 │         │                 │      │
│   │  - 写 Extent    │         │  - Leader 选举  │      │
│   │  - 读 Extent    │         │  - 成员变更     │      │
│   │  - 删除 Extent  │         │  - 配置同步     │      │
│   └─────────────────┘         └─────────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 9. 核心代码位置

| 功能 | 文件:行号 |
|------|-----------|
| ReplProtocol | repl/repl_protocol.go:39 |
| FollowerTransport | repl/repl_protocol.go:67 |
| Packet | repl/packet.go:74 |
| FollowerPacket | repl/packet.go:90 |
| 写入 Follower | repl/repl_protocol.go:90 |
| 读取 Follower 响应 | repl/repl_protocol.go:114 |

---

## 下一步
- Layer 6: Raft 一致性

*创建时间：2026-04-12*
