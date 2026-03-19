# S3 API 映射详解

## 架构

```
S3 Client (aws-sdk, s3cmd, ...)
    │ HTTP (S3 Protocol)
    ▼
ObjectNode
    ├── Auth Middleware (Signature V2/V4, STS)
    ├── Policy Middleware (Bucket/Object Policy)
    ├── CORS Middleware
    ├── Gorilla mux Router
    │   ├── /{bucket} → Bucket 操作
    │   └── /{bucket}/{object:.+} → Object 操作
    ▼
VolumeManager → Volume
    ├── MetaWrapper → MetaNode (元数据)
    ├── ExtentClient → DataNode (热数据)
    └── BlobStoreClient → BlobStore (冷数据)
```

## 核心 API 映射

### Object 操作

| S3 API | HTTP 方法 | 路径 | CubeFS 操作 |
|--------|----------|------|------------|
| PutObject | PUT | `/{bucket}/{key}` | CreateInode → CreateDentry → Write |
| GetObject | GET | `/{bucket}/{key}` | Lookup → InodeGet → Read |
| HeadObject | HEAD | `/{bucket}/{key}` | Lookup → InodeGet |
| DeleteObject | DELETE | `/{bucket}/{key}` | Delete (Unlink + EvictInode) |
| CopyObject | PUT + x-amz-copy-source | `/{bucket}/{key}` | Read src → Write dst |

### Bucket 操作

| S3 API | HTTP 方法 | 路径 | CubeFS 操作 |
|--------|----------|------|------------|
| CreateBucket | PUT | `/{bucket}` | Master CreateVolume |
| DeleteBucket | DELETE | `/{bucket}` | Master DeleteVolume |
| HeadBucket | HEAD | `/{bucket}` | GetVolume |
| ListObjectsV1/V2 | GET | `/{bucket}` | ReadDir 递归遍历 |

### Multipart Upload

| S3 API | CubeFS 操作 |
|--------|------------|
| CreateMultipartUpload | MetaNode CreateMultipart |
| UploadPart | 写入临时数据 |
| CompleteMultipartUpload | MetaNode CompleteMultipart → 合并 |
| AbortMultipartUpload | MetaNode RemoveMultipart |
| ListParts | MetaNode GetMultipart |

## 认证

ObjectNode 支持多种 S3 认证方式：

| 认证方式 | 说明 |
|---------|------|
| Signature V2 | 旧版签名（Authorization 头） |
| Signature V4 | AWS4-HMAC-SHA256 签名 |
| Presigned URL | URL 中携带签名参数 |
| STS | 临时安全凭证 |

认证流程：
1. 解析 Authorization 头或 URL 参数
2. 提取 Access Key
3. 从 Master 获取 Secret Key
4. 验证签名

## Volume 管理

```go
type Volume struct {
    mw    *meta.MetaWrapper     // 元数据客户端
    ec    *stream.ExtentClient  // 热数据客户端
    store Store                 // 存储抽象
    name  string
}
```

VolumeManager 维护 Bucket Name → Volume 的映射，Volume 初始化时创建 MetaWrapper 和 ExtentClient。

## PutObject 流程详解

```
S3 PutObject 请求
    ▼
1. 认证和授权检查
2. 解析 Object Key → 文件路径
3. 创建中间目录（如有必要）
4. vol.PutFile():
   · Lookup 父目录
   · 如果文件存在 → 截断
   · 如果不存在 → CreateInode + CreateDentry
   · 分块读取请求体 → ec.Write
   · ec.Flush
5. 设置对象元数据（Content-Type, 自定义 headers → XAttr）
6. 返回 ETag
```
