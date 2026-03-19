# MetaWrapper 元数据 SDK

## 概述

MetaWrapper (`sdk/meta/meta.go`) 是客户端访问元数据的核心组件，负责：
- 维护 MetaPartition 视图（从 Master 定期刷新）
- 按 Inode ID 路由请求到正确的 MetaPartition
- 管理与 MetaNode 的 TCP 连接池

## 核心结构体

```go
type MetaWrapper struct {
    volname     string
    owner       string
    masterClient *masterSDK.MasterClient

    // 分区管理
    partitions   map[uint64]*MetaPartition  // ID -> MP
    ranges       *btree.BTree              // 按 Start 排序的 B-Tree
    rwPartitions []*MetaPartition           // 可写分区列表

    // 连接管理
    conns    *util.ConnectPool

    // 缓存
    dirCache map[uint64]dirInfoCache
    qc       *QuotaCache

    // 配置
    totalSize, usedSize uint64
}
```

## 分区路由

### 按 Inode ID 查找分区

```go
func (mw *MetaWrapper) getPartitionByInode(ino uint64) *MetaPartition {
    // 在 ranges B-Tree 中查找
    // 找到 Start <= ino 的最大分区
    // 验证 ino <= End
    // 返回目标 MetaPartition
}
```

### 路由示例

```
Inode ID = 50000

MetaPartition 视图:
  MP-1: [0, 30000)
  MP-2: [30000, 60000)    ← 匹配！
  MP-3: [60000, ∞)

路由结果: 请求发往 MP-2
```

## 高层 API

定义在 `sdk/meta/api.go`，提供 POSIX 风格的接口：

### 创建文件

```go
func (mw *MetaWrapper) Create_ll(parentID uint64, name string, mode uint32,
    uid, gid uint32, target []byte, fullPath string, ignoreExist bool) (*proto.InodeInfo, error) {

    // 1. getPartitionByInode(parentID) → 获取父目录所在 MP
    // 2. getRWPartitions() → 选择一个可写 MP 创建新 inode
    // 3. icreate(mp, mode, uid, gid, target) → 发送 OpMetaCreateInode
    // 4. dcreate(parentMP, parentID, name, inode, mode) → 发送 OpMetaCreateDentry
    // 5. 返回新 inode 信息
}
```

### 查找文件

```go
func (mw *MetaWrapper) Lookup_ll(parentID uint64, name string) (uint64, uint32, error) {
    // 1. getPartitionByInode(parentID)
    // 2. lookup(parentMP, parentID, name) → 发送 OpMetaLookup
    // 3. 返回 (inode, mode)
}
```

### 读取目录

```go
func (mw *MetaWrapper) ReadDir_ll(parentID uint64) ([]proto.Dentry, error) {
    // 循环调用 ReadDirLimit_ll 直到读完
    // 每次: getPartitionByInode(parentID) → readDirLimit → OpMetaReadDirLimit
}
```

### 其他 API

| 方法 | 说明 |
|------|------|
| `InodeGet_ll(ino)` | 获取 Inode 信息 |
| `Delete_ll(parentID, name, isDir, fullPath)` | 删除文件/目录 |
| `Rename_ll(srcParent, srcName, dstParent, dstName, ...)` | 重命名 |
| `AppendExtentKey(inode, ek, ...)` | 追加 Extent Key |
| `GetExtents(inode)` | 获取 Extent Key 列表 |
| `Truncate(inode, size)` | 截断文件 |
| `SetAttr(inode, ...)` | 设置属性 |
| `SetXAttr/GetXAttr(inode, ...)` | 扩展属性 |

## 视图刷新

MetaWrapper 后台定期刷新 MP 视图：

```go
func (mw *MetaWrapper) refresh() {
    // 定期执行:
    // 1. updateMetaPartitions() → 从 Master 获取最新 MP 列表
    // 2. updateVolStatInfo() → 更新卷统计信息
    // 3. 更新 ranges B-Tree 和 rwPartitions 列表
}
```
