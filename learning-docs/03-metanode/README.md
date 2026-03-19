# MetaNode 模块概述

## 模块定位

MetaNode 负责管理文件系统的元数据，包括 Inode（文件/目录的属性）和 Dentry（目录项/名字映射）。每个 MetaNode 上运行多个 MetaPartition，每个 MetaPartition 是一个独立的 Raft Group，管理一个 Inode ID 范围内的元数据。

## 核心数据结构

```
MetaNode
  └── metadataManager
       └── partitions map[uint64]MetaPartition
            └── metaPartition
                 ├── inodeTree    (BTree)   // Inode ID -> Inode
                 ├── dentryTree   (BTree)   // (ParentID, Name) -> Dentry
                 ├── extendTree   (BTree)   // XAttr 扩展属性
                 ├── multipartTree (BTree)  // Multipart 上传
                 ├── txProcessor            // 分布式事务
                 ├── raftPartition          // Raft 实例
                 └── freeList               // 待回收 inode 列表
```

## 代码结构

```
metanode/
├── metanode.go           # MetaNode 主结构体
├── server.go             # TCP/Smux 服务器
├── manager.go            # metadataManager 分区管理
├── manager_op.go         # 操作处理分发
├── partition.go          # MetaPartition 结构和生命周期
├── partition_fsm.go      # Raft FSM Apply 逻辑
├── partition_fsmop.go    # FSM 操作实现
├── partition_op*.go      # 各类操作处理器
├── inode.go              # Inode 结构体和操作
├── dentry.go             # Dentry 结构体和操作
├── btree.go              # B-Tree 封装
├── sorted_extents.go     # Extent Key 排序管理
├── transaction.go        # 分布式事务
├── multipart.go          # Multipart 上传
├── api_handler.go        # HTTP API
└── ...
```

## 通信方式

| 通信对端 | 协议 | 说明 |
|---------|------|------|
| Client/SDK | TCP Packet | 元数据 CRUD 操作 |
| Client/SDK | Smux | 多路复用（可选） |
| Master | HTTP | 心跳上报、获取卷信息 |
| 其他 MetaNode | Raft | 副本间数据同步 |

## 关键流程

1. **写操作**: Client → MetaNode Leader → Raft Submit → 多数确认 → FSM Apply → B-Tree 更新 → 响应客户端
2. **读操作**: Client → MetaNode（Leader 或 Follower Read）→ B-Tree 查询 → 响应
3. **快照**: 定期将 B-Tree 序列化到磁盘，用于 Raft 快照和故障恢复
