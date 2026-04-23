# Layer 6: 网络传输

## 1. Transport 接口

```go
// transport.go:22
type Transport interface {
    Send(m *proto.Message)
    SendSnapshot(m *proto.Message, rs *snapshotStatus)
    Stop()
}
```

## 2. MultiTransport 结构

tiglabs/raft 使用双通道传输：心跳和复制分离。

```go
// transport_multi.go:22
type MultiTransport struct {
    heartbeat *heartbeatTransport   // 心跳传输
    replicate *replicateTransport   // 复制传输
}
```

## 3. 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                   MultiTransport 架构                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    RaftServer                        │   │
│   │                                                     │   │
│   │    raft.send(msg) → msgs 队列                       │   │
│   └───────────────────────┬─────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                  MultiTransport                      │   │
│   │                                                     │   │
│   │    Send(msg) {                                      │   │
│   │        if msg.IsHeartbeatMsg() {                    │   │
│   │            heartbeat.send(msg)                      │   │
│   │        } else {                                     │   │
│   │            replicate.send(msg)                      │   │
│   │        }                                            │   │
│   │    }                                                │   │
│   └─────────────────────────────────────────────────────┘   │
│              │                              │               │
│              ▼                              ▼               │
│   ┌──────────────────────┐    ┌──────────────────────┐     │
│   │  heartbeatTransport  │    │  replicateTransport  │     │
│   │                      │    │                      │     │
│   │  端口: HeartbeatPort │    │  端口: ReplicaPort   │     │
│   │  消息: 心跳/投票      │    │  消息: 日志追加/快照  │     │
│   │  特点: 小包、高频     │    │  特点: 大包、低频    │     │
│   └──────────────────────┘    └──────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 创建 MultiTransport

```go
// transport_multi.go:27
func NewMultiTransport(raft *RaftServer, config *TransportConfig) (Transport, error) {
    mt := new(MultiTransport)

    // 创建心跳传输
    ht, _ := newHeartbeatTransport(raft, config)
    mt.heartbeat = ht

    // 创建复制传输
    rt, _ := newReplicateTransport(raft, config)
    mt.replicate = rt

    // 启动
    mt.heartbeat.start()
    mt.replicate.start()
    
    return mt, nil
}
```

## 5. 发送消息

```go
// transport_multi.go:51
func (t *MultiTransport) Send(m *proto.Message) {
    // 根据消息类型选择传输通道
    if m.IsHeartbeatMsg() {
        t.heartbeat.send(m)   // 心跳消息
    } else {
        t.replicate.send(m)   // 复制消息
    }
}

func (t *MultiTransport) SendSnapshot(m *proto.Message, rs *snapshotStatus) {
    t.replicate.sendSnapshot(m, rs)  // 快照走复制通道
}
```

## 6. 消息类型分类

```go
// 心跳通道消息
func (m *Message) IsHeartbeatMsg() bool {
    return m.Type == ReqMsgHeartBeat || 
           m.Type == RespMsgHeartBeat ||
           m.Type == ReqMsgVote ||
           m.Type == RespMsgVote
}

// 复制通道消息
// - ReqMsgAppend / RespMsgAppend
// - ReqMsgSnap
// - 其他
```

## 7. 心跳传输

```go
// transport_heartbeat.go
type heartbeatTransport struct {
    config    *TransportConfig
    raft      *RaftServer
    listener  net.Listener
    mu        sync.RWMutex
    senders   map[uint64]*transportSender
    stopc     chan struct{}
}

func (t *heartbeatTransport) send(m *proto.Message) {
    // 获取或创建到目标节点的 sender
    sender := t.getSender(m.To)
    sender.send(m)
}
```

## 8. 复制传输

