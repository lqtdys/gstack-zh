<!-- AUTO-GENERATED from test-coverage.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 第 7 步：测试覆盖率审计

**将此步骤作为子代理** 调度，使用 Agent 工具并指定 `subagent_type: "general-purpose"`。子代理在独立的上下文窗口中运行覆盖率审计 —— 父代理仅看到结论，不接触中间的读取文件。这是上下文衰减防御。

**子代理提示词：** 将以下指令传递给子代理，`<base>` 替换为基础分支：

> 你正在执行 ship 工作流的测试覆盖率审计。按需运行 `git diff <base>...HEAD`。不要提交或推送 —— 仅输出报告。
>
> 100% 覆盖率是目标 —— 每条未测试的路径都是躲藏 bug 的地方，凭感觉编码会变成瞎搞编码。评估**实际编写了什么**（来自 diff），而不是计划了什么。

### 测试框架检测

在分析覆盖率之前，检测项目的测试框架：

1. **阅读 CLAUDE.md** —— 查找 `## Testing` 部分，包含测试命令和框架名称。如果找到，将其作为权威来源使用。
2. **如果 CLAUDE.md 没有 Testing 部分，自动检测：**

```bash
setopt +o nomatch 2>/dev/null || true  # zsh 兼容
# 检测项目运行时
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
# 检查现有测试基础设备
ls jest.config.* vitest.config.* playwright.config.* cypress.config.* .rspec pytest.ini phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
```

3. **如果未检测到框架：** 进入测试框架初始化步骤（第 4 步），该步骤负责完整设置。

**0. 前后测试计数：**

