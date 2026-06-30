# 为 gstack 做贡献

欢迎让 gstack 改进。无论你正在修复技能中的错别字，还是构建一个全新的工作流，这份指南都会让你快速上手。

## 快速开始

gstack 技能是由 Claude Code 从 `skills/` 目录发现的 Markdown 文件。它们通常位于 `~/.claude/skills/gstack/`（你的全局安装目录）。但在开发 gstack 本身时，你希望 Claude Code 使用你工作树中的技能 — 这样编辑会立即生效，没有复制或部署的麻烦。

这就是开发模式在做的事。它将你的仓库符号链接到本地 `.claude/skills/` 目录，所以 Claude Code 直接从你的检出读取技能。

```bash
git clone https://github.com/garrytan/gstack.git && cd gstack
bun install                    # 安装依赖
bin/dev-setup                   # 激活开发模式
```

> **完整克隆 vs 浅克隆。** README 的用户安装用了 `--depth 1` 增速。作为贡献者，使用完整克隆（没有 `--depth` 标志） — 你需要历史记录来进行 `git log`、`git blame`、`git bisect`，并审查之前版本的 PR。如果你已经有来自 README 的 `--depth 1` 克隆，用 `git fetch --unshallow` 升级为完整克隆。

现在编辑任何 `SKILL.md`，在 Claude Code 中调用它（如 `/review`），并实时看到你的更改。当你完成开发时：

```bash
bin/dev-teardown                # 停用 — 回到你的全局安装
```

## 运维自我改进

gstack 自动从失败中学习。在每个技能会话结尾，代理反思出了什么问题（CLI 错误、错误方法、项目怪癖），并将运维学习记录到 `~/.gstack/projects/{slug}/learnings.jsonl`。未来的会话会自动吸收这些学习，因此随着时间推移 gstack 对你的代码库更聪明。

无需设置。学习自动记录。用 `/learn` 查看。

### 贡献者工作流

1. **正常使用 gstack** — 运维学习自动捕获
2. **查看你的学习：** `/learn` 或 `ls ~/.gstack/projects/*/learnings.jsonl`
3. **Fork 并克隆 gstack**（如果你还没有的话）
4. **将你的 fork 符号链接到你遇到 bug 的项目中：**
   ```bash
   # 在你的核心项目（那个 gstack 气到过你的项目）
   ln -sfn /path/to/your/gstack-fork .claude/skills/gstack
   cd .claude/skills/gstack && bun install && bun run build && ./setup
   ```
   Setup 创建每个技能目录，内含 SKILL.md 符号链接（`qa/SKILL.md -> gstack/qa/SKILL.md`），
   并询问你的前缀偏好。传入 `--no-prefix` 跳过提示并使用短名称。
5. **修复问题** — 你的更改在此项目中立即生效
6. **实际使用 gstack 进行测试** — 做那件气到过你的事，确认它已修复
7. **从你的 fork 打开 PR**

这是贡献的最佳方式：在做你的真实工作的同时修复 gstack，在你真正感到痛苦的项目中。

### 会话感知

当你同时打开 3+ 个 gstack 会话时，每个问题都会告诉你哪个项目、哪个分支、正在发生什么。不再盯着问题思考"等等，这是哪个窗口？"格式在所有技能中保持一致。

## 在 gstack 仓库内开发 gstack

当你正在编辑 gstack 技能，并想通过在实际使用 gstack 来测试它们
在同一仓库中，`bin/dev-setup` 负责连线。它创建 `.claude/skills/`
符号链接（gitignore）指回你的工作树，所以 Claude Code 使用
你的本地编辑而不是全局安装。

```
gstack/                          <- 你的工作树
├── .claude/skills/              <- dev-setup 创建（gitignore）
│   ├── gstack -> ../../         <- 符号链接回仓库根目录
│   ├── review/                  <- 真实目录（短名称，默认）
│   │   └── SKILL.md -> gstack/review/SKILL.md
│   ├── ship/                    <- 或 gstack-review/、gstack-ship/ 如果 --prefix
│   │   └── SKILL.md -> gstack/ship/SKILL.md
│   └── ...                      <- 每个技能一个目录
├── review/
│   └── SKILL.md                 <- 编辑这，用 /review 测试
├── ship/
│   └── SKILL.md
├── browse/
│   ├── src/                     <- TypeScript 源码
│   └── dist/                    <- 编译二进制（gitignore）
└── ...
```

