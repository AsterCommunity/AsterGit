# AsterGit

AsterGit 是面向个人、小团队和自托管组织的软件开发 forge。产品以 Git repository 和代码协作为核心，长期覆盖用户、组织、团队、权限、代码浏览、Issue、Pull Request、Review、Merge、Release、Webhook、通知、制品、审计、备份和 workflow runner。

AsterGit 的单机、单节点 HA、Distributed 和 Large/Multi-zone 是同一产品契约的部署 Profile，不是四套实现。单机需要提供完整核心能力；企业 Profile 需要在同一行为契约上增加水平扩展、repository placement、writer fencing、复制、故障转移、恢复演练和供应链证明。

## 开始工作

涉及代码、协议、数据、部署或架构的任务按以下顺序建立上下文：

1. 读 [`developer-docs/zh-CN/architecture/project-contract.md`](developer-docs/zh-CN/architecture/project-contract.md)，确认产品边界、权威来源和长期不变量。
2. 读 [`developer-docs/zh-CN/architecture/index.md`](developer-docs/zh-CN/architecture/index.md)，按任务读取相关 RFC；跨层任务至少读 RFC-0001、RFC-0002 和对应领域 RFC。
3. 需要理解数据流时读 [`developer-docs/zh-CN/architecture/platform-architecture-diagrams.md`](developer-docs/zh-CN/architecture/platform-architecture-diagrams.md)。
4. 检查当前 branch、HEAD、worktree、远端和实际实现；issue、PR、review comment 或历史测试必须与当前 checkout 对得上。
5. 沿入口到 use case、domain、repository/storage、event/worker 的调用链阅读相邻代码和测试，事实明确后再编辑。

项目以 RFC-first 启动。RFC 是设计事实源，README 是入口摘要，代码、API schema、migration、部署 manifest 和测试必须与 RFC 对齐；如果实现暴露设计缺口，先记录并修订 RFC，不能用代码偷偷改变产品边界。当前实现进度仍以 checkout 和测试为准。

任务开始后默认持续执行到完成标准。状态更新不是等待审批的停顿点；除非涉及产品语义冲突、不可逆数据操作、公开协议破坏或真实外部凭据/环境缺失，否则自行完成侦察、实现、验证和收尾。

## 事实和冲突

- 当前用户任务决定范围和优先级。
- 当前 checkout、代码、测试和远端状态决定已经实现什么；设计 RFC 不等于实现证据。
- `project-contract.md` 定义长期边界；RFC 定义可评审的架构选择；README 不替代它们。
- 官方 Git protocol、Git 文档、数据库/对象存储规范和供应商文档是外部协议事实源。
- 历史记录、旧 issue、旧 benchmark、生成产物和旧部署日志不能替代当前证据。
- 设计、实现和权威规范发生实质冲突时继续调查并向 1547 确认，不凭感觉选一个方向。

## 硬约束

- 保留用户和其他执行者的未提交改动；不使用 reset、checkout 或批量覆盖制造表面干净。
- 只修改任务相关文件；同文件存在交叉修改时先读完整 diff。
- RFC、代码、API、migration、部署和测试必须保持同一产品契约；不要只改其中一层。
- 不手动伪造生成文件、OpenAPI/TypeScript client、Protobuf 代码、SBOM、镜像 digest 或 migration 历史。
- 内部 API、trait、crate 或模块重命名一次迁移全部调用方；不保留只有转发、改名或 re-export 价值的薄兼容层。
- 公开 Git protocol、Webhook/API、滚动部署和数据 migration 需要兼容期时，必须写明版本、边界、测试和删除条件。
- 不把 Gitea/GitHub 的功能清单直接当作设计；每个能力必须有权威状态、权限、并发、失败、恢复和审计语义。
- 不把 S3 FUSE 当作在线 Git repository 的 POSIX 一致性实现。
- 不把数据库中的 refs 镜像当作 Git objects/refs 的第二权威来源。
- 不把 Redis lock 或本地 mutex 当作 Distributed repository writer fencing 的唯一依据。
- 不让 API、worker 或 runner 绕过 repository service 直接修改在线 bare repository。
- 不在 AsterGit 主进程内执行不可信 workflow；runner 必须具有独立的身份、资源、网络和 workspace 隔离。
- fire-and-forget、webhook、outbox、备份和异步任务必须记录失败；禁止静默吞掉错误。
- token、密码、SSH 私钥、MFA secret、workflow secret、webhook secret、provider credential 和完整用户源码不得进入日志、错误、事件 payload 或普通审计 detail。

