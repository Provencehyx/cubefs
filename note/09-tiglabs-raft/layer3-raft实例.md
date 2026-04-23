# Layer 3: Raft 实例

## 设计概述

raft 实例是 tiglabs/raft 的核心运行单元，它封装了 Raft 协议状态机（raftFsm），并负责：
- **消息调度**：接收来自业务层的提案和网络层的消息，分发给状态机处理
- **异步应用**：将已提交的日志异步应用到业务状态机
- **生命周期管理**：管理单个 Raft Group 的启动、运行、停止

### 为什么要有 raft 这一层？

tiglabs/raft 采用了**分层设计**，将 Raft 协议实现分为两层：

| 层次 | 组件 | 职责 |
|------|------|------|
| 调度层 | raft | 消息分发、异步处理、与外部系统交互 |
| 协议层 | raftFsm | 纯粹的 Raft 协议逻辑（状态转换、日志复制） |

这种分层的好处：
1. **关注点分离**：raftFsm 专注于协议逻辑，不关心消息怎么来、日志怎么应用
2. **可测试性**：raftFsm 是纯状态机，可以单独进行确定性测试
3. **灵活性**：可以替换调度策略而不影响协议实现

---

## 1. raft 结构体

每个 Raft Group 对应一个 raft 实例。

```go
// raft.go:99
type raft struct {
    raftFsm           *raftFsm           // ★ Raft 协议状态机
    config            *Config            // 全局配置
    raftConfig        *RaftConfig        // 本 Raft Group 配置
    restoringSnapshot util.AtomicBool    // 是否正在恢复快照
    curApplied        util.AtomicUInt64  // 当前已应用的索引
    curSoftSt         unsafe.Pointer     // 软状态（leader, term）
    prevSoftSt        softState
    prevHardSt        proto.HardState    // 硬状态（需持久化）
    peerState         peerState          // peer 列表
    pending           map[uint64]*Future // 等待中的提案
    snapping          map[uint64]*snapshotStatus
    mStatus           *monitorStatus
    
    // ★ 核心通道
    propc     chan *proposal          // 提案输入
    applyc    chan *apply             // 应用输出
    recvc     chan *proto.Message     // 网络消息输入
    snapRecvc chan *snapshotRequest   // 快照请求
    truncatec chan uint64             // 日志截断
    readIndexC chan *Future           // 读索引
    statusc    chan chan *Status      // 状态查询
    tickc      chan struct{}          // tick 信号
    readyc     chan struct{}          // 就绪通知
    electc     chan struct{}          // 选举触发
    stopc      chan struct{}          // 停止信号
    done       chan struct{}          // 完成信号
    
    mu      sync.Mutex
    applyLk sync.Mutex
}
```

### 结构体设计解析

**为什么用 channel 而不是直接调用？**

采用 **Actor 模型**设计，每个 raft 实例是一个独立的 Actor：
- 所有外部交互都通过 channel 进行
- 避免了复杂的锁竞争
- 天然支持异步和批量处理

**通道缓冲区大小的选择：**
| 通道 | 缓冲区 | 设计考量 |
|------|--------|----------|
| recvc | ReqBufferSize | 网络消息可能瞬时大量到达，需要较大缓冲 |
| applyc | AppBufferSize | 应用可能较慢，需要缓冲已提交的日志 |
| propc | 256 | 业务提案有限，固定大小足够 |
| tickc | 64 | tick 信号允许少量堆积，避免阻塞定时器 |
| snapRecvc | 1 | 快照传输是重操作，同时只处理一个 |
| stopc/done | 0 | 同步信号，需要阻塞等待 |

**pending map 的作用：**
- 跟踪"已提交但未应用"的提案
- key 是日志索引，value 是 Future（等待响应的客户端）
- 当日志被应用时，通过 Future 通知客户端

---

## 2. 创建 raft 实例

```go
// raft.go:129
func newRaft(config *Config, raftConfig *RaftConfig) (*raft, error) {
    // 1. 创建 raftFsm
    r, err := newRaftFsm(config, raftConfig)
    if err != nil {
        return nil, err
    }

    // 2. 初始化 raft 实例
    raft := &raft{
        raftFsm:    r,
        config:     config,
        raftConfig: raftConfig,
        pending:    make(map[uint64]*Future),
        snapping:   make(map[uint64]*snapshotStatus),
        
        // 初始化通道
        recvc:     make(chan *proto.Message, config.ReqBufferSize),
        applyc:    make(chan *apply, config.AppBufferSize),
        propc:     make(chan *proposal, 256),
        snapRecvc: make(chan *snapshotRequest, 1),
        truncatec: make(chan uint64, 1),
        tickc:     make(chan struct{}, 64),
        readyc:    make(chan struct{}, 1),
        electc:    make(chan struct{}, 1),
        stopc:     make(chan struct{}),
        done:      make(chan struct{}),
    }
    
    // 3. 启动两个 goroutine
    util.RunWorker(raft.runApply, raft.handlePanic)  // 应用循环
    util.RunWorker(raft.run, raft.handlePanic)       // 主循环
    
    return raft, nil
}
```