Setup 创建真实目录（不是符号链接）在顶层，内含 SKILL.md
符号链接在里面。这确保 Claude 将它们发现为顶层技能，而不是嵌套在
`gstack/` 下。名称取决于你的前缀设置（`~/.gstack/config.yaml`）。
短名称（`/review`、`/ship`）是默认的。运行 `./setup --prefix` 如果你
更喜欢命名空间名称（`/gstack-review`、`/gstack-ship`）。

## 日常工作流

```bash
# 1. 进入开发模式
bin/dev-setup

# 2. 编辑一个技能
vim review/SKILL.md

# 3. 在 Claude Code 中测试 — 更改实时生效
#    > /review

# 4. 编辑浏览源码？重建二进制
bun run build

# 5. 今天完成了？拆除
bin/dev-teardown
```

### 开发工作区中的脑感知块（gbrain 已安装）

如果 gbrain 已安装且可用（`bin/gstack-gbrain-detect --is-ok` 退出码为 0），
`bin/dev-setup` 保持你的已跟踪 `SKILL.md` 文件规范，并渲染
脑感知变体（`GBRAIN_CONTEXT_LOAD` / `GBRAIN_SAVE_RESULTS` 块）
到 `.claude/gstack-rendered/`（gitignore，每个工作区）。它将工作区的
`SKILL.md` 符号链接重定向到该渲染，所以你的 Claude 会话获得完整的
gbrain 体验，同时 `git status` 保持干净。在底层，dev-setup
内联传递 `GSTACK_SKIP_GBRAIN_REGEN=1` 给嵌套的 `./setup`（所以它永远不会
弄脏跟踪的源码）并运行 `gen:skill-docs:user --out-dir .claude/gstack-rendered`，
这仅将节基路径重定向到渲染。`bin/dev-teardown`
移除渲染。要在你的*其他*项目的 Claude 会话中让这些块生效，
运行 `gstack-config gbrain-refresh`，它将它们渲染到全局
安装（`~/.claude/skills/gstack`），有保护确保它从不触及符号链接或非 gstack 目录。

## 测试 & 评估

### Setup

```bash
# 1. 复制 .env.example 并添加你的 API 密钥
cp .env.example .env
# 编辑 .env → set ANTHROPIC_API_KEY=sk-ant-...

# 2. 安装依赖（如果你还没有）
bun install
```

Bun 自动加载 `.env` — 无需额外配置。Conductor 工作区自动从主工作树继承 `.env`（见下方"Conductor 工作区"）。

### 测试等级

| 等级 | 命令 | 成本 | 测试内容 |
|------|---------|------|---------------|
| 1 — 静态 | `bun test` | 免费 | 命令验证、快照标志、SKILL.md 正确性、TODOS-format.md 参考、可观测性单元测试 |
| 2 — E2E | `bun run test:e2e` | ~$3.85 | 通过 `claude -p` 子进程执行完整技能 |
| 3 — LLM 评估 | `bun run test:evals` | ~$0.15 独立 | LLM-as-judge 对生成的 SKILL.md 文档评分 |
| 2+3 | `bun run test:evals` | ~$4 合并 | E2E + LLM-as-judge（同时运行两者） |

```bash
bun test                     # 仅等级 1（每次提交都在运行，<5s）
bun run test:e2e              # 等级 2：仅 E2E（需要 EVALS=1，无法在 Claude Code 内运行）
bun run test:evals            # 等级 2 + 3 合并（~$4/次）
```

### 等级 1：静态验证（免费）

用 `bun test` 自动运行。不需要 API 密钥。

- **技能解析器测试**（`test/skill-parser.test.ts`） — 从 SKILL.md bash 代码块提取每个 `$B` 命令，并根据 `browse/src/commands.ts` 中的命令注册表进行验证。捕获错别字、已删除的命令和无效的快照标志。
- **技能验证测试**（`test/skill-validation.test.ts`） — 验证 SKILL.md 文件仅引用真实命令和标志，并且命令描述符质量阈值。
- **生成器测试**（`test/gen-skill-docs.test.ts`） — 测试模板系统：验证占位符正确解析，输出包含标志值提示（如 `-d <N>` 而非仅 `-d`），关键命令的丰富描述（如 `is` 列出有效状态，`press` 列出按键示例）。

