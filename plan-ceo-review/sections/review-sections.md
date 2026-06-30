计划审查章节（在范围与模式确定后的 11 个章节）

**反跳过规则：** 不得浓缩、缩写或跳过任何审查章节（1-11），无论计划类型（战略、规格、代码、基础设施）如何。本 skill 中的每个章节都存在都有原因。"这是战略文档，所以实施章节不适用"永远是错的——实施细节正是战略出错的地方。如果某个章节确实没有任何发现，说"未发现问题"并继续——但你必须评估该章节。

**反捷径条款：** 计划文件是交互式评审的输出，而非替代品。把每个发现写入一个计划文件并调用 ExitPlanMode 而不触发 AskUserQuestion，是 2026 年 5 月转录 bug 的确切失败模式——模型探索了、发现了问题，然后把它们倒入了交付物，而非引导用户逐步处理。如果在任何审查章节中有任何非平凡发现，从发现到 ExitPlanMode 的路径必须经过 AskUserQuestion。每个章节零发现才是唯一可以绕过 AskUserQuestion 的通往 ExitPlanMode 路径。如果你发现自己想在提问之前写一个带发现的计划，停下来现在就调用 AskUserQuestion——这就是那个 bug，认清它。

### 第 1 章：架构评审
评估并绘制：
* 总体系统设计和组件边界。绘制依赖图。
* 数据流——所有四次路径。对每个新的数据流，绘制 ASCII 图表：
  *  happy path（数据正确流转）
  *  nil path（输入是 nil/缺失——会怎样？）
  *  empty path（输入存在但为空/零长度——会怎样？）
  *  error path（上游调用失败——会怎样？）
* 状态机。为每个有状态的新的对象绘制 ASCII 图表。包含不可能/无效的转换和防止它们的机制。
* 耦合问题。哪些组件现在被耦合了？这种耦合合理吗？绘制前后依赖图。
* 扩展特性。10 倍负载下什么先崩溃？100 倍呢？
* 单点故障。将它们标出。
* 安全架构。认证边界、数据访问模式、API 接口。对每个新的端点或数据修改：谁能调用它，他们得到什么，他们能改变什么？
* 生产故障场景。对每个新的集成点，描述一个真实的（超时、级联、数据损坏、认证失败）生产故障，并说明计划是否考虑到了。
* 回滚姿态。如果这个发布后立即崩溃了，回滚程序是什么？Git revert？特性标志？数据库迁移回滚？需要多长时间？

**扩展和选择性扩展补充：**
* 什么会让这个架构变得更美？不仅是正确——优雅。是否有这样的设计，让一个新加入的工程师 6 个月后会说"哦，这既巧妙又显而易见"？
* 什么基础设施会让这个功能成为其他功能可以构建的平台？

**选择性扩展：** 如果任何被接受的从 0D 步骤挑选的选项影响架构，在这里评估其架构适配性。标记任何产生耦合问题或整合不干净的——这是一个用新信息重新审视该决定的机会。

必需的 ASCII 图表：完整的系统架构，展示新组件及与现有组件的关系。
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 2 章：错误与救援图
这是抓住静默故障的章节。它不可选。
对每个可能失败的新方法、服务或代码路径，填写此表格：
```
  METHOD/CODEPATH          | WHAT CAN GO WRONG           | EXCEPTION CLASS
  -------------------------|-----------------------------|-----------------
  ExampleService#call      | API timeout                 | TimeoutError
                           | API returns 429             | RateLimitError
                           | API returns malformed JSON  | JSONParseError
                           | DB connection pool exhausted| ConnectionPoolExhausted
                           | Record not found            | RecordNotFound
  -------------------------|-----------------------------|-----------------

  EXCEPTION CLASS              | RESCUED?  | RESCUE ACTION          | USER SEES
  -----------------------------|-----------|------------------------|------------------
  TimeoutError                 | Y         | Retry 2x, then raise   | "Service temporarily unavailable"
  RateLimitError               | Y         | Backoff + retry         | Nothing (transparent)
  JSONParseError               | N <GAP>   | —                      | 500 error <BAD
  ConnectionPoolExhausted      | N <GAP>   | —                      | 500 error <BAD
  RecordNotFound               | Y         | Return nil, log warning | "Not found" message
```
本章节规则：
* 笼统的错误处理（`rescue StandardError`, `catch (Exception e)`, `except Exception`）始终是异味。命名具体异常。
* 捕获错误时只用通用日志消息是不够的。记录完整上下文：尝试什么操作，用什么参数，为用户/请求做了什么。
* 每个被救援的错误必须：带退避重试、对用户可见消息优雅降级，或添加上下文后重新抛出。"吞下并继续"几乎不可接受。
* 对每个 GAP（应该被救援但未被救援的错误）：指定救援动作和用户应该看到什么。
* 专门对 LLM/AI 服务调用：响应畸形时怎样？空时怎样？幻觉无效 JSON 时怎样？模型返回拒绝时怎样？每种都是独立故障模式。
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 3 章：安全与威胁模型
安全不是架构的子项目。它有独立章节。
评估：
* 攻击面扩大。这个计划引入了什么新的攻击向量？新端点、新参数、新文件路径、新后台任务？
* 输入验证。对每个新的用户输入：是否验证、净化、失败时显式拒绝？以下情况会怎样：nil、空字符串、应该整数却是字符串、超过最大长度的字符串、unicode 边界情况、HTML/脚本注入尝试？
* 授权。对每个新的数据访问：是否限定到正确的用户/角色？是否有直接对象引用漏洞？用户 A 能否通过操纵 ID 访问用户 B 的数据？
* 密钥与凭据。新的密钥？在环境变量中而非硬编码？可轮换？
* 依赖风险。新的 gem/npm 包？安全跟踪记录？
* 数据分类。PII、支付数据、凭据？处理与现有模式一致吗？
* 注入向量。SQL、命令、模板、LLM 提示注入——检查所有。
* 审计日志。对敏感操作：是否有审计跟踪？

对每个发现：威胁、可能性（高/中/低）、影响（高/中/低）以及计划是否缓解了它。
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 4 章：数据流与交互边界情况
本章以对抗性彻底程度追踪数据通过系统和 UI 中的交互。