### 为什么是两个 goroutine？

这是一个关键的设计决策：**将 Raft 协议处理与状态机应用分离**。

```
┌─────────────────────────────────────────────────────────────┐
│                    单 goroutine 方案（不采用）              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   run() {                                                   │
│       处理消息 → 更新状态 → 应用日志 → 处理下一条消息       │
│   }                                                         │
│                                                             │
│   问题：应用日志可能很慢（写磁盘、更新索引），              │
│        会阻塞 Raft 协议处理，影响选举/心跳时效性            │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    双 goroutine 方案（采用）                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   run()      ─── 快速路径：消息处理、状态转换、心跳响应     │
│       │                                                     │
│       │ applyc                                              │
│       ▼                                                     │
│   runApply() ─── 慢速路径：调用业务状态机 Apply()           │
│                                                             │
│   优势：                                                    │
│   1. Raft 协议响应不受应用速度影响                          │
│   2. 心跳/选举不会因为慢应用而超时                          │
│   3. 可以批量应用提升效率                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

这种设计类似于 **生产者-消费者模式**：
- `run()` 是生产者，生产已提交的日志
- `runApply()` 是消费者，消费并应用这些日志
- `applyc` 是两者之间的缓冲队列

---

## 3. 主循环 (run)

```go
// raft.go:273
func (s *raft) run() {
    for {
        select {
        case <-s.stopc:
            return

        // ★ tick 信号 - 驱动选举/心跳超时
        case <-s.tickc:
            s.raftFsm.tick()
            s.maybeChange(true)

        // ★ 提案 - 来自 Submit()
        case pr := <-s.propc:
            if s.raftFsm.leader != s.config.NodeID {
                pr.future.respond(nil, ErrNotLeader)
                break
            }
            
            // 构造消息，处理第一个提案
            msg := proto.GetMessage()
            msg.Type = proto.LocalMsgProp
            starti := s.raftFsm.raftLog.lastIndex() + 1
            s.pending[starti] = pr.future
            msg.Entries = append(msg.Entries, &proto.Entry{...})

            // ★ 批量获取：尝试再取最多 63 个 (非阻塞)
            for i := 1; i < 64; i++ {
                select {
                case pr := <-s.propc:
                    starti++
                    s.pending[starti] = pr.future
                    msg.Entries = append(msg.Entries, &proto.Entry{...})
                default:
                    break  // 没有更多提案，退出循环
                }
            }
            
            s.raftFsm.Step(msg)  // 一次性提交所有 Entries

        // ★ 网络消息 - 来自其他节点
        case m := <-s.recvc:
            s.raftFsm.Step(m)

        // ... 其他 case
        }
    }
}
```

### 主循环的设计理念

**事件驱动 + 集中处理**：

1. **select 多路复用**：同时监听多个事件源，避免轮询
2. **统一入口**：所有事件都在一个 goroutine 中顺序处理，避免并发问题
3. **批量处理**：每次循环结束调用 `handleReady()`，批量处理状态机的输出

**为什么提案要先检查 Leader？**
```go
case pr := <-s.propc:
    if s.raftFsm.leader != s.config.NodeID {
        pr.future.respond(nil, ErrNotLeader)  // 快速失败
        break
    }
```
- 在 raft 层快速拒绝非 Leader 的提案
- 避免将消息传入 raftFsm 后再拒绝
- 提供更好的错误反馈（可以告诉客户端当前 Leader 是谁）

**handleReady() 的职责：**
- 发送 raftFsm 产生的消息（通过 transport）
- 将已提交的日志发送到 applyc
- 持久化需要持久化的状态
- 更新软状态（leader、term 变化通知）

---

## 4. 应用循环 (runApply)

```go
// raft.go:210
func (s *raft) runApply() {
    for {
        select {
        case <-s.stopc:
            return

        case apply := <-s.applyc:
            // 跳过已应用的
            if apply.index <= s.curApplied.Get() {
                continue
            }

            var resp interface{}
            var err error
            
            switch cmd := apply.command.(type) {
            // 成员变更
            case *proto.ConfChange:
                resp, err = s.raftConfig.StateMachine.ApplyMemberChange(cmd, apply.index)
                
            // 普通命令
            case []byte:
                resp, err = s.raftConfig.StateMachine.Apply(cmd, apply.index)
            }

            // 响应等待的 Future
            if apply.future != nil {
                apply.future.respond(resp, err)
            }
            
            // 更新已应用索引
            s.curApplied.Set(apply.index)
        }
    }
}
```

### 应用循环的设计考量

**为什么要跳过已应用的日志？**
```go
if apply.index <= s.curApplied.Get() {
    continue
}
```
- 快照恢复后，可能会重放已应用的日志
- 保证 **幂等性**：同一条日志不会被应用两次
- `curApplied` 使用原子变量，支持并发读取

**成员变更的特殊处理：**
- 成员变更（ConfChange）走单独的 `ApplyMemberChange` 接口
- 这是因为成员变更需要修改 Raft 的配置，而不仅仅是业务状态
- 典型场景：添加/移除节点、节点地址变更

**Future 响应机制：**
```
业务层 Submit()
    │
    ▼
