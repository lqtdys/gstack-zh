<!-- AUTO-GENERATED from review-sections.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Review Sections (after scope is agreed)

**反跳过规则：** 绝不压缩、缩写或跳过任何审查环节（1-4），无论计划类型（战略、规格、代码、基础设施）。此技能中的每个环节都有其存在的理由。"这是一份战略文档，所以实施部分不适用"永远是错误的——实施细节正是战略出问题的地方。如果某个环节确实没有发现任何问题，说"未发现问题"然后继续——但你必须对其进行评估。

**反捷径条款：** 计划文件是交互式审查的输出，而不是它的替代。将所有发现写入一个计划文件然后调用ExitPlanMode而不触发AskUserQuestion，正是2026年5月转录bug的精确失败模式——模型进行了探索，发现了问题，然后将它们转储到一个交付物中，而不是引导用户解决这些问题。如果**任何审查环节中有任何非平凡的发现**，从发现到ExitPlanMode的路径**必须经过AskUserQuestion**。每个环节零发现才是唯一可以绕过AskUserQuestion直接到达ExitPlanMode的路径。如果你发现自己想在提问之前写一个包含发现的计划，现在就停下来调用AskUserQuestion——那就是这个bug，识别它。

## Prior Learnings

从前序会话中搜索相关经验：

```bash
_CROSS_PROJ=$(~/.claude/skills/gstack/bin/gstack-config get cross_project_learnings 2>/dev/null || echo "unset")
echo "CROSS_PROJECT: $_CROSS_PROJ"
if [ "$_CROSS_PROJ" = "true" ]; then
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 --cross-project 2>/dev/null || true
else
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 2>/dev/null || true
fi
```

如果`CROSS_PROJECT`为`unset`（首次使用）：使用AskUserQuestion：

> gstack可以从此机器上的其他项目中搜索经验，以找到可能
> 适用的模式。这保持在本地（数据不会离开您的机器）。
> 推荐独立开发者使用。如果您在多个客户端代码库上工作，
> 且交叉污染是一个问题，则可跳过。

选项：
- A) 启用跨项目经验推荐）
- B) 保持经验仅限项目范围

如果A：运行`~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings true`
如果B：运行`~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings false`

然后用适当的标志重新运行搜索。

如果发现经验，将其纳入你的分析。当审查发现与过去的学习匹配时，显示：

**"已应用先验经验：[key]（置信度 N/10，来自 [date]）"**

这使不断增强的能力可视化。用户应该看到gstack正在对他们的代码库变得越来越聪明。

### 1. Architecture review
评估：
* 整体系统设计和组件边界。
* 依赖图和耦合问题。
* 数据流模式和潜在瓶颈。
* 扩展特征和单点故障。
* 安全架构（身份验证、数据访问、API边界）。
* 关键流程是否值得在计划或代码注释中使用ASCII图表。
* 对于每个新的代码路径或集成点，描述一个真实的生产故障场景，以及计划是否考虑了它。
* **分布式架构：** 如果引入了一个新构件（二进制文件、包、容器），它如何被构建、发布和更新？CI/CD流水线是计划的一部分还是被推迟了？

对于此环节中发现的每个问题，单独调用AskUserQuestion。每个问题一次调用。提供选项，陈述你的建议，解释**原因**。不要将多个问题打包到一个AskUserQuestion中。使用前言的AskUserQuestion格式部分。AskUserQuestion调用是tool_use，不是散文——直接调用工具。

**停止。** 在用户回复之前，不要进入下一个审查环节，不要编辑计划文件添加修复方案，也不要调用ExitPlanMode。一个有"明显修复方案"的问题仍然是问题，在写入计划之前仍然需要用户显式批准。通过ToolSearch加载AskUserQuestionSchema然后将建议写成聊天散文，是这个门控旨在防止的失败模式。

## Confidence Calibration

每个发现**必须**包含置信度评分（1-10）：

| 评分 | 含义 | 显示规则 |
|------|------|---------|
| 9-10 | 通过阅读特定代码验证。已证明具体bug或利用方式。 | 正常显示 |
| 7-8 | 高置信度模式匹配。极有可能是正确的。 | 正常显示 |
| 5-6 | 中等。可能是误报。 | 带警告显示："中等置信度，请验证这确实是问题" |
| 3-4 | 低置信度。模式可疑但可能没问题。 | 从主报告中抑制。仅包含在附录中。 |
| 1-2 | 猜测。 | 仅在严重性为P0时报告。 |

**发现格式：**

