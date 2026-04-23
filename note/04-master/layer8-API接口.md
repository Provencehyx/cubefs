# Layer 8: Master API 接口

## 1. API 总体设计

### 1.1 按调用者分类

```
┌─────────────────────────────────────────────────────────────┐
│                    Master API 分类                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  【客户端调用】/client/*                                     │
│   ├── /client/partitions     → 定位数据在哪（最频繁）        │
│   ├── /client/metaPartitions → 定位元数据在哪               │
│   └── /client/vol            → 获取卷信息                   │
│                                                             │
│  【运维管理】/admin/*                                        │
│   ├── 卷管理：createVol, deleteVol, updateVol               │
│   ├── 分区管理：createDP, decommissionDP, diagnoseDP        │
│   └── 集群管理：clusterFreeze, setConfig                    │
│                                                             │
│  【节点上报】/dataNode/*, /metaNode/*                        │
│   ├── add        → 节点注册                                 │
│   ├── response   → 任务结果上报                             │
│   └── heartbeat  → 心跳（通过 response 隐式实现）            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 请求处理流程

```
HTTP 请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                   Middleware 中间件                          │
├─────────────────────────────────────────────────────────────┤
│  1. 是否 Leader？                                           │
│     ├── 是 Leader → 继续处理                                │
│     ├── 是 FollowerRead 请求 → 返回缓存                     │
│     └── 否 → 转发给 Leader（ReverseProxy）                  │
│                                                             │
│  2. API 限流检查                                            │
│     └── 超限 → 返回 429 Too Many Requests                   │
│                                                             │
│  3. 认证检查（如果启用）                                     │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
业务 Handler 处理
```

---

## 2. 客户端 API（/client/*）

这类 API 由 CubeFS 客户端（FUSE/SDK）调用，是**最频繁**的请求。

| API | 路径 | 说明 | FollowerRead |
|-----|------|------|--------------|
| 获取数据分区 | `/client/partitions` | 返回卷的所有 DP 位置 | ✓ 支持 |
| 获取元数据分区 | `/client/metaPartitions` | 返回卷的所有 MP 位置 | ✓ 支持 |
| 获取卷信息 | `/client/vol` | 返回卷的基本信息 | ✗ |
| 获取卷统计 | `/client/volStat` | 返回卷的使用量统计 | ✗ |

### 2.1 /client/partitions（最重要）

**用途**：客户端读写文件前，需要知道数据存在哪些 DataNode。

```
请求：GET /client/partitions?name=my-vol

响应：
{
  "code": 0,
  "data": {
    "volName": "my-vol",
    "dataPartitions": [
      {
        "PartitionID": 1,
        "Status": 2,                    // 2=ReadWrite
        "ReplicaNum": 3,
        "Hosts": ["10.0.0.1:17310", "10.0.0.2:17310", "10.0.0.3:17310"],
        "LeaderAddr": "10.0.0.1:17310"
      },
      // ... 更多分区
    ]
  }
}
```

**为什么支持 FollowerRead？**
- 这是最频繁的请求，全走 Leader 会成为瓶颈
- 分区列表变化不频繁，秒级过期可接受
- 客户端有本地缓存，不会每次都请求

---

## 3. 卷管理 API（/admin/vol*）

| API | 路径 | 说明 |
|-----|------|------|
| 创建卷 | `/admin/createVol` | 创建新卷，初始化分区 |
| 删除卷 | `/admin/deleteVol` | 标记删除（延迟删除） |
| 更新卷 | `/admin/updateVol` | 更新配置（副本数、配额等） |
| 获取卷 | `/admin/getVol` | 获取卷详细信息 |
| 卷列表 | `/admin/listVols` | 列出所有卷 |
| 卷扩容 | `/admin/volExpand` | 增加卷配额 |
| 卷缩容 | `/admin/volShrink` | 减少卷配额 |
| 禁用卷 | `/admin/volForbidden` | 禁止读写 |

### 3.1 创建卷示例

```
请求：POST /admin/createVol
参数：
  - name: my-vol           # 卷名
  - owner: admin           # 所有者
  - capacity: 100          # 容量(GB)
  - replicaNum: 3          # 副本数
  - zoneName: zone1        # 可用区（可选）

