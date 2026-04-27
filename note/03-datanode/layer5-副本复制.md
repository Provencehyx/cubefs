# Layer 5: 链式复制协议

## 核心问题

1. **为什么不用 Raft 复制数据？**
2. **链式复制如何保证一致性？**
3. **一个副本失败怎么处理？**

---

## 1. 为什么选择链式复制？

### Raft vs 链式复制

```
Raft 复制 (3 副本):
┌──────────┐
│  Leader  │ ──────┬──────▶ Follower1 ───ACK───┐
│          │       │                            │
│          │       └──────▶ Follower2 ───ACK───┼──▶ 提交
└──────────┘                                    │
     ▲                                          │
     └──────────────────────────────────────────┘
     
网络往返: 2 次 (Leader→Follower, Follower→Leader)
带宽利用: Leader 发送 2 份完整数据


链式复制 (3 副本):
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Leader  │────▶│ Follower1│────▶│ Follower2│
│          │     │          │     │          │
│          │◀────│          │◀────│    ACK   │
└──────────┘     └──────────┘     └──────────┘

网络往返: 1 次 (沿链传递)
带宽利用: 每节点只发 1 份数据
```

### 性能对比

| 指标 | Raft | 链式复制 |
|------|------|----------|
| 写延迟 | 2 RTT | 链长 × 1 RTT |
| Leader 带宽 | N × 数据量 | 1 × 数据量 |
| 适合场景 | 小数据、强一致 | 大数据、高吞吐 |

**结论**：
- 数据写入量大（GB 级别），链式复制带宽效率更高
- 元数据操作量小但要求强一致，用 Raft

---

## 2. 链式复制流程

### 正常写入

```
Client 写入 100MB 数据
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│                        Leader (DN1)                          │
│  1. 接收数据                                                  │
│  2. 写入本地 ExtentStore                                      │
│  3. 转发给 Follower1                                          │
└─────────────────────────┬────────────────────────────────────┘
                          │ 100MB
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                      Follower1 (DN2)                         │
│  1. 接收数据                                                  │
│  2. 写入本地 ExtentStore                                      │
│  3. 转发给 Follower2                                          │
└─────────────────────────┬────────────────────────────────────┘
                          │ 100MB
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                      Follower2 (DN3)                         │
│  1. 接收数据                                                  │
│  2. 写入本地 ExtentStore                                      │
│  3. 链尾，返回 ACK                                            │
└─────────────────────────┬────────────────────────────────────┘
                          │ ACK
                          ▼
               ACK 沿链回传到 Leader → 返回 Client
```

### 为什么能保证一致性？

**关键：写入顺序 = 链顺序**

```
写操作 W1, W2, W3 到达 Leader

Leader:   W1 → W2 → W3  (按到达顺序处理)
    │
    ▼
Follower1: W1 → W2 → W3  (保持相同顺序)
    │
    ▼
Follower2: W1 → W2 → W3  (保持相同顺序)

所有副本的写入顺序一致 → 最终状态一致
```

**但是**：如果是覆盖写（随机写），可能有并发问题，所以随机写需要 Raft。

---

## 3. 核心组件

### ReplProtocol（复制协议管理器）

```go
// repl/repl_protocol.go:39
type ReplProtocol struct {
    sourceConn       net.Conn                        // 上游连接
    followerConnects map[string]*FollowerTransport  // 下游连接
    
    toBeProcessedCh  chan *Packet   // 待处理队列
    responseCh       chan *Packet   // 响应队列
    
    prepareFunc      func(p *Packet) error   // 预处理
    operatorFunc     func(p *Packet, c net.Conn) error  // 本地操作
    postFunc         func(p *Packet) error   // 后处理
}
```

### FollowerTransport（下游连接管理）

```go
// repl/repl_protocol.go:67
type FollowerTransport struct {
    addr    string
    conn    net.Conn
    sendCh  chan *FollowerPacket  // 异步发送，缓冲 200
    recvCh  chan *FollowerPacket  // 异步接收
}
```

**为什么用 channel 异步？**
- 写磁盘和网络发送可以并行
- 提高吞吐量

---

## 4. 失败处理

### 场景：Follower2 故障

