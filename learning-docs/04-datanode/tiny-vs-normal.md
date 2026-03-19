# DataNode 大小文件策略

## 两种 Extent 类型

CubeFS 针对大文件和小文件采用不同的存储策略：

| 特性 | Tiny Extent | Normal Extent |
|------|-------------|---------------|
| **ID 范围** | 1 ~ 64 | >= 1024 |
| **数量** | 每个 DP 固定 64 个 | 动态增长 |
| **文件映射** | 多个小文件共享一个 Extent | 一个大文件对应一个或多个 Extent |
| **写入方式** | 追加写入 | 追加或随机写 |
| **删除方式** | punch hole (fallocate) | 删除文件 |
| **适用场景** | 小文件 (< 128KB) | 大文件 |

## Tiny Extent 详解

### 设计动机

分布式存储系统面对大量小文件（如日志、配置文件、图片缩略图）时，如果每个小文件占一个独立文件，会造成：
- 大量文件系统 inode 消耗
- 磁盘空间浪费（文件系统块对齐）
- 随机 I/O 增多

Tiny Extent 通过将多个小文件数据"打包"到同一个 Extent 文件中解决这个问题。

### 存储结构

```
DataPartition 目录/
├── 1        ← Tiny Extent (ID=1)
├── 2        ← Tiny Extent (ID=2)
├── ...
├── 64       ← Tiny Extent (ID=64)
├── 1024     ← Normal Extent
├── 1025     ← Normal Extent
└── ...
```

### 写入流程

```
写入小文件数据 (size < 128KB)
    ▼
从 availableTinyExtentC 获取一个 Tiny Extent ID
    ▼
追加写入到该 Tiny Extent 文件末尾
    ▼
记录 ExtentKey:
    · ExtentId = tinyExtentId (1-64)
    · ExtentOffset = 追加写入时的偏移
    · Size = 数据大小
    ▼
ExtentKey 记录在 MetaNode 的 Inode 中
```

### 删除流程

```
删除小文件
    ▼
MetaNode 返回该文件的 ExtentKey
    ▼
DataNode 不删除整个 Tiny Extent 文件
    ▼
使用 fallocate(FALLOC_FL_PUNCH_HOLE) "打洞"
    · 将对应的 [offset, offset+size] 区域标记为空洞
    · 文件系统释放这些块的磁盘空间
    · 文件逻辑大小不变
    ▼
记录到 tinyExtentDeleteFp 文件
    · 用于副本间同步删除操作
```

### Tiny Extent 管理

```go
// 可用 Tiny Extent channel
availableTinyExtentC chan uint64  // 还有空间的 Tiny Extent ID

// 损坏 Tiny Extent channel
brokenTinyExtentC chan uint64    // 损坏的 Tiny Extent ID
```

修复时通过 `sendAllTinyExtentsToC()` 重新分类。

## Normal Extent 详解

### 存储结构

每个 Normal Extent 是一个独立文件：

```
Normal Extent 文件:
┌──────────────────────────────┐
│ Header (Block CRC 数组)       │  每个 Block (通常 128KB) 一个 CRC
├──────────────────────────────┤
│ Block 0 (128KB)              │
├──────────────────────────────┤
│ Block 1 (128KB)              │
├──────────────────────────────┤
│ ...                          │
├──────────────────────────────┤
│ Block N                      │
└──────────────────────────────┘
```

### 写入流程

```
写入大文件数据
    ▼
ExtentStore.NextExtentID() → 分配新 Extent ID
    ▼
ExtentStore.Create(extentID) → 创建新文件
    ▼
Extent.Write(data, offset) → 写入数据
    · 更新对应 Block 的 CRC
    · 如果 IsSync=true，调用 fsync
    ▼
记录 ExtentKey:
    · ExtentId = newExtentId (>= 1024)
    · ExtentOffset = 0 (从头开始)
    · Size = 数据大小
```

### 删除流程

```
删除 Normal Extent
    ▼
os.Remove(extentFilePath)  // 直接删除文件
    ▼
记录到 normalExtentDeleteFp  // 持久化删除记录
    ▼
从 extentInfoMap 移除
```

## 如何判断使用哪种 Extent

判断逻辑在 SDK/Client 端：

1. 客户端写入数据时，根据写入大小选择 Extent 类型
2. 小于阈值（128KB）的数据使用 Tiny Extent
3. 大于阈值的数据使用 Normal Extent
4. ExtentKey 中的 `ExtentType` 字段标识类型

## 数据修复的差异

| 步骤 | Tiny Extent | Normal Extent |
|------|-------------|---------------|
| 比较 | 按偏移和大小比较 | 按文件大小比较 |
| 创建缺失 | 创建空 Tiny Extent | 创建空文件 |
| 数据同步 | TinyExtentRecover (写入+punch hole) | 从源读取差异部分追加 |
| 删除同步 | 同步 TinyDeleteRecord | N/A（直接删除文件） |

## 性能考量

### Tiny Extent 优势
- 减少小文件的元数据开销
- 顺序追加写，磁盘友好
- punch hole 释放空间，无需 compaction

### Tiny Extent 劣势
- 随着删除增多，文件变得稀疏（碎片化）
- 读取需要定位到精确偏移
- 修复逻辑更复杂（需要同步删除记录）

### Normal Extent 优势
- 读写简单直接
- 删除彻底（文件级删除）
- 适合大块顺序 I/O

### Normal Extent 劣势
- 每个 Extent 一个文件，大量 Extent 时 inode 消耗大
- 小文件使用会浪费空间
