# RFC-0003：AsterGit Identity, Organizations and Authorization

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-25 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md) |
| 影响范围 | 用户、组织、团队、SSH/HTTP 认证、权限、审计和多租户边界 |

## 1. 身份模型

```text
User
├── login identity
├── password / external identity bindings
├── SSH public keys
├── personal access tokens
├── sessions / MFA factors
└── memberships

Organization
├── owners
├── teams
├── repositories
├── webhooks / secrets
└── policies
```

Repository 可以属于个人 namespace 或 Organization。团队是权限主体，不用把每个成员复制成 repository ACL 行作为唯一模型。

## 2. 认证入口

- Web/API：session、OIDC/OAuth、MFA 和 token；
- SSH：public key -> key id -> user -> authorization；只允许 Git 原生命令，拒绝任意 shell；
- Smart HTTP：PAT/OAuth bearer，公开 repository 可以匿名读；
- internal worker/runner：短期 service identity、registration token、job lease；
- webhook：按 endpoint secret/signature 验证，不能把 URL 本身当作信任。

密码、token、私钥和 workflow secret 只在最短验证链路存在，不进入日志、事件、Git environment、审计 detail 或错误文本。

## 3. 授权决策

授权输入必须显式包含：actor、organization/repository、resource、action、ref/branch（适用时）、visibility、team membership、policy revision 和 request context。

基本动作：

```text
repository: discover, read, write, admin, delete, transfer, mirror
code: browse, clone, create_branch, push_ref, force_push, merge
collaboration: issue_read/write, pull_read/write, review, merge
admin: manage_members, manage_keys, manage_hooks, manage_runners, audit_read
```

拒绝默认优先。public visibility 只扩大匿名读取，不自动授予写入、Issue 创建、代码搜索或 webhook 管理。

## 4. 权限继承

```text
instance policy
  -> organization policy
  -> team membership/role
  -> repository role
  -> branch/ref policy
  -> action-specific decision
```

显式 deny、suspension、repository archived 和 branch protection 必须在最终决策中处理。缓存只保存带 policy revision 的可重建投影；权限变更后，writer 和 API 在权威 revision 复检前不能依赖旧缓存放行。

## 5. 多租户边界

所有 repository、issue、PR、artifact、webhook、runner 和 audit 查询必须带 namespace/repository scope。禁止通过客户端传入的 display path 直接拼接文件路径或 SQL；display path 先解析为稳定 ID，再由 placement 解析实际存储。

## 6. 审计和撤销

以下行为必须审计：登录失败/成功、key/token 创建撤销、成员与团队变化、repository transfer/delete、force push、protected branch 修改、hook/secret 修改、runner 注册、workflow 取消、权限拒绝和管理员 impersonation（若支持）。

撤销 token/key 后，新 transport 请求必须立即或在明确短 TTL 内失效；已建立的 Git process 必须有 session/lease policy，不允许永久绕过撤销。

## 7. 高并发授权

授权服务必须无状态可扩展；policy snapshot 带 revision，避免每个 Git packet 都查询数据库。一次 Git operation 在开始时建立 authorization context，涉及 ref policy 的提交前必须进行权威复检，防止长时间 upload 后权限已撤销仍能写入。

## 8. 验收

覆盖匿名/认证读写、组织/团队继承、显式 deny、suspension、token/key 撤销、path traversal、跨租户 ID 混淆、权限缓存失效、force push 门禁和并发成员变更。
