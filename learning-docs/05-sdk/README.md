# SDK 层概述

## 模块定位

SDK 层位于 `sdk/` 目录，是客户端（FUSE Client、ObjectNode 等）访问 CubeFS 集群的核心库。它封装了与 Master、MetaNode、DataNode 的所有通信逻辑。

## 三个核心组件

```
SDK Layer
├── MetaWrapper  (sdk/meta/)     → 元数据操作
├── ExtentClient (sdk/data/)     → 数据读写
└── MasterClient (sdk/master/)   → Master 通信
```

## 组件交互

```
FUSE Client / ObjectNode
    │
    ├── MetaWrapper
    │   ├── 维护 MetaPartition 视图 (B-Tree by Start)
    │   ├── 按 Inode ID 路由到正确的 MP
    │   ├── TCP Packet 协议与 MetaNode 通信
    │   └── 连接池管理
    │
    ├── ExtentClient
    │   ├── 每个 Inode 一个 Streamer（流管理器）
    │   ├── 写入：ExtentHandler → DataPartition → 追加/覆盖
    │   ├── 读取：ExtentReader → DataPartition → 流式读
    │   └── 支持 BCache / RemoteCache 加速
    │
    └── MasterClient
        ├── HTTP 请求 Master Leader
        ├── 自动 Leader 切换
        └── 获取 Volume/DP/MP 视图
```
