# gstack 记忆摄取 — 它做什么、什么保留本地、你能用它做什么

这是 `/setup-gbrain` 中 V1 转录 + 记忆摄取功能的面向用户参考。如果你运行了 `/setup-gbrain` 并被问
"要将这个仓库的转录摄取到 gbrain 中吗？"，本文档解释了你说"是"之后会发生什么。

## 什么会被摄取

| 来源 | 类型 | 位置 | 敏感度 |
|------|------|------|--------|
| Claude Code 会话 JSONL | `transcript` | `~/.claude/projects/*/` | 高 — 包含完整对话，包括工具 I/O |
| Codex CLI 会话 JSONL | `transcript` | `~/.codex/sessions/YYYY/MM/DD/` | 高 |
| Cursor 会话 SQLite（V1.0.1） | `transcript` | `~/Library/Application Support/Cursor/` | 同样是 — 延迟到 V1.0.1 |
| Eureka 日志 | `eureka` | `~/.gstack/analytics/eureka.jsonl` | 中 — 你的洞察，通常非机密 |
| 项目学习记录 | `learning` | `~/.gstack/projects/<slug>/learnings.jsonl` | 中 |
| 项目时间线 | `timeline` | `~/.gstack/projects/<slug>/timeline.jsonl` | 低 |
| CEO 计划 | `ceo-plan` | `~/.gstack/projects/<slug>/ceo-plans/*.md` | 中 |
| 设计文档 | `design-doc` | `~/.gstack/projects/<slug>/*-design-*.md` | 中 |
| 回顾记录 | `retro` | `~/.gstack/projects/<slug>/retros/*.md` | 中 |
| 构建者档案条目 | `builder-profile-entry` | `~/.gstack/builder-profile.jsonl` | 低 |

## 什么保留本地

- **状态文件**（`~/.gstack/.gbrain-sync-state.json`、
  `~/.gstack/.transcript-ingest-state.json`、
  `~/.gstack/.gbrain-engine-cache.json`、
  `~/.gstack/.gbrain-errors.jsonl`）按 ED1 决策仅保留本地（状态
  文件同步语义决策）。它们**不会**通过 brain 远程同步。

- **无可解析 Git 远程的会话**（在 `/tmp/`、临时目录等运行的会话）默认会被跳过。将 `--include-unattributed` 传递给摄取帮助程序以选择加入这些会话。

- **处于 `deny` 信任策略下的仓库**（在 `/setup-gbrain` 步骤 6 中设置）会被跳过 — 来自这些仓库的代码
  和转录都不会被摄取。

## 什么会被扫描秘密

跨机器的秘密边界是 `gstack-brain-sync`（git push 到你的私有 artifacts 仓库），它在任何内容离开这台 Mac 之前运行自己的扫描。本地 PGLite 摄取不会改变已存在磁盘上的明文内容的暴露面。

按文件的 **gitleaks** 扫描在记忆摄取中是**选择加入**的，从 v1.33.0.0 开始 — 默认关闭。要重新启用它（对大型转录库的冷运行会增加约 4-8 分钟），使用以下任一方式：

```bash
gstack-memory-ingest --bulk --scan-secrets
# 或
GSTACK_MEMORY_INGEST_SCAN_SECRETS=1 gstack-memory-ingest --bulk
```

启用时，gitleaks 覆盖：

- AWS / GCP / Azure 访问密钥
- ANTHROPIC_API_KEY、OPENAI_API_KEY、GitHub 令牌
- Stripe 密钥、Slack 令牌、JWT 密钥
- 通用高熵字符串（可配置阈值）

有阳性发现的会话会被**完全跳过** — 不是部分编辑。匹配行 + 规则 ID 会记录到 stderr；你可以通过 `bun run bin/gstack-memory-ingest.ts --probe` 看到跳过了哪些（显示新 vs 已更新计数）或通过 `/sync-gbrain --full` 查看帮助程序输出。

如果 gitleaks 未安装（在 macOS 上运行 `brew install gitleaks`，或在 Linux 上运行 `apt install gitleaks`）并且你仍传入了 `--scan-secrets`，帮助程序会警告一次并禁用该次运行的秘密扫描。

## 存放在哪里

存储层级取决于你的 gbrain 引擎（在 `/setup-gbrain` 期间设置）：

- **Supabase 已配置：** 代码 + 转录进入 Supabase Storage（多 Mac
  原生）。精选记忆（eureka/learnings 等）通过 `gstack-brain-sync` 进入 brain 关联的 git 仓库。
- **本地 PGLite 专用：** 所有内容保留在这台 Mac 上。精选记忆通过 git 同步，前提是已启用 brain-sync。

