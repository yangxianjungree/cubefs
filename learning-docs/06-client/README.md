# FUSE Client 概述

## 模块定位

FUSE Client (`client/`) 提供 POSIX 文件系统语义，通过 Linux FUSE 接口将 CubeFS 卷挂载为本地文件系统。

## 架构

```
用户空间进程 (ls, cat, cp, ...)
    │ POSIX 系统调用
    ▼
Linux VFS
    │
    ▼
FUSE 内核模块 (/dev/fuse)
    │
    ▼
cfs-client (FUSE 用户态守护进程)
    │
    ├── Super (超级块)
    │   ├── InodeCache (inode 缓存)
    │   ├── DentryCache (目录项缓存)
    │   ├── MetaWrapper → MetaNode (TCP)
    │   └── ExtentClient → DataNode (TCP)
    │
    └── Master (HTTP) ← 获取集群视图
```

## 挂载流程

```
cfs-client -c config.json
    ▼
1. parseMountOption(cfg) → 解析配置
2. loadConfFromMaster(opt) → 从 Master 获取卷信息
3. checkPermission(opt) → ACL 权限检查
4. NewSuper(opt):
   · meta.NewMetaWrapper() → 初始化元数据 SDK
   · stream.NewExtentClient() → 初始化数据 SDK
   · NewInodeCache() → inode 缓存
   · NewDcache() → dentry 缓存
5. fuse.Mount(mountPoint, ...) → FUSE 挂载
6. fs.Serve(conn, super, ...) → 开始服务
```

## 核心结构

### Super（超级块）

```go
type Super struct {
    ic  *InodeCache          // inode 缓存
    dc  *Dcache              // dentry 缓存（全局）
    mw  *meta.MetaWrapper    // 元数据 SDK
    ec  *stream.ExtentClient // 数据 SDK
    nodeCache map[uint64]fs.Node  // inode → FUSE Node 映射
    ebsc *blobstore.BlobStoreClient  // BlobStore 客户端（冷卷）
    bc   *bcache.BcacheClient        // 块缓存客户端
}
```

### File（文件节点）

```go
type File struct {
    super    *Super
    info     *proto.InodeInfo
    parentIno uint64
    name     string
}
```

### Dir（目录节点）

```go
type Dir struct {
    super    *Super
    info     *proto.InodeInfo
    dcache   *DentryCache  // 目录级 dentry 缓存
    parentIno uint64
    name     string
}
```
