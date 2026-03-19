# 故障恢复流程

## 故障类型与恢复

### 1. DataNode 节点故障

```
DataNode-X 宕机
    ▼
Master 检测:
    · scheduleToCheckHeartbeat() → 心跳超时 (默认 6s 检查)
    · 标记 DataNode-X 为离线
    ▼
影响范围:
    · DataNode-X 上所有 DataPartition 的副本变为 Missing
    ▼
自动恢复:
    · checkDataPartitions():
      - 检查每个受影响的 DP
      - checkMissingReplicas() → 发现 Missing 副本
      - checkReplicaNum() → 副本数不足
    · 如果 DataNode-X 长时间未恢复:
      - 创建新副本到其他健康 DataNode
      - 新副本通过修复流程从存活副本同步数据
    ▼
客户端影响:
    · 写入: 如果丢失的是 Leader → 重新选举 → 短暂中断后恢复
    · 读取: Follower Read → 可从其他副本读取
```

### 2. MetaNode 节点故障

```
MetaNode-A 宕机
    ▼
Master 检测:
    · 心跳超时 → 标记离线
    ▼
影响范围:
    · MetaNode-A 上所有 MetaPartition 的副本变为 Missing
    ▼
自动恢复:
    · 每个受影响的 MP:
      - Raft 自动选举新 Leader（如果原 Leader 在 A 上）
      - 剩余 2/3 副本继续工作
    · 如果长时间未恢复:
      - Master 创建新 MP 副本到其他 MetaNode
      - 新副本通过 Raft Snapshot 同步数据
```

### 3. 磁盘故障

```
DataNode-X 的 /disk1 故障
    ▼
DataNode 检测:
    · Disk.checkDiskStatus() → 写入/读取探活文件失败
    · triggerDiskError() → 错误计数超过阈值
    · 标记磁盘为 Unavailable
    ▼
上报 Master:
    · 心跳中标记磁盘状态
    ▼
Master 恢复:
    · scheduleToBadDisk() → 处理坏盘
    · 为受影响的 DP 创建新副本
    · 发送 OpRecoverBadDisk 到 DataNode
    ▼
DataNode 处理:
    · 停止受影响的 DP 的 Raft
    · 标记受影响的 DP
```

### 4. Master 节点故障

```
Master Leader 宕机
    ▼
Raft 自动选举:
    · 剩余 Master 节点选出新 Leader
    · 选举时间: 通常 < 10 秒
    ▼
新 Leader 恢复:
    · loadMetadata() → 从 RocksDB 加载所有集群状态
    · 重新启动所有后台调度任务
    ▼
客户端影响:
    · MasterClient 自动切换到新 Leader
    · 通过 403 响应发现新 Leader 地址
    · 短暂中断后自动恢复
```

## DataPartition 修复详细流程

```
Leader DataNode 定期 repair():
    ▼
1. buildDataPartitionRepairTask():
   · 获取所有副本的 Extent 信息
   · Leader: 本地 ExtentStore.SnapShot()
   · Follower: OpGetAllWatermarks → 远程获取
    ▼
2. prepareRepairTasks():
   · 比较所有副本:
     - ExtentsToBeCreated: 某副本缺少的 Extent
     - ExtentsToBeRepaired: 大小不一致的 Extent
    ▼
3. NotifyExtentRepair():
   · OpNotifyReplicasToRepair → 通知 Follower
    ▼
4. DoRepair() - Leader 修复:
   · 创建缺失的 Extent
   · streamRepairExtent: 从健康副本流式复制数据
    ▼
5. Follower 自修复:
   · 收到通知后，从 Leader 获取缺失数据
   · Tiny Extent: TinyExtentRecover
   · Normal Extent: 流式读取差异部分
```

## 脑裂防护

CubeFS 通过 Raft 的多数确认机制防止脑裂：
- 写入必须获得多数副本确认
- Leader 选举需要多数投票
- 网络分区时，少数派自动变为只读
