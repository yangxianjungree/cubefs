# DataNode 模块概述

## 模块定位

DataNode 负责实际数据的存储和管理。每个 DataNode 管理本地磁盘上的多个 DataPartition，每个 DataPartition 通过 Raft 保证多副本一致性。数据以 Extent（数据块）为基本存储单元。

## 核心数据结构

```
DataNode
  └── SpaceManager
       └── disks map[string]*Disk
            └── Disk
                 └── partitionMap map[uint64]*DataPartition
                      └── DataPartition
                           ├── ExtentStore    // Extent 存储引擎
                           ├── raftPartition  // Raft 实例
                           └── replicas       // 副本列表
```

## 代码结构

```
datanode/
├── server.go                  # DataNode 启动/停止
├── partition.go               # DataPartition 结构和生命周期
├── partition_raftfsm.go       # Raft FSM
├── partition_raft.go          # Raft 操作
├── partition_op_by_raft.go    # 通过 Raft 的操作
├── space_manager.go           # 磁盘空间管理
├── disk.go                    # 磁盘抽象
├── server_handler.go          # HTTP 处理器
├── data_partition_repair.go   # 数据修复
├── storage/
│   ├── extent_store.go        # Extent 存储引擎
│   ├── extent.go              # Extent 文件操作
│   └── extent_cache.go        # Extent 缓存
└── repl/
    └── repl_protocol.go       # 副本协议
```

## 通信方式

| 通信对端 | 协议 | 说明 |
|---------|------|------|
| Client/SDK | TCP (Repl Protocol) | 数据读写 |
| Master | HTTP | 心跳上报 |
| 其他 DataNode | TCP | 副本同步 / 修复 |
| 其他 DataNode | Raft | 副本一致性 |
