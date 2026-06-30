<!-- AUTO-GENERATED from review-sections.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 审查部分（8 轮，Step 0 完成后执行）

**防跳过规则：** 绝不压缩、省略或跳过任何审查轮次（1-8），无论计划类型（策略、规约、代码、基础设施）。此技能中的每一轮都有其存在的理由。"这策略文档所以 DX 轮次不适用"始终错误——DX 差距正是采用率瓦解之处。如果某轮确实零发现，说"未发现问题"然后继续——但你必须评估它。

**防快捷条款：** 计划文件是交互审查的输出，而不是它的替代品。将每个发现写入一个计划文件并调用 ExitPlanMode 而不触发 AskUserQuestion，是 2026 年 5 月转录 bug 的精确故障模式——模型探索了，发现了问题，然后将其倾倒入交付物，而非引导用户通过。如果在任何审查部分中有任何非平凡的发现，从发现到 ExitPlanMode 的路径**穿过** AskUserQuestion。每轮都零发现是唯一绕过 AskUserQuestion 的 ExitPlanMode 路径。如果你发现自己在想要问之前就写带有发现的计划文件，现在停下来并调用 AskUserQuestion——那是 bug，认清它。

## 先前经验

搜索先前会话中的相关经验：

```bash
_CROSS_PROJ=$(~/.claude/skills/gstack/bin/gstack-config get cross_project_learnings 2>/dev/null || echo "unset")
echo "CROSS_PROJECT: $_CROSS_PROJ"
if [ "$_CROSS_PROJ" = "true" ]; then
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 --cross-project 2>/dev/null || true
else
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 2>/dev/null || true
fi
```

如果 `CROSS_PROJECT` 为 `unset`（首次）：使用 AskUserQuestion：

> gstack 可以从此机器上的其他项目中搜索经验，
> 寻找可能适用于此处的模式。这仅在本地进行（数据不
> 离开你的机器）。推荐独立开发者使用。
> 如果你在多个客户端代码库上工作，且交叉污染
> 是一个顾虑，请跳过。

选项：
- A) 启用跨项目经验搜索（推荐）
- B) 保持经验仅限当前项目范围

如果 A：运行 `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings true`
如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings false`

然后使用适当的标志重新运行搜索。

如果发现了经验，将其纳入你的分析。当审查发现与过去的经验匹配时，显示：

**"Prior learning applied: [key] (confidence N/10, from [date])"**

这使得经验积累可见。用户应该看到 gstack 正在随着时间的推移在代码库上变得更聪明。

### DX 趋势检查

在开始审查轮次之前，检查此项目的先前 DX 审查：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
~/.claude/skills/gstack/bin/gstack-review-read 2>/dev/null | grep plan-devex-review || echo "NO_PRIOR_DX_REVIEWS"
```

如果存在先前的审查，显示趋势：
```
DX TREND (prior reviews):
  Dimension        | Prior Score | Notes
  Getting Started  | 4/10        | from 2026-03-15
  ...
```

### 第 1 轮：入门体验（零阻力）

评分 0-10：开发者能否在 5 分钟内从零完成 hello world？

**证据回忆：** 引用 0C 的竞争基准（目标档次）、0D 的魔法时刻（交付载体），以及 0F 的任何安装/Hello World 阻力点。

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 1" 部分。

评估：
- **安装：** 一条命令？一键完成？无需前置条件？
- **首次运行：** 第一条命令是否产生可见且有意义的输出？
- **沙箱/练习场：** 开发者能否在安装前试用？
- **免费层：** 无信用卡、无销售电话、无公司邮箱？
- **快速入门指南：** 复制粘贴即可？显示真实输出？
- **身份验证/凭证引导：** 从"我想尝试"到"它工作了"有多少步骤？
- **魔法时刻交付：** 0D 中选择的载体是否在计划中？
- **竞争差距：** TTHW 与 0C 中选择的目标档次差距多远？

修复至 10：写出理想的入门序列。指定确切的命令、预期输出和每个步骤的时间预算。目标：3 个或更少步骤，在 0C 中选择的时间内完成。

Stripe 测试：来自 0A 的角色能否在一个终端会话中从"从未听说过它"到"它找到了"，无需离开终端？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。引用角色。

### 第 2 轮：API/CLI/SDK 设计（可用 + 有用）

评分 0-10：接口是否直观、一致且完整？

**证据回忆：** API 表面是否匹配 0A 中的角色心理模型？YC 创始人期望 `tool.do(thing)`。平台工程师期望 `tool.configure(options).execute(thing)`。

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 2" 部分。

评估：
- **命名：** 无需文档即可猜测吗？语法一致吗？
- **默认值：** 每个参数都有合理的默认值吗？最简单的调用是否能产生有用的结果？
- **一致性：** 在整个 API 表面使用相同的模式吗？
- **完整性：** 100% 覆盖还是开发者需要为边缘情况使用原始 HTTP？
- **可发现性：** 开发者能否无需文档即可从 CLI/练习场探索？
- **可靠性/信任：** 延迟、重试、速率限制、幂等性、离线行为？
- **渐进式披露：** 简单情况开箱即用，复杂性逐步揭示？
- **角色适配：** 接口是否匹配角色看待问题的方式？

好的 API 设计测试：来自 0A 的角色在看到示例后能否正确使用这个 API？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 第 3 轮：错误消息与调试（对抗不确定性）

