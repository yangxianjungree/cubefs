# Master 模块概述

## 模块定位

Master 是 CubeFS 集群的资源管理中心，负责集群全局元数据管理、卷生命周期、分区调度、节点健康监控和拓扑管理。Master 节点通过 Raft 保证多副本一致性，所有元数据持久化到 RocksDB。

## 代码结构

```
master/
├── server.go                 # Server 结构体、启动/停止流程
├── cluster.go                # Cluster 核心结构、后台调度任务
├── metadata_fsm.go           # Raft 状态机（Apply/Snapshot/Restore）
├── metadata_fsm_op.go        # FSM 操作码定义
├── vol.go                    # 卷管理（创建/删除/扩缩容）
├── data_partition.go         # 数据分区管理
├── data_partition_map.go     # 数据分区映射
├── meta_partition.go         # 元数据分区管理
├── meta_partition_manager.go # MP 管理器
├── data_node.go              # DataNode 管理
├── meta_node.go              # MetaNode 管理
├── topology.go               # 拓扑管理（Zone/NodeSet）
├── cluster_balance.go        # 负载均衡
├── http_server.go            # HTTP 服务启动
├── api_service.go            # REST API 路由注册
├── admin_task_manager.go     # AdminTask 异步任务管理
├── id_allocator.go           # ID 分配器
├── config.go                 # 配置管理
├── const.go                  # 常量定义
└── ...
```

## 核心结构体

### Server

```go
type Server struct {
    id           uint64
    clusterName  string
    ip, port     string
    walDir       string             // Raft WAL 目录
    storeDir     string             // RocksDB 存储目录
    cluster      *Cluster           // 集群核心状态
    fsm          *MetadataFsm       // Raft 状态机
    partition    raftstore.Partition // Raft 分区（GroupID=1）
    raftStore    raftstore.RaftStore
    rocksDBStore *RocksDBStore
    reverseProxy *httputil.ReverseProxy // Follower -> Leader 转发
    apiServer    *http.Server
    leaderInfo   *LeaderInfo
}
```

### Cluster

Cluster 是 Master 最核心的结构体，管理整个集群状态：

```go
type Cluster struct {
    Name       string
    leaderInfo *LeaderInfo
    cfg        *clusterConfig
    fsm        *MetadataFsm
    partition  raftstore.Partition

    // 嵌入子结构
    ClusterVolSubItem   // vols map, 延迟删除卷
    ClusterTopoSubItem  // dataNodes, metaNodes, topology, idAlloc
    ClusterDecommission // 下线相关

    DisableAutoAllocate bool
    followerReadManager *followerReadManager
    lcMgr               *lifecycleManager
}
```

## 关键文件速览

| 文件 | 关注重点 |
|------|---------|
| `server.go` | 启动流程 9 步骤 |
| `cluster.go` | 20+ 后台调度 goroutine |
| `metadata_fsm.go` | Raft Apply 逻辑、Snapshot/Restore |
| `vol.go` | 卷创建/删除、MP/DP 自动创建 |
| `topology.go` | Zone -> NodeSet -> Node 三级拓扑 |
| `cluster_balance.go` | MP/DP 迁移均衡策略 |