创建 Future ─── 阻塞等待
    │
    │ (Raft 复制...)
    │
    ▼
runApply 调用 future.respond()
    │
    ▼
业务层收到响应
```
- Future 是一个简单的同步原语（类似 Promise）
- 允许业务层同步等待提案结果
- 也支持超时等待，避免无限阻塞

---

## 5. 提交命令

```go
// raft.go
func (s *raft) propose(cmd []byte, future *Future) {
    pr := pool.getProposal()
    pr.cmdType = proto.EntryNormal
    pr.data = cmd
    pr.future = future
    
    select {
    case s.propc <- pr:  // 发送到 propc 通道
    case <-s.stopc:
        future.respond(nil, ErrStopped)
    }
}
```

### 提交命令的设计

**为什么用 select 而不是直接发送？**
```go
select {
case s.propc <- pr:
case <-s.stopc:
    future.respond(nil, ErrStopped)
}
```
- 防止在 raft 停止后阻塞在 channel 发送上
- 提供优雅的退出机制
- 避免 goroutine 泄漏

**对象池的使用：**
```go
pr := pool.getProposal()
```
- proposal 对象会被频繁创建和销毁
- 使用对象池减少 GC 压力
- 这是高性能 Go 程序的常见优化手段

---

## 6. 数据流图

下图展示了一条命令从提交到应用的完整路径：

```
┌─────────────────────────────────────────────────────────────┐
│                      raft 实例数据流                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   业务层 Submit()                                           │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────┐                                              │
│   │  propc  │ ◄── proposal{cmd, future}                   │
│   └────┬────┘                                              │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                   run() 主循环                       │  │
│   │                                                     │  │
│   │  1. 构造 Entry                                      │  │
│   │  2. raftFsm.Step(LocalMsgProp)                     │  │
│   │  3. raftFsm 追加到 raftLog                          │  │
│   │  4. raftFsm 生成 MsgApp 消息                        │  │
│   │  5. 发送到 Follower                                 │  │
│   └─────────────────────────────────────────────────────┘  │
│        │                                                    │
│        │ 收到 MsgAppResp                                   │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  更新 replicas 进度 → 计算 committed                 │  │
│   │                                                     │  │
│   │  committed > applied → 生成 apply                   │  │
│   └─────────────────────────────────────────────────────┘  │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────┐                                              │
│   │ applyc  │ ◄── apply{index, cmd, future}              │
│   └────┬────┘                                              │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────────────────┐  │
│   │                runApply() 循环                       │  │
│   │                                                     │  │
│   │  StateMachine.Apply(cmd, index)                     │  │
│   │  future.respond(resp, err)                          │  │
│   │  curApplied = index                                 │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 7. 网络消息处理

```go
// 收到网络消息
case m := <-s.recvc:
    // 检查消息来源是否合法
    if _, ok := s.raftFsm.replicas[m.From]; ok || ... {
        switch m.Type {
        case proto.RespMsgAppend:
            // 日志追加响应
            s.raftFsm.Step(m)
            
        case proto.RespMsgVote:
            // 投票响应
            s.raftFsm.Step(m)
            
        default:
            s.raftFsm.Step(m)
        }
    }
```

## 8. tick 处理

### tick 机制的设计

**为什么 Raft 需要 tick？**

Raft 协议依赖两种超时：
- **选举超时**：Follower 长时间收不到 Leader 消息，触发选举
- **心跳超时**：Leader 定期发送心跳，维持 Leadership

实现超时有两种方式：

| 方式 | 优点 | 缺点 |
|------|------|------|
| 真实时间（time.After） | 精确 | 难以测试，每个 Group 需要独立定时器 |
| 逻辑时钟（tick） | 可控、可测试、共享定时器 | 依赖外部驱动 |

tiglabs/raft 选择了 **逻辑时钟**方案：
- RaftServer 有一个共享的 ticker
- 每次 tick 触发时，通知所有 raft 实例
- 各实例内部计数，达到阈值时触发相应动作

