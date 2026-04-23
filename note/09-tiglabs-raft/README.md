# tiglabs/raft 库学习笔记

## 简介

tiglabs/raft 是 CubeFS 使用的 Multi-Raft 共识库，基于 etcd/raft 修改而来。它提供了多个 Raft Group 共存于单一进程的能力，是 CubeFS MetaNode 和 Master 实现强一致性的基础。

## 从这里开始

**如果你是初学者**，请先阅读：

> **[00-初学者指南.md](00-初学者指南.md)** - 自上而下的架构讲解，适合入门

## 库位置

```
depends/tiglabs/raft/
```

## 学习目标

理解 Raft 协议在 tiglabs 库中的实现，包括：
- Multi-Raft 架构设计
- Leader 选举机制
- 日志复制流程
- 快照机制
- 网络传输层

## 笔记结构

| 层级 | 主题 | 文件 | 说明 |
|------|------|------|------|
| **入门** | **初学者指南** | **[00-初学者指南.md](00-初学者指南.md)** | **自上而下的架构讲解** |
| Layer 1 | 整体架构 | [layer1-整体架构.md](layer1-整体架构.md) | 目录结构、核心概念 |
| Layer 2 | RaftServer | [layer2-raftserver.md](layer2-raftserver.md) | Multi-Raft 管理 |
| Layer 3 | Raft 实例 | [layer3-raft实例.md](layer3-raft实例.md) | 单个 Raft Group 运行时 |
| Layer 4 | Raft 状态机 | [layer4-raft状态机.md](layer4-raft状态机.md) | 协议核心实现 |
| Layer 5 | 日志管理 | [layer5-日志管理.md](layer5-日志管理.md) | raftLog、unstable |
| Layer 6 | 网络传输 | [layer6-网络传输.md](layer6-网络传输.md) | 双端口设计 |
| Layer 7 | WAL 存储 | [layer7-wal存储.md](layer7-wal存储.md) | 持久化实现 |

## 核心代码文件

```
depends/tiglabs/raft/
├── server.go              # RaftServer - Multi-Raft 管理
├── raft.go                # raft 实例 - 单个 Raft Group
├── raft_fsm.go            # raftFsm - Raft 状态机核心
├── raft_fsm_leader.go     # Leader 状态处理
├── raft_fsm_follower.go   # Follower 状态处理
├── raft_fsm_candidate.go  # Candidate 状态处理
├── raft_log.go            # Raft 日志管理
├── raft_log_unstable.go   # 未持久化日志
├── raft_replica.go        # 副本管理
├── raft_snapshot.go       # 快照处理
├── transport.go           # 网络传输抽象
├── transport_heartbeat.go # 心跳传输
├── transport_replicate.go # 日志复制传输
├── statemachine.go        # StateMachine 接口定义
├── config.go              # 配置
├── proto/                 # 协议定义
│   └── proto.go           # Message, Entry 等结构
└── storage/
    └── wal/               # WAL 存储实现
        ├── storage.go     # Storage 接口
        └── log_storage.go # 日志存储
```

## 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    tiglabs/raft 架构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                   RaftServer                          │  │
│  │                  (Multi-Raft 管理)                     │  │
│  │                                                       │  │
│  │    rafts map[uint64]*raft                            │  │
│  │    ┌─────────┐  ┌─────────┐  ┌─────────┐            │  │
│  │    │ raft 1  │  │ raft 2  │  │ raft 3  │  ...       │  │
│  │    └────┬────┘  └────┬────┘  └────┬────┘            │  │
│  └─────────┼────────────┼────────────┼───────────────────┘  │
│            │            │            │                      │
│            ▼            ▼            ▼                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    raftFsm                          │    │
│  │              (Raft 协议状态机)                       │    │
│  │                                                     │    │
│  │   state: Leader / Follower / Candidate              │    │
│  │   term, vote, leader                                │    │
│  │   raftLog (日志)                                    │    │
│  │   replicas (副本)                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│            │                                                │
│            ▼                                                │
│  ┌───────────────────┐    ┌───────────────────┐            │
│  │    Transport      │    │    WAL Storage    │            │
│  │   (网络传输)       │    │   (持久化存储)     │            │
│  └───────────────────┘    └───────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 与 CubeFS 的关系

```
┌─────────────────────────────────────────────────────────────┐
│                      调用关系                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CubeFS metanode/master                                     │
│         │                                                   │
│         │ 实现 StateMachine 接口                            │
│         │ 调用 RaftStore API                                │
│         ▼                                                   │
│  raftstore (CubeFS 封装层)                                  │
│         │                                                   │
│         │ 封装 tiglabs API                                  │
│         ▼                                                   │
│  tiglabs/raft (Raft 实现)                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---
*创建时间：2026-04-22*