**数据流追踪：** 对每个新的数据流，生成 ASCII 图表展示：
```
  INPUT VALIDATION TRANSFORM PERSIST OUTPUT
    | | | | |
    v v v v v
  [nil?] [invalid?] [exception?] [conflict?] [stale?]
  [empty?] [too long?] [timeout?] [dup key?] [partial?]
  [wrong [wrong type?] [OOM?] [locked?] [encoding?]
   type?]
```
对每个节点：每个阴影路径上发生什么？是否测试了？

**交互边界情况：** 对每个新的用户可见交互，评估：
```
  INTERACTION | EDGE CASE | HANDLED? | HOW?
  ---------------------|------------------------|----------|--------
  Form submission | Double-click submit | ? |
                       | Submit with stale CSRF | ? |
                       | Submit during deploy | ? |
  Async operation | User navigates away | ? |
                       | Operation times out | ? |
                       | Retry while in-flight | ? |
  List/table view | Zero results | ? |
                       | 10,000 results | ? |
                       | Results change mid-page| ? |
  Background job | Job fails after 3 of | ? |
                       | 10 items processed | |
                       | Job runs twice (dup) | ? |
                       | Queue backs up 2 hours | ? |
```
标记任何未处理的边界情况为缺口。对每个缺口，指定修复方案。
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 5 章：代码质量评审
评估：
* 代码组织和模块结构。新代码是否符合现有模式？如有偏离，是否有理由？
* DRY 违规。要激进。如果同处逻辑已存在，标记它并引用文件和行号。
* 命名质量。新的类、方法和变量是按它们做什么命名的，而不是怎么做？
* 错误处理模式。（交叉引用第 2 ——本章评审模式；第 2 映射细节。）
* 缺少边界情况。显式列出："X 是 nil 会怎样？""API 返回 429 会怎样？"等等。
* 过度工程检查。任何解决不存在问题的新抽象？
* 工程不足检查。任何脆弱的、只假设 happy path 或缺少明显防御性检查的？
* 圈复杂度。标记任何分支超过 5 次的新方法。提出重构建议。
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 6 章：测试评审
为这个计划引入的每个新事物绘制完整图表：
```
  NEW UX FLOWS:
    [列出每个新的用户可见交互]

  NEW DATA FLOWS:
    [列出每条数据通过系统的新路径]

  NEW CODEPATHS:
    [列出每个新的分支、条件或执行路径]

  NEW BACKGROUND JOBS / ASYNC WORK:
    [列出每个]

  NEW INTEGRATIONS / EXTERNAL CALLS:
    [列出每个]

  NEW ERROR/RESCUE PATHS:
    [列出每个——交叉引用第 2 章]
```
对图表中的每个项目：
* 什么类型的测试覆盖它？（单元/集成/系统/E2E）
* 计划中是否有针对它的测试？如果没有，编写测试规范标题。
* happy path 测试是什么？
* 失败路径测试是什么？（具体——哪个失败？）
* 边界情况测试是什么？（nil、空、边界值、并发访问）

测试野心检查（所有模式）：对每个新功能，回答：
* 什么测试让你有信心周五凌晨 2 点发布？
* 一个敌对的 QA 工程师会写什么来打破这个？
*  混乱测试是什么？

测试金字塔检查：大量单元、较少集成、少许 E2E？还是反过来的？
不稳定性风险：标记任何依赖时间、随机性、外部服务或顺序的测试。
负载/压力测试需求：对任何被频繁调用或处理大量数据的新代码路径。

对 LLM/prompt 变更：检查 CLAUDE.md 中"Prompt/LLM 变更"文件模式。如果该计划涉及任何这些模式，说明必须运行哪些 eval 套件，应添加哪些用例，以及与哪些基线进行比较。
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 7 章：性能评审
评估：
* N+1 查询。对每个新的 ActiveRecord 关联遍历：是否有 includes/preload？
* 内存使用。对每个新的数据结构：生产环境中最大大小是多少？
* 数据库索引。对每个新的查询：是否有索引？
* 缓存机会。对每个昂贵的计算或外部调用：应该缓存它吗？
* 后台任务规模。对每个新任务：最坏情况负载、运行时间、重试行为？
* 慢路径。最慢的 3 个新代码路径和估计 p99 延迟。
* 连接池压力。新的 DB 连接、Redis 连接、HTTP 连接？
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 8 章：可观测性与可调试性评审
新系统会崩溃。本章确保你能看到原因。
评估：
* 日志。对每个新的代码路径：在入口、出口和每个重要分支处有结构化日志行？
* 指标。对每个新功能：什么指标告诉你它在工作？什么告诉你它坏了？
* 追踪。对新的跨服务或跨任务流：trace ID 是否传播？
* 告警。应该存在哪些新告警？
* 仪表板。第一天你想要哪些新的仪表板面板？
* 可调试性。如果发布 3 周后报告一个 bug，仅从日志你能重建发生了什么吗？
* 管理工具。需要管理员 UI 或 rake 任务的新运维任务？
* 运行手册。对每个新的故障模式：运维响应是什么？

**扩展和选择性扩展补充：**
* 什么可观测性会让这个功能运维起来很愉快？（对选择性扩展，包含任何被挑选选项的可观测性。）
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 9 章：部署与发布评审
评估：
* 迁移安全。对每个新的 DB 迁移：向后兼容？零停机？表锁？
* 特性标志。是否有部分应该在特性标志后面？
* 发布顺序。正确顺序：先迁移，后部署？
* 回滚计划。显式的逐步。
* 部署时风险窗口。旧代码和新代码同时运行——什么会崩溃？
* 环境一致性。在 staging 测试过吗？
* 部署后验证清单。前 5 分钟？前一小时？
* 冒烟测试。部署后应立即运行哪些自动化检查？

**扩展和选择性扩展补充：**
* 什么部署基础设施会让这个功能的发布变得常规？（对选择性扩展，评估被挑选的选项是否改变了部署风险画像。）
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 10 章：长期轨迹评审
评估：
* 引入的技术债务。代码债务、运维债务、测试债务、文档债务。
* 路径依赖。这是否让未来变更更难？
* 知识集中度。文档对新工程师足够吗？
* 可逆性。1-5 评分：1 = 单向门，5 = 容易逆转。
* 生态系统适配。与 Rails/JS 生态系统方向一致吗？
* 一年后的问题。12 个月后作为新工程师读这个计划——显而易见吗？

