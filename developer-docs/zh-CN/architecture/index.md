# AsterGit 架构概览

AsterGit 采用 RFC-first 设计。平台不会先做一套单机玩具，再因为高可用、权限和 runner 约束全部返工。

## RFC

- [RFC-0001：AsterGit Platform Architecture](rfcs/0001-astergit-platform-architecture.md)：产品边界、控制面、Git 数据面、协作面、扩展面和部署 profile。
- [RFC-0002：Technology Stack and Deployment Profiles](rfcs/0002-astergit-technology-stack-and-deployment-profiles.md)：Rust、AsterForge、HTTP/gRPC、数据库、事件传输、存储、扩缩容和高可用。
- [RFC-0003：Identity, Organizations and Authorization](rfcs/0003-astergit-identity-organizations-and-authorization.md)：用户、组织、团队、SSH key、token、可见性和权限决策。
- [RFC-0004：Repository Storage and Git Transport](rfcs/0004-astergit-repository-storage-and-git-transport.md)：SSH/HTTP Git、protocol v2、repository placement、single-writer、push transaction、GC 和镜像。
- [RFC-0005：Workers, Runners, Events and Artifacts](rfcs/0005-astergit-workers-runners-events-and-artifacts.md)：可信 worker、不可信 runner、outbox、webhook、制品和扩展组件边界。

## RFC（续）

- [RFC-0006：Code Collaboration](rfcs/0006-astergit-code-collaboration.md)：Issue、Pull Request、Review、Merge、branch policy、merge queue 和代码读模型。
- [RFC-0007：LFS, Releases and Package Registry](rfcs/0007-astergit-lfs-releases-and-package-registry.md)：LFS、release asset、附件、package blob、对象存储、配额和清理。
- [RFC-0008：Availability, Backup and Disaster Recovery](rfcs/0008-astergit-availability-backup-and-disaster-recovery.md)：副本、placement、fencing、备份、恢复、RPO/RTO 和升级回滚。
- [RFC-0009：Security and Supply Chain](rfcs/0009-astergit-security-and-supply-chain.md)：transport trust、hook、SSRF、runner、secret、签名、SBOM 和 provenance。

这九篇 RFC 共同构成当前的架构设计基线。它们是 Proposed 设计，不代表对应功能已经实现或已经作出生产兼容承诺。
