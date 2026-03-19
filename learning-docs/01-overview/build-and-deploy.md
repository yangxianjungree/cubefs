# CubeFS 构建与部署指南

## 项目构建

### 前置依赖

CubeFS 使用 CGO 编译，需要以下本地依赖库：
- **zlib**: 数据压缩
- **bzip2**: 数据压缩
- **lz4**: 快速压缩
- **zstd**: Zstandard 压缩
- **snappy**: Google 快速压缩
- **gperftools (tcmalloc)**: 内存分配器
- **RocksDB 6.3.6**: 持久化 KV 存储

这些依赖的源码存放在 `depends/` 目录下，由构建脚本自动编译。

### 构建命令

项目根目录的 `Makefile` 定义了所有构建目标：

```bash
# 构建全部组件
make build

# 单独构建各组件
make server      # cfs-server（Master/MetaNode/DataNode/AuthNode/ObjectNode/LcNode/FlashNode）
make client      # cfs-client（FUSE 客户端）
make cli         # cfs-cli（命令行工具）
make authtool    # cfs-authtool（认证工具）
make libsdk      # libcfs.so（C 共享库，Java SDK 使用）
make fsck         # cfs-fsck（文件系统检查工具）
make fdstore     # fdstore（文件描述符存储）
make bcache      # cfs-bcache（块缓存）
make blobstore   # BlobStore 全部组件（clustermgr/blobnode/access/scheduler/proxy）
make deploy      # cfs-deploy（部署工具）

# 测试
make test        # 运行单元测试
make testcover   # 运行覆盖率测试

# 清理
make clean       # 清理构建产物
make dist-clean  # 深度清理（包括本地依赖库）
```

### 构建产物

所有二进制文件输出到 `build/bin/` 目录：

| 文件 | 说明 | 入口代码 |
|------|------|---------|
| `cfs-server` | 统一服务端二进制 | `cmd/cmd.go` |
| `cfs-client` | FUSE 客户端 | `client/fuse.go` |
| `cfs-cli` | CLI 工具 | `cli/cli.go` |
| `cfs-authtool` | 认证工具 | `authnode/authtool/authtool.go` |
| `cfs-fsck` | 文件系统检查 | `tool/fsck/main.go` |
| `cfs-deploy` | 部署工具 | `deploy/deploy_cli.go` |
| `cfs-bcache` | 块缓存 | `client/blockcache/cmd.go` |
| `libcfs.so` | C 共享库 | `client/libsdk/libsdk.go` |

BlobStore 组件（位于 `build/bin/blobstore/`）：
| 文件 | 说明 |
|------|------|
| `clustermgr` | 集群管理器 |
| `blobnode` | Blob 存储节点 |
| `access` | 访问层 |
| `scheduler` | 调度器 |
| `proxy` | 代理 |

### 构建脚本详解

核心构建逻辑在 `build/build.sh` 中：

1. **编译本地 C/C++ 依赖**: zlib -> bzip2 -> lz4 -> zstd -> snappy -> tcmalloc -> RocksDB
2. **设置 CGO 环境变量**: `CGO_CFLAGS`, `CGO_LDFLAGS` 指向本地编译的库
3. **执行 Go 编译**: `go build` 带 CGO 链接

## Docker 部署

### Docker 相关文件

```
docker/
├── Dockerfile          # 基础镜像（Go 1.18, RocksDB 等）
├── Dockerfile-cfs      # CubeFS 镜像
├── Dockerfile-ltp      # LTP 测试镜像
├── docker-compose.yml  # 主 compose 配置
├── docker-compose-ci.yml  # CI compose 配置
└── run_docker.sh       # Docker 操作脚本
```

### 使用 Docker 启动集群

```bash
# 构建并启动 Docker 集群
make docker

# 或手动操作
docker/run_docker.sh --build   # 构建镜像
docker/run_docker.sh --clean   # 清理
```

### 最小集群组成

一个最小可运行的 CubeFS 集群需要：

| 组件 | 最少实例数 | 说明 |
|------|-----------|------|
| Master | 3 | Raft 要求奇数个节点 |
| MetaNode | 3 | 保证元数据副本数 |
| DataNode | 3 | 保证数据副本数 |

可选组件：
| 组件 | 说明 |
|------|------|
| ObjectNode | 需要 S3 兼容访问时部署 |
| AuthNode | 需要认证时部署 |
| LcNode | 需要生命周期管理时部署 |
| FlashNode | 需要分布式缓存时部署 |
| Console | 需要 Web 管理界面时部署 |

## 配置文件

CubeFS 使用 JSON 格式配置文件，通过 `-c` 参数指定：

```bash
cfs-server -c /path/to/config.json
```

### 配置文件关键字段

```json
{
    "role": "master",          // 角色：master/metanode/datanode/objectnode/...
    "localIP": "192.168.0.1",  // 本机 IP
    "bindIp": false,           // 是否绑定 IP
    "logDir": "/var/log/cfs",  // 日志目录
    "logLevel": "info",        // 日志级别：debug/info/warn/error/critical
    "prof": "17010",           // pprof 端口
    "buffersTotalLimit": 0     // 缓冲区总限制（0 表示不限制）
}
```

各角色有额外的特定配置项（如 Master 的 `peers`、DataNode 的 `disks` 等），详见 `docs/source/ops/configs/` 下的配置文档。

## 升级顺序

当需要升级集群时，建议按以下顺序进行：

```
flashnode → master → datanode → metanode → objectnode → lcnode → cli → client
```

## CI/CD

项目使用 GitHub Actions 进行持续集成：

| 工作流 | 文件 | 说明 |
|--------|------|------|
| CI | `.github/workflows/ci.yml` | 构建 + 单元测试 |
| PR Check | `.github/workflows/check_pull_request.yml` | PR 检查 |
| CodeQL | `.github/workflows/codeql.yml` | 代码安全扫描 |
| Release | `.github/workflows/goreleaser.yml` | 自动发布 |
| SLSA | `.github/workflows/slsa-releaser.yml` | 供应链安全 |

## 开发工具

- **代码格式化**: `gofumpt`
- **Lint**: `golangci-lint` v1.43.0
- **Mock 生成**: `mockgen`（用于生成 raftstore 的 mock）
- **Protobuf**: `gogo/protobuf`（proto 文件编译）
