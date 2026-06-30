# gbrain-sync 错误速查表

`gstack-brain-*` 可以打印的每条错误消息，包含问题、原因和修复方法。

按 `BRAIN_SYNC:` 后的前缀或在命令输出中按二进制名称搜索本文件。

---

## `BRAIN_SYNC: brain repo detected: <url>`

**问题。** 你正在一台拥有 `~/.gstack-brain-remote.txt`（从另一台机器复制而来）但在 `~/.gstack/.git` 没有本地 git 仓库的机器上。

**原因。** 你在别处设置了 GBrain 同步，而 gstack 尚未在这台机器上恢复。

**修复。**
```bash
gstack-brain-restore
```
这会将仓库拉取到 `~/.gstack/` 并重新注册合并驱动。

如果你不想在此处恢复，用以下命令消除提示：
```bash
gstack-config set artifacts_sync_mode_prompted true
```

---

## `BRAIN_SYNC: blocked: <pattern-family>:<snippet>`

**问题。** 同步停止，因为密钥扫描器在暂存文件中检测到类似凭证的内容。队列已保存；未推送任何内容。

**原因。** 其中一个提交前密钥模式匹配了文件内容 — 可能是嵌入在 JSON 中的 AWS 密钥、GitHub token、OpenAI 密钥、PEM 块、JWT 或 bearer token。

**修复（三种选择）。**

1. **如果是真正的密钥**：编辑违规文件移除密钥，然后重新运行任意技能以重试同步。

2. **如果模式是误报**（例如，你的学习内容中包含你想发布的示例字符串里的 GitHub token 模式）：
   ```bash
   gstack-brain-sync --skip-file <path>
   ```
   这会永久将该路径从未来同步中排除。

3. **如果你想放弃这个同步批次**（重新开始）：
   ```bash
   gstack-brain-sync --drop-queue --yes
   ```
   这会清空队列但不提交。未来写入会正常重新填充。

---

## `BRAIN_SYNC: push failed: auth.`

**问题。** Git push 被拒绝，因为你对远程的身份验证已过期或缺失。

**原因。** 使用当前凭证无法访问远程。

**修复。** 根据你的远程类型刷新身份验证：

- **GitHub**: `gh auth status`（如需则 `gh auth refresh`）
- **GitLab**: `glab auth status`
- **其他**: `git remote -v` + 检查 SSH 密钥或凭证助手

修复身份验证后，运行任意技能会自动重试同步。

---

## `BRAIN_SYNC: push failed: <first-line-of-error>`

**问题。** 因非认证原因推送失败。git 错误的第一行出现在冒号之后。

**原因。** 可能是网络问题、推送被拒绝（远程领先）、服务器 500 或仓库访问被撤销。

**修复。** 查看 `~/.gstack/.brain-sync-status.json` 获取详情，或者运行：
```bash
cd ~/.gstack && git status && git push origin HEAD
```
查看 git 的完整错误。队列在任何推送尝试后都会清空，但本地提交仍然存在 — 下次技能运行时会重试推送。

---

## `gstack-brain-init: ~/.gstack/.git is already a git repo pointing at <url>`

**问题。** 你用一个不匹配现有远程的 URL 试图初始化。

**原因。** 你已经用不同的远程运行过 `gstack-brain-init`。

**修复。** 任选：

- 使用现有远程：不带 `--remote` 运行 `gstack-brain-init`，或传入匹配的 URL。
- 切换远程：先 `gstack-brain-uninstall`，然后用新 URL 重新初始化。这不删除你的数据。

---

## `Remote not reachable: <url>`

**问题。** Init 无法连接 git 远程以验证连通性。

**原因。** URL 错误、未认证、网络问题。

**修复。** 手动测试：
```bash
git ls-remote <url>
```
如果失败，检查：
- URL 拼写
- GitHub: `gh auth status`
- GitLab: `glab auth status`
- 私有网络 / VPN / DNS

---

## `gstack-brain-init: failed to create or find '<name>'`

**问题。** 通过 `gh repo create` 自动创建仓库失败，且 `gh repo view` 也找不到这个仓库。

**原因。** `gh` 未认证、那个名称的仓库已属于他人、或你的 GitHub 账户触及配额限制。

**修复。**
```bash
gh auth status
```
如果未认证，运行 `gh auth login`。如果仓库名称冲突，传入不同名称：
```bash
gstack-brain-init --remote git@github.com:YOURUSER/custom-name.git
```

---

## `gstack-brain-restore: ~/.gstack/.git already points at <url>`

**问题。** 你用一个不匹配现有 git 配置的 URL 试图恢复。

**原因。** 之前用不同远程进行初始化留下的过期 `.git`。

**修复。** `gstack-brain-uninstall`，然后重新运行 `gstack-brain-restore <url>`。

---

## `gstack-brain-restore: ~/.gstack/ has existing allowlisted files that would be clobbered`

**问题。** 你正在尝试恢复，但 `~/.gstack/` 已包含会被覆盖的学习内容或计划。

**原因。** 要么（a）这台机器已从同步前的 gstack 会话积累了状态，要么（b）之前失败的恢复留下了部分状态。

**修复（三种选择）。**

1. **如果这台机器的状态应该成为新的真相来源**：运行 `gstack-brain-init` 而非 restore — 这会从这台机器的状态创建一个全新的 brain 仓库。

2. **如果你想采用远程并丢弃这台机器的状态**：先备份 `~/.gstack/projects/`，然后移除违规文件，再重新运行 restore。

3. **如果你想合并**：此场景没有自动合并。将学习内容从 `~/.gstack/` 手动复制到已开启同步的机器上的 gstack 运行中，然后在此处恢复。

---

## `gstack-brain-restore: <url> does not look like a gstack-brain repo`

**问题。** 克隆成功，但仓库缺少 `.brain-allowlist` 和 `.gitattributes`。

**原因。** 你将 restore 指向了随机 git 仓库，或有人从 brain 仓库中删除了标准配置文件。

**修复。** 验证 URL。如果正确，运行 `gstack-brain-init --remote <url>` 以重新植入标准配置。

---

## Nothing is syncing but I expect it to

**不是错误，但常见陷阱。** 依次检查：

1. `gstack-brain-sync --status` — 模式是 `off` 吗？
2. `~/.gstack/.git` 存在吗？
3. `gstack-config get artifacts_sync_mode` — 应该是 `full` 或 `artifacts-only`。
4. 你期望同步的文件 — 它在允许列表中吗？
   `cat ~/.gstack/.brain-allowlist`
5. 隐私类过滤 — 如果模式是 `artifacts-only`，行为文件（时间线、开发者档案）会被有意跳过。

如果以上都正确，运行：
```bash
gstack-brain-sync --discover-new
gstack-brain-sync --once
```
强制清空。
