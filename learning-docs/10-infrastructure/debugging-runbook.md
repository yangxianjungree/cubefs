# 集群调试与故障排查手册

## 目标

这份手册回答 4 个实战问题：

1. 集群有问题时，先看哪里？
2. 每个模块日志在哪？
3. 关键日志怎么搜？
4. 一条请求如何跨模块串起来定位？

## 先记住的排障顺序

发生问题时，建议按下面顺序排查（不要一上来就看 DataNode 代码）：

1. **先判定故障面**：是全局（挂载都失败）还是单卷/单路径问题。
2. **先看 Master**：集群视图、分区状态、节点在线状态是否异常。
3. **再看 MetaNode**：是否是 inode/dentry/事务/分区路由问题。
4. **再看 DataNode**：是否是写入、副本、磁盘、修复问题。
5. **最后看客户端**（FUSE/SDK/ObjectNode）：是否是缓存、重试、鉴权、参数问题。

## 日志目录与文件

`cfs-server` 启动时会初始化：

- 业务日志（`log.InitLog`）
- 审计日志（`auditlog.InitAuditWithHeadRoom`）
- 标准输出重定向（`output.log`）

关键代码在 `cmd/cmd.go`：`LoggerOutput = "output.log"`，并调用 `log.OutputPid(logDir, role)`。

### 通用路径规则

日志根目录由配置 `logDir` 决定，按模块落盘：

- `logDir/<module>/...`

常见模块名：`master`、`metaNode`、`dataNode`、`objectNode`、`authNode`、`lcnode`、`flashNode`。

### 建议重点关注的日志

- `output.log`：进程 stdout/stderr，启动失败/panic 首选。
- `error` / `warn` 级别日志：优先看错误链路。
- `read` / `write` 日志：读写热点与失败模式。
- 审计日志：跨模块操作记录（尤其是删除、迁移、事务）。

## 模块级排查入口

### 1) Master

先确认 Master 是否健康、是否选主正常：

- 看 `master/output.log` 是否有：
  - 启动失败、配置错误、端口占用
  - leader 切换频繁
  - raft apply/snapshot 异常

再看 Master API：

- `/admin/getCluster`
- `/cluster/stat`
- `/admin/cluster/getAllDataNodes`
- `/admin/cluster/getAllMetaNodes`
- `/dataPartition/get`
- `/metaPartition/*` 相关

关注点：

- 某个 volume 的 DP/MP 是否缺副本、状态是否只读。
- 某个节点是否长时间离线。
- 是否存在 decommission/recover 任务卡住。

### 2) MetaNode

适合排查：`ls` 慢、目录读失败、创建/rename/删除失败、事务冲突。

可用接口（`metanode/api_handler.go`）：

- `/getPartitions`
- `/getPartitionById`
- `/getRaftStatus`
- `/setGOGC`、`/getGOGC`

日志关键词建议：

- `create inode` / `create dentry`
- `lookup` / `read dir`
- `tx` / `rollback` / `commit`
- `raft` / `apply` / `snapshot`

### 3) DataNode

适合排查：写入失败、读超时、crc 错误、副本不一致、磁盘异常。

可用接口（`datanode/server_handler.go`）：

- `/stats`
- `/raftStatus`
- `/partition`
- `/setDiskQos`、`/getDiskQos`
- `/setGOGC`、`/getGOGC`

日志关键词建议：

- `extent` / `write` / `read`
- `repair` / `decommission`
- `disk` / `unavailable`
- `raft` / `applied`

### 4) ObjectNode

适合排查：S3 请求失败（403/404/5xx）、签名不通过、桶/对象映射错误。

优先看：

- 鉴权中间件日志（signature、ak/sk）
- policy/acl 拒绝日志
- 后端转发到 MetaWrapper/ExtentClient 的报错

### 5) AuthNode / LcNode / FlashNode

- **AuthNode**：票据、caps、ak/sk、raft keystore 一致性。
- **LcNode**：生命周期扫描任务是否下发、扫描是否卡住、迁移失败原因。
- **FlashNode**：缓存命中率、预热任务、磁盘缓存异常。

## 常用排查命令（Linux）

> 把 `<LOG_DIR>`、`<module>` 替换成实际值。

```bash
# 1) 先看最新错误
rg -n "error|fatal|panic|raft|timeout" "<LOG_DIR>/<module>"

# 2) 盯住最近日志
tail -f "<LOG_DIR>/<module>/output.log"

# 3) 按请求关键词过滤（例如 ReqID、inode、partition）
rg -n "Req\(|inode|partition|extent|tx" "<LOG_DIR>/<module>"

# 4) 对比多个模块同一时间窗口
rg -n "2026-" "<LOG_DIR>/master" "<LOG_DIR>/metaNode" "<LOG_DIR>/dataNode"
```

## 三个高频故障场景

### 场景 A：挂载成功但 `ls` 卡住/报错

1. 看 Master：目标 volume 的 MP 是否健康。
2. 看 MetaNode：`lookup/read dir` 是否报错，Raft leader 是否频繁切换。
3. 看客户端缓存：是否 stale，尝试重挂载或清缓存后复现。

### 场景 B：写入返回成功但读不到新数据

1. 看 DataNode：写入是否真实落盘（extent write）。
2. 看 MetaNode：`OpMetaExtentsAdd` 是否成功，inode extent 列表是否更新。
3. 看 Follower Read：是否读到了旧副本，尝试关闭/绕过 follower read 做对比。

### 场景 C：集群偶发抖动（大量超时）

1. 先看 Master 是否频繁选主。
2. 再看 MetaNode/DataNode 的 Raft 状态与磁盘健康。
3. 检查 QoS/限流是否过严（磁盘 QoS、API QPS）。
4. 看 GC 与内存（必要时临时调 `GOGC` 做验证）。

## 最小化定位方法（建议团队统一）

每次报障，至少收集：

- 时间范围（精确到分钟）
- volume / inode / partition ID
- 客户端错误码与请求路径
- Master + 一个 MetaNode + 一个 DataNode 的对应时间日志

这样可以把“现象”快速收敛到“哪个模块先失败”。

## 与现有文档的关系

- 架构与模块职责：见 `01-overview/architecture.md`
- 协议层：见 `01-overview/protocol.md`
- 监控与日志框架：见 `10-infrastructure/observability.md`
- 端到端流程：见 `11-e2e-flows/`
