# RFC-0001：AsterGit Platform Architecture

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-25 |
| 负责产品组 | AsterGit Platform |
| 影响范围 | 产品边界、模块所有权、数据权威、部署、高可用、扩展和恢复 |
| 相关契约 | [`project-contract.md`](../project-contract.md)、[RFC-0002](0002-astergit-technology-stack-and-deployment-profiles.md)、[RFC-0003](0003-astergit-identity-organizations-and-authorization.md)、[RFC-0004](0004-astergit-repository-storage-and-git-transport.md)、[RFC-0005](0005-astergit-workers-runners-events-and-artifacts.md) |

## 1. 摘要

AsterGit 是以 Git 仓库和代码协作为核心的自托管软件开发 forge。它不是只接受 SSH push 的 Git server，也不是把所有能力强行塞进单一进程的 GitHub 克隆。单机、单节点高可用、分布式和大规模部署是同一产品契约的不同 profile。

核心决策：

1. Git repository 是核心业务资源；Git objects、refs、HEAD、pack 和原生 ref transaction 属于 repository storage 权威。
2. 产品数据库拥有用户、组织、团队、权限、协作实体、任务、审计和 placement 元数据，不与 Git refs 双写。
3. API 与读模型可以水平扩展；一个 repository 在同一 fencing epoch 内只有一个 ref writer。
4. repository placement、lease、epoch 和 generation 是高并发与高可用的核心控制机制。
5. worker 处理可信后台任务；runner 执行不可信用户 workflow，必须隔离部署和授权。
6. 事件通过 outbox 和幂等消费传播；异步系统故障不能静默改变 Git push 的结果。
7. AsterForge 提供共享运行时和基础设施机制，AsterGit 保留产品语义。

## 2. 产品平面

```text
AsterGit Platform
├── Git Data Plane
│   ├── SSH / Smart HTTP gateway
│   ├── repository placement
│   ├── receive-pack / upload-pack
│   ├── ref policy and maintenance
│   └── mirror / import / export
├── Collaboration Plane
│   ├── users / organizations / teams
│   ├── issues / pull requests / reviews
│   ├── branches / tags / releases
│   └── notifications / webhooks / audit
├── Execution Plane
│   ├── trusted workers
│   ├── isolated workflow runners
│   └── artifact processing
└── Platform Plane
    ├── API and Web
    ├── identity and authorization
    ├── database and event outbox
    ├── placement directory
    ├── observability
    └── backup / restore / upgrade
```

这些是 ownership boundary，不要求每个平面一开始都运行成独立进程。Standalone 可以将它们组合在一个 binary；Distributed profile 通过相同的 port/contract 拆分。

## 3. 核心模块

| 模块 | 权威职责 | 不承担 |
| --- | --- | --- |
| `identity` | account、credential、SSH key、组织和团队身份 | Git ref、Issue 状态、runner 执行 |
| `authorization` | repository/action/resource 的 allow/deny 决策 | HTTP 文案、数据库事务编排 |
| `repository` | Git storage、transport、ref policy、maintenance | 用户组织规则、PR 状态 |
| `collaboration` | Issue、PR、review、merge、branch policy | pack 解析、socket 写入 |
| `placement` | repository owner、replica、writer lease、epoch、generation | 业务权限和 Git object 内容 |
| `worker` | webhook、通知、索引、备份、同步等可信任务 | 用户提交代码执行 |
| `runner` | 隔离执行 workflow，上传 log/status/artifact | 主数据库任意写入、直接改 refs |
| `artifact` | release、attachment、package、workflow blob | 在线 repository refs |
| `web/api` | 对外 HTTP、Web、OpenAPI、session | 直接绕过 service 写数据库 |

## 4. 数据流和一致性

一次 push 的权威路径是：

```text
authenticate
  -> authorize repository/write/ref policy
  -> resolve placement and writer epoch
  -> run receive-pack in quarantine
  -> validate hooks and reference transaction
  -> commit refs with epoch/generation precondition
  -> append durable PushAccepted outbox record
  -> return Git success
```

Push 成功不等于 webhook、索引、通知或 CI 已完成；这些是带有 observed generation 的异步后续状态。Push 失败或结果未知时，客户端必须能通过 ref 查询和 audit/recovery 识别实际状态，不能用重试把一个未知提交伪装成确定失败。

## 5. 高并发模型

- API request 不持有 repository writer lock 处理普通读请求。
- upload-pack 读操作按 snapshot/generation 读取；GC、repack 和写入使用 Git 原生锁及 placement maintenance policy。
- receive-pack 以 repository 为最小写入串行单元；不同 repository 可以并行，不同 shard 可以水平扩展。
- 热门 repository 不通过无限加 API 副本解决写竞争；通过 queue、fair scheduling、pack 限额和写入 backpressure 控制。
- 每个连接、请求、worker、runner 和事件 consumer 都有明确上限、deadline、取消和指标。
- 单个租户或 repository 的资源配额不能被全局线程池抢空。

## 6. 高可用模型

API 可以无状态多副本；repository writer 通过 lease + fencing epoch 选主。副本复制、故障切换和恢复必须证明：

- 旧 writer 无法在新 epoch 写 refs；
- 已提交 generation 的 objects 在 ref 可见前已经可读；
- outbox 可以从数据库或 Git generation 重建/补偿；
- 读副本明确报告 generation/lag，不把陈旧 projection 当作权威；
- 单 AZ、节点、磁盘、数据库和 worker 故障都有受支持的恢复路径。

不承诺“任何组件都零中断”。承诺的是边界清晰、状态可判定、失败可恢复、旧持有者不能继续写入。

## 7. 版本和演进

跨进程 contract 使用版本化 DTO、capability 和 migration。repository refs、Git protocol 和协作状态机必须向后兼容或通过明确的升级窗口切换。内部模块重构一次迁移全部调用方，不留下没有产品边界价值的转发层。

## 8. 验收矩阵

必须覆盖：多用户并发 clone/fetch/push、non-fast-forward、ref policy、writer crash、epoch fencing、replica lag、outbox replay、worker 重启、runner 失联、数据库恢复、repository restore 和 rolling upgrade。
