# ObjectNode S3 网关概述

## 模块定位

ObjectNode (`objectnode/`) 提供 S3 兼容的对象存储接口，将 S3 API 请求映射到 CubeFS 的文件系统操作。

## S3 → CubeFS 映射关系

| S3 概念 | CubeFS 概念 |
|---------|------------|
| Bucket | Volume（卷） |
| Object Key | 文件路径 |
| Object Data | 文件内容（通过 ExtentClient 读写） |
| Object Metadata | Inode 属性 + XAttr |
| ACL | XAttr 存储 |
| Policy | XAttr 存储 |