```go
// RaftServer 定时触发
func (s *raft) tick() {
    select {
    case s.tickc <- struct{}{}:
    default:  // ★ 非阻塞发送
    }
}

// run() 中处理
case <-s.tickc:
    s.raftFsm.tick()  // 调用状态机的 tick
    s.maybeChange(true)
```

**为什么 tick 发送是非阻塞的？**
```go
select {
case s.tickc <- struct{}{}:
default:  // 如果 channel 满了就丢弃
}
```
- 防止慢的 raft 实例阻塞整个 RaftServer 的定时器
- tick 丢失是可接受的（最多延迟一个 tick 周期）
- tickc 有 64 的缓冲，正常情况不会丢失

## 9. 核心通道职责

| 通道 | 方向 | 职责 |
|------|------|------|
| propc | 输入 | 接收提案（Submit） |
| recvc | 输入 | 接收网络消息 |
| tickc | 输入 | 接收 tick 信号 |
| applyc | 输出 | 发送待应用的日志 |
| snapRecvc | 输入 | 接收快照请求 |
| truncatec | 输入 | 日志截断请求 |
| stopc | 控制 | 停止信号 |
| done | 控制 | 完成信号 |

## 10. 对外方法

raft 实例的主要方法 (raft.go):

### 10.1 生命周期

| 方法 | 位置 | 说明 |
|------|------|------|
| `stop()` | :177 | 停止 raft 实例 |
| `doStop()` | :196 | 执行停止清理 |

### 10.2 核心循环

| 方法 | 位置 | 说明 |
|------|------|------|
| `run()` | :273 | 主循环，处理消息/tick |
| `runApply()` | :210 | 应用循环，执行状态机 |

### 10.3 提案与消息

| 方法 | 位置 | 说明 |
|------|------|------|
| `propose(cmd []byte, future *Future)` | :456 | 提交普通命令 |
| `proposeMemberChange(cc *proto.ConfChange, future *Future)` | :474 | 提交成员变更 |
| `reciveMessage(m *proto.Message)` | :492 | 接收网络消息 |
| `reciveSnapshot(m *snapshotRequest)` | :506 | 接收快照 |

### 10.4 状态查询

| 方法 | 位置 | 说明 |
|------|------|------|
| `status() *Status` | :520 | 获取状态 |
| `isLeader() bool` | :575 | 是否 Leader |
| `leaderTerm() (leader, term uint64)` | :567 | Leader 和 Term |
| `applied() uint64` | :580 | 已应用索引 |
| `committed() uint64` | :584 | 已提交索引 |
| `getPeers() []uint64` | :817 | 获取成员列表 |

### 10.5 其他操作

| 方法 | 位置 | 说明 |
|------|------|------|
| `tick()` | :442 | 触发 tick |
| `truncate(index uint64)` | :539 | 截断日志 |
| `tryToLeader(future *Future)` | :553 | 尝试成为 Leader |
| `readIndex(future *Future)` | :821 | 线性一致性读 |
| `getEntries(future *Future, startIndex, maxSize uint64)` | :834 | 获取日志条目 |

---

## 11. 设计总结

### 核心设计模式

| 模式 | 应用位置 | 效果 |
|------|----------|------|
| Actor 模型 | raft 实例整体 | 通过 channel 通信，避免锁竞争 |
| 生产者-消费者 | run() + runApply() | 协议处理与应用解耦 |
| 状态机 | raftFsm | 清晰的状态转换逻辑 |
| Future/Promise | pending map | 异步提案的同步等待 |
| 对象池 | proposal pool | 减少 GC 压力 |

### 关键设计决策

1. **两个 goroutine 分离**
   - 问题：应用日志可能阻塞协议处理
   - 方案：run() 快速处理协议，runApply() 独立应用
   - 效果：心跳/选举不受应用速度影响

2. **Channel 通信**
   - 问题：多个外部调用可能并发修改状态
   - 方案：所有交互通过 channel，run() 顺序处理
   - 效果：无需加锁，代码更简单

3. **逻辑时钟（tick）**
   - 问题：每个 Raft Group 都需要定时器
   - 方案：RaftServer 共享一个定时器，广播 tick 信号
   - 效果：资源高效，便于测试

4. **非阻塞发送**
   - 问题：慢的实例可能阻塞其他实例
   - 方案：tick 发送使用 select+default
   - 效果：保证系统整体响应性

### 与 etcd/raft 的对比

| 特性 | etcd/raft | tiglabs/raft |
|------|-----------|--------------|
| 应用方式 | 用户轮询 Ready() | 自动 applyc 推送 |
| 消息发送 | 返回给用户发送 | 内部集成 transport |
| 使用复杂度 | 较高（需要自己编排） | 较低（开箱即用） |
| 灵活性 | 高 | 中等 |

tiglabs/raft 是 etcd/raft 的"带电池"版本，封装了更多细节，更易于集成。

---
*创建时间：2026-04-22*