\`[SEVERITY] (confidence: N/10) file:line — description\`

示例：
\`[P1] (confidence: 9/10) app/models/user.rb:42 — SQL injection via string interpolation in where clause\`
\`[P2] (confidence: 5/10) app/controllers/api/v1/users_controller.rb:18 — Possible N+1 query, verify with production logs\`

### Pre-emit verification gate (#1539 — 消灭"字段不存在"FP类)

在将任何发现提升到报告之前，门控要求：

1. **引用激发该发现的特定代码行** — 文件:行号加上
   触发它的代码行的逐字文本。如果发现是"模型Y上没有字段X"，引用类Y将
   字段纳入的行。如果"dict.get()可能返回None"，引用dict初始化。
   如果"在A和B之间存在竞争条件"，引用A和B。

2. **如果你无法引用激发该发现的行，则该发现是未验证的。**
   强制其置信度为4-5（从主报告中抑制）。但它仍然进入附录，
   以便审查者可以审计校准，但用户**不会**在关键过程输出中看到它。不要用编造的
   推测性置信度7+来绕过这个限制——这会破坏门控。

**框架-元提示：** 当符号由框架元类、描述符、ORM Meta内部类或迁移历史生成时（Django的`Meta`、Rails的`has_many`/`scope`、SQLAlchemy的`relationship`/`Column`、TypeORM装饰器、Sequelize的`init`/`belongsTo`、Prisma生成的客户端），引用元构造（`Meta`块、迁移、装饰器、模式文件）而不是期望在类体中出现字面名称。验证是"我阅读了创建这个符号的源"，而不是"我用grep搜索名称但没找到。"更深入的框架感知验证（模型内省、迁移历史感知检查、ORM方言检测）故意不在较轻门控的范围内——参见延期的`~/.gstack-dev/plans/1539-framework-aware-review.md`设计文档。

门控消灭的FP类（在Django Sprint 2.5 #1539中测量）：

| FP类 | 门控如何捕获它 |
|------|-------------|
| "模型上不存在字段" | 需要引用模型类体或Meta；不存在变得明显 |
| "dict.get()可能是None" | 需要引用dict初始化（例如，Django表单的`cleaned_data`是`{}`初始化的） |
| "save()可能丢失字段" | 需要引用ORM签名或模型定义 |
| "update_fields可能错过X" | 需要引用字段集合；如果X不存在，FP是不言自明的 |

**校准学习：** 如果你报告一个置信度 < 7的发现，而用户确认它确实是一个问题，那就是校准事件。你的初始置信度太低了。记录修正后的模式作为学习，以便未来的审查能以更高的置信度捕获它。

### 2. Code quality review
评估：
* 代码组织和模块结构。
* DRY违规——在这里要严格。
* 错误处理模式和缺失的边缘情况（明确指出这些）。
* 技术债务热点。
* 与我的偏好相比过度设计或设计不足的区域。
* 被修改文件中现有的ASCII图表——在此更改后它们是否仍然准确？

对于此环节中发现的每个问题，单独调用AskUserQuestion。每个问题一次调用。提供选项，陈述你的建议，解释**原因**。不要将多个问题打包到一个AskUserQuestion中。使用前言的AskUserQuestion格式部分。AskUserQuestion调用是tool_use，不是散文——直接调用工具。

**停止。** 在用户回复之前，不要进入下一个审查环节，不要编辑计划文件添加修复方案，也不要调用ExitPlanMode。一个有"明显修复方案"的问题仍然是问题，在写入计划之前仍然需要用户显式批准。通过ToolSearch加载AskUserQuestionSchema然后将建议写成聊天散文，是这个门控旨在防止的失败模式。

### 3. Test review

100%覆盖率是目标。评估计划中的每个代码路径，并确保计划包含每个代码路径的测试。如果计划缺少测试，添加它们——计划应该足够完整，以便从一开始就包含完整的测试覆盖率。

### Test Framework Detection

在分析覆盖率之前，检测项目的测试框架：

1. **阅读CLAUDE.md** —— 寻找一个`## Testing`部分，其中包含测试命令和框架名称。如果找到，将其作为权威来源。
2. **如果CLAUDE.md没有测试部分，自动检测：**

```bash
setopt +o nomatch 2>/dev/null || true  # zsh compat
# Detect project runtime
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
# Check for existing test infrastructure
ls jest.config.* vitest.config.* playwright.config.* cypress.config.* .rspec pytest.ini phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
```

3. **如果未检测到框架：** 仍然生成覆盖率图表，但跳过测试生成。

**Step 1. Trace every codepath in the plan:**

阅读计划文件。对于描述的每个新功能、服务、端点或组件，追踪数据将如何流过代码——不只是列出计划的功能，而是实际跟踪计划的执行：

1. **Read the plan.** 对于每个计划组件，理解它做什么以及它如何连接到现有代码。
2. **Trace data flow.** 从每个入口点（路由处理程序、导出的函数、事件监听器、组件渲染）开始，通过每个分支追踪数据：
   - 输入来自哪里？（请求参数、属性、数据库、API调用）
   - 什么转换它？（验证、映射、计算）
   - 它到哪里去？（数据库写入、API响应、渲染输出、副作用）
   - 每一步可能出什么问题？（null/undefined、无效输入、网络故障、空集合）
3. **Diagram the execution.** 对于每个修改的文件，绘制一个ASCII图表显示：
   - 添加或修改的每个函数/方法
   - 每个条件分支（if/else、switch、三元运算符、卫语句、提前返回）
   - 每个错误路径（try/catch、rescue、错误边界、回退）
   - 对另一个函数的每个调用（追踪进去——它有未测试的分支吗？）
   - 每个边界：空输入怎么办？空数组？无效类型？

这是关键步骤——你正在构建一张地图，每条代码线都可能基于输入以不同方式执行。图中的每个分支都需要一个测试。

**Step 2. Map user flows, interactions, and error states:**

代码覆盖率不够——你需要覆盖真实用户与更改代码交互的方式。对于每个更改的功能，通过以下思考：

- **User flows:** 什么序列的操作使用户接触这段代码？映射完整的旅程（例如，"用户点击'支付' → 表单验证 → API调用 → 成功/失败屏幕"）。旅程中的每一步都需要一个测试。
- **Interaction edge cases:** 当用户做意外的事情时会发生什么？
  - 双击/快速重新提交
  - 操作中途导航离开（后退按钮、关闭标签、点击另一个链接）
  - 使用过时数据提交（页面保持打开30分钟，会话已过期）
  - 慢连接（API需要10秒——用户看到什么？）
  - 并发操作（两个标签，同一个表单）
- **Error states the user can see:** 对于代码处理的每个错误，用户实际体验的是什么？
  - 有明确的错误消息还是静默失败？
  - 用户可以恢复（重试、返回、修正输入）还是卡住了？
  - 没有网络？API返回500？服务器返回无效数据？
- **Empty/zero/boundary states:** UI显示零结果是什么？10,000个结果？单个字符输入？最大长度输入？

将这些添加到图表中，与代码分支并列。没有测试的用户流就像未测试的if/else一样是一个缺口。


**Step 3. Check each branch against existing tests:**