```bash
# 在任何生成之前的测试文件计数
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

存储此数字用于 PR 正文。

**1. 追踪每条变更的代码路径** 使用 `git diff origin/<base>...HEAD`：

读取每个已变更文件。对于每个文件，追踪数据如何在代码中流动 —— 不要仅列出函数，要真正追踪执行过程：

1. **读取 diff。** 对于每个已变更文件，读取完整文件（不仅仅是 diff 块）以理解上下文。
2. **追踪数据流。** 从每个入口点（路由处理函数、导出函数、事件监听器、组件渲染）开始，追踪数据经过的每个分支：
   - 输入从哪里来？（请求参数、props、数据库、API 调用）
   - 它被什么转换？（验证、映射、计算）
   - 它去向哪里？（数据库写入、API 响应、渲染输出、副作用）
   - 每一步可能出什么问题？（null/undefined、无效输入、网络故障、空集合）
3. **绘制执行图表。** 对于每个已变更文件，绘制一个 ASCII 图表展示：
   - 每个新增或修改的函数/方法
   - 每个条件分支（if/else、switch、三元表达式、保护子句、提前返回）
   - 每个错误路线（try/catch、rescue、错误边界、回退）
   - 每个调用其他函数的调用（追踪进去 —— 它本身有未测试的分支吗？）
   - 每个边缘：null 输入会怎么样？空数组？无效类型？

这是关键步骤 —— 你在构建一张地图，记录每行可能根据输入不同执行的代码。此图表中的每个分支都需要一个测试。

**2. 映射用户流程、交互和错误状态：**

代码覆盖率不够 —— 你需要覆盖真实用户如何与变更代码交互。对于每个变更的功能，思考：

- **用户流程：** 用户采取什么操作序列触及此代码？映射整个旅程（例如，"用户点击 '支付' → 表单验证 → API 调用 → 成功/失败界面"）。旅程中的每个步骤都需要一个测试。
- **交互边界情况：** 用户做了什么意外操作？
  - 双击/快速重复提交
  - 中途导航离开（后退按钮、关闭标签页、点击其他链接）
  - 使用过期数据提交（页面打开了 30 分钟，会话已过期）
  - 慢连接（API 调用耗时 10 秒 —— 用户看到什么？）
  - 并发操作（两个相同窗口，同一个表单）
- **用户可见的错误状态：** 对于代码处理的每个错误，用户实际体验是什么？
  - 是否有明确的错误信息，还是静默失败？
  - 用户可以恢复（重试、返回、修正输入）还是卡住？
  - 无网络会怎样？API 返回 500 会怎样？服务器返回无效数据会怎样？
- **空/零/边界状态：** UI 显示零条结果时怎样？10,000 条时怎样？单字符输入时怎样？最大长度输入时怎样？

将这些添加到图表中，与代码路径放在一起。无测试的用户流程与无测试的 if/else 一样是缺口。

**3. 对现有测试检查每个分支：**

逐个检查图表中的每个分支 —— 代码路径和用户流程。对于每个分支，搜索是否有测试对其进行覆盖：
- 函数 `processPayment()` → 查找 `billing.test.ts`、`billing.spec.ts`、`test/billing_test.rb`
- 一个 if/else → 查找覆盖**真**和**假**两个路径的测试
- 错误处理器 → 查找触发该特定错误条件的测试
- 调用 `helperFn()` 且其自身有分支 → 那些分支也需要测试
- 用户流程 → 查找遍历旅程的集成或 E2E 测试
- 交互边界情况 → 查找模拟意外操作的测试

质量评分标准：
- ★★★  带边界情况和错误路径测试行为
- ★★   测试行为正确，仅快乐路径
- ★    冒烟测试/存在检查/简单断言（例如，"它渲染了"，"它没抛出"）

### E2E 测试决策矩阵

在检查每个分支时，还要判断单元测试或 E2E/集成测试是否是合适的工具：

**推荐 E2E（在图表中标记为 [→E2E]）：**
- 涉及 3+ 组件/服务的普通用户流程（例如，注册 → 验证邮箱 → 首次登录）
- 模拟会隐藏真实故障的集成点（例如，API → 队列 → worker → 数据库）
- 认证/支付/数据销毁流程 —— 太重要不能仅信任单元测试

**推荐 EVAL（在图表中标记为 [→EVAL]）：**
- 需要质量评估的关键 LLM 调用（例如，提示词变更 → 输出是否仍满足质量标准）
- 对提示词模板、系统指令或工具定义的变更

**保持单元测试：**
- 具有清晰输入/输出的纯函数
- 无副作用的内部辅助函数
- 单个函数的边界情况（null 输入、空数组）
- 非客户面向的罕见/晦涩流程

### 回归规则（强制）

**铁律：** 当覆盖率审计识别到一个 **回归** —— 之前能正常工作的代码但被 diff 破坏了，必须立即编写回归测试。不用 AskUserQuestion。不能跳过。回归是最高优先级的测试，因为它们证明了某些东西确实坏了。

回归的定义：
- diff 修改了现有行为（不是新代码）
- 现有测试套件（如有）没有覆盖变更的路径
- 变更为现有调用者引入了新的故障模式

在变更是否为回归有疑问时，偏向于编写测试。

格式：提交为 `test: regression test for {what broke}`

**4. 输出 ASCII 覆盖率图表：**

在同一图表中包含代码路径和用户流程。标记适合 E2E 和评估的路径：

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

图例：★★★ 行为+边界+错误  |  ★★ 快乐路径  |  ★ 冒烟检查
[→E2E] = 需要的集成测试  |  [→EVAL] = 需要的 LLM 评估

**快速路径：** 所有路径都被覆盖 → "Step 7: 所有新代码路径都有测试覆盖率 ✓" 继续。

**5. 为未覆盖路径生成测试：**

如果检测到测试框架（或在第 4 步中初始化的）：
- 优先错误处理器和边界情况（快乐路径更可能已被测试）
- 阅读 2-3 个现有测试文件以完全匹配项目约定
- 生成单元测试。模拟所有外部依赖（DB、API、Redis）
- 对于标记 [→E2E] 的路径：使用项目的 E2E 框架（Playwright、Capybara 等）生成集成/E2E 测试
- 对于标记 [→EVAL] 的路径：使用项目的 eval 框架生成评估测试，或标记为手动评估（如果不存在）
- 编写用真实断言测试特定未覆盖路径的测试
- 运行每个测试。通过 → 提交为 `test: coverage for {feature}`
- 失败 → 修复一次。仍然失败 → 回退，在图表中记录缺口

上限：最多 30 个代码路径，最多生成 20 个测试（代码 + 用户流程合计），每测试探索时间上限 2 分钟。

如果无测试框架**且**用户拒绝初始化 → 仅生成图表，不生成测试。注意："测试生成跳过 —— 未配置测试框架。"

**diff 仅变更测试：** 完全跳过第 7 步："没有新的应用程序代码路径需要审计。"

**6. 后计数和覆盖率摘要：**

```bash
# 生成后的测试文件计数
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

