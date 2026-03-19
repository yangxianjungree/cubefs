# MetaNode Raft FSM

## 概述

每个 MetaPartition 运行一个独立的 Raft Group（Multi-Raft 模式）。所有写操作必须通过 Raft 提交，确保副本间数据一致性。FSM（有限状态机）负责将 Raft 日志应用到内存 B-Tree。

## Apply 主流程

```go
func (mp *metaPartition) Apply(command []byte, index uint64) (interface{}, error) {
    // 1. 反序列化 MetaItem
    msg := &MetaItem{}
    msg.Unmarshal(command)

    // 2. 加锁（保证非幂等操作的顺序性）
    mp.nonIdempotent.Lock()
    defer mp.nonIdempotent.Unlock()

    // 3. 根据 Op 分发到对应处理函数
    switch msg.Op {
    case opFSMCreateInode:
        return mp.fsmCreateInode(msg.V)
    case opFSMCreateDentry:
        return mp.fsmCreateDentry(msg.V)
    // ... 50+ 种操作
    }

    // 4. 更新 appliedID
    mp.uploadApplyID(index)
}
```

## FSM 操作分类

### Inode 操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMCreateInode` | `fsmCreateInode` | 创建 inode，插入 inodeTree，更新 Cursor |
| `opFSMUnlinkInode` | `fsmUnlinkInode` | 减少 NLink，NLink=0 时加入 freeList |
| `opFSMUnlinkInodeOnce` | `fsmUnlinkInode` | 幂等 Unlink（带 UniqID） |
| `opFSMCreateLinkInode` | `fsmCreateLinkInode` | 硬链接，增加 NLink |
| `opFSMEvictInode` | `fsmEvictInode` | 驱逐 inode（释放资源） |
| `opFSMSetAttr` | `fsmSetAttr` | 设置 inode 属性（mode/uid/gid/时间） |
| `opFSMExtentTruncate` | `fsmExtentsTruncate` | 截断文件，删除多余 Extent Key |
| `opFSMInternalDeleteInode` | `internalDelete` | 内部删除（GC 使用） |

### Dentry 操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMCreateDentry` | `fsmCreateDentry` | 创建目录项 |
| `opFSMDeleteDentry` | `fsmDeleteDentry` | 删除目录项 |
| `opFSMUpdateDentry` | `fsmUpdateDentry` | 更新目录项（rename） |
| `opFSMDeleteDentryBatch` | `fsmBatchDeleteDentry` | 批量删除目录项 |

### Extent 操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMExtentsAdd` | `fsmAppendExtents` | 追加 Extent Key 到 inode |
| `opFSMExtentsAddWithCheck` | `fsmAppendExtentsWithCheck` | 带冲突检测的追加 |
| `opFSMObjExtentsAdd` | `fsmAppendObjExtents` | 追加 BlobStore Extent Key |
| `opFSMSentToChan` | `fsmSendToChan` | 将待删除 Extent Key 发送到删除 channel |

### 事务操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMTxInit` | `fsmTxInit` | 初始化事务 |
| `opFSMTxCreateInode` | `fsmTxCreateInode` | 事务内创建 inode |
| `opFSMTxCreateDentry` | `fsmTxCreateDentry` | 事务内创建 dentry |
| `opFSMTxCommit` | `fsmTxCommit` | 提交事务 |
| `opFSMTxRollback` | `fsmTxRollback` | 回滚事务 |
| `opFSMTxCommitRM` | `fsmTxCommitRM` | RM 提交确认 |
| `opFSMTxRollbackRM` | `fsmTxRollbackRM` | RM 回滚确认 |

### XAttr 操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMSetXAttr` | `fsmSetXAttr` | 设置扩展属性 |
| `opFSMRemoveXAttr` | `fsmRemoveXAttr` | 删除扩展属性 |
| `opFSMUpdateXAttr` | `fsmUpdateXAttr` | 更新扩展属性 |

### Multipart 操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMCreateMultipart` | `fsmCreateMultipart` | 创建 Multipart 上传 |
| `opFSMRemoveMultipart` | `fsmRemoveMultipart` | 删除 Multipart |
| `opFSMAppendMultipart` | `fsmAppendMultipart` | 追加 Part |

### 管理操作

| Op | 函数 | 说明 |
|----|------|------|
| `opFSMUpdatePartition` | `fsmUpdatePartition` | 更新分区范围（分裂时） |
| `opFSMStoreTick` | — | 触发快照持久化 |
| `opFSMSyncCursor` | — | 同步 Cursor |
| `opFSMVersionOp` | `fsmVersionOp` | 版本列表操作 |

## 操作提交流程

以 CreateInode 为例：

```
Client 发送 OpMetaCreateInode Packet
    ▼
MetaNode 接收 Packet
    ▼
HandleMetadataOperation → opCreateInode()
    ▼
构建 CreateInodeReq，验证参数
    ▼
mp.submit(opFSMCreateInode, reqData)
    ▼
序列化为 MetaItem{Op, V}
    ▼
raftPartition.Submit(data)
    ▼
Raft 复制到多数节点
    ▼
Apply(command, index)
    ▼
fsmCreateInode(data):
    · 反序列化 Inode
    · 更新 Cursor = max(Cursor, inode.Inode)
    · inodeTree.ReplaceOrInsert(inode, false)
    · 如果已存在，返回 OpExistErr
    ▼
返回结果给客户端
```

## Snapshot 和 Restore

### Snapshot（生成快照）

```go
func (mp *metaPartition) Snapshot() (raft.Snapshot, error) {
    // 返回 MetaItemIterator
    // 遍历 applyID + inodeTree + dentryTree + extendTree + multipartTree + tx + uniq + multiVersion
}
```

### ApplySnapshot（应用快照）

```go
func (mp *metaPartition) ApplySnapshot(peers []raft.Peer, iter raft.SnapIterator) error {
    // 1. 清空当前所有 B-Tree
    // 2. 逐条从 iterator 读取 MetaItem
    // 3. 根据 Op 类型插入到对应 B-Tree
    // 4. 恢复 applyID, cursor, txID 等
}
```

### HandleLeaderChange

```go
func (mp *metaPartition) HandleLeaderChange(leader uint64) {
    if isLeader {
        // 启动定期 store tick（触发快照）
        // 初始化 root inode（如果是新分区）
    } else {
        // 停止 store tick
    }
}
```