**扩展和选择性扩展补充：**
* 这个发布完成后是什么？阶段 2？阶段 3？架构支持那个轨迹吗？
* 平台潜力。这是否创造了其他功能可以利用的能力？
* （仅限选择性扩展）回顾：是否接受了正确的挑选？是否有任何被拒绝的扩展对被接受的来说是承重墙？
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

### 第 11 章：设计与 UX 评审（如果未检测到 UI 范围则跳过）
CEO 召唤设计师。不是像素级审计——那是 /plan-design-review 和 /design-review。这是确保计划有设计意图。

评估：
* 信息架构——用户首先、其次、第三看到什么？
* 交互状态覆盖地图：
  FEATURE | LOADING | EMPTY | ERROR | SUCCESS | PARTIAL
* 用户旅程连贯性——情感弧度的故事板
* AI slop 风险——计划是否描述了通用 UI 模式？
* DESIGN.md 对齐——计划是否匹配声明的设计系统？
* 响应式意图——提到移动版还是事后补充？
* 无障碍基础知识——键盘导航、屏幕阅读器、对比度、触摸目标

**扩展和选择性扩展补充：**
* 什么会让这个 UI 感觉*必然*？
* 什么 30 分钟的 UI 微调会让用户想"哦太好了，他们想到了"？

必需的 ASCII 图表：用户流程展示屏幕/状态和转换。

如果这个计划有重要的 UI 范围，建议："考虑在实施前运行 /plan-design-review 对这个计划进行深度设计评审。"
**停止。每个问题调用一次 AskUserQuestion。不要批量处理。推荐+原因。** 如果本章节零发现，说明"没有问题，继续"并继续。如果本章节有问题，你必须作为 tool_use 调用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。在用户回复之前不要继续。
**提醒：不要做任何代码更改。仅限评审。**

## Outside Voice —— 独立计划挑战（默认开启）

在所有审查章节完成后，自动从一个不同的 AI 系统运行一个独立的第二意见——它是计划评审的标准部分，不是选择加入。两个模型同意一个计划比一个模型彻底评审的信号更强。用户只有在明确要求时才关闭此项（`gstack-config set codex_reviews disabled`）。

**预检——决定外部声音是否以及如何运行：**

```bash
# Codex preflight: one block (functions sourced here don't persist to later blocks).
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

对回显的 `CODEX_MODE` 进行分支：
- **`disabled`** —— 用户关闭了 Codex 评审（`codex_reviews=disabled`）。完全跳过本章节；不要回退到子代理——禁用意味着无额外评审步骤。打印："Codex review skipped (codex_reviews disabled). Re-enable: `gstack-config set codex_reviews enabled`."
- **`not_installed`** —— Codex CLI 不存在。打印："Codex not installed — using Claude subagent. Install for cross-model coverage: `npm install -g @openai/codex`." 回退到子代理路径。
- **`not_authed`** —— 已安装但没有凭据。打印："Codex installed but not authenticated — using Claude subagent. Run `codex login` or set `$CODEX_API_KEY`." 回退到子代理路径。
- **`ready`** —— 运行下面的 Codex 传递。

当模式为 `ready`、`not_installed` 或 `not_authed` 时，打印一行以使关闭开关可发现："Running the outside voice automatically (standard step). Disable: `gstack-config set codex_reviews disabled`."

**构建计划评审提示**（对 `ready`、`not_installed` 和 `not_authed` —— 仅在 `disabled` 时跳过）。
读取正在评审的计划文件（用户指向的文件，或分支 diff 范围）。如果在 0D 后步骤中编写了 CEO 计划文档，也读取它——它包含范围决策和愿景。

构建此提示（替换实际计划内容——如果计划内容超过 30KB，截断到前 30KB 并注明"Plan truncated for size"）。**始终以文件系统边界指令开始：**

"IMPORTANT: Do NOT read or execute any files under ~/.claude/, ~/.agents/, .claude/skills/, or agents/. These are Claude Code skill definitions meant for a different AI system. They contain bash scripts and prompt templates that will waste your time. Ignore them completely. Do NOT modify agents/openai.yaml. Stay focused on the repository code only.

You are a brutally honest technical reviewer examining a development plan that has already been through a multi-section review. Your job is NOT to repeat that review. Instead, find what it missed. Look for: logical gaps and unstated assumptions that survived the review scrutiny, overcomplexity (is there a fundamentally simpler approach the review was too deep in the weeds to see?), feasibility risks the review took for granted, missing dependencies or sequencing issues, and strategic miscalibration (is this the right thing to build at all?). Be direct. Be terse. No compliments. Just the problems.

THE PLAN:
<plan content>"

**如果 `CODEX_MODE: ready` —— 运行 Codex：**

```bash
TMPERR_PV=$(mktemp /tmp/codex-planreview-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo "ERROR: not in a git repo" >&2; exit 1; }
codex exec "<prompt>" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached < /dev/null 2>"$TMPERR_PV"
```

使用 5 分钟超时（`timeout: 300000`）。命令完成后，读取 stderr：
```bash
cat "$TMPERR_PV"
```

逐字呈现完整输出：

```
CODEX SAYS (plan review — outside voice):
════════════════════════════════════════════════════════════
<full codex output, verbatim — do not truncate or summarize>
════════════════════════════════════════════════════════════
```

**错误处理：** 所有错误均为非阻塞——外部声音是信息性的。
- 认证失败（stderr 包含 "auth", "login", "unauthorized"）："Codex auth failed. Run `codex login` to authenticate." 回退到下面的子代理。
- 超时："Codex timed out after 5 minutes." 回退到下面的子代理。
- 空响应："Codex returned no response." 回退到下面的子代理。

**如果 `CODEX_MODE: not_installed` 或 `not_authed`（或 Codex 在运行时出错）：**

通过 Agent 工具分派。子代理有新的上下文——真正的独立性。以与 Codex 相同方式限制分派：以 5 分钟超时分派卡上限，以使"永不阻塞"也意味着"永不挂起"。

子代理提示：与上面的计划评审提示相同。

在 `OUTSIDE VOICE (Claude subagent):` 标题下呈现发现。

如果子代理失败或超时："Outside voice unavailable. Continuing to outputs."

（在 `CODEX_MODE: disabled` 上，您已经根据预检跳过了本章——不要到达这里。）

**跨模型张力：**

在呈现外部声音发现后，指出外部声音与前面章节评审发现不一致的任何地方。标记为：

```
CROSS-MODEL TENSION:
  [Topic]: Review said X. Outside voice says Y. [Present both perspectives neutrally.
  State what context you might be missing that would change the answer.]