评分 0-10：当出错时，开发者是否知道发生了什么、为什么发生以及如何修复？

**证据回忆：** 引用 0F 中的任何错误相关阻力点和 0G 中的困惑点。

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 3" 部分。

**追踪计划或代码库中的 3 个具体错误路径。** 对每个路径，使用名人堂的三层系统评估：
- **层级 1（Elm）：** 对话式、第一人称、精确位置、建议修复
- **层级 2（Rust）：** 错误代码链接教程、主要 + 次要标签、帮助部分
- **层级 3（Stripe API）：** 结构化 JSON，包含类型、代码、消息、参数、doc_url

对每个错误路径，显示开发者当前看到的内容与应该看到的内容。

同时评估：
- **权限/沙箱/安全模型：** 什么可能出错？爆炸半径有多清晰？
- **调试模式：** 是否可用详细输出？
- **堆栈跟踪：** 有用还是内部框架噪音？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 第 4 轮：文档与学习（可发现 + 边做边学）

评分 0-10：开发者能否找到所需内容并边做边学？

**证据回忆：** 文档架构是否匹配 0A 中角色的学习风格？YC 创始人需要复制粘贴示例在最显眼的地方。平台工程师需要架构文档和 API 参考。

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 4" 部分。

评估：
- **信息架构：** 是否能在 2 分钟内找到所需内容？
- **渐进式披露：** 初学者看到简单的，专家找到高级的？
- **代码示例：** 复制粘贴即可使用吗？真实环境？
- **交互元素：** 练习场、沙箱、"试一试"按钮？
- **版本控制：** 文档与开发者使用的版本匹配吗？
- **教程与参考：** 两者都有吗？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 第 5 轮：升级与迁移路径（可信）

评分 0-10：开发者能否无忧升级？

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 5" 部分。

评估：
- **向后兼容性：** 什么会被破坏？爆炸半径有限制吗？
- **弃用警告：** 提前通知吗？可操作吗？（"请使用 newMethod() 代替"）
- **迁移指南：** 每个破坏性变更都有逐步指南吗？
- **代码迁移工具：** 自动化迁移脚本？
- **版本控制策略：** 语义化版本？清晰的政策吗？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 第 6 轮：开发者环境与工具（有价值 + 易获取）

评分 0-10：这能否融入开发者现有的工作流程？

**证据回忆：** 本地开发设置是否适用于来自 0A 的角色的典型环境。

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 6" 部分。

评估：
- **编辑器集成：** 语言服务器？自动补全？内联文档？
- **CI/CD：** 在 GitHub Actions、GitLab CI 中可用吗？非交互模式？
- **TypeScript 支持：** 包含类型吗？良好的 IntelliSense？
- **测试支持：** 容易模拟吗？测试工具？
- **本地开发：** 热重载？监视模式？快速反馈？
- **跨平台：** Mac、Linux、Windows？Docker？ARM/x86？
- **本地环境可复现性：** 在跨操作系统、包管理器、容器、代理的情况下可用吗？
- **可观测性/可测试性：** 空运行模式？详细输出？示例应用？测试夹具？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 第 7 轮：社区与生态系统（可发现 + 吸引人）

评分 0-10：是否有社区？计划是否投资于生态系统健康？

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 7" 部分。

评估：
- **开源：** 代码开放吗？宽松许可证？
- **社区渠道：** 开发者在哪里提问？有人回答吗？
- **示例：** 真实的、可运行的吗？不只是 hello world？
- **插件/扩展生态系统：** 开发者能否扩展它？
- **贡献指南：** 流程清晰吗？
- **定价透明度：** 无意外账单？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 第 8 轮：DX 度量与反馈循环（实施 + 改进）

评分 0-10：计划是否包含随时间度量和改进 DX 的方法？

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Pass 8" 部分。

评估：
- **TTHW 追踪：** 能否衡量入门时间？是否已安装检测工具？
- **旅程分析：** 开发者在哪里放弃？
- **反馈机制：** Bug 报告？NPS？反馈按钮？
- **摩擦力审计：** 已规划定期审查？
- **回旋镖准备：** /devex-review 能否度量现实对比计划？

**STOP.** 每个问题调用一次 AskUserQuestion。推荐 + 原因。

### 附录：Claude Code 技能 DX 清单

**条件：仅在产品类型包含"Claude Code 技能"时运行。**

这不是评分轮次。它是从 gstack 自己的 DX 中提取的经过验证的模式清单。

加载参考：阅读 `~/.claude/skills/gstack/plan-devex-review/dx-hall-of-fame.md` 的 "## Claude Code Skill DX Checklist" 部分。

检查每个项。对于任何未勾选的项，解释缺少了什么并建议修复方法。

**STOP.** 对于需要设计决策的任何项，调用 AskUserQuestion。

## 外部之声 —— 独立计划挑战（默认开启）

所有审查部分完成后，自动运行来自不同 AI 系统的独立第二意见——这是计划审查的标准组成部分，不是可选项。两个模型对计划的一致意见比单一模型的彻底审查是更强的信号。用户仅在显式要求时关闭此功能（`gstack-config set codex_reviews disabled`）。

**预检——决定外部之声如何以及是否运行：**