逐个分支地遍历你的图表——代码路径和用户流程。对于每个分支，搜索一个对其进行测试的测试：
- 函数`processPayment()` → 查找`billing.test.ts`、`billing.spec.ts`、`test/billing_test.rb`
- 一个if/else → 查找同时覆盖true AND false路径的测试
- 一个错误处理器 → 查找触发该特定错误条件的测试
- 对`helperFn()`的调用，它有自己的分支 → 那些分支也需要测试
- 一个用户流 → 查找遍历整个旅程的集成或E2E测试
- 一个交互边缘情况 → 查找模拟意外动作的测试

质量评分标准：
- ★★★ 测试行为及边缘情况和错误路径
- ★★   测试正确行为，仅happy path
- ★    冒烟测试/存在性检查/琐碎断言（例如，"它渲染了"，"它不会抛出"）

### E2E Test Decision Matrix

在检查每个分支时，还要确定单元测试还是E2E/集成测试是正确的工具：

**推荐E2E（在图表中标记为[→E2E]）：**
- 跨越3+个组件/服务的常见用户流程（例如，注册 → 验证邮箱 → 首次登录）
- 模拟隐藏真实故障的集成点（例如，API → 队列 → worker → DB）
- 身份验证/支付/数据销毁流程 —— 太重要了不能仅信任单元测试

**推荐EVAL（在图表中标记为[→EVAL]）：**
- 需要质量评估的关键LLM调用（例如，prompt更改 → 测试输出仍满足质量标准）
- 对prompt模板、系统指令或工具定义的更改

**坚持使用单元测试：**
- 具有清晰输入/输出的纯函数
- 没有副作用的内部辅助函数
- 单个函数的边缘情况（null输入、空数组）
- 不面向客户的模糊/罕见流程

### REGRESSION RULE (mandatory)

**铁规则：** 当覆盖率审计识别出一个回归 —— 以前有效但diff破坏的代码 —— 计划中添加一个回归测试作为关键要求。不使用AskUserQuestion。不跳过。回归是最高优先级的测试，因为它们证明某些东西被破坏了。

回归是当：
- diff修改了现有行为（不是新代码）
- 现有测试套件（如果有）未覆盖更改的路径
- 更改为现有调用者引入了新的失败模式

当不确定一个更改是否是回归时，偏向于编写测试。

**Step 4. Output ASCII coverage diagram:**

在同一图表中包括代码路径和用户流。标记适合E2E和评估的路径：

```
CODE PATHS                                            USER FLOWS
[+] src/services/billing.ts                           [+] Payment checkout
  ├── processPayment()                                  ├── [★★★ TESTED] Complete purchase — checkout.e2e.ts:15
  │   ├── [★★★ TESTED] happy + declined + timeout      ├── [GAP] [→E2E] Double-click submit
  │   ├── [GAP]         Network timeout                 └── [GAP]        Navigate away mid-payment
  │   └── [GAP]         Invalid currency
  └── refundPayment()                                 [+] Error states
      ├── [★★  TESTED] Full refund — :89                ├── [★★  TESTED] Card declined message
      └── [★   TESTED] Partial (non-throw only) — :101  └── [GAP]        Network timeout UX

LLM integration: [GAP] [→EVAL] Prompt template change — needs eval test

COVERAGE: 5/13 paths tested (38%)  |  Code paths: 3/5 (60%)  |  User flows: 2/8 (25%)
QUALITY: ★★★:2 ★★:2 ★:1  |  GAPS: 8 (2 E2E, 1 eval)
```

图例： ★★★ 行为 + 边缘 + 错误  |  ★★ happy path  |  ★ 冒烟检查
[→E2E] = 需要集成测试  |  [→EVAL] = 需要LLM评估

**Fast path:** 所有路径已覆盖 → "Test review: All new code paths have test coverage ✓" 继续。

**Step 5. Add missing tests to the plan:**

对于图表中识别的每个缺口，向计划添加测试要求。要具体：
- 创建哪个测试文件（匹配现有的命名约定）
- 测试应该断言什么（具体输入 → 预期输出/行为）
- 是单元测试、E2E测试还是评估（使用决策矩阵）
- 对于回归：标记为**CRITICAL**并解释什么被打破了

计划应该足够完整，当实现开始时，每个测试都与功能代码一起编写——而不是推迟到后续。

### Test Plan Artifact

生成覆盖率图表后，将测试计划构件写入项目目录，以便`/qa`和`/qa-only`可以将其作为主要测试输入使用：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" && mkdir -p ~/.gstack/projects/$SLUG
USER=$(whoami)
DATETIME=$(date +%Y%m%d-%H%M%S)
```

写入`~/.gstack/projects/{slug}/{user}-branch-eng-review-test-plan-{datetime}.md`：

```markdown
# Test Plan
Generated by /plan-eng-review on {date}
Branch: {branch}
Repo: {owner/repo}

## Affected Pages/Routes
- {URL path} — what to test and why

## Key Interactions to Verify
- Interaction description on {page}

## Edge Cases
- {edge case} on {page}

