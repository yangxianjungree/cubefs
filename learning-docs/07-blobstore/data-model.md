# BlobStore 数据模型

## 四层数据结构

```
Volume (逻辑卷)
  └── Chunk (数据块组)
       └── Shard (分片)
            └── Blob (数据对象)
```

### Volume
- 逻辑卷，由 ClusterMgr 管理
- 包含多个 Chunk
- 有编码模式（如 EC 4+2）

### Chunk
- 数据块组，分布在多个 BlobNode 上
- 由数据 Shard + 校验 Shard 组成
- 对应编码模式的一个条带 (stripe)

### Shard
- 分片，存储在单个 BlobNode 的磁盘上
- 一个 Chunk 的数据 Shard 和校验 Shard 分别存储在不同节点

### Blob
- 实际的数据对象
- 写入时被编码分片存储到各 Shard
- 读取时从 Shard 组装恢复

## 数据写入流程

```
Client → Access.Put(data)
    ▼
1. Proxy.Alloc() → 从 ClusterMgr 分配 Volume 空间
2. Reed-Solomon 编码:
   · 将数据分为 N 个数据块
   · 计算 M 个校验块
3. 将各 Shard 写入不同的 BlobNode
4. 返回 BlobKey (Volume + Offset + Size)
```

## 数据读取流程

```
Client → Access.Get(blobKey)
    ▼
1. 解析 BlobKey → Volume + Offset + Size
2. 从对应的 BlobNode 读取数据 Shard
3. 如果某些 Shard 不可用 → 读取校验 Shard + 解码恢复
4. 返回完整数据
```

## 与主存储系统的集成（冷热分层）

BlobStore 通过生命周期管理 (LcNode) 与主存储集成：

```
热数据 (DataNode 副本存储)
    │
    │ LcNode 扫描 → 满足迁移规则
    ▼
冷数据 (BlobStore 纠删码存储)
    │
    │ 读取时
    ▼
FlashNode 缓存加速 → Client
```

文件的 Inode 中同时支持副本存储的 ExtentKey 和纠删码存储的 ObjExtentKey。