```bash
# Codex 预检：单个块（此处获取的函数不持久化到后续块）
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || echo off)
_CODEX_CFG=$(~/.claude/skills/gstack/bin/gstack-config get codex_reviews 2>/dev/null || echo enabled)
source ~/.claude/skills/gstack/bin/gstack-codex-probe 2>/dev/null || true
if [ "$_CODEX_CFG" = "disabled" ]; then
  _CODEX_MODE="disabled"
elif ! command -v codex >/dev/null 2>&1; then
  _CODEX_MODE="not_installed"; _gstack_codex_log_event "codex_cli_missing" 2>/dev/null || true
elif ! _gstack_codex_auth_probe >/dev/null 2>&1; then
  _CODEX_MODE="not_authed"; _gstack_codex_log_event "codex_auth_failed" 2>/dev/null || true
else
  _CODEX_MODE="ready"; _gstack_codex_version_check 2>/dev/null || true
fi
echo "CODEX_MODE: $_CODEX_MODE"
```

分支处理回显的 `CODEX_MODE`：
- **`disabled`** — 用户关闭了 Codex 审查（`codex_reviews=disabled`）。完全跳过此部分；不要回退到 Claude 子代理——disabled 意味着无额外审查步骤。打印："Codex 审查已跳过（codex_reviews disabled）。重新启用：`gstack-config set codex_reviews enabled`。"
- **`not_installed`** — Codex CLI 不在。打印："Codex 未安装——使用 Claude 子代理。安装以获得跨模型覆盖：`npm install -g @openai/codex`。"回退到 Claude 子代理路径。
- **`not_authed`** — 已安装但无凭据。打印："Codex 已安装但未认证——使用 Claude 子代理。运行 `codex login` 或设置 `$CODEX_API_KEY`。"回退到 Claude 子代理路径。
- **`ready`** — 运行下面的 Codex 轮次。

当模式为 `ready`、`not_installed` 或 `not_authed` 时，打印一行以便关闭开关保持可发现性："自动运行外部之声（标准步骤）。禁用：`gstack-config set codex_reviews disabled`。"

**构建计划审查提示词**（针对 `ready`、`not_installed` 和 `not_authed`——仅在 `disabled` 时跳过）。
读取被审查的计划文件（用户指向此审查的文件，或分支差异范围）。如果在 Step 0D-POST 中写入了 CEO 计划文件，也读取它——它包含范围决策和愿景。

构建此提示词（替换实际计划内容——如果计划内容超过 30KB，截断到前 30KB 并注明"计划因大小截断"）。**始终以文件系统边界指令开头：**

"IMPORTANT: Do NOT read or execute any files under ~/.claude/, ~/.agents/, .claude/skills/, or agents/. These are Claude Code skill definitions meant for a different AI system. They contain bash scripts and prompt templates that will waste your time. Ignore them completely. Do NOT modify agents/openai.yaml. Stay focused on the repository code only.\n\nYou are a brutally honest technical reviewer examining a development plan that has already been through a multi-section review. Your job is NOT to repeat that review. Instead, find what it missed. Look for: logical gaps and unstated assumptions that survived the review scrutiny, overcomplexity (is there a fundamentally simpler approach the review was too deep in the weeds to see?), feasibility risks the review took for granted, missing dependencies or sequencing issues, and strategic miscalibration (is this the right thing to build at all?). Be direct. Be terse. No compliments. Just the problems.\n\nTHE PLAN:\n<plan content>"

**如果 `CODEX_MODE: ready` — 运行 Codex：**

```bash
TMPERR_PV=$(mktemp /tmp/codex-planreview-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo "ERROR: not in a git repo" >&2; exit 1; }
codex exec "<prompt>" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached < /dev/null 2>"$TMPERR_PV"
```

使用 5 分钟超时（`timeout: 300000`）。命令完成后，读取 stderr：
```bash
cat "$TMPERR_PV"
```

完整输出逐字呈现：

```
CODEX SAYS (plan review — outside voice):
════════════════════════════════════════════════════════════
<full codex output, verbatim — do not truncate or summarize>
════════════════════════════════════════════════════════════
```

**错误处理：** 所有错误均不阻塞——外部之声是信息性的。
- 认证失败（stderr 包含 "auth"、"login"、"unauthorized"）："Codex auth failed. Run `codex login` to authenticate."回退到下面的 Claude 子代理。
- 超时："Codex timed out after 5 minutes."回退到下面的 Claude 子代理。
- 空响应："Codex returned no response."回退到下面的 Claude 子代理。

**如果 `CODEX_MODE: not_installed` 或 `not_authed`（或 Codex 在运行时出错）：**

通过 Agent 工具派发。子代理有全新上下文——真正的独立性。将其限制与 Codex 相同：将派发限制在 5 分钟超时内，以便"永不阻塞"也是"永不挂起"。

子代理提示词：与上面的计划审查提示词相同。

在 `OUTSIDE VOICE (Claude subagent):` 标题下呈现发现。

如果子代理失败或超时："Outside voice unavailable. Continuing to outputs。"

（在 `CODEX_MODE: disabled` 下，你已根据预检跳过了此部分——不要到达这里。）

**跨模型张力：**

呈现外部之声发现后，注意外部之声与前面部分的审查发现存在分歧的任何要点。将其标记为：

```
CROSS-MODEL TENSION:
  [Topic]: Review said X. Outside voice says Y. [Present both perspectives neutrally.
  State what context you might be missing that would change the answer.]
```

