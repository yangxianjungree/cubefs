# Master HTTP API

## 概述

Master 通过 gorilla/mux 提供 HTTP RESTful API，注册在 `api_service.go` 的 `registerAPIRoutes()` 中。API 分为以下类别。

## 请求处理流程

```
HTTP 请求
    ▼
中间件链:
    ├── API 限流器 (apiLimiter)
    ├── Leader 检查:
    │   · 写请求 → 必须在 Leader 执行
    │   · 读请求 → Leader 或 Follower (followerRead)
    │   · 非 Leader → 代理转发到 Leader (reverseProxy)
    └── 认证 (parseAndCheckClientIDKey)
    ▼
Handler 处理
    ▼
JSON 响应
```

## 集群管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/admin/getCluster` | 获取集群信息 |
| GET | `/cluster/stat` | 集群统计（总空间/已用/节点数等） |
| POST | `/cluster/freeze` | 冻结/解冻集群（禁止自动分配） |
| POST | `/admin/setClusterInfo` | 设置集群信息 |
| GET | `/admin/getIp` | 获取 Master IP |
| GET | `/admin/getClusterValue` | 获取集群配置值 |

## 卷管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/admin/createVol` | 创建卷 |
| GET | `/admin/getVol` | 获取卷详情 |
| GET | `/vol/list` | 列出所有卷 |
| POST | `/vol/delete` | 删除卷 |
| POST | `/vol/update` | 更新卷配置 |
| POST | `/vol/expand` | 扩容卷 |
| POST | `/vol/shrink` | 缩容卷 |
| POST | `/vol/forbidden` | 禁止/允许卷操作 |
| POST | `/vol/auditlog` | 启用/禁用审计日志 |

## 数据分区 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/dataPartition/get` | 获取 DP 信息 |
| POST | `/dataPartition/create` | 创建 DP |
| POST | `/dataPartition/load` | 加载 DP |
| POST | `/dataPartition/decommission` | 下线 DP |
| GET | `/dataPartition/diagnose` | 诊断 DP |
| POST | `/dataPartition/changeleader` | 变更 DP Leader |
| POST | `/dataReplica/add` | 添加数据副本 |
| POST | `/dataReplica/delete` | 删除数据副本 |

## 元数据分区 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/metaPartition/create` | 创建 MP |
| POST | `/metaPartition/load` | 加载 MP |
| POST | `/metaPartition/decommission` | 下线 MP |
| POST | `/metaPartition/changeleader` | 变更 MP Leader |
| POST | `/metaReplica/add` | 添加元数据副本 |
| POST | `/metaReplica/delete` | 删除元数据副本 |

## 节点管理 API

### DataNode
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/dataNode/add` | 添加 DataNode |
| POST | `/dataNode/decommission` | 下线 DataNode |
| POST | `/dataNode/migrate` | 迁移 DataNode |
| GET | `/admin/cluster/getAllDataNodes` | 获取所有 DataNode |

### MetaNode
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/metaNode/add` | 添加 MetaNode |
| POST | `/metaNode/decommission` | 下线 MetaNode |
| POST | `/metaNode/migrate` | 迁移 MetaNode |
| GET | `/admin/cluster/getAllMetaNodes` | 获取所有 MetaNode |

### 磁盘管理
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/disk/decommission` | 下线磁盘 |
| POST | `/disk/recommission` | 重新上线磁盘 |

## 客户端 API

客户端（FUSE Client / SDK）使用的 API：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/client/partitions` | 获取 DP 列表（客户端定期刷新） |
| GET | `/client/vol` | 获取卷信息 |
| GET | `/client/metaPartitions` | 获取 MP 列表 |
| GET | `/client/metaPartition` | 获取单个 MP 信息 |

## 用户管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/user/create` | 创建用户 |
| POST | `/user/delete` | 删除用户 |
| POST | `/user/update` | 更新用户 |
| POST | `/user/updatePolicy` | 更新用户策略 |
| POST | `/user/transferVol` | 转移卷所有权 |

## QoS API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/qos/update` | 更新 QoS 策略 |
| POST | `/qos/updateZoneLimit` | 更新 Zone 级限流 |
| POST | `/qos/updateMasterLimit` | 更新 Master 级限流 |

## GraphQL API

| 路径 | 说明 |
|------|------|
| `/api/cluster` | 集群管理 GraphQL |
| `/api/user` | 用户管理 GraphQL |
| `/api/volume` | 卷管理 GraphQL |

## Follower Read

部分读请求支持从 Follower 节点直接返回（减轻 Leader 压力）：

- `/client/partitions` - 客户端获取 DP 列表
- 使用 `followerReadManager` 维护缓存
- 定期从 Leader 同步最新数据