### 等级 2：通过 `claude -p` 进行 E2E（~$3.85/次）

将 `claude -p` 作为子进程生成，使用 `--output-format stream-json --verbose`，流式传输 NDJSON 以进行实时进度，并扫描浏览错误。这是最接近"这个技能是否真的端到端工作"的形式。

```bash
# 必须从纯终端运行 — 不能嵌套在 Claude Code 或 Conductor 内
EVALS=1 bun test test/skill-e2e-*.test.ts
```

- 由 `EVALS=1` 环境变量控制（防止意外的高成本运行）
- 如果在 Claude Code 内运行则自动跳过（`claude -p` 无法嵌套）
- API 连接前置检查 — 在烧钱前 ConnectionRefused 快速失败
- 实时进度到 stderr：`[Ns] turn T tool #C: Name(...)`
- 保存完整的 NDJSON 转录和失败 JSON 用于调试
- 测试生活在 `test/skill-e2e-*.test.ts`（按类别分开），运行逻辑在 `test/helpers/session-runner.ts`

**默认为密封。** 每个 E2E 运行器（claude -p、真实 PTY 计划模式运行器、Agent SDK 运行器、以及 codex 和 gemini 运行器）通过 `test/helpers/hermetic-env.ts` 生成它的子进程：允许列表清除的环境、新鲜播种的 `CLAUDE_CONFIG_DIR`、临时的 `GSTACK_HOME` 和 `--strict-mcp-config`。你的操作员 `~/.claude` 配置、MCP 服务器（gbrain、Conductor）、技能、`~/.gstack`
决策日志和 `CONDUCTOR_*` 环境永远不会泄漏到子进程中，因此本地评估
信号与 CI 一致，而不是因为与代码无关的原因产生分歧。设置 `EVALS_HERMETIC=0` 针对你的真实操作员状态进行调试（这也
会丢弃 `--strict-mcp-config`）。接线由 `test/hermetic-wiring.test.ts`（
免费静态引信）和 `test/skill-e2e-hermetic-canary.test.ts` 中的两个门级金丝雀固定。

### E2E 可观测性

当 E2E 测试运行时，它们产生机器可读的构件在 `~/.gstack-dev/` 中：

| 构件 | 路径 | 用途 |
|----------|------|---------|
| 心跳 | `e2e-live.json` | 当前测试状态（每次工具调用后更新） |
| 部分结果 | `evals/_partial-e2e.json` | 已完成的测试（可以在 kill 后存活） |
| 进度日志 | `e2e-runs/{runId}/progress.log` | 仅追加文本日志 |
| NDJSON 转录 | `e2e-runs/{runId}/{test}.ndjson` | 每个测试的原始 `claude -p` 输出 |
| 失败 JSON | `e2e-runs/{runId}/{test}-failure.json` | 失败时的诊断数据 |

**实时仪表盘：** 在第二个终端运行 `bun run eval:watch` 以查看实时显示已完成测试、当前运行测试和成本的仪表盘。使用 `--tail` 同时显示 progress.log 的最后 10 行。

**评估历史工具：**

```bash
bun run eval:list            # 列出所有评估运行（轮数、持续时间、每次运行成本）
bun run eval:compare         # 比较两次运行 — 显示每项测试增量 + 要点评论
bun run eval:summary         # 跨运行聚合统计 + 每项测试效率平均值
```

**用于代理和长套件的分离运行。** 当代理（或你，对于你不想照看的长运行）启动长评估时，使用 `eval:bg*` 脚本。它们将评估命令包装在 `bin/gstack-detach` 中：一个逃避轮次边界 SIGTERM 的会话、阻止空闲睡眠的 `caffeinate` 包装、机器范围的 `gstack-evals` 锁以便并发工作树串行而非饱和模型 API、运行范围的日志在 `~/.gstack-dev/eval-runs/` 下、看门狗和保证的 `### gstack-detach EXIT=<code> ###` 标记以便轮询器永远不会把静默误认为成功。

