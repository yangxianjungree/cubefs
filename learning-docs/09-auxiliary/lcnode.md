# LcNode 生命周期管理

## 概述

LcNode (`lcnode/`) 执行 S3 生命周期规则，支持对象过期删除和冷热数据迁移。

## 架构

```
Master
    │ 下发扫描任务 (OpLcNodeScan)
    ▼
LcNode
    ├── LcScanner (生命周期扫描器)
    │   ├── 遍历目录树
    │   ├── 检查过期规则
    │   └── 执行删除/迁移
    │
    └── SnapshotScanner (快照版本删除)
```

## 生命周期规则

支持 S3 BucketLifecycleConfiguration 规则：

| 规则类型 | 说明 |
|---------|------|
| Expiration | 对象过期后自动删除 |
| Transition (HDD) | 从 SSD 迁移到 HDD |
| Transition (EBS) | 从副本存储迁移到纠删码存储 |

## 扫描流程

```
Master 下发 OpLcNodeScan:
    · Volume 名称
    · 生命周期规则
    · 前缀过滤
    ▼
LcScanner.Start():
    ├── FindPrefixInode() → 定位扫描起点
    ├── 遍历目录树:
    │   ├── handleDirChan → 递归处理子目录
    │   └── handleFileChan → 处理每个文件
    ▼
handleFile(inode):
    ├── inodeExpired(rule) → 检查是否满足规则
    ├── 如果过期:
    │   └── DeleteWithCond_ll → 条件删除
    ├── 如果需要迁移到 HDD:
    │   └── transitionMgr.migrate() → 修改存储类别
    └── 如果需要迁移到 EBS:
        └── transitionMgr.migrateToEbs():
            ├── 从 ExtentClient 读取数据
            ├── 写入 BlobStore (ebsClient)
            └── UpdateExtentKeyAfterMigration
```

## 与 Master 的交互

1. LcNode 启动时向 Master 注册
2. Master 定期检查生命周期规则，生成扫描任务
3. 通过 `OpLcNodeScan` 下发任务到 LcNode
4. LcNode 完成后上报结果