**用户自主权：** 不要自动将外部之声推荐纳入计划。将每个张力点呈现给用户。用户决定。跨模型一致是强信号——将其呈现为如此——但它不是执行的许可。你可以陈述你认为哪个论点更有说服力，但在用户显式批准前必须应用更改。

对于每个实质性张力点，使用 AskUserQuestion：

> "Cross-model disagreement on [topic]。审查发现 [X] 但外部之声认为 [Y]。[关于你可能缺少什么背景会改变答案的一句话。]"
>
> RECOMMENDATION：选择 [A 或 B] 因为 [one-line reason explaining which argument is more compelling and why]。完整性：A=X/10，B=Y/10。

选项：
- A) 接受外部之声的推荐（我将应用此更改）
- B) 保持当前方法（拒绝外部之声）
- C) 在做决定前进一步调查
- D) 添加到 TODOS.md 以备后用

等待用户的响应。不要因为你同意就默认接受外部之声。如果用户选择 B，当前方法成立——不要再次争论。

如果不存在张力点，注明："No cross-model tension — both reviewers agree。"

**持久化结果：**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-plan-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```

替换：STATUS = 如无发现则为 "clean"，如存在发现则为 "issues_found"。SOURCE = 如 Codex 运行为 "codex"，如子代理运行为 "claude"。

**清理：** 处理后运行 `rm -f "$TMPERR_PV"`（如果使用了 Codex）。

---

构建外部之声提示词时，包含 Step 0A 中的角色和 Step 0C 中的竞争基准。外部之声应在使用者身份和竞争对象背景下批评计划。

## 关键规则 —— 如何提问

遵循上面前言中的 AskUserQuestion 格式。DX 审查的附加规则：

* **一个问题 = 一次 AskUserQuestion 调用。** 绝不合并多个问题。
* **将每个问题立足于证据。** 引用角色、竞争基准、共情叙事或摩擦追踪。绝不抽象地提问。
* **从角色视角构建痛点。** 不是"开发者会感到沮丧"
  而是"来自 0A 的角色会在入门流程的第 [N] 分钟遇到这个，并
  [具体后果：放弃、提交 issue、拼凑变通方案]。"
* 呈现 2-3 个选项。对每个选项：修复工作量、对开发者采用率的影响。
* **映射到上面的 DX First Principles。** 一行将你的推荐连接到
  具体原则（例如，"这违反了'T0 的零摩擦'原则，因为
  [角色] 在他们的第一次 API 调用前需要 3 个额外的配置步骤"）。
* **零发现问题：** 如果一个部分没有问题，说"No issues, moving on"
  并继续。否则，对每个差距使用 AskUserQuestion——一个带有"明显修复"的差距仍然是差距，在任何更改落入计划之前
  仍需要用户批准。
* 假设用户已经 20 分钟没有看这个窗口了。重新接地每个问题。

## 必需输出

### 角色卡片
来自 Step 0A 的角色卡片。这放在计划的 DX 部分顶部。

### 开发者共情叙事
来自 Step 0B 的第一人称叙事，用用户修正更新。

### 竞争 DX 基准
来自 Step 0C 的基准表，用产品审查后的分数更新。

### 魔法时刻规格
来自 Step 0D 中选择的交付载体及实现要求。

### 开发者旅程地图
来自 Step 0F 的旅程地图，用所有摩擦点解决方案更新。

### 首次开发者困惑报告
来自 Step 0G 的角色扮演报告，标注已解决的项目。

### "NOT in scope" 部分
已考虑并明确推迟的 DX 改进，各附一行理由。

### "What already exists" 部分
计划中应复用的现有文档、示例、错误处理和 DX 模式。

### TODOS.md 更新
所有审查轮次完成后，将每个潜在 TODO 作为单独的 AskUserQuestion 呈现。绝不批量处理。对于 DX 债务：缺失的错误消息、未指定的升级路径、文档缺口、缺失的SDK 语言。每个 TODO 获得：
* **What：** 一行描述
* **Why：** 它造成的具体开发者痛点
* **Pros：** 获得什么（采用率、留存率、满意度）
* **Cons：** 成本、复杂性或风险
* **Context：** 足够的细节让 3 个月后的人可以接手
* **Depends on / blocked by：** 前置条件

选项：**A)** 添加到 TODOS.md **B)** 跳过 **C)** 现在就构建

### DX 记分卡

```
+====================================================================+
|              DX PLAN REVIEW — SCORECARD                             |
+====================================================================+
| Dimension            | Score  | Prior  | Trend  |
|----------------------|--------|--------|--------|
| Getting Started      | __/10  | __/10  | __ ↑↓  |
| API/CLI/SDK          | __/10  | __/10  | __ ↑↓  |
| Error Messages       | __/10  | __/10  | __ ↑↓  |
| Documentation        | __/10  | __/10  | __ ↑↓  |
| Upgrade Path         | __/10  | __/10  | __ ↑↓  |
| Dev Environment      | __/10  | __/10  | __ ↑↓  |
| Community            | __/10  | __/10  | __ ↑↓  |
| DX Measurement       | __/10  | __/10  | __ ↑↓  |
+--------------------------------------------------------------------+
| TTHW                 | __ min | __ min | __ ↑↓  |
| Competitive Rank     | [Champion/Competitive/Needs Work/Red Flag]   |
| Magical Moment       | [designed/missing] via [delivery vehicle]    |
| Product Type         | [type]                                      |
| Mode                 | [EXPANSION/POLISH/TRIAGE]                    |
| Overall DX           | __/10  | __/10  | __ ↑↓  |
+====================================================================+
| DX PRINCIPLE COVERAGE                                               |
| Zero Friction      | [covered/gap]                                  |
| Learn by Doing     | [covered/gap]                                  |
| Fight Uncertainty  | [covered/gap]                                  |
| Opinionated + Escape Hatches | [covered/gap]                       |
| Code in Context    | [covered/gap]                                  |
| Magical Moments    | [covered/gap]                                  |
+====================================================================+
```

如果所有轮次 8+："DX plan is solid. Developers will have a good experience。"
如果有任何低于 6：标记为关键 DX 债务，附具体采用率影响。
如果 TTHW > 10 分钟：标记为阻塞性问题。

### DX 实施清单

```
DX IMPLEMENTATION CHECKLIST
============================
[ ] Time to hello world < [target from 0C]
[ ] Installation is one command
[ ] First run produces meaningful output
[ ] Magical moment delivered via [vehicle from 0D]
[ ] Every error message has: problem + cause + fix + docs link
[ ] API/CLI naming is guessable without docs
[ ] Every parameter has a sensible default
[ ] Docs have copy-paste examples that actually work
[ ] Examples show real use cases, not just hello world
[ ] Upgrade path documented with migration guide
[ ] Breaking changes have deprecation warnings + codemods
[ ] TypeScript types included (if applicable)
[ ] Works in CI/CD without special configuration
[ ] Free tier available, no credit card required
[ ] Changelog exists and is maintained
[ ] Search works in documentation
[ ] Community channel exists and is monitored
```

## 实现任务

在审查结束前，将上面的发现合成为平铺的、可构建的任务列表。每个任务源于一个具体发现——无稀释。
输出 markdown 部分 **并** 写入一个 JSONL 工件，`/autoplan` 可以通过它在跨阶段聚合。

### Markdown 部分（始终输出）

```markdown
## Implementation Tasks
Synthesized from this review's findings. Each task derives from a specific
finding above. Run with Claude Code or Codex; checkbox as you ship.