```bash
bun run eval:bg              # 分离 test:evals（基于 diff）
bun run eval:bg:all          # 分离 test:evals:all
bun run eval:bg:gate         # 分离门级套件
bun run eval:bg:periodic     # 分离周期级套件
```

每个打印其日志路径。在自己的终端中前台运行 `bun run test:evals` 的人不需要这个 — Ctrl-C 是有意的。

**评估比较评论：** `eval:compare` 生成自然语言 Takeaway 部分，解释运行之间发生了什么突出回归、指出改进、调用效率提高（轮数更少、更快、更便宜），并产生总体摘要。这是由 `eval-store.ts` 中 `generateCommentary()` 驱动。

构件永远不会被清理 — 它们积累在 `~/.gstack-dev/` 中以供事后调试和趋势分析。

### 等级 3：LLM-as-judge（~$0.15/次）

使用 Claude Sonnet 从三个维度对生成的 SKILL.md 文档进行评分：

- **清晰度** — AI 代理是否能无歧义地理解指令？
- **完整性** — 是否记录了所有命令、标志和使用模式？
- **可操作性** — 代理仅使用文档中的信息就能执行任务？

每个维度评分 1-5。阈值：每个维度必须评分 **≥ 4**。还有一个回归测试，将生成的文档与来自 `origin/main` 的手维护基线进行比较 — 生成必须评分等于或更高。

```bash
# 需要在 .env 中有 ANTHROPIC_API_KEY — 包含在 bun run test:evals 中
```

- 使用 `claude-sonnet-4-6` 以获得评分稳定性
- 测试生活在 `test/skill-llm-eval.test.ts`
- 直接调用 Anthropic API（非 `claude -p`），所以它在任何地方工作，包括在 Claude Code 内

### CI

GitHub Action（`.github/workflows/skill-docs.yml`）在每次推送和 PR 上运行 `bun run gen:skill-docs --dry-run`。如果生成的 SKILL.md 文件与提交的不同，CI 失败。在合并之前捕获过时的文档

测试直接针对浏览二进制运行 — 它们不需要开发模式。

## 编辑 SKILL.md 文件

SKILL.md 文件是从 `.tmpl` 模板**生成**的。不要直接编辑 `.md` — 你的更改将在下次构建时被覆盖。

```bash
# 1. 编辑模板
vim SKILL.md.tmpl              # 或 browse/SKILL.md.tmpl

# 2. 重新生成所有主机
bun run gen:skill-docs --host all

# 3. 检查健康（报告所有主机）
bun run skill:check

# 或使用监视模式 — 保存时自动重新生成
bun run dev:skill
```

有关模板编写最佳实践（自然语言优于 bash 技巧、动态分支检测、`{{BASE_BRANCH_DETECT}}` 用法），参见 CLAUDE.md 的"编写 SKILL 模板"部分。

要添加浏览命令，添加到 `browse/src/commands.ts`。要添加快照标志，添加到 `browse/src/snapshot.ts` 中的 `SNAPSHOT_FLAGS`。然后重建。

**不要在技能中捆绑 puppeteer/Chromium。** `browse` 是每个盒子共享的
Chromium，包括离线本地渲染工作负载。一个需要光栅化自己的 HTML/JSON（
图表、卡片、og-images）的技能应路由到
`browse` — `screenshot --selector` 用于视觉输出，`load-html` + `js --out` 用于
渲染函数返回的字节 — 而不是 `npm i puppeteer` 并下载不同步的第二个 Chromium；一个安装就能固定，一个守护进程就能管理。

## 行话列表（V1 写作风格）

gstack 的写作风格部分（注入每个 tier-≥2 技能的序言）
在每次技能调用时对技术术语进行首次使用术语表。符合术语表条件的术语列表位于 `scripts/jargon-list.json` 中 — 约 50 个精选的高频术语（幂等、竞态条件、N+1 背压等）。不在列表中的术语被假定足够"白话"。

