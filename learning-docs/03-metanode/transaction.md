# MetaNode 分布式事务

## 概述

CubeFS 的分布式事务用于保证跨 MetaPartition 操作（如 rename）的原子性。例如，当文件从一个目录 rename 到另一个目录时，可能涉及两个不同的 MetaPartition，需要事务保证要么全部成功，要么全部回滚。

## 事务架构

采用两阶段提交 (2PC) 模型：

```
Client (Coordinator)
    │
    ├── MetaPartition-A (RM: Resource Manager)
    │   └── TxRollbackInode / TxRollbackDentry
    │
    └── MetaPartition-B (RM: Resource Manager)
        └── TxRollbackInode / TxRollbackDentry
```

## 核心结构

### TransactionManager (TM)

```go
type TransactionManager struct {
    txIdAlloc   *TxIDAllocator  // 事务 ID 分配器
    txTree      *BTree          // 活跃事务 B-Tree
    txProcessor *TransactionProcessor
    blacklist   *util.Set       // 黑名单
    opLimiter   *rate.Limiter   // 操作限流
}
```

### TransactionResource (RM)

```go
type TransactionResource struct {
    txRbInodeTree  *BTree  // Inode 回滚记录
    txRbDentryTree *BTree  // Dentry 回滚记录
    txProcessor    *TransactionProcessor
}
```

### 回滚记录

```go
type TxRollbackInode struct {
    inode      *Inode
    txInodeInfo *proto.TxInodeInfo
    rbType     uint32  // TxNoOp / TxAdd / TxDelete / TxUpdate
    quotaIds   []uint32
}

type TxRollbackDentry struct {
    dentry      *Dentry
    txDentryInfo *proto.TxDentryInfo
    rbType      uint32
}
```

### 回滚类型

| 类型 | 含义 | 回滚动作 |
|------|------|---------|
| `TxNoOp` | 无操作 | 不做任何事 |
| `TxAdd` | 事务中新增了资源 | 回滚时恢复原始值 |
| `TxDelete` | 事务中删除了资源 | 回滚时重新删除 |
| `TxUpdate` | 事务中更新了资源 | 回滚时恢复旧值 |

## 事务流程

### 1. 创建事务

```
Client → opMetaTxCreate → MP Leader
    ▼
registerTransaction(txInfo):
    · 分配 TxID
    · 设置超时时间
    · 插入 txTree
    · Raft Submit (opFSMTxInit)
```

### 2. 执行操作（Prepare 阶段）

```
Client → opMetaTxCreateInode → MP-A
    ▼
fsmTxCreateInode:
    · 创建 inode
    · 记录 TxRollbackInode{rbType: TxAdd}
    · 标记 inode 被事务占用

Client → opMetaTxCreateDentry → MP-B
    ▼
fsmTxCreateDentry:
    · 创建 dentry
    · 记录 TxRollbackDentry{rbType: TxAdd}
```

### 3. 提交事务

```
Client → opTxCommit → TM所在的MP
    ▼
commitTx(txId):
    · 设置 TxState = TxStateCommit
    · 向每个 RM 发送 opTxCommitRM
    ▼
各 RM 收到 commitRM:
    · 清除回滚记录 (TxRollbackInode/Dentry)
    · 解除 inode/dentry 的事务锁
    ▼
TM 收到全部 RM 确认:
    · 从 txTree 删除事务
    · Raft Submit (opFSMTxCommit)
```

### 4. 回滚事务

```
超时或显式回滚 → opTxRollback → TM
    ▼
rollbackTx(txId):
    · 设置 TxState = TxStateRollback
    · 向每个 RM 发送 opTxRollbackRM
    ▼
各 RM 收到 rollbackRM:
    · 根据回滚记录恢复:
      - TxAdd: 恢复原始 inode/dentry
      - TxDelete: 删除新增的 inode/dentry
      - TxUpdate: 恢复旧值
    · 清除回滚记录
    ▼
TM 收到全部 RM 确认:
    · 从 txTree 删除事务
```

## 超时处理

`processExpiredTransactions()` 定期扫描 txTree，处理过期事务：

- 超过超时时间未提交的事务 → 自动回滚
- 已提交但未收到所有 RM 确认的事务 → 重试提交
- 已回滚但未收到所有 RM 确认的事务 → 重试回滚

## 典型使用场景

### Rename 操作

当 rename 涉及两个不同的 MetaPartition 时：

```
rename(src: MP-A:/dir1/file1, dst: MP-B:/dir2/file1)
    ▼
1. TxCreate → MP-A (创建事务)
2. TxDeleteDentry(dir1/file1) → MP-A (删除源 dentry)
3. TxCreateDentry(dir2/file1) → MP-B (创建目标 dentry)
4. TxCommit → MP-A (提交事务)
```

如果步骤 3 失败，则步骤 2 的删除会被回滚（恢复 dir1/file1）。
