# DataNode 数据修复

## 概述

数据修复是 DataNode 的关键能力，确保副本间数据一致性。当发现副本不一致（Extent 缺失、大小不一致等）时，自动触发修复流程。

## 修复触发

修复由 `LaunchRepair()` 定期触发（通过 `statusUpdateScheduler`），调用 `DoExtentStoreRepair()`。

## 修复流程

```
repair()
    │
    ├── 1. 获取损坏的 Tiny Extent
    │
    ├── 2. buildDataPartitionRepairTask()
    │      · Leader: 获取本地 Extent 信息
    │      · Follower: 通过 OpGetAllWatermarks 获取远程 Extent 信息
    │      · 构建每个副本的修复任务
    │
    ├── 3. prepareRepairTasks()
    │      · 比较所有副本的 Extent 列表
    │      · buildExtentCreationTasks(): 找出缺失的 Extent
    │      · buildExtentRepairTasks(): 找出大小不一致的 Extent
    │
    ├── 4. NotifyExtentRepair()
    │      · 通知 Follower 需要修复的 Extent
    │      · 发送 OpNotifyReplicasToRepair
    │
    ├── 5. DoRepair()
    │      · Leader 创建缺失的 Extent
    │      · Leader 修复大小不一致的 Extent
    │
    └── 6. sendAllTinyExtentsToC()
           · 更新 availableTinyExtentC / brokenTinyExtentC
```

## Extent 比较

### 构建修复任务

对于一个 3 副本的 DataPartition：

```
Leader (Node-A):  [E1: 1MB, E2: 2MB, E3: 500KB]
Follower (Node-B): [E1: 1MB, E3: 500KB]           ← 缺少 E2
Follower (Node-C): [E1: 1MB, E2: 1.5MB, E3: 500KB] ← E2 大小不一致
```

修复任务：
- Node-B: 创建 E2，从 Leader 同步 2MB 数据
- Node-C: 修复 E2，从 1.5MB 补齐到 2MB

### 比较逻辑

```go
// 构建每个 Extent 的最大大小 map
extentInfoMap[extentID] = maxSize(across all replicas)

// 对每个副本检查
for each replica:
    for each extent in maxSizeMap:
        if replica 没有该 extent:
            → ExtentsToBeCreated (需要创建)
        if replica 的 extent 比 maxSize 小:
            → ExtentsToBeRepaired (需要修复)
```

## 修复数据传输

### streamRepairExtent

从源副本流式读取数据修复本地 Extent：

```
streamRepairExtent(remoteAddr, extentID)
    │
    ├── 比较本地 vs 远程大小
    │
    ├── Tiny Extent 修复:
    │   · TinyExtentRecover(data)
    │   · 包括写入新数据和 punch hole 已删除区域
    │
    └── Normal Extent 修复:
        · 发送 OpExtentRepairRead 到源副本
        · 源副本从 localSize 开始流式读取
        · 本地写入接收到的数据
        · 处理空洞（hole）：写入空数据块
```

### Tiny 删除记录同步

```
doStreamFixTinyDeleteRecord()
    │
    ├── 比较 Leader 的 TinyDeleteRecordFileSize 与本地大小
    │
    ├── 如果本地落后:
    │   · 从 Leader 读取缺失的删除记录
    │   · OpReadTinyDeleteRecord
    │   · 本地 RecordTinyDelete()
    │
    └── 确保 Tiny Extent 的删除操作在所有副本一致
```

## 磁盘故障处理

### 故障检测

```go
func (d *Disk) triggerDiskError(dp *DataPartition, err error) {
    // 1. 增加错误计数 (ReadErrCnt / WriteErrCnt)
    // 2. 将 DP 加入 DiskErrPartitionSet
    // 3. 如果错误次数超过阈值:
    //    · 调用 doDiskError()
    //    · 标记磁盘为 Unavailable
}
```

### 磁盘探活

```go
func (d *Disk) checkDiskStatus() {
    // 写入 .diskStatus 文件
    // 读取并验证
    // 如果失败: 标记为 Unavailable
}
```

### 坏盘恢复

当 Master 检测到某个 DataNode 的磁盘故障后：
1. Master 发送 `OpRecoverBadDisk` 到 DataNode
2. DataNode 标记受影响的 DataPartition
3. Master 为受影响的 DP 创建新副本（在其他节点）
4. 新副本通过修复流程从健康副本同步数据

## 修复限流

为避免修复操作影响正常读写，DataNode 提供限流机制：

```go
// Disk 级别的修复读限流
extentRepairReadLimit chan struct{}  // 信号量，限制并发修复读

// 获取修复读 token
func (d *Disk) RequireReadExtentToken() {
    d.extentRepairReadLimit <- struct{}{}
}

// 释放修复读 token
func (d *Disk) ReleaseReadExtentToken() {
    <-d.extentRepairReadLimit
}
```
