# 使用 GBrain 与 GStack

你的编码代理现在有了真正的记忆能力。

[GBrain](https://github.com/garrytan/gbrain) 是一个专为 AI 代理设计的持久化知识库。它存储你的代理学到的东西、你做出的决策、什么有效、什么无效，并让代理按需搜索所有内容。GStack 提供了一条从零到"gbrain 已运行，我的代理可以调用它"的一键路径——支持本地试用、团队共享及介于两者之间的各种场景。

这是最完整的版本：每个场景、每个标志、每个辅助二进制文件、每个故障排除步骤。快速介绍请参见 [README 的 GBrain 部分](README.md#gbrain--persistent-knowledge-for-your-coding-agent)。错误代码和同步相关问题请参见 [docs/gbrain-sync.md](docs/gbrain-sync.md)。

---

## 一键安装

```bash
/setup-gbrain
```

就这样。该技能检测你的当前状态，最多问三个问题，引导你完成安装、初始化、为 Claude Code 注册 MCP，以及按仓库的信任策略。在干净的 Mac 上，5 分钟内即可完成。在已有配置的 Mac 上只需几秒（它检测已有状态并跳过已完成的步骤）。

## 设置后获得什么

`/setup-gbrain` 完成后，你的编码代理拥有了两个以前不具备的检索功能：

- **跨本仓库的语义代码搜索。** `gbrain search "browser security canary"` 返回按相关性排列的文件区域，而非精确的 grep 匹配。`gbrain code-def`、`code-refs`、`code-callers`、`code-callees` 按符号遍历调用图——在你不知道哪个文件包含实现但知道它做什么时特别有用。当问题是语义性问题时，代理优先使用这些工具；CLAUDE.md 会包含一个 `## GBrain Search Guidance` 块来教它路由规则。
- **跨会话记忆。** 来自过去会话的计划、回顾、决策和学习存储在 `~/.gstack/` 中，如果你启用了工件推送同步，还会推送到一个 gbrain 索引的私有 git 仓库。`gbrain search "what did we decide about auth?"` 能真正找到之前的 CEO 计划，而不是让你每个会话都重复描述上下文。

如果你还启用了远程 MCP（下方 Path 4），大脑查询会路由到一个共享的大脑服务器，其他机器也可以向其写入——你的笔记本、台式机和队友的机器都能看到相同的记忆。

## 四条路径

当技能问"Where should your brain live?"（你的大脑应该在哪？）时，你选择其一。

### Path 1: Supabase，你已有连接字符串

最适合：你（或队友的云代理）已配置了一个 Supabase 大脑，想让你本地机器使用相同的数据。

**操作流程：** 粘贴 Session Pooler URL（Settings → Database → Connection Pooler → Session → 复制 URI，端口 6543）。技能以关闭回显的方式读取，向你显示脱敏预览（`aws-0-us-east-1.pooler.supabase.com:6543/postgres`——主机可见，密码隐藏），通过 `GBRAIN_DATABASE_URL` 环境变量将其传给 `gbrain init`，URL 永远不会写入 argv 或 shell 历史。

**信任警告：** 粘贴此 URL 会为你的本地 Claude Code 提供对共享大脑中每个页面的完整读写访问权限。如果这不是你想要的信任级别，请选择下方 PGLite 本地（Path 3），并接受两个大脑独立运行。

### Path 2a: Supabase，自动配置新项目

最适合：全新 Supabase 账户，想要一个干净的新项目，无需任何点击。

**操作流程：** 你粘贴一个 Supabase 个人访问令牌（PAT）。技能先显示权限披露——*该令牌授予对你 Supabase 账户中所有项目的完全访问权限，不仅仅是即将创建的那个项目*。它列出你的组织，询问选择哪个组织和哪个区域（默认 `us-east-1`），生成数据库密码，调用 `POST /v1/projects`，每 5 秒轮询一次 `GET /v1/projects/{ref}`，直到项目达到 `ACTIVE_HEALTHY` 状态（180 秒超时），获取 pooler URL，将其传给 `gbrain init`。端到端耗时约 90 秒。

结束时：明确的提醒，要求你在 https://supabase.com/dashboard/account/tokens 撤销 PAT。技能已从内存中将其丢弃。

**如果你在配置过程中 Ctrl-C：** SIGINT 信号处理程序打印进行中的项目 ref + 恢复命令。你可以在 Supabase 仪表板上删除孤立项，或运行 `/setup-gbrain --resume-provision <ref>` 从离开处继续。

### Path 2b: Supabase，手动创建

最适合：你更愿意自己点击 supabase.com 而不是粘贴 PAT。

**操作流程：** 技能引导你完成四个手动步骤（注册 → 新项目 → 等待约 2 分钟 → 复制 Session Pooler URL），然后从 Path 1 的粘贴步骤接管。安全性处理与 Path 1 相同。

### Path 3: PGLite 本地

最适合：先试一下，无需账户、不联网、不共享。或者一个专用的"本 Mac 大脑"，与任何云代理隔离。

**操作流程：** `gbrain init --pglite`。大脑位于 `~/.gbrain/brain.pglite`。初始化本身无需网络调用。30 秒内完成。

**嵌入模型。** 当设置了 `VOYAGE_API_KEY` 时，gstack 使用 `voyage-code-3`（1024 维）初始化 PGLite——Voyage 的代码专用嵌入模型，在本代码库符号查询上击败了通用模型 `voyage-4-large` 和 OpenAI 的 `text-embedding-3-large`。没有 `VOYAGE_API_KEY` 时，gbrain 会自动选择（如果存在 `OPENAI_API_KEY` 则使用 OpenAI 1536 维，否则沿其提供方链降级）。无论哪种方式，嵌入调用在同步期间都会访问所选提供方的 API——在运行 `/sync-gbrain` 之前设置你想用的提供方的 key。

这是只是想感受一下 gbrain 是什么体验时最佳的首选项。你可以随时通过 `/setup-gbrain --switch` 迁移。

### Path 4: 远程 gbrain MCP（分离引擎）

最适合：你的大脑运行在你控制的另一台机器上（Tailscale、ngrok、内部局域网），或者在队友的服务器上。你想要跨机器记忆的好处，但又不想在本机启动数据库，同时你仍然要在本 Mac 上进行符号感知的代码搜索。

**操作流程：** 你粘贴一个 MCP URL（例如 `https://wintermute.tail554574.ts.net:3131/mcp`）和一个 bearer 令牌。技能通过线路验证 URL，在用户范围将 gbrain 注册为 `~/.claude.json` 中的 HTTP MCP，并提供启动一个用于代码搜索的小型本地 PGLite（约 30 秒，约 120 MB 磁盘空间）。

如果你接受了本地 PGLite，你将进入**分离引擎模式**：

- **大脑/上下文查询**（`mcp__gbrain__search`、`mcp__gbrain__query`、`mcp__gbrain__get_page`）路由到远程 MCP。计划、回顾、学习、跨机器记忆——全部在共享服务器上。
- **代码查询**（`gbrain code-def`、`code-refs`、`code-callers`、`code-callees`、`gbrain search` 搜索代码）通过每个 worktree 中的 `.gbrain-source` 固定文件路由到本地 PGLite。本地索引，永不离开本机。

两个引擎独立。擦除本地 PGLite 不影响远程大脑；轮换远程 MCP bearer 不影响本地代码搜索。这也是你的远程大脑管理员不能（也不应该）索引每个开发者的检出时的正确配置——本地代码留在本地。

## 为 Claude Code 注册 MCP

默认情况下技能会问："为 Claude Code 提供 gbrain 的 typed 工具接口？"如果你同意，它运行：

```bash
claude mcp add gbrain -- gbrain serve
```

这为 Claude Code 注册了 gbrain 的 stdio MCP 服务器。现在 `gbrain search`、`gbrain put`、`gbrain get` 等作为一等工具出现在每个会话中，而不是 shell 调用。

**如果 `claude` 不在 PATH 上**，技能优雅地跳过 MCP 注册并给出手动注册提示。只要技能调用 `gbrain`，CLI 解析器仍然有效——MCP 是增强，不是先决条件。

**其他本地代理**（Cursor、Codex CLI 等）需要自己的 MCP 注册。该技能在 v1 中针对 Claude Code；其他主机可以在自己的 MCP 配置中手动注册 `gbrain serve`。

## 每个远程的信任策略（三元组）

你机器上的每个仓库都会得到一个策略决策：**读写**、**只读** 或 **拒绝**。

- **读写** —— 你的代理不仅可以从此仓库的上下文 `gbrain search`，还可以向大脑写入新页面。你自己项目的默认策略。
- **只读** —— 你的代理可以搜索大脑，但永远不从此仓库的会话中写入新页面。多客户顾问的理想选择：搜索共享大脑，但不在客户 A 的仓库中工作时用客户 A 的代码污染大脑。
- **拒绝** —— 完全不与 gbrain 交互。仓库对 gbrain 工具不可见。

技能在你首次在该仓库运行 gstack 技能时询问一次。此后该决策是粘性的——相同 git 远程的每个 worktree + 分支共享相同的策略，所以你设置一次，它跟着你。

SSH 和 HTTPS 远程变体合并为相同的键：`https://github.com/foo/bar.git` 和 `git@github.com:foo/bar.git` 是同一个仓库。

**要更改策略：**

```bash
/setup-gbrain --repo      # 仅重新提示此仓库

# 或者直接操作：
~/.claude/skills/gstack/bin/gstack-gbrain-repo-policy set "github.com/foo/bar" read-only
```

**查看所有策略：**

```bash
~/.claude/skills/gstack/bin/gstack-gbrain-repo-policy list
```

存储位置：`~/.gstack/gbrain-repo-policy.json`，权限模式 0600，带模式版本号以便未来迁移保持确定性。

## 保持大脑最新：`/sync-gbrain`

`/setup-gbrain` 是一次性的引导。`/sync-gbrain` 是每次你想让 gbrain 看到本仓库代码更新时要运行的命令。

```bash
/sync-gbrain                # 增量：mtime 快速路径，干净树上约数秒
/sync-gbrain --full         # 完全重新索引（大 Mac 上约 25-35 分钟）
/sync-gbrain --code-only    # 仅代码阶段；跳过 memory + brain-sync
/sync-gbrain --dry-run      # 预览将要同步的内容；不写入
```

该技能独立运行三个阶段——code、memory 和 brain-sync。一个阶段失败不会阻止其他阶段。状态持久化到 `~/.gstack/.gbrain-sync-state.json`，因此重新运行会干净地恢复。

**在全新 worktree 上的操作：**

1. **飞行前检查。** 检查 `gbrain_local_status`（本地引擎的健康状态）。如果引擎为 `broken-db` 或 `broken-config`，技能停止并提供修复菜单——拒绝静默降级。如果本地引擎缺失且你处于远程 MCP 模式（Path 4），代码阶段干净跳过，仅运行 brain-sync。
2. **代码阶段。** 通过 `gbrain sources add` 将 cwd 注册为联邦源，在仓库根写 `.gbrain-source` 固定文件（kubectl 风格的上下文——每个 worktree 获得自己的固定文件，Conductor 兄弟 worktree 不会冲突），运行 `gbrain sync --strategy code`。
3. **内存阶段。** 暂存你的 `~/.gstack/` 会话语料 + 精选记忆。在本地 stdio MCP 模式下，导入到本地引擎。在远程 HTTP MCP 模式下，将暂存的 markdown 持久化到 `~/.gstack/transcripts/run-<pid>-<ts>/` 供远程大脑管理员的拉取管道使用。导入超时默认为 30 分钟；对大型大脑用 `GSTACK_INGEST_TIMEOUT_MS` 提高（接受 1 分钟-24 小时）。超时时 gbrain 导入检查点会保留，以便下次 `/sync-gbrain` 恢复而不是重新开始。
4. **Brain-sync 阶段。** 将精选工件（计划、设计、回顾）推送到你在配置时设置的私有工件仓库。
5. **CLAUDE.md 指导。** 能力检查往返（写页面 → 搜索 → 找到）。如果正常，将 `## GBrain Search Guidance` 块写入项目的 CLAUDE.md。如果异常，删除该块——代理永远不应被告知使用未安装的工具。

**水印。** 同步状态通过 commit 哈希推进。如果 gbrain 遇到它无法索引的文件（每文件硬性 5 MB 限制，或文件在同步期间消失），水印保持不动，后续同步重试。要确认无法修复的失败并向前移动：

```bash
gbrain sync --source <source-id> --skip-failed
```

可重运行，幂等，可从同一机器的多个终端安全运行（锁定在 `~/.gstack/.sync-gbrain.lock`）。

## 稍后切换引擎

已选 PGLite 现在想加入团队大脑？一条命令：

```bash
/setup-gbrain --switch
```

该技能在 `timeout 180s` 内运行 `gbrain migrate --to supabase --url "$URL"`。迁移是双向的（Supabase → PGLite 也可行）且无损——页面、块、嵌入、链接、标签和时间线全部复制。你原始的大脑保留为备份。

**如果迁移挂起：** 另一个 gstack 会话可能正在持有源大脑的锁。超时会在 3 分钟时触发并提供可操作的信息。关闭其他工作区并重新运行。

## GStack 内存同步（一个单独的关注点）

这与 gbrain 本身不同。你的 gstack 状态（`~/.gstack/` — 学习、计划、回顾、时间线、开发者档案）默认是本地的。"GStack 内存同步"可选地将精选的、经过机密扫描的子集推送到私有 git 仓库，让你的记忆在跨机器时跟随你——如果你在运行 gbrain，那个 git 仓库也在那里可被索引。

用以下命令启用：

```bash
gstack-brain-init
```

你会得到一次性的隐私提示：**全部放行** / **仅工件**（计划、设计、回顾、学习——跳过时间线等行为数据） / **关闭**。每次技能运行都会在开始和结束时同步队列——无守护进程，无后台进程。

机密形状的内容（AWS key、GitHub 令牌、PEM 块、JWT、bearer 令牌）在离开你机器之前会被阻止。

**在新机器上：** 复制 `~/.gstack-brain-remote.txt`，运行 `gstack-brain-restore`，昨天的学习就能在今天的笔记本上呈现。

完整指南：[docs/gbrain-sync.md](docs/gbrain-sync.md)。错误索引：[docs/gbrain-sync-errors.md](docs/gbrain-sync-errors.md)。

`/setup-gbrain` 在初始设置结束时提供为你接通这个——只是一个额外的 AskUserQuestion，它与相同的私有仓库基础设施集成。

## 清理孤立项

如果你在配置过程中 Ctrl-C 了，在最终确定名称前试了三个不同的名称，或以其他方式积累了不使用的 gbrain 形状的 Supabase 项目，有一个子命令：

```bash
/setup-gbrain --cleanup-orphans
```

该技能重新收集一个 PAT（一次性，之后丢弃），列出你 Supabase 账户中名称以 `gbrain` 开头且 ref 不匹配你的活动 `~/.gbrain/config.json` pooler URL 的每个项目。对每个孤立项单独询问：*"删除孤立项 `<ref>`（`<name>`，创建于 `<date>`）？"* — 不批量处理，无"全部删除"快捷方式。活动大脑永远不会被提供删除。

## 命令 + 标志参考

### `/setup-gbrain` 入口模式

| 调用方式 | 功能描述 |
|---|---|
| `/setup-gbrain` | 完整流程：检测状态、选择路径、安装、初始化、MCP、策略、可选 memory 同步 |
| `/setup-gbrain --repo` | 仅翻转当前仓库的每个远程信任策略 |
| `/setup-gbrain --switch` | 引擎迁移（PGLite ↔ Supabase），不重新运行其他步骤 |
| `/setup-gbrain --resume-provision <ref>` | 恢复在轮询期间中断的 path-2a 自动配置 |
| `/setup-gbrain --cleanup-orphans` | 列出 + 按项目删除 Supabase 孤立项 |

### 辅助二进制文件（用于脚本）

| 二进制文件 | 用途 |
|---|---|
| `gstack-gbrain-detect` | 以 JSON 输出当前状态：gbrain 在 PATH、版本、配置引擎、doctor 状态、同步模式 |
| `gstack-gbrain-install` | 首次检测安装器（探测 `~/git/gbrain`、`~/gbrain`，然后全新 clone）。有 `--dry-run` 和 `--validate-only` 标志。PATH 遮蔽检查以退出码 3 并提供修复菜单。 |
| `gstack-gbrain-lib.sh` | 引用执行，不直接运行。提供 `read_secret_to_env VARNAME "prompt" [--echo-redacted "<sed-expr>"]` |
| `gstack-gbrain-supabase-verify` | 结构性 URL 检查。拒绝直接连接 URL（`db.*.supabase.co:5432`），退出码 3 |
| `gstack-gbrain-supabase-provision` | Management API 包装器。子命令：`list-orgs`、`create`、`wait`、`pooler-url`、`list-orphans`、`delete-project`。全部要求在 env 中有 `SUPABASE_ACCESS_TOKEN`。`create` 和 `pooler-url` 还要求 `DB_PASS`。每个子命令都有 `--json` 模式。 |
| `gstack-gbrain-repo-policy` | 每个远程的信任三元组。子命令：`get`、`set`、`list`、`normalize` |
| `gstack-gbrain-source-wireup` | 通过 `gbrain sources add` + `git worktree` 将你的 `~/.gstack/` 大脑仓库注册为 gbrain 的联邦源，然后运行初始 `gbrain sync`。幂等。取代了 v1.12.x 中已废弃的 `consumers.json + /ingest-repo` HTTP 连线。标志：`--strict`、`--source-id <id>`、`--no-pull`、`--uninstall`、`--probe`。 |

### gbrain CLI（上游工具）

Gbrain 本身附带了 gstack 所包装的命令：

| 命令 | 用途 |
|---|---|
| `gbrain init --pglite` | 初始化本地 PGLite 大脑 |
| `gbrain init --non-interactive` | 通过 env 初始化（`GBRAIN_DATABASE_URL` 或 `DATABASE_URL`）。决不要将 URL 作为 argv 传递——它会泄漏到 shell 历史。 |
| `gbrain doctor --json` | 健康检查。返回 `{status: "ok"|"warnings"|"error", health_score: 0-100, checks: [...]}` |
| `gbrain migrate --to supabase --url ...` | 将 PGLite 大脑移至 Supabase（无损，保留源作为备份） |
| `gbrain migrate --to pglite` | 反向迁移 |
| `gbrain search "query"` | 搜索大脑 |
| `gbrain put "<slug>" --content "<markdown-with-frontmatter>"` | 写页面（标题/标签位于 `--content` 内的 YAML frontmatter 中） |
| `gbrain get "<slug>"` | 获取页面 |
| `gbrain serve` | 启动 MCP stdio 服务器（由 `claude mcp add` 使用） |

### 配置文件 + 状态

| 路径 | 内容 |
|---|---|
| `~/.gbrain/config.json` | 引擎（pglite/postgres）、数据库 URL 或路径、API key。权限 0600。由 `gbrain init` 写入。 |
| `~/.gstack/gbrain-repo-policy.json` | 每个远程的信任三元组。Schema v2。权限 0600。 |
| `~/.gstack/.setup-gbrain.lock.d` | 并发运行锁（原子 mkdir）。正常退出 + SIGINT 时释放。 |
| `~/.gstack/.brain-queue.jsonl` | gstack 内存同步挂起的条目 |
| `~/.gstack/.brain-last-push` | 上次推送同步的时间戳（用于 `/health` 评分） |
| `~/.gstack-brain-remote.txt` | 你的 gstack 内存同步远程 URL（可在机器间安全复制） |
| `~/.gstack/.setup-gbrain-inflight.json` | 保留供未来 `--resume-provision` 持久化状态使用 |

### 环境变量

| 变量 | 读取位置 | 功能 |
|---|---|---|
| `SUPABASE_ACCESS_TOKEN` | `gstack-gbrain-supabase-provision` | Management API 调用的 PAT。每次设置运行后丢弃。 |
| `DB_PASS` | `gstack-gbrain-supabase-provision`（create、pooler-url） | 生成的数据库密码。决不在 argv 中。 |
| `GBRAIN_DATABASE_URL` | `gbrain init`、`gbrain doctor` 等 | Postgres 连接字符串（对我们来说是 Supabase pooler URL）。Env 优先于 `~/.gbrain/config.json`。 |
| `DATABASE_URL` | `gbrain init`（回退） | 与 `GBRAIN_DATABASE_URL` 语义相同；第二检查。 |
| `SUPABASE_API_BASE` | `gstack-gbrain-supabase-provision` | 覆盖 Management API 主机。测试用于指向 mock 服务器。 |
| `GBRAIN_INSTALL_DIR` | `gstack-gbrain-install` | 覆盖默认安装路径（`~/gbrain`） |
| `GSTACK_HOME` | 每个辅助二进制文件 | 覆盖 `~/.gstack` 状态目录。大量测试使用。 |
| `VOYAGE_API_KEY` | `gbrain embed` 子进程；gstack PGLite 初始化 | 设置后，gstack 使用 `voyage-code-3`（1024 维）初始化 PGLite，Voyage 的代码专用嵌入模型。在本代码库符号查询上击败了 `voyage-4-large` 和 OpenAI 的 `text-embedding-3-large`。参见 CHANGELOG v1.43.1.0 了解 A/B 数据。 |
| `OPENAI_API_KEY` | `gbrain embed` 子进程 | 当未设置 `VOYAGE_API_KEY` 时，在 `gbrain sync` / `/sync-gbrain` 期间用于嵌入（gbrain 自动选择的回退，`text-embedding-3-large` 1536 维）。没有这两个 key，页面以结构化方式导入（符号表、块）但语义搜索会降级——你会在同步日志中看到 `[gbrain] embedding failed for code file ...`。 |
| `ANTHROPIC_API_KEY` | `claude-agent-sdk`，付费评分 | `bun run test:evals` 和任何对 Claude 的直接 `query()` 调用所需。 |
| `GSTACK_OPENAI_API_KEY` | `lib/conductor-env-shim.ts` | 由 Conductor 注入的回退。当规范名称为空时提升为 `OPENAI_API_KEY`。 |
| `GSTACK_ANTHROPIC_API_KEY` | `lib/conductor-env-shim.ts` | 同上，对应 Anthropic。 |

## Conductor + GSTACK_* 环境变量

如果你在 [Conductor](https://conductor.build) 工作区中运行 gstack，**Conductor 显式地从工作区环境变量中剥离 `ANTHROPIC_API_KEY` 和 `OPENAI_API_KEY`**。在 `~/.zshrc` 或 `.env` 中设置它们没有帮助——剥离发生在环境继承之后。要在工作区中获得可用的 API key，需改为在 Conductor 的工作区 env 配置中设置 `GSTACK_ANTHROPIC_API_KEY` 和 `GSTACK_OPENAI_API_KEY`。Conductor 会原封不动地传递它们。

`lib/conductor-env-shim.ts` 在 gstack 端桥接了这个差距：作为副作用导入（`import "../lib/conductor-env-shim"`），当规范名称不存在时它将 `GSTACK_FOO_API_KEY` 提升为每个子进程的 `FOO_API_KEY`。该 shim 已接入：

- `bin/gstack-gbrain-sync.ts` —— 让 `/sync-gbrain` 获取用于嵌入的 OpenAI
- `bin/gstack-model-benchmark` —— 让 `--judge` 运行无需手动 env 映射
- `scripts/preflight-agent-sdk.ts` —— 让付费评分认证探测工作
- `test/helpers/e2e-helpers.ts` —— 让 `bun run test:evals` 找到 Anthropic

如果你添加了一个新的访问付费 API 或需要 gbrain 嵌入的 TypeScript 入口点，在顶部添加相同的单行导入。参见 [CONTRIBUTING.md "Conductor workspaces"](CONTRIBUTING.md#conductor-workspaces) 了解贡献者检查清单。

`bin/gstack-codex-probe` 是 bash 且不直接读取这些——它依赖由 Codex CLI 管理的 `~/.codex/` 认证。

## 安全模型

一条适用于此技能接触的每个机密的规则：**只有 env var，决不是 argv，决不是日志，决不是由我们写入磁盘。** 唯一的持久存储是 gbrain 自己权限 0600 的 `~/.gbrain/config.json`，这是 gbrain 的纪律，不是我们的。

**代码执行检查：**

- `test/skill-validation.test.ts` 中的 CI grep 测试在 `$SUPABASE_ACCESS_TOKEN` 或 `$GBRAIN_DATABASE_URL` 出现在 argv 位置时会让构建失败
- CI grep 测试在 `--insecure`、`-k` 或 `NODE_TLS_REJECT_UNAUTHORIZED=0` 出现在 `bin/gstack-gbrain-supabase-provision` 中时会让构建失败
- `set +x` 在配置辅助文件的顶部防止调试跟踪泄漏 PAT
- 遥测负载仅包含枚举的分类值（场景、安装结果、MCP 选择加入、信任级别）——永远不是可能包含机密的自由格式字符串

**通过测试执行：**

- `test/secret-sink-harness.test.ts` 将每个处理机密的二进制文件以预设种子运行，并断言种子永远不会出现在任何捕获通道中（stdout、stderr、`$HOME` 下的文件、遥测 JSONL）。每个种子有四个匹配规则：精确匹配、URL 解码、前 12 字符前缀、base64。
- 同一测试文件中的正控制故意在每个覆盖通道中泄漏种子并断言约束机制捕获每个。没有正控制，一个静默低报的约束机制看起来与一个正常工作的约束机制没有区别。

**你仍可能泄漏的内容**（v1 的诚实限制）：

- 如果你在正常聊天消息中（`read -s` 之外）粘贴了机密，它存在于会话 Transcript 和任何主机端的日志记录中
- 泄漏转储机制不会倾倒子进程环境——一个 `env >> ~/.log` 的二进制文件可以逃避检测（v1 中没有二进制文件这样做；grep 测试防止它们）
- 你 shell 自己的 `HISTFILE` 行为是你 shell 的，不是我们的——我们永远不将机密传给 argv 以便通过我们的代码落到那里，但你自己在原始 `curl` 命令中粘贴一个我们也无能为力

## 故障排除

### 安装过程中出现 "PATH SHADOWING DETECTED"

在 PATH 中有另一个 `gbrain` 二进制文件比安装程序刚链接的更靠前。安装程序的版本检查捕获了它。修复方法：

- 如果你不需要另一个，`rm $(which gbrain)`
- 在你的 shell rc 中将 `~/.bun/bin` 添加到 PATH 前面，使链接的二进制文件获胜
- 将 `GBRAIN_INSTALL_DIR` 设置为遮蔽二进制文件的安装目录并重新运行

然后重新运行 `/setup-gbrain`。

### "rejected direct-connection URL"

你粘贴了 `db.<ref>.supabase.co:5432` URL。这些是仅 IPv6 的，在大多数环境中会失败。改用 Session Pooler URL：Supabase 仪表板 → Settings → Database → Connection Pooler → **Session** → 复制 URI（端口 6543）。

### 自动配置在 180 秒超时

Supabase 项目仍在初始化。你的 ref 已在退出消息中打印。等待一分钟，然后：

```bash
/setup-gbrain --resume-provision <ref>
```

技能重新收集 PAT，跳过项目创建，恢复轮询。

### "另一个 `/setup-gbrain` 实例正在运行"

你有陈旧的锁目录。如果确定没有其他实例在实际运行：

```bash
rm -rf ~/.gstack/.setup-gbrain.lock.d
```

然后重新运行。

### 策略文件上出现 "No cross-model tension"

你用旧的 `allow` 值手动编辑了 `~/.gstack/gbrain-repo-policy.json`？没问题。下次读取时，gstack 将自动迁移 `allow` → `read-write` 并添加 `_schema_version: 2`。stderr 上有一行日志，幂等，确定性。

### `gbrain doctor` 说 "warnings"

`/health` 将其视为黄色，而不是红色。检查 `gbrain doctor --json | jq .checks` 查看哪些子检查在警告。典型原因：solver MECE 重叠（skill 名称冲突）或数据库连接尚未配置。

### `/sync-gbrain` 报告 `OK` 但 `gbrain search` 没有返回语义结果

嵌入导入期间可能失败了。符号查询（`code-def`、`code-refs`）仍然工作因为它们不需要嵌入，但 `gbrain search "<terms>"` 降级到 BM25 路径。在同步输出中查找类似：

```
[gbrain] embedding failed for code file <name>: OpenAI embedding requires OPENAI_API_KEY
```

修复方法是在重新运行之前将提供方 API key 放入进程环境中。`VOYAGE_API_KEY` 首选于代码（gstack 在设置时默认为 PGLite `voyage-code-3`）；否则 `OPENAI_API_KEY` 回退到 `text-embedding-3-large`。在裸 Mac shell 上，从 `~/.zshrc` 引用 key 再调用。在 Conductor 中，`lib/conductor-env-shim.ts` shim 自动将 `GSTACK_ANTHROPIC_API_KEY` / `GSTACK_OPENAI_API_KEY` 提升为其规范名称；对于 `VOYAGE_API_KEY`，直接在你的 Conductor 工作区环境中设置。重新运行 `/sync-gbrain --code-only` 为已导入页面回填嵌入。

### `gbrain sync` 被 commit 哈希阻塞 —— `FILE_TOO_LARGE`

树中的文件超过了 gbrain 的 5 MB 硬性限制（`gbrain/src/core/import-file.ts` 中的 `MAX_FILE_SIZE`）。常见原因：响应重放缓存、捕获的截图、大型 JSON 固定文件。Gbrain 不支持代码同步的 `.gitignore` 风格的排除列表；唯一的旋钮是确认失败：

```bash
gbrain sync --source <source-id> --skip-failed
```

水印越过有问题的文件。如果同一文件再次更改，它会再次失败；当发生这种情况时重新跳过。

### 切换 PGLite → Supabase 挂起

兄弟 Conductor 工作区中的另一个 gstack 会话可能通过其前导的 `gstack-brain-sync` 调用持有你的本地 PGLite 文件锁。关闭其他工作区，重新运行 `/setup-gbrain --switch`。超时固定在 180 秒，所以你永远不会真正等待永远不会结束。

## 为什么这样设计

**为什么是每远程信任三元组而不是简单的允许/拒绝？** 多客户顾问需要搜索但不回写。一个早上为客户 A 工作、下午为客户 B 工作的自由职业开发者不能让 A 的代码洞察泄漏到 B 可以搜索的大脑中。只读干净地解决了这个问题。

**为什么不将 gbrain 捆绑到 gstack 中？** Gbrain 是一个独立、积极开发的项目，有自己的发布节奏、schema 迁移和 MCP 界面。捆绑意味着 gstack 必须控制 gbrain 更新，这减缓了 gbrain 改进触达用户的速度。分离但集成让每个发布在各自的节奏上。

**为什么 `gbrain init --non-interactive` 通过 env var 而不是标志？** 连接字符串包含数据库密码。将它们作为 argv 传递会落在 `ps`、shell 历史和进程列表中。Env var 交接保持机密仅在进程内存中。Gbrain 同时支持 `GBRAIN_DATABASE_URL` 和 `DATABASE_URL`；我们使用前者以避免与非 gbrain 工具冲突。

**为什么在 PATH 遮蔽上硬性失败而不是警告并继续？** 被遮蔽的 `gbrain` 意味着每个后续命令调用的是与我们刚安装的不同二进制文件。这是一个静默的版本漂移 bug，会导致数周后出现神秘的功能缺口。设置技能的工作只有一个——设置一个工作环境。拒绝在损坏的环境中安装是设置技能正确的行为。

**为什么不自动导入每个仓库？** 隐私 + 噪音。一个你接触每个仓库就自动导入的前导钩子会：（a）在未经同意的情况下将工作代码泄漏到共享大脑中，以及（b）用临时仓库污染搜索。每远程策略使导入成为明确的、按仓库的决策。`/setup-gbrain` 今天没有安装自动导入钩子——但策略存储为未来的一个向前兼容。

## 相关技能 + 下一步

- `/health` —— 包含 GBrain 维度（doctor 状态、同步队列深度、上次推送年龄）在其 0-10 综合评分中。当 gbrain 未安装时省略该维度；在非 gbrain 机器上运行 `/health` 不会惩罚该选择。
- `/gstack-upgrade` —— 保持 gstack 本身最新。**不**单独升级 gbrain。gbrain 默认安装最新 HEAD；要刷新，在你的 gbrain clone（默认 `~/gbrain`）中 `git pull` 并重新运行 `/setup-gbrain`。如果需要可重复性，用 `gstack-gbrain-install --pinned-commit <sha>` 锁定特定 commit。低于最低测试版本的安装会被拒绝。
- `/retro` —— 每周回顾在记忆同步开启时从 gbrain 提取学习和计划，让回顾参考跨机器历史。

运行 `/setup-gbrain` 看看效果如何。
