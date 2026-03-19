# 端到端读取流程

## 完整读取链路

以 FUSE Client 读取文件 `/mnt/vol/dir1/file.txt` 为例：

```
┌──────────────────────────────────────────────────────────────────┐
│ 应用程序: fd = open("/mnt/vol/dir1/file.txt", O_RDONLY)          │
│          read(fd, buf, 4096)                                     │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ 步骤 1: Lookup + Open                                            │
│                                                                  │
│ FUSE → Dir.Lookup("file.txt"):                                   │
│   · DentryCache.Get("file.txt") → miss                          │
│   · MetaWrapper.Lookup_ll(parentIno=200, "file.txt"):            │
│     - getPartitionByInode(200) → MP-1                            │
│     - lookup(MP-1, 200, "file.txt") → OpMetaLookup               │
│     - 返回 (inode=12345, mode)                                   │
│   · InodeCache.Put(12345, info)                                  │
│   · DentryCache.Put("file.txt", 12345)                           │
│                                                                  │
│ FUSE → File.Open():                                              │
│   · ExtentClient.OpenStream(12345):                              │
│     - 获取/创建 Streamer                                          │
│     - RefreshExtentsCache:                                       │
│       · MetaWrapper.GetExtents(12345) → OpMetaExtentsList         │
│       · 返回 ExtentKey 列表                                       │
│       · 缓存到 Streamer.extents                                  │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ 步骤 2: read(buf, offset=0, size=4096)                           │
│                                                                  │
│ FUSE → File.Read():                                              │
│   · ExtentClient.Read(12345, data, 0, 4096):                    │
│     - Streamer.read(data, 0, 4096):                              │
│                                                                  │
│       1. PrepareReadRequests(0, 4096):                            │
│          · 查找 ExtentKey 缓存                                    │
│          · 定位: offset=0 → ExtentKey{                            │
│              FileOffset: 0,                                      │
│              PartitionId: 100,                                   │
│              ExtentId: 1024,                                     │
│              Size: 1MB                                           │
│            }                                                     │
│          · 生成 ReadRequest{offset=0, size=4096, ek}              │
│                                                                  │
│       2. 多级缓存查找:                                             │
│          a. AheadRead 预读窗口 → miss                             │
│          b. BCache 本地缓存 → miss                                │
│          c. RemoteCache FlashNode → miss (或跳过)                 │
│                                                                  │
│       3. 从 DataNode 读取:                                        │
│          · GetDataPartition(100) → DP-100 信息                    │
│          · SortHostsByPingElapsed():                              │
│            - 如果开启 FollowerRead: 选延迟最低的节点               │
│            - 否则: 选 Leader                                      │
│          · NewExtentReader(dp, ek)                                │
│          · reader.Read():                                        │
│            · TCP Packet(OpStreamRead/OpStreamFollowerRead)        │
│            · 发送到 DataNode-X                                    │
│            ▼                                                     │
│          DataNode-X:                                             │
│            · ExtentStore.Read(extentID=1024, offset=0, size=4096)│
│            · Extent.Read(): pread 从文件读取                      │
│            · 返回数据 + CRC                                       │
│                                                                  │
│       4. 异步缓存:                                                │
│          · asyncBlockCache: 写入 BCache                          │
│          · 可选: 写入 FlashNode RemoteCache                      │
│                                                                  │
│       5. 返回数据给 FUSE                                          │
└──────────────────────────────────────────────────────────────────┘
```

## 缓存命中路径

```
最快: AheadRead → 直接返回（内存）
    ↓ miss
次快: BCache → 本地 SSD 读取
    ↓ miss
中等: RemoteCache (FlashNode) → 同机房 SSD
    ↓ miss
最慢: DataNode → 可能跨机房 / HDD
```

## Follower Read

开启 Follower Read 后，读请求可以发送到任何副本（不限于 Leader），按延迟排序选择最优节点：

```go
func (dp *DataPartition) SortHostsByPingElapsed() {
    // 按 ping 延迟排序所有副本节点
    // 选择延迟最低的节点发送读请求
}
```

优势：
- 分散读压力到所有副本
- 选择地理最近的节点降低延迟
- Leader 故障时仍可读（最终一致）

## 大文件读取

大文件由多个 ExtentKey 组成，读取时自动分割为多个子请求：

```
文件 (10MB):
  ExtentKey-1: [0, 4MB)    → DP-100, Extent-1024
  ExtentKey-2: [4MB, 8MB)  → DP-101, Extent-2048
  ExtentKey-3: [8MB, 10MB) → DP-100, Extent-1025

read(offset=3MB, size=6MB):
  → ReadReq-1: DP-100, Extent-1024, offset=3MB, size=1MB
  → ReadReq-2: DP-101, Extent-2048, offset=0, size=4MB
  → ReadReq-3: DP-100, Extent-1025, offset=0, size=1MB
```