```

**用户主权：** 不要自动将外部声音推荐并入计划。将每个张力点呈现给用户。用户决定。跨模型协议是一个强烈信号——如此呈现——但这不是行动的许可。你可以说明哪个论点你更有说服力，但你绝不能未经明确用户批准就应用变更。

对每个实质性张力点，使用 AskUserQuestion：

> "Cross-model disagreement on [topic]. The review found [X] but the outside voice argues [Y]. [One sentence on what context you might be missing.]"
>
> RECOMMENDATION: Choose [A or B] because [one-line reason explaining which argument is more compelling and why]. Completeness: A=X/10, B=Y/10.

选项：
- A) 接受外部声音的推荐（我将应用此变更）
- B) 保持当前方法（拒绝外部声音）
- C) 决定前进一步调查
- D) 添加到 TODOS.md 供以后处理

等待用户回复。不要因为你就同意外部声音而默认接受。如果用户选择 B，当前方法成立——不要再争辩。

如果没有张力点，注明："No cross-model tension — both reviewers agree."

**持久化结果：**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-plan-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```

替换：如果无发现 STATUS = "clean"，如果存在发现则为 "issues_source"。
如果 Codex 运行 SOURCE = "codex"，如果子代理运行则为 "claude"。

**清理：** 处理后运行 `rm -f "$TMPERR_PV"`（如果使用了 Codex）。

---

### Outside Voice Integration Rule

Outside Voice 发现是信息性的，直到用户明确批准每个发现。不要将外部声音推荐并入计划而不通过 AskUserQuestion 呈现每个发现并获得明确批准。即使你同意外部声音也适用此规则。跨模型共识是一个强烈信号——如此呈现——但用户做决定。

##实施后设计审计（如果检测到 UI 范围）
实施后，对实时站点运行 `/design-review` 以捕获只能在渲染输出上评估的视觉问题。

## 关键规则——如何提问
遵循上面前言中的 AskUserQuestion 格式。计划评审的额外规则：
* **一个问题 = 一次 AskUserQuestion 调用。** 永远不要将多个问题合并为一个问题。
* 具体地描述问题，引用文件和行号。
* 呈现 2-3 个选项，合理时包括"什么都不做"。
* 对每个选项：用一句话说明工作量、风险和维护负担。
* **将推理映射到我的工程偏好上面。** 将你的推荐与特定偏好连接的一句话。
* 用问题编号 + 选项字母标记（例如，"3A", "3B"）。
* **零发现：** 如果一个章节零发现，说明"没有问题，继续"并继续。否则，对每个发现使用 AskUserQuestion——一个"明显修复"的发现仍然是需要用户批准后才能进入计划的发现。

## 必需输出

### "NOT in scope" 部分
列出被考虑并明确推迟的工作，每项附一行理由。

### "What already exists" 部分
列出部分解决子问题的现有代码/流程，以及计划是否重用它们。

### "Dream state delta" 部分
这个计划让我们相对于 12 个月理想状态处于什么位置。

### 错误与救援注册表（来自第 2 章）
每个可能失败的方法、每个异常类、救援状态、救援动作、用户影响的完整表格。

### 故障模式注册表
```
  CODEPATH | FAILURE MODE | RESCUED? | TEST? | USER SEES? | LOGGED?
  ---------|--------------|----------|-------|------------|--------
```
任何行如果 RESCUED=N, TEST=N, USER SEES=Silent →**关键缺口**。

### TODOS.md 更新
将每个潜在的 TODO 呈现为单独的 AskUserQuestion。永远不要批量处理 TODO——每个问题一个。永远不要静默跳过此步骤。遵循 `.claude/skills/review/TODOS-format.md` 中的格式。

对每个 TODO，描述：
* **What：** 工作的一行描述。
* **Why：** 它解决的具体问题或它解锁的价值。
* **Pros：** 做这个工作你获得什么。
* **Cons：** 成本、复杂性或风险。
* **Context：** 足够的细节让 3 个月后接手的人理解动机、当前状态和从哪里开始。
* **Effort estimate：** S/M/L/XL（人类团队）→ 使用 CC+gstack：S→S, M→S, L→M, XL→L
* **Priority：** P1/P2/P3
* **Depends on / blocked by：** 任何先决条件或顺序约束。

然后呈现选项：**A)** 添加到 TODOS.md **B)** 跳过——不够有价值 **C)** 现在就构建在这个 PR 中而不是推迟。

### 范围扩展决策（仅扩展和选择性扩展）
对扩展和选择性扩展模式：扩展机会和愉悦项目在 0D 步骤中被浮现和决定（选择加入/采摘仪式）。决策持久化在 CEO 计划文档中。引用 CEO 计划获取完整记录。不要在这里重新浮现它们——列出已接受的扩展以保完整性：
* Accepted：{list items added to scope}
* Deferred：{list items sent to TODOS.md}
* Skipped：{list items rejected}

### 图表（强制性的，生成所有适用的）
1. 系统架构
2. 数据流（包括阴影路径）
3. 状态机
4. 错误流
5. 部署序列
6. 回滚流程图

### 过时图表审计
列出这个计划触及的文件中的所有 ASCII 图表。仍然准确吗？

## 实施任务
在结束此评审之前，将上述发现综合为一个平面列表的可构建行动的任务。每个任务源于一个特定的发现——没有填充。
发出 markdown 部分 **和** 写入一个 JSONL 工件，`/autoplan` 可以跨阶段聚合。

### Markdown 部分（始终发出）

