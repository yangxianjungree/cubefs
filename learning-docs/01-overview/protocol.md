# CubeFS 协议层梳理

## 概述

CubeFS 的节点间通信使用自定义的二进制 Packet 协议，定义在 `proto/packet.go`。所有 Client -> MetaNode/DataNode 的通信、以及 Master 下发到各节点的管理任务，都通过这套协议进行。

## Packet 结构

```go
// proto/packet.go
type Packet struct {
    Magic              uint8   // 魔数，固定为 0xFF
    ExtentType         uint8   // Extent 类型（Tiny/Normal），最高位用于版本标记
    Opcode             uint8   // 操作码，标识请求类型
    ResultCode         uint8   // 响应码
    RemainingFollowers uint8   // 剩余需要转发的 follower 数量
    CRC                uint32  // 数据 CRC 校验
    Size               uint32  // Data 字段的长度
    ArgLen             uint32  // Arg 字段的长度
    KernelOffset       uint64  // 内核偏移量
    PartitionID        uint64  // 分区 ID
    ExtentID           uint64  // Extent ID
    ExtentOffset       int64   // Extent 内偏移量
    ReqID              int64   // 请求 ID（全局递增）
    Arg                []byte  // 附加参数（如创建/追加操作的地址信息）
    Data               []byte  // 数据载荷
    StartT             int64   // 请求开始时间戳
    // mesg string 为未导出字段，用于日志等，此处略
    HasPrepare         bool    // 是否已 prepare
    VerSeq             uint64  // 版本序列号（多版本快照）
    ProtoVersion       uint32  // 协议版本（见下表）
    VerList            []*VolVersionInfo  // 多版本列表
    noPrefix           bool    // 内部用，日志格式控制
}
```

### 协议版本

**说明**：版本以 `proto/packet.go` 中常量注释为准。代码中 Packet 结构体上方注释写 “version-0: before v3.4; version-1: from v3.4”，与常量注释不一致，以常量为准。

| 版本 | 值 | 说明 |
|------|---|------|
| `PacketProtoVersion0` | 0 | v3.5.0 之前的协议格式 |
| `PacketProtoVersion1` | 1 | v3.5.0 起的协议格式，增加了版本和标志位支持 |

### 关键常量

| 常量 | 值 | 说明 |
|------|---|------|
| `ProtoMagic` | 0xFF | 包头魔数 |
| `TinyExtentType` | 0 | 小文件 Extent |
| `NormalExtentType` | 1 | 普通 Extent |
| `MultiVersionFlag` | 0x80 | 多版本标记位 |
| `VersionListFlag` | 0x40 | 版本列表标记位 |

## OpCode 分类

### 1. DataNode 操作 (0x01 - 0x1F)

客户端 / MetaNode 与 DataNode 之间的数据操作（下表为主要操作，副本与修复相关会用到 0x0A、0x10、0x11、0x16–0x18 等）：

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpCreateExtent` | 0x01 | 创建新 Extent |
| `OpMarkDelete` | 0x02 | 标记删除 Extent |
| `OpWrite` | 0x03 | 写入数据 |
| `OpRead` | 0x04 | 读取数据 |
| `OpStreamRead` | 0x05 | 流式读取 |
| `OpStreamFollowerRead` | 0x06 | 从 Follower 流式读取 |
| `OpGetAllWatermarks` | 0x07 | 获取所有水位标记 |
| `OpNotifyReplicasToRepair` | 0x08 | 通知副本修复 |
| `OpExtentRepairRead` | 0x09 | 修复读 |
| `OpBroadcastMinAppliedID` | 0x0A | 广播最小已应用 ID |
| `OpRandomWrite` | 0x0F | 随机写 |
| `OpGetAppliedId` | 0x10 | 获取已应用 ID |
| `OpGetPartitionSize` | 0x11 | 获取分区大小 |
| `OpSyncRandomWrite` | 0x12 | 同步随机写 |
| `OpSyncWrite` | 0x13 | 同步写 |
| `OpReadTinyDeleteRecord` | 0x14 | 读取 Tiny 删除记录 |
| `OpTinyExtentRepairRead` | 0x15 | Tiny Extent 修复读 |
| `OpGetMaxExtentIDAndPartitionSize` | 0x16 | 获取最大 ExtentID 与分区大小 |
| `OpSnapshotExtentRepairRead` | 0x17 | 快照 Extent 修复读 |
| `OpSnapshotExtentRepairRsp` | 0x18 | 快照 Extent 修复响应 |

### 2. MetaNode 操作 (0x20 - 0x3F)

客户端与 MetaNode 之间的元数据操作：

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpMetaCreateInode` | 0x20 | 创建 Inode |
| `OpMetaUnlinkInode` | 0x21 | 取消 Inode 链接 |
| `OpMetaCreateDentry` | 0x22 | 创建目录项 |
| `OpMetaDeleteDentry` | 0x23 | 删除目录项 |
| `OpMetaOpen` | 0x24 | 打开文件 |
| `OpMetaLookup` | 0x25 | 查找目录项 |
| `OpMetaReadDir` | 0x26 | 读取目录 |
| `OpMetaInodeGet` | 0x27 | 获取 Inode 信息 |
| `OpMetaBatchInodeGet` | 0x28 | 批量获取 Inode |
| `OpMetaExtentsAdd` | 0x29 | 添加 Extent Key |
| `OpMetaExtentsDel` | 0x2A | 删除 Extent Key |
| `OpMetaExtentsList` | 0x2B | 列出 Extent Key |
| `OpMetaUpdateDentry` | 0x2C | 更新目录项（rename） |
| `OpMetaTruncate` | 0x2D | 截断文件 |
| `OpMetaLinkInode` | 0x2E | 硬链接 |
| `OpMetaEvictInode` | 0x2F | 驱逐 Inode |
| `OpMetaSetattr` | 0x30 | 设置属性 |
| `OpMetaSetXAttr` | 0x35 | 设置扩展属性 |
| `OpMetaGetXAttr` | 0x36 | 获取扩展属性 |
| `OpMetaReadDirLimit` | 0x3D | 分页读取目录 |