```
写入请求
    │
    ▼
Leader ───写入成功─▶ Follower1 ───写入成功─▶ Follower2
                                                 ╳ 故障
                                                 │
                                                 ▼
                                           超时 (5秒)
                                                 │
                                                 ▼
                              ┌──────────────────────────────┐
                              │ 返回部分成功:                 │
                              │ Leader + Follower1 已写入    │
                              │ Follower2 失败               │
                              └──────────────────────────────┘
                                                 │
                                                 ▼
                              ┌──────────────────────────────┐
                              │ 上报 Master:                  │
                              │ Follower2 副本异常           │
                              └──────────────────────────────┘
                                                 │
                                                 ▼
                              ┌──────────────────────────────┐
                              │ Master 触发修复:              │
                              │ 从 Leader 读数据修复 Follower2│
                              └──────────────────────────────┘
```

### 超时设置

```go
// proto/packet.go
const (
    ReadDeadlineTime  = 5   // 读超时 5 秒
    WriteDeadlineTime = 5   // 写超时 5 秒
)
```

**为什么是 5 秒？**
- 太短：正常网络抖动误判
- 太长：故障检测慢
- 5 秒：平衡点

---

## 5. 数据面 vs 控制面

```
┌─────────────────────────────────────────────────────────────────┐
│                        DataPartition                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────┐  ┌────────────────────────────┐ │
│  │       数据面 (Data Plane)   │  │    控制面 (Control Plane)  │ │
│  │                            │  │                            │ │
│  │  协议: 链式复制            │  │  协议: Raft                │ │
│  │                            │  │                            │ │
│  │  操作:                     │  │  操作:                     │ │
│  │  - 追加写 Extent           │  │  - 成员变更 (AddNode)      │ │
│  │  - 读 Extent               │  │  - 随机写 (覆盖写)         │ │
│  │  - 删除 Extent             │  │  - Leader 选举             │ │
│  │                            │  │                            │ │
│  │  特点:                     │  │  特点:                     │ │
│  │  - 高吞吐                  │  │  - 强一致                  │ │
│  │  - 最终一致                │  │  - 低频操作                │ │
│  └────────────────────────────┘  └────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么随机写需要 Raft？

```
场景：两个 Client 同时覆盖同一位置

没有 Raft:
  Client1: Write(offset=0, data=A)
  Client2: Write(offset=0, data=B)
  
  Leader:     先收到 A，写 A
  Follower1:  先收到 B，写 B（网络延迟不一致）
  
  结果: Leader 是 A，Follower1 是 B → 不一致！

有 Raft:
  所有随机写提交到 Raft 日志
  Raft 确定全局顺序: A 在前，B 在后
  所有副本按此顺序执行 → 一致
```

---

## 6. 请求处理流程

```
Client 请求到达 Leader
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ ReplProtocol.ServerConn()                                     │
│        │                                                      │
│        ▼                                                      │
│ prepareFunc()                                                 │
│   - 解析 Packet                                               │
│   - 获取 Follower 地址列表                                     │
│        │                                                      │
│        ▼                                                      │
│ toBeProcessedCh ← Packet                                      │
│        │                                                      │
└────────┼──────────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────────┐
│ OperatorAndForwardPktGoRoutine()                              │
│        │                                                      │
│        ├── operatorFunc()  → 本地 ExtentStore.Write()         │
│        │                                                      │
│        └── 并行转发到 Followers                                │
│            ┌──────────────────┐                               │
│            │ FollowerTransport│                               │
│            │   sendCh ← pkt   │ → serverWriteToFollower()     │
│            │   recvCh → resp  │ ← serverReadFromFollower()    │
│            └──────────────────┘                               │
│                    │                                          │
│                    ▼                                          │
│        等待所有 Follower 响应                                  │
│                    │                                          │
└────────────────────┼──────────────────────────────────────────┘
                     │
                     ▼
            postFunc() → responseCh → 返回 Client
```

---

## 7. 关键代码

| 功能 | 位置 | 要点 |
|------|------|------|
| 复制协议 | repl/repl_protocol.go:39 | `ReplProtocol` 管理整个复制流程 |
| 下游连接 | repl/repl_protocol.go:67 | `FollowerTransport` 异步发送/接收 |
| 写入下游 | repl/repl_protocol.go:90 | `serverWriteToFollower()` |
| 读取响应 | repl/repl_protocol.go:114 | `serverReadFromFollower()` |
| 超时控制 | proto/packet.go | `ReadDeadlineTime = 5s` |

---

*更新时间：2026-04-27*
