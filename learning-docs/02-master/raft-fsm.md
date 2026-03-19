# Master Raft 状态机

## 概述

Master 使用单个 Raft Group（GroupID=1）管理所有集群元数据。多个 Master 节点通过 Raft 保证元数据一致性，持久化到 RocksDB。

## MetadataFsm 结构体

```go
// master/metadata_fsm.go，store 类型来自 raftstore_db 包
type MetadataFsm struct {
    store               *raftstore.RocksDBStore  // RocksDB 存储（包名：raftstore/raftstore_db）
    rs                  *raft.RaftServer
    applied             uint64
    retainLogs          uint64
    leaderChangeHandler raftLeaderChangeHandler
    peerChangeHandler   raftPeerChangeHandler
    snapshotHandler     raftApplySnapshotHandler
    UserAppCmdHandler   raftUserCmdApplyHandler
    onSnapshot          bool    // 是否正在应用快照
    raftLk              sync.Mutex
}
```

## Raft 操作码

Master FSM 支持以下操作类型（定义在 `metadata_fsm_op.go`）：

### 添加操作 (opSyncAdd*)
| 操作码 | 说明 |
|--------|------|
| `opSyncAddMetaNode` | 添加 MetaNode |
| `opSyncAddDataNode` | 添加 DataNode |
| `opSyncAddDataPartition` | 添加 DataPartition |
| `opSyncAddVol` | 添加 Volume |
| `opSyncAddMetaPartition` | 添加 MetaPartition |
| `opSyncAddNodeSet` | 添加 NodeSet |
| `opSyncAddUserInfo` | 添加用户信息 |
| `opSyncAddLcNode` | 添加 LcNode |
| `opSyncAddDecommissionDisk` | 添加下线磁盘 |
| `opSyncAddFlashNode` | 添加 FlashNode |
| `opSyncAddFlashGroup` | 添加 Flash Group |
| `opSyncAddBalanceTask` | 添加均衡任务 |

### 删除操作 (opSyncDelete*)
| 操作码 | 说明 |
|--------|------|
| `opSyncDeleteDataNode` | 删除 DataNode |
| `opSyncDeleteMetaNode` | 删除 MetaNode |
| `opSyncDeleteVol` | 删除 Volume |
| `opSyncDeleteDataPartition` | 删除 DataPartition |
| `opSyncDeleteMetaPartition` | 删除 MetaPartition |
| `opSyncDeleteUserInfo` | 删除用户 |

### 更新操作 (opSyncPut*)
| 操作码 | 说明 |
|--------|------|
| `opSyncPutCluster` | 更新集群配置 |
| `opSyncPutApiLimiterInfo` | 更新 API 限流信息 |
| `opSyncPutFollowerApiLimiterInfo` | 更新 Follower API 限流 |

### 批量操作
| 操作码 | 说明 |
|--------|------|
| `opSyncBatchPut` | 批量写入（用于原子更新多个键） |

## Apply 逻辑

```go
func (mf *MetadataFsm) Apply(command []byte, index uint64) (interface{}, error) {
    // 1. 反序列化 RaftCmd
    cmd := new(RaftCmd)
    json.Unmarshal(command, cmd)

    // 2. 构建 cmdMap（待写入 RocksDB 的 KV）
    cmdMap := make(map[string][]byte)

    // 3. 根据操作类型处理
    switch cmd.Op {
    case 删除类操作:
        // 从 RocksDB 删除 key，写入 applied index
        delKeyAndPutIndex(cmd.K, cmdMap)

    case opSyncBatchPut:
        // 批量操作：反序列化嵌套命令，分为删除集和写入集
        // BatchDeleteAndPut(deleteSet, cmdMap, true)

    case API限流操作:
        // 调用 UserAppCmdHandler，然后 BatchPut

    default:
        // 写入 key=value + applied=index
        cmdMap[cmd.K] = cmd.V
        cmdMap[applied] = index
        BatchPut(cmdMap, true)
    }

    // 4. 更新 applied index
    mf.applied = index

    // 5. 定期截断 Raft 日志
    if applied % retainLogs == 0 {
        mf.truncateRaftLog()
    }
}
```

### RaftCmd 结构

```go
type RaftCmd struct {
    Op uint32  // 操作码
    K  string  // Key（如 "vol_myvolume", "dp_1234"）
    V  []byte  // Value（JSON 序列化后的对象）
}
```

## Snapshot 机制

### 生成快照

```go
func (mf *MetadataFsm) Snapshot() (raft.Snapshot, error) {
    // 返回 RocksDB 的迭代器作为快照
    return mf.store.RocksDBSnapshot()
}
```

### 应用快照

```go
func (mf *MetadataFsm) ApplySnapshot(peers []raft.Peer, iter raft.SnapIterator) error {
    // 1. 清空当前 RocksDB
    mf.store.Clear()

    // 2. 从迭代器逐条恢复 KV
    for {
        data, err := iter.Next()
        if err == io.EOF { break }
        mf.store.Put(data.K, data.V)
    }

    // 3. 调用 snapshotHandler 重新加载内存状态
    mf.snapshotHandler()
}
```

## Leader 切换

```go
func (mf *MetadataFsm) HandleLeaderChange(leader uint64) {
    if leader == myNodeID {
        // 成为 Leader: 从 RocksDB 加载所有元数据到内存
        go leaderChangeHandler(leader)
        // → loadMetadata()
    } else {
        // 变为 Follower: 清理内存状态
        go leaderChangeHandler(leader)
        // → clearMetadata()
    }
}
```

## 数据流示意

```
Client API 请求 (如 CreateVol)
    ▼
Master Leader 处理
    ▼
构建 RaftCmd{Op: opSyncAddVol, K: "vol_xxx", V: volJSON}
    ▼
partition.Submit(cmd)  // 提交到 Raft
    ▼
Raft 复制到多数节点
    ▼
Apply(cmd, index)  // 各节点应用
    ├── Leader: BatchPut 到 RocksDB + 更新内存
    └── Follower: BatchPut 到 RocksDB（内存由 Leader 管理）
```

## 持久化 Key 命名规则

RocksDB 中存储的 Key 遵循 `master/const.go` 中的定义：`keySeparator = "#"`，格式为 `#前缀#id#...`（前缀与 id 之间用 `#` 分隔）。与 `metadata_fsm_op.go` 中 build 函数一致：

| 前缀 | 实际格式 | 示例 | 存储内容 |
|------|----------|------|---------|
| `#mn#` | `#mn#id#addr` | `#mn#1#192.168.0.1:17210` | MetaNode |
| `#dn#` | `#dn#id#addr` | `#dn#2#192.168.0.1:17310` | DataNode |
| `#vol#` | `#vol#volID` | `#vol#100` | Volume（volID 为数字） |
| `#mp#` | `#mp#volID#metaPartitionID` | `#mp#100#1` | MetaPartition |
| `#dp#` | `#dp#volID#partitionID` | `#dp#100#200` | DataPartition |
| `#user#` | `#user#userid` | `#user#admin` | 用户信息（前缀为 user，非 usr） |
| `#c#` | `#c#name` | `#c#clusterName` | 集群配置（前缀为 c，非 cluster） |
