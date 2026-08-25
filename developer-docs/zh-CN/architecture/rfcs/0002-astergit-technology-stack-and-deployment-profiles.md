# RFC-0002：AsterGit Technology Stack and Deployment Profiles

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-25 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md) |
| 影响范围 | Rust workspace、HTTP、数据库、事件、存储、部署、观测和供应链 |

## 1. 技术栈原则

Rust 2024、Tokio、tracing 和 AsterForge 是基础。对外 HTTP/Web 使用现有 Aster 生态的 Actix/Tower 约定；内部服务通信可以采用版本化 gRPC/Protobuf。具体框架版本由 workspace 根 `Cargo.toml` 和 lockfile 统一治理，不在 RFC 中伪造永久版本号。

原则：

- 数据库、消息、对象存储和 runner 都有显式 adapter，不把外部系统类型泄漏到 domain；
- 同一 contract 提供 embedded/local 和 remote implementation；
- 默认 feature 保持小，Standalone 不隐式拉起大规模基础设施；
- schema、API、protobuf 和事件都有 drift/breaking 检查；
- 所有网络调用声明 deadline、retry、幂等键、取消和 unknown outcome。

## 2. Profile

### 2.1 Standalone

```text
astergit
├── HTTP/API + Web
├── SSH/Smart HTTP Git transport
├── embedded repository service
├── SQLite
├── local POSIX repository/artifact storage
└── embedded trusted worker
```

这是完整核心产品，不是 mock。它适合个人、小团队、开发机和低运维环境。可以配置 S3-compatible backup，但在线 repository 仍使用 POSIX storage。

### 2.2 Single-node HA

目标是进程和宿主机故障后的快速恢复，而不是宣称多节点无损 HA：systemd/container restart、外部 PostgreSQL（可选）、本地/网络卷、S3 versioned backup、restore rehearsal 和健康探针必须完整。对于广州这类单机部署，此 profile 是实际的第一生产落点。

### 2.3 Distributed

```text
edge -> API replicas -> service contracts
                         ├── PostgreSQL
                         ├── event/outbox transport
                         ├── repository placement + writer leases
                         ├── worker pool
                         ├── isolated runner pool
                         └── artifact/backup object storage
```

API 可横向扩展；repository 写入按 placement 分片；worker 通过 lease claim；runner 通过短期 registration token 和 job lease 执行。

### 2.4 Large / Multi-zone

多可用区、读副本、跨区域备份和冷热分层属于扩展 profile。repository 的同步复制、切换 RPO/RTO、跨区域写入和冲突策略必须在 RFC-0008 中逐仓库说明，不允许用“数据库多副本”概括 Git storage 一致性。

## 3. 数据系统

| 数据 | Standalone | Distributed/Large | 一致性 |
| --- | --- | --- | --- |
| 产品元数据 | SQLite | PostgreSQL | 事务权威 |
| 在线 Git repository | 本地 POSIX FS | placement-owned POSIX/专用 repo store | ref/object 权威 |
| release/package/artifact | 本地 FS | S3-compatible/object store | content-addressed 或版本化 |
| outbox | SQLite | PostgreSQL + transport | at-least-once、幂等 |
| cache | memory | Redis/本地 cache | 可重建、非权威 |
| backup | bundle + restic/S3 | snapshot + object storage | 可验证恢复 |

禁止将 S3 FUSE 当作 refs/lock/rename 语义的替代品。若未来需要远端 repository storage，必须提供明确的 lock、atomic ref、quarantine、fsck 和 snapshot contract。

## 4. 事件与任务

Standalone 使用进程内 bounded channel + durable outbox；Distributed 可以使用 NATS JetStream、Kafka 或兼容 adapter，但 contract 固定为至少一次投递、幂等消费、可重放和显式 dead letter。消息 broker 不能取代数据库事务，也不能成为 Git ref 权威。

## 5. 观测与运维

所有 profile 至少提供：健康/就绪、repository generation、writer epoch、push/fetch latency、pack bytes、ref transaction result、outbox lag、worker lease、runner queue、replica lag、backup verification 和 restore history。

日志不记录 password、token、私钥、workflow secret、原始 webhook secret 或完整用户源码。审计记录 actor、repository、action、generation、request id 和结果分类。

## 6. 供应链

正式构建需要 lockfile、SBOM、镜像 digest、构建 provenance、依赖许可证扫描和可复现/可审计的发布入口。生产升级采用 migration gate、滚动兼容窗口、健康检查和可回滚 artifact。

## 7. 验证

每个 profile 需要对应的最小可运行验证；Distributed/Large 不能只验证 YAML 渲染。必须实际验证 image、数据库、placement、lease、service route、事件重放、备份恢复和故障注入。
