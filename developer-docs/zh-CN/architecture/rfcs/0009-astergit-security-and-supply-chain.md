# RFC-0009：AsterGit Security and Supply Chain

| 字段 | 内容 |
| --- | --- |
| 状态 | Proposed |
| 日期 | 2026-08-26 |
| 上位 RFC | [RFC-0001](0001-astergit-platform-architecture.md)、[RFC-0003](0003-astergit-identity-organizations-and-authorization.md)、[RFC-0004](0004-astergit-repository-storage-and-git-transport.md)、[RFC-0005](0005-astergit-workers-runners-events-and-artifacts.md) |
| 影响范围 | 威胁模型、Git transport、权限、hooks、runner、SSRF、secret、签名、构建和供应链 |

## 1. 信任区

```text
Public edge
  -> authenticated control/API
  -> Git transport
  -> trusted AsterGit services
  -> repository content (untrusted bytes)
  -> isolated workflow runner (untrusted code)
  -> artifact/object storage
```

仓库源码、提交信息、文件名、Git config、LFS pointer、Issue 内容、webhook payload 和 workflow 都按不可信输入处理。一个用户能 push 代码，不等于他能执行 server hook、读取其他租户、访问 metadata service 或修改 AsterGit 数据库。

## 2. Transport 安全

- SSH 只允许公钥认证、forced Git command、无 PTY/forwarding/interactive shell；命令和路径使用结构化解析。
- HTTP 使用 TLS、认证/授权、request/body/pack 限制、CSRF（浏览器 session）、rate limit 和 abuse detection。
- token、SSH key、webhook secret 和 runner registration token 支持 scope、过期、轮换和撤销；只保存不可逆摘要或加密密文。
- 登录和权限错误避免用户、repository、key 和 token enumeration；审计保留内部原因，外部错误收敛。

## 3. Repository 和 hook 隔离

系统 Git hooks 由 AsterGit 管理，用户不能通过 push 写入服务端执行路径。pre-receive/update/reference-transaction policy 在受限环境运行，输入只来自 Git 官方 hook contract；hook 失败必须保持 refs 未提交。

读取 repository config、attributes、submodule URL、alternates、upload-pack 参数和 path 时禁止把内容解释为 AsterGit 命令。所有子进程使用清理后的环境、绝对 executable path、资源限制和可审计的 repository id。

## 4. Runner 和 SSRF

Runner 是独立安全边界：默认无云 metadata、无宿主 Docker socket、无 AsterGit 管理网络、无其他租户 workspace。网络出口按 allowlist/policy 控制，DNS rebinding、redirect、代理和 IPv6 literal 都必须按最终连接地址校验。

服务器侧 webhook、mirror、import、avatar、OIDC discovery、package upstream 和 LFS signed URL 都是 SSRF 风险入口。请求必须有 scheme/port/redirect/size/timeout 限制，解析和连接两次校验，禁止访问 loopback、link-local、metadata、Unix socket 和内部 control plane，除非显式配置并审计。

## 5. Secret 与数据保护

workflow secret 只在满足 repository/ref/environment policy 的 job 中短期注入，日志默认遮蔽并在取消/失败后清理 workspace。AsterGit 不把用户源码、token、密码、私钥、secret 或 provider credential 写入普通日志、事件 payload 或错误文本。

数据库、备份、artifact 和跨服务连接使用加密配置；key id、rotation epoch 和恢复依赖写入 manifest。备份加密不能替代访问控制，恢复流程必须验证最小权限。

## 6. 代码和提交信任

AsterGit 可以支持 signed commit/tag、protected branch、required review、required check 和 provenance，但签名验证结果必须绑定具体 object id、key identity、trust policy revision 和时间。签名通过不等于代码安全；未签名也不能被系统静默标为已验证。

## 7. 供应链

正式构建要求：

- lockfile 和依赖来源固定；
- Rust/Node/系统构建工具版本可审计；
- SBOM、许可证和漏洞扫描；
- 镜像、binary、migration 和前端产物有 digest；
- 构建 provenance 记录 source revision、toolchain、依赖和 builder identity；
- release artifact 可重建或有签名证明；
- runner image、action/plugin 和 hook policy 具备独立更新与撤销路径。

第三方 GitHub Actions workflow 可以作为兼容目标，但 action 下载、第三方镜像和脚本都属于不可信供应链，不能因为 YAML 来源于仓库就提升到 AsterGit server trust。

## 8. 事件响应

安全事件至少记录 actor、tenant/repository、request id、source address（按隐私策略）、object/ref/generation、policy revision、runner/job 和结果分类。支持 token/key 撤销、runner quarantine、repository read-only、artifact quarantine、hook disable、备份隔离和管理员告警。审计日志不可被普通 repository admin 删除。

## 9. 验收

覆盖 SSH command injection、path traversal、恶意 Git config、hook bypass、submodule/alternates、HTTP auth/CSRF、SSRF redirect/DNS rebinding、runner breakout、secret leakage、tenant isolation、签名误判、SBOM/provenance drift、密钥轮换和事件响应演练。
