# 集群扩缩容

## 扩容

### 扩容 DataNode

```
1. 部署新 DataNode，配置 Master 地址
2. DataNode 启动 → 向 Master 注册:
   · Master 分配 NodeID
   · 加入对应 Zone 和 NodeSet
   · 更新集群拓扑
3. 后续新建 DataPartition 时，Master 会选择新节点
4. 负载均衡自动将部分 DP 迁移到新节点
```

### 扩容 MetaNode

```
1. 部署新 MetaNode，配置 Master 地址
2. MetaNode 启动 → 向 Master 注册
3. 后续新建 MetaPartition 和分裂时会选择新节点
4. 负载均衡自动迁移部分 MP
```

### 扩容 Master

```
1. 部署新 Master 节点
2. 通过 AdminAddRaftNode API 添加到 Raft Group
3. 新节点自动从 Leader 同步数据（Raft Snapshot）
4. 加入后参与选举
注意: Master 节点数建议为奇数 (3/5/7)
```

## 缩容（下线节点）

### DataNode 下线

```
AdminAPI: POST /dataNode/decommission {addr: "192.168.0.10"}
    ▼
Master 处理:
    1. 标记 DataNode 为 ToBeOffline
    2. 遍历该节点上的所有 DataPartition
    3. 对每个 DP:
       · 选择新的目标 DataNode
       · 创建新副本 (OpCreateDataPartition)
       · 添加新 Raft 成员 (OpAddDataPartitionRaftMember)
       · 等待数据同步完成
       · 移除旧 Raft 成员 (OpRemoveDataPartitionRaftMember)
    4. 所有 DP 迁移完成后，标记 DataNode 为 Offline
```

### MetaNode 下线

```
AdminAPI: POST /metaNode/decommission {addr: "192.168.0.20"}
    ▼
类似 DataNode 下线:
    1. 标记 ToBeOffline
    2. 为每个 MP 在新节点创建副本
    3. 通过 Raft Snapshot 同步数据
    4. 移除旧副本
```

### 磁盘下线

```
AdminAPI: POST /disk/decommission {addr: "192.168.0.10", disk: "/data1"}
    ▼
只迁移该磁盘上的 DP，不下线整个节点
    ▼
支持并行下线（可配置并行度）
    ▼
进度查询: AdminQueryDiskDecommissionInfoStat
```

## 负载均衡

### MetaPartition 均衡

```
scheduleStartBalanceTask() → 每 1 分钟检查
    ▼
GetMetaNodePressureView():
    · 识别内存负载高的 MetaNode
    · 识别内存负载低的 MetaNode
    ▼
CreateMetaPartitionMigratePlan():
    · 从高负载节点选择 MP 迁移
    · FindMigrateDestination: 选择低负载目标节点
    · 执行迁移:
      - 在目标节点创建新副本
      - Raft 同步数据
      - 移除源节点副本
```

### DataPartition 均衡

配置项:
- `DataNodeBalanceOn`: 开关
- `DataNodeBalanceInterval`: 检查间隔
- `DataNodeBalanceByDiskUsageHigh/Low`: 磁盘使用率阈值
- `DataNodeBalanceByDPCountHigh/Low`: DP 数量阈值

均衡策略:
- 磁盘使用率超过 High → 作为源节点
- 磁盘使用率低于 Low → 作为目标节点
- 从源节点迁移 DP 到目标节点

## 扩容对客户端的影响

| 操作 | 客户端影响 |
|------|-----------|
| 添加 DataNode | 无感知，新 DP 自动分配到新节点 |
| 添加 MetaNode | 无感知，新 MP 自动分配 |
| 下线 DataNode | DP 迁移期间可能有短暂延迟增加 |
| 下线 MetaNode | MP 迁移期间 Raft 重选举，短暂中断 |
| 磁盘下线 | 仅影响该磁盘上的 DP |

## 容量规划

| 组件 | 扩容建议 |
|------|---------|
| Master | 3/5/7 节点，不频繁扩容 |
| MetaNode | 按内存规划，每 GB 内存约支持 1000 万 inode |
| DataNode | 按存储容量规划，注意单节点磁盘数 |
| ObjectNode | 无状态，按 QPS 需求水平扩容 |
| FlashNode | 按缓存容量和读 QPS 规划 |
