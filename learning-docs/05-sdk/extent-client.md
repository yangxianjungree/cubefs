# ExtentClient 数据 SDK

## 概述

ExtentClient (`sdk/data/stream/extent_client.go`) 管理文件数据的读写，为每个打开的文件（inode）维护一个 Streamer 实例。

## 核心结构体

```go
type ExtentClient struct {
    streamers    map[uint64]*Streamer  // inode -> Streamer
    streamerList *list.List            // LRU 列表
    dataWrapper  *wrapper.Wrapper      // DataPartition 访问
    metaWrapper  *meta.MetaWrapper     // 元数据访问

    // 回调函数（由上层注入）
    appendExtentKey func(inode uint64, ek proto.ExtentKey, ...) error
    getExtents      func(inode uint64) (uint64, uint64, []proto.ExtentKey, error)
    truncate        func(inode, size uint64, fullPath string) error

    // 限流
    readLimiter, writeLimiter *rate.Limiter

    // 缓存
    RemoteCache *RemoteCacheConfig
    AheadRead   *AheadReadConfig
}
```

## Streamer（流管理器）

每个打开的文件对应一个 Streamer：

```go
type Streamer struct {
    inode     uint64
    extents   *ExtentCache     // Extent Key 缓存
    handler   *ExtentHandler   // 当前写入 handler
    dirtylist *DirtyExtentList // 脏 handler 列表
    request   chan interface{} // 请求 channel
    done      chan struct{}    // 完成信号

    client    *ExtentClient
    aheadReadWindow *AheadReadWindow
}
```

### Streamer 服务循环

```go
func (s *Streamer) server() {
    for {
        select {
        case req := <-s.request:
            s.handleRequest(req)
        case <-ticker.C:
            s.traverse()  // 定期 flush 脏数据
        }
    }
}
```

## 写入流程

### 追加写 (Append)

```
ec.Write(inode, offset, data)
    ▼
Streamer.IssueWriteRequest(data, offset, size)
    ▼
Streamer.write():
    ▼
PrepareWriteRequests(offset, size) → 分割写请求
    ▼
对每个子请求:
    ├── 如果是覆盖写 → doOverwrite()
    └── 如果是追加写 → doWriteAppend()

doWriteAppend():
    ├── tryInitExtentHandlerByLastEk() → 复用或创建 ExtentHandler
    ├── ExtentHandler.write(data) → 缓冲到 handler
    └── 加入 dirtylist
    ▼
定期 traverse() 或显式 flush():
    ├── 遍历 dirtylist 中的 handler
    ├── handler.flush() → 发送 Packet 到 DataNode
    └── appendExtentKey() → 更新 MetaNode 的 Extent Key
```

### 覆盖写 (Overwrite)

```
doOverwrite():
    ├── 先 flush() 当前脏数据
    ├── 查找目标 Extent Key (通过 offset 定位)
    ├── 构建 OverwritePacket
    └── 发送 OpRandomWrite/OpSyncRandomWrite 到 DataNode
```

## 读取流程

```
ec.Read(inode, data, offset, size)
    ▼
Streamer.read():
    ├── 如果有脏数据 → 先 flush
    ├── PrepareReadRequests(offset, size) → 分割读请求
    └── 对每个子请求:
         ├── 如果是空洞 (hole) → 填零
         ├── 尝试 AheadRead → 预读取缓存
         ├── 尝试 BCache → 本地块缓存
         ├── 尝试 RemoteCache → FlashNode 远程缓存
         └── GetExtentReader → 从 DataNode 读取
              ├── GetDataPartition(partitionId)
              ├── NewExtentReader(dp, ek)
              └── reader.Read(data, offset, size)
```

## DataPartition 选择

### 写入选择

```go
func (w *Wrapper) GetDataPartitionForWrite(exclude) (*DataPartition, error) {
    // 使用 dpSelector 选择一个可写 DP
    // 排除 exclude 中的 DP（用于重试）
    // 考虑介质类型（SSD/HDD）
}
```

### 读取选择

```go
func (dp *DataPartition) SortHostsByPingElapsed() {
    // 如果开启 Follower Read:
    //   按 ping 延迟排序，选最近的节点
    // 否则:
    //   只返回 Leader 节点
}
```

## ExtentHandler（写入处理器）

```go
type ExtentHandler struct {
    stream    *Streamer
    extentID  uint64
    inode     uint64
    dp        *wrapper.DataPartition
    packet    *Packet   // 待发送的数据包
    size      int       // 当前累积的数据大小
}
```

每个 ExtentHandler 对应一个正在写入的 Extent。当写满或需要 flush 时，将数据发送到 DataNode。

## 生命周期管理

```
OpenStream(inode)  → 获取/创建 Streamer
    ▼
Write/Read/Flush   → 通过 Streamer 操作
    ▼
CloseStream(inode) → 释放 Streamer
    ▼
EvictStream(inode) → LRU 驱逐（超过 maxStreamerLimit 时）
```

后台 `backgroundEvictStream()` 在 Streamer 数量超过限制时自动驱逐。
