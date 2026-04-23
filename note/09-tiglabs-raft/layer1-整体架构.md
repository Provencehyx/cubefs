# Layer 1: 整体架构

## 1. tiglabs/raft 简介

tiglabs/raft 是一个基于 etcd/raft 修改的 Multi-Raft 库，主要特点：
- 支持单进程内多个 Raft Group
- 共享网络传输层
- 共享定时器
- 独立的状态机和存储

## 2. 目录结构

```
depends/tiglabs/raft/
├── server.go              # RaftServer 入口
├── raft.go                # 单个 raft 实例
├── raft_fsm.go            # Raft 协议状态机
├── raft_fsm_leader.go     # Leader 逻辑
├── raft_fsm_follower.go   # Follower 逻辑
├── raft_fsm_candidate.go  # Candidate 逻辑
├── raft_fsm_state.go      # 状态定义
├── raft_log.go            # 日志管理
├── raft_log_unstable.go   # 内存中未持久化日志
├── raft_replica.go        # 副本进度管理
├── raft_snapshot.go       # 快照处理
├── transport.go           # 网络传输接口
├── transport_heartbeat.go # 心跳网络
├── transport_replicate.go # 复制网络
├── transport_sender.go    # 发送器
├── statemachine.go        # 业务层接口
├── config.go              # 配置
├── future.go              # 异步结果
├── proto/
│   ├── proto.go           # 消息定义
│   └── codec.go           # 编解码
└── storage/
    ├── storage.go         # 存储接口
    └── wal/               # WAL 实现
```

## 3. 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                     核心概念关系                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RaftServer (1个)                                           │
│      │                                                      │
│      ├── rafts map ─┬── raft (Raft Group 1)                │
│      │              ├── raft (Raft Group 2)                │
│      │              └── raft (Raft Group N)                │
│      │                    │                                 │
│      │                    └── raftFsm (协议状态机)           │
│      │                          │                           │
│      │                          ├── state (Leader/...)     │
│      │                          ├── raftLog (日志)          │
│      │                          ├── replicas (副本)         │
│      │                          └── sm (业务状态机)         │
│      │                                                      │
│      └── transport (共享网络)                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 核心结构体

### 4.1 RaftServer

```go
// server.go:31
type RaftServer struct {
    config *Config
    ticker *time.Ticker          // 共享定时器
    heartc chan *proto.Message   // 心跳通道
    stopc  chan struct{}
    mu     sync.RWMutex
    rafts  map[uint64]*raft      // ★ 多个 Raft Group
}
```

### 4.2 raft (单个实例)

```go
// raft.go:99
type raft struct {
    raftFsm    *raftFsm          // Raft 协议状态机
    config     *Config
    raftConfig *RaftConfig
    
    // 状态
    curApplied  util.AtomicUInt64
    peerState   peerState
    pending     map[uint64]*Future
    
    // 通道
    propc     chan *proposal      // 提案
    applyc    chan *apply         // 应用
    recvc     chan *proto.Message // 接收消息
    snapRecvc chan *snapshotRequest
    truncatec chan uint64
    tickc     chan struct{}       // tick 信号
    stopc     chan struct{}
}
```

### 4.3 raftFsm (协议状态机)

```go
// raft_fsm.go:49
type raftFsm struct {
    id               uint64
    term             uint64        // 任期
    vote             uint64        // 投票给谁
    leader           uint64        // 当前 Leader
    electionElapsed  int           // 选举计时
    heartbeatElapsed int           // 心跳计时
    
    state    fsmState              // Leader/Follower/Candidate
    sm       StateMachine          // 业务状态机
    config   *Config
    raftLog  *raftLog              // 日志
    replicas map[uint64]*replica   // 副本进度
    
    step stepFunc                  // 状态处理函数
    tick func()                    // tick 处理函数
    msgs []*proto.Message          // 待发送消息
}
```

## 5. 状态转换

```
                    超时/收到更高 term
           ┌──────────────────────────────┐
           │                              │
           ▼                              │
    ┌─────────────┐    超时发起选举    ┌─────────────┐
    │  Follower   │ ─────────────────► │  Candidate  │
    │             │                    │             │
    └─────────────┘                    └─────────────┘
           ▲                              │
           │                              │ 获得多数票
           │         收到更高 term        │
           │   ◄─────────────────────     │
           │                              ▼
           │                        ┌─────────────┐
           └─────────────────────── │   Leader    │
                  发现更高 term      │             │
                                    └─────────────┘
```

## 6. 消息类型

```go
// proto/proto.go
const (
    MsgHup            MsgType = 1   // 触发选举
    MsgBeat           MsgType = 2   // 触发心跳
    MsgProp           MsgType = 3   // 提案
    MsgApp            MsgType = 4   // 日志追加
    MsgAppResp        MsgType = 5   // 追加响应
    MsgVote           MsgType = 6   // 投票请求
    MsgVoteResp       MsgType = 7   // 投票响应
    MsgSnap           MsgType = 8   // 快照
    MsgHeartbeat      MsgType = 9   // 心跳
    MsgHeartbeatResp  MsgType = 10  // 心跳响应
    // ...
)
```

## 7. 调用流程概览

```
业务层 (MetaPartition)
        │
        │ Submit(cmd)
        ▼
┌───────────────────────────────────────────────────────────┐
│                     raft 实例                             │
│                                                           │
│  propc ◄── proposal{cmd}                                 │
│    │                                                      │
│    ▼                                                      │
│  raftFsm.step() ── 追加到 raftLog                        │
│    │                                                      │
│    ├── 生成 MsgApp 消息                                   │
│    │                                                      │
│    ▼                                                      │
│  transport.Send() ─── 发送到 Follower                    │
│                                                           │
│  收到 MsgAppResp ─── 更新 replicas 进度                   │
│    │                                                      │
│    ├── 多数确认 → commit                                  │
│    │                                                      │
│    ▼                                                      │
│  applyc ◄── apply{cmd, index}                            │
│    │                                                      │
│    ▼                                                      │
│  sm.Apply(cmd) ─── 回调业务状态机                         │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

## 8. 关键文件速查

| 文件 | 职责 |
|------|------|
| server.go | RaftServer，Multi-Raft 管理 |
| raft.go | raft 实例，消息分发 |
| raft_fsm.go | Raft 协议核心状态机 |
| raft_fsm_leader.go | Leader 状态下的处理 |
| raft_fsm_follower.go | Follower 状态下的处理 |
| raft_fsm_candidate.go | Candidate 状态下的处理 |
| raft_log.go | 日志管理 |
| transport.go | 网络传输 |
| statemachine.go | 业务接口定义 |

---
*创建时间：2026-04-22*
