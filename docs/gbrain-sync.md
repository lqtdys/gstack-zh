# 跨机器记忆与 GBrain 同步

gstack 将大量有用的状态写入 `~/.gstack/` — 学习成果、retro、CEO 计划、设计文档、开发者档案。默认情况下，切换笔记本时这些全部消失。**GBrain 同步**将精选子集推送到私有 git 仓库，让你的记忆跟着你跨机器走，并由 GBrain 索引。

## 你得到了什么

- 在机器 A 上工作，在机器 B 无缝接手。
- 你的学习、计划和设计在 GBrain 中可见（如果你在使用的话）。
- 一个干净的退出路径（`gstack-brain-uninstall`），永不触碰你的数据。
- 无守护进程、无系统服务、无后台进程。

## 什么不会离开你的机器

设计上，即使同步开启，以下内容仍保留在本地：

- 凭证：`.auth.json`、`auth-token.json`、`sidebar-sessions/`、
  `security/device-salt`、`config.yaml` 中的消费者 token
- 机器特定状态：Chromium 配置文件、ONNX 模型权重、
  缓存、eval-cache、CDP-profile、一次性提示标记
  （`.welcome-seen`、`.telemetry-prompted`、`.vendoring-warned-*` 等）
- 问题偏好：每台机器的 UX 偏好
  （`question-preferences.json`、`question-log.jsonl`、`question-events.jsonl`）。

确切的允许列表位于 `~/.gstack/.brain-allowlist`。CLI 管理它；你可以在标记行下方追加自己的条目。

## 首次运行设置（30–90 秒）

```bash
gstack-brain-init
```

该命令：

1. 将 `~/.gstack/` 变为 git 仓库。
2. 询问远程 URL（默认：`gh repo create --private
   gstack-brain-$USER`）。适用于任何 git 远程 — GitHub、GitLab、Gitea、
   自建。
3. 推送仅含配置的初始提交。
4. 写入 `~/.gstack-brain-remote.txt`（仅 URL，不含密钥 —
   安全复制到另一台机器）。
5. 将 gbrain 仓库连接到你本地的 gbrain 作为联邦源
   （通过 `gbrain sources add` + `git worktree`）使 `gbrain search`
   可以索引你同步的学习、计划和设计。实现位于
   `bin/gstack-gbrain-source-wireup`。旧的
   `gstack-brain-reader add --ingest-url ...` HTTP 路径已在
   v1.15.1.0 中移除 — 它依赖一个 gbrain 从未交付的 `/ingest-repo` 端点。

初始化后，**你运行的下一个技能**会问你一个关于隐私模式的问题：

- **全部允许列入（推荐）**：学习、审查、计划、设计、retro、时间线和开发者档案全部同步。
- **仅成果物（artifacts）**：计划、设计、retro、学习 — 跳过行为数据（时间线、开发者档案）。
- **拒绝**：保留所有本地内容。你可以随时用 `gstack-config set artifacts_sync_mode full` 开启同步。

你的回答会被持久化。不会再问。

## 跨机器工作流

在机器 A 上：运行一次 `gstack-brain-init`。就这样 — 现在每次技能调用都会在起始和结束边界排空同步队列（每次技能约 200–800 毫秒网络暂停）。

在机器 B 上：

1. 从机器 A 复制 `~/.gstack-brain-remote.txt` 到机器 B
   （密码管理器、dotfile 仓库、U盘 — 你选）。
2. 运行任意 gstack 技能。序言看到 URL 文件并打印：
   ```
   BRAIN_SYNC: brain repo detected: <url>
   BRAIN_SYNC: run 'gstack-brain-restore' to pull your cross-machine memory
   ```
3. 运行 `gstack-brain-restore`。这会克隆仓库，恢复你的
   学习/计划/retro，并重新注册 git 合并驱动。
4. 重新输入消费者 token（它们是机器本地的，不同步 —
   `gstack-config set gbrain_token <your-token>`）。
5. 下一个技能：你昨天在机器 A 上的学习浮现。这就是神奇时刻。

## 状态、健康状况和队列深度

```bash
gstack-brain-sync --status
```

显示：上次成功推送、待处理队列深度、任何同步块和当前隐私模式。

每次技能运行都会在序言输出顶部附近打印一行 `BRAIN_SYNC:`。扫描它查看问题。

## 隐私模式详情

| 模式 | 同步内容 |
|------|------------|
| `off` | 无（默认）。 |
| `artifacts-only` | 计划、设计、retro、学习、审查。跳过时间线 + developer-profile。 |
| `full` | 允许列表中所有内容包括行为状态。 |

随时更改：
```bash
gstack-config set artifacts_sync_mode full
gstack-config set artifacts_sync_mode off
```

## 密钥保护

在你的机器上内容离开之前，每次提交都会被扫描类似凭证的内容。被阻止的模式包括：

- AWS access key（`AKIA…`）
- GitHub token（`ghp_`、`gho_`、`ghu_`、`ghs_`、`ghr_`、`github_pat_`）
- OpenAI key（`sk-…`）
- PEM 块（`-----BEGIN …-----`）
- JWT（`eyJ…`）
- JSON 中的 Bearer token（`"authorization": "…"`、`"api_key": "…"` 等）

如果扫描命中，同步停止，队列保存，你的序言打印：

```
BRAIN_SYNC: blocked: <pattern-family>:<snippet>
```

修复方法：

1. 审查违规文件。
2. 如果匹配是你明确想同步的内容的误报，运行 `gstack-brain-sync --skip-file <path>` 永久排除该路径。
3. 否则，编辑文件移除密钥并重新运行任意技能。

在 `~/.gstack/.git/hooks/pre-commit` 有一条深度防御钩子，如果你手动 `git commit` 会运行同样的扫描。

## 双机器冲突

如果你同一天在机器 A 和机器 B 上写入，两者都会推送追加提交。Git 默认会在文件尾部冲突，但 `.jsonl` 和 markdown 文件注册了自定义合并驱动：

- JSONL 文件使用按 ISO 时间戳排序并去重的驱动（回退到每行的 SHA-256 哈希以保证确定性）。
- Markdown 成果物（retro、计划、设计）使用并集合并驱动，拼接两侧内容。

你不应该看到冲突提示。如果你看到了（真正的语义冲突，比如两台机器编辑同一个计划），git 会停止并提示。

## 跨机器拉取节奏

序言每 24 小时运行一次 `git fetch` + `git merge --ff-only`
（通过 `~/.gstack/.brain-last-pull` 缓存）。你不需要思考这件事 — 每天第一次技能调用时自动发生。

## 卸载

```bash
gstack-brain-uninstall
```

这会：

- 移除 `~/.gstack/.git/` 和所有 `.brain-*` 配置文件。
- 在 `gstack-config` 中清除 `artifacts_sync_mode`。
- 不触碰你的学习、计划、retro 或开发者档案。

加上 `--delete-remote` 也会删除私有 GitHub 仓库（仅 GitHub，使用 `gh repo delete`）。

随时用 `gstack-brain-init` 重新初始化。

## 故障排除

参见 [gbrain-sync-errors.zh-CN.md](gbrain-sync-errors.zh-CN.md) 获取 gbrain 可能打印的每条错误消息的索引，附每条的原因/修复。

## 底层实现

关于此功能背后的架构决策（允许列表 vs 拒绝列表、守护进程 vs 序言边界同步、JSONL 合并驱动、隐私停止门），参见 gstack 计划目录中的
[approved plan](../system-instruction-you-are-working-jaunty-kahn.md)。