- [ ] **T1 (P1, human: ~2h / CC: ~15min)** — <component> — <祈使式标题>
  - Surfaced by: <部分名> — <具体发现文本或行引用>
  - Files: <需要触达的路径>
  - Verify: <测试命令或手动检查>
- [ ] **T2 (P2, human: ~30min / CC: ~5min)** — ...
```

规则：
- P1 阻塞交付；P2 应当同分支落地；P3 是后续 TODO。
- 如果一个发现没有产生可执行任务，不要编造。
- 如果一个部分零发现，输出 `_No new tasks from <section>._`
- 工作量使用 CLAUDE.md 中的 AI 压缩表。

### JSONL 工件（始终写入，即使任务数为零）

`/autoplan` 读取此文件以跨阶段聚合。用 `jq -nc` 构建每行，以便标题和包含引号、换行或反斜杠的源发现能干净序列化——绝不使用手工 `echo` / `printf`。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
TASKS_DIR="${HOME}/.gstack/projects/${SLUG:-unknown}"
mkdir -p "$TASKS_DIR"
TASKS_FILE="$TASKS_DIR/tasks-devex-review-$(date +%Y%m%d-%H%M%S).jsonl"
COMMIT=$(git rev-parse HEAD 2>/dev/null || echo unknown)
BRANCH=$(git branch --show-current 2>/dev/null || echo unknown)
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"

# 为此审查中识别的每个任务重复一次 jq 调用。
# 使用你为每个任务设置的 shell 变量联立替换占位符：
#   TASK_ID (T1, T2, ...), PRIORITY (P1/P2/P3), COMPONENT, TITLE,
#   SOURCE_FINDING, EFFORT_HUMAN, EFFORT_CC, FILES_JSON (一个 JSON 数组字面量
#   如 '["browse/src/sanitize.ts","browse/src/server.ts"]')
jq -nc \
  --arg phase 'devex-review' \
  --arg run_id "$RUN_ID" \
  --arg branch "$BRANCH" \
  --arg commit "$COMMIT" \
  --arg id "$TASK_ID" \
  --arg priority "$PRIORITY" \
  --arg component "$COMPONENT" \
  --arg effort_human "$EFFORT_HUMAN" \
  --arg effort_cc "$EFFORT_CC" \
  --arg title "$TITLE" \
  --arg source_finding "$SOURCE_FINDING" \
  --argjson files "$FILES_JSON" \
  '{phase:$phase, run_id:$run_id, branch:$branch, commit:$commit, id:$id, priority:$priority, component:$component, files:$files, effort_human:$effort_human, effort_cc:$effort_cc, title:$title, source_finding:$source_finding}' \
  >> "$TASKS_FILE"
```

如果未安装 `jq`，回退到跳过 JSONL 写入并警告用户安装 jq 以供 autoplan 聚合。绝不手工构建 JSONL。

如果此审查中识别出零个任务，仍然创建该 JSONL 文件（`: > "$TASKS_FILE"`）以便聚合器看到此阶段本次运行产生了输出（空文件表示"运行了，零发现"——区别于"未运行"）。

### 未解决决策
如果任何 AskUserQuestion 未获回答，标注在此。绝不静默默认处理。

