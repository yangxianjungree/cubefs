# RPC 与通信机制

## 通信协议概览

| 通信路径 | 协议 | 端口 |
|---------|------|------|
| Client → Master | HTTP (REST) | 17010 (默认) |
| Client → MetaNode | TCP (Packet) | MetaNode listen port |
| Client → MetaNode | Smux | listen port + shift |
| Client → DataNode | TCP (Repl Protocol) | DataNode listen port |
| Master → MetaNode/DataNode | TCP (AdminTask) | 各节点端口 |
| Raft 节点间 | TCP | heartbeat port / replica port |
| ObjectNode → MetaNode/DataNode | TCP | 同 Client |
| BlobStore 组件间 | HTTP | 各组件端口 |

## TCP Packet 协议

所有 Client ↔ MetaNode/DataNode 的通信使用自定义 Packet 二进制协议：

### 数据帧格式

```
┌─────────┬───────────┬────────┬────────────┬──────────┐
│ Header  │ Arg       │ Data   │            │          │
│ (固定)   │ (变长)     │ (变长)  │            │          │
└─────────┴───────────┴────────┴────────────┴──────────┘

Header 字段:
  Magic(1B) + ExtentType(1B) + Opcode(1B) + ResultCode(1B)
  + RemainingFollowers(1B) + CRC(4B) + Size(4B) + ArgLen(4B)
  + KernelOffset(8B) + PartitionID(8B) + ExtentID(8B)
  + ExtentOffset(8B) + ReqID(8B)
```

### 读写流程

```
发送:
  1. 填充 Packet 字段
  2. 序列化 Header
  3. 写入 Header + Arg + Data 到 TCP 连接

接收:
  1. 读取固定长度 Header
  2. 解析 ArgLen 和 Size
  3. 读取 Arg 和 Data
  4. 校验 Magic 和 CRC
```

## Smux 多路复用

CubeFS 支持通过 Smux (Stream Multiplexer) 在单个 TCP 连接上多路复用多个逻辑流：

### SmuxConnectPool

```go
type SmuxConnectPool struct {
    pools     map[string]*SmuxPool  // addr → 连接池
    cfg       SmuxConnPoolConfig
    tokenBkt  simpleTokenBucket     // 全局流控
}

type SmuxConnPoolConfig struct {
    TotalStreams    int  // 全局最大 stream 数
    StreamsPerConn int  // 每连接最大 stream 数
    ConnsPerAddr   int  // 每地址最大连接数
    DialTimeout    time.Duration
    StreamIdleTimeout time.Duration
}
```

### 使用场景

- MetaNode 监听 Smux 端口（主端口 + shift）
- DataNode 的修复连接
- SDK 客户端可选择 TCP 或 Smux

### 优势
- 减少连接数（一个连接复用多个 stream）
- 降低连接建立开销
- 更好的连接管理

## AdminTask（管理任务协议）

Master 向 MetaNode/DataNode 下发管理任务使用 AdminTask 结构：

```go
type AdminTask struct {
    ID         string
    OpCode     uint8
    OperAddr   string
    Request    interface{}
    Response   interface{}
    SendTime   int64
    Status     int8
}
```

AdminTask 通过 Packet 协议发送，AdminTaskManager 管理任务的发送、重试和超时。

## 连接池管理

### TCP 连接池 (util/ConnectPool)

```go
type ConnectPool struct {
    pools map[string]*Pool  // addr → Pool
}
```

- 每个目标地址维护一个连接池
- 连接复用，减少创建开销
- 支持超时和健康检查