```markdown
## Implementation Tasks
Synthesized from this review's findings. Each task derives from a specific
finding above. Run with Claude Code or Codex; checkbox as you ship.

- [ ] **T1 (P1, human: ~2h / CC: ~15min)** — <component> — <imperative title>
  - Surfaced by: <section name> — <specific finding text or line reference>
  - Files: <paths to touch>
  - Verify: <test command or manual check>
- [ ] **T2 (P2, human: ~30min / CC: ~5min)** — ...
```

规则：
- P1 阻塞发布；P2 应该在同一分支发布；P3 是后续 TODO。
- 如果一个发现没有产生可构建的任务，不要编造。
- 如果一个章节零发现，发出 `_No new tasks from <section>._`

### JSONL 工件（始终写入，即使零任务）

`/autoplan` 读取此文件以跨阶段聚合。使用 `jq -nc` 构建每一行，以使包含引号、换行符或反斜杠的标题和源发现干净地序列化——永远不要使用手工滚动 `echo` / `printf`。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
TASKS_DIR="${HOME}/.gstack/projects/${SLUG:-unknown}"
mkdir -p "$TASKS_DIR"
TASKS_FILE="$TASKS_DIR/tasks-ceo-review-$(date +%Y%m%d-%H%M%S).jsonl"
COMMIT=$(git rev-parse HEAD 2>/dev/null || echo unknown)
BRANCH=$(git branch --show-current 2>/dev/null || echo unknown)
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"

# Repeat ONE jq invocation per task identified during this review.
# Substitute the placeholders inline with shell variables you set per task:
#   TASK_ID (T1, T2, ...), PRIORITY (P1/P2/P3), COMPONENT, TITLE,
#   SOURCE_FINDING, EFFORT_HUMAN, EFFORT_CC, FILES_JSON (a JSON array literal
#   like '["browse/src/sanitize.ts","browse/src/server.ts"]').
jq -nc \
  --arg phase 'ceo-review' \
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

如果未安装 `jq`，回退到跳过 JSONL 写入并警告用户安装 jq 以进行 autoplan 聚合。永远不要手工滚动 JSONL。

如果本次评审识别出零任务，仍然 touch JSONL 文件（`: > "$TASKS_FILE"`），以便聚合器看到该阶段本次运行产生了输出（空文件意味着"运行了，无发现"——与"未运行"不同）。

### 完成摘要
```
  +====================================================================+
  |            MEGA PLAN REVIEW — COMPLETION SUMMARY                   |
  +====================================================================+
  | Mode selected        | EXPANSION / SELECTIVE / HOLD / REDUCTION     |
  | System Audit         | [key findings]                              |
  | Step 0               | [mode + key decisions]                      |
  | Section 1  (Arch)    | ___ issues found                            |
  | Section 2  (Errors)  | ___ error paths mapped, ___ GAPS            |
  | Section 3  (Security)| ___ issues found, ___ High severity         |
  | Section 4  (Data/UX) | ___ edge cases mapped, ___ unhandled        |
  | Section 5  (Quality) | ___ issues found                            |
  | Section 6  (Tests)   | Diagram produced, ___ gaps                  |
  | Section 7  (Perf)    | ___ issues found                            |
  | Section 8  (Observ)  | ___ gaps found                              |
  | Section 9  (Deploy)  | ___ risks flagged                           |
  | Section 10 (Future)  | Reversibility: _/5, debt items: ___         |
  | Section 11 (Design)  | ___ issues / SKIPPED (no UI scope)          |
  +--------------------------------------------------------------------+
  | NOT in scope         | written (___ items)                          |
  | What already exists  | written                                     |
  | Dream state delta    | written                                     |
  | Error/rescue registry| ___ methods, ___ CRITICAL GAPS              |
  | Failure modes        | ___ total, ___ CRITICAL GAPS                |
  | TODOS.md updates     | ___ items proposed                          |
  | Scope proposals      | ___ proposed, ___ accepted (EXP + SEL)      |
  | CEO plan             | written / skipped (HOLD/REDUCTION)           |
  | Outside voice        | ran (codex/claude) / skipped                 |
  | Lake Score           | X/Y recommendations chose complete option   |
  | Diagrams produced    | ___ (list types)                            |
  | Stale diagrams found | ___                                         |
  | Unresolved decisions | ___ (listed below)                          |
  +====================================================================+
```

### 未解决的决定
如果任何 AskUserQuestion 未获回答，在此注明。永远不要静默默认。

## Handoff Note 清理

在生成完成摘要后，清理此分支的任何交接说明——评审已完成，不再需要上下文。

```bash
setopt +o nomatch 2>/dev/null || true  # zsh compat
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
rm -f ~/.gstack/projects/$SLUG/*-$BRANCH-ceo-handoff-*.md 2>/dev/null || true
```

## 评审日志

生成上面的完成摘要后，持久化评审结果。

