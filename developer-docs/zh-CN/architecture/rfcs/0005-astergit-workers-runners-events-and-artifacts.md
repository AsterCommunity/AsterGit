# RFC-0005：AsterGit Workers, Runners, Events and Artifacts

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-25 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md)、[RFC-0002](0002-astergit-technology-stack-and-deployment-profiles.md) |
| 影响范围 | 后台任务、事件、webhook、通知、workflow runner、制品和扩展部署 |

## 1. Worker 与 Runner 的边界

```text
Trusted worker
  = AsterGit 定义的后台机械任务

Untrusted runner
  = 用户 workflow 和任意构建代码的执行环境
```

Worker 可以访问 AsterGit service contract 和受控 repository operation；Runner 不进入主进程，不获得数据库管理员权限，不直接写 refs，不读取其他 repository 的 secret 或 workspace。

## 2. Durable events

Git push、issue/PR 状态、review、merge、release、member、permission、runner 和 artifact 状态变化先写权威事务与 outbox。outbox 记录 event id、aggregate id、observed generation、schema version、tenant scope、created_at 和 delivery state。

```text
transaction
  -> product state + outbox row
  -> claim lease
  -> delivery
  -> idempotent consumer / retry / dead letter
```

Standalone 使用本地 worker；Distributed 可以使用 durable broker。broker 丢失时可以从 outbox 重放；消费者重复时必须按 event id/aggregate generation 幂等。

## 3. Worker jobs

典型任务：webhook、notification、email、search/index、commit statistics、mirror、repository maintenance、backup、release processing 和 artifact cleanup。每个 job 需要：状态、attempt、lease、heartbeat、deadline、cancel、retry policy、dedupe key、错误分类和审计关联。

任务失败不修改已提交 Git ref；post-receive 派生任务丢失时可以从 Git generation/audit/outbox 重建。

## 4. Runner contract

Runner 注册包含短期 token、capability、platform、version、labels、resource limit、network policy 和 trust profile。Job claim 使用 lease/fence；runner 心跳失效后 job 进入可判定的 retry/failed 状态，不能同时由两个 runner 写同一 workspace 或发布同一 artifact。

Runner 至少隔离：

- filesystem/workspace；
- CPU、memory、disk、process、time；
- network egress 和 metadata service；
- secrets 的最小 scope 与短期注入；
- log、status、artifact upload channel；
- cancel/kill 和 cleanup。

Workflow 语法、权限、secret、checks 和 artifact 引用属于 AsterGit 产品 contract；执行器可以替换，不要求主服务理解每个 shell 命令。

## 5. Artifact storage

制品分为 release asset、issue attachment、workflow artifact、package blob 和 backup object。它们拥有 content type、size、digest、owner、retention、visibility、generation 和下载授权。对象上传采用临时 session、校验 digest、完成标记和清理任务，不能把客户端直接上传成功当成产品记录已经完成。

在线 Git repository、artifact 和 backup 是三个不同 storage contract：不能共用一个没有一致性边界的“万能 S3 adapter”。

## 6. Backpressure and isolation

每个 tenant/repository 的 event、worker、runner 和 artifact 配额独立计算。webhook 目标慢时只占用自己的 delivery budget；workflow 高峰不能耗尽 Git transport 线程；backup 和 GC 在低优先级队列运行。队列满时公开可观察地拒绝、延迟或降级，不静默丢事件。

## 7. Observability

任务和 runner 日志带 job id、event id、repository id、attempt、lease epoch、runner id 和结果分类；不记录 token、secret、完整源码或任意用户 payload。指标包括 outbox lag、claim age、retry/dead-letter、runner queue、execution duration、artifact bytes、cleanup backlog 和 webhook delivery rate。

## 8. 验收

覆盖重复事件、broker/worker 重启、lease 过期、cancel、runner 崩溃、secret scope、网络隔离、artifact digest、webhook retry/dead letter、Git push 与 worker/runner 高峰互不拖垮、备份任务限流和恢复重放。