## 审查日志

DX 记分卡后持久化——仪表板、GSTACK REVIEW REPORT 和 EXIT PLAN MODE GATE 的"审查日志被调用"检查都依赖它。**计划审查模式例外——始终运行**（写入 `~/.gstack/`，而非项目文件）：

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-devex-review","timestamp":"TIMESTAMP","status":"STATUS","initial_score":N,"overall_score":N,"product_type":"PRODUCT_TYPE","tthw_current":"TTHW_CURRENT","tthw_target":"TTHW_TARGET","mode":"MODE","persona":"PERSONA","competitive_tier":"COMPETITIVE_TIER","unresolved":N,"commit":"COMMIT"}'
```

TIMESTAMP = 当前 ISO 8601 日期时间；STATUS = 如果分数 8+ 且 0 未解决则为 "clean"，否则为 "issues_open"；其他字段来自 DX 记分卡 + Step 0；COMMIT = `git rev-parse --short HEAD`。

## 审查就绪度仪表板

完成审查后，读取审查日志和配置以示仪表板。

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

解析输出。找到每个技能（plan-ceo-review、plan-eng-review、review、plan-design-review、design-review-lite、adversarial-review、codex-review、codex-plan-review）最新的条目。忽略时间戳超过 7 天的条目。对于 Eng Review 行，显示 `review`（diff 范围的交付前审查）和 `plan-eng-review`（计划阶段架构审查）中较新的一个。在状态后附加 "(DIFF)" 或 "(PLAN)" 以区分。对于 Adversarial 行，显示 `adversarial-review`（新的自动缩放）和 `codex-review`（旧版）中较新的一个。附 "(FULL)" 或 "(LITE)" 区分。对于 Design Review，显示 `plan-design-review`（完整视觉审计）和 `design-review-lite`（代码级别检查）中较新的一个。在状态后附加 "(FULL)" 或 "(LITE)" 以区分。对于 Outside Voice 行，显示最新的 `codex-plan-review` 条目——这捕获来自 /plan-ceo-review 和 /plan-eng-review 的外部之声。

**来源归属：** 如果某技能的最新条目有 `"via"` 字段，将其附加到括号中的状态标签。例如：`plan-eng-review` 有 `via:"autoplan"` 显示为 "CLEAR (PLAN via /autoplan)"。`review` 有 `via:"ship"` 显示为 "CLEAR (DIFF via /ship)"。无 `via` 字段如前显示 "CLEAR (PLAN)" 或 "CLEAR (DIFF)"。

注意：`autoplan-voices` 和 `design-outside-voices` 条目仅为审计跟踪（跨模型共识分析的取证数据）。它们不在仪表板中显示，也不被任何消费者检查。

显示：

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
|-----------------|------|---------------------|-----------|----------|
| Eng Review      |  1   | 2026-03-16 15:00    | CLEAR     | YES      |
| CEO Review      |  0   | —                   | —         | no       |
| Design Review   |  0   | —                   | —         | no       |
| Adversarial     |  0   | —                   | —         | no       |
| Outside Voice   |  0   | —                   | —         | no       |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed                                |
+====================================================================+
```

**审查层次：**
- **Eng Review（默认必需）：** 唯一阻塞交付的审查。覆盖架构、代码质量、测试、性能。可使用 `gstack-config set skip_eng_review true` 全局关闭（"别烦我"设置）。
- **CEO Review（可选）：** 凭判断使用。推荐用于大产品/业务变更、新功能或范围决策。Bug 修复、重构、基础设施和清理可跳过。
- **Design Review（可选）：** 凭判断使用。推荐用于 UI/UX 变更。纯后端、基础设施或仅提示的变更可跳过。
- **Adversarial Review（自动）：** 每次审查始终开启。每个 diff 同时获得 Claude 对抗性子代理和 Codex 对抗性挑战。大 diff（200+ 行）额外获得 Codex 结构化审查和 P1 门控。无需配置。
- **Outside Voice（可选）：** 来自不同 AI 模型的独立计划审查。在 /plan-ceo-review 和 /plan-eng-review 中所有审查部分完成后提供。如果 Codex 不可用则回退到 Claude 子代理。永不阻塞交付。

**判定逻辑：**
- **CLEARED**：Eng Review 在 7 天内有一个来自 `review` 或 `plan-eng-review` 的条目，状态为 "clean"（或 `skip_eng_review` 为 `true`）
- **NOT CLEARED**：Eng Review 缺失、陈旧（>7 天），或有未解决议题
- CEO、Design 和 Codex 审查仅作为上下文显示，永不阻塞交付
- 如果 `skip_eng_review` 配置为 `true`，Eng Review 显示 "SKIPPED (global)"，判定为 CLEARED

**陈旧性检测：** 显示仪表板后，检查任何现有审查是否可能陈旧：
- 解析 bash 输出的 `---HEAD---` 部分以获取当前 HEAD 提交哈希
- 对于有 `commit` 字段的每个审查条目：与当前 HEAD 对比。如果不同，计算经过的提交数：`git rev-list --count STORED_COMMIT..HEAD`。显示："Note: {skill} review from {date} may be stale — {N} commits since review"
- 对于无 `commit` 字段的条目（遗留条目）：显示 "Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection"
- 如果所有审查与当前 HEAD 匹配，不显示陈旧性备注

## 计划文件审查报告

