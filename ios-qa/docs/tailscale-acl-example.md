# Tailscale ACL 示例（适用于 iOS QA daemon）

当你传递 `--tailnet` 标志时，Mac 端 daemon 才绑定 Tailscale 接口。默认情况下 daemon 是本地 USB 专用。本文档引导你完成安全暴露 iPhone 以供远程代理驱动的步骤，以便通过 tailnet 进行 iOS QA。

## 威胁模型回顾

- **iOS 应用 StateServer：** 始终是仅本地回环。通过 CoreDevice IPv6 隧道可从 Mac 访问。从不直接绑定到 tailnet。
- **Mac daemon：** 拥有 tailnet 接口。绑定两个监听器——本地回环（完整表面，从不被转发）和 tailnet（带能力等级的锁定白名单）。
- **认证：** 通过本地 `tailscaled` socket（`/var/run/tailscale.sock` LocalAPI WhoIs）进行 Tailscale 身份验证。白名单文件 `~/.gstack/ios-qa-allowlist.json` 是谁能做什么的唯一可信来源。

## 步骤 1：安装并运行 Tailscale

```bash
brew install --cask tailscale
# 登录 + 启动 tailscaled，然后验证：
tailscale status
```

确认 daemon 可以读取 LocalAPI socket：

```bash
test -S /var/run/tailscale.sock && echo "socket present" || echo "MISSING"
```

如果缺失，daemon 将拒绝打开 tailnet listener（fail-closed）。

## 步骤 2：设置 daemon 的 ACL

daemon 需要知道哪些 Tailscale 身份被允许以何种能力级别控制哪些设备。白名单文件为 JSON 格式：

```json
{
  "version": 1,
  "entries": [
    {
      "identity": "you@example.com",
           "capabilities": ["restore"],
      "expires_at": null,
      "note": "Owner — full access"
    },
    {
      "identity": "ci@example.com",
      "capabilities": ["mutate"],
      "expires_at": "2026-12-31T00:00:00Z",
      "note": "CI runner — can write state but not full restore"
    },
    {
      "identity": "tag:claude-readonly",
      "capabilities": ["observe"],
      "expires_at": null,
      "note": "Agents that should only read"
    }
  ]
}
```

身份通过 WhoIs 规范化：

- **User OAuth：** `user@example.com`（无 `acct:`，无域名重写）。
- **Tagged 节点：** `tag:<tagname>`（小写）。
- **Node keys：** `node:<nodekey-hex>`（罕见；建议使用 tags）。

能力层级按升序排列：`observe` < `interact` < `mutate` < `restore`。
授予 `restore` 意味着所有低级别能力也被包含。

## 步骤 3：为远程代理 Mint 会话 token

你可以让代理自行 mint（如果他们拥有 allowlist 中的身份），也可以在服务端 mint：

```bash
# 服务端 mint（仅限 owner，在连接设备的本地 Mac 上运行）：
gstack-ios-qa-mint --remote ci@example.com --capability mutate --ttl 1h

# 自助 mint（通过 tailnet 的代理）：
curl -X POST http://<mac-tailnet-ip>:9999/auth/mint \
  -H "Content-Type: application/json" \
  -d '{"capability": "interact"}'
# → {"session_token": "...", "expires_at": "...", "capability": "interact"}
```

## 步骤 4：收紧 Tailscale ACL（纵深防御）

daemon 的 allowlist 是主要的访问控制。为了保险，可以限制 tailnet ACL 使其他人无法*到达* daemon 端口。

```jsonc
// 在 tailscale 管理控制台中：
{
  "acls": [
    // 仅允许 CI runner 访问 iOS QA Mac 的 9999 端口。
    {
      "action": "accept",
      "src": ["ci@example.com"],
      "dst": ["ios-qa-mac:9999"]
    },
    // Tagged Claude 代理人 — 仅 observe 层级（由 daemon 而非 ACL 执行）。
    {
      "action": "accept",
      "src": ["tag:claude-readonly"],
      "dst": ["ios-qa-mac:9999"]
    },
    // 默认拒绝。
    {
      "action": "drop",
      "src": ["*"],
      "dst": ["ios-qa-mac:9999"]
    }
  ]
}
```

## 步骤 5：审计跟踪

通过 tailnet listener 发出的每个经过认证的变异请求都会写入
`~/.gstack/security/ios-qa-audit.jsonl`：

```jsonl
{"ts":"2026-05-18T14:23:00Z","identity":"ci@example.com","device_udid":"00008101-XXXX","endpoint":"/tap","session_id":"abc...","capability":"interact","request_id":"req_001","status":200}
```

被拒绝的请求（无 token、token 过期、能力不足、身份不在 allowlist、速率限制命中）写入 `~/.gstack/security/attempts.jsonl`。

## 速率限制

- `/auth/mint`：每个身份 10 次 mint / 60s。第 11 次返回 429。
- 每个 tailnet 请求的请求体：硬性上限 1MB（超出返回 413）。
- 截图响应：硬性上限 10MB（超出返回 500，附带清理后的错误信息）。

## Token 生命周期

- Daemon 铸造的会话 token：默认 1h TTL，最大通过 `--tailnet-session-ttl` 配置为 24h。
- 可通过 `POST /session/heartbeat` 刷新（延长 `ttl_seconds`，但以原始最大值为上限）。
- Boot token（iOS 应用启动和 daemon 轮换之间的时间）：约 5s 生命周期——daemon 在首次抓取时立即轮换。

## 故障模式

| 症状 | 原因 | 处理措施 |
|---|---|---|
| Daemon 拒绝打开 tailnet listener | `/var/run/tailscale.sock` 缺失或无权限 | 安装 Tailscale；以运行 daemon 的用户身份验证 `tailscale status` 正常 |
| `403 identity_not_allowed` | 身份不在 allowlist 中 | Owner mint：`gstack-ios-qa-mint --remote <identity>` |
| `403 capability_insufficient` | token 级别低于端点要求 | 使用更高的 `--capability` 级别进行 owner mint |
| `429 rate_limited` | 同一身份 >10 次 mint/分钟 | 等待 60s；调查代理为什么重新 mint 这么频繁 |
| `409 schema_mismatch`（`/state/restore` 上） | 来自旧应用构建版本的快照 | 丢弃快照；从当前应用构建重新捕获 |
