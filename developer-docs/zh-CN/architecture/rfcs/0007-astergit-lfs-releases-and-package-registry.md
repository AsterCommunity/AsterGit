# RFC-0007：AsterGit LFS, Releases and Package Registry

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-26 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md)、[RFC-0004](0004-astergit-repository-storage-and-git-transport.md)、[RFC-0005](0005-astergit-workers-runners-events-and-artifacts.md) |
| 影响范围 | Git LFS、release asset、issue/workflow attachment、package registry、对象存储、配额和清理 |

## 1. 存储边界

AsterGit 分别管理三种对象：

```text
Git repository storage
  -> Git objects, refs, packs, commit-graph

Artifact storage
  -> LFS objects, release assets, attachments, workflow artifacts, package blobs

Backup/archive storage
  -> bundles, database snapshots, repository replicas, restore manifests
```

它们共享身份和授权 contract，但不共享一致性假设。对象存储可以是本地 FS、S3-compatible 或未来专用 adapter；在线 Git repository 仍必须满足 RFC-0004 的 POSIX/ref transaction 语义。

## 2. Content-addressed object lifecycle

对象上传使用声明 digest/size/type 的 session：

```text
create upload session
  -> authorize namespace/repository/artifact
  -> upload parts or signed URLs
  -> verify size and digest
  -> atomically mark object complete
  -> create product reference
  -> asynchronous retention/GC
```

未完成、digest 不匹配或过期 session 的对象不可被下载。产品引用先于清理任务建立；GC 必须从权威引用、retention policy 和 pending upload 重新计算，不能依据对象存储列举结果直接删除。

对象 dedupe 默认以 content digest 为候选，但跨租户共享必须经过授权和隐私评估。对象是否存在不能成为未授权用户枚举其他组织内容的 oracle。

## 3. Git LFS

提供 Git LFS Batch API、download/upload action 和 signed URL adapter。LFS pointer 在 Git repository 内，LFS blob 在 artifact storage；一次 push 成功不代表客户端已经上传全部 LFS blob，服务端必须返回明确的 missing object 状态并可重试。

LFS 权限至少同时检查 repository read/write、object ownership、token scope、repository visibility 和 retention。删除 branch 或 PR 不自动删除仍被其他 refs、release 或 package 引用的 LFS object。

## 4. Releases 和附件

Release 绑定 immutable tag/object id、发布者、release state、notes revision 和 assets。草稿 release 可以变更资产；发布后 tag/object identity 和已发布资产 digest 不得静默替换，重新上传需要新的 asset version 或显式管理员操作并审计。

Issue/PR/comment attachment 绑定主体、上传者、visibility、digest、size 和 retention。下载先做主体权限检查，再创建短期下载 URL；对象存储错误不可泄露 bucket key 或 provider credential。

## 5. Package Registry

Package registry 与 repository 共享 namespace、身份和审计，但每种生态拥有独立协议 adapter、版本规则、索引、下载授权和清理策略：

```text
registry contract
├── package kind / protocol
├── name / version / metadata
├── immutable blob digests
├── yanked/deprecated state
├── owner / visibility / policy
└── retention / download audit
```

不把 package 版本塞进 Git tag，也不把 registry 索引直接当作对象存储目录。覆盖发布、删除、yank 和下载需要按生态协议定义；公共 registry 的匿名读取不能扩大私有 package 的可见性。

## 6. 配额和流控

配额按 instance、organization、repository、user 和 artifact kind 分层，至少统计 logical bytes、physical bytes、请求数、并发上传、下载带宽和保留对象。配额更新使用权威数据库条件写入，缓存只是显示投影。

大对象使用 multipart/resumable upload、单租户并发上限和 provider rate limit。对象下载可以 CDN 化，但授权 token、撤销和 audit 必须在边缘缓存 TTL 内有可解释语义。

## 7. 灾备与清理

对象 storage versioning、跨区域复制和生命周期只能作为机制，不能代替 restore manifest。每次 backup/restore 记录对象 digest、source generation、数据库 revision、加密 key id 和校验结果。GC、retention 和 legal/administrative hold 冲突时默认保留并记录原因。

## 8. 验收

覆盖 LFS Batch、断点续传、digest mismatch、missing LFS object、私有下载、release immutable asset、package protocol adapter、匿名枚举防护、配额并发、GC race、对象存储短暂故障、versioned restore 和跨租户隔离。