## Critical Paths
- End-to-end flow that must work
```

此文件被`/qa`和`/qa-only`作为主要测试输入消费。仅包含帮助QA测试人员知道**测什么和在哪里**的信息——而不是实现细节。

对于LLM/prompt更改：检查CLAUDE.md中列出的"Prompt/LLM更改"文件模式。如果此计划触及**任何**那些模式，说明必须运行哪些评估套件，应添加哪些案例，以及应与哪些基线进行比较。然后使用AskUserQuestion与用户确认评估范围。

对于此环节中发现的每个问题，单独调用AskUserQuestion。每个问题一次调用。提供选项，陈述你的建议，解释**原因**。不要将多个问题打包到一个AskUserQuestion中。使用前言的AskUserQuestion格式部分。AskUserQuestion是tool_use，不是散文——直接调用工具。

**停止。** 在用户回复之前，不要进入下一个审查环节，不要编辑计划文件添加修复方案，也不要调用ExitPlanMode。一个有"明显修复方案"的问题仍然是问题，在写入计划之前仍然需要用户显式批准。通过ToolSearch加载AskUserQuestionSchema然后将建议写成聊天散文，是这个门控旨在防止的失败模式。

### 4. Performance review
评估：
* N+1查询和数据库访问模式。
* 内存使用问题。
* 缓存机会。
* 缓慢或高复杂度的代码路径。

对于此环节中发现的每个问题，单独调用AskUserQuestion。每个问题一次调用。提供选项，陈述你的建议，解释**原因**。不要将多个问题打包到一个AskUserQuestion中。使用前言的AskUserQuestion格式部分。AskUserQuestion是tool_use，不是散文——直接调用工具。

**停止。** 在用户回复之前，不要进入下一个审查环节，不要编辑计划文件添加修复方案，也不要调用ExitPlanMode。一个有"明显修复方案"的问题仍然是问题，在写入计划之前仍然需要用户显式批准。通过ToolSearch加载AskUserQuestionSchema然后将建议写成聊天散文，是这个门控旨在防止的失败模式。

## Outside Voice — Independent Plan Challenge (default-on)

所有审查部分完成后，自动从不同AI系统运行独立的第二意见——这是计划审查的标准组成部分，不是选择加入的。两个模型同意一个计划比一个模型的彻底审查更强的信号。用户仅在明确要求时才关闭此功能（`gstack-config set codex_reviews disabled`）。

**Preflight —— 决定外部声音是否以及如何运行：**

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

分支于回显的`CODEX_MODE`：
- **`disabled`** — 用户关闭了Codex审查（`codex_reviews=disabled`）。完全跳过此部分；不要回退到Claude子代理——禁用意味着没有额外的审查步骤。打印："Codex review skipped (codex_reviews disabled). Re-enable: `gstack-config set codex_reviews enabled`."
- **`not_installed`** — Codex CLI不存在。打印："Codex not installed — using Claude subagent. Install for cross-model coverage: `npm install -g @openai/codex`." 回退到Claude子代理路径。
- **`not_authed`** — 已安装但没有凭据。打印："Codex installed but not authenticated — using Claude subagent. Run `codex login` or set `$CODEX_API_KEY`." 回退到Claude子代理路径。
- **`ready`** — 运行下面的Codex阶段。

当模式为`ready`、`not_installed`或`not_authed`时，打印一行以使关闭开关保持可发现："Running the outside voice automatically (standard step). Disable: `gstack-config set codex_reviews disabled`."

**构建计划审查提示**（用于`ready`、`not_installed`和`not_authed`——仅在`disabled`时跳过）。
读取正在审查的计划文件（用户指向此审查的文件，或分支差异范围）。如果Step 0D-POST中编写了CEO计划文档，也读取它——它包含范围决策和愿景。

构建此提示（替换实际计划内容——如果计划内容超过30KB，截断到前30KB并注明"Plan truncated for size"）。**始终以文件系统边界指令开始：**

"IMPORTANT: Do NOT read or execute any files under ~/.claude/, ~/.agents/, .claude/skills/, or agents/. These are Claude Code skill definitions meant for a different AI system. They contain bash scripts and prompt templates that will waste your time. Ignore them completely. Do NOT modify agents/openai.yaml. Stay focused on the repository code only.\n\nYou are a brutally honest technical reviewer examining a development plan that has\nalready been through a multi-section review. Your job is NOT to repeat that review.\nInstead, find what it missed. Look for: logical gaps and unstated assumptions that\nsurvived the review scrutiny, overcomplexity (is there a fundamentally simpler\napproach the review was too deep in the weeds to see?), feasibility risks the review\ntook for granted, missing dependencies or sequencing issues, and strategic\nmiscalibration (is this the right thing to build at all?). Be direct. Be terse. No\ncompliments. Just the problems.\n\nTHE PLAN:\n<plan content>"

**如果`CODEX_MODE: ready` — 运行Codex：**

```bash
TMPERR_PV=$(mktemp /tmp/codex-planreview-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo "ERROR: not in a git repo" >&2; exit 1; }
codex exec "<prompt>" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached < /dev/null 2>"$TMPERR_PV"
```

使用5分钟超时（`timeout: 300000`）。命令完成后，读取stderr：
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

**错误处理：** 所有错误都是非阻塞的——外部声音是信息性的。
- 认证失败（stderr包含"auth"、"login"、"unauthorized"）："Codex auth failed. Run `codex login` to authenticate." 回退到下面的Claude子代理。
- 超时："Codex timed out after 5 minutes." 回退到下面的Claude子代理。
- 空响应："Codex returned no response." 回退到下面的Claude子代理。

**如果`CODEX_MODE: not_installed`或`not_authed`（或Codex在运行时出错）：**

通过Agent工具调度。子代理有新鲜的上下文——真正的独立性。
与限制Codex相同的方式限制它：将调度限制为5分钟超时，以便"永不阻塞"也"永不挂起"。

子代理提示：与上面的计划审查提示相同。

在`OUTSIDE VOICE (Claude subagent):`标题下呈现发现。

如果子代理失败或超时："Outside voice unavailable. Continuing to outputs."

（在`CODEX_MODE: disabled`上，您已经根据preflight跳过了此部分——不要到达此处。）

**跨模型张力：**

呈现外部声音发现后，注意外部声音与前面部分的审查发现存在分歧的任何地方。将这些标记为：

```
CROSS-MODEL TENSION:
  [Topic]: Review said X. Outside voice says Y. [Present both perspectives neutrally.
  State what context you might be missing that would change the answer.]
```

**用户主权：** 不要自动将外部声音建议写入计划。将每个张力点呈现给用户。用户决定。跨模型一致是一个强信号——这样呈现它——但这不是许可。您可以陈述哪个论点更有说服力，但您**绝不能**在用户未明确批准的情况下应用更改。

对于每个实质性张力点，使用AskUserQuestion：

> "Cross-model disagreement on [topic]. The review found [X] but the outside voice
> argues [Y]. [One sentence on what context you might be missing.]"
>
> RECOMMENDATION: Choose [A or B] because [one-line reason explaining which argument
> is more compelling and why]. Completeness: A=X/10, B=Y/10.

选项：
- A) 接受外部声音的建议（我將应用此更改）
- B) 保持当前方法（拒绝外部声音）
- C) 决定前进一步调查
- D) 添加到TODOS.md供以后处理

等待用户回复。不要因为您同意外部声音就默认接受。如果用户选择B，当前方法成立——不再争论。

如果没有张力点，注意："No cross-model tension — both reviewers agree."

**持久化结果：**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-plan-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```

