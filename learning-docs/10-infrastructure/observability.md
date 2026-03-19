# 监控与可观测性

## Prometheus 指标导出

### 初始化

```go
// util/exporter/exporter.go
func Init(role string, cfg *config.Config) {
    // 1. 解析配置: exporterEnable, exporterPort, consulAddr, pushAddr
    // 2. 启动 HTTP Server: /metrics
    // 3. 注册 Consul（可选）
    // 4. 设置 namespace: cfs_<role>
}
```

### 指标类型

| 类型 | 工具类 | 使用场景 |
|------|--------|---------|
| Counter | `NewCounter(name)` | 累计计数（请求数、错误数） |
| Gauge | `NewGauge(name)` / `NewGaugeVec(name, labels)` | 瞬时值（连接数、内存使用） |
| Histogram | `NewTPCnt(name)` | 请求延迟分布 |
| Alarm | — | 告警指标 |

### 使用示例

```go
// 延迟统计
tp := exporter.NewTPCnt("meta_create_inode")
defer tp.Set(err)  // 自动记录延迟和成功/失败计数

// 计数器
counter := exporter.NewCounter("data_write_bytes")
counter.Add(float64(writeSize))

// 仪表盘
gauge := exporter.NewGauge("meta_partition_count")
gauge.Set(float64(len(partitions)))
```

### 各模块指标

| 模块 | 典型指标 |
|------|---------|
| Master | 集群容量、节点数、DP/MP 数量 |
| MetaNode | inode/dentry 操作延迟、分区数、内存使用 |
| DataNode | 读写延迟、磁盘使用率、Extent 数量 |
| ObjectNode | S3 请求延迟、吞吐量 |
| SDK | 客户端操作延迟 |

## 日志系统

### 架构

```go
// util/log/log.go
type Log struct {
    debugLogger    *LogObject
    infoLogger     *LogObject
    warnLogger     *LogObject
    errorLogger    *LogObject
    readLogger     *LogObject   // 读操作日志
    updateLogger   *LogObject   // 写操作日志
    criticalLogger *LogObject
    qosLogger      *LogObject
}
```

### 日志级别

| 级别 | 函数 | 说明 |
|------|------|------|
| Debug | `LogDebugf()` | 调试信息 |
| Info | `LogInfof()` | 一般信息 |
| Warn | `LogWarnf()` | 警告 |
| Error | `LogErrorf()` | 错误 |
| Critical | `LogCriticalf()` | 严重错误 |
| Read | `LogRead()` | 读操作记录 |
| Write | `LogWrite()` | 写操作记录 |

### 日志特性

- **异步写入**: 4MB 缓冲区，1 秒定时刷新
- **日志轮转**: 按大小轮转，可配置大小和保留空间
- **动态级别**: 通过 HTTP API `/loglevel/set` 动态调整
- **磁盘空间保护**: 当磁盘剩余空间低于阈值时降低日志级别

## 审计日志

### 架构

```go
// util/auditlog/auditlog.go
type Audit struct {
    logDir     string
    logModule  string
    logMaxSize int64
    bufferC    chan string  // 异步写入 channel (100K 容量)
    writer     *bufio.Writer
}
```

### 审计事件类型

| 函数 | 说明 |
|------|------|
| `LogClientOp` | 客户端操作（op, src, dst, err, latency） |
| `LogDentryOp` | 目录项操作 |
| `LogInodeOp` | Inode 操作 |
| `LogTxOp` | 事务操作 |
| `LogMasterOp` | Master 操作 |
| `LogDataNodeOp` | DataNode 操作 |
| `LogMigrationOp` | 数据迁移操作 |

### 特性

- 异步写入，不阻塞业务
- 按大小轮转
- 支持通过 HTTP API 动态启用/禁用
- 超过 channel 容量时丢弃（不阻塞业务）

## Consul 注册

支持将服务注册到 Consul，用于服务发现和 Prometheus 的服务端抓取。
