# AsterGit

AsterGit 是面向个人、小团队和自托管组织的软件开发 forge。它以 Git 仓库和代码协作为核心，并把单机、单节点高可用、分布式和大规模部署视为同一产品的不同 infrastructure profile。

## 产品契约

AsterGit 的长期目标不是做一个只会接受 `git push` 的 SSH 服务，也不是把 Gitea 的页面照搬成 Rust。它提供完整的代码托管与协作能力：

- Git over SSH 和 Smart HTTP；
- 用户、组织、团队、仓库可见性和细粒度权限；
- 代码浏览、commit、branch、tag 和 release；
- Issue、Pull Request、review、merge 和保护策略；
- webhook、通知、审计、备份和恢复；
- 可选的 LFS、制品、包注册表和 workflow runner。

单机部署是正式的一等形态：单个 AsterGit 进程、SQLite、本地仓库存储和内置 worker 即可运行完整核心能力。随着吞吐、隔离性和可靠性要求提高，可以把 API、worker、runner、仓库存储、制品存储和事件传输拆成独立组件；组件拆分不能改变产品契约、权限语义、Git 协议和一致性要求。

## 架构入口

- [项目契约](developer-docs/zh-CN/architecture/project-contract.md)
- [架构索引](developer-docs/zh-CN/architecture/index.md)
- [平台架构图](developer-docs/zh-CN/architecture/platform-architecture-diagrams.md)
- [RFC-0001：平台架构](developer-docs/zh-CN/architecture/rfcs/0001-astergit-platform-architecture.md)
- [RFC-0002：技术栈与部署 Profile](developer-docs/zh-CN/architecture/rfcs/0002-astergit-technology-stack-and-deployment-profiles.md)
- [RFC-0003：身份、组织与授权](developer-docs/zh-CN/architecture/rfcs/0003-identity-organizations-and-authorization.md)
- [RFC-0004：仓库存储与 Git Transport](developer-docs/zh-CN/architecture/rfcs/0004-repository-storage-and-git-transport.md)
- [RFC-0005：Worker、Runner、事件与制品](developer-docs/zh-CN/architecture/rfcs/0005-workers-runners-events-and-artifacts.md)

后续 RFC 会覆盖代码协作模型、LFS/Package Registry、可用性与灾备、安全与供应链。当前仓库只保存设计，不代表这些 RFC 所描述的能力已经实现。

## 设计原则

1. Git objects、refs 和 pack 的在线权威来源是 Git repository storage，不是数据库镜像。
2. 一个 repository 在同一 epoch 内只有一个 ref writer；故障切换必须通过 lease 和 fencing 防止 split-brain。
3. API 可以水平扩展，仓库写入和需要顺序的 Git 操作按 repository placement 串行化。
4. Worker 是可信后台任务；Runner 执行用户代码，必须处于独立的不可信执行边界。
5. S3-compatible storage 优先承载制品、备份和归档，不直接假装成 Git 的 POSIX 在线目录。
6. 每个远程副作用都要有 deadline、重试、幂等、取消、审计和恢复语义。

## 状态

当前阶段是 RFC 设计。没有实现、兼容性承诺或生产部署声明。

## License

计划采用 MIT OR Apache-2.0 双许可证，与 AsterForge、AsterDrive 的公共 Rust 项目保持一致；正式提交许可证文件前仍需完成仓库级确认。