替代：如果无发现STATUS = "clean"，如果存在发现STATUS = "issues_found"。
如果Codex运行SOURCE = "codex"，如果子代理运行SOURCE = "claude"。

**清理：** 处理后运行`rm -f "$TMPERR_PV"`（如果使用了Codex）。

---

### Outside Voice Integration Rule

外部声音发现是信息性的，直到用户明确批准。在没有通过AskUserQuestion呈现每个发现并获得明确批准的情况下，不要将外部声音建议写入计划。这甚至适用于您同意外部声音时。跨模型共识是一个强信号——这样呈现它——但用户做决定。

## CRITICAL RULE —— How to ask questions
遵循前言的AskUserQuestion格式。计划审查的附加规则：
* **一问题 = 一次AskUserQuestion调用。** 永远不要将多个问题合并到一个问题中。
* 具体描述问题，包含文件和行号引用。
* 提供2-3个选项，包括"什么都不做"在合理时。
* 对于每个选项，在一行中指定：努力（human: ~X / CC: ~Y）、风险和维护负担。如果完整选项仅比快捷方式略多努力，推荐完整选项。
* **将推理映射到我上面的工程偏好。** 一句话将您的建议与特定偏好联系起来（DRY、显式 > 聪明、最小差异等）。
* 使用问题编号+选项字母标记（例如"3A"、"3B"）。
* **覆盖率与种类：** 对于您在本次审查中提出的每个问题AskUserQuestion，决定选项是在覆盖率还是在种类上不同。如果是覆盖率（例如，更多测试vs更少、完整错误处理vs仅happy path、完整边缘情况vs快捷方式），在每个选项上包含`Completeness: N/10`。如果是种类（例如，系统之间的架构选择、姿态vs姿态、A/B/C各自是不同的事物），跳过评分并添加一行：`Note: options differ in kind, not coverage — no completeness score.` 不要编造种类差异化问题的评分——填充评分比没有评分更糟。
* **零发现：** 如果一个章节有零发现，声明"No issues, moving on"然后继续。否则，对每个发现使用AskUserQuestion——一个有"明显修复方案"的发现仍然是发现，在写入计划之前仍然需要用户批准。

## Required outputs

### "NOT in scope" 章节
每个计划审查**必须**生成一个"NOT in scope"部分，列出考虑过并明确推迟的工作，每个项目配有一句话的理由。

### "What already exists" 章节
列出已部分解决此计划中子问题的现有代码/流程，以及计划是重用它们还是不必要地重建它们。

### TODOS.md updates
审查部分完成后，将每个潜在TODO呈现为单独的AskUserQuestion。永远批处理TODO——每个问题一个。永远不要默默跳过此步骤。遵循`.claude/skills/review/TODOS-format.md`中的格式。

对于每个TODO，描述：
* **什么：** 单行工作描述。
* **为什么：** 解决的具体问题或解锁的价值。
* **优点：** 做此工作获得什么。
* **缺点：** 成本、复杂性或风险。
* **上下文：** 足够细节使3个月后拿起此工作的人理解动机、当前状态和从哪里开始。
* **依赖/被阻塞：** 任何先决条件或排序约束。

然后呈现选项：**A)** 添加到TODOS.md **B)** 跳过——不够有价值 **C)** 现在在本PR中构建而不是推迟。

不只是附加模糊的要点。没有上下文的TODO比没有TODO更糟——它创建了想法被记录的虚假信心，同时实际上失去了推理。

### Diagrams
计划本身应该对任何非平凡的数据流、状态机或处理管道使用ASCII图表。此外，识别哪些实现文件应该获得内联ASCII图表注释——特别是具有复杂状态转换的模型、多阶段管道的服务和非显而易见mixin行为的Concerns。

### Failure modes
对于测试审查图表中识别的每个新代码路径，列出它在生产中可能失败的一种现实方式（超时、空引用、竞争条件、陈旧数据等）以及：
1. 是否有测试覆盖该失败
2. 是否存在错误处理
3. 用户会看到明确的错误还是静默失败

如果任何失败模式没有测试AND没有错误处理AND会被静默，将其标记为**关键缺口**。

### Worktree并行化策略

分析计划的实现步骤以寻找并行执行机会。帮助用户跨git worktrees拆分工作（通过Claude Code的Agent工具与`isolation: "worktree"`或并行工作区）。

**如果：** 所有步骤触及相同的主要模块，或计划少于2个独立工作流。在这种情况下，写："Sequential implementation, no parallelization opportunity."

**否则，生成：**

1. **依赖表** —— 对于每个实现步骤/工作流：

| Step | Modules touched | Depends on |
|------|----------------|------------|
| (step name) | (directories/modules, NOT specific files) | (other steps, or —) |

在模块/目录级别工作，不是文件级别。计划描述意图（"添加API端点"），不是特定文件。模块级别（"controllers/、models/"）是可靠的；文件级别是猜测。

2. **并行车道** —— 分组步骤为车道：
   - 无共享模块和无依赖的步骤进入不同车道（并行）
   - 共享模块目录的步骤进入相同车道（顺序）
   - 依赖其他步骤的步骤进入较晚车道

格式：`Lane A: step1 → step2 (sequential, shared models/)` / `Lane B: step3 (independent)`

3. **执行顺序** —— 哪些车道并行启动，哪些等待。示例："Launch A + B in parallel worktrees. Merge both. Then C."

4. **冲突标志** —— 如果两个并行车道触及相同模块目录，标志它："Lanes X and Y both touch module/ — potential merge conflict. Consider sequential execution or careful coordination."