### 3. Master -> MetaNode 管理操作 (0x40 - 0x4F)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpCreateMetaPartition` | 0x40 | 创建元数据分区 |
| `OpMetaNodeHeartbeat` | 0x41 | 元数据节点心跳 |
| `OpDeleteMetaPartition` | 0x42 | 删除元数据分区 |
| `OpUpdateMetaPartition` | 0x43 | 更新元数据分区 |
| `OpLoadMetaPartition` | 0x44 | 加载元数据分区 |
| `OpDecommissionMetaPartition` | 0x45 | 下线元数据分区 |
| `OpAddMetaPartitionRaftMember` | 0x46 | 添加 Raft 成员 |
| `OpRemoveMetaPartitionRaftMember` | 0x47 | 移除 Raft 成员 |
| `OpMetaPartitionTryToLeader` | 0x48 | MP 尝试成为 Leader |
| `OpFreezeEmptyMetaPartition` | 0x49 | 冻结空 MP |
| `OpBackupEmptyMetaPartition` | 0x4A | 备份空 MP |
| `OpRemoveBackupMetaPartition` | 0x4B | 移除备份 MP |
| `OpIsRaftStatusOk` | 0x4C | 检查 Raft 状态 |

### 4. Master -> DataNode 管理操作 (0x60 - 0x8F)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpCreateDataPartition` | 0x60 | 创建数据分区 |
| `OpDeleteDataPartition` | 0x61 | 删除数据分区 |
| `OpLoadDataPartition` | 0x62 | 加载数据分区 |
| `OpDataNodeHeartbeat` | 0x63 | 数据节点心跳 |
| `OpDecommissionDataPartition` | 0x66 | 下线数据分区 |
| `OpAddDataPartitionRaftMember` | 0x67 | 添加 Raft 成员 |
| `OpRemoveDataPartitionRaftMember` | 0x68 | 移除 Raft 成员 |
| `OpRecoverBadDisk` | 0x6E | 坏盘恢复 |
| `OpQueryBadDiskRecoverProgress` | 0x6F | 查询坏盘恢复进度 |
| `OpDeleteBackupDirectories` | 0x80 | 删除备份目录 |
| `OpDeleteLostDisk` | 0x8A | 删除丢失盘 |
| `OpReloadDisk` | 0x8B | 重载磁盘 |
| `OpSetRepairingStatus` | 0x8C | 设置修复状态 |

### 5. 分布式事务操作 (0xA0 - 0xAD)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpMetaTxCreate` | 0xA0 | 创建事务 |
| `OpMetaTxCreateInode` | 0xA1 | 事务内创建 Inode |
| `OpMetaTxUnlinkInode` | 0xA2 | 事务内取消 Inode 链接 |
| `OpMetaTxCreateDentry` | 0xA3 | 事务内创建目录项 |
| `OpTxCommit` | 0xA4 | 提交事务 |
| `OpTxRollback` | 0xA5 | 回滚事务 |

### 6. Multipart 操作 (0x70 - 0x77)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpCreateMultipart` | 0x70 | 创建 Multipart 上传 |
| `OpGetMultipart` | 0x71 | 获取 Multipart 信息 |
| `OpAddMultipartPart` | 0x72 | 添加 Part |
| `OpRemoveMultipart` | 0x73 | 删除 Multipart |
| `OpListMultiparts` | 0x74 | 列出 Multipart |

