# 端到端写入流程

## 完整写入链路

以 FUSE Client 写入文件 `/mnt/vol/dir1/file.txt` 为例：

```
┌──────────────────────────────────────────────────────────────────┐
│ 应用程序: fd = open("/mnt/vol/dir1/file.txt", O_CREAT|O_WRONLY) │
│          write(fd, data, 1MB)                                    │
│          close(fd)                                               │
└──────────────┬───────────────────────────────────────────────────┘
               │ FUSE 系统调用
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ 步骤 1: open/create                                              │
│                                                                  │
│ FUSE → Dir.Create("file.txt"):                                   │
│   · MetaWrapper.Create_ll(parentIno=200, name="file.txt"):       │
│     - getPartitionByInode(200) → MP-1 (Leader: MetaNode-A)      │
│     - getRWPartitions() → 选择 MP-2 分配 inode                    │
│     - icreate(MP-2, mode, uid, gid):                             │
│       · TCP Packet(OpMetaCreateInode) → MetaNode-B               │
│       · MetaNode-B Raft Submit → 多数确认                         │
│       · FSM Apply: inodeTree.Insert(Inode{ID=12345})             │
│       · 返回 Inode{ID=12345}                                     │
│     - dcreate(MP-1, parentIno=200, name="file.txt", ino=12345): │
│       · TCP Packet(OpMetaCreateDentry) → MetaNode-A              │
│       · Raft Submit → FSM Apply: dentryTree.Insert               │
│   · InodeCache.Put(12345, info)                                  │
│   · ExtentClient.OpenStream(12345) → 创建 Streamer               │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ 步骤 2: write(data, 1MB)                                         │
│                                                                  │
│ FUSE → File.Write():                                             │
│   · ExtentClient.Write(inode=12345, offset=0, data=1MB):         │
│     - Streamer.IssueWriteRequest → request channel               │
│     - Streamer.write():                                          │
│       · PrepareWriteRequests(0, 1MB)                             │
│       · doWriteAppend():                                         │
│         - NewExtentHandler(streamer, dp) → 分配 DataPartition     │
│         - DataPartition 选择:                                     │
│           · Wrapper.GetDataPartitionForWrite(exclude)             │
│           · dpSelector 选择可写 DP (Straw 加权)                    │
│           · 选中 DP-100 (Leader: DataNode-X)                      │
│         - handler.write(data):                                   │
│           · 缓冲到 handler.packet                                 │
│           · 加入 dirtylist                                        │
└──────────────┬───────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ 步骤 3: flush (close 或定期触发)                                   │
│                                                                  │
│ handler.flush():                                                 │
│   · 构建 Packet(OpWrite):                                        │
│     - PartitionID = 100                                          │
│     - ExtentID = NextExtentID()                                  │
│     - Data = 1MB                                                 │
│   · TCP 发送到 DataNode-X (DP-100 Leader):                       │
│     ▼                                                            │
│   DataNode-X (ReplProtocol):                                     │
│     - prepareFunc: 校验包                                         │
│     - 转发到 Follower DataNode-Y, DataNode-Z                     │
│     - operatorFunc: ExtentStore.Write(data)                      │
│       · 创建 Extent 文件                                          │
│       · 写入数据到磁盘                                             │
│       · 更新 Block CRC                                            │
│     - Raft Submit → 多数确认                                      │
│     - 等待 Follower 响应                                          │
│     - postFunc: 返回成功                                          │
│                                                                  │
│ appendExtentKey:                                                 │
│   · MetaWrapper.AppendExtentKey(12345, ExtentKey):               │
│     - getPartitionByInode(12345) → MP-2                          │
│     - TCP Packet(OpMetaExtentsAdd) → MetaNode-B                  │
│     - Raft Submit → FSM Apply:                                   │
│       · inode.AppendExtents([ExtentKey{                          │
│           FileOffset: 0,                                         │
│           PartitionId: 100,                                      │
│           ExtentId: 1024,                                        │
│           Size: 1MB                                              │
│         }])                                                      │
└──────────────────────────────────────────────────────────────────┘
```

## 关键性能因素

| 步骤 | 延迟来源 | 优化 |
|------|---------|------|
| 元数据创建 | Raft 多数确认 | 批量操作、连接池 |
| 数据写入 | 链式复制 + fsync | 异步刷盘、写缓冲 |
| Extent Key 更新 | Raft 多数确认 | 批量追加 |

## 顺序写 vs 随机写

| 类型 | 触发条件 | 协议 | 性能 |
|------|---------|------|------|
| 顺序写 | 追加写入（新数据） | Primary-Backup 链式复制 | 高吞吐 |
| 随机写 | 覆盖已有数据 | Raft Multi-Raft | 强一致但较慢 |
