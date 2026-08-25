# AsterGit 项目契约

本文定义 AsterGit 的长期产品边界、权威状态、依赖方向和高可用不变量。RFC 可以扩展实现方案，但不能静默改变这些边界。

## 产品身份

AsterGit 是自托管软件开发 forge，面向个人、小团队和组织。Git 仓库是核心资源，代码协作是核心产品行为。用户、组织、团队、权限、Issue、Pull Request、Review、Merge、Webhook、通知、Release、审计和恢复围绕 repository 组成统一产品。

```text
Client
  -> HTTP/API or SSH Git transport
  -> identity + authorization
  -> repository placement
  -> Git operation / collaboration use case
  -> durable state + event outbox
  -> projection, notification, webhook, worker or runner
```

## Deployment Profile

单机、单节点高可用、分布式和大规模部署是同一份 contract 的 profile：

| Profile | 目标 | 典型实现 |
| --- | --- | --- |
| Standalone | 最小运维、完整核心能力 | 单进程、SQLite、本地 FS、embedded worker |
| Single-node HA | 宿主机服务可重启、数据可恢复 | systemd、PostgreSQL 或 SQLite、快照、S3 backup |
| Distributed | API/worker/repository 独立扩展 | PostgreSQL、queue/event bus、repo placement、object storage |
| Large | 多租户、高吞吐、跨可用区 | 多 API、worker pool、repo shard、同步副本、独立 runner |

Profile 不能改变用户看到的权限、Git ref 更新、merge 结果或事件语义。性能和容量目标可以不同，但失败必须显式返回或进入可观测的异步状态。

## 权威来源

```text
Git repository storage
  = objects, refs, HEAD, pack and receive-pack transaction authority

Product database
  = accounts, organizations, permissions, issues, pull requests,
    reviews, jobs, policies, audit metadata and placement authority

Artifact/backup storage
  = releases, packages, workflow artifacts and recovery copies
```

数据库不能镜像 refs 后再与 Git 目录双写。协作记录引用 Git object/ref 时，必须保存 observed commit/ref identity 和 generation，读取时明确是否允许 projection lag。

## 核心不变量

- 每个 repository 在一个 fencing epoch 内只有一个 ref writer。
- ref 更新必须使用 Git 原生 quarantine、hook、reference transaction 和原子 ref 语义；校验失败不得留下可见 refs。
- placement lease 过期后，旧 writer 即使继续运行也不能提交 refs 或发布成功事件。
- API、worker 和 runner 不绕过 repository service 直接修改在线 bare repository。
- 事件采用 outbox、幂等 consumer 和可重放记录；不能把“消息发出”伪装成事务已经完成。
- 慢消费者、失联 worker、不可用 runner、落后的 projection 不得阻塞 Git push 的权威提交路径，除非产品策略明确要求同步门禁。
- 备份必须能验证并完成真实恢复，而不是只证明对象上传成功。

## 所有权

### AsterGit 拥有

用户与组织语义、repository 元数据、授权决策、Git transport policy、Issue/PR/Review/Merge、webhook 语义、产品审计、任务定义、runner contract、制品引用和部署 profile。

### AsterForge 可以提供

runtime lifecycle、配置、数据库连接与 migration 机械层、通用任务 lease、audit log plumbing、日志、指标、限流、健康检查和测试支持。Forge 不定义 AsterGit 的 repository、PR、权限继承或 workflow payload。

## 禁止的捷径

- 不用 S3 FUSE 作为在线 Git repository 的一致性实现。
- 不把 Redis lock 当作 repository writer fencing 的唯一权威。
- 不在 API 进程内执行不可信 workflow。
- 不为兼容旧内部路径保留没有边界价值的 facade。
- 不用“最终一致”掩盖公开 Git push 已经成功或失败的未知状态。

## 完成标准

任何实现阶段都至少需要证明成功、失败、并发、取消、恢复和权限边界。企业 Profile 还要证明故障转移、旧 epoch fencing、复制延迟、replay、backup restore、容量上限和升级回滚。