**添加或删除术语：** 编辑 `scripts/jargon-list.json` 提 PR。
编辑后运行 `bun run gen:skill-docs` — 术语在生成时被烘焙到每个生成的 SKILL.md 中，
所以更改仅在重新生成后才生效。没有运行时加载；没有用户侧覆盖。仓库列表才是真实来源。

适合添加的候选词：非技术用户在评审输出中遇到但没有上下文的高频术语（通用数据库/并发术语、安全术语、前端框架概念）。不要添加仅在一两个小众技能中出现的术语 — 成本收益不值得审查开销。

## 多主机开发

gstack 从一组 `.tmpl` 模板生成 8 台主机的 SKILL.md 文件。
每台主机是 `hosts/*.ts` 中的类型化配置。生成器读取这些配置以生成主机适当的输出（不同的前置元数据、路径、工具名）。

**支持的主机：** Claude（主）、Codex、Factory、Kiro、OpenCode、Slate、Cursor、OpenClaw。

### 为所有主机生成

```bash
# 为特定主机生成
bun run gen:skill-docs                    # Claude（默认）
bun run gen:skill-docs --host codex       # Codex
bun run gen:skill-docs --host opencode    # OpenCode
bun run gen:skill-docs --host all         # 所有 8 台主机

# 或使用构建，做所有主机 + 编译二进制
bun run build
```

### 主机之间有什么区别

每台主机配置（`hosts/*.ts`）控制：

| 方面 | 示例（Claude vs Codex） |
|--------|---------------------------|
| 输出目录 | `{skill}/SKILL.md` vs `.agents/skills/gstack-{skill}/SKILL.md` |
| 前置元数据 | 完整（名称、描述、钩子、版本） vs 最小化（名称 + 描述） |
| 路径 | `~/.claude/skills/gstack` vs `$GSTACK_ROOT` |
| 工具名 | "使用 Bash 工具" vs 相同（Factory 重写为"运行此命令"） |
| 钩子技能 | `hooks:` 前置元数据 vs 内联安全建议说明 |
| 抑制部分 | None vs Codex 自调用部分被剥离 |

参见 `scripts/host-config.ts` 获取完整的 `HostConfig` 接口。

### 测试主机输出

```bash
# 运行所有静态测试（包括所有主机的参数化探针测试）
bun test

# 检查所有主机的最新状态
bun run gen:skill-docs --host all --dry-run

# 健康仪表盘涵盖所有主机
bun run skill:check
```

### 添加新主机

参见 [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md) 获取完整指南。简短版本：

1. 创建 `hosts/myhost.ts`（从 `hosts/opencode.ts` 复制）
2. 添加到 `hosts/index.ts`
3. 将 `.myhost/` 添加到 `.gitignore`
4. 运行 `bun run gen:skill-docs --host myhost`
5. 运行 `bun test`（参数化测试自动覆盖它）

生成器、设置或工具代码零更改。

### 添加新技能

当你添加新技能模板时，所有主机自动获得它：
1. 创建 `{skill}/SKILL.md.tmpl`
2. 运行 `bun run gen:skill-docs --host all`
3. 动态模板发现会上报它，没有需要更新的静态列表
4. 提交 `{skill}/SKILL.md`，外部主机输出在设置时生成并被忽略

## Conductor 工作区

