# gstack

<!-- ⚠️ 翻译项目声明：本仓库是对 garrytan/gstack 的简体中文翻译版。原始项目：https://github.com/garrytan/gstack | 代码块/命令/路径/技能名保留原文 -->
<!-- 免责：非官方翻译。MIT License (c) 2026 Garry Tan，中文翻译部分无额外限制。 -->

> **📖 简体中文翻译版**
> 
> 本仓库是 [gstack](https://github.com/garrytan/gstack) 的**非官方简体中文翻译版**，旨在帮助中文用户无障碍使用 gstack AI 工程工作流工具。
> 
> - 原始项目：https://github.com/garrytan/gstack
> - 许可证：[MIT License](LICENSE) — Copyright (c) 2026 Garry Tan
> - **代码、CLI 命令、JSON 键名、文件路径、技能名称（如 `/office-hours`）、VO/Tech 术语仍保持英文。**
> 
> 中文翻译部分无额外限制。欢迎提交改进翻译的 PR。

当我听到 Karpathy 这样说时，我想知道如何做到。一个人怎么能像二十人的团队一样发布产品？Peter Steinberger 构建了 [OpenClaw](https://github.com/openclaw/openclaw) — 247K GitHub star — 实质上是 solo 使用 AI agent 完成的。革命已经来临。拥有合适工具的单个建设者可以比传统团队移动得更快。

我是 [Garry Tan](https://x.com/garrytan)，[Y Combinator](https://www.ycombinator.com/) 的总裁兼 CEO。我曾在数千家初创公司工作时 — Coinbase、Instart、Rippling — 当它们还是车库里一两个人的时候。在 YC 之前，我是 Palantir 最早的 eng/PM/设计师之一，联合创立了 Posterous（卖给了 Twitter），并构建了 Bookface，YC 的内部社交网络。

**gstack 是我的回答。** 我已经构建产品二十年了，现在我发布的产品比以往任何时候都多。在过去的 60 天里：3 个生产服务，40+ 已发布功能，兼职，同时全职运营 YC。在逻辑代码变更上 — 不是原始 LOC，AI 会膨胀它 — 我的 2026 运行速度是 **~810× 我的 2013 节奏**（11,417 vs 14 逻辑行/天）。年初至今（截至 4 月 18 日），2026 年已经产生了 **2013 全年的 240×**。跨越 40 个公共 + 私有 `garrytan/*` 仓库（包括 Bookface）测量，排除一个演示仓库后。AI 写了大部分。重点不在于谁输入的，而在于发布了什么。

> LOC 批评者没有错，原始行数因 AI 而膨胀。他们错了的是，经过通胀调整后，我生产力更低了。我更有生产力，多得多。完整方法论、注意事项和复现脚本：**[On the LOC Controversy](docs/ON_THE_LOC_CONTROVERSY.md)**。

**2026 — 1,237 次贡献及持续增长：**

![GitHub contributions 2026 — 1,237 contributions, massive acceleration in Jan-Mar](docs/images/github-2026.png)

**2013 — 当我在 YC 构建 Bookface 时（772 次贡献）：**

![GitHub contributions 2013 — 772 contributions building Bookface at YC](docs/images/github-2013.png)

同一个人。不同的时代。区别在于工具。

**gstack 是我如何做到的。** 它将 Claude Code 变成一个虚拟工程团队 — 一个重新思考产品的 CEO，一个锁定架构的工程经理，一个捕捉 AI 拖沓的设计师，一个找到生产 bugs 的审查员，一个打开真实浏览器的 QA 主管，一个运行 OWASP + STRIDE 审计的安全官，以及一个发布 PR 的发布工程师。二十个专家和八个动力工具，全部 slash 命令，全部 Markdown，全部免费，MIT 许可证。

这是我的开源软件工厂。我每天都在使用它。我分享它是因为这些工具应该对每个人可用。

Fork 它。改进它。让它成为你的。如果你想讨厌免费的开源软件 — 欢迎，但我宁愿你先试试。

**适合谁：**
- **创始人和 CEO** — 特别是那些仍然想发布的技术型创始人
- **第一次使用 Claude Code 的用户** — 结构化角色而不是空白提示
- **技术负责人和资深工程师** — 对每个 PR 进行严格的审查、QA 和发布自动化

## 快速开始

1. 安装 gstack（30 秒 — 见下文）
2. 运行 `/office-hours` — 描述你在构建什么
3. 对任何功能想法运行 `/plan-ceo-review`
4. 对任何有变更的分支运行 `/review`
5. 对你的 staging URL 运行 `/qa`
6. 停在那里。你会知道这是否适合你。

## 安装 — 30 秒

**要求：** [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Git](https://git-scm.com/)、[Bun](https://bun.sh/) v1.0+、[Node.js](https://nodejs.org/)（仅 Windows）

### 第 1 步：安装到你的机器上

打开 Claude Code 并粘贴这个。Claude 做剩下的事情。

> 安装 gstack：运行 **`git clone --single-branch --depth 1 https://github.com/lqtdys/gstack-zh.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`** 然后在 CLAUDE.md 中添加一个 "gstack" 部分，说使用 gstack 的 /browse skill 进行所有网络浏览，永远不要使用 mcp__claude-in-chrome__* 工具，并列出可用 skill：/office-hours、/plan-ceo-review、/plan-eng-review、/plan-design-review、/design-consultation、/design-shotgun、/design-html、/review、/ship、/land-and-deploy、/canary、/benchmark、/browse、/connect-chrome、/qa、/qa-only、/design-review、/setup-browser-cookies、/setup-deploy、/setup-gbrain、/retro、/investigate、/document-release、/document-generate、/codex、/cso、/autoplan、/plan-devex-review、/devex-review、/careful、/freeze、/guard、/unfreeze、/gstack-upgrade、/learn。然后询问用户是否也要将 gstack 添加到当前项目以便队友获得它。

### 第 2 步：团队模式 — 共享仓库的自动更新（推荐）

在你的仓库内部，粘贴这个。将你切换到团队模式，引导仓库以便队友自动获得 gstack，并提交变更：

```bash
(cd ~/.claude/skills/gstack && ./setup --team) && ~/.claude/skills/gstack/bin/gstack-team-init required && git add .claude/ CLAUDE.md && git commit -m "require gstack for AI-assisted work"
```

你的仓库中没有 vendored 文件，没有版本漂移，没有手动升级。每个 Claude Code 会话以快速自动更新检查开始（限制为每小时一次，网络故障安全，完全静默）。

将 `required` 替换为 `optional` 而不是如果你想提醒队友而不是阻止他们。

### OpenClaw

OpenClaw 通过 ACP 生成 Claude Code 会话，所以当 Claude Code 安装了 gstack 时，每个 gstack skill 都能工作。将此粘贴到你的 OpenClaw 代理：

> 安装 gstack：运行 `git clone --single-branch --depth 1 https://github.com/lqtdys/gstack-zh.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup` 为 Claude Code 安装 gstack。然后在 AGENTS.md 中添加"Coding Tasks"部分，说：为编码工作生成 Claude Code 会话时，告诉会话使用 gstack skill。包含这些示例 — 安全审计："Load gstack. Run /cso"，代码审查："Load gstack. Run /review"，QA 测试 URL："Load gstack. Run /qa https://..."，端到端构建功能："Load gstack. Run /autoplan, implement the plan, then run /ship"，构建前计划："Load gstack. Run /office-hours then /autoplan. Save the plan, don't implement."

**设置后，只需自然地与你的 OpenClaw agent 交谈：**

| 你说的 | 发生什么 |
|--------|---------|
| "Fix the typo in README" | 简单 — Claude Code 会话，不需要 gstack |
| "Run a security audit on this repo" | 生成 Claude Code 并执行 `Run /cso` |
| "Build me a notifications feature" | 生成 Claude Code 并执行 /autoplan → implement → /ship |
| "Help me plan the v2 API redesign" | 生成 Claude Code 并执行 /office-hours → /autoplan，保存计划 |

高级调度路由和 gstack-lite/gstack-full 提示模板见 [docs/OPENCLAW.md](docs/OPENCLAW.md)。

### 原生 OpenClaw Skill（通过 ClawHub）

四个方法论 skill 直接在你的 OpenClaw agent 中工作，不需要 Claude Code 会话。从 ClawHub 安装：

```
clawhub install gstack-openclaw-office-hours gstack-openclaw-ceo-review gstack-openclaw-investigate gstack-openclaw-retro
```

| Skill | 做什么 |
|-------|--------|
| `gstack-openclaw-office-hours` | 6 个强制问题的产品审视 |
| `gstack-openclaw-ceo-review` | 具有 4 种范围模式的战略挑战 |
| `gstack-openclaw-investigate` | 根因调试方法论 |
| `gstack-openclaw-retro` | 每周工程回顾 |

这些是对话式 skill。你的 OpenClaw agent 通过聊天直接运行它们。

### 其他 AI Agent

gstack 适用于 10 个 AI 编码 agent，不仅仅是 Claude。设置自动检测你安装了哪些 agent：

```bash
git clone --single-branch --depth 1 https://github.com/lqtdys/gstack-zh.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

或使用 `./setup --host <name>` 针对特定 agent：

| Agent | 标志 | Skill 安装到 |
|-------|------|-----------|
| OpenAI Codex CLI | `--host codex` | `~/.codex/skills/gstack-*/` |
| OpenCode | `--host opencode` | `~/.config/opencode/skills/gstack-*/` |
| Cursor | `--host cursor` | `~/.cursor/skills/gstack-*/` |
| Factory Droid | `--host factory` | `~/.factory/skills/gstack-*/` |
| Slate | `--host slate` | `~/.slate/skills/gstack-*/` |
| Kiro | `--host kiro` | `~/.kiro/skills/gstack-*/` |
| Hermes | `--host hermes` | `~/.hermes/skills/gstack-*/` |
| GBrain (mod) | `--host gbrain` | `~/.gbrain/skills/gstack-*/` |

**想为另一个 agent 添加支持？** 见 [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md)。只有一个 TypeScript 配置文件，零代码变更。

## 看它工作

```
You:    I want to build a daily briefing app for my calendar.
You:    /office-hours
Claude: [问具体例子的问题，不是假设性的]

You:    Multiple Google calendars, events with stale info, wrong locations.
        Prep takes forever and the results aren't good enough...

Claude: I'm going to push back on the framing. You said "daily briefing
        app." But what you actually described is a personal chief of
        staff AI.
        [提取你没有意识到的 5 个功能]
        [挑战 4 个前提 — 你同意、反对或调整]
        [生成 3 种实施方法及努力估计]
        RECOMMENDATION: 明天发布最窄的 wedge，从实际使用中学习。
        完整愿景是一个 3 个月的项目 — 从真正有效的每日简报开始。
        [写设计文档 → 自动输入到下游 skill]

You:    /plan-ceo-review
        [阅读设计文档，挑战范围，运行 10 部分审查]

You:    /plan-eng-review
        [数据流、状态机、错误路径的 ASCII 图]
        [测试矩阵、失败模式、安全问题]

You:    批准计划。退出计划模式。
        [在 11 个文件中写 2,400 行。约 8 分钟。]

You:    /review
        [自动修复] 2 个问题。[询问] 竞态条件 → 你批准修复。

You:    /qa https://staging.myapp.com
        [打开真实浏览器，点击流程，发现并修复错误]

You:    /ship
        测试：42 → 51（+9 新）。PR：github.com/you/app/pull/42
```

你说"每日简报 app。"agent 说"你在构建一个 AI 首席幕僚" — 因为它听了你的痛点，而不是你的功能请求。八个命令，端到端。那不是副驾驶。那是一支团队。

## 冲刺

gstack 是一个过程，不是工具的集合。Skill 按冲刺运行的顺序运行：

**思考 → 计划 → 构建 → 审查 → 测试 → 发布 → 反思**

每个 skill 都输入下一个。`/office-hours` 写一个设计文档，`/plan-ceo-review` 读取它。`/plan-eng-review` 写一个测试计划，`/qa` 拿起它。`/review` 捕获 bugs，`/ship` 验证它们已修复。没有遗漏，因为每一步都知道之前发生了什么。

| Skill | 你的专家 | 做什么 |
|-------|---------|--------|
| `/office-hours` | **YC Office Hours** | 从这里开始。在写代码之前重构你的产品的 6 个强制问题。挑战你的框架，挑战前提，生成实施替代方案。设计文档输入到每个下游 skill。 |
| `/plan-ceo-review` | **CEO / Founder** | 重新思考问题。找到请求中隐藏的 10 星产品。四种模式：Expansion、Selective Expansion、Hold Scope、Reduction。 |
| `/plan-eng-review` | **Eng Manager** | 锁定架构、数据流、图表、边缘情况和测试。强制隐藏的假设暴露。 |
| `/plan-design-review` | **Senior Designer** | 对每个设计维度评分 0-10，解释 10 是什么样子，然后编辑计划以达到那里。AI Slop 检测。交互式 — 每个设计选择一个 AskUserQuestion。 |
| `/plan-devex-review` | **Developer Experience Lead** | 交互式 DX 审查：探索开发者角色，与竞争对手的 TTHW 基准对比，设计你的神奇时刻，逐步追踪摩擦点。三种模式：DX EXPANSION、DX POLISH、DX TRIAGE。20-45 个强制问题。 |
| `/design-consultation` | **Design Partner** | 从零构建完整的设计系统。研究格局，提出创意风险，生成真实的产品模型。 |
| `/review` | **Staff Engineer** | 找到通过 CI 但在生产中爆发的 bugs。自动修复明显的。标记完整性差距。 |
| `/investigate` | **Debugger** | 系统性根因调试。铁律：没有调查不修复。追踪数据流，测试假设，3 次失败后停止。 |
| `/design-review` | **Who Codes 的设计师** | 与 /plan-design-review 相同的审计，然后修复它发现的。原子提交，前后截图。 |
| `/devex-review` | **DX Tester** | 实时开发者体验审计。实际测试你的入门：导航文档，尝试入门流程，计时 TTHW，截图错误。与 `/plan-devex-review` 分数对比 — 显示你的计划是否匹配现实。 |
| `/design-shotgun` | **Design Explorer** | "给我看选项。" 生成 4-6 个 AI 模型变体，在浏览器中打开比较板，收集你的反馈，迭代。味觉记忆学习你喜欢什么。重复直到你喜欢，然后交给 `/design-html`。 |
| `/design-html` | **Design Engineer** | 将模型变成可生产的 HTML。Pretext 计算布局：文本回流，高度调整，布局动态。30KB，零依赖。检测 React/Svelte/Vue。按设计类型智能 API 路由（登陆页 vs 仪表板 vs 表单）。输出是可发布的，不是演示。 |
| `/qa` | **QA Lead** | 测试你的应用，找到错误，用原子提交修复，重新验证。为每个修复自动生成回归测试。 |
| `/qa-only` | **QA Reporter** | 与 /qa 方法论相同但仅报告。纯错误报告，无代码变更。 |
| `/pair-agent` | **Multi-Agent Coordinator** | 与任何 AI agent 共享你的浏览器。一个命令，一次粘贴，已连接。适用于 OpenClaw、Hermes、Codex、Cursor 或任何能 curl 的。每个 agent 获得自己的标签页。自动启动有头模式以便你观看一切。自动启动 ngrok 隧道供远程 agent。作用域令牌、标签页隔离、速率限制、活动归因。 |
| `/cso` | **Chief Security Officer** | OWASP Top 10 + STRIDE 威胁模型。零噪音：17 个误报排除，8/10+ 置信度门控，独立发现验证。每个发现包括具体的利用场景。 |
| `/ship` | **Release Engineer** | 同步 main，运行测试，审计覆盖率，推送，打开 PR。如果你没有则引导测试框架。 |
| `/land-and-deploy` | **Release Engineer** | 合并 PR，等待 CI 和部署，验证生产健康。一个命令从"批准"到"在生产中验证"。 |
| `/canary` | **SRE** | 部署后监控循环。监视控制台错误、性能退化和页面失败。 |
| `/benchmark` | **Performance Engineer** | 基线页面加载时间、Core Web Vitals 和资源大小。在每个 PR 前后对比。 |
| `/document-release` | **Technical Writer** | 更新所有项目文档以匹配你刚刚发布的内容。自动捕获过时的 README。构建 Diataxis 覆盖图（reference / how-to / tutorial / explanation）以便差距在 PR 正文中可见。 |
| `/document-generate` | **Documentation Author** | 使用 Diataxis 框架从头生成缺失的文档。首先研究代码库，然后编写与代码匹配的 reference / how-to / tutorial / explanation 文档。可独立调用或在覆盖率图发现差距时从 `/document-release` 链式调用。了解更多：[tutorial](docs/tutorial-document-generate.md) • [how-to](docs/howto-document-a-shipped-feature.md) • [为什么 Diataxis](docs/explanation-diataxis-in-gstack.md)。 |
| `/retro` | **Eng Manager** | 团队感知的每周回顾。个人分解、发布连胜、测试健康趋势、成长机会。`/retro global` 跨你的所有项目和 AI 工具（Claude Code、Codex、Gemini）运行。 |
| `/browse` | **QA Engineer** | 给 agent 眼睛。真正的 Chromium 浏览器，真正的点击，真正的截图。每个命令约 100ms。`/open-gstack-browser` 启动带有侧边栏、反机器人伪装和自动模型路由的 GStack Browser。 |
| `/setup-browser-cookies` | **Session Manager** | 从你的真实浏览器（Chrome、Arc、Brave、Edge）导入 cookie 到无头会话。测试认证页面。 |
| `/autoplan` | **Review Pipeline** | 一个命令，完全审查的计划。自动运行 CEO → design → eng 审查，编码决策原则。仅呈现供你批准的味道决策。 |
| `/spec` | **Spec Author** | 在五阶段（why、scope、带有强制代码阅读的 technical、draft、file）中将模糊意图变成精确的、可执行的规范。Codex 质量门控在文件之前（低于 7/10 阻止），故障关闭的秘密脱敏，针对现有问题的重复数据删除，存档到 `$GSTACK_STATE_ROOT/projects/$SLUG/specs/` 供团队语料库召回。`--execute` 在新的 worktree 中生成 `claude -p`；`/ship` 在合并时自动关闭源问题。计划模式感知。 |
| `/learn` | **Memory** | 管理 gstack 跨会话学到的东西。审查、搜索、修剪和导出项目特定的模式、陷阱和偏好。学习跨会话积累，所以 gstack 在你的代码库上随时间变得更聪明。 |
| `/make-pdf` | **Publisher** | Markdown 输入，出版物质量的文档输出。Mermaid 和 excalidraw 围栏渲染为矢量图表，完全离线。图像缩放以适应页面并且永不截断；宽图表获得自己的横向页面。`--to html` 发出一个自包含文件，`--to docx` 一个 Word 文档。 |
| `/diagram` | **Diagram Maker** | 英语输入，可编辑图表输出。发出三元组：mermaid 源代码，可以在 excalidraw.com 上打开和编辑的 `.excalidraw`（手绘风格），以及渲染的 SVG/PNG。零网络。将源代码嵌入 markdown 中，`/make-pdf` 渲染它。 |

### 我应该使用哪种审查？

| 构建目标 | 计划阶段（代码之前） | 实时审计（发布后） |
|---------|-------------------|------------------|
| **最终用户**（UI、web app、移动端） | `/plan-design-review` | `/design-review` |
| **开发者**（API、CLI、SDK、文档） | `/plan-devex-review` | `/devex-review` |
| **架构**（数据流、性能、测试） | `/plan-eng-review` | `/review` |
| **以上所有** | `/autoplan`（自动运行 CEO → design → eng → DX，自动检测适用的） | — |

### 动力工具

| Skill | 做什么 |
|-------|--------|
| `/codex` | **第二意见** — 来自 OpenAI Codex CLI 的独立代码审查。三种模式：审查（通过/失败门控）、对抗性挑战和开放咨询。当 `/review` 和 `/codex` 都运行时进行跨模型分析。 |
| `/careful` | **安全护栏** — 在破坏性命令（rm -rf、DROP TABLE、force-push）之前警告。说"be careful"激活。覆盖任何警告。 |
| `/freeze` | **编辑锁** — 将文件编辑限制为一个目录。防止在调试期间意外更改范围外。 |
| `/guard` | **完整安全** — `/careful` + `/freeze` 一个命令。用于生产工作的最大安全。 |
| `/unfreeze` | **解锁** — 移除 `/freeze` 边界。 |
| `/open-gstack-browser` | **GStack Browser** — 启动带有侧边栏、反机器人伪装、自动模型路由（Sonnet 用于动作、Opus 用于分析）、一键 cookie 导入和 Claude Code 集成的 GStack Browser。清理页面、智能截图、编辑 CSS 并将信息传回终端。 |
| `/setup-deploy` | **Deploy Configurator** — `/land-and-deploy` 的一次性设置。检测你的平台、生产 URL 和部署命令。 |
| `/setup-gbrain` | **GBrain Onboarding** — 从零到运行 gbrain 不到 5 分钟。PGLite 本地、Supabase 现有 URL，或通过 Management API 自动配置新的 Supabase 项目。Claude Code + 每个仓库信任三元组（read-write/read-only/deny）的 MCP 注册。[完整指南](USING_GBRAIN_WITH_GSTACK.md)。 |
| `/sync-gbrain` | **Keep Brain Current** — 通过 `gbrain sources add` + `gbrain sync --strategy code` 将此仓库的代码重新索引到 gbrain 中，刷新 CLAUDE.md 中的 `## GBrain Search Guidance` 块，并在能力检查失败时自动移除指导。`--incremental`（默认）、`--full`、`--dry-run`。幂等；可安全地重新运行。 |
| `/gstack-upgrade` | **Self-Updater** — 将 gstack 升级到最新版本。检测全局 vs vendored 安装，同步两者，显示什么改变了。 |
| `/ios-qa` | **iOS 实时设备 QA（v1.43.0.0+）** — 通过 USB CoreDevice 隧道 + 应用内嵌入的 `StateServer` 驱动真实 iPhone。读取 Swift 源、代码生成类型化 `@Observable` 访问器、运行 agent 循环。可选 `--tailnet` 标志将设备暴露给你 Tailscale tailnet 上的 OpenClaw 或任何 HTTP-capable agent，以便远程 agent 无需接触硬件即可运行 iOS QA。能力层允许列表（observe/interact/mutate/restore）、每设备会话锁定、审计日志。 |
| `/ios-fix`、`/ios-design-review`、`/ios-clean`、`/ios-sync` | iOS bug 修复循环、设计师眼睛 HIG 审计、调试桥清理和访问器重新同步。见 `docs/skills.md`。端到端演练：[docs/howto-ios-testing-with-gstack.md](docs/howto-ios-testing-with-gstack.md)。 |

### 新二进制文件（v0.19）

除了 slash-command skill 之外，gstack 还为不属于会话的工作流提供独立 CLI：

| 命令 | 做什么 |
|------|--------|
| `gstack-model-benchmark` | **跨模型基准** — 通过 Claude、GPT（通过 Codex CLI）和 Gemini 运行相同的提示；比较延迟、令牌、成本和（可选）LLM-评审质量分数。每个提供商检测认证，不可用的提供商干净地跳过。输出为表格、JSON 或 markdown。`--dry-run` 验证标志 + 认证而不消耗 API 调用。 |
| `gstack-taste-update` | **设计味觉学习** — 将批准和拒绝写入 `/design-shotgun` 到持久的项目特定味觉档案。每周衰减 5%。反馈到未来的变体生成中，所以系统学习你实际选择的。 |
| `gstack-ios-qa-daemon` | **iOS QA 守护进程** — Mac-side 代理，在 agent 和通过 USB CoreDevice 连接的 iPhone 之间。默认回环；`--tailnet` 打开一个面向 Tailscale 的监听器，具有身份门控的能力层。通过 `~/.gstack/ios-qa-daemon.pid` 上的 flock 实现单实例。见 [docs/howto-ios-testing-with-gstack.md](docs/howto-ios-testing-with-gstack.md)。 |
| `gstack-ios-qa-mint` | **iOS 允许列表管理器** — 用于 tailnet 允许列表的 owner-grant CLI。`grant`/`revoke`/`list` 针对 `~/.gstack/ios-qa-allowlist.json`（模式 0600）。远程 agent 永远不会自动允许列表；这是明确意图路径。 |

### 连续检查点模式（选择加入，默认本地）

设置 `gstack-config set checkpoint_mode continuous` 并且 skill 自动提交你的工作进行，使用 `WIP:` 前缀加上结构化的 `[gstack-context]` 正文（决策、剩余工作、失败的方法）。在崩溃和上下文切换中存活。`/context-restore` 读取这些提交以重建会话状态。`/ship` 在 PR 之前过滤压缩 WIP 提交（保留非 WIP 提交）以便 bisect 保持干净。推送通过 `checkpoint_push=true` 选择加入 — 默认是仅本地的，所以你不会在每次 WIP 提交时触发 CI。

### 领域 skill + 原始 CDP 逃生舱

两个新的浏览器原语随时间复利 gstack agent：

- **`$B domain-skill save`** — agent 保存每站点注释（例如，"LinkedIn 的 Apply 按钮在 iframe 中"），下次访问该主机名时自动触发。隔离 → 3 次成功后激活 → 通过 `$B domain-skill promote-to-global` 可选的跨项目推广。存储与 `/learn` 的项目学习文件一起存在。完整参考：**[docs/domain-skills.md](docs/domain-skills.md)**。
- **`$B cdp <Domain.method>`** — 用于精选命令错过的罕见情况的原始 Chrome DevTools Protocol 逃生舱。拒绝默认：方法必须通过一行理由明确添加到 `browse/src/cdp-allowlist.ts`。两级互斥锁序列化浏览器范围的 CDP 调用与每标签页工作。数据渗出方法的输出包装在 UNTRUSTED 信封中。

> 想要没有栏杆、没有允许列表、没有守护进程的原始 CDP——只是从 agent 到 Chrome 的薄传输？[browser-use/browser-harness-js](https://github.com/browser-use/browser-harness-js) 是不同的哲学（agent 编写的助手 vs gstack 的精选命令），如果你不想要 gstack 的安全堆栈很适合。两者可以共存：gstack 的 `$B cdp` 和工具都可以通过 Playwright 的 `newCDPSession` 附加到同一个 Chrome。

**[每个 skill 的深入探讨、示例和哲学 →](docs/skills.md)**

### Karpathy 的四种失败模式？已经覆盖。

Andrej Karpathy 的[AI 编码规则](https://github.com/forrestchang/andrej-karpathy-skills)（17K star）准确命中了四种失败模式：错误假设、过度复杂性、正交编辑、命令式优于声明式。gstack 的工作流 skill 强制执行所有四种。`/office-hours` 在写代码之前强制假设暴露。Confusion Protocol 停止 Claude 在架构决策上猜测。`/review` 捕获不必要的复杂性。`/ship` 将任务转化为可验证的目标和测试优先执行。如果你已经使用 Karpathy 风格的 CLAUDE.md 规则，gstack 是工作流执行层，使它们在整个冲刺中坚持，而不仅仅是在单个提示上。

## 并行冲刺

gstack 在一个冲刺中工作得很好。十个同时运行时变得有趣。

**设计是核心。** `/design-consultation` 从零构建你的设计系统，研究现有的，提出创意风险，并写 `DESIGN.md`。但真正的魔力是 shotgun 到 HTML 流水线。

**`/design-shotgun` 是你如何探索。** 描述你想要的。它使用 GPT Image 生成 4-6 个 AI 模型变体。然后在你的浏览器中并排打开一个比较板。你选择最喜欢的，留下反馈（"更多留白"、"更大胆的标题"、"去掉渐变"），它生成新一轮。重复直到你喜欢。几轮后味觉记忆开始，所以它开始偏向你实际喜欢的。不再用文字描述你的愿景并希望 AI 得到它。你看选项，选择好的，视觉迭代。

**`/design-html` 使其真实。** 采用批准的模型（来自 `/design-shotgun`、CEO 计划、设计审查或只是一个描述）并将其变成可生产的 HTML/CSS。不是那种 AI HTML 在一个视口宽度看起来很好但在其他地方都坏了。这使用 Pretext 进行计算文本布局：文本在调整大小时实际回流，高度调整到内容，布局动态。30KB 开销，零依赖。它检测你的框架（React、Svelte、Vue）并输出正确的格式。智能 API 路由根据是登陆页、仪表板、表单或卡片布局选择不同的 Pretext 模式。输出是你真正会发布的东西，不是演示。

**`/qa` 是一个巨大的解锁。** 它让我从 6 个并行工作者增加到 12 个。Claude Code 说 *"I SEE THE ISSUE"* 然后实际修复它、生成回归测试并验证修复 — 这改变了我工作方式。Agent 现在有了眼睛。

**智能审查路由。** 就像在运行良好的初创公司：CEO 不必查看基础设施 bug 修复，后端变更不需要设计审查。gstack 追踪运行了什么审查，找出什么是合适的，只做聪明的事。审查准备仪表盘告诉你发布前你站在哪里。

**测试一切。** `/ship` 如果你的项目没有则从头引导测试框架。每个 `/ship` 运行产生覆盖率审计。每个 `/qa` bug 修复生成回归测试。100% 测试覆盖率是目标 — 测试使 vibe coding 安全而不是 yolo coding。

**`/document-release` 是你从未有过的工程师。** 它读取你项目中的每个文档文件，交叉引用 diff，并更新所有漂移的。README、ARCHITECTURE、CONTRIBUTING、CLAUDE.md、TODOS — 全部自动保持最新。现在 `/ship` 自动调用它 — 文档保持最新而无需额外命令。

**真实浏览器模式。** `/open-gstack-browser` 启动 GStack Browser，一个 AI 控制的 Chromium，具有反机器人伪装、自定义品牌和内置侧边栏扩展。像 Google 和 NYTimes 这样的网站无需验证码即可工作。菜单栏显示 "GStack Browser" 而不是 "Chrome for Testing。" 你常规的 Chrome 保持不变。所有现有的 browse 命令继续不变地工作。`$B disconnect` 返回无头。只要窗口打开浏览器就保持存活……没有空闲超时杀死你的工作。

**侧边栏 agent — 你的 AI 浏览器助手。** 在 Chrome 侧面板中输入自然语言，子 Claude 实例执行。"导航到设置页面并截图。" "用测试数据填写此表单。" "遍历此列表中的每个项目并提取价格。" 侧边栏自动路由到正确的模型：Sonnet 用于快速动作（点击、导航、截图），Opus 用于阅读和分析。每个任务最多 5 分钟。侧边栏 agent 在隔离会话中运行，所以它不会干扰你的主 Claude Code 窗口。从侧边栏页脚一键导入 cookie。

**个人自动化。** 侧边栏 agent 不仅用于开发工作流。示例："浏览我孩子的学校家长门户并将所有其他家长的名字、电话和照片添加到我的 Google 联系人。" 两种认证方式：(1) 在有头浏览器中登录一次，你的会话持久存在，或 (2) 点击侧边栏页脚中的"cookies"按钮从你的真实 Chrome 导入 cookie。一旦认证，Claude 导航目录，提取数据并创建联系人。

**提示注入防御。** 敌对的网页试图劫持你的侧边栏 agent。gstack 提供分层防御：一个 22MB ML 分类器与浏览器捆绑，在本地扫描每个页面和工具输出，一个 Claude Haiku 转录检查对整个对话形状投票，系统提示中的随机金丝雀令牌捕获跨文本、工具参数、URL 和文件写入的会话外泄尝试，判决组合器需要两个分类器同意才阻止（防止对 Stack Overflow 样式指令页的单一模型误报）。侧边栏标题中的盾牌图标显示状态（绿色/琥珀色/红色）。通过 `GSTACK_SECURITY_ENSEMBLE=deberta` 选择加入一个 721MB DeBERTa-v3 集成，以获得 2/3 一致。紧急终止开关：`GSTACK_SECURITY_OFF=1`。完整堆栈见 [ARCHITECTURE.md](ARCHITECTURE.md#prompt-injection-defense-sidebar-agent)。

**当 AI 卡住时的浏览器交接。** 遇到验证码、认证墙或 MFA 提示？`$B handoff` 在你所有的 cookie 和标签页完好的情况下，在完全相同的页面打开一个可见的 Chrome。解决问题，告诉 Claude 你完成了，`$B resume` 从它停止的地方继续。Agent 甚至在连续 3 次失败后自动建议它。

**`/pair-agent` 是跨 agent 协调。** 你在 Claude Code 中。你还运行着 OpenClaw。或 Hermes。或 Codex。你想让他们都看同一个网站。输入 `/pair-agent`，选择你的 agent，一个 GStack Browser 窗口打开以便你观看。该 skill 打印一块指令。将该块粘贴到另一个 agent 的聊天中。它交换一次性设置密钥以获取会话令牌，创建自己的标签页，开始浏览。你看到两个 agent 在同一个浏览器中工作，每个在自己的标签页中，彼此无法干扰。如果安装了 ngrok，隧道自动启动以便另一台机器可以在完全不同的机器上获得。同机器 agent 获得直接写入凭据的零摩擦捷径。这是来自不同供应商的 AI agent 第一次可以通过具有真正安全性的共享浏览器进行协调：作用域令牌、标签页隔离、速率限制、域限制和活动归因。

**多 AI 第二意见。** `/codex` 从 OpenAI 的 Codex CLI 获得独立审查 — 一个完全不同的 AI 看着同一个 diff。三种模式：带有通过/失败门控的代码审查、积极尝试破坏你的代码的对抗性挑战，以及具有会话连续性的开放咨询。当 `/review`（Claude）和 `/codex`（OpenAI）都审查了同一个分支时，你得到一个跨模型分析，显示哪些发现重叠，哪些是每个独有的。

**按需安全护栏。** 说"be careful"，`/careful` 在任何破坏性命令之前警告 — rm -rf、DROP TABLE、force-push、git reset --hard。`/freeze` 在调试时将编辑锁定到一个目录，这样 Claude 就不能意外地"修复"不相关的代码。`/guard` 同时激活两者。`/investigate` 自动冻结到正在调查的模块。

**主动 skill 建议。** gstack 注意到你在什么阶段 — 头脑风暴、审查、调试、测试 — 并建议正确的 skill。不喜欢它？说"停止建议"，它会跨会话记住。

## 10-15 个并行冲刺

gstack 在一个冲刺中很强大。在十个同时运行时变革性。

[Conductor](https://conductor.build) 并行运行多个 Claude Code 会话 — 每个在自己的隔离工作区中。一个会话在新想法上运行 `/office-hours`，另一个在 PR 上运行 `/review`，第三个实现一个功能，第四个在 staging 上运行 `/qa`，还有六个在其他分支上。全部同时运行。我经常运行 10-15 个并行冲刺 — 这是目前的实用最大值。

冲刺结构使并行工作成为可能。没有过程，十个 agent 是十个混乱的源头。有了一个过程 — 思考、计划、构建、审查、测试、发布 — 每个 agent 都知道该做什么以及何时停止。你像一个 CEO 管理一个团队一样管理它们：检查重要的决策，让其余的跑。

### 语音输入（AquaVoice、Whisper 等）

gstack skill 有语音友好的触发短语。自然地描述你想要的 — "run a security check"、"test the website"、"do an engineering review" — 正确的 skill 激活。你不需要记住 slash 命令名称或首字母缩写。

## 卸载

### 选项 1：运行卸载脚本

如果 gstack 安装在你的机器上：

```bash
~/.claude/skills/gstack/bin/gstack-uninstall
```

这处理 skill、符号链接、全局状态（`~/.gstack/`）、项目本地状态、浏览守护进程和临时文件。使用 `--keep-state` 保留配置和分析。使用 `--force` 跳过确认。

### 选项 2：手动删除（无本地仓库）

如果你没有克隆仓库（例如，你通过 Claude Code 粘贴安装，后来删除了克隆）：

```bash
# 1. 停止浏览守护进程
pkill -f "gstack.*browse" 2>/dev/null || true

# 2. 移除 SKILL.md 指向 gstack/ 的每 skill 目录
find ~/.claude/skills -mindepth 1 -maxdepth 1 -type d ! -name gstack 2>/dev/null |
while IFS= read -r dir; do
  link="$dir/SKILL.md"
  [ -L "$link" ] || continue
  target=$(readlink "$link" 2>/dev/null) || continue
  case "$target" in
    gstack/*|*/gstack/*)
      rm -f "$link"
      rmdir "$dir" 2>/dev/null || true
      ;;
  esac
done

# 3. 移除 gstack
rm -rf ~/.claude/skills/gstack

# 4. 移除全局状态
rm -rf ~/.gstack

# 5. 移除集成（跳过你从未安装的任何）
rm -rf ~/.codex/skills/gstack* 2>/dev/null
rm -rf ~/.factory/skills/gstack* 2>/dev/null
rm -rf ~/.kiro/skills/gstack* 2>/dev/null
rm -rf ~/.openclaw/skills/gstack* 2>/dev/null

# 6. 移除临时文件
rm -f /tmp/gstack-* 2>/dev/null

# 7. 项目本地清理（从每个项目根目录运行）
rm -rf .gstack .gstack-worktrees .claude/skills/gstack 2>/dev/null
rm -rf .agents/skills/gstack* .factory/skills/gstack* 2>/dev/null
```

### 清理 CLAUDE.md

卸载脚本不编辑 CLAUDE.md。在每个添加了 gstack 的项目中，移除 `## gstack` 和 `## Skill routing` 部分。

### Playwright

`~/Library/Caches/ms-playwright/`（macOS）留在原地，因为其他工具可能共享它。如果没有其他需要就移除它。

---

免费、MIT 许可、开源。无高级层，无候补名单。

我开源了我构建软件的方式。你可以 fork 它并让它成为你的。

> **我们在招聘。** 想要以 AI 编码速度发布真实产品并加固 gstack？
> 来 YC 工作 — [ycombinator.com/software](https://ycombinator.com/software)
> 极具竞争力的薪水和股权。旧金山，Dogpatch 区。

## GBrain — 你的编码 agent 的持久知识

[GBrain](https://github.com/garrytan/gbrain) 是 AI agent 的持久知识库 — 把它看作你的 agent 跨会话实际保留的记忆。GStack 提供从零到"它正在运行，我的 agent 可以调用它"的单命令路径。

```bash
/setup-gbrain
```

四种方式，选一个：

- **Supabase，现有 URL** — 你的云 agent 已经配置了一个大脑；粘贴 Session Pooler URL，现在这台笔记本使用相同的数据。
- **Supabase，自动配置** — 粘贴 Supabase Personal Access Token；skill 创建一个新项目，轮询到健康，获取池器 URL，交给 `gbrain init`。端到端约 90 秒。
- **PGLite 本地** — 零账户，零网络，约 30 秒。仅此 Mac 的隔离大脑。非常适合先试用；以后用 `/setup-gbrain --switch` 迁移到 Supabase。
- **远程 gbrain MCP** — 你的大脑运行在另一台机器上（Tailscale、ngrok、内部 LAN）或队友的服务器上；粘贴 MCP URL 和 bearer token。可选择与本地 PGLite 配对用于分裂引擎模式中的符号感知代码搜索。最适合跨机器内存而不搭建本地 DB。

init 后，skill 提供将 gbrain 注册为 Claude Code 的 MCP 服务器（`claude mcp add gbrain -- gbrain serve`）以便 `gbrain search`、`gbrain put` 等显示为第一流的类型化工具 — 不是 bash shell-outs。

**保持大脑最新。** 从任何仓库运行 `/sync-gbrain` 将其代码重新索引到 gbrain（默认增量，`--full` 完全重新索引，`--dry-run` 预览）。Skill 通过 `gbrain sources add` 将 cwd 注册为联合源，运行 `gbrain sync --strategy code`，并将 `## GBrain Search Guidance` 块写入你的项目 CLAUDE.md 以便 agent 优先使用 `gbrain search`/`code-def`/`code-refs` 而不是 Grep。如果能力检查失败，块会自动移除 — 没有指向未安装工具的陈旧指导。

**每远程信任策略。** 你机器上的每个仓库获得三个层之一：

- `read-write` — agent 可以搜索大脑 AND 从该仓库写回新页面
- `read-only` — agent 可以搜索但从不写（最适合多客户端顾问：搜索共享大脑，不要在 Client B 的仓库中用 Client A 的工作污染它）
- `deny` — 完全没有 gbrain 交互

Skill 每个仓库问一次。决策在同一远程的 worktree 和分支之间是粘性的。

**GStack 内存同步（不同功能，相同私有仓库基础设施）。** 可选择将你的 gstack 状态（学习、CEO 计划、设计文档、回顾、开发者资料）推送到私有 git 仓库，以便你的记忆跟随你跨机器，具有一次性隐私提示（全部允许列表 / 仅工件 / 关闭）和纵深防御秘密扫描器，在它们离开你的机器之前阻止 AWS 密钥、令牌、PEM 块和 JWT。

```bash
gstack-brain-init
```

**在 Conductor 中运行 gstack？** Conductor 显式地从每个工作区的 process env 中剥离 `ANTHROPIC_API_KEY` 和 `OPENAI_API_KEY`，所以付费评估和 gbrain 嵌入不能开箱即用。在 Conductor 的工作区 env 配置中设置 `GSTACK_ANTHROPIC_API_KEY` 和 `GSTACK_OPENAI_API_KEY` — gstack 的 TS 入口点在运行时将它们提升为规范名称。完整详细信息和为新入口点添加导入的贡献者清单：[Conductor + GSTACK\_* env vars](USING_GBRAIN_WITH_GSTACK.md#conductor--gstack_-env-vars)。

**完整 monty — 每个场景、每个标志、每个 bin 助手、每个故障排除步骤：** [USING_GBRAIN_WITH_GSTACK.md](USING_GBRAIN_WITH_GSTACK.md)

其他参考：[docs/gbrain-sync.md](docs/gbrain-sync.md)（同步特定指南） • [docs/gbrain-sync-errors.md](docs/gbrain-sync-errors.md)（错误索引）

## 文档

| 文档 | 涵盖内容 |
|------|--------|
| [Skill Deep Dives](docs/skills.md) | 每个 skill 的哲学、示例和工作流（包括 Greptile 集成） |
| [Diagrams & Document Formats](docs/howto-diagrams-and-formats.md) | PDF 中的 Mermaid/excalidraw 围栏，图像大小和安全默认值，`--to html\|docx`，`/diagram` 三元组 |
| [Builder Ethos](ETHOS.md) | 建设者哲学：沸腾海洋、先搜索后构建，三层知识 |
| [Using GBrain with GStack](USING_GBRAIN_WITH_GSTACK.md) | `/setup-gbrain` 的每个路径、标志、bin 助手和故障排除步骤 |
| [GBrain Sync](docs/gbrain-sync.md) | 跨机器内存设置、隐私模式、故障排除 |
| [Architecture](ARCHITECTURE.md) | 决策和系统内部 |
| [Browser Reference](BROWSER.md) | `/browse` 的完整命令参考 |
| [Contributing](CONTRIBUTING.md) | 开发设置、测试、贡献者模式和开发模式 |
| [Changelog](CHANGELOG.md) | 每个版本的新内容 |

## 隐私与遥测

gstack 包括**选择加入**的使用遥测以帮助改进项目。以下是确切发生的事情：

- **默认关闭。** 除非你明确同意，否则不会发送任何内容。
- **首次运行时，** gstack 询问你是否要共享匿名使用数据。你可以说不。
- **发送的内容（如果你选择加入）：** skill 名称、持续时间、成功/失败、gstack 版本、操作系统。就是这样。
- **永远不发送的内容：** 代码、文件路径、仓库名称、分支名称、提示或任何用户生成的内容。
- **随时更改：** `gstack-config set telemetry off` 立即禁用一切。

数据存储在 [Supabase](https://supabase.com)（开源 Firebase 替代方案）。架构在 [`supabase/migrations/`](supabase/migrations/) 中 — 你可以验证收集的确切内容。仓库中的 Supabase 可发布密钥是公钥（如 Firebase API 键）— 行级安全策略拒绝所有直接访问。遥测通过经过验证的边沿函数流动，强制执行架构检查、事件类型允许列表和字段长度限制。

**本地分析始终可用。** 运行 `gstack-analytics` 从本地 JSONL 文件查看你的个人使用仪表盘 — 无需远程数据。

## 故障排除

**Skill 不出现？** `cd ~/.claude/skills/gstack && ./setup`

**`/browse` 失败？** `cd ~/.claude/skills/gstack && bun install && bun run build`

**陈旧的安装？** 运行 `/gstack-upgrade` — 或在 `~/.gstack/config.yaml` 中设置 `auto_upgrade: true`

**想要更短的命令？** `cd ~/.claude/skills/gstack && ./setup --no-prefix` — 从 `/gstack-qa` 切换到 `/qa`。你的选择被记住供将来升级。

**想要命名空间命令？** `cd ~/.claude/skills/gstack && ./setup --prefix` — 从 `/qa` 切换到 `/gstack-qa`。如果你与其他 skill 包一起运行 gstack 很有用。

**Codex 说"由于无效的 SKILL.md 跳过了加载 skill"？** 你的 Codex skill 描述已过时。修复：`cd ~/.codex/skills/gstack && git pull && ./setup --host codex` — 或对于仓库本地安装：`cd "$(readlink -f .agents/skills/gstack)" && git pull && ./setup --host codex`

**Windows 用户：** gstack 在 Windows 11 上通过 Git Bash 或 WSL 工作。除了 Bun 还需要 Node.js — Bun 在 Windows 上有 Playwright 管道传输的已知 bug（[bun#4253](https://github.com/oven-sh/bun/issues/4253)。浏览服务器自动回退到 Node.js。确保 `bun` 和 `node` 都在你的 PATH 上。

在没有开发者模式的 Windows 上（MSYS2 / Git Bash），`setup` 回退到文件复制而不是符号链接，因为 `ln -snf` 产生在 `git pull` 时不刷新的冻结副本。**每次 `git pull` 后重新运行 `cd ~/.claude/skills/gstack && ./setup`** 以便你的 skill 文件与仓库匹配。`setup` 打印一行注释提醒你。Unix 和 WSL 保持符号链接，不需要重新运行。

**Claude 说它看不到 skill？** 确保你的项目 `CLAUDE.md` 有一个 gstack 部分。添加：

```
## gstack
Use /browse from gstack for all web browsing. Never use mcp__claude-in-chrome__* tools.
Available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review,
/design-consultation, /design-shotgun, /design-html, /review, /ship, /land-and-deploy,
/canary, /benchmark, /browse, /open-gstack-browser, /qa, /qa-only, /design-review,
/setup-browser-cookies, /setup-deploy, /setup-gbrain, /sync-gbrain, /retro, /investigate,
/document-release, /document-generate, /codex, /cso, /autoplan, /pair-agent, /careful, /freeze,
/guard, /unfreeze, /gstack-upgrade, /learn.
```

## 许可证

MIT。永远免费。去构建一些东西。
