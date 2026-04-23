# Layer 4: Raft 状态机 (raftFsm)

## 1. raftFsm 结构

raftFsm 是 Raft 协议的核心状态机实现。

```go
// raft_fsm.go:49
type raftFsm struct {
    id               uint64              // Raft Group ID
    term             uint64              // 当前任期
    vote             uint64              // 投票给谁
    leader           uint64              // 当前 Leader
    electionElapsed  int                 // 选举计时器
    heartbeatElapsed int                 // 心跳计时器
    randElectionTick int                 // 随机选举超时
    pendingConf      bool                // 是否有待处理的配置变更
    
    state    fsmState                    // 当前状态
    sm       StateMachine                // 业务状态机
    config   *Config
    raftLog  *raftLog                    // Raft 日志
    rand     *rand.Rand                  // 随机数生成器
    votes    map[uint64]bool             // 投票结果
    acks     map[uint64]bool             // 确认结果
    replicas map[uint64]*replica         // 副本进度
    readOnly *readOnly                   // 只读请求
    msgs     []*proto.Message            // 待发送消息
    
    step stepFunc                        // ★ 状态处理函数
    tick func()                          // ★ tick 处理函数
}
```

## 2. 状态定义

```go
// raft_fsm_state.go
type fsmState int

const (
    stateFollower  fsmState = 0
    stateCandidate fsmState = 1
    stateLeader    fsmState = 2
)
```

## 3. 状态转换

```
┌─────────────────────────────────────────────────────────────┐
│                      状态转换图                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    ┌─────────────┐                          │
│                    │  启动/恢复   │                          │
│                    └──────┬──────┘                          │
│                           │                                 │
│                           ▼                                 │
│                    ┌─────────────┐                          │
│           ┌───────│  Follower   │◄────────┐                │
│           │        └──────┬──────┘         │                │
│           │               │                │                │
│           │   选举超时    │    收到更高    │                │
│           │   无 Leader   │    term 消息   │                │
│           │               ▼                │                │
│           │        ┌─────────────┐         │                │
│           │        │  Candidate  │─────────┤                │
│           │        └──────┬──────┘         │                │
│           │               │                │                │
│           │   获得多数票  │                │                │
│           │               ▼                │                │
│           │        ┌─────────────┐         │                │
│           └───────│   Leader    │──────────┘                │
│       选举超时     └─────────────┘   发现更高 term           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 三种状态的 step 函数

### 4.1 Follower 状态

```go
// raft_fsm_follower.go:27
func (r *raftFsm) becomeFollower(term, lead uint64) {
    r.step = stepFollower       // 设置处理函数
    r.reset(term, 0, false)
    r.tick = r.tickElection     // 选举超时 tick
    r.leader = lead
    r.state = stateFollower
}

// raft_fsm_follower.go:38
func stepFollower(r *raftFsm, m *proto.Message) {
    switch m.Type {
    case proto.LocalMsgProp:
        // 转发给 Leader
        m.To = r.leader
        r.send(m)

    case proto.ReqMsgAppend:
        // 处理日志追加
        r.electionElapsed = 0    // 重置选举计时
        r.leader = m.From
        r.handleAppendEntries(m)

    case proto.ReqMsgHeartBeat:
        // 处理心跳
        r.electionElapsed = 0
        r.leader = m.From

    case proto.ReqMsgVote:
        // 处理投票请求
        r.handleVote(m)
    }
}
```

### 4.2 Candidate 状态

```go
// raft_fsm_candidate.go:26
func (r *raftFsm) becomeCandidate() {
    r.step = stepCandidate      // 设置处理函数
    r.reset(r.term+1, 0, false) // term + 1
    r.tick = r.tickElection
    r.vote = r.config.NodeID    // 投票给自己
    r.state = stateCandidate
}

// raft_fsm_candidate.go:42
func stepCandidate(r *raftFsm, m *proto.Message) {
    switch m.Type {
    case proto.LocalMsgProp:
        // 丢弃提案
        return

    case proto.ReqMsgAppend:
        // 收到 Leader 的追加消息，变为 Follower
        r.becomeFollower(r.term, m.From)
        r.handleAppendEntries(m)

    case proto.ReqMsgHeartBeat:
        // 收到心跳，变为 Follower
        r.becomeFollower(r.term, m.From)

    case proto.RespMsgVote:
        // 收到投票响应
        r.handleVoteResp(m)
    }
}
```

### 4.3 Leader 状态

```go
// raft_fsm_leader.go:28
func (r *raftFsm) becomeLeader() {
    r.step = stepLeader         // 设置处理函数
    r.reset(r.term, lasti, true)
    r.tick = r.tickHeartbeat    // 心跳 tick
    r.leader = r.config.NodeID
    r.state = stateLeader
    
    // 追加空日志确认 Leadership
    r.appendEntry(&proto.Entry{Term: r.term, Index: lasti + 1, Data: nil})
}