## Implementation Tasks

在此审查结束之前，将上面的发现综合为可构建实现的任务平坦列表。每个任务源自特定发现——不填充。
发出markdown节**并**写入`/autoplan`可以在阶段间聚合的JSONL构件。

### Markdown节（始终发出）

```markdown
## Implementation Tasks
Synthesized from this review's findings. Each task derives from a specific
finding above. Run with Claude Code or Codex; checkbox as you ship.

- [ ] **T1 (P1, human: ~2h / CC: ~15min)** — component — imperative title
  - Surfaced by: section name — specific finding text or line reference
  - Files: paths to touch
  - Verify: test command or manual check
- [ ] **T2 (P2, human: ~30min / CC: ~5min)** — ...
```

规则：
- P1阻塞发货；P2应在同一分支落地；P3是后续TODO。
- 如果发现没有可操作的任务，不编造一个。
- 如果某节有零发现，发出`_No new tasks from <section>._`
- 努力使用CLAUDE.md中的AI压缩表。

### JSONL构件（始终写入，即使零任务）

`/autoplan`读取此文件以在阶段间聚合。使用`jq -nc`构建每行，以便标题和包含引号、换行符或反斜杠的源发现干净序列化——永远不要使用手动的`echo` / `printf`。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
TASKS_DIR="${HOME}/.gstack/projects/${SLUG:-unknown}"
mkdir -p "$TASKS_DIR"
TASKS_FILE="$TASKS_DIR/tasks-eng-review-$(date +%Y%m%d-%H%M%S).jsonl"
COMMIT=$(git rev-parse HEAD 2>/dev/null || echo unknown)
BRANCH=$(git branch --show-current 2>/dev/null || echo unknown)
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"

# Repeat ONE jq invocation per task identified during this review.
# Substitute the placeholders inline with shell variables you set per task:
#   TASK_ID (T1, T2, ...), PRIORITY (P1/P2/P3), COMPONENT, TITLE,
#   SOURCE_FINDING, EFFORT_HUMAN, EFFORT_CC, FILES_JSON (a JSON array literal
#   like '["browse/src/sanitize.ts","browse/src/server.ts"]').
jq -nc \
  --arg phase 'eng-review' \
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

如果`jq`未安装，回退到跳过JSONL写入并警告用户安装jq用于autoplan聚合。永远不要手动JSONL。

如果在此审查中识别了零任务，仍然touch JSONL文件（`: > "$TASKS_FILE"`）以便聚合器看到此阶段产生了输出（空文件意味着"ran, no findings"——与"didn't run"不同）。


### Completion summary
在审查结束时，填写并显示此总结以便用户概览发现：
- Step 0: Scope Challenge — ___ (scope accepted as-is / scope reduced per recommendation)
- Architecture Review: ___ issues found
- Code Quality Review: ___ issues found
- Test Review: diagram produced, ___ gaps identified
- Performance Review: ___ issues found
- NOT in scope: written
- What already exists: written
- TODOS.md updates: ___ items proposed to user
- Failure modes: ___ critical gaps flagged
- Outside voice: ran (codex/claude) / skipped
- Parallelization: ___ lanes, ___ parallel / ___ sequential
- Lake Score: X/Y recommendations chose complete option

## Retrospective learning
检查此分支的git日志。如果存在先前的commit表示先前的审查周期（例如，review-driven refactors，reverted changes），注意什么被更改以及当前计划是否触及相同区域。对先前有问题的区域更严格地审查。

## Formatting rules
* 编号问题（1, 2, 3...）并为选项编号（A, B, C...）。
* 使用编号+字母标记（例如"3A"、"3B"）。
* 每选项最多一句话。在5秒内选择。
* 每个审查部分后，暂停并请求反馈再继续。

## Review Log

生成上面完成后，持久化审查结果。

