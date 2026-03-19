# Master 分区管理

## 概述

Master 管理两类分区：**DataPartition（数据分区）** 和 **MetaPartition（元数据分区）**。每个分区都有多个副本分布在不同节点上，通过 Raft 保证一致性。

## DataPartition 管理

### DataPartition 结构体

```go
type DataPartition struct {
    PartitionID   uint64
    ReplicaNum    uint8
    Status        int8          // ReadWrite / ReadOnly / Unavailable
    Replicas      []*DataReplica
    Hosts         []string      // 副本节点地址列表
    Peers         []proto.Peer  // Raft peers
    VolName       string
    VolID         uint64
    MediaType     uint32        // SSD / HDD
    MissingNodes  map[string]int64
    isRecover     bool
    // 下线相关字段...
}

type DataReplica struct {
    Addr       string
    dataNode   *DataNode
    Used       uint64
    Total      uint64
    Status     int8
    IsLeader   bool
    NeedsToCompare bool
    HasLoadResponse bool
}
```

### 创建流程

```
创建卷 / DP 数量不足
    ▼
batchCreateDataPartition(vol, count)
    ▼
选择节点:
    ├── 域管理模式: getHostFromDomainZone()
    │   └── 从 NodeSet 组中选择，确保跨域
    └── 普通模式: getHostFromNormalZone()
        └── 从 Zone 的 NodeSet 中 Straw 加权选择
    ▼
为每个选中节点创建 AdminTask (OpCreateDataPartition)
    ▼
发送到各 DataNode 执行
    ▼
syncAddDataPartition() → Raft 提交到集群元数据
```

### 状态检查

`checkDataPartitions()` 定期执行：

1. **checkReplicaStatus**: 检查每个副本是否正常
2. **checkStatus**: 根据副本状态更新 DP 状态
3. **checkLeader**: 确保有且仅有一个 Leader
4. **checkMissingReplicas**: 检查是否有丢失的副本
5. **checkReplicaNum**: 副本数是否达标
6. **checkDiskError**: 是否有磁盘错误
7. **checkReplicationTask**: 是否需要触发复制任务

### 状态转换

```
ReadWrite ──(所有副本正常)──> ReadWrite
    │
    ├──(部分副本异常)──> ReadOnly
    │
    └──(超过半数异常)──> Unavailable
```

### 自动创建

当卷的可写 DP 数量低于阈值时，`autoCreateDataPartitions()` 自动创建新 DP：

- 检查集群未冻结
- 检查 ReadWrite 状态的 DP 数量
- 按需批量创建

## MetaPartition 管理

### MetaPartition 结构体

```go
type MetaPartition struct {
    PartitionID uint64
    Start       uint64       // Inode ID 范围起始
    End         uint64       // Inode ID 范围结束
    MaxInodeID  uint64       // 当前最大已分配的 Inode ID
    InodeCount  uint64
    DentryCount uint64
    Replicas    []*MetaReplica
    ReplicaNum  uint8
    Status      int8
    Hosts       []string
    Peers       []proto.Peer
    volID       uint64
    volName     string
    MissNodes   map[string]int64
}

type MetaReplica struct {
    Addr       string
    metaNode   *MetaNode
    MaxInodeID uint64
    InodeCount uint64
    DentryCount uint64
    Status     int8
    IsLeader   bool
}
```

### Inode ID 范围分配

每个 MetaPartition 管理一个 Inode ID 区间 `[Start, End)`：

```
MP-1: [0, 16M)
MP-2: [16M, 32M)
MP-3: [32M, 48M)
...
```

当客户端创建文件时，根据 Inode ID 路由到对应的 MetaPartition。

### 分裂机制

MetaPartition 的分裂是 Master 的核心机制之一：

```
checkSplitMetaPartition()
    │
    ├── 条件判断：
    │   · 是当前最大 MP（End = maxPartitionID）
    │   · Inode 使用率 > 75% (MaxInodeID - Start) / (End - Start)
    │   · 或者 MP 内存使用过高
    │   · 或者 ReadWrite MP 数量 < 3
    │
    └── 触发分裂
         ▼
doSplitMetaPartition()
    ├── 1. 收缩当前 MP 的 End = MaxInodeID + step
    ├── 2. 创建新 MP: Start = 旧End, End = maxPartitionID
    ├── 3. syncBatchCommitCmd: 原子提交 update + add
    ├── 4. updateInodeIDRangeForAllReplicas
    └── 5. 通知副本更新
```

### 创建流程

```
创建卷时 / 分裂触发时
    ▼
doCreateMetaPartition(vol, start, end)
    ▼
选择节点:
    ├── 域管理: chooseTargetMetaHosts(domainManager)
    └── 普通: chooseTargetMetaHosts(zones)
    ▼
allocateMetaPartitionID()  // 分配全局唯一 MP ID
    ▼
构建 AdminTask (OpCreateMetaPartition)
    ▼
发送到各 MetaNode
    ▼
syncAddMetaPartition() → Raft 提交
```

## 卷管理

### Vol（卷）结构体

```go
type Vol struct {
    ID, Name, Owner  string
    Status           uint8
    VolType          int        // 副本卷 / 纠删码卷
    Capacity         uint64

    dpReplicaNum     uint8      // DP 副本数
    mpReplicaNum     uint8      // MP 副本数
    dataPartitionSize uint64    // DP 大小

    MetaPartitions   map[uint64]*MetaPartition
    dataPartitions   *DataPartitionMap

    qosManager       *QosCtrlManager
    quotaManager     *MasterQuotaManager
    VersionMgr       *VolVersionManager
}
```

### 卷创建流程

```
AdminCreateVol HTTP API
    ▼
cluster.createVol(name, owner, capacity, ...)
    ▼
1. 分配 Volume ID
2. newVol() 创建卷对象
3. syncAddVol() → Raft 持久化
4. initMetaPartitions():
   · 创建初始 MP (默认 3 个)
   · Start/End 范围依次递增
5. 自动创建初始 DataPartitions
6. 注册到 cluster.vols map
```

### 卷删除流程

```
AdminDeleteVol
    ▼
markDeleteVol()
    ├── 设置 Status = MarkDelete
    ├── 检查 DeleteLockTime（延迟删除保护）
    └── syncUpdateVol() → Raft
    ▼
（延迟到期后）
deleteVolFromStore()
    ├── 删除所有 DataPartition（发送删除任务到 DataNode）
    ├── 删除所有 MetaPartition（发送删除任务到 MetaNode）
    ├── syncDeleteVol() → Raft 删除卷元数据
    └── 从 cluster.vols 移除
```