### 7. FlashNode 操作 (0xC1 - 0xCD)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpFlashNodeHeartbeat` | 0xC1 | FlashNode 心跳 |
| `OpFlashNodeCachePrepare` | 0xC2 | 缓存预热 |
| `OpFlashNodeCacheRead` | 0xC3 | 缓存读取 |
| `OpFlashNodeCachePutBlock` | 0xC4 | 写入缓存块 |
| `OpFlashNodeCacheDelete` | 0xC5 | 删除缓存 |
| `OpFlashNodeScan` | 0xC9 | 缓存扫描 |

### 8. 多版本快照操作 (0xB1 - 0xB8)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpRandomWriteAppend` | 0xB1 | 追加式随机写 |
| `OpSyncRandomWriteAppend` | 0xB2 | 同步追加式随机写 |
| `OpRandomWriteVer` | 0xB3 | 带版本随机写 |
| `OpVersionOp` | 0xB8 | 版本操作 |

### 9. 错误/状态码 (0xEE - 0xFF)

| OpCode | 值 | 说明 |
|--------|---|------|
| `OpOk` | 0xF0 | 成功 |
| `OpErr` | 0xF8 | 通用错误 |
| `OpNotExistErr` | 0xF5 | 不存在 |
| `OpDiskNoSpaceErr` | 0xF6 | 磁盘空间不足 |
| `OpDiskErr` | 0xF7 | 磁盘错误 |
| `OpAgain` | 0xF9 | 请重试 |
| `OpExistErr` | 0xFA | 已存在 |
| `OpInodeFullErr` | 0xFB | Inode 已满 |
| `OpTryOtherAddr` | 0xFC | 请尝试其他地址 |
| `OpNotPerm` | 0xFD | 无权限 |
| `OpNotEmpty` | 0xFE | 非空 |
| `OpNoSpaceErr` | 0xEE | 无空间 |

## Admin HTTP API

Master 对外暴露 HTTP RESTful API，定义在 `proto/admin_proto.go`，主要分类如下：

### 集群管理
- `GET /admin/getCluster` - 获取集群信息
- `GET /cluster/stat` - 集群统计
- `POST /cluster/freeze` - 冻结集群

### 卷管理
- `POST /admin/createVol` - 创建卷
- `GET /admin/getVol` - 获取卷信息
- `GET /vol/list` - 列出所有卷
- `POST /vol/delete` - 删除卷
- `POST /vol/update` - 更新卷配置
- `POST /vol/expand` - 扩容卷
- `POST /vol/shrink` - 缩容卷

### 数据分区
- `GET /dataPartition/get` - 获取数据分区信息
- `POST /dataPartition/create` - 创建数据分区
- `POST /dataPartition/decommission` - 下线数据分区
- `GET /dataPartition/diagnose` - 诊断数据分区

### 元数据分区
- `POST /metaPartition/create` - 创建元数据分区

### 节点管理
- `GET /admin/cluster/getAllDataNodes` - 获取所有数据节点
- `GET /admin/cluster/getAllMetaNodes` - 获取所有元数据节点

### 客户端 API
- `GET /client/partitions` - 客户端获取数据分区列表
- `GET /client/vol` - 客户端获取卷信息
- `GET /client/metaPartitions` - 客户端获取元数据分区列表

## Packet 读写流程

### 发送请求

```
1. 构造 Packet 对象，设置 Opcode、PartitionID、ExtentID 等字段
2. 序列化为二进制：Header (固定长度) + Arg (变长) + Data (变长)
3. 通过 TCP 连接发送
```

### 接收响应

```
1. 从 TCP 连接读取固定长度的 Header
2. 根据 Header 中的 ArgLen 和 Size 读取 Arg 和 Data
3. 校验 Magic 和 CRC
4. 检查 ResultCode 判断操作是否成功
```

### 超时控制

| 常量 | 值 | 说明 |
|------|---|------|
| `WriteDeadlineTime` | 5s | 写超时 |
| `ReadDeadlineTime` | 5s | 读超时 |
| `SyncSendTaskDeadlineTime` | 30s | 同步发送任务超时 |
| `BatchDeleteExtentReadDeadLineTime` | 120s | 批量删除 Extent 读超时 |
| `GetAllWatermarksDeadLineTime` | 60s | 获取水位超时 |

## Smux 多路复用

除了直接 TCP 连接外，CubeFS 还支持通过 Smux（Stream Multiplexer）在单个 TCP 连接上多路复用多个逻辑流，减少连接数开销。MetaNode 在常规 TCP 端口之外还会监听一个 Smux 端口。
