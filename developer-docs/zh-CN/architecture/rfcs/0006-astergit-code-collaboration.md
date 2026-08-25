# RFC-0006：AsterGit Code Collaboration

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-26 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md)、[RFC-0003](0003-astergit-identity-organizations-and-authorization.md)、[RFC-0004](0004-astergit-repository-storage-and-git-transport.md) |
| 影响范围 | Issue、Pull Request、Review、Merge、branch policy、通知和代码读模型 |

## 1. 目标

AsterGit 的“代码协作”不是把 Issue 和 Pull Request 作为 repository 页面上的附属表。它们必须与 Git generation、commit、ref policy 和权限建立可验证的关系，同时允许代码浏览、评论、diff 和通知独立扩展。

本 RFC 定义产品语义，不规定最终 UI。HTTP、WebSocket、CLI 和未来第三方 API 都通过同一 use case contract 进入协作域。

## 2. 领域模型

```text
Repository
├── Issue
│   ├── timeline comments
│   ├── labels / milestone / assignees
│   └── state: open -> closed
├── PullRequest
│   ├── source repository/ref/generation
│   ├── target repository/ref/generation
│   ├── commits / diff snapshot
│   ├── review decisions
│   ├── checks/statuses
│   └── state: draft -> open -> merged/closed
└── BranchPolicy
    ├── required reviews
    ├── required checks
    ├── linear-history/merge strategy
    └── force-push/delete rules
```

Issue number、PR number 和评论 id 在 repository 内稳定递增，但跨 repository 不要求全局连续。删除用户不能删除历史作者身份；作者、reviewer 和 merger 使用不可变 user id 与显示名快照。

## 3. Pull Request 语义

创建 PR 时保存：source/target repository id、source ref、target ref、observed source/target generation、base commit、head commit、创建者、权限策略 revision 和状态。source branch 后续移动不会静默改变 PR 的比较对象；系统通过同步任务更新 observed head，并将 force-push 或 deleted ref 作为明确的 timeline event。

Diff 计算是可重建 projection，不是 PR 的第二个 Git 权威。每次 diff 绑定 base/head object id、merge base 算法版本和 repository generation；对象不可用、历史重写或算法升级时，状态变为 stale/recomputing，而不是展示上一版 diff 冒充当前结果。

## 4. Review 和 checks

Review decision 绑定 reviewer、commit id、policy revision 和 result。`Approve` 不能跨越新的 head commit、权限撤销或 branch policy revision 自动复用。Comment 可以绑定 file path、line range 和 blob/commit id；上下文消失后保留历史内容并标记 outdated。

Checks/statuses 使用独立的 check run id、head commit、source、conclusion 和 observed generation。外部 CI 可以写入 status，但不能直接修改 merge 状态；merge service 统一计算当前门禁。

## 5. Merge 和 merge queue

Merge 是 repository service 的一次受保护写操作，不是 API 先写 `merged_at` 再后台执行 Git 命令：

```text
authorize merge
  -> re-read target ref and branch policy
  -> verify current head, review and checks
  -> reserve merge slot / writer epoch
  -> create merge/squash/rebase result in quarantine
  -> atomically update target ref
  -> commit PR state + merge metadata + outbox
```

如果 target ref、policy、required check 或 writer epoch 在提交前变化，merge 必须失败并要求重算。Merge queue 可以串行化同一 target branch 的候选，也可以在临时 ref 上 speculative merge；队列结果必须绑定被验证的 base generation，过期后自动失效。

支持的 merge strategy 由 branch policy 明确：merge commit、squash、fast-forward 或 rebase。服务器不得把客户端看到的 strategy 静默替换为另一种结果。

## 6. Issue 与协作事件

Issue、PR、review、label、milestone、assignee 和 policy 变化写入产品事务并生成版本化事件。timeline 是可查询的审计友好 projection，不允许通过删除评论破坏已发生的权限和 merge 证据；编辑和隐藏操作保留审计记录。

通知和 webhook 通过 RFC-0005 的 outbox/worker 处理。通知失败不回滚已经提交的 Issue/PR 状态；需要同步门禁的 check 则属于 merge policy，必须在 merge 前权威复检。

## 7. 高并发和缓存

- 浏览和 diff 查询可以走带 generation 的读模型；显示 lag 和 stale 状态。
- 同一 PR 的状态转换使用 optimistic revision 或数据库条件更新，禁止最后写入者覆盖前一状态。
- 同一 target branch 的 merge 由 repository writer/queue 串行化；不同 repository/branch 可以并行。
- 计数、贡献者和搜索是可重建 projection，不作为 merge/permission 权威。

## 8. 验收

覆盖并发评论和状态变更、source force-push、target fast-forward、权限撤销、policy 修改、required check 过期、merge queue 重排、冲突、取消、重复提交、webhook 重试、diff stale/recompute 和完整 timeline 审计。
