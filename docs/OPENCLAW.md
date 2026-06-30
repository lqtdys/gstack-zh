# gstack 与 OpenClaw 集成

gstack 与 OpenClaw 集成作为方法论来源，而不是移植的代码库。
OpenClaw 的 ACP 运行时原生生成 Claude Code 会话。gstack 提供规划方法论，让这些会话变得更好。

这是一个编码为提示文本的轻量级协议。没有守护进程。没有 JSON-RPC。
没有兼容性矩阵。提示就是桥梁。

## 架构

```
  OpenClaw                               gstack 存储库
  ─────────────────────                    ──────────────
  Orchestrator: messaging,                 方法论 + 规划的
  calendar, memory, EA                      真实来源
       │                                        │
       ├── 原生技能（对话式）                    ├── 通过 gen-skill-docs 管道
       │   office-hours, ceo-review,           │   生成原生技能
       │   investigate, retro                  │
       │                                        ├── 生成 gstack-lite
       ├── sessions_spawn(runtime: "acp")       │   （规划方法）
       │       │                                │
       │       └── Claude Code                  ├── 生成 gstack-full
       │           └── gstack 已安装在          │   （完整管道）
       │               ~/.claude/skills/gstack  │
       │                                        └── docs/OPENCLAW.md（本文件）
       └── 调度路由（AGENTS.md）
```

## 调度路由

OpenClaw 在生成时决定使用哪一级别的 gstack 支持：

| 级别 | 情况 | 提示前缀 |
|------|------|---------------|
| **简单** | 单文件编辑、拼写错误、配置更改 | 不注入 gstack 上下文 |
| **中等** | 多文件功能、重构 | 追加 gstack-lite CLAUDE.md |
| **重度** | 需要特定 gstack 技能 | "加载 gstack。运行 /X" |
| **完整** | 完整功能、目标、项目 | 追加 gstack-full 管道 |
| **计划** | "帮我规划一个 Claude Code 项目" | 追加 gstack-plan 管道 |

### 决策启发式

- 能在 <10 行代码内完成？ → **简单**
- 是否涉及多文件但方法显而易见？ → **中等**
- 用户是否指定了特定技能（/cso、/review、/qa）？ → **重度**
- 是否是功能、项目或目标（而非任务）？ → **完整**
- 用户是否想为 Claude Code 做规划但不立即实现？ → **计划**

### 调度路由指南（适用于 AGENTS.md）

完整的现成部分位于 `openclaw/agents-gstack-section.md`。
将其复制到你的 OpenClaw AGENTS.md 中。

关键行为规则（这些放在调度级别之上）：

1. **始终生成，从不重定向。** 当用户要求使用任何 gstack 技能时，
   始终生成一个 Claude Code 会话。永远不要告诉用户打开 Claude Code。
2. **确定存储库。** 如果用户指定了存储库，设置工作目录。如果
   不知道，询问是哪个存储库。
3. 自动计划运行到底。** 生成，让它运行完整管道，报告回
   聊天。用户永远不需要离开 Telegram。

### CLAUDE.md 冲突处理

在已有 CLAUDE.md 的存储库中生成 Claude Code 时，将
gstack-lite/full 作为新节追加。不要替换存储库现有指令。

## gstack 为 OpenClaw 生成的内容

所有产物都位于 `openclaw/` 目录下，由
`bun run gen:skill-docs --host openclaw` 生成：

### gstack-lite（中等级别）
`openclaw/gstack-lite-CLAUDE.md` — 约 15 行的规划方法：
1. 修改前读取每个文件
2. 写一个 5 行计划：做什么、为什么、哪些文件、测试用例、风险
3. 使用决策原则解决歧义
4. 报告完成前自我审查
5. 完成报告：已发布的内容、做出的决策、任何不确定事项

A/B 测试：时间翻倍，输出显著提升。

### gstack-full（完整级别）
`openclaw/gstack-full-CLAUDE.md` — 链接现有 gstack 技能：
1. 读取 CLAUDE.md 并理解项目
2. 运行 /autoplan（CEO + eng + 设计审查）
3. 实施批准的计划
4. 运行 /ship 创建 PR
5. 报告 PR URL 和决策

### gstack-plan（计划级别）
`openclaw/gstack-plan-CLAUDE.md` — 完整审查过程，不实施：
1. 运行 /office-hours 生成设计文档
2. 运行 /autoplan（CEO + eng + design + DX 审查 + codex 对抗性审查）
3. 将审查过的计划保存到 `plans/<project-slug>-plan-<date>.md`
4. 报告：计划路径、摘要、关键决策、建议的下一步

编排器将计划链接持久化到其自己的内存存储（brain repo、知识库或 AGENTS.md 中配置的任何内容）。当用户准备就绪时，生成一个 FULL 会话，引用已保存的计划。

### 原生方法论技能
发布到 ClawHub。使用 `clawhub install`：
- `gstack-openclaw-office-hours` — 产品质询（6 个强制性问题）
- `gstack-openclaw-ceo-review` — 战略挑战（10 部分审查，4 种模式）
- `gstack-openclaw-investigate` — 运营调试（4 阶段方法论）
- `gstack-openclaw-retro` — 运营回顾（每周审查）

源代码位于 gstack 存储库中的 `openclaw/skills/` 下。这些是手工制作的 gstack 方法论在 OpenClaw 对话上下文中的改编版本。
没有 gstack 基础设施（没有 browse，没有遥测，没有前置提示）。

## 生成会话检测

当 Claude Code 在 OpenClaw 生成的会话中运行时，应设置 `OPENCLAW_SESSION`
环境变量。gstack 检测此项并调整：
- 跳过交互式提示（自动选择推荐选项）
- 跳过升级检查和遥测提示
- 专注于任务完成和散文式报告

在 sessions_spawn 中设置环境变量：`env: { OPENCLAW_SESSION: "1" }`

## 安装

对于 OpenClaw 用户：告诉你的 OpenClaw 代理 "为 openclaw 安装 gstack"。

代理应该：
1. 将 gstack-lite CLAUDE.md 安装到其编码会话模板中
2. 安装 4 个原生方法论技能
3. 将调度路由添加到 AGENTS.md
4. 通过测试生成验证

对于 gstack 开发：`./setup --host openclaw` 输出此文档。
实际产物由 `bun run gen:skill-docs --host openclaw` 生成。

## 我们不做什么

- 没有调度守护进程（ACP 处理会话生成）
- 没有 Clawvisor 中继（不需要安全层）
- 没有双向学习桥接（brain repo 是知识库）
- 没有 JSON 架构或协议版本控制
- 没有来自 gstack 的 SOUL.md（OpenClaw 有自己的）
- 没有完整技能移植（编码技能保留为 Claude Code 原生）
