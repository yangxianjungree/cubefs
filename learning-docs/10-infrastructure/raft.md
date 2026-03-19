# Raft 层深度解析

## 概述

CubeFS 的 Raft 实现基于 `depends/tiglabs/raft`（源自 etcd Raft 的 Go 实现），通过 `raftstore/` 包封装为统一接口，供 Master、MetaNode、DataNode、AuthNode 等模块使用。

## RaftStore 接口

```go
type RaftStore interface {
    CreatePartition(cfg *PartitionConfig) (Partition, error)
    Stop()
    RaftConfig() *raft.Config
    RaftStatus(raftID uint64) *raft.Status
    NodeManager
    RaftServer() *raft.RaftServer
}
```

## Partition 接口

```go
type Partition interface {
    Submit(cmd []byte) (interface{}, error)  // 提交命令（仅 Leader）
    ChangeMember(changeType, peer, context)  // 成员变更
    Stop()                                    // 停止
    Status() *PartitionStatus                 // 状态
    IsRaftLeader() bool                       // 是否 Leader
    AppliedIndex() uint64                     // 已应用的日志索引
    CommittedIndex() uint64                   // 已提交的日志索引
    Truncate(index uint64)                    // 截断日志
    TryToLeader(nodeID uint64) error          // 尝试成为 Leader
    LeaderTerm() (leaderID, term uint64)      // 获取 Leader 和任期
}
```

## 状态机接口

各模块实现 `raft.StateMachine` 接口：

```go
type StateMachine interface {
    Apply(command []byte, index uint64) (interface{}, error)
    ApplyMemberChange(confChange ConfChange, index uint64) (interface{}, error)
    Snapshot() (Snapshot, error)
    ApplySnapshot(peers []Peer, iter SnapIterator) error
    HandleFatalEvent(err *FatalError)
    HandleLeaderChange(leader uint64)
}
```

## 各模块的 Raft 使用

| 模块 | Raft Group | 状态机 | 存储后端 | 说明 |
|------|-----------|--------|---------|------|
| Master | 1 个 (GroupID=1) | MetadataFsm | RocksDB | 集群全局元数据 |
| MetaNode | 每个 MP 一个 | metaPartition | 内存 B-Tree + 磁盘快照 | 文件元数据 |
| DataNode | 每个 DP 一个 | DataPartition | Extent 文件 | 数据写入一致性 |
| AuthNode | 1 个 | KeystoreFsm | RocksDB | 认证 Keystore |
| FlashGroupMgr | 1 个 | — | RocksDB | Flash Group 管理 |

## NodeResolver

管理 Raft 节点 ID 到网络地址的映射：

```go
type nodeResolver struct {
    nodes sync.Map  // nodeID → addresses
}
```

每个节点有两个端口：
- **Heartbeat Port**: Raft 心跳通信
- **Replica Port**: Raft 日志复制

## WAL（预写日志）

Raft 日志持久化到磁盘（WAL 目录）：
- Master: `walDir` 配置
- MetaNode: `raftDir/<partitionID>`
- DataNode: `raftDir/<partitionID>`

WAL 日志定期截断（`Truncate`）以释放磁盘空间。

## 配置参数

```go
type Config struct {
    NodeID        uint64  // 节点 ID
    RaftPath      string  // WAL 路径
    IPAddr        string  // 本机 IP
    HeartbeatPort int     // 心跳端口
    ReplicaPort   int     // 复制端口
    NumOfLogsToRetain uint64 // 保留日志数量
    TickInterval  int     // tick 间隔（ms）
    RecvBufSize   int     // 接收缓冲区大小
    ElectionTick  int     // 选举超时（tick 数）
}
```