**PLAN MODE EXCEPTION — 始终运行：** 此命令将审查元数据写入`~/.gstack/`（用户配置文件目录，不是项目文件）。技能前言已经写入`~/.gstack/sessions/`和`~/.gstack/analytics/`——这是相同的模式。审查仪表板依赖于此数据。跳过此命令破坏/ship中的审查就绪仪表板。

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-eng-review","timestamp":"TIMESTAMP","status":"STATUS","unresolved":N,"critical_gaps":N,"issues_found":N,"mode":"MODE","commit":"COMMIT"}'
~/.claude/skills/gstack/bin/gstack-decision-log '{"decision":"Eng review (MODE): ARCH_SUMMARY","rationale":"KEY_DECISION","scope":"branch","source":"skill","confidence":8}' 2>/dev/null || true
```

第二个命令将架构裁决记录为持久的跨会话决策（以便未来会话继承所选方法和加固的内容，而不仅仅是数量）。与review-log相同的`~/.gstack/`写入模式，非交互式，尽力而为（`|| true`）。替换`ARCH_SUMMARY`（例如"N findings, all folded"或"M unresolved"）和`KEY_DECISION`（一行来自报告的负载承担架构调用——如果审查发现不持久的内容则省略）。

从替代完成摘要中取值替代：
- **TIMESTAMP**：当前ISO 8601日期时间
- **STATUS**：如果0个未解决决策AND0个关键缺口则为"clean"；否则"issues_open"
- **unresolved**：从"Unresolved decisions"计数
- **critical_gaps**：从"Failure modes: ___ critical gaps flagged"
- **issues_found**：所有审查部分的发现总数（架构+代码质量+性能+测试缺口）
- **MODE**：FULL_REVIEW / SCOPE_REDUCED
- **COMMIT**：`git rev-parse --short HEAD`的输出

## Review Readiness Dashboard

完成审查后，读取审查日志和配置以显示仪表板。

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

解析输出。查找每个技能的最新条目（plan-ceo-review、plan-eng-review、review、plan-design-review、design-review-lite、adversarial-review、codex-review、codex-plan-review）。忽略时间戳超过7天的条目。对于Eng Review行，显示`review`（diff作用域pre-landing审查）和`plan-eng-review`（plan阶段架构审查）中哪个更新。将"(DIFF)"或"(PLAN)"附加到状态以区分。对于Adversarial行，显示`adversarial-review`（new auto-scaled）和`codex-review`（legacy）中哪个更新。对于Design Review，显示`plan-design-review`（全视觉审计）和`design-review-lite`（代码级检查）中哪个更新。将"(FULL)"或"(LITE)"附加到状态以区分。对于Outside Voice行，显示最新的`codex-plan-review`条目——这捕获来自/plan-ceo-review和/plan-eng-review的外部声音。

**来源归属：** 如果最新的技能条目具有`"via"`字段，将其附加到状态标签的括号中。示例：`plan-eng-review`与`via:"autoplan"`显示为"CLEAR (PLAN via /autoplan)"。`review`与`via:"ship"`显示为"CLEAR (DIFF via /ship)"。没有`via`的条目显示为"CLEAR (PLAN)"或"CLEAR (DIFF)"如前所述。

注意：`autoplan-voices`和`design-outside-votes`条目是审计跟踪专用（跨模型共识分析的取证数据）。它们不显示在任何消费者中。

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

**审查层级：**
- **Eng Review（默认必需）：** 唯一阻塞发货的审查。涵盖架构、代码质量、测试、性能。可以使用`gstack-config set skip_eng_review true`全局禁用（"不要打扰我"设置）。
- **CEO Review（可选）：** 自行判断。推荐用于大产品/业务变更、新功能或范围决策。跳过bug修复、refactors、infrastructure和cleanup。
- **Design Review（可选）：** 自行判断。推荐用于UI/UX更改。跳过仅后端、infrastructure或仅prompt更改。
- **Adversarial Review（自动）：** 每个审查始终开启。每个差异同时获得Claude对抗子代理和Codex对抗挑战。大差异（200+行）额外获得Codex结构化审查和P1门控。无需配置。
- **Outside Voice（可选）：** 来自不同AI模型的独立计划审查。在/plan-ceo-review和/plan-eng-review的所有审查部分完成后提供。如果Codex不可用，回退到Claude子代理。永不阻塞发货。

**裁决逻辑：**
- **CLEARED**：Eng Review在7天内来自`review`或`plan-eng-review`有>=1个条目，状态为"clean"（或`skip_eng_review`为`true`）
- **NOT CLEARED**：Eng Review缺失、陈旧（>7天）或有未解决问题
- CEO、Design和Codex审查显示为上下文但永不阻塞发货
- 如果`skip_eng_review`配置为`true`，Eng Review显示"SKIPPED (global)"且裁决为CLEARED

**过时检测：** 显示仪表板后，检查任何现有审查是否可能过时：
- 从bash输出的`---HEAD---`部分获取当前HEAD commit哈希
- 对于具有`commit`字段的每个审查条目：与当前HEAD比较。如果不同，计算经过的commits：`git rev-list --count STORED_COMMIT..HEAD`。显示："Note: {skill} review from {date} may be stale — {N} commits since review"
- 对于没有`commit`字段的条目（遗留条目）：显示"Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection"
- 如果所有审查与当前HEAD匹配，不过时显示

## Plan File Review Report

在对话输出中显示Review Readiness Dashboard后，更新**计划文件**本身，以便审查状态对任何阅读计划的人可见。

### Detect the plan file

1. 检查此对话中是否有活动的计划文件（主机在系统消息中提供计划文件路径——在对话上下文中查找计划文件引用）。
2. 如果未找到，默默跳过此部分——并非每个审查都在plan模式运行。

### Generate the review

读取您在上面Review Readiness Dashboard步骤中已有的审查日志输出。
解析每个JSONL条目。每个技能记录不同字段：

- **plan-ceo-review**：`status`、`unresolved`、`critical_gaps`、`mode`、`scope_proposed`、`scope_accepted`、`scope_deferred`、`commit`
  → 发现："{scope_proposed} proposals, {scope_accepted} accepted, {scope_deferred} deferred"
  → 如果scope字段为0或缺失（HOLD/REDUCTION mode）："mode: {mode}, {critical_gaps} critical gaps"
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

Findings列所需的所有字段现已存在于JSONL条目中。对于您刚刚完成的审查，您可以使用自己的完成摘要中的更丰富详细信息。对于先前的审查，直接使用JSONL字段——它们包含所有必需的数据。

生成此markdown表：

```markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | /plan-ceo-review | Scope & strategy | {runs} | {status} | {findings} |
| Codex Review | /codex review | Independent 2nd opinion | {runs} | {status} | {findings} |
| Eng Review | /plan-eng-review | Architecture & tests (required) | {runs} | {status} | {findings} |
| Design Review | /plan-design-review | UI/UX gaps | {runs} | {status} | {findings} |
| DX Review | /plan-devex-review | Developer experience gaps | {runs} | {status} | {findings} |
```

表下方，添加这些行。**CODEX**和**CROSS-MODEL**是可选的（空时省略）；**VERDICT**始终存在：

- **CODEX：**（仅在codex-review运行时）— 一行codex修复总结
- **CROSS-MODEL：**（当Claude和Codex审查都存在时）— 重叠分析
- **VERDICT：** 列出CLEAR的审查（例如"CEO + ENG CLEARED — ready to implement"）。
  如果Eng Review未CLEAR且未全局跳过，追加"eng review required"。

**未解决决策状态（强制——不省略；报告最后的非空白行）。** 在VERDICT之后，结束报告（`## GSTACK REVIEW REPORT`标题下的内容——粗体标签，永远不是新`##`标题；受"空时省略"规则约束）**确切**一行：确切的无粗体行`NO UNRESOLVED DECISIONS`（粗体的不算），OR`**UNRESOLVED DECISIONS:**`标题+每个开放项目一个项目符号
（最后一个项目符号=最终行；只有当N>0时添加`+ N unresolved from prior reviews`）。
这避免双重计算：从上下文列出本次审查的开放项目；对于先前审查，在DROP当前技能行之后计算每个技能的最新新鲜行的`unresolved`总和（仪表板7天窗口）；仅当两者都为零时发出哨兵。