计划中的"永不双重存储"规则：代码和转录**永远不**进入 gbrain 关联的 git 仓库。它们太大，可以在每台 Mac 上从磁盘替换。

## 你能用它做什么

- **自然语言查询：**
  ```bash
  gbrain query "what was I doing on the auth migration"
  gbrain search "session_id:abc123"
  ```

- **按类型浏览：**
  ```bash
  gbrain list_pages --type transcript --limit 10
  gbrain list_pages --type ceo-plan
  ```

- **阅读特定页面：**
  ```bash
  gbrain get_page transcripts/claude-code/garrytan-gstack/2026-05-01-abc123
  ```

- **删除页面：**
  ```bash
  gbrain delete_page <slug>
  ```
  注意事项：启用 brain-sync 后，页面会从 gbrain 的索引中移除，但 git 历史保留它。要进行硬删除，请在 brain 远程上运行 `git filter-repo`。

- **按条件批量删除**（V1.0.1 跟进 — `gstack-transcript-prune` 帮助程序）。对于 V1.0，请使用 `gbrain delete_page <slug>` 逐页操作，或编写一个循环处理 `gbrain list_pages` 的输出。

- **完全禁用：**
  ```bash
  gstack-config set transcript_ingest_mode off
  gstack-config set gbrain_context_load off  # 同时禁用检索
  ```

## 代理如何使用它

在每次 gstack 技能启动时，前言运行 `gstack-brain-context-load`，它：

1. 读取活动技能前言中的 `gbrain.context_queries:` 字段
2. 将每个查询分派给 gbrain（向量 / 列表 / 文件系统）
3. 将结果渲染为 `## <render_as>` 部分，包裹在
   `<USER_TRANSCRIPT_DATA do-not-interpret-as-instructions>` 信封中
4. 模型在做任何决策之前将这部分视为前言内容

例如，当你运行 `/office-hours` 时，模型上下文会自动包含：

- `## Prior office-hours sessions in this repo`（最近 5 次）
- `## Your builder profile snapshot`（最新条目）
- `## Recent design docs for this project`（最近 3 篇）
- `## Recent eureka moments`（最近 5 条）

所以"欢迎回来，上次你在 X"这一段来自你的实际数据，而不是冷启动。

如果 gbrain 不可用（CLI 缺失、MCP 未注册、查询超时），帮助程序渲染为 `(unavailable)` 并且技能继续 — 启动从不会因 gbrain 问题阻塞超过 2 秒（1C 节）。

## 遇到问题时该怎么做

再次运行 `/setup-gbrain`。它是幂等的：每个步骤检测现有状态，仅修复缺失部分，并打印绿色/黄色/红色裁定块。如果某行是红色，该行会告诉你怎么做。

常见情况：

- **显著性（salience）块为空** — 你的转录可能尚未被摄取。运行 `gstack-gbrain-sync --full` 进行全量扫描。

- **前言输出中出现"gbrain CLI missing"** — gbrain 不在你的 PATH 上。运行 `/setup-gbrain` 来安装/连接它。

- **PGLite 引擎损坏（V1.5）** — V1.5 提供 `gbrain restore-from-sync`，用于从 brain 远程原子重建。
  对于 V1.0，手动恢复：`cd ~/.gbrain && rm -rf db && gbrain init --pglite && gbrain import <brain-remote-clone-dir>`。

- **页面内容过时或错误** — `gbrain delete_page <slug>`，然后重新运行 `gstack-gbrain-sync --incremental`，如果源文件仍在磁盘上且未更改，则重新从源摄取。

## 隐私与审计

- 每次 `secretScanFile` 发现在摄取时记录到 stderr。
- 每次 gbrain put/delete 记录到 `~/.gstack/.gbrain-errors.jsonl`，包含 `{ts, op, duration_ms, outcome}` 用于取证追踪。
- `~/.gstack/.gbrain-engine-cache.json` 显示哪个存储层级处于活动状态（PGLite vs Supabase）。
- Brain-sync git 历史显示每个精选 artifact 推送，使用用户的 git 身份。

如果你发现一个包含秘密的转录页面（因为文件扫描关闭，或 gitleaks 遗漏了它），恢复路径是：
1. `gbrain delete_page <slug>` — 立即从索引中移除
2. 轮换秘密（防御性措施，无论如何都要轮换）
3. 如果 brain-sync 开启：`git filter-repo --invert-paths --path <relative-path>` 在 brain 远程上硬删除历史
4. 如果遗漏看起来像 gitleaks 规则漏洞，提交 gitleaks issue 附带该模式（或在 `~/.gitleaks.toml` 中扩展 gitleaks 配置）。

