# Master 集群管理

## Cluster 后台调度任务

Master Cluster 启动后会运行 20+ 个后台 goroutine，每个负责一类管理任务：

### 核心调度任务

| 任务 | 周期 | 职责 |
|------|------|------|
| `scheduleToCheckHeartbeat` | 6s | 检查 DataNode/MetaNode 心跳，标记离线节点 |
| `scheduleToCheckDataPartitions` | 5s | 检查 DP 状态、副本健康、自动创建新 DP |
| `scheduleToCheckMetaPartitions` | 定期 | 检查 MP 状态、触发分裂、检查副本数 |
| `scheduleToCheckVolStatus` | 定期 | 检查卷状态、处理卷删除、刷新视图缓存 |
| `scheduleToCheckDelayDeleteVols` | 5s | 处理延迟删除的卷 |
| `scheduleToUpdateStatInfo` | 2min | 更新集群统计信息 |
| `scheduleToManageDp` | 启动后2min | 开启 DP 自动创建 |

### 数据管理任务

| 任务 | 职责 |
|------|------|
| `scheduleToLoadDataPartitions` | 从 DataNode 加载 DP 指标 |
| `scheduleToCheckReleaseDataPartitions` | 释放已加载 DP 的内存 |
| `scheduleToLoadMetaPartitions` | 从 MetaNode 加载 MP 指标 |
| `scheduleToCheckDataReplicaMeta` | 检查 DP 副本元数据一致性 |

### 下线/修复任务

| 任务 | 职责 |
|------|------|
| `scheduleToCheckDecommissionDataNode` | DataNode 下线进度检查 |
| `scheduleToCheckDecommissionDisk` | 磁盘下线进度检查 |
| `scheduleToCheckDiskRecoveryProgress` | 磁盘恢复进度 |
| `scheduleToCheckMetaPartitionRecoveryProgress` | MP 恢复进度 |
| `scheduleToBadDisk` | 坏盘处理 |
| `scheduleToCheckDataPartitionRepairingStatus` | DP 修复状态 |

### 负载均衡任务

| 任务 | 职责 |
|------|------|
| `scheduleStartBalanceTask` | 每1分钟检查 MP 负载均衡 |
| `scheduleToCheckVolQos` | 每1秒检查卷 QoS |
| `scheduleToCheckNodeSetGrpManagerStatus` | 检查 NodeSet 组状态 |

### 其他任务

| 任务 | 职责 |
|------|------|
| `scheduleToCheckFollowerReadCache` | 刷新 Follower 读缓存 |
| `scheduleToLcScan` | 生命周期扫描 |
| `scheduleToSnapshotDelVerScan` | 快照版本删除扫描 |
| `scheduleToUpdateFlashGroupRespCache` | Flash Group 响应缓存 |
| `scheduleToUpdateFlashGroupSlots` | Flash Group 槽位更新 |
| `scheduleToCheckVolUid` | Volume UID 检查 |

## 节点心跳处理

### DataNode 心跳

```
DataNode -> Master (定期上报)
    │
    ├── 上报信息：
    │   · 节点 ID、地址、Zone
    │   · Total/Used/Available 空间
    │   · DataPartition 列表和状态
    │   · 磁盘信息
    │
    └── Master 处理：
        · 更新 DataNode 状态
        · 更新 DataPartition 副本信息
        · 检测故障（心跳超时 → 标记离线）
        · 触发副本修复
```

### MetaNode 心跳

```
MetaNode -> Master (定期上报)
    │
    ├── 上报信息：
    │   · 节点 ID、地址、Zone
    │   · 内存 Total/Used/Threshold
    │   · MetaPartition 列表和状态
    │   · 每个 MP 的 InodeCount、DentryCount、MaxInodeID
    │
    └── Master 处理：
        · 更新 MetaNode 状态
        · 更新 MetaPartition 副本信息
        · 检查是否需要分裂 MP
        · 检测故障
```

## 拓扑管理

### 三级拓扑结构

```
Cluster
  └── Zone（可用区）
       └── NodeSet（节点集合，默认容量18）
            ├── DataNode
            └── MetaNode
```

### Zone

- 代表一个可用区（机房/机架）
- 包含多个 NodeSet
- 有独立的节点选择器（NodesetSelector）
- 副本放置时确保跨 Zone 分布

### NodeSet

- 节点集合，默认容量 18
- 同一个 NodeSet 内的节点地理位置相近
- 包含 DataNode 和 MetaNode 的选择器
- 管理下线 DP 列表

### 副本放置策略

创建 DataPartition 或 MetaPartition 时，Master 按以下策略选择节点：

1. **域管理模式** (`domainOn = true`):
   - 从 DomainNodeSetGrpManager 获取 NodeSet 组
   - 确保副本分布在不同 NodeSet
   - 支持 3-zone、2+1、单 zone 等策略

2. **普通模式**:
   - 从指定 Zone 或默认 Zone 选择节点
   - 在 NodeSet 内选择可用节点
   - 使用加权随机算法（Straw）按可用空间选择

## AdminTask 任务管理

Master 通过 `AdminTaskManager` 向 DataNode/MetaNode 下发管理任务：

```go
type AdminTask struct {
    OpCode    uint8       // 操作码（如 OpCreateDataPartition）
    OperAddr  string      // 目标节点地址
    Request   interface{} // 请求体
    Response  interface{} // 响应体
    SendTime  int64       // 发送时间
    Status    int8        // 任务状态
}
```

任务通过 `AdminTaskManager.sender` 协程异步发送，支持重试和超时控制。

## ID 分配

Master 通过 `IDAllocator` 为全局资源分配唯一 ID：

| ID 类型 | 用途 |
|---------|------|
| Volume ID | 卷 ID |
| DataPartition ID | 数据分区 ID |
| MetaPartition ID | 元数据分区 ID |
| Common ID | 通用 ID |

ID 分配通过 Raft 保证全局唯一：先从 Raft 预分配一批，用完后再预分配。
