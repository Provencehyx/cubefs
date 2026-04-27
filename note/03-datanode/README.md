# DataNode 学习笔记

> **推荐起点**：[00-初学者指南.md](00-初学者指南.md)

## 设计概述

DataNode 的核心设计决策：
1. **数据与元数据分离** — 独立扩展、独立优化
2. **Extent 双轨制** — TinyExtent 解决小文件问题，NormalExtent 处理大文件
3. **链式复制替代 Raft** — 数据复制追求吞吐，Raft 只用于成员变更和随机写

## 笔记结构

| 层级 | 主题 | 核心问题 |
|------|------|----------|
| [Layer 1](layer1-启动与配置.md) | 启动与配置 | 为什么要先注册再加载分区？启动顺序有什么讲究？ |
| [Layer 2](layer2-磁盘与空间管理.md) | 磁盘与空间管理 | SpaceManager 如何平衡磁盘负载？磁盘故障如何检测？ |
| [Layer 3](layer3-datapartition.md) | DataPartition | 为什么分区大小是 120GB？分区元数据如何持久化？ |
| [Layer 4](layer4-extent存储.md) | Extent 存储 | TinyExtent 为什么只有 64 个？小文件删除为什么用 punch hole？ |
| [Layer 5](layer5-副本复制.md) | 链式复制 | 为什么不用 Raft 复制数据？链式复制如何保证一致性？ |
| [Layer 6](layer6-raft一致性.md) | Raft 一致性 | 哪些操作必须走 Raft？为什么随机写需要 Raft？ |
| [Layer 7](layer7-数据修复.md) | 数据修复 | 副本不一致如何检测？如何选择修复源？ |

## 核心设计决策

| 问题 | 决策 | 原因 |
|------|------|------|
| 数据量大，如何复制？ | 链式复制（非 Raft） | 带宽效率高 |
| 小文件存储效率？ | TinyExtent 共享 | 减少文件数 |
| 覆盖写一致性？ | 随机写走 Raft | 需要全局顺序 |
| 分区大小？ | 120GB | 故障恢复时间平衡 |

## 核心代码文件

```
datanode/
├── server.go              # DataNode 启动
├── space_manager.go       # 磁盘管理、Straw 算法
├── disk.go                # 磁盘状态、QoS 限流
├── partition.go           # DataPartition 核心
├── partition_raft.go      # Raft 启动
├── partition_raftfsm.go   # Raft Apply (随机写)
├── data_partition_repair.go # 副本修复
├── repl/                  # 链式复制协议
│   └── repl_protocol.go
└── storage/               # Extent 存储引擎
    ├── extent.go          # Extent 文件操作
    └── extent_store.go    # TinyExtent/NormalExtent 管理
```

---
*更新时间：2026-04-27*
