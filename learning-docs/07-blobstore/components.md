# BlobStore 组件职责

## Access（数据访问网关）

**入口**: `blobstore/access/service.go`

核心 API:
- `Put(data)` → 编码 + 分片写入 BlobNode
- `Get(blobKey)` → 从 BlobNode 读取 + 解码
- `Delete(blobKey)` → 删除 Blob
- `Alloc()` → 通过 Proxy 分配空间
- `Sign(location)` → 签名访问凭证

Access 是无状态的，可水平扩展。

## ClusterMgr（集群元数据管理）

**入口**: `blobstore/clustermgr/service.go`

职责:
- 管理 Volume、Disk、BlobNode、ShardNode 元数据
- 通过 Raft 保证多副本一致性
- 提供服务注册和发现
- 管理路由表（数据如何分布）

API 分类:
- 服务管理: `ServiceRegister`, `ServiceGet`
- Volume 管理: Volume CRUD
- Disk 管理: Disk 状态
- Route 管理: 数据分布路由

## BlobNode（数据存储节点）

**入口**: `blobstore/blobnode/svr.go`

职责:
- 存储实际的 Chunk/Shard 数据
- 处理数据读写请求
- 定期向 ClusterMgr 发送心跳
- Chunk 的创建、读取、删除

每个 BlobNode 管理本地磁盘，每块磁盘上存储多个 Chunk。

## Scheduler（后台任务调度）

**入口**: `blobstore/scheduler/startup.go`

职责:
- **Shard 修复**: 从 Kafka 消费修复任务，重建丢失的 Shard
- **Blob 删除**: 从 Kafka 消费删除任务，清理已标记删除的 Blob
- **数据巡检**: 定期检查数据完整性
- **负载均衡**: 在 BlobNode 间迁移数据

## Proxy（代理层）

**入口**: `blobstore/proxy/service.go`

职责:
- **空间分配**: 为写入请求分配 Volume 空间
- **缓存**: 缓存 Volume 信息，减少 ClusterMgr 压力
- **消息队列**: Shard 修复和 Blob 删除的 MQ (Kafka) 管理
- **负载均衡**: 请求分发

## ShardNode（分片管理）

**入口**: `blobstore/shardnode/svr.go`

职责:
- 分片级别的元数据管理
- 基于 Raft 的分片复制
- Blob/Item 的 CRUD 操作
- 分片生命周期管理

## 组件间通信

```
Access ──HTTP──> Proxy (分配)
Access ──HTTP──> BlobNode (数据I/O)
Access ──HTTP──> ClusterMgr (路由)

Proxy ──HTTP──> ClusterMgr (Volume信息)

Scheduler ──Kafka──> (shard_repair, blob_delete)
Scheduler ──HTTP──> BlobNode (修复/删除)
Scheduler ──HTTP──> ClusterMgr (集群拓扑)

BlobNode ──HTTP──> ClusterMgr (心跳)
ShardNode ──Raft──> ShardNode (副本同步)
```
