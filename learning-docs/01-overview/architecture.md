# CubeFS 架构总览

## 项目简介

CubeFS 是 CNCF 毕业级云原生分布式存储系统，支持 S3、POSIX、HDFS 三种访问协议，提供副本存储和纠删码存储两种数据引擎。项目使用 Go 语言编写，核心依赖 Raft 共识协议和 RocksDB 持久化存储。

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                  │
│  ┌─────────┐ ┌───────────┐ ┌──────────┐ ┌─────┐ ┌──────────┐  │
│  │FUSE     │ │ObjectNode │ │Java SDK  │ │ CLI │ │Web       │  │
│  │Client   │ │(S3 网关)   │ │(libcfs)  │ │     │ │Console   │  │
│  └────┬────┘ └─────┬─────┘ └────┬─────┘ └──┬──┘ └────┬─────┘  │
└───────┼────────────┼────────────┼───────────┼─────────┼────────┘
        │            │            │           │         │
┌───────┼────────────┼────────────┼───────────┼─────────┼────────┐
│       ▼            ▼            ▼           ▼         ▼        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Master (资源管理节点)                     │   │
│  │  · 集群元数据管理 (Raft + RocksDB)                         │   │
│  │  · 卷 (Volume) 管理                                       │   │
│  │  · 数据分区 / 元数据分区调度                                 │   │
│  │  · 节点心跳与故障检测                                       │   │
│  │  · 拓扑管理 (Zone / NodeSet)                              │   │
│  └────────────┬──────────────────────┬─────────────────────┘   │
│               │                      │                         │
│       ┌───────▼───────┐      ┌───────▼───────┐                │
│       │  元数据子系统    │      │  数据子系统      │                │
│       │               │      │               │                │
│       │  ┌──────────┐ │      │  ┌──────────┐ │                │
│       │  │MetaNode  │ │      │  │DataNode  │ │                │
│       │  │(多个实例)  │ │      │  │(多个实例)  │ │                │
│       │  │          │ │      │  │          │ │                │
│       │  │·Multi-Raft│ │      │  │·Raft 副本 │ │                │
│       │  │·Inode     │ │      │  │·Extent   │ │                │
│       │  │ B-Tree   │ │      │  │ Store    │ │                │
│       │  │·Dentry   │ │      │  │·Tiny/    │ │                │
│       │  │ B-Tree   │ │      │  │ Normal   │ │                │
│       │  └──────────┘ │      │  └──────────┘ │                │
│       └───────────────┘      │               │                │
│                              │  ┌──────────┐ │                │
│                              │  │BlobStore │ │                │
│                              │  │(纠删码)   │ │                │
│                              │  │·Access   │ │                │
│                              │  │·BlobNode │ │                │
│                              │  │·Cluster  │ │                │
│                              │  │ Mgr      │ │                │
│                              │  │·Scheduler│ │                │
│                              │  │·Proxy    │ │                │
│                              │  └──────────┘ │                │
│                              └───────────────┘                │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐    │
│  │                    辅助服务                              │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │    │
│  │  │AuthNode  │  │LcNode    │  │FlashNode │            │    │
│  │  │(认证服务)  │  │(生命周期)  │  │(分布式缓存)│            │    │
│  │  └──────────┘  └──────────┘  └──────────┘            │    │
│  └───────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

## 核心组件详解

### 1. Master（资源管理节点）

**代码位置**: `master/`

Master 是整个集群的"大脑"，负责全局资源管理：

- **集群元数据管理**: 通过 Raft 一致性协议保证多 Master 节点间的元数据一致性，持久化到 RocksDB
- **卷 (Volume) 管理**: 创建、删除、扩容卷，每个卷由若干 MetaPartition 和 DataPartition 组成
- **分区调度**: 创建和管理 MetaPartition（元数据分区）、DataPartition（数据分区），负责副本放置策略
- **节点管理**: 通过心跳监控所有 DataNode、MetaNode 的健康状态，实现故障检测和自动修复
- **拓扑管理**: 维护 Zone（可用区）和 NodeSet（节点集合）的拓扑结构，确保副本跨故障域分布
- **负载均衡**: 监控各节点负载，触发数据迁移实现均衡

**关键结构体** (`master/server.go`):
```go
type Server struct {
    id          uint64
    clusterName string
    cluster     *Cluster     // 集群核心状态
    fsm         *MetadataFsm // Raft 状态机
    partition   raftstore.Partition
    raftStore   raftstore.RaftStore
    // ...
}
```

### 2. MetaNode（元数据节点）

**代码位置**: `metanode/`

MetaNode 负责管理文件系统的元数据：

- **数据结构**: 每个 MetaPartition 包含两棵内存 B-Tree：
  - **InodeBTree**: 存储 inode 信息（文件大小、权限、时间戳、Extent Key 列表等）
  - **DentryBTree**: 存储目录项（ParentId + Name -> Inode 的映射）
- **Multi-Raft**: 每个 MetaPartition 运行独立的 Raft Group，保证元数据的强一致性
- **分区范围**: 每个 MetaPartition 管理一个 Inode ID 区间 [Start, End)
- **分裂机制**: 当 MetaPartition 的 inode 数量超过阈值时，自动分裂为两个分区

**关键操作**:
- `CreateInode`: 创建 inode（文件/目录）
- `CreateDentry`: 创建目录项（将名字映射到 inode）
- `Lookup`: 按名字查找 inode
- `ReadDir`: 列出目录内容
- `ExtentsAdd`: 关联数据 Extent Key 到 inode

