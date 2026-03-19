# DataNode Extent 存储引擎

## 概述

ExtentStore 是 DataNode 的核心存储引擎，负责管理 Extent（数据块）文件的创建、读写、删除和修复。

## ExtentStore 结构体

```go
type ExtentStore struct {
    dataPath         string                  // 数据目录
    baseExtentID     uint64                  // 下一个可用的 Extent ID
    extentInfoMap    map[uint64]*ExtentInfo  // Extent 元信息
    cache            *ExtentCache            // Extent 文件句柄缓存
    tinyExtentDeleteFp  *os.File            // Tiny Extent 删除记录文件
    normalExtentDeleteFp *os.File           // Normal Extent 删除记录文件
    verifyExtentFp   *os.File               // Block CRC 校验文件
    availableTinyExtentC chan uint64         // 可用的 Tiny Extent ID channel
    brokenTinyExtentC    chan uint64         // 损坏的 Tiny Extent ID channel
    ApplyId          uint64                  // 当前 Raft Apply ID
}
```

## Extent 文件结构

### Extent 结构体

```go
type Extent struct {
    file            *os.File    // 数据文件句柄
    readFile        *os.File    // O_DIRECT 读文件句柄
    filePath        string
    extentID        uint64
    dataSize        int64       // 数据大小
    snapshotDataOff uint64      // 快照数据偏移
    header          []byte      // Block CRC Header
    dirty           atomicutil.Bool
}
```

### Block CRC Header

每个 Normal Extent 有一个 Header 区域（通常在文件开头），存储各数据块的 CRC 校验码。写入数据时同步更新对应块的 CRC。

## Extent 类型

### Tiny Extent (ID 1-64)

用于存储小文件（< 128KB）：

```
Tiny Extent 文件结构:
┌────────────────────────────────────────────┐
│ 小文件数据-1 (追加写入)                       │
├────────────────────────────────────────────┤
│ 小文件数据-2 (追加写入)                       │
├────────────────────────────────────────────┤
│ 小文件数据-3 (追加写入)                       │
├────────────────────────────────────────────┤
│ [已删除区域 - punch hole]                    │
├────────────────────────────────────────────┤
│ 小文件数据-5 (追加写入)                       │
└────────────────────────────────────────────┘
```

特点：
- 数据追加写入，多个小文件共享同一个 Extent 文件
- 删除时使用 `fallocate` punch hole，不实际释放文件空间
- 系统启动时预创建 64 个 Tiny Extent
- 通过 `availableTinyExtentC` channel 分配可用的 Tiny Extent

### Normal Extent (ID >= 1024)

用于存储大文件：

```
Normal Extent 文件结构:
┌─────────────────────────┐
│ Header (Block CRC 校验)   │
├─────────────────────────┤
│ 数据块-0                  │
├─────────────────────────┤
│ 数据块-1                  │
├─────────────────────────┤
│ ...                      │
├─────────────────────────┤
│ 数据块-N                  │
└─────────────────────────┘
```

特点：
- 每个 Extent 对应一个独立文件
- 删除时直接删除文件
- 通过 `baseExtentID` 递增分配新 ID

## 核心操作

### 创建 Extent

```go
func (s *ExtentStore) Create(extentID uint64) error {
    // 1. 初始化 Extent 文件 (O_EXCL 确保唯一)
    // 2. 添加到 extentInfoMap
    // 3. 加入 cache
    // 4. 如果是 Tiny Extent，放入 availableTinyExtentC
}
```

### 写入数据

```go
func (s *ExtentStore) Write(param WriteParam) error {
    // 1. 检查 Extent 是否被 GC 锁定
    // 2. 从 cache 获取 Extent
    // 3. 调用 extent.Write(param):
    //    - 追加写: 写入数据到文件末尾
    //    - 随机写: 写入到指定偏移
    //    - 更新 CRC Header
    //    - 如果需要同步: fsync
}
```

### 读取数据

```go
func (s *ExtentStore) Read(extentID, offset, size, ...) ([]byte, error) {
    // 1. 从 cache 获取 Extent
    // 2. 调用 extent.Read():
    //    - Normal: pread 对齐读取
    //    - Tiny: ReadTiny 读取指定偏移
    // 3. 返回数据
}
```

### 标记删除

```go
func (s *ExtentStore) MarkDelete(extentID uint64, offset, size int64) error {
    if isTinyExtent(extentID) {
        // Tiny: fallocate punch hole（不删除文件）
        // 记录到 tinyExtentDeleteFp
    } else {
        // Normal: 删除文件
        // 记录到 normalExtentDeleteFp
        // 从 extentInfoMap 移除
    }
}
```

## 快照

```go
func (s *ExtentStore) SnapShot() []*proto.File {
    // 遍历 extentInfoMap
    // 返回所有 Extent 的 ID、Size、Modified、CRC 信息
    // 用于副本间对比和修复
}
```

## Extent Cache

```go
type ExtentCache struct {
    cache map[uint64]*Extent  // extentID -> Extent
}
```

- 缓存打开的 Extent 文件句柄，避免重复 open
- 定期驱逐长时间未使用的 Extent
- LRU 策略

## 后台任务

| 任务 | 说明 |
|------|------|
| Auto CRC | 定期计算和验证 Block CRC |
| Clean Delete Cache | 清理删除缓存 |
| Extent Eviction | 驱逐不活跃的 Extent 缓存 |