## 所有权速记

AsterForge 拥有产品无关的运行时和基础设施机制；AsterGit 保留软件开发 forge 的产品语义和 Git 数据一致性。

- **Forge**：runtime lifecycle、配置、数据库连接/迁移机械层、通用任务 lease、audit plumbing、日志、指标、限流、健康检查和测试支持。
- **AsterGit Identity**：用户、组织、团队、SSH key、PAT/OAuth、MFA、session、suspension 和外部身份绑定。
- **AsterGit Authorization**：repository/action/ref 的 allow/deny、可见性、团队继承、branch policy 和 policy revision。
- **AsterGit Repository**：bare repository、Git transport、objects、refs、pack、quarantine、hooks、GC、mirror、placement、writer lease、epoch 和 generation。
- **AsterGit Collaboration**：Issue、Pull Request、Review、Checks、Merge、merge queue、labels、milestones、timeline 和通知语义。
- **AsterGit Worker**：可信后台任务、outbox、webhook、索引、统计、维护、镜像、备份和清理。
- **AsterGit Runner**：不可信用户 workflow 执行、job lease、日志、secret scope、artifact upload 和取消；不得成为主服务的插件。
- **AsterGit Artifact/Backup**：LFS、release asset、附件、package blob、workflow artifact、bundle、snapshot 和 restore manifest。

“多个项目可能使用”不是迁入 Forge 的充分理由。抽取前写清：

```text
旧模块 -> Forge API/component/schema/store -> AsterGit 保留的产品责任 -> 必测行为
```

## 后端实现速记

目标链路：

```text
HTTP/SSH transport
  -> authentication/authorization
  -> service/use case
  -> domain policy
  -> repository placement / storage / transaction
  -> outbox / projection / worker / runner
```

- Transport 只做 framing、认证入口、参数提取、限流、调用 service 和协议响应映射。
- Service 编排完整 use case，不堆 SQL、Git shell 字符串、权限矩阵或 UI 展示逻辑。
- Domain 承载 repository path normalization、ref policy、generation、merge、placement、lease、artifact lifecycle 等可测试规则。
- Repository/DB store 只做数据访问和原子 SQL；不直接产生 Git wire response。
- Repository storage adapter 负责 Git process、POSIX/remote storage contract、quarantine、ref transaction、GC 和 fsck；业务层不直接拼具体路径或 SDK。
- Outbox/worker 负责可靠异步副作用；runner 通过显式 job contract 访问，不共享 worker 的信任等级。
- API、Web、SSH 和 Smart HTTP 可以有不同 adapter，但必须调用同一授权和 use case contract。
- 跨 transport、service、domain、repository、storage、event、worker、runner 的改动，实施前列出每层职责和失败边界。

## 数据、并发和安全

- writer database 用于用户、组织、权限、协作实体、任务、审计、placement metadata 和 outbox 的事务权威；reader 只能用于允许滞后的纯读。
- Git objects、refs、HEAD、pack 和 receive-pack transaction 由 repository storage 权威；数据库不能与 refs 双写。
- 每个 repository 在同一 fencing epoch 内只有一个 ref writer；旧 epoch 无法提交 refs、返回 push 成功或发布成功事件。
- API 可以无状态水平扩展；repository read 可并发；repository write、同一 target branch merge 和维护任务按 placement/queue 串行化。
- generation、policy revision、observed commit/ref 和 lease epoch 必须进入跨服务事件、审计和 projection，避免把陈旧结果当成当前事实。
- Git quarantine、pre-receive/update/reference-transaction、原子 refs、non-fast-forward 和客户端取消必须保持可观察语义。
- cache 是可重建投影，不是权限、writer lock、配额、refs 或恢复的权威来源。
- S3-compatible storage 优先承载 artifact、backup 和 archive；在线 Git repository 只有在明确证明 lock、rename、quarantine、fsck 和 snapshot contract 后才允许使用远端实现。
- webhook、mirror、import、avatar、OIDC discovery、package upstream 和 signed URL 都按 SSRF 风险处理；必须限制 scheme、port、redirect、DNS、IP、大小和超时。
- 不可信 workflow 不能读取宿主 Docker socket、云 metadata、其他租户 workspace、AsterGit 管理网络或数据库管理员凭据。