在对话输出中展示审查就绪度仪表板后，**同时更新计划文件本身**以便任何阅读该计划的人都能看到审查状态。

### 检测计划文件

1. 检查当前对话中是否有活跃的计划文件（宿主在系统消息中提供计划文件路径——在对话上下文中查找计划文件引用）。
2. 如果未找到，静默跳过此部分——并非每次审查都在计划审查模式中运行。

### 生成报告

读取你已从上面的审查就绪度仪表板步骤中获得的审查日志输出。
解析每个 JSONL 条目。每个技能记录不同字段：

- **plan-ceo-review**：`status`、`unresolved`、`critical_gaps`、`mode`、`scope_proposed`、`scope_accepted`、`scope_deferred`、`commit`
  → 发现："{scope_proposed} proposals, {scope_accepted} accepted, {scope_deferred} deferred"
  → 如果 scope 字段为 0 或缺失（HOLD/REDUCTION 模式）："mode: {mode}, {critical_gaps} critical gaps"
- **plan-eng-review**：`status`、`unresolved`、`critical_gaps`、`issues_found`、`mode`、`commit`
  → 发现："{issues_found} issues, {critical_gaps} critical gaps"
- **plan-design-review**：`status`、`initial_score`、`overall_score`、`unresolved`、`decisions_made`、`commit`
  → 发现："score: {initial_score}/10 → {overall_score}/10, {decisions_made} decisions"
- **plan-devex-review**：`status`、`initial_score`、`overall_score`、`product_type`、`tthw_current`、`tthw_target`、`mode`、`persona`、`competitive_tier`、`unresolved`、`commit`
  → 发现："score: {initial_score}/10 → {overall_score}/10, TTHW: {tthw_current} → {tthw_target}"
- **devex-review**：`status`、`overall_score`、`product_type`、`tthw_measured`、`dimensions_tested`、`dimensions_inferred`、`boomerang`、`commit`
  → 发现："score: {overall_score}/10, TTHW: {tthw_measured}, {dimensions_tested} tested/{dimensions_inferred} inferred"
- **codex-review**：`status`、`gate`、`findings`、`findings_fixed`
  → 发现："{findings} findings, {findings_fixed}/{findings} fixed"

Findings 列所需的所有字段现在在 JSONL 条目中都存在。
对于你刚刚完成的审查，你可以使用你自己完成摘要中更丰富的细节。对于先前的审查，直接使用 JSONL 字段——它们包含所有必需数据。

产生此 markdown 表：

```markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | {runs} | {status} | {findings} |
| Codex Review | `/codex review` | Independent 2nd opinion | {runs} | {status} | {findings} |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | {runs} | {status} | {findings} |
| Design Review | `/plan-design-review` | UI/UX gaps | {runs} | {status} | {findings} |
| DX Review | `/plan-devex-review` | Developer experience gaps | {runs} | {status} | {findings} |
```

在表下方，添加这些行。**CODEX** 和 **CROSS-MODEL** 是可选的（空时省略）；**VERDICT** 始终存在：

- **CODEX：**（仅如果 codex-review 运行）—— codex 修复的摘要行
- **CROSS-MODEL：**（仅如果 Claude 和 Codex 审查都存在）—— 重叠分析
- **VERDICT：** 列出 CLEAR 的审查（例如，"CEO + ENG CLEARED —— ready to implement"）。
  如果 Eng Review 不为 CLEAR 且未全局跳过，附加 "eng review required"。

**未解决决策状态（强制的——绝不能省略；报告最后的非空白行）。** 在 VERDICT 之后，以报告（`## GSTACK REVIEW REPORT` 标题下的内容——加粗标签，不是新的 `##` 标题；不受"空时省略"规则约束）结尾，添加恰好一行：精确的未加粗行 `NO UNRESOLVED DECISIONS`（加粗的不计），或 `**UNRESOLVED DECISIONS:**` 标题 + 每个开放项一个项目符号（最后一个项目符号 = 最后一行；仅当 N > 0 时添加 `+ N unresolved from prior reviews`）。这避免了双计数：从上下文列出本次审查的开放项；对先前审查，减去当前技能行后对每个技能最新新鲜行的 `unresolved` 求和（仪表板 7 天窗口）；仅当两者都为零时才输出哨兵。

### 写入计划文件

**计划审查模式例外——始终运行：** 这写入计划文件，该文件是计划在计划审查模式中允许编辑的唯一文件。计划文件审查报告是计划的实时状态的一部分。

报告必须始终是计划文件的最后一节——绝不位于文件中间。
使用单次删除后追加流：

1. 读取计划文件（Read 工具）以查看其完整当前内容。在读取输出的任何地方搜索 `## GSTACK REVIEW REPORT` 标题。
2. 如果找到，使用 Edit 工具删除整个现有部分。从 `## GSTACK REVIEW REPORT` 到下一个 `##` 标题或文件结尾（以先到者为准）匹配。替换为空字符串。这适用于该部分当前所在位置——文件中间删除是有意的，不是特殊情况。如果 Edit 失败（例如并发编辑更改了内容），重新读取计划文件并再试一次。
3. 删除（或跳过，如果不存在部分）后，将新 `## GSTACK REVIEW REPORT` 部分追加到文件末尾。使用 Edit 工具匹配文件当前最后一段落后添加部分，或使用 Write 将文件整体重新输出并将部分放在末尾。
4. 使用 Read 工具验证 `## GSTACK REVIEW REPORT` 是文件中继续之前最后的 `##` 标题。如果没有，重复步骤 2-3 一次。