// raft_fsm_leader.go:65
func stepLeader(r *raftFsm, m *proto.Message) {
    switch m.Type {
    case proto.LocalMsgProp:
        // 处理提案
        r.appendEntry(m.Entries...)  // 追加到日志
        r.bcastAppend()              // 广播给 Follower

    case proto.RespMsgAppend:
        // 处理追加响应
        r.handleAppendResp(m)

    case proto.RespMsgHeartBeat:
        // 处理心跳响应
        r.handleHeartbeatResp(m)
    }
}
```

## 5. tick 机制

```go
// raft_fsm_follower.go:133
// Follower/Candidate: 选举超时
func (r *raftFsm) tickElection() {
    r.electionElapsed++
    
    // 超时触发选举
    if r.electionElapsed >= r.randElectionTick {
        r.electionElapsed = 0
        r.Step(&proto.Message{Type: proto.LocalMsgHup})
    }
}

// raft_fsm_leader.go:395
// Leader: 心跳超时
func (r *raftFsm) tickHeartbeat() {
    r.heartbeatElapsed++
    
    // 超时发送心跳
    if r.heartbeatElapsed >= r.config.HeartbeatTick {
        r.heartbeatElapsed = 0
        r.Step(&proto.Message{Type: proto.LocalMsgBeat})
    }
}
```

## 6. Step 主函数

```go
// raft_fsm.go:196
func (r *raftFsm) Step(m *proto.Message) {
    // 1. 处理 Hup 消息（触发选举）
    if m.Type == proto.LocalMsgHup {
        if r.state != stateLeader && r.promotable() {
            r.campaign(campaignElection)
        }
        return
    }

    // 2. 处理 term 变化
    switch {
    case m.Term == 0:
        // 本地消息
    case m.Term > r.term:
        // 收到更高 term，变为 Follower
        r.becomeFollower(m.Term, m.From)
    case m.Term < r.term:
        // 忽略低 term 消息
        return
    }

    // 3. 调用当前状态的 step 函数
    r.step(r, m)
}
```

## 7. 日志追加处理

```go
// raft_fsm_follower.go:156
func (r *raftFsm) handleAppendEntries(m *proto.Message) {
    // 1. 检查日志是否匹配
    if m.Index < r.raftLog.committed {
        // 已提交，返回 committed
        r.send(&proto.Message{
            Type:  proto.RespMsgAppend,
            To:    m.From,
            Index: r.raftLog.committed,
        })
        return
    }

    // 2. 尝试追加日志
    if mlastIndex, ok := r.raftLog.maybeAppend(m.Index, m.LogTerm, m.Commit, m.Entries...); ok {
        // 追加成功
        r.send(&proto.Message{
            Type:  proto.RespMsgAppend,
            To:    m.From,
            Index: mlastIndex,
        })
    } else {
        // 追加失败，返回 reject
        r.send(&proto.Message{
            Type:   proto.RespMsgAppend,
            To:     m.From,
            Index:  m.Index,
            Reject: true,
        })
    }
}
```

## 8. 投票处理

```go
func (r *raftFsm) handleVote(m *proto.Message) {
    // 1. 检查是否已投票
    canVote := r.vote == m.From || 
               (r.vote == NoLeader && r.leader == NoLeader)
    
    // 2. 检查日志是否足够新
    if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
        r.vote = m.From
        r.electionElapsed = 0
        r.send(&proto.Message{
            Type: proto.RespMsgVote,
            To:   m.From,
        })
    } else {
        r.send(&proto.Message{
            Type:   proto.RespMsgVote,
            To:     m.From,
            Reject: true,
        })
    }
}
```

## 9. 广播追加

```go
// raft_fsm_leader.go:480
// Leader 广播日志到所有 Follower
func (r *raftFsm) bcastAppend() {
    for id := range r.replicas {
        if id == r.config.NodeID {
            continue
        }
        r.sendAppend(id)
    }
}