PR 正文使用：`Tests: {before} → {after} (+{delta} new)`
覆盖率行：`Test Coverage Audit: N new code paths. M covered (X%). K tests generated, J committed.`

**7. 覆盖率门禁：**

继续之前，检查 CLAUDE.md 中是否有 `## Test Coverage` 部分包含 `Minimum:` 和 `Target:` 字段。如果找到，使用这些百分比。否则使用默认值：Minimum = 60%，Target = 80%。

使用子步骤 4 中图表的覆盖率百分比（`COVERAGE: X/Y (Z%)` 行）：

- **>= target：** 通过。"Coverage gate: PASS ({X}%)." 继续。
- **>= minimum，< target：** 使用 AskUserQuestion：
  - "AI 评估的覆盖率为 {X}%。{N} 个代码路径未测试。目标为 {target}%。"
  - RECOMMENDATION: 选择 A，因为未测试的代码路径是生产 bug 躲藏的地方。
  - 选项：
    - A) 为剩余缺口生成更多测试（推荐）
    - B) 直接发布 —— 我接受覆盖率风险
    - C) 这些路径不需要测试 —— 标记为有意不覆盖
  - 如果 A：循环回子步骤 5（生成测试），针对剩余缺口。第二次通过后，如果仍低于 target，用更新后的数字再次呈现 AskUserQuestion。总共最多 2 次生成。
  - 如果 B：继续。在 PR 正文中包含："Coverage gate: {X}% —— 用户接受风险。"
  - 如果 C：继续。在 PR 正文中包含："Coverage gate: {X}% —— {N} 个路径有意不覆盖。"

- **< minimum：** 使用 AskUserQuestion：
  - "AI 评估的覆盖率极低（{X}%）。{M} 个代码路径中的 {N} 个没有测试。最低门槛为 {minimum}%。"
  - RECOMMENDATION: 选择 A，因为低于 {minimum}% 意味着未测试代码多于已测试代码。
  - 选项：
    - A) 为剩余缺口生成测试（推荐）
    - B) 覆盖 —— 以低覆盖率发布（我理解风险）
  - 如果 A：循环回子步骤 5。最多 2 次。如果 2 次后仍低于 minimum，再次呈现覆盖选项。
  - 如果 B：继续。在 PR 正文中包含："Coverage gate: OVERRIDDEN at {X}%。"

**覆盖率百分比不确定：** 如果覆盖率图表没有产生清晰的数字百分比（模糊的输出、解析错误），**跳过门禁**并提示："Coverage gate: 无法确定百分比 —— 跳过。" 不要默认为 0% 或卡住。

**仅测试变更 diff：** 跳过门禁（同现有的快速路径）。

**100% 覆盖率：** "Coverage gate: PASS (100%)." 继续。

### 测试计划产物

在生成覆盖率图表后，写一个测试计划产物，供 `/qa` 和 `/qa-only` 消费：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" && mkdir -p ~/.gstack/projects/$SLUG
USER=$(whoami)
DATETIME=$(date +%Y%m%d-%H%M%S)
```

写入 `~/.gstack/projects/{slug}/{user}-{branch}-ship-test-plan-{datetime}.md`：

```markdown
# Test Plan
Generated by /ship on {date}
Branch: {branch}
Repo: {owner/repo}

## Affected Pages/Routes
- {URL 路径} —— 测试内容和原因

## Key Interactions to Verify
- {交互描述} on {page}

## Edge Cases
- {boundary case} on {page}

## Critical Paths
- {end-to-end flow that must work}
```

> 分析完成后，在你输出的最后一行输出一个 JSON 对象（之后没有任何其他文本）：
> `{"coverage_pct":N,"gaps":N,"diagram":"<完整 markdown 覆盖率图表，用于 PR 正文>","tests_added":["path",...]}`

**父代理处理：**

1. 阅读子代理的最终输出。将最后一行为 JSON 进行解析。
2. 存储 `coverage_pct`（用于第 20 步指标）、`gaps`（用户摘要）、`tests_added`（用于提交）。
3. 将 `diagram` 逐字嵌入 PR 正文的 `## Test Coverage` 部分（第 19 步）。
4. 打印单行摘要：`Coverage: {coverage_pct}%, {gaps} gaps. {tests_added.length} tests added.`

**如果子代理失败、超时或返回无效 JSON：** 回退到在父代理中内联运行审计。不要让 /ship 因子代理失败而阻塞 —— 部分结果也好过没有。

---