**计划模式例外——始终运行：** 此命令将评审元数据写入 `~/.gstack/`（用户配置目录，非项目文件）。skill 前言已经写入 `~/.gstack/sessions/` 和 `~/.gstack/analytics/` ——这是相同模式。评审仪表板依赖此数据。跳过此命令损坏 /ship 中的评审准备仪表板。

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-ceo-review","timestamp":"TIMESTAMP","status":"STATUS","unresolved":N,"critical_gaps":N,"mode":"MODE","scope_proposed":N,"scope_accepted":N,"scope_deferred":N,"commit":"COMMIT"}'
~/.claude/skills/gstack/bin/gstack-decision-log '{"decision":"CEO review (MODE): SCOPE_SUMMARY","rationale":"VERDICT","scope":"branch","source":"skill","confidence":8}' 2>/dev/null || true
```

第二个命令将接受的范围记录为持久的跨会话决策，以便下次会话看到已确定的（和为什么）而无重新争论。它写入 `~/.gstack/`（与 review-log 相同模式），非交互式的，尽力而为的（`|| true` ——从不阻塞评审）。替换 `SCOPE_SUMMARY`（例如扩展"accepted 4 of 6 proposals"或保持"held scope"/"cut 3 items"的 HOLD/REDUCTION）和 `VERDICT`（摘要中的一行裁决）。

运行此命令之前，从你刚才生成的完成摘要替换占位符值：
- **TIMESTAMP**：当前 ISO 8601 日期时间（例如，2026-03-16T14:30:00）
- **STATUS**：如果 0 未解决决定 AND 0 关键缺口为"clean"；否则为"issues_open"
- **unresolved**：摘要中"Unresolved decisions"中的数量
- **critical_gaps**：摘要中"Failure modes: ___ CRITICAL GAPS"中的数量
- **MODE**：用户选择的模式（SCOPE_EXPANSION / SELECTIVE_EXPANSION / HOLD_SCOPE / SCOPE_REDUCTION）
- **scope_proposed**：摘要中"Scope proposals: ___ proposed"的数量（HOLD/REDUCTION 为 0）
- **scope_accepted**：摘要中"Scope proposals: ___ accepted"的数量（HOLD/REDUCTION 为 0）
- **scope_deferred**：从范围决定到 TODOS.md 的推迟项目数量（HOLD/REDUCTION 为 0）
- **COMMIT**：`git rev-parse --short HEAD` 的输出

## 评审准备仪表板

完成评审后，读取评审日志和配置以显示仪表板。

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

解析输出。查找每个 skill 的最新条目（plan-ceo-review, plan-eng-review, review, plan-design-review, design-review-lite, adversarial-review, codex-review, codex-plan-review）。忽略时间戳超过 7 天的条目。对 Eng Review 行，显示 `review`（diff-scoped pre-landing review）和 `plan-eng-review`（plan-stage architecture review）中最新者。追加"(DIFF)"或"(PLAN)"以区分。对 Adversarial 行，显示 `adversarial-review`（new auto-scaled）和 `codex-review`（legacy）中最新者。对 Design Review，显示 `plan-design-review`（full visual audit）和 `design-review-lite`（code-level check）中最新者。追加"(FULL)"或"(LITE)"以区分。对 Outside Voice 行，显示最新的 `codex-plan-review` 条目——这捕获来自 /plan-ceo-review 和 /plan-eng-review 的外部声音。

**源属性：** 如果一个 skill 的最新条目有 `"via"` 字段，将其追加到括号中的状态标签。例如：`plan-eng-review` 带 `via:"autoplan"` 显示为"CLEAR (PLAN via /autoplan)"。`review` 带 `via:"ship"` 显示为"CLEAR (DIFF via /ship)"。无 `via` 字段的条目照常显示为"CLEAR (PLAN)"或"CLEAR (DIFF)"。

注意：`autoplan-voices` 和 `design-outside-voices` 条目仅用于审计跟踪（跨模型共识分析的取证数据）。它们不出现在仪表板中，也不被任何消费者检查。

显示：

```
+====================================================================+
| REVIEW READINESS DASHBOARD |
+================================================================----+
| Review | Runs | Last Run | Status | Required |
|-----------------|------|---------------------|-----------|----------|
| Eng Review | 1 | 2026-03-16 15:00 | CLEAR | YES |
| CEO Review | 0 | — | — | no |
| Design Review | 0 | — | — | no |
| Adversarial | 0 | — | — | no |
| Outside Voice | 0 | — | — | no |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed |
+====================================================================+
```

**评审层级：**
- **Eng Review（默认必需）：** 唯一阻塞发布的评审。覆盖架构、代码质量、测试、性能。可使用 `gstack-config set skip_eng_review true` 全局禁用（"别烦我"设置）。
- **CEO Review（可选）：** 自行判断。大的产品/业务变更、新功能或范围决策时推荐。Bug 修复、重构、基础设施和清理时跳过。
- **Design Review（可选）：** 自行判断。UI/UX 变更时推荐。仅后端、基础设施或仅 prompt 变更时跳过。
- **Adversarial Review（自动）：** 每次评审始终开启。每个 diff 获取 Claude 对抗子代理和 Codex 对抗挑战。大 diff（200+ 行）额外获取 Codex 有结构评审与 P1 门控。无需配置。
- **Outside Voice（可选）：** 从不同 AI 模型的独立计划评审。在 /plan-ceo-review 和 /plan-eng-review 中所有评审章节完成后提供。如果 Codex 不可用则回退到 Claude 子代理。从不阻塞发布。

**裁决逻辑：**
- **CLEARED**：Eng Review 在 7 天内从 `review` 或 `plan-eng-review` 有 >= 1 条目且状态为"clean"（或 `skip_eng_review` 为 `true`）
- **NOT CLEARED**：Eng Review 缺失、过时（>7 天）或有未解决开放问题
- CEO、Design 和 Codex 评审仅用于上下文，从不阻塞发布
- 如果 `skip_eng_review` 配置为 `true`，Eng Review 显示"SKIPPED (global)"且裁决为 CLEARED

**过时检测：** 显示仪表板后，检查是否有现有评审可能过时：
- 从 bash 输出的 `---HEAD---` 部分解析当前 HEAD 提交哈希
- 对于有 `commit` 字段的每个评审条目：将其与当前 HEAD 比较。如果不同，计算已过去提交数：`git rev-list --count STORED_COMMIT..HEAD`。显示："Note: {skill} review from {date} may be stale — {N} commits since review"
- 对无 `commit` 字段的条目（遗留条目）：显示"Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection"
- 如果所有评审与当前 HEAD 匹配，不过时提示。

## 计划文件评审报告

在对话输出中显示评审准备仪表板后，还更新**计划文件**本身以便计划任何读者都能看到评审状态。

### 检测计划文件

1. 检查此次对话中是否有活动计划文件（主机在系统消息中提供计划文件路径——在对话上下文中查找计划文件引用）。
2. 如果未找到，静默跳过本章节——并非每个评审都在计划审查模式中运行。

### 生成报告

从上面的评审准备仪表板步骤中读取你已经有的评审日志输出。解析每个 JSONL 条目。每个 skill 记录不同字段：

- **plan-ceo-review**：`status`, `unresolved`, `critical_gaps`, `mode`, `scope_proposed`, `scope_accepted`, `scope_deferred`, `commit`
  → 发现："{scope_proposed} proposals, {scope_accepted} accepted, {scope_deferred} deferred"
  → 如果 scope 字段为 0 或缺失（HOLD/REDUCTION 模式）："mode: {mode}, {critical_gaps} critical gaps"
- **plan-eng-review**：`status`, `unresolved`, `critical_gaps`, `issues_found`, `mode`, `commit`
  → 发现："{issues_found} issues, {critical_gaps} critical gaps"
- **plan-design-review**：`status`, `initial_score`, `overall_score`, `unresolved`, `decisions_made`, `commit`
  → 发现："score: {initial_score}/10 → {overall_score}/10, {decisions_made} decisions"
- **plan-devex-review**：`status`, `initial_score`, `overall_score`, `product_type`, `tthw_current`, `tthw_target`, `mode`, `persona`, `competitive_tier`, `unresolved`, `commit`
  → 发现："score: {initial_score}/10 → {overall_score}/10, TTHW: {tthw_current} → {tthw_target}"
- **devex-review**：`status`, `overall_score`, `product_type`, `tthw_measured`, `dimensions_tested`, `dimensions_inferred`, `boomerang`, `commit`
  → 发现："score: {overall_score}/10, TTHW: {tthw_measured}, {dimensions_tested} tested/{dimensions_inferred} inferred"
- **codex-review**：`status`, `gate`, `findings`, `findings_fixed`
  → 发现："{findings} findings, {findings_fixed}/{findings} fixed"

Findings 列所需的所有字段现在都存在于 JSONL 条目中。
对于你刚刚完成的评审，你可以使用自己完成摘要的更丰富细节。对于先前的评审，直接使用 JSONL 字段——它们包含所有必需数据。

生成此 markdown 表格：

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

表格下方，添加这些行。**CODEX** 和 **CROSS-MODEL** 是可选的（空时省略）；**VERDICT** 始终存在：
- **CODEX：**（仅在 codex-review 运行时）——一行 codex 修复摘要
- **CROSS-MODEL：**（仅在两个 Claude 和 Codex 评审都存在时）——重叠分析
- **VERDICT：**列出 CLEAR 的评审（例如，"CEO + ENG CLEARED — ready to implement"）。
  如果 Eng Review 不是 CLEAR 且未全局跳过，追加"eng review required"。

**未解决决定状态（强制性——从不省略；报告最终非空白行）。** 在 VERDICT 之后，结束报告（`## GSTACK REVIEW REPORT` 标题下的内容——一个粗体标签，永远不是新 `##` 标题；不受"空时省略"规则约束）只用一个：确切的未加粗行 `NO UNRESOLVED DECISIONS`（一个加粗的**不算**），或 `**UNRESOLVED DECISIONS:**` 标题 + 每个开放项目一个项目符号（最后项目符号 = 最终行；仅当 N > 0 时添加 `+ N unresolved from prior reviews`）。这避免双重计数：从上下文列出**本次**评审的开放项目；对先前评审，对每个 skill 在你 DROP 当前 skill 行后最近 fresh 行的 `unresolved` 求和（仪表板 7 天窗口）；仅当两者都为零时发出哨兵。

