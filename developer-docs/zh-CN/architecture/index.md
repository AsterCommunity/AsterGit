# AsterGit 架构概览

AsterGit 采用 RFC-first 设计。平台不会先做一套单机玩具，再因为高可用、权限和 runner 约束全部返工。

## RFC

- [RFC-0001：AsterGit Platform Architecture](rfcs/0001-astergit-platform-architecture.md)：产品边界、控制面、Git 数据面、协作面、扩展面和部署 profile。
- [RFC-0002：Technology Stack and Deployment Profiles](rfcs/0002-technology-stack-and-deployment-profiles.md)：Rust、AsterForge、HTTP/gRPC、数据库、事件传输、存储、扩缩容和高可用。
- [RFC-0003：Identity, Organizations and Authorization](rfcs/0003-identity-organizations-and-authorization.md)：用户、组织、团队、SSH key、token、可见性和权限决策。
- [RFC-0004：Repository Storage and Git Transport](rfcs/0004-repository-storage-and-git-transport.md)：SSH/HTTP Git、protocol v2、repository placement、single-writer、push transaction、GC 和镜像。
- [RFC-0005：Workers, Runners, Events and Artifacts](rfcs/0005-workers-runners-events-and-artifacts.md)：可信 worker、不可信 runner、outbox、webhook、制品和扩展组件边界。

## Planned RFC

- `0006` Code Collaboration：Issue、Pull Request、Review、Merge、branch protection 和 merge queue。
- `0007` LFS, Releases and Package Registry：大文件、release asset、package blob、配额和清理。
- `0008` Availability, Backup and Disaster Recovery：副本、placement、快照、恢复、跨区域和升级回滚。
- `0009` Security and Supply Chain：SSH/HTTP trust、hook 隔离、SSRF、secret、签名、SBOM 和 provenance。

Planned RFC 不是当前实现承诺；它们记录未来必须与前五篇保持一致的问题域。
