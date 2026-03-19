# 冷热数据分层

## 概述

CubeFS 支持冷热数据自动分层，将不常访问的数据从高性能副本存储迁移到低成本纠删码存储。

## 分层架构

```
热数据层 (Hot Tier)               冷数据层 (Cold Tier)
┌───────────────────┐            ┌───────────────────┐
│ DataNode (3副本)   │ ─迁移──>  │ BlobStore (纠删码)  │
│ · SSD / HDD       │            │ · HDD / 对象存储    │
│ · 高吞吐、低延迟    │            │ · 低成本、高可靠     │
└───────────────────┘            └───────────────────┘
         ▲                                ▲
         │                                │
    ┌────┴────┐                    ┌──────┴──────┐
    │  Client  │ ← 读取时自动路由 → │  FlashNode   │
    └─────────┘                    │  (分布式缓存)  │
                                   └─────────────┘
```

## 迁移触发

### 方式一：S3 生命周期规则

通过 LcNode 执行 S3 BucketLifecycleConfiguration：

```json
{
    "Rules": [{
        "ID": "move-to-cold",
        "Prefix": "logs/",
        "Status": "Enabled",
        "Transition": {
            "Days": 30,
            "StorageClass": "GLACIER"
        }
    }]
}
```

### 方式二：存储类别迁移

根据 StorageClass 配置：
- `Replica_SSD` → `Replica_HDD`: SSD 到 HDD（介质降级）
- `Replica_*` → `BlobStore`: 副本到纠删码（存储引擎切换）

## 迁移流程

### 副本 → 纠删码迁移

```
LcNode 扫描发现符合迁移条件的文件
    ▼
transitionMgr.migrateToEbs(inode):
    1. 获取 inode 的 ExtentKey 列表（副本数据位置）
    2. 从 ExtentClient 读取数据
    3. 通过 BlobStoreClient 写入 BlobStore
       · 数据被 Reed-Solomon 编码
       · 分片写入多个 BlobNode
    4. 获取 ObjExtentKey（纠删码数据位置）
    5. MetaNode.UpdateExtentKeyAfterMigration:
       · 替换 inode 的 ExtentKey → ObjExtentKey
       · 更新 StorageClass
    6. 标记旧副本数据可删除
```

### SSD → HDD 迁移

```
LcNode 发现符合条件的文件
    ▼
transitionMgr.migrate(inode):
    1. 获取 inode 的 ExtentKey
    2. 请求 Master 在 HDD 类型的 DataNode 上创建新 DP
    3. 将数据从 SSD DP 复制到 HDD DP
    4. 更新 MetaNode 的 ExtentKey 指向新 DP
    5. 删除旧 SSD 上的数据
```

## 读取路由

客户端读取时，根据 Inode 的 StorageClass 自动选择读取路径：

```go
// client/fs/file.go
func (f *File) Read(req, resp) {
    if f.info.StorageClass == BlobStore {
        // 冷数据: BlobStoreClient → BlobStore
        fReader.Read(data, offset, size)
    } else {
        // 热数据: ExtentClient → DataNode
        ec.Read(ino, data, offset, size)
    }
}
```

### 冷数据读取加速

冷数据读取通过 FlashNode 分布式缓存加速：

```
Client → FlashNode (SSD 缓存)
    ↓ miss
Client → BlobStore (纠删码存储)
    ↓ 读取成功
异步缓存到 FlashNode
```

## Inode 中的混合存储

一个 Inode 可以同时包含副本和纠删码的 Extent Key：

```go
type Inode struct {
    // 副本存储 Extent Key
    HybridCloudExtents *SortedHybridCloudExtents

    // 迁移中的 Extent Key
    HybridCloudExtentsMigration *SortedHybridCloudExtentsMigration

    StorageClass uint32  // 当前存储类别
}
```

这允许文件在迁移过程中部分数据已迁移、部分仍在原位。
