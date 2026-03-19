# MetaPartition 分裂机制

## 为什么需要分裂

每个 MetaPartition 管理一段 Inode ID 范围 `[Start, End)`。当一个分区中的 inode 数量过多时，内存使用量会增大、Raft 快照变慢，因此需要分裂来分摊负载。

## 分裂触发条件

Master 定期调用 `checkSplitMetaPartition()` 检查是否需要分裂。满足以下任一条件时触发：

1. **Inode 使用率高**: `(MaxInodeID - Start) / (End - Start) > 0.75`
2. **内存使用高**: MetaPartition 内存占用超过阈值
3. **可写分区不足**: ReadWrite 状态的 MP 数量 < 3

前提条件：
- 该 MP 必须是当前卷的**最后一个 MP**（End = maxPartitionID）
- 集群未冻结
- 分区有 Leader

## 分裂流程

```
Master: checkSplitMetaPartition()
    │
    ├── 确认是最后一个 MP 且满足分裂条件
    │
    └── vol.splitMetaPartition(mp)
         │
         └── doSplitMetaPartition(vol, mp)
              │
              ├── 1. 计算新的分裂点
              │      curEnd = mp.MaxInodeID + step
              │      newStart = curEnd
              │      newEnd = maxPartitionID
              │
              ├── 2. 收缩当前 MP 范围
              │      mp.End = curEnd
              │      构建 update RaftCmd
              │
              ├── 3. 创建新 MP
              │      doCreateMetaPartition(vol, newStart, newEnd)
              │      选择节点、分配 MP ID
              │      构建 add RaftCmd
              │
              ├── 4. 原子批量提交
              │      syncBatchCommitCmd([update, add])
              │      → Raft 保证原子性
              │
              ├── 5. 更新所有副本的 Inode ID 范围
              │      updateInodeIDRangeForAllReplicas()
              │      向旧 MP 的所有副本发送 OpUpdateMetaPartition
              │
              └── 6. 添加更新任务
                     addUpdateMetaReplicaTask()
```

## 分裂前后对比

假设卷有一个 MP，Inode 范围为 [0, 2^63)：

### 分裂前
```
MP-1: [0, 2^63)  ← 所有 inode 都在这个分区
      MaxInodeID = 100000
      InodeCount = 80000
```

### 分裂后
```
MP-1: [0, 100100)          ← 收缩范围（MaxInodeID + step）
      MaxInodeID = 100000
      InodeCount = 80000   ← 现有数据不动

MP-2: [100100, 2^63)       ← 新分区接管后续范围
      MaxInodeID = 0
      InodeCount = 0       ← 空分区，后续新 inode 分配到这里
```

## 关键细节

### 分裂是"逻辑分裂"

- 不涉及数据搬迁：旧 MP 的数据不动，只是收缩了 End 值
- 新创建的 MP 从新的 Start 开始，接收后续的 inode 创建请求
- 客户端通过 Inode ID 路由到正确的 MP

### 客户端路由更新

分裂完成后，客户端需要更新 MP 视图：
1. Master 更新卷的 MP 列表
2. 客户端定期从 Master 拉取最新的 MP 视图
3. SDK 的 MetaWrapper 根据 Inode ID 路由到正确的 MP

### 原子性保证

分裂操作通过 `syncBatchCommitCmd` 原子提交两个操作（更新旧 MP + 添加新 MP），保证不会出现中间状态。

## 分裂与负载均衡的关系

分裂只是"纵向扩展"（增加 MP 数量），不涉及"横向均衡"（将 MP 迁移到其他节点）。负载均衡由 Master 的 `cluster_balance.go` 单独处理：

```
分裂: 增加 MP 数量 → 分散后续写入负载
均衡: 迁移 MP 副本 → 均衡各 MetaNode 的负载
```
