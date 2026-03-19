# FlashNode 分布式缓存

## 概述

FlashNode (`remotecache/`) 提供分布式读缓存服务，加速冷数据（特别是纠删码卷数据）的访问。

## 架构

```
Client (FUSE / SDK)
    │ 读请求
    ├── 本地缓存 miss
    ▼
FlashNode
    ├── CacheEngine (LRU 块缓存)
    │   ├── 磁盘存储（SSD）
    │   └── 内存索引
    │
    └── 如果缓存 miss:
        └── 从 DataNode / BlobStore 读取源数据
            └── 缓存到本地
```

## CacheEngine

```go
type CacheEngine struct {
    lruCacheMap   map[string]*LruCache  // 每个磁盘一个 LRU
    lruFhCache    *LruCache             // 文件句柄缓存
    keyToDiskMap  map[string]string      // key → 磁盘映射
    readSourceFunc func(...)             // 从源读取数据的回调
}
```

### 缓存 Key

```
Volume + Inode + FileOffset + Version → CacheBlock
```

### 操作

| 操作 | 说明 |
|------|------|
| `GetCacheBlockForRead` | 查找缓存块 |
| `CreateBlock` | 创建新缓存块（从源读取后缓存） |
| `PrepareCache` | 预热缓存（Master 触发） |
| `DeleteCache` | 删除缓存块 |
| `Scan` | 扫描清理过期缓存 |

## FlashNode 与 Master 交互

| OpCode | 说明 |
|--------|------|
| `OpFlashNodeHeartbeat` | 心跳上报 |
| `OpFlashNodeCachePrepare` | 缓存预热任务 |
| `OpFlashNodeCacheRead` | 缓存读取 |
| `OpFlashNodeCacheDelete` | 删除缓存 |
| `OpFlashNodeScan` | 扫描任务 |

## 读取路径

```
Client SDK:
    1. 检查本地 BCache → hit: 返回
    2. 检查 FlashNode RemoteCache → hit: 返回
    3. 从 DataNode/BlobStore 读取
    4. 异步写入 FlashNode 缓存
```

FlashNode 集群可以水平扩展，使用一致性哈希分配缓存块到不同节点。
