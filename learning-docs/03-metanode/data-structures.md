# MetaNode 数据结构

## Inode 结构体

```go
type Inode struct {
    sync.RWMutex
    Inode       uint64    // Inode ID（全局唯一）
    Type        uint32    // 文件类型（普通文件/目录/符号链接等）
    Uid         uint32    // 所有者用户 ID
    Gid         uint32    // 所有者组 ID
    Size        uint64    // 文件大小
    Generation  uint64    // 版本号（每次 Extent 变更递增）
    CreateTime  int64     // 创建时间
    AccessTime  int64     // 最后访问时间
    ModifyTime  int64     // 最后修改时间
    NLink       uint32    // 硬链接计数（目录初始为2，文件为1）
    Flag        int32     // 标志位
    Reserved    uint64    // 保留字段
    StorageClass uint32   // 存储类别（SSD/HDD/BlobStore）
    LinkTarget  []byte    // 符号链接目标路径

    // 数据布局
    HybridCloudExtents          *SortedHybridCloudExtents          // 副本存储 Extent Key 列表
    HybridCloudExtentsMigration *SortedHybridCloudExtentsMigration // 迁移中的 Extent Key

    // 多版本快照
    multiSnap   *InodeMultiSnap
}
```

### Inode 排序

Inode 在 B-Tree 中按 `Inode ID` 排序（`Less()` 方法比较 Inode 字段）。

### Extent Key

每个文件的数据由一组 Extent Key 描述：

```go
// proto/obj_extent_key.go
type ExtentKey struct {
    FileOffset   uint64  // 文件内偏移
    PartitionId  uint64  // DataPartition ID
    ExtentId     uint64  // Extent ID
    ExtentOffset uint64  // Extent 内偏移
    Size         uint32  // 数据大小
    CRC          uint32  // CRC 校验
}
```

Extent Key 由 `SortedExtents` 按 `FileOffset` 排序管理，实现文件数据到 DataPartition 的映射。

## Dentry 结构体

```go
type Dentry struct {
    ParentId uint64    // 父目录的 Inode ID
    Name     string    // 文件/子目录名称
    Inode    uint64    // 指向的 Inode ID
    Type     uint32    // 文件类型

    // 多版本快照
    multiSnap *DentryMultiSnap
}
```

### Dentry 排序

Dentry 在 B-Tree 中按 `(ParentId, Name)` 排序。`Less()` 方法先比较 ParentId，相同时按 Name 字典序比较。

### 目录结构示意

```
/ (Inode=1)
├── file1.txt (Dentry: ParentId=1, Name="file1.txt", Inode=100)
├── dir1/     (Dentry: ParentId=1, Name="dir1", Inode=200)
│   └── file2.txt (Dentry: ParentId=200, Name="file2.txt", Inode=101)
└── dir2/     (Dentry: ParentId=1, Name="dir2", Inode=201)
```

## B-Tree 实现

```go
type BTree struct {
    sync.RWMutex
    tree *btree.BTree  // google/btree, degree=32
}
```

### 主要操作

| 方法 | 说明 |
|------|------|
| `ReplaceOrInsert(key, replace)` | 插入或替换（replace=true 时覆盖） |
| `Delete(key)` | 删除节点 |
| `Get(key)` | 查找（RLock） |
| `CopyGet(key)` | 查找并返回深拷贝（Lock） |
| `Has(key)` | 是否存在 |
| `Ascend(fn)` | 正序遍历 |
| `AscendRange(ge, lt, fn)` | 范围遍历 |
| `GetTree()` | 克隆整棵树（用于快照） |
| `Len()` | 节点数 |
| `MaxItem()` | 最大节点 |

### 四棵 B-Tree

每个 MetaPartition 维护四棵 B-Tree：

| B-Tree | Key | Value | 用途 |
|--------|-----|-------|------|
| `inodeTree` | Inode ID | Inode 对象 | 所有 inode 信息 |
| `dentryTree` | (ParentId, Name) | Dentry 对象 | 目录树结构 |
| `extendTree` | Inode ID | XAttr Map | 扩展属性 |
| `multipartTree` | MultipartKey | Multipart | S3 Multipart 上传 |

## SortedExtents（Extent Key 管理）

### 副本存储 Extent

```go
type SortedExtents struct {
    sync.RWMutex
    eks []proto.ExtentKey  // 按 FileOffset 排序
}
```

关键方法：

| 方法 | 说明 |
|------|------|
| `Append(ek)` | 追加 Extent Key，处理重叠 |
| `AppendWithCheck(inodeID, ek, ...)` | 带冲突检测的追加（幂等） |
| `Truncate(offset, ...)` | 截断，返回被删除的 Extent Key |
| `Range(f)` | 遍历所有 Extent Key |
| `Size()` | 总数据大小 |
| `Clone()` | 深拷贝 |

### 对象存储 Extent

```go
type SortedObjExtents struct {
    sync.RWMutex
    eks []proto.ObjExtentKey  // BlobStore 对象 Extent
}
```

### 混合云 Extent

`SortedHybridCloudExtents` 包装了 `SortedExtents` 和 `SortedObjExtents`，根据 `StorageClass` 决定使用哪种：

- `StorageClass_Replica_*` → 使用 SortedExtents
- `StorageClass_BlobStore` → 使用 SortedObjExtents

## 多版本快照支持

### InodeMultiSnap

```go
type InodeMultiSnap struct {
    verSeq        uint64        // 版本序列号
    multiVersions InodeBatch    // 旧版本 inode 列表
    ekRefMap      *sync.Map     // Extent 分裂引用计数
}
```

### DentryMultiSnap

```go
type DentryMultiSnap struct {
    VerSeq     uint64
    dentryList DentryBatch  // 旧版本 dentry 列表
}
```

版本机制允许创建文件系统快照，每个快照对应一个 `verSeq`。读取特定版本时，从当前版本回溯到目标版本。

## 数据持久化

MetaPartition 的数据通过快照机制持久化到磁盘：

```
partition_<id>/
├── inode              # Inode 快照文件
├── dentry             # Dentry 快照文件
├── extend             # XAttr 快照文件
├── multipart          # Multipart 快照文件
├── applyid            # 最近应用的 Raft index
├── txinfo             # 事务信息
├── txrbdentry         # 事务回滚 Dentry
├── txrbinode          # 事务回滚 Inode
├── uniqchecker        # 幂等检查器
└── multiVersionList   # 版本列表
```

快照通过 `opFSMStoreTick` Raft 操作定期触发：
1. 克隆所有 B-Tree
2. 序列化到临时目录
3. 原子重命名替换旧快照