```go
// transport_replicate.go
type replicateTransport struct {
    config    *TransportConfig
    raft      *RaftServer
    listener  net.Listener
    mu        sync.RWMutex
    senders   map[uint64]*transportSender
    stopc     chan struct{}
}

func (t *replicateTransport) send(m *proto.Message) {
    sender := t.getSender(m.To)
    sender.send(m)
}

func (t *replicateTransport) sendSnapshot(m *proto.Message, rs *snapshotStatus) {
    sender := t.getSender(m.To)
    sender.sendSnapshot(m, rs)
}
```

## 9. transportSender

```go
// transport_sender.go
type transportSender struct {
    nodeID    uint64
    addr      string
    conn      net.Conn
    mu        sync.Mutex
    msgc      chan *proto.Message
    snapc     chan *snapshotRequest
    stopc     chan struct{}
}

func (s *transportSender) send(m *proto.Message) {
    select {
    case s.msgc <- m:
    default:
        // 队列满，丢弃
    }
}

// 发送循环
func (s *transportSender) run() {
    for {
        select {
        case m := <-s.msgc:
            s.doSend(m)
        case <-s.stopc:
            return
        }
    }
}
```

## 10. 双端口设计原因

```
┌─────────────────────────────────────────────────────────────┐
│                    为什么分两个端口？                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  问题: 如果心跳和日志复制共用一个连接                         │
│                                                             │
│  ┌────────────────────────────────────────────┐            │
│  │  大日志包 (MB级)                            │            │
│  │  ████████████████████████████████████████  │ ← 阻塞     │
│  │  心跳包 (几十字节)                          │            │
│  │  ░                                         │ ← 等待     │
│  └────────────────────────────────────────────┘            │
│                                                             │
│  结果: 心跳被大包阻塞 → 选举超时 → 不必要的 Leader 切换       │
│                                                             │
│  解决: 分离心跳和复制                                        │
│                                                             │
│  HeartbeatPort (17230)         ReplicaPort (17240)         │
│  ┌──────────────────┐          ┌──────────────────┐        │
│  │ 心跳包 ░░░░░░░░   │          │ 大日志包 ████████ │        │
│  │ 小、快、优先级高  │          │ 大、慢、可排队    │        │
│  └──────────────────┘          └──────────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 11. 消息编解码

```go
// 发送
func (s *transportSender) doSend(m *proto.Message) error {
    // 编码
    buf := m.Encode()
    
    // 发送
    _, err := s.conn.Write(buf)
    return err
}

// 接收
func reciveMessage(r *util.BufferReader) (*proto.Message, error) {
    msg := proto.GetMessage()
    err := msg.Decode(r)
    return msg, err
}
```

## 12. 连接管理

```go
// 获取到目标节点的 sender
func (t *transport) getSender(nodeID uint64) *transportSender {
    t.mu.RLock()
    sender, ok := t.senders[nodeID]
    t.mu.RUnlock()
    
    if ok {
        return sender
    }
    
    // 创建新 sender
    t.mu.Lock()
    defer t.mu.Unlock()
    
    // 双重检查
    if sender, ok = t.senders[nodeID]; ok {
        return sender
    }
    
    // 获取目标地址
    addr := t.raft.resolver.NodeAddress(nodeID)
    
    // 创建并启动
    sender = newTransportSender(nodeID, addr)
    t.senders[nodeID] = sender
    go sender.run()
    
    return sender
}
```

## 13. 配置参数

```go
type TransportConfig struct {
    HeartbeatAddr string  // 心跳监听地址
    ReplicateAddr string  // 复制监听地址
    SendBufferSize int    // 发送缓冲区
    RecvBufferSize int    // 接收缓冲区
    MaxMsgSize     int    // 最大消息大小
}
```

## 14. 关键代码位置

| 功能 | 文件:行号 |
|------|-----------|
| Transport 接口 | transport.go:22 |
| MultiTransport | transport_multi.go:22 |
| NewMultiTransport | transport_multi.go:27 |
| Send | transport_multi.go:51 |
| heartbeatTransport | transport_heartbeat.go |
| replicateTransport | transport_replicate.go |
| transportSender | transport_sender.go |

---
*创建时间：2026-04-22*
