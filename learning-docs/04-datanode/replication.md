# DataNode 副本协议

## 概述

CubeFS DataNode 使用链式复制 (Chain Replication) + Raft 保证数据副本一致性。写入操作由客户端发送到 Leader DataNode，Leader 通过 Raft 保证副本一致性。

## ReplProtocol 结构

```go
type ReplProtocol struct {
    packetList      *list.List           // 待处理 Packet 队列
    ackCh           chan struct{}         // Follower 响应确认
    toBeProcessedCh chan *Packet          // 待处理 channel
    responseCh      chan *Packet          // 响应 channel
    sourceConn      net.Conn             // 客户端连接
    followerConnects map[string]*FollowerTransport // 到 Follower 的连接
    prepareFunc     func(*Packet) error  // 预处理函数
    operatorFunc    func(*Packet, *net.TCPConn) error // 操作函数
    postFunc        func(*Packet) error  // 后处理函数
}
```

## 写入流程

### 三副本写入（链式复制 + Raft）

```
Client
  │
  │ 发送 Write Packet (OpWrite / OpRandomWrite)
  ▼
Leader DataNode (ReplProtocol)
  │
  ├── 1. ServerConn(): 读取 Packet
  │      · ReadFromConnWithVer()
  │      · resolveFollowersAddr(): 解析 Follower 地址
  │      · prepareFunc(): 预处理（校验）
  │      · 放入 toBeProcessedCh
  │
  ├── 2. OperatorAndForwardPktGoRoutine():
  │      · 从 toBeProcessedCh 取出
  │      · 如果有 Follower:
  │      │   · 发送到所有 Follower (FollowerTransport)
  │      │   · operatorFunc(): 本地执行（写入 Extent + Raft）
  │      │   · 等待 Follower 响应 (ackCh)
  │      · 如果无 Follower:
  │      │   · operatorFunc(): 本地执行
  │      · 放入 responseCh
  │
  ├── 3. ReceiveResponseFromFollowersGoRoutine():
  │      · 等待 ackCh
  │      · 收集所有 Follower 的响应
  │      · 合并结果
  │
  └── 4. writeResponseToClientGoRoutine():
         · 从 responseCh 取出
         · postFunc(): 后处理
         · 写响应到客户端连接
```

### FollowerTransport

```go
type FollowerTransport struct {
    sendCh chan *FollowerPacket  // 发送 channel
    recvCh chan *FollowerPacket  // 接收 channel
    conn   *net.TCPConn         // 到 Follower 的连接
}
```

- `serverWriteToFollower()`: 从 sendCh 读取 Packet，发送到 Follower
- `serverReadFromFollower()`: 读取 Follower 响应，放入 recvCh

## Raft 写入

DataPartition 的写入操作通过 Raft 保证一致性：

### Apply 逻辑 (partition_raftfsm.go)

```go
func (dp *DataPartition) Apply(command []byte, index uint64) (interface{}, error) {
    // 1. 解析命令版本
    // 2. 如果 index > metaAppliedID:
    //    · ApplyRandomWrite(): 将数据写入 Extent 文件
    // 3. 更新 appliedID
}
```

### 顺序写 vs 随机写

| 类型 | OpCode | Raft | 说明 |
|------|--------|------|------|
| 顺序写 | `OpWrite` / `OpSyncWrite` | 链式复制 | 追加写，使用 Primary-Backup 协议 |
| 随机写 | `OpRandomWrite` / `OpSyncRandomWrite` | Multi-Raft | 覆盖写，使用 Raft 保证强一致性 |

### 写入类型

```go
type WriteParam struct {
    ExtentID   uint64
    Offset     int64
    Size       int64
    Data       []byte
    Crc        uint32
    WriteType  int    // AppendWrite / RandomWrite
    IsSync     bool   // 是否同步刷盘
    IsHole     bool   // 是否是空洞
    IsRepair   bool   // 是否是修复写
}
```

## 读取流程

```
Client → DataNode (Leader 或 Follower)
    ▼
OpRead / OpStreamRead / OpStreamFollowerRead
    ▼
ExtentStore.Read(extentID, offset, size)
    ▼
Extent.Read() → pread 从文件读取
    ▼
返回数据 + CRC
```

### Follower Read

对于读请求，支持从 Follower 读取：
- `OpStreamFollowerRead`: 客户端直接发送到 Follower
- 减轻 Leader 压力
- 可能读到稍旧的数据（最终一致）

## 连接管理

### TCP 连接池

```go
// DataNode 与其他 DataNode 之间的连接池
type ConnectPool struct {
    sync.RWMutex
    pools map[string]*Pool  // addr -> Pool
}
```

### Smux 连接池

可选使用 Smux 多路复用：
- 在单个 TCP 连接上复用多个流
- 减少连接数，提高效率
- 用于修复等场景

## 副本状态同步

### Applied ID 同步

Leader 定期收集所有副本的 Applied ID：

```
Leader:
  1. getAllReplicaAppliedID(): 向所有 Follower 查询 appliedID
  2. updateMaxMinAppliedID(): 计算 min/max
  3. broadcastMinAppliedID(): 广播 minAppliedID 给所有 Follower
  4. 当 minAppliedID > lastTruncateID 时，截断 Raft 日志
```

### Raft 日志截断

定期（10 分钟）检查是否可以截断 Raft 日志：
- 条件: `minAppliedID > lastTruncateID + 1`
- 截断后更新 `lastTruncateID`
- 减少磁盘空间占用