## 测试和验证

新增或修改行为必须有测试。验证范围跟风险走，不能用编译通过或一个 focused test 代替完整验收。

### 设计和契约

- RFC 变更：检查术语、依赖方向、权威来源、失败语义、profile 差异和文档链接。
- API/schema/protocol：运行 schema drift、兼容性、生成产物和真实 Git client/HTTP client 验证。
- 公共 contract：覆盖 capability、版本、deadline、retry、幂等、取消、unknown outcome 和错误映射。

### Git 和协作

- clone/fetch/push、SSH、Smart HTTP、protocol v2、空仓库、non-fast-forward、atomic multi-ref、force-push policy 和客户端取消。
- quarantine、hook rejection、reference transaction、writer crash、旧 epoch fencing、repack/GC、fsck、mirror 和 restore。
- Issue、PR、Review、Checks、merge、merge queue、冲突、source force-push、policy revision 变化和 stale diff。

### 身份、数据和扩展

- 用户/组织/团队权限继承、显式 deny、token/key 撤销、suspension、跨租户边界和 path traversal。
- SQLite 是 Standalone 的最低运行验证；Distributed Profile 必须验证 PostgreSQL。未来增加其他数据库方言时，再补对应索引、事务、并发和 migration 矩阵。
- outbox replay、重复事件、worker lease、dead letter、runner crash/cancel、secret scope、artifact digest、quota 和 GC race。
- backup manifest、空目录/临时数据库恢复、replica lag、failover、promotion、upgrade rollback 和真实 RPO/RTO。
- SSRF/DNS rebinding、恶意 Git config/submodule/alternates、hook bypass、runner 网络隔离、secret leakage、SBOM 和 provenance。

结束前运行：

```bash
git diff --check
```

并在结果中准确列出实际运行和未运行的验证。来自旧 checkout、被中断或仍在运行的命令不算当前通过证据。

## 文档与 RFC 收尾

架构、协议、数据、部署或公开 API 行为变更时，同步 README、项目契约、架构索引、相关 RFC、配置示例、API/schema 文档和测试说明。新增 RFC 必须在 `developer-docs/zh-CN/architecture/index.md` 建立入口，互相引用使用实际存在的文件名。

实现性任务结束前检查 CHANGELOG 是否需要记录；纯 RFC/内部文档调整只有在项目已有 release-note 约定时才进入 `Unreleased`，不要制造无意义的版本噪声。

## Code Review Fixes

用户粘贴 Greptile、CodeRabbit、Gemini 或人工 review comment 时：

1. 对照当前 revision、RFC、路径、符号和行为逐条判断真实问题或误报。
2. 只修仍成立的真实问题，保持修改最小，并检查是否改变了高可用、权限或 Git 一致性不变量。
3. 按相关性分批修复，每批完成后运行对应编译、协议或集成测试。
4. 最终列出已修、误报、跳过原因和验证命令。

## 文档入口

- [项目契约](developer-docs/zh-CN/architecture/project-contract.md)
- [架构概览](developer-docs/zh-CN/architecture/index.md)
- [平台架构图](developer-docs/zh-CN/architecture/platform-architecture-diagrams.md)
- [AsterGit README](README.md)

完整架构选择以 RFC-0001 至 RFC-0009 为准；如果实现超出 RFC，先补设计和验收，不把未经讨论的功能悄悄变成长期边界。