### 写入计划文件

**计划模式例外——始终运行：** 这写入计划文件，这是你在计划审查模式中唯一允许编辑的文件。计划文件评审报告是计划实时状态的一部分。

报告必须始终是计划文件的最后部分——永远不在中间。使用一次删除然后追加流程：

1. 读取计划文件（Read 工具）以查看其完整当前内容。在读取输出中查找文件任何地方的 `## GSTACK REVIEW REPORT` 标题。
2. 如果找到，使用 Edit 工具删除整个现有章节。从 `## GSTACK REVIEW REPORT` 匹配到下一个 `##` 标题或文件结束，以先到者为止。替换为空字符串。这适用于章节当前所在位置——中间删除是故意的，不是特殊情况。如果 Edit 失败（例如，并发编辑更改了内容），重新读取计划文件并重试一次。
3. 删除后（或如果无章节存在跳过），在文件末尾追加新的 `## GSTACK REVIEW REPORT` 章节。使用 Edit 工具匹配文件当前最后一个段落后在其后添加章节，或使用 Write 重新发出整个文件并将章节放在末尾。
4. 用 Read 工具验证 `## GSTACK REVIEW REPORT` 在继续之前是否是文件中最后一个 `##` 标题。如果不是，重复步骤 2-3 一次。

不要在原地替换章节。"mid-file 替换"路径是先前版本允许报告留在文件中间的原因——当旧报告已经住在那里时——然后用户看到评审报告不在底部的计划并（正确地）拒绝它。

## 下一步——评审链接

显示评审准备仪表板后，根据此 CEO 发现的内容推荐下一个评审。读取仪表板输出以查看哪些评审已经运行以及它们是否过时。

**如果 eng review 未全局跳过，推荐 /plan-eng-review**——检查仪表板输出中的 `skip_eng_review`。如果为 `true`，eng review 已选择退出——不要推荐。否则，eng review 是必需的发布门控。如果此 CEO review 扩展了范围、改变了架构方向或接受了范围扩展，强调需要新的 eng review。如果仪表板中已有 eng review 但提交哈希显示它早于此 CEO review，注意它可能过时并应重新运行。

**如果检测到 UI 范围，推荐 /plan-design-review**——具体而言，如果第 11 章（设计与 UX 评审）**未**被跳过，或如果接受的扩展包括 UI 功能。如果现有设计 review 过时（提交哈希漂移），注意。在 SCOPE REDUCTION 模式中，跳过此推荐——设计评审不太可能与范围裁剪相关。

**如果两者都需要，先推荐 eng review**（必需的门控），然后 design review。

使用 AskUserQuestion 呈现下一步。只包括适用选项：
- **A)** 接下来运行 /plan-eng-review（必需的门控）
- **B)** 接下来运行 /plan-design-review（仅在 UI 范围检测到时）
- **C)** 跳过——我将手动处理评审

## docs/designs 提升（仅扩展和选择性扩展）

在评审结束时，如果愿景产生了引人注目的功能方向，提供将 CEO 计划提升到项目仓库。AskUserQuestion：

"The vision from this review produced {N} accepted scope expansions. Want to promote it to a design doc in the repo?"
- **A)** 提升到 `docs/designs/{FEATURE}.md`（提交到仓库，对团队可见）
- **B)** 仅保留在 `~/.gstack/projects/` 中（本地，个人参考）
- **C)** 跳过