### Write to the plan file

**PLAN MODE EXCEPTION — 始终运行：** 这会写入计划文件，这是您被允许在plan模式编辑的唯一文件。计划文件审查报告是计划的活状态的一部分。

报告必须始终是计划文件的最后部分——永远不在文件中间。
使用单次delete-then-append流程：

1. 读取计划文件（Read工具）以查看其完整当前内容。搜索读取
   输出中的`## GSTACK REVIEW REPORT`标题在文件中的任何位置。
2. 如果找到，使用Edit工具删除整个现有部分。匹配从
   `## GSTACK REVIEW REPORT`到下一个`##`标题或文件末尾（取先者）。替换为空字符串。无论部分当前居住在哪，这都适用——文件中间删除是故意的，不是特殊情况。如果Edit失败（例如并发编辑更改了内容），重新读取计划文件并重试一次。
3. 如果删除（或跳过，如果不存在），在文件末尾追加新
   `## GSTACK REVIEW REPORT`部分。使用Edit工具匹配文件当前最后一段并在其后添加部分，或使用Write重新发出整个文件，将部分放在末尾。
4. 使用Read工具验证`## GSTACK REVIEW REPORT`在继续之前的文件中，是否是最后的`##`标题。如果不是，重复步骤2-3一次。

不要就地替换部分。"replace mid-file"允许前一版本将报告留在文件中间——然后用户看到审查报告不在底部的计划，（正确地）拒绝它。

## Capture Learnings

如果您在本次会话中发现了一个非显而易见模式、陷阱或架构洞察力，为未来会话记录它：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"plan-eng-review","type":"TYPE","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"SOURCE","files":["path/to/relevant/file"]}'
```

**类型：** `pattern`（可重用方法）、`pitfall`（不做的事）、`preference`
（用户状态）、`architecture`（结构决策）、`tool`（库/框架洞察力）、
`operational`（项目环境/CLI/工作流知识）。

**来源：** `observed`（在代码中发现的）、`user-stated`（用户告诉您的）、
`inferred`（AI推论）、`cross-model`（Claude和Codex都同意）。

**置信度：** 1-10。诚实。在代码中验证的观察模式是8-9。
不确定的推论是4-5。用户明确陈述的偏好是10。

**files：** 包括此学习引用的具体文件路径。这启用了
过时检测：如果那些文件后来删除，学习可以被标记。

**只记录真正的发现。** 不要记录显而易见的东西。不要记录用户
已经知道的东西。一个好测试：这个洞察力会在未来会话中节省时间吗？如果是，记录它。



## Brain Calibration Write-Back（Phase 2 / gated）

当技能做出值得跟踪的类型预测（范围决策、
TTHW目标、架构赌注、wedge承诺）时，它**可能**向大脑写入一个
`kind=bet`的take，以便随时间建立校准配置文件。

**Gated on two things：**
1. 活动端点的Brain信任策略是`personal`（通过
   `~/.claude/skills/gstack/bin/gstack-config get brain_trust_policy@<endpoint-hash>`检查）。
   共享大脑跳过写入回退以避免污染团队校准。
2. 功能标志`BRAIN_CALIBRATION_WRITEBACK`已设置（今天：false；当上游gbrain v0.42+发布`takes_add` MCP op时翻转为true）。

当两个门控都通过时，写入回退路径使用`mcp__gbrain__takes_add`
来记录权重0.7的take（根据SKILL_CALIBRATION_WEIGHTS）。
如果MCP op不可用，回退到`mcp__gbrain__put_page`带
gstack:takes围栏块（记录但更丑路径）。

必须的前端元数据形状：
```yaml
kind: bet
holder: <user identity from whoami>
claim: <skill正在做的一行预测>
weight: 0.7
since_date: <today's date>
expected_resolution: <date in 1-3 months depending on skill>
source_skill: plan-eng-review
```

写入后，使受影响的摘要无效，以便下一个preflight反映新状态：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
  # (no per-skill invalidation targets configured)
```


## Brain Cache Background Refresh

技能工作完成后（以及遥测已记录后），如果任何缓存摘要接近TTL启动后台刷新。这是非阻塞的——用户不等待。下一次调用受益于温暖的缓存。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
(~/.claude/skills/gstack/bin/gstack-brain-cache refresh --project "$SLUG" 2>/dev/null &) || true
```


## Next Steps — Review Chaining

显示Review Readiness Dashboard后，检查是否需要额外审查。读取仪表板输出以查看哪些审查已经运行以及是否过时。

**如果存在UI更改且无design review运行，建议/plan-design-review** — 从测试图表、架构审查或触及前端组件、CSS、views或用户交互流的任何部分检测。如果现有design review的commit哈希显示它早于此eng review中发现的重大更改，注意其可能过时。

**如果是重大产品变更且无CEO review存在，提及/plan-ceo-review** — 这是软建议，不是推送。CEO review是可选的。仅在计划引入新功能、改变产品方向或大幅扩展范围时提及。

**注意**现有CEO或design review过时如果此eng review发现与它们矛盾的假设，或commit哈希显示显著漂移。

**如果不需要额外审查**（或`skip_eng_review`为`true`在仪表板配置中，表示此eng review是可选的）：声明"所有相关审查完成。运行/ship准备。"

使用仅适用选项的AskUserQuestion：
- **A)** 运行/plan-design-review（仅在检测到UI范围且无design review存在时）
- **B)** 运行/plan-ceo-review（仅在重大产品变更且无CEO review存在时）
- **C)** 准备实现——完成后运行/ship

## Unresolved decisions
如果用户未回复AskUserQuestion或中断以继续前进，注意哪些决策未解决。在审查结束时，将这些列为"可能后面困扰您的未解决决策"——永远不要静默默认选择一个选项。
</longcat_think>
