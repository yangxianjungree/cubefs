# MasterClient SDK

## 概述

MasterClient (`sdk/master/client.go`) 封装了与 Master 节点的 HTTP 通信，支持 Leader 自动发现和故障切换。

## 核心结构体

```go
type MasterClient struct {
    masters    []string        // Master 节点地址列表
    leaderAddr string          // 当前 Leader 地址
    timeout    time.Duration
    client     *http.Client

    adminAPI  *AdminAPI   // 管理 API
    clientAPI *ClientAPI  // 客户端 API
    nodeAPI   *NodeAPI    // 节点管理 API
    userAPI   *UserAPI    // 用户管理 API
}
```

## Leader 发现

```go
func (mc *MasterClient) serveRequest(r *request) (repsData []byte, err error) {
    // 1. 获取 Leader 和所有 Master 地址
    // 2. 先尝试 Leader
    // 3. 如果失败（连接错误、超时）→ 轮询其他 Master
    // 4. 如果返回 403 → 解析新 Leader 地址，更新 leaderAddr
    // 5. 解压响应（gzip），解析 HTTPReplyRaw
    // 6. 检查 code == 0，返回 data
}
```

## ClientAPI（客户端使用）

| 方法 | 路径 | 说明 |
|------|------|------|
| `GetDataPartitions` | `/client/partitions` | 获取 DP 列表 |
| `GetMetaPartitions` | `/client/metaPartitions` | 获取 MP 列表 |
| `GetVolume` | `/client/vol` | 获取卷信息 |

## AdminAPI（管理使用）

| 方法 | 路径 | 说明 |
|------|------|------|
| `GetClusterInfo` | `/admin/getCluster` | 获取集群信息 |
| `CreateVolume` | `/admin/createVol` | 创建卷 |
| `DeleteVolume` | `/vol/delete` | 删除卷 |
| `GetVolumeSimpleInfo` | `/admin/getVol` | 获取卷简要信息 |

## 请求重试策略

```
尝试 Leader → 失败 → 尝试 Master-2 → 失败 → 尝试 Master-3
                         ↑ 403 → 更新 Leader
```

所有请求都会自动重试所有已知的 Master 节点，确保高可用。
