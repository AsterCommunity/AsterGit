# RFC-0008：AsterGit Availability, Backup and Disaster Recovery

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-26 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md)、[RFC-0002](0002-astergit-technology-stack-and-deployment-profiles.md)、[RFC-0004](0004-astergit-repository-storage-and-git-transport.md) |
| 影响范围 | 副本、placement、故障切换、备份、恢复、跨区域、升级、RPO/RTO 和运维验收 |

## 1. 可用性模型

AsterGit 不用一个“HA”标签掩盖不同部署能力。每个 profile 必须声明保护对象、RPO、RTO、可接受的读写中断和人工介入点：

| Profile | 保护重点 | 典型恢复 |
| --- | --- | --- |
| Standalone | 进程和单机数据 | restart + local backup restore |
| Single-node HA | 宿主机/磁盘损坏 | systemd/container restart + external backup |
| Distributed | API、worker、repo node 故障 | placement failover + fenced promotion |
| Multi-zone | AZ/区域级故障 | replicated repo + DB/object recovery |

具体 RPO/RTO 属于部署配置和运营承诺，不能从“使用 PostgreSQL/S3”自动推导出来。

## 2. Repository placement 和副本

每个 repository 有 primary/replica generation、writer epoch、placement revision 和 replication health。副本状态至少区分：

```text
healthy -> catching_up -> ready_for_read -> ready_for_promotion
       -> stale -> quarantined -> removed
```

只有明确标记为 `ready_for_promotion` 的副本可被提升。提升先写入新的 epoch/fencing record，再开放 writer；旧 primary 即使网络恢复也只能以旧 epoch 读或被隔离。

## 3. 故障切换

```text
detect failure
  -> stop/evict old writer or establish fencing authority
  -> verify replica object completeness and ref generation
  -> allocate new writer epoch
  -> promote one candidate
  -> reconcile outbox/audit/generation
  -> publish placement revision
  -> reopen writes
```

无法确认旧 writer 已经被隔离时，系统宁可暂停该 repository 写入，也不允许两个 writer 同时接受 push。未知提交结果必须通过 ref/generation 查询和 recovery audit 判定。

## 4. 备份矩阵

必须分别备份：

| 数据 | 备份内容 | 验证 |
| --- | --- | --- |
| Git | refs、objects、pack、config、bundle/快照 | `fsck`、bundle verify、ref/object closure |
| Product DB | schema、数据、migration version | 临时实例 restore、查询 smoke、revision 对账 |
| Artifact | immutable object、manifest、digest、retention | 随机抽样和完整 manifest 校验 |
| Secrets/config | 加密配置、key id、部署 manifest | 在隔离环境解密并启动最小服务 |
| Outbox/audit | 未投递事件、审计和 generation | replay、幂等和时间线对账 |

只上传 tar 或只看到 S3 object count 都不算备份通过。恢复演练必须从空目录/临时数据库开始，证明用户能 clone、ref 完整、权限仍然有效、事件可补偿。

## 5. RPO/RTO 和降级

API、读 projection、worker、runner、webhook 和搜索可以分别降级。Git read/write 的可用性由 repository placement 和 storage generation 决定；不要让搜索、通知或 CI 故障阻塞已授权的 clone/fetch/push，除非明确配置为同步门禁。

恢复过程要公开状态：repository read-only、replication lag、backup restore、workflow unavailable 和 projection stale 都应该有机器可读健康状态和管理员审计。

## 6. 升级与回滚

数据库 migration、repository maintenance、event schema 和 Git service binary 需要兼容窗口：

1. 发布兼容 reader/writer；
2. 迁移可观测且可暂停；
3. 验证新的 artifact digest、schema 和 health；
4. 滚动切换 API/worker；
5. 最后切换需要新 contract 的 writer/runner；
6. 失败时回到已验证 artifact，保留新数据和 migration 回滚/forward-fix 路径。

禁止把 `kubectl rollout undo` 或重新启动进程当成数据恢复策略。不可逆 migration 必须在执行前有可验证备份和明确人工确认边界。

## 7. 演练和验收

至少定期演练：进程崩溃、节点断电、磁盘满、数据库不可用、对象存储不可用、网络分区、旧 writer 复活、replica lag、误删 repository、密钥轮换、完整 restore 和跨区域 promotion。每次记录实际 RPO/RTO、丢失/重放事件和残留人工步骤。