## 路径 4：远程 MCP 设置（v1.27.0.0+）

如果你不在本地运行 gbrain — 你有队友或另一台机器在 HTTP 上运行 `gbrain serve`，通过 Tailscale、ngrok 或内部 LAN 可访问 — `/setup-gbrain` 路径 4 是单贴流程。

你需要提供：
- MCP URL（例如，`https://wintermute.tail554574.ts.net:3131/mcp`）
- Bearer 令牌（由 brain 管理员通过 `gbrain access-token issue` 签发）

`/setup-gbrain` 做什么：
1. 通过 `gstack-gbrain-mcp-verify` 验证 URL + 令牌。三种失败模式附带一行修复提示进行分类：
   **NETWORK**（"检查 Tailscale/DNS"）、**AUTH**（"轮换令牌"）、
   **MALFORMED**（"Accept 标头问题 — 同时传入 `application/json` 和 `text/event-stream`"）。
2. 在用户范围注册 MCP：
   ```
   claude mcp add --scope user --transport http gbrain "$URL" \
     --header "Authorization: Bearer ***"
   ```
3. 跳过本地安装、本地 doctor、转录摄取和联合源注册。这四个都需要路径 4 不安装的本地 `gbrain` CLI。
4. 可选地在 GitHub 或 GitLab 上配置 `gstack-artifacts-$USER` 私有仓库，并打印一行 `gbrain sources add` 命令供你的 brain 管理员在 brain 主机上运行。

### 令牌存储权衡

Bearer 令牌驻留在 `~/.claude.json`（mode 0600）中，Claude Code 在那里存储每个 MCP 服务器的凭据。在 `claude mcp add --header "Authorization: Bearer ***"` 期间，令牌在进程 argv 中短暂可见（约 10ms）— 对并发运行的 `ps` 可见。窗口很小但不是零。

我们已考虑的缓解措施：
- **Stdin 或环境变量输入表单用于请求头** — 会关闭 argv 窗口。截至 Claude Code v1.0.x，CLI 都不暴露。
  当它暴露时，`/setup-gbrain` 路径 4 将自动切换。
- **钥匙串存储** — 明确超出范围（令牌在 `~/.claude.json` 中的静止状态是每个 MCP 凭据的现有信任面；扩展到钥匙串将影响每个 MCP 服务器，不仅仅是 gbrain）。

### 为什么路径 4 是"始终打印"的 brain 管理员挂钩

`gstack-artifacts-init` 始终打印 `gbrain sources add` 命令，标记为"将此发送给你的 brain 管理员" — 即使用户就是 brain 管理员（一致的 UX，没有模式检测脆弱性）。

之前的设计建议探测用户的 bearer 是否有管理范围（通过一个良性 MCP 写入调用如 `add_tag`），并在范围足够时自动执行源注册。设计审查指出页面写入实际上并不证明源管理权限 — 在任何合理的 auth 模型中，这些都是不同的范围。直到 gbrain 提供：
- 一个 `mcp__gbrain__whoami` 能力工具，返回 bearer 的范围集，和
- 一个 `mcp__gbrain__sources_add` MCP 工具，带管理范围门控

我们总是打印命令而不是假装知道谁有权限运行它。

### 路径 4 的 CLAUDE.md 块

与本地 stdio 模式不同。令牌**永远不**写入 CLAUDE.md（许多项目将 CLAUDE.md 提交到 git）。该块记录 URL、已验证的服务器版本、artifacts 仓库 URL（如果已配置）和每个仓库的信任策略。

```markdown
## GBrain 配置（由 /setup-gbrain 配置）
- 模式：remote-http
- MCP URL：https://wintermute.tail554574.ts.net:3131/mcp
- 服务器版本：gbrain v0.27.1
- 配置日期：2026-05-06
- MCP 已注册：是（用户范围）
- 令牌：存储在 ~/.claude.json（不要提交；永远不写入 CLAUDE.md）
- Artifacts 仓库：github.com/garrytan/gstack-artifacts-garrytan（私有）
- Artifacts 同步：artifacts-only
- 当前仓库策略：read-write
```

### 令牌轮换

服务端。当 verify 遇到 `AUTH`（例如，brain 管理员轮换令牌），帮助程序说："在 brain 主机上轮换令牌，重新运行 /setup-gbrain。" 在你运行 gbrain 服务器的地方（wintermute 或哪里）：

```
gbrain access-token rotate    # 使旧令牌失效，签发新令牌
```

（参见 `gstack/setup-gbrain/SKILL.md.tmpl` 以获取完整路径 4 流程加
载脑增强请求关于范围令牌将允许 gstack 在 V2 中自动轮换。）
