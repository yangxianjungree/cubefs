# Web Console

## 概述

Console (`console/`) 提供 Web 管理界面，基于 GraphQL API 和静态前端。

## 架构

```
浏览器
    │ HTTP
    ▼
ConsoleNode
    ├── 静态资源 (index.html, JS, CSS) → SPA 前端
    ├── GraphQL API:
    │   ├── /api/cluster → Master ClusterAPI
    │   ├── /api/user → Master UserAPI
    │   └── /api/volume → Master VolumeAPI
    ├── /login → LoginService (认证)
    ├── /cfs_monitor → MonitorService (Prometheus/Grafana)
    ├── /file → FileService (文件浏览/上传/下载)
    └── /iql → GraphQL 交互式查询
```

## 功能

| 功能 | 路由 | 说明 |
|------|------|------|
| 集群概览 | `/overview` | 集群状态、容量、节点数 |
| 卷管理 | `/volumeList` | 创建/删除/配置卷 |
| 用户管理 | — | 用户 CRUD |
| 监控 | `/cfs_monitor` | 跳转 Prometheus/Grafana |
| 文件浏览 | `/file` | 通过 S3 协议浏览/上传/下载文件 |
| 登录 | `/login` | 用户认证 |

## 实现

ConsoleNode 本质上是一个反向代理 + 静态文件服务器：
- GraphQL 请求代理到 Master
- 文件操作通过 ObjectNode 的 S3 接口
- 监控通过配置的 Prometheus/Grafana 地址
- 前端使用嵌入的静态资源（Go embed）
