# FUSE 挂载流程

## 关键操作映射

| FUSE 操作 | CubeFS 实现 | 说明 |
|-----------|------------|------|
| `Lookup` | `Dir.Lookup()` | dcache → mw.Lookup_ll → InodeGet |
| `Create` | `Dir.Create()` | mw.Create_ll → ec.OpenStream |
| `Mkdir` | `Dir.Mkdir()` | mw.Create_ll (目录模式) |
| `Remove` | `Dir.Remove()` | mw.Delete_ll → orphan 管理 |
| `Rename` | `Dir.Rename()` | mw.Rename_ll → 更新缓存 |
| `ReadDir` | `Dir.ReadDir()` | mw.ReadDirLimit_ll → 批量 InodeGet |
| `Open` | `File.Open()` | ec.OpenStream → 刷新 Extent 缓存 |
| `Read` | `File.Read()` | ec.Read → ExtentReader |
| `Write` | `File.Write()` | ec.Write → Streamer |
| `Flush` | `File.Flush()` | ec.Flush |
| `Fsync` | `File.Fsync()` | ec.Flush |
| `Release` | `File.Release()` | ec.CloseStream → 驱逐缓存 |
| `Getattr` | `File/Dir.Attr()` | InodeCache → mw.InodeGet_ll |
| `Setattr` | `File.Setattr()` | ec.Truncate / mw.SetAttr |
| `Forget` | `File/Dir.Forget()` | 清理缓存 |

## 文件创建流程详解

```
应用程序: open("/mnt/vol/dir1/newfile.txt", O_CREAT)
    ▼
FUSE 内核: Lookup "dir1" → Create "newfile.txt"
    ▼
Dir.Create(name="newfile.txt", mode, flags):
    ├── mw.Create_ll(parentIno, "newfile.txt", mode, uid, gid):
    │   ├── getPartitionByInode(parentIno) → 父目录 MP
    │   ├── getRWPartitions() → 选择 MP 创建 inode
    │   ├── icreate(mp, mode, uid, gid) → OpMetaCreateInode
    │   └── dcreate(parentMP, parentIno, name, ino) → OpMetaCreateDentry
    ├── ic.Put(inodeInfo) → 缓存新 inode
    ├── NewFile(super, info) → 创建 FUSE File 节点
    ├── ec.OpenStream(ino) → 创建 Streamer
    └── nodeCache[ino] = child
```

## 文件读取流程详解

```
应用程序: read(fd, buf, 4096)
    ▼
FUSE 内核: Read request
    ▼
File.Read(req, resp):
    ├── ec.Read(ino, data, offset, size):
    │   ├── GetStreamer(ino)
    │   ├── 如果有脏数据 → flush
    │   ├── streamer.read(data, offset, size):
    │   │   ├── PrepareReadRequests(offset, size)
    │   │   │   → 根据 Extent Key 分割读请求
    │   │   └── 对每个请求:
    │   │       ├── 尝试预读缓存 (AheadRead)
    │   │       ├── 尝试本地缓存 (BCache)
    │   │       ├── 尝试远程缓存 (FlashNode)
    │   │       └── 从 DataNode 读取
    │   │           ├── GetDataPartition(partitionId)
    │   │           ├── SortHostsByPingElapsed()
    │   │           └── 发送 OpStreamRead 到最近节点
    │   └── 返回读取的数据
    └── 填充 resp.Data
```

## 文件写入流程详解

```
应用程序: write(fd, data, 1MB)
    ▼
FUSE 内核: Write request
    ▼
File.Write(req, resp):
    ├── ec.Write(ino, offset, data):
    │   ├── GetStreamer(ino)
    │   ├── IssueWriteRequest(data, offset, size)
    │   ├── Streamer.write():
    │   │   ├── PrepareWriteRequests(offset, size)
    │   │   └── 对每个请求:
    │   │       ├── doWriteAppend():
    │   │       │   ├── 获取/创建 ExtentHandler
    │   │       │   ├── handler.write(data) → 缓冲
    │   │       │   └── 加入 dirtylist
    │   │       └── doOverwrite():
    │   │           ├── flush() 当前脏数据
    │   │           └── 发送 OpRandomWrite
    │   └── 返回写入字节数
    └── 填充 resp.Size

// 后续 flush（定期 or 显式）:
dirtylist 中的 handler → flush:
    ├── 发送数据到 DataNode (OpWrite)
    └── mw.AppendExtentKey() → 更新 MetaNode
```