### 3. DataNode（数据节点）

**代码位置**: `datanode/`

DataNode 负责实际数据的存储：

- **Extent 存储**: 数据以 Extent（数据块）为单位存储在本地磁盘上
  - **Tiny Extent** (ID 1-64): 小文件（<128KB），多个小文件追加写入同一个 Extent 文件
  - **Normal Extent** (ID >= 1024): 大文件，每个 Extent 独立存储
- **Raft 副本**: 每个 DataPartition 运行独立的 Raft Group，写入操作先提交 Raft 再持久化
- **磁盘管理**: SpaceManager 管理多块磁盘，Disk 对象抽象单块磁盘的分区和空间分配
- **副本协议**: 通过 `repl` 包实现客户端到 Leader DataNode 再到 Follower 的链式复制

### 4. FUSE Client（POSIX 客户端）

**代码位置**: `client/`

FUSE Client 提供 POSIX 文件系统语义：

- **挂载流程**: 解析配置 -> 连接 Master 获取视图 -> 初始化 MetaWrapper + ExtentClient -> FUSE Mount
- **SDK 层** (`sdk/`):
  - **MetaWrapper**: 维护 MetaPartition 视图，将请求路由到正确的 MetaNode
  - **ExtentClient**: 管理每个 inode 的 Streamer，负责数据读写
- **缓存**: InodeCache（inode 缓存）和 DentryCache（目录项缓存）减少网络请求

### 5. BlobStore（纠删码子系统）

**代码位置**: `blobstore/`

BlobStore 提供纠删码（Erasure Coding）存储能力，用于冷数据存储：

- **数据模型**: Volume -> Chunk -> Shard -> Blob 四层结构
- **Reed-Solomon 编码**: 将数据分片并生成校验片，提供高可靠低成本存储
- **核心组件**:
  - **Access**: 数据访问入口（Put/Get/Delete）
  - **ClusterMgr**: 集群元数据管理
  - **BlobNode**: 实际数据存储节点
  - **Scheduler**: 后台任务调度（数据修复、均衡、巡检）
  - **Proxy**: 代理层（分配、缓存、负载均衡）

### 6. ObjectNode（S3 网关）

**代码位置**: `objectnode/`

ObjectNode 提供 S3 兼容的对象存储接口：

- **映射关系**: S3 Bucket = CubeFS Volume, Object Key = 文件路径
- **支持功能**: PutObject, GetObject, ListObjects, Multipart Upload, ACL, Policy
- **认证**: 支持 AWS Signature V2/V4

### 7. 辅助服务

| 服务 | 代码位置 | 职责 |
|------|---------|------|
| AuthNode | `authnode/` | Kerberos 风格票据认证，Raft 复制 keystore |
| LcNode | `lcnode/` | S3 生命周期规则执行，冷热数据迁移 |
| FlashNode | `remotecache/` | 分布式读缓存，加速冷数据访问 |
| Console | `console/` | Web 管理界面，GraphQL API |

## 模块间通信

| 通信路径 | 协议 | 说明 |
|---------|------|------|
| Client -> Master | HTTP | 获取 MetaPartition/DataPartition 视图 |
| Client -> MetaNode | TCP (自定义 Packet) | 元数据操作（inode/dentry CRUD） |
| Client -> DataNode | TCP (Repl Protocol) | 数据读写 |
| Master -> MetaNode/DataNode | TCP (AdminTask) | 管理任务下发 |
| MetaNode/DataNode -> Master | HTTP (心跳) | 节点状态上报 |
| ObjectNode -> MetaNode/DataNode | TCP | 同 FUSE Client 路径 |

## Raft 使用模式

CubeFS 中 Raft 被广泛使用，但使用方式各不相同：

| 组件 | Raft Group 数量 | 状态机存储 | 用途 |
|------|----------------|-----------|------|
| Master | 1 个 | RocksDB | 集群全局元数据 |
| MetaNode | 每个 MetaPartition 一个 (Multi-Raft) | 内存 B-Tree | inode/dentry 元数据 |
| DataNode | 每个 DataPartition 一个 (Multi-Raft) | Extent 文件 | 数据写入一致性 |
| AuthNode | 1 个 | RocksDB | 认证 keystore |

## Volume（卷）概念

Volume 是 CubeFS 的核心逻辑单元：

- 从 POSIX 视角看，Volume 是一个可挂载的文件系统实例
- 从 S3 视角看，Volume 对应一个 Bucket
- 一个 Volume 包含：
  - 若干 **MetaPartition**（分布在不同 MetaNode 上）
  - 若干 **DataPartition**（分布在不同 DataNode 上）
- 支持多客户端同时挂载同一 Volume

## 代码入口

所有服务节点编译为同一个二进制文件 `cfs-server`，通过配置文件中的 `role` 字段决定运行角色：

```go
// cmd/cmd.go
switch role {
case "master":    server = master.NewServer()
case "metanode":  server = metanode.NewServer()
case "datanode":  server = datanode.NewServer()
case "authnode":  server = authnode.NewServer()
case "objectnode": server = objectnode.NewServer()
case "lcnode":    server = lcnode.NewServer()
case "flashnode": server = flashnode.NewFlashNode()
// ...
}
```

FUSE Client 编译为独立二进制文件 `cfs-client`，入口在 `client/fuse.go`。
