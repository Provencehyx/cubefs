# MetaNode 七层解析 · Layer 1：启动与配置

> 入口文件：[metanode/metanode.go](../../metanode/metanode.go)
> 核心问题：**MetaNode 进程是怎么从一个空对象，一步步把自己变成一个能对外提供元数据服务的节点的？**

---

## 1. 顶层入口：`Start` / `Shutdown`

`MetaNode` 是 CubeFS 元数据节点的顶层结构体，定义在 [metanode.go:63](../../metanode/metanode.go#L63)。
它的生命周期被三个方法包住：

| 方法 | 作用 | 实现 |
|---|---|---|
| `Start(cfg)` | 启动 | 委托给 `m.control.Start(m, cfg, doStart)` |
| `Shutdown()` | 停止 | 委托给 `m.control.Shutdown(m, doShutdown)` |
| `Sync()` | 阻塞主 goroutine 等待退出 | `m.control.Sync()` |

`control` 来自 `cmd/common`，是 CubeFS 各个角色（master/datanode/metanode/objectnode）共用的服务控制框架——**所有节点都套这个壳**。
真正干活的是 [doStart](../../metanode/metanode.go#L146) 这个函数。

---

## 2. `doStart` 启动流水线（最关键）

[metanode.go:146-191](../../metanode/metanode.go#L146-L191) 把 MetaNode 的启动拆成 9 个有序步骤：

```go
func doStart(s common.Server, cfg *config.Config) (err error) {
    1. m.parseConfig(cfg)          // 解析配置
    2. m.register()                // 向 Master 注册，拿 nodeId
    3. m.startRaftServer(cfg)      // 起 raftstore（多 raft 共享层）
    4. m.newMetaManager(cfg)       // 构造 MetadataManager（不启动）
    5. m.startServer()             // 起 TCP 服务端口（对外/对客户端）
    6. m.startSmuxServer()         // 起 smux 服务端口（多路复用）
    7. m.startMetaManager()        // 真正加载所有 MetaPartition
    8. m.registerAPIHandler()      // 注册 HTTP API
    9. go m.startUpdateNodeInfo()  // 后台心跳/上报
       m.startStat()               // 监控统计
       exporter.RegistConsul(...)  // 注册到 consul
    10. m.checkLocalPartitionMatchWithMaster() // 自检：本地 mp 是否和 master 一致
}
```

**这个顺序是有讲究的，记住三条原则**：

1. **先拿身份再干活**：`parseConfig` → `register` 必须最先，`register` 拿到 `nodeId` 后，后面 raft 才知道自己是哪一个 peer。
2. **先底层后上层**：`raftServer`（底层共享 raft 引擎）→ `metaManager`（基于 raft 的状态机）→ `server`（对外服务）。
3. **先建对象、再启服务**：`newMetaManager` 只是构造，`startMetaManager` 才会去 load 各 mp 的 snapshot 和 raft log。这样的好处是 **TCP 端口先 ready**，避免 master / client 在加载期间打过来直接拒连。

> ⚠️ 注意第 5 步 `startServer` 在第 7 步 `startMetaManager` 之前，意味着**端口已开但 mp 还没全部加载完**。请求进来时分发层会按 mp 状态返回错误码——这是我们 Layer 3 要重点看的细节。

---

## 3. `parseConfig` 都解析了什么

[metanode.go:213-369](../../metanode/metanode.go#L213-L369)，配置项可分 6 组：

| 组 | 字段 | 含义 |
|---|---|---|
| **网络** | `listen`、`bindIp`、`localAddr` | TCP 监听端口、是否绑 IP |
| **存储路径** | `metadataDir`、`raftDir` | mp 数据根目录、raft log 根目录 |
| **Raft 端口** | `raftHeartbeatPort`、`raftReplicatePort` | raft 心跳端口、复制数据端口（**注意是两个独立端口**）|
| **Raft 调优** | `tickInterval`、`raftRecvBufSize`、`raftRetainLogs`、`raftSyncSnapFormatVersion` | tick 周期、收包 buffer、保留日志数、快照格式版本 |
| **资源约束** | `cfgMemRatio` 或 `cfgTotalMem` | MetaNode 允许使用的总内存（**inode/dentry 全在内存，这是硬上限**）|
| **集群接入** | `masterAddr`、`zoneName`、`serviceIDKey` | master 地址、所属 zone、鉴权 key |

**几个值得记住的细节**：

- **内存配置二选一**：要么给 `cfgMemRatio`（占物理内存的百分比，1～100），要么给 `cfgTotalMem`（绝对字节数）。`configTotalMem` 是个**包级全局变量**（[metanode.go:43](../../metanode/metanode.go#L43)），后面 inode 子系统会读它来判断"是否到达上限、是否要踢冷数据"。
- **常量配置一致性校验**：[CheckOrStoreConstCfg](../../metanode/metanode.go#L320) 会把 `listen / raftHeartbeatPort / raftReplicaPort` 写到 `metadataDir/.const_config` 文件。**重启时必须和上次一致**——这是为了防止运维改了端口又用同一份数据目录，导致 raft peer 错乱。
- **Master 客户端是带 DNS Resolver 的**：[metanode.go:353](../../metanode/metanode.go#L353) 用的是 `NewMasterCLientWithResolver`，会按 `configNameResolveInterval`（默认上限 60s）周期性解析域名。这意味着 **master 地址支持写域名而不是固定 IP**。

---

## 4. `register`：向 Master 报到拿身份

[metanode.go:491-581](../../metanode/metanode.go#L491-L581) 是个**无限重试**的循环（每次失败 sleep 3s），干三件事：

1. **拉集群信息** `getClusterInfo()`
   → 写入 `clusterId`、`clusterUuid`、`clusterEnableSnapshot`、`raftPartitionCanUsingDifferentPort` 等集群级开关。
   → 如果本地没配 IP，就用 master 看到的对端 IP 作为 `localAddr`（**自动探测出口 IP**的小技巧）。

2. **拉升级兼容设置** `getUpgradeCompatibleSettings()`
   → 关键拿到 `VolsForbidWriteOpOfProtoVer0`（哪些卷禁止旧协议写）和 `LegacyDataMediaType`（老副本默认介质类型）。
   → 这是 CubeFS 引入"分级存储 / 混合云"后为兼容老版本加的字段，**新人看到 ProtoVer0 这种字眼就知道是历史包袱**。

3. **真正注册** `AddMetaNodeWithAuthNode(nodeAddress, raftHeartbeatPort, raftReplicatePort, zoneName, serviceIDKey)`
   → 成功后 master 返回 `nodeId`，写入 `m.nodeId`。
   → **`nodeId` 是后续 raft peer 标识的来源**，必须先有它才能起 raft。

> 💡 这就解释了为什么 `doStart` 里 `register` 排在 `startRaftServer` 前面——raft peer 必须有 ID。

---

## 5. `newMetaManager`：构造元数据管理器

[metanode.go:427-476](../../metanode/metanode.go#L427-L476)。注意它**只是构造**，不会真的去加载 mp。

关键点：

- 确保 `metadataDir` 存在（不存在就 mkdir）
- 再次做一次 `CheckOrStoreConstCfg` 校验（防御性写法）
- 解析 `gcRecyclePercent`（GC 回收比例阈值）
- 用 `MetadataManagerConfig` 把以下东西塞进去：
  - `NodeID`（来自 register）
  - `RootDir = metadataDir`
  - `RaftStore`（来自 startRaftServer，**它必须先于本步执行**）
  - `ZoneName`、`EnableGcTimer`、`GcRecyclePercent`
- 调 `NewMetadataManager(conf, m)` 创建

真正的 mp 加载发生在第 7 步 [startMetaManager](../../metanode/metanode.go#L478) 里调的 `metadataManager.Start()`——那是 Layer 4 的内容。

---

## 6. `Shutdown`：反向拆解

[doShutdown](../../metanode/metanode.go#L193-L206) 几乎是 `doStart` 的逆序：

```
stopUpdateNodeInfo → stopStat → stopServer → stopSmuxServer
→ stopMetaManager → stopRaftServer → masterClient.Stop
```

**逆序很重要**：先停外部入口（不再接新请求），再停内部状态机和 raft，最后断 master 连接。这样不会出现"请求进来但 mp 已经关了"的窗口。

---

## 7. 本层关键问题自测

读完 Layer 1，应该能回答：

1. ✅ MetaNode 的启动顺序为什么是"register → raftServer → newMetaManager → startServer → startMetaManager"？
2. ✅ 为什么 TCP 端口在 mp 加载完成之前就开了？带来什么问题？怎么处理？（→ Layer 3 答）
3. ✅ `configTotalMem` 这个全局变量约束的是什么？为什么必须有？（→ Layer 5 inode 答）
4. ✅ raftHeartbeatPort 和 raftReplicatePort 为什么要分开？（→ Layer 4 raft 答）
5. ✅ `.const_config` 文件起什么作用？

---

## 8. 给后续模块的钩子

- **Layer 2** 要读 `m.startServer()` / `m.startSmuxServer()`，看 TCP/SMux 监听器是怎么把请求扔给分发层的
- **Layer 3** 要读 `manager_op.go`，看 op code 是怎么 dispatch 到 mp 的
- **Layer 4** 要读 `m.startRaftServer()` 和 `metadataManager.Start()`，看 mp 怎么 load snapshot + replay raft log
- **Layer 5/6** 要读 `inode.go` / `dentry.go`，回头看 `configTotalMem` 在哪里起作用
- **Layer 7** 要读 `partition_store.go`，看 mp 持久化的格式
