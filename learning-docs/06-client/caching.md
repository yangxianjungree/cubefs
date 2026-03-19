# FUSE Client 缓存机制

## 概述

FUSE Client 实现了多级缓存来减少网络请求，提升性能。

## 缓存层次

```
应用程序
    ▼
┌─────────────────────────────────┐
│ InodeCache (inode 属性缓存)       │  内存 LRU
├─────────────────────────────────┤
│ DentryCache (目录项缓存)          │  内存 Map
├─────────────────────────────────┤
│ Extent Key Cache (数据布局缓存)    │  Streamer 内
├─────────────────────────────────┤
│ BCache (本地块缓存)               │  本地磁盘
├─────────────────────────────────┤
│ RemoteCache (FlashNode 远程缓存)  │  分布式缓存
└─────────────────────────────────┘
    ▼
MetaNode / DataNode (源数据)
```

## InodeCache

```go
type InodeCache struct {
    cache      map[uint64]*list.Element  // inode ID → LRU 元素
    lruList    *list.List                // LRU 链表
    expiration time.Duration             // 过期时间
    maxElements int                      // 最大缓存数
}
```

### 操作

| 方法 | 说明 |
|------|------|
| `Put(info)` | 插入/更新缓存，设置过期时间，LRU 提升 |
| `Get(ino)` | 查找缓存，检查过期 |
| `Delete(ino)` | 删除缓存项 |
| `evict()` | LRU 驱逐过期项 |

### 使用场景
- `Attr()`: 先查 InodeCache，miss 则请求 MetaNode
- `Create/Lookup/ReadDir`: 成功后 Put 到缓存
- `Setattr`: 更新缓存
- `Forget`: 删除缓存

## DentryCache

### 目录级缓存 (DentryCache)

每个目录节点 (Dir) 有自己的 DentryCache：

```go
type DentryCache struct {
    cache      map[string]uint64  // name → inode ID
    expiration time.Duration
}
```

### 全局缓存 (Dcache)

Super 维护一个全局 Dcache，用于跨目录缓存：

```go
type Dcache struct {
    cache map[string]*proto.DentryInfo  // "parentIno_name" → DentryInfo
}
```

### 使用场景
- `Lookup`: 先查 dcache，miss 则请求 MetaNode
- `ReadDir`: 结果 Put 到 dcache
- `Remove/Rename`: 删除相关缓存

## Extent Key Cache

Streamer 内部缓存文件的 Extent Key 列表：

```go
type ExtentCache struct {
    inode    uint64
    gen      uint64
    size     uint64
    extents  []proto.ExtentKey  // 按 FileOffset 排序
}
```

### 刷新策略
- `OpenStream` 时从 MetaNode 获取最新 Extent Key 列表
- 写入后 append/update Extent Key
- 冲突时从 MetaNode 重新获取

## BCache（本地块缓存）

用于纠删码卷 (EC Volume) 的本地缓存加速：

- 部署在客户端机器上
- 使用本地 SSD 缓存热数据块
- 读取 miss 时从 DataNode/BlobStore 获取，异步写入 BCache
- 容量受本地磁盘限制

## RemoteCache（FlashNode 远程缓存）

分布式缓存服务，加速冷数据读取：

- FlashNode 集群提供缓存服务
- 客户端读取时先查 RemoteCache
- 比直接访问 BlobStore 更快（同机房 SSD）
- 支持预热 (Prepare) 和按需缓存

### 读取路径优先级

```
1. AheadRead（预读窗口命中）
   ↓ miss
2. BCache（本地块缓存）
   ↓ miss
3. RemoteCache（FlashNode）
   ↓ miss
4. DataNode / BlobStore（源数据）
   ↓ 读取成功后
5. 异步写入 BCache / RemoteCache
```

## 缓存一致性

CubeFS 的缓存是"弱一致性"的：

- **InodeCache**: TTL 过期后重新获取，可能短暂不一致
- **DentryCache**: 同上
- **Extent Key Cache**: 写入后立即更新，保证本客户端一致
- 多客户端并发写入同一文件时，由 MetaNode 的 Raft 保证元数据一致
- POSIX 语义适度放松以提升性能