如果你使用 [Conductor](https://conductor.build) 并行运行多个 Claude Code 会话，`conductor.json` 自动连接线工作区生命周期：

| 脚本 | 做什么 |
|--------|-------------|
| `setup` | `bin/dev-setup` | 从主工作树复制 `.env`，安装依赖，符号链接技能，非交互式运行 `./setup`，并且（如果 gbrain 已安装）将脑感知块渲染到 `.claude/gstack-rendered/` 而不弄脏跟踪的源码 |
| `archive` | `bin/dev-teardown` | 移除技能符号链接、`.claude/gstack-rendered/` 渲染，清理 `.claude/` 目录 |

当 Conductor 创建新工作区时，`bin/dev-setup` 自动运行。它检测主工作树（通过 `git worktree list`），复制你的 `.env` 以便 API 密钥传递，并设置开发模式 — 无需手动步骤。

`bin/dev-setup` 完全非交互式运行 `./setup`（它传递 `--plan-tune-hooks=prompt` 并关闭 stdin），所以转发的 Conductor TTY 永远不能挂在隐藏的设置提示上。它也从不安装计划调谐 Claude Code 钩子，这意味着一个临时工作树不能重写你的全局 `~/.claude/settings.json` 指向短暂的工作树路径。要刻意安装计划调谐钩子，在开发环境外运行 `./setup --plan-tune-hooks`（或 `gstack-config set plan_tune_hooks yes`）。

**首次设置：** 将你的 `ANTHROPIC_API_KEY` 放入 `.env` 在主仓库中（见 `.env.example`）。每个 Conductor 工作区自动继承。

**`GSTACK_*` 环境变量前缀（Conductor 注入的密钥）。** Conductor 明确剥离 `ANTHROPIC_API_KEY` 和 `OPENAI_API_KEY` 从每个工作树的环境。`.env` 复制路径也不会恢复它们 — 剥离在 env 继承后发生。想要在 Conductor 工作树中启用付费评估、`/sync-gbrain` 嵌入或 `claude-agent-sdk` 调用的用户必须在 Conductor 的工作树环境配置中设置 `GSTACK_ANTHROPIC_API_KEY` 和 `GSTACK_OPENAI_API_KEY`；Conductor 不触动地传递它们。在 gstack 端，TS 入口点将 `lib/conductor-env-shim.ts` 导入为副作用，当规范名称为空时，将 `GSTACK_FOO_API_KEY` 提升为 `FOO_API_KEY`。如果你添加一个新的击中付费 API 需要 gbrain 嵌入的 TS 入口点，在文件顶部添加 `import "../lib/conductor-env-shim";`。今天 shim 是从 `bin/gstack-gbrain-sync.ts`、`bin/gstack-model-benchmark`、`scripts/reft-agent-sdk.ts` 和 `test/helpers/e2e-helpers.ts` 导入的。

## 需要知道的事项

- **SKILL.md 文件是生成的。** 编辑 `.tmpl` 模板，不是 `.md`。运行 `bun run gen:skill-docs` 重新生成。
- **TODOS.md 是统一待办列表。** 按技能/组件组织，P0-P4 优先级。`/ship` 自动检测已完成项目。所有规划/审查/回顾技能都读取它获取背景。
- **浏览源码更改需要重建。** 如果你触动了 `browse/src/*.ts`，运行 `bun run build`。
- **开发模式遮蔽你的全局安装。** 项目本地技能优先于 `~/.claude/skills/gstack`。`bin/dev-teardown` 恢复全局设置。
- **Conductor 工作区是独立的。** 每个工作区是它自己的 git worktree。`bin/dev-setup` 通过 `conductor.json` 自动运行。
- **`.env` 跨工作树传播。** 在主仓库中设置一次，所有 Conductor 工作区都获取。
- **`.claude/skills/` 是被忽略的。** 符号链接从不被提交。
- **永远不要在 `setup` 中写原生 `ln -snf`。** `setup` 中每个链接站点都必须通过 `_link_or_copy SRC DST` 助手（靠近 `IS_WINDOWS` 探测）。助手在 Unix 上保留 `ln -snf`，并在 Windows 没有开发者模式下切换到 `cp -R` / `cp -f`，原生 `ln -snf` 停止处它被卡住。`test/setup-windows-fallback.test.ts` 用静态不变式强制这一点 — 助手主体外的单个原始 `ln` 调用失败 CI。

## 在真实项目中测试你的更改

**这是开发 gstack 的推荐方式。** 将你的 gstack 检出符号链接到你实际使用它的项目，让你在做真实工作的同时实时看到你的更改。

### 第1步：符号链接你的检出

```bash
# 在你的核心项目（不是 gstack 仓库）
ln -sfn /path/to/your/gstack-checkout .claude/skills/gstack
```

### 第2步：运行 setup 创建每个技能的符号链接

只有 `gstack` 符号链接是不够的。Claude Code 通过
独立的顶层目录（`qa/SKILL.md`、`ship/SKILL.md` 等）发现技能，而不是通过
`gstack/` 目录本身。运行 `./setup` 来创建它们：

```bash
cd .claude/skills/gstack && bun install && bun run build && ./setup
```

Setup 会询问你要短名称（`/qa`）还是命名空间（`/gstack-qa`）。
你的选择被保存到 `~/.gstack/config.yaml` 并记住未来运行。
跳过提示，传入 `--no-prefix`（短名称）或 `--prefix`（命名空间）。

### 第3步：开发

编辑一个模板，运行 `bun run gen:skill-docs`，下一次会立即生效。无需重启。

### 回到稳定的全局安装

移除项目本地符号链接。Claude Code 回退到 `~/.claude/skills/gstack/`：

```bash
rm .claude/skills/gstack
```

每个技能目录（`qa/`、`ship/` 等）包含指向 `gstack/...` 的 SKILL.md 符号链接，所以他们自动解析到全局安装。

### 切换前缀模式

如果你安装了一个前缀设置并想切换：

```bash
cd .claude/skills/gstack && ./setup --no-prefix   # 切换到 /qa、/ship
cd .claude/skills/gstack && ./setup --prefix       # 切换到 /gstack-qa、/gstack-ship
```

Setup 自动清理旧符号链接。不需要手动清理。

## 社区 PR 分类（波次流程）

当社区 PR 堆积时，将它们分成主题波次：

1. **分类** — 按主题分组（安全、功能、基础设施、文档）
2. **去重** — 如果两个 PR 修复同一事物，选择改变行数的那个。关闭另一个，注释指出胜者。
3. **收集分支** — 创建 `pr-wave-N`，合并干净的 PR，解决差劲 PR 的冲突，用 `bun test && bun run build` 验证
4. **带上下文关闭** — 每个关闭的 PR 得到一个注释解释原因和什么（如果有的话）取代了它。贡献者做了真实的工作；用清晰的沟通尊重他们。
5. **作为单个 PR 发布** — 单个 PR 到 main 合并提交。包含合并和关闭内容汇总表。

参见 [PR #205](../../pull/205)（v0.8.3）获取第一个波次的示例。

## 升级迁移

当版本更改表根上的状态（目录结构、配置格式、
旧文件）在 `./setup` 独自不能修复的方式时，添加迁移脚本
所以现有用户获得干净升级。

### 当添加迁移时

- 更改如何创建技能目录（符号链接 vs 真实目录）
- 重命名或移动了 `~/.gstack/config.yaml` 中的配置键
- 需要删除上一版本的孤立文件
- 更改了 `~/.gstack/` 状态文件的格式

不要为以下添加迁移：新功能（用户自动获取）、新技能
（setup 发现他们）、或代码更改为（没有根状态）。

### 如何添加一个

1. 创建 `gstack-upgrade/migrations/v{VERSION}.sh` 其中 `{VERSION}` 匹配
   需要修复的版本文件。
2. 使它可以执行：`chmod +x gstack-upgrade/migrations/v{VERSION}.sh`
3. 脚本必须是**幂等的**（可安全运行多次）和
   **非致命的**（失败被记录但不会阻止升级）。
4. 在顶部包含一个注释块，说明为什么
   需要迁移，以及哪些用户受影响。

示例：

```bash
#!/usr/bin/env bash
# Migration: v0.15.2.0 — 修复技能目录结构
# 受影响：v0.15.2.0 之前使用 --no-prefix 安装的用户
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")/../.." && pwd)"
"$SCRIPT_DIR/bin/gstack-relink" 2>/dev/null || true
```

### 它如何运行

在 `/gstack-upgrade` 期间，在 `./setup` 完成后（Step 4.75），升级技能扫描 `gstack-upgrade/migrations/` 并运行每个比用户的旧版本新的 `v*.sh` 脚本。脚本按版本顺序运行。
失败被记录但从不阻止升级。

### 测试迁移

迁移作为 `bun test` 的一部分被测试（等级 1，免费）。测试套件验证 `gstack-upgrade/migrations/` 中的所有迁移脚本都是可执行且无语法错误。

## 推送你的更改

当你对你的技能编辑满意时：

```bash
/ship
```

这运行测试、审查 diff、对 Greptile 评论进行分类（含 2 级升级）、管理 TODOS.md、提升版本并打开 PR。参见 `ship/SKILL.md` 获取完整工作流。