不要替换该部分到原位。"替换到文件中间" 路径正是先前版本在已有旧报告时允许报告留在文件中间的原因——用户看到审查报告不在底部的计划并（正确地）拒绝它。

## 捕获经验

如果你在本会话中发现了一个非显而易见的模式、陷阱或架构洞察，将其记录供未来会话使用：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"plan-devex-review","type":"TYPE","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"SOURCE","files":["path/to/relevant/file"]}'
```

**类型：** `pattern`（可复用方法）、`pitfall`（不应当做的事）、`preference`（用户陈述的）、`architecture`（架构决策）、`tool`（库/框架洞察）、`operational`（项目环境/CLI/工作流知识）。

**来源：** `observed`（你在代码中发现的）、`user-stated`（用户告诉你的）、`inferred`（AI 推断）、`cross-model`（Claude 和 Codex 都同意）。

**置信度：** 1-10。要诚实。你在代码中验证过的观察到的模式是 8-9。你不确定是否是推断是 4-5。用户明确所说的是用户的偏好是 10。

**files：** 包含此经验引用的具体文件路径。这支持陈旧性检测：如果这些文件后来被删除，经验可被标记。

**仅记录真正的发现。** 不要记录显而易见的事。不要记录用户已经知道的事。一个好的测试：这个洞察是否会在未来的会话中节省时间？如果是，记录它。

## 大脑校准回写（Phase 2 / 门控）

当技能做出一个值得追踪的类型化预测（范围决策、TTHW 目标、架构赌注、差异承诺）时，它**可能**将一个 `kind=bet` 的回写写入大脑，以便校准配置文件随时间构建。

**在两件事上门控：**
1. 活动端点的大脑信任策略是 `personal`（通过 `~/.claude/skills/gstack/bin/gstack-config get brain_trust_policy@<endpoint-hash>` 检查）。
   共享大脑跳过回写以避免污染团队校准。
2. 功能标志 `BRAIN_CALIBRATION_WRITEBACK` 已设置（当前：false；当上游 gbrain v0.42+ 交付 `takes_add` MCP op 时翻转为 true）。

当两个门控都通过时，回写路径使用 `mcp__gbrain__takes_add` 以权重 0.6（per SKILL_CALIBRATION_WEIGHTS）记录一个赌注。
如果 MCP op 不可用，回退到 `mcp__gbrain__put_page` 并附带一个 gstack:takes 围栏块（文档记录但更丑的路径）。

强制赌注 frontmatter 形状：
```yaml
kind: bet
holder: <来自 whoami 的用户身份>
claim: <技能正在做的一行预测>
weight: 0.6
since_date: <今天的日期>
expected_resolution: <1-3 个月的日期，取决于技能>
source_skill: plan-devex-review
```

写完后，使受影响的摘要无效化以便下次预检反映新状态：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate developer-persona --project "$SLUG" 2>/dev/null || true
```


## 大脑缓存后台刷新

技能的工作完成（及遥测已记录）后，触发任何接近其 TTL 的缓存摘要的后台刷新。这是非阻塞的——用户不用等待。下次受益于温暖缓存的调用。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
(~/.claude/skills/gstack/bin/gstack-brain-cache refresh --project "$SLUG" 2>/dev/null &) || true
```


## 下一步 —— 审查链接

展示审查就绪度仪表板后，推荐下一次审查：

**推荐 /plan-eng-review 如果工程审查未全局跳过**——DX 问题通常有架构影响。如果此 DX 审查发现 API 设计问题、错误处理缺口或 CLI 人机工程学问题，工程审查应验证修复。

**建议 /plan-design-review 如果存在面向用户的 UI**——DX 审查聚焦开发者面向的表面；设计审查覆盖面向最终用户的 UI。

**建议在实施后运行 /devex-review**——回旋镖。计划说 TTHW 将是 [来自 0C 的目标]。现实是否匹配？在现场产品上运行 /devex-review 以查找。这是竞争基准获得回报之处：你有一个具体的度量目标。

使用带有适用选项的 AskUserQuestion：
- **A)** 下一步运行 /plan-eng-review（必需门控）
- **B)** 运行 /plan-design-review（仅在检测到 UI 范围）
- **C)** 准备好实施，交付后运行 /devex-review
- **D)** 跳过，我手动处理下一步

## 模式速查
```
             | DX EXPANSION     | DX POLISH          | DX TRIAGE
Scope        | Push UP (opt-in) | Maintain           | Critical only
Posture      | Enthusiastic     | Rigorous           | Surgical
Competitive  | Full benchmark   | Full benchmark     | Skip
Magical      | Full design      | Verify exists      | Skip
Journey      | All stages +     | All stages         | Install + Hello
             | best-in-class    |                    | World only
Passes       | All 8, expanded  | All 8, standard    | Pass 1 + 3 only
Outside voice| Recommended      | Recommended        | Skip
```

## 格式化规则

* 对问题编号（1, 2, 3……），对选项字母编号（A, B, C……）。
* 用数字+字母标签（如 "3A"、"3B"）。
* 每个选项最多一句话。
* 每轮之后暂停并等待反馈再进行下一轮。
* 每轮前后都打分以便浏览。