响应：
{
  "code": 0,
  "msg": "success",
  "data": {
    "Name": "my-vol",
    "Owner": "admin",
    "Status": 0,
    "TotalSize": 107374182400,
    "DpReplicaNum": 3,
    "MpReplicaNum": 3
  }
}
```

---

## 4. 数据分区 API（DataPartition）

| API | 路径 | 说明 |
|-----|------|------|
| 创建 DP | `/admin/createDataPartition` | 手动创建分区 |
| 获取 DP | `/admin/getDataPartition` | 获取分区详情 |
| 加载 DP | `/admin/loadDataPartition` | 重新加载分区信息 |
| 下线 DP | `/admin/decommissionDataPartition` | 迁移分区到其他节点 |
| 诊断 DP | `/admin/diagnoseDataPartition` | 检查分区健康状态 |
| 添加副本 | `/admin/addDataReplica` | 添加副本 |
| 删除副本 | `/admin/deleteDataReplica` | 删除副本 |

---

## 5. 元数据分区 API（MetaPartition）

| API | 路径 | 说明 |
|-----|------|------|
| 创建 MP | `/admin/createMetaPartition` | 手动创建分区 |
| 加载 MP | `/admin/loadMetaPartition` | 重新加载分区 |
| 下线 MP | `/admin/decommissionMetaPartition` | 迁移分区 |
| 诊断 MP | `/admin/diagnoseMetaPartition` | 检查健康状态 |
| 切换 Leader | `/admin/changeMetaPartitionLeader` | 手动切换 Leader |

---

## 6. 节点管理 API

### 6.1 DataNode

| API | 路径 | 说明 |
|-----|------|------|
| 注册 | `/dataNode/add` | DataNode 启动时注册 |
| 下线 | `/dataNode/decommission` | 下线节点，迁移数据 |
| 获取信息 | `/dataNode/get` | 获取节点详情 |
| 获取所有 | `/admin/getClusterDataNodes` | 获取所有 DataNode |
| 任务响应 | `/dataNode/response` | 节点上报任务结果 |

### 6.2 MetaNode

| API | 路径 | 说明 |
|-----|------|------|
| 注册 | `/metaNode/add` | MetaNode 启动时注册 |
| 下线 | `/metaNode/decommission` | 下线节点 |
| 迁移 | `/metaNode/migrate` | 迁移节点上的分区 |
| 获取信息 | `/metaNode/get` | 获取节点详情 |
| 获取所有 | `/admin/getClusterMetaNodes` | 获取所有 MetaNode |

---

## 7. 集群管理 API

| API | 路径 | 说明 |
|-----|------|------|
| 集群状态 | `/admin/getCluster` | 获取集群统计信息 |
| 冻结集群 | `/admin/clusterFreeze` | 禁止自动创建分区 |
| 设置配置 | `/admin/setConfig` | 动态更新配置 |
| 获取配置 | `/admin/getConfig` | 获取当前配置 |
| Raft 状态 | `/raftStatus` | 查看 Raft 复制状态 |
| 切换 Leader | `/admin/changeMasterLeader` | 手动切换 Master Leader |
| 添加 Master | `/admin/addRaftNode` | 添加 Master 节点 |
| 移除 Master | `/admin/removeRaftNode` | 移除 Master 节点 |
| 拓扑信息 | `/topo/get` | 获取 Zone/NodeSet 拓扑 |

---

## 8. QoS 限流 API

| API | 路径 | 说明 |
|-----|------|------|
| QoS 上报 | `/qos/upload` | 客户端上报 IO 统计 |
| QoS 状态 | `/qos/getStatus` | 获取限流状态 |
| 更新 QoS | `/qos/update` | 更新限流配置 |
| Zone 限流 | `/qos/updateZoneLimit` | 设置 Zone 级别限流 |

---

## 9. 用户管理 API

| API | 路径 | 说明 |
|-----|------|------|
| 创建用户 | `/user/create` | 创建用户 |
| 删除用户 | `/user/delete` | 删除用户 |
| 更新用户 | `/user/update` | 更新用户信息 |
| 用户列表 | `/user/list` | 获取所有用户 |
| 获取用户 | `/user/info` | 获取用户详情 |

---

## 10. 代码位置

| 文件 | 说明 |
|------|------|
| `http_server.go` | 路由注册（registerAPIRoutes） |
| `api_service.go` | API Handler 实现（主要） |
| `api_service_user.go` | 用户相关 API |
| `api_service_balance.go` | 负载均衡相关 API |
| `api_args_parse.go` | 参数解析 |

---

## 11. 关键设计点

### 11.1 FollowerRead 机制

```go
// http_server.go:71
func (m *Server) isFollowerRead(r *http.Request) bool {
    // 只有特定路径允许 Follower 处理
    if r.URL.Path == proto.ClientDataPartitions {
        return m.cluster.followerReadManager.IsVolViewReady(volName)
    }
    return false
}
```

### 11.2 Leader 转发

```go
// http_server.go:150
if !m.partition.IsRaftLeader() && !isFollowerRead {
    // 转发给 Leader
    m.reverseProxy.ServeHTTP(w, r)
    return
}
```

### 11.3 API 限流

```go
// http_server.go:103
if err := m.cluster.apiLimiter.Wait(r.URL.Path); err != nil {
    http.Error(w, "too many requests", 429)
    return
}
```

---

*更新时间: 2026-04-23*