func (r *raftFsm) sendAppend(to uint64) {
    pr := r.replicas[to]
    
    m := proto.GetMessage()
    m.Type = proto.ReqMsgAppend
    m.To = to
    m.Index = pr.next - 1
    m.LogTerm = r.raftLog.term(pr.next - 1)
    m.Entries = r.raftLog.entries(pr.next, maxMsgSize)
    m.Commit = r.raftLog.committed
    
    r.send(m)
}
```

## 10. 消息类型汇总

| 消息类型 | 方向 | 说明 |
|----------|------|------|
| LocalMsgHup | 本地 | 触发选举 |
| LocalMsgBeat | 本地 | 触发心跳 |
| LocalMsgProp | 本地 | 提案 |
| ReqMsgVote | C→All | 投票请求 |
| RespMsgVote | All→C | 投票响应 |
| ReqMsgAppend | L→F | 日志追加请求 |
| RespMsgAppend | F→L | 日志追加响应 |
| ReqMsgHeartBeat | L→F | 心跳请求 |
| RespMsgHeartBeat | F→L | 心跳响应 |
| ReqMsgSnap | L→F | 快照请求 |

## 11. 核心方法

raftFsm 的主要方法:

### 11.1 状态转换

| 方法 | 文件:行号 | 说明 |
|------|-----------|------|
| `becomeFollower(term, lead uint64)` | raft_fsm_follower.go:27 | 转为 Follower |
| `becomeCandidate()` | raft_fsm_candidate.go:26 | 转为 Candidate |
| `becomePreCandidate()` | raft_fsm_leader.go:317 | 转为 PreCandidate |
| `becomeLeader()` | raft_fsm_leader.go:28 | 转为 Leader |

### 11.2 消息处理

| 方法 | 文件:行号 | 说明 |
|------|-----------|------|
| `Step(m *proto.Message)` | raft_fsm.go:196 | 消息入口 |
| `handleAppendEntries(m *proto.Message)` | raft_fsm_follower.go:156 | 处理日志追加 |
| `campaign(force bool, t CampaignType)` | raft_fsm_candidate.go:97 | 发起选举 |
| `poll(id uint64, v bool) int` | raft_fsm_candidate.go:139 | 统计投票 |

### 11.3 Leader 操作

| 方法 | 文件:行号 | 说明 |
|------|-----------|------|
| `bcastAppend()` | raft_fsm_leader.go:480 | 广播日志 |
| `sendAppend(to uint64)` | raft_fsm_leader.go:489 | 发送日志到指定节点 |
| `appendEntry(es ...*proto.Entry)` | raft_fsm_leader.go:566 | 追加日志条目 |
| `maybeCommit() bool` | raft_fsm_leader.go:458 | 尝试提交 |
| `bcastReadOnly()` | raft_fsm_leader.go:572 | 广播只读请求 |

### 11.4 Tick 处理

| 方法 | 文件:行号 | 说明 |
|------|-----------|------|
| `tickElection()` | raft_fsm_follower.go:133 | 选举超时 tick |
| `tickHeartbeat()` | raft_fsm_leader.go:395 | 心跳超时 tick |
| `tickElectionAck()` | raft_fsm_leader.go:427 | 选举确认 tick |

### 11.5 成员管理

| 方法 | 文件:行号 | 说明 |
|------|-----------|------|
| `applyConfChange(cc *proto.ConfChange) bool` | raft_fsm.go:426 | 应用配置变更 |
| `addPeer(peer proto.Peer)` | raft_fsm.go:443 | 添加节点 |
| `removePeer(peer proto.Peer) bool` | raft_fsm.go:455 | 移除节点 |
| `updatePeer(peer proto.Peer)` | raft_fsm.go:484 | 更新节点 |
| `quorum() int` | raft_fsm.go:491 | 获取多数派数量 |
| `peers() []proto.Peer` | raft_fsm.go:546 | 获取成员列表 |

### 11.6 其他

| 方法 | 文件:行号 | 说明 |
|------|-----------|------|
| `send(m *proto.Message)` | raft_fsm.go:495 | 发送消息 |
| `reset(term, lasti uint64, isLeader bool)` | raft_fsm.go:505 | 重置状态 |
| `restore(meta proto.SnapshotMeta)` | raft_fsm.go:565 | 恢复快照 |
| `promotable() bool` | raft_fsm_follower.go:207 | 是否可提升 |
| `checkLeaderLease() bool` | raft_fsm_leader.go:439 | 检查租约 |

---
*创建时间: 2026-04-22*
