# DataNode 学习笔记

## 学习目标
理解 DataNode 的架构设计和核心实现，包括数据存储、副本复制、Raft 一致性等。

## 笔记结构

| 层级 | 主题 | 文件 | 状态 |
|------|------|------|------|
| Layer 1 | 启动与配置 | layer1-启动与配置.md | 已完成 |
| Layer 2 | 磁盘与空间管理 | layer2-磁盘与空间管理.md | 已完成 |
| Layer 3 | DataPartition 核心 | layer3-datapartition.md | 已完成 |
| Layer 4 | Extent 存储引擎 | layer4-extent存储.md | 已完成 |
| Layer 5 | 副本复制协议 | layer5-副本复制.md | 已完成 |
| Layer 6 | Raft 一致性 | layer6-raft一致性.md | 已完成 |
| Layer 7 | 数据修复 | layer7-数据修复.md | 已完成 |

## 核心代码文件

```
datanode/
├── server.go              # DataNode 主结构和启动
├── server_handler.go      # HTTP/TCP 请求处理
├── space_manager.go       # 空间管理器
├── disk.go                # 磁盘管理
├── partition.go           # DataPartition 核心
├── partition_raft.go      # Raft 相关
├── partition_raftfsm.go   # Raft 状态机
├── partition_op_by_raft.go # Raft 操作
├── data_partition_repair.go # 数据修复
├── repl/                  # 副本复制协议
│   ├── packet.go
│   └── repl_protocol.go
└── storage/               # 底层存储引擎
    ├── extent.go          # Extent 数据结构
    ├── extent_store.go    # Extent 存储
    └── extent_cache.go    # Extent 缓存
```

## DataNode vs MetaNode 对比

| 维度 | MetaNode | DataNode |
|------|----------|----------|
| 存储内容 | 元数据 (inode/dentry) | 文件数据 (extent) |
| 分区单位 | MetaPartition | DataPartition |
| 存储格式 | B-Tree (内存) + Snapshot | Extent 文件 |
| 一致性 | Raft | Raft + 副本复制 |
| 数据量 | 较小 | 很大 |

---
*创建时间：2026-04-12*
