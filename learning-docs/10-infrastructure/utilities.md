# 工具库

## 流控 (util/flowctrl/)

### Controller（令牌桶）

```go
type Controller struct {
    rate   int          // 每秒令牌数
    tokens int          // 当前令牌数
    ch     chan struct{} // 令牌 channel
}
```

- 基于令牌桶算法，50ms 间隔补充令牌
- `acquire(size)`: 消费令牌，不足时阻塞等待
- `fill(size)`: 归还未使用的令牌
- `Reader(r io.Reader)` / `Writer(w io.Writer)`: 包装 I/O 为限速版本

### KeyFlowCtrl（按 Key 限流）

```go
type KeyFlowCtrl struct {
    controllers map[string]*refController  // key → 控制器 + 引用计数
}
```

用于 ObjectNode 的按用户/桶限流。

## 缓冲池 (util/buf/ & util/bytespool/)

### BufferPool

预分配固定大小的字节缓冲区池：
- 减少 GC 压力
- 适用于 Packet 的 Data 字段

### BytesPool

```go
var pools = [...]sync.Pool{
    {New: func() interface{} { return make([]byte, 512) }},
    {New: func() interface{} { return make([]byte, 1024) }},
    // ... 不同大小的预分配池
}
```

根据请求大小选择合适的池，避免频繁内存分配。

## 错误处理 (util/errors/)

### 增强型错误

```go
func NewError(msg string) error
func NewErrorf(format string, args ...interface{}) error
```

### Panic Hook

```go
func AtPanic(fn func()) error
func SupportPanicHook() bool
```

注册 panic 时的回调函数，用于在 panic 前刷新日志。

## 限流器 (util/ratelimit/)

提供 IO 限流功能：

```go
type IoLimiter struct {
    readLimiter  *rate.Limiter
    writeLimiter *rate.Limiter
}
```

用于 DataNode 的磁盘 I/O 限流（读/写/删除分别限制）。

## 配置管理 (util/config/)

```go
type Config struct {
    data map[string]interface{}
}

func LoadConfigFile(path string) (*Config, error)
```

JSON 配置文件解析，支持 `GetString`, `GetInt`, `GetBool`, `GetFloat`, `GetSlice` 等方法。

## 其他工具

| 包 | 说明 |
|----|------|
| `util/atomic` | 原子操作封装（Float64, Flag） |
| `util/crypto` | 加密工具 |
| `util/ump` | UMP 监控告警 |
| `util/sys` | 系统工具（CPU/内存信息、fd 重定向） |
| `util/stat` | 统计信息收集 |
| `util/timeutil` | 时间工具 |
| `util/unit` | 存储单位转换 |
