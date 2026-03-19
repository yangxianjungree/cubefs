# Master 启动流程

## 启动入口

Master 的启动入口在 `cmd/cmd.go` 中，通过 `master.NewServer()` 创建 Server 实例，然后调用 `server.Start(cfg)` 启动。

## 启动步骤 (server.go Start → doStart)

```
┌──────────────────────────────────────────────────┐
│ 1. 解析配置 (newClusterConfig + checkConfig)       │
│    · peers、端口、节点容量、超时时间等                  │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│ 2. 初始化 RocksDB (NewRocksDBStoreAndRecovery)    │
│    · storeDir 下创建 RocksDB 实例                   │
│    · 持久化集群元数据                                │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│ 3. 创建 Raft Server (createRaftServer)            │
│    · NewRaftStore: NodeID, WAL路径, 端口            │
│    · newMetadataFsm: 创建状态机                     │
│    · initFsm: 注册 handler, restore                │
│    · CreatePartition: GroupID=1, 所有 peers         │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│ 4. 初始化 Cluster (initCluster)                    │
│    · newCluster: 绑定 FSM、partition、config        │
│    · 关联 idAlloc.partition                         │
│    · 设置 MasterSecretKey                           │
│    · scheduleTask(): 启动 20+ 后台 goroutine        │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│ 5. 初始化 User (initUser)                          │
│    · newUser: 用户/AK/卷用户管理                     │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│ 6. 启动 HTTP 服务 (startHTTPService)               │
│    · registerAPIRoutes: 注册所有 REST/GraphQL API   │
│    · 中间件: API限流、Leader检查、认证                 │
│    · 启动 http.Server                               │
└──────────────────┬───────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────┐
│ 7. 注册 Consul / 监控                              │
│    · Consul 服务注册                                │
│    · Prometheus 指标导出                             │
│    · 统计日志                                       │
└──────────────────────────────────────────────────┘
```

## Raft 选主后的加载流程

当 Master 节点成为 Raft Leader 时，`MetadataFsm.HandleLeaderChange()` 被调用：

```
Leader 当选
    ▼
leaderChangeHandler(leader)   // cluster.handleLeaderChange
    ▼
loadMetadata()                // 从 RocksDB 加载所有集群元数据
    ├── loadClusterValue()     // 集群配置
    ├── loadNodeSets()         // NodeSet
    ├── loadDataNodes()        // DataNode
    ├── loadMetaNodes()        // MetaNode
    ├── loadVols()             // Volume
    ├── loadMetaPartitions()   // MetaPartition
    ├── loadDataPartitions()   // DataPartition
    ├── loadUsers()            // 用户信息
    └── ...
    ▼
Follower 切换时
    ▼
clearMetadata()               // 清理内存状态
```

## 关闭流程

```
server.Shutdown()
    ├── 设置 stopFlag
    ├── 关闭 stopc channel
    ├── 等待所有后台 goroutine 退出 (wg.Wait)
    ├── 关闭 HTTP Server
    ├── 停止 FSM
    └── 关闭 RocksDB
```

## 配置文件关键项

```json
{
    "role": "master",
    "ip": "192.168.0.1",
    "port": "17010",
    "prof": "17020",
    "logDir": "/var/log/cubefs/master",
    "logLevel": "info",
    "walDir": "/var/data/cubefs/master/wal",
    "storeDir": "/var/data/cubefs/master/store",
    "retainLogs": 20000,
    "clusterName": "my-cluster",
    "peers": "1:192.168.0.1:17010,2:192.168.0.2:17010,3:192.168.0.3:17010"
}
```
