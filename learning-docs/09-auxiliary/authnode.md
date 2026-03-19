# AuthNode 认证服务

## 概述

AuthNode (`authnode/`) 提供 Kerberos 风格的票据认证服务，基于 Raft + RocksDB 保证高可用。

## 架构

```
Client → AuthNode Leader ← Raft → AuthNode Follower(s)
                 │
                 ▼
            RocksDB (keystore)
```

## 认证流程

### 获取票据 (getTicket)

```
Client 发送 AuthGetTicketReq:
    · ClientID: 客户端标识
    · ServiceID: 目标服务标识
    · Verifier: 加密的验证信息（用客户端密钥加密）
    ▼
AuthNode:
    1. extractClientReqInfo → 解析请求
    2. getSecretKey(clientID) → 从 keystore 获取客户端密钥
    3. ParseVerifier(verifier, clientKey) → 解密验证
    4. genGetTicketAuthResp:
       · 生成 SessionKey
       · 构建 Ticket (ClientID, ServiceID, SessionKey, IP, Caps, Expiration)
       · 用服务端密钥加密 Ticket
       · 用 SessionKey 加密响应
    5. 返回加密的 Ticket + SessionKey
```

### 使用票据

```
Client 访问目标服务时:
    1. 在请求中携带 Ticket
    2. 服务端用自己的密钥解密 Ticket
    3. 验证 Ticket 有效性（时间、IP、权限）
    4. 允许/拒绝请求
```

## 密钥管理

| 操作 | API | 说明 |
|------|-----|------|
| 创建密钥 | `AdminCreateKey` | 为用户/服务创建密钥对 |
| 删除密钥 | `AdminDeleteKey` | 删除密钥 |
| 获取密钥 | `AdminGetKey` | 获取密钥信息 |
| 添加权限 | `AdminAddCaps` | 为密钥添加权限 |
| 删除权限 | `AdminDeleteCaps` | 删除权限 |

## Raft 状态机

```go
type KeystoreFsm struct {
    keystore       *Keystore      // 用户密钥存储
    accessKeystore *AccessKeystore // Access Key 存储
    store          *RocksDBStore   // RocksDB 持久化
}
```

所有密钥操作通过 Raft 提交，确保多节点一致性。
