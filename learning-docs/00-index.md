# CubeFS 学习笔记索引

本文档库旨在系统梳理 CubeFS 各模块的架构设计与实现细节，帮助从零开始深入理解整个分布式存储系统。

## 学习进度

| 阶段 | 主题 | 状态 |
|------|------|------|
| 阶段一 | [全局认知](01-overview/) | 完成 |
| 阶段二 | [Master](02-master/) / [MetaNode](03-metanode/) / [DataNode](04-datanode/) | 完成 |
| 阶段三 | [SDK](05-sdk/) / [FUSE Client](06-client/) | 完成 |
| 阶段四 | [BlobStore](07-blobstore/) / [ObjectNode](08-objectnode/) / [辅助服务](09-auxiliary/) | 完成 |
| 阶段五 | [基础设施](10-infrastructure/) | 完成 |
| 阶段六 | [端到端流程](11-e2e-flows/) | 完成 |

## 目录结构

```
learning-docs/
├── 00-index.md                    # 本文件
├── 01-overview/                   # 阶段一：全局认知
│   ├── architecture.md            # 架构总览
│   ├── protocol.md                # 协议层梳理
│   └── build-and-deploy.md        # 构建与部署
├── 02-master/                     # Master 模块
│   ├── README.md
│   ├── startup-flow.md
│   ├── cluster-management.md
│   ├── partition-management.md
│   ├── raft-fsm.md
│   └── api.md
├── 03-metanode/                   # MetaNode 模块
│   ├── README.md
│   ├── data-structures.md
│   ├── raft-fsm.md
│   ├── partition-split.md
│   └── transaction.md
├── 04-datanode/                   # DataNode 模块
│   ├── README.md
│   ├── extent-store.md
│   ├── replication.md
│   ├── repair.md
│   └── tiny-vs-normal.md
├── 05-sdk/                        # SDK 层
│   ├── README.md
│   ├── meta-wrapper.md
│   ├── extent-client.md
│   └── master-client.md
├── 06-client/                     # FUSE Client
│   ├── README.md
│   ├── fuse-mount.md
│   └── caching.md
├── 07-blobstore/                  # BlobStore 纠删码子系统
│   ├── README.md
│   ├── erasure-coding.md
│   ├── data-model.md
│   └── components.md
├── 08-objectnode/                 # ObjectNode S3 网关
│   ├── README.md
│   └── s3-mapping.md
├── 09-auxiliary/                  # 辅助服务
│   ├── authnode.md
│   ├── lcnode.md
│   ├── flashnode.md
│   └── console.md
├── 10-infrastructure/             # 基础设施
│   ├── raft.md
│   ├── communication.md
│   ├── observability.md
│   └── utilities.md
└── 11-e2e-flows/                  # 端到端流程
    ├── write-flow.md
    ├── read-flow.md
    ├── failure-recovery.md
    ├── data-tiering.md
    └── scaling.md
```

## 项目信息

- **项目地址**: https://github.com/cubefs/cubefs
- **官方文档**: https://cubefs.io
- **License**: Apache 2.0
- **CNCF 状态**: Graduated
