# RFC-0004：AsterGit Repository Storage and Git Transport

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-25 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md) |
| 影响范围 | bare repository、SSH、Smart HTTP、refs、push、fetch、GC、镜像、存储和并发 |

## 1. Git 是协议和存储权威

AsterGit 使用标准 Git 客户端、标准 wire protocol 和系统 Git 的 `upload-pack`、`receive-pack`、`http-backend`。不自定义 object 格式，不把 SQL 表拼成 Git server，不复制 pack negotiation。

```text
repository metadata DB
  -> stable repository id, owner, policy, placement

repository storage
  -> HEAD, refs, objects, packs, alternates and Git config
```

display path 只作为路由输入；实际路径由 repository id/placement 解析，禁止路径穿越和用户控制的 shell 参数。

## 2. Transport

### SSH

OpenSSH 将 public key 映射到 user/key id，forced command 只接受结构化的：

```text
git-upload-pack 'namespace/repository.git'
git-receive-pack 'namespace/repository.git'
```

禁止 PTY、port forwarding、agent forwarding 和交互 shell。Git gateway 对命令、路径、环境变量、进程数、IO 和超时设上限。

### Smart HTTP

HTTP gateway 支持 `info/refs`、`git-upload-pack` 和 `git-receive-pack`，使用 Git smart protocol；协议层可以启用 protocol v2 和能力协商。HTTP 认证、CSRF、限流和 body timeout 由 AsterGit transport policy 管理。匿名 read 与 authenticated write 是独立决策。

## 3. Repository placement

```text
RepositoryDirectory
├── repository_id
├── placement_id
├── primary_generation
├── writer_epoch
├── replica generations
├── maintenance state
└── fencing lease
```

Placement directory 是产品数据库中的元数据权威；repository 内容的 generation/ref 仍由 Git storage 证明。API gateway 不自行猜节点；每次写入解析 placement，并获得带 epoch 的 writer lease。

## 4. Push transaction

一次 push：

1. authenticate actor 和 key/token；
2. authorize repository、ref、force-push、branch policy；
3. 获取 repository writer lease 和当前 epoch；
4. 启动 `receive-pack`，让 Git 将新 objects 放入 quarantine；
5. 运行 pre-receive/update/reference-transaction policy；
6. 用 epoch/generation 条件提交 refs；
7. 成功后记录 `PushAccepted` outbox，并运行 post-receive 派生任务；
8. lease 失效、policy 复检失败或 hook 失败时，refs 不可见，quarantine 丢弃。

Git 官方 quarantine 语义保证预接收对象在检查通过前不会污染主 object store；AsterGit 的产品 hook 不能在 pre-receive 阶段把 quarantined object 写入另一个 repository。

## 5. 并发和 fencing

同一 repository 同一 epoch 只有一个 writer。多个 read operation 可以并发；不同 repository 可以并行。旧 writer 即使持有过期进程和文件句柄，也必须在 ref commit、成功响应和事件发布前检查 epoch。任何“只靠本地 mutex”的实现只适用于 Standalone，不是 Distributed 证明。

Non-fast-forward、atomic push、delete ref、tag overwrite 和 force push 都必须保留 Git 的可观察结果。AsterGit 可以增加 branch policy，但不能把 Git 客户端看到的失败变成静默成功。

## 6. Maintenance

GC、repack、commit-graph、multi-pack-index、fsck 和 mirror sync 是受 placement 调度的 repository jobs。维护任务必须避开 active receive-pack，或使用 Git 支持的锁和 generation policy；不能手工删除 objects。失败必须可重试且不会把 refs 指向缺失 object。

## 7. Replication and failover

复制以 generation 和 object completeness 为单位。新 primary 激活前必须证明目标副本拥有 refs 所需 objects；promotion 产生新 epoch。异步副本读取必须暴露 lag，不能作为写入前的权威 policy source。

## 8. Large repository controls

支持 per-repository/tenant 的 clone、fetch、push、pack bytes、concurrent process、disk 和 maintenance 配额；上传 body、pack 临时空间和 process lifetime 都有限制。大仓库和 Git LFS 的对象策略在 RFC-0007 单独确定。

## 9. 验收

真实 Git CLI 覆盖 clone/fetch/push、protocol v2、空仓库、并发 push、non-fast-forward、atomic multi-ref、force-push policy、hook rejection、客户端取消、writer crash、epoch fencing、repack/GC、镜像和从备份恢复。