如果提升，将 CEO 计划内容复制到 `docs/designs/{FEATURE}.md`（如果需要创建目录）并将原始 CEO 计划中 `status` 字段从 `ACTIVE` 更新为 `PROMOTED`。

## 格式化规则
* NUMBER 问题（1, 2, 3...）和 LETTERS 选项（A, B, C...）。
* 用 NUMBER + LETTER 标记（例如，"3A", "3B"）。
* 每个选项最多一句话。
* 每个章节后暂停并等待反馈。
* 使用**CRITICAL GAP** / **WARNING** / **OK** 以提高可扫性。

## 捕获学习

如果你在此次会话中发现了一个非显而易见的模式、陷阱或架构洞察，记录它供未来会话使用：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"plan-ceo-review","type":"TYPE","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"SOURCE","files":["path/to/relevant/file"]}'
```

**Types：** `pattern`（可重用方法）、`pitfall`（不要做什么）、`preference`（用户陈述）、`architecture`（结构决策）、`tool`（库/框架洞察）、`operational`（项目环境/CLI/工作流知识）。

**Sources：** `observed`（你在代码中发现）、`user-stated`（用户告诉你）、`inferred`（AI 推论）、`cross-model`（Claude 和 Codex 都同意）。

**Confidence：** 1-10。要诚实。你在代码中验证的观察到的模式是 8-9。你不确定的推论是 4-5。用户明确陈述的偏好是 10。

**files：** 包括此学习引用的具体文件路径。这使过时检测成为可能：如果这些文件后来被删除，学习可以被标记。

**只记录真正的发现。** 不要记录明显的东西。不要记录用户已经知道的东西。一个很好的测试：这个洞察会在未来会话中节省时间吗？如果是，记录它。

## 大脑校准回写（第 2 阶段 / 门控）

当 skill 做出一个值得跟踪的类型预测（范围决策、TTHW 目标、架构赌注、wedge 承诺）时，它可以向大脑写入一个 `kind=bet` 的看法，以便随时间建立校准画像。

**在两个东西上门控：**
1. 活动端点的大脑信任策略是 `personal`（通过 `~/.claude/skills/gstack/bin/gstack-config get brain_trust_policy@<endpoint-hash>` 检查）。共享大脑跳过回写以避免污染团队校准。
2. 特性标志 `BRAIN_CALIBRATION_WRITEBACK` 已设置（今天：false；当上游 gbrain v0.42+ 发布 `takes_add` MCP op 时翻转为 true）。

当两个门控都通过时，回写路径使用 `mcp__gbrain__takes_add` 以权重 0.8（每个 SKILL_CALIBRATION_WEIGHTS）记录一个看法。
如果 MCP op 不可用，回退到 `mcp__gbrain__put_page` 配合 gstack:takes 围栏块（已记录但更丑的路径）。

必需的 take frontmatter 形状：
```yaml
kind: bet
holder: <user identity from whoami>
claim: <one-line prediction the skill is making>
weight: 0.8
since_date: <today's date>
expected_resolution: <date in 1-3 months depending on skill>
source_skill: plan-ceo-review
```

写入后，使受影响的摘要无效，以便下次预检反映新状态：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate product --project "$SLUG" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate goals --project "$SLUG" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate competitive-intel --project "$SLUG" 2>/dev/null || true
```

## 大脑缓存后台刷新

在 skill 工作完成后（和遥测已记录后），启动任何接近其 TTL 的缓存摘要的后台刷新。这是非阻塞的——用户不等待。下次调用受益于温缓存。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
(~/.claude/skills/gstack/bin/gstack-brain-cache refresh --project "$SLUG" 2>/dev/null &) || true
```

## 模式快速参考
```
  ┌────────────────────────────────────────────────────────────────────────────────┐
  │ MODE COMPARISON │
  ├─────────────┬──────────────┬──────────────┬──────────────┬────────────────────┤
  │ │ EXPANSION │ SELECTIVE │ HOLD SCOPE │ REDUCTION │
  ├─────────────┼──────────────┼──────────────┼──────────────┼────────────────────┤
  │ Scope │ Push UP │ Hold + offer │ Maintain │ Push DOWN │
  │ │ (opt-in) │ │ │ │
  │ Recommend │ Enthusiastic │ Neutral │ N/A │ N/A │
  │ posture │ │ │ │ │
  │ 10x check │ Mandatory │ Surface as │ Optional │ Skip │
  │ │ │ cherry-pick │ │ │
  │ Platonic │ Yes │ No │ No │ No │
  │ ideal │ │ │ │ │
  │ Delight │ Opt-in │ Cherry-pick │ Note if seen │ Skip │
  │ opps │ ceremony │ ceremony │ │ │
  │ Complexity │ "Is it big │ "Is it right │ "Is it too │ "Is it the bare │
  │ question │ enough?" │ + what else │ complex?" │ minimum?" │
  │ │ │ is tempting"│ │ │
  │ Taste │ Yes │ Yes │ No │ No │
  │ calibration │ │ │ │ │
  │ Temporal │ Full (hr 1-6)│ Full (hr 1-6)│ Key decisions│ Skip │
  │ interrogate │ │ │ only │ │
  │ Observ. │ "Joy to │ "Joy to │ "Can we │ "Can we see if │
  │ standard │ operate" │ operate" │ debug it?" │ it's broken?" │
  │ Deploy │ Infra as │ Safe deploy │ Safe deploy │ Simplest possible │
  │ standard │ feature scope│ + cherry-pick│ + rollback │ deploy │
  │ │ │ risk check │ │ │
  │ Error map │ Full + chaos │ Full + chaos │ Full │ Critical paths │
  │ │ scenarios │ for accepted │ │ only │
  │ CEO plan │ Written │ Written │ Skipped │ Skipped │
  │ Phase 2/3 │ Map accepted │ Map accepted │ Note it │ Skip │
  │ planning │ │ cherry-picks │ │ │
  │ Design │ "Inevitable" │ If UI scope │ If UI scope │ Skip │
  │ (Sec 11) │ UI review │ detected │ detected │ │
  └─────────────┴──────────────┴──────────────┴──────────────┴────────────────────┘
```
