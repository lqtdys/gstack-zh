<!-- AUTO-GENERATED from plan-completion.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## Step 8：计划完成度审计

**将此步骤作为子代理派发**，使用 Agent 工具并设置 `subagent_type: "general-purpose"`。子代理在自身的全新上下文中读取计划文件及所有引用的代码文件。父流程只接收结论。

**子代理提示词：** 将以下指令传递给子代理：

> 你正在执行 ship 工作流的计划完成度审计。基准分支是 `<base>`。使用 `git diff <base>...HEAD` 查看已交付的内容。不要提交或推送——仅作报告。
>
> ### 计划文件发现
>
> 1. **对话上下文（首选）：** 检查当前对话中是否有活跃的计划文件。当处于计划审查模式时，宿主 agent 的系统消息会包含计划文件路径。如果找到，直接使用——这是最可靠的信号。
>
> 2. **基于内容的搜索（后备）：** 如果对话上下文中未引用计划文件，则按内容搜索：
>
> ```bash
> setopt +o nomatch 2>/dev/null || true  # zsh 兼容
> BRANCH=$(git branch --show-current 2>/dev/null | tr '/' '-')
> REPO=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)")
> # 计算项目路径段，用于 ~/.gstack/projects/ 查找
> _PLAN_SLUG=$(git remote get-url origin 2>/dev/null | sed 's|.*[:/]\\([^/]*/[^/]*\\)\\.git$|\\1|;s|.*[:/]\\([^/]*/[^/]*\\)$|\\1|' | tr '/' '-' | tr -cd 'a-zA-Z0-9._-') || true
> _PLAN_SLUG="${_PLAN_SLUG:-$(basename "$PWD" | tr -cd 'a-zA-Z0-9._-')}"
> # 搜索常见计划文件位置（先项目设计文件，再个人/本地）
> for PLAN_DIR in "$HOME/.gstack/projects/$_PLAN_SLUG" "$HOME/.claude/plans" "$HOME/.codex/plans" ".gstack/plans"; do
>   [ -d "$PLAN_DIR" ] || continue
>   PLAN=$(ls -t "$PLAN_DIR"/*.md 2>/dev/null | xargs grep -l "$BRANCH" 2>/dev/null | head -1)
>   [ -z "$PLAN" ] && PLAN=$(ls -t "$PLAN_DIR"/*.md 2>/dev/null | xargs grep -l "$REPO" 2>/dev/null | head -1)
>   [ -z "$PLAN" ] && PLAN=$(find "$PLAN_DIR" -name '*.md' -mmin -1440 -maxdepth 1 2>/dev/null | xargs ls -t 2>/dev/null | head -1)
>   [ -n "$PLAN" ] && break
> done
> [ -n "$PLAN" ] && echo "PLAN_FILE: $PLAN" || echo "NO_PLAN_FILE"
> ```
>
> 3. **验证：** 如果通过内容搜索（非对话上下文）找到了计划文件，读取前 20 行并验证其与当前分支的工作相关。如果似乎来自其他项目或功能，则视为"未找到计划文件"。
>
>**错误处理：**
>- 未找到计划文件 → 跳过，提示"未检测到计划文件——跳过。"
>- 计划文件找到但不可读（权限、编码问题）→ 跳过，提示"计划文件找到但不可读——跳过。"

### 可执行项提取

读取计划文件。提取所有可执行项——即描述待完成工作的任何内容。查找以下模式：

- **复选项：** `- [ ] ...` 或 `- [x] ...`
- **实现标题下的编号步骤：** "1. 创建……"，"2. 添加……"，"3. 修改……"
- **祈使语句：** "在 Y 中添加 X"，"创建 Z 服务"，"修改 W 控制器"
- **文件级别规范：** "新文件：path/to/file.ts"，"修改 path/to/existing.rb"
- **测试要求：** "测试 X"，"为 Y 添加测试"，"验证 Z"
- **数据模型变更：** "在表 Y 中添加列 X"，"为 Z 创建迁移"

**忽略：**
- 上下文/背景部分（`## Context`、`## Background`、`## Problem`）
- 问题和开放项（标有 ?、"TBD"、"TODO: decide"）
- 审查报告部分（`## GSTACK REVIEW REPORT`）
- 明确推迟的项（"Future:"、"Out of scope:"、"NOT in scope:"、"P2:"、"P3:"、"P4:"）
- CEO 审查决策部分（这些记录的是选择，而非工作项）

**上限：** 最多提取 50 项。如果计划超过 50 项，注明："显示前 50 项，共 N 项——完整列表见计划文件。"

**未找到项：** 如果计划不包含任何可提取的可执行项，跳过并提示："计划文件不含可执行项——跳过完成度审计。"

对每个项目，记录：
- 项文本（原文或简洁摘要）
- 类别：CODE | TEST | MIGRATION | CONFIG | DOCS

### 验证模式

在判断完成度之前，先对每项进行分类，确定**如何验证**。仅凭 diff 无法证明所有类型的工作。当前仓库或系统之外的项在 `git diff` 中是结构性不可见的。

- **DIFF-VERIFIABLE（可通过 diff 验证）** — 此仓库中的代码变更会体现在 `git diff <base>...HEAD` 中。例如："添加 UserService"（文件出现）、"验证输入 X"（验证逻辑出现）、"创建 users 表"（迁移文件出现）。
- **CROSS-REPO（跨仓库）** — 项目命名了兄弟仓库中的文件或变更（例如 `domain-hq/docs/dashboard.md`，`~/Development/<other-repo>/...`）。当前 diff 无法证明此项。
- **EXTERNAL-STATE（外部状态）** — 项目命名了外部系统中的状态：Supabase 配置/RLS、Cloudflare DNS、Vercel 环境变量、OAuth 提供方白名单、第三方 SaaS、DNS 记录。当前 diff 无法证明此项。
- **CONTENT-SHAPE（内容形态）** — 项目要求文件遵循特定约定。如果文件在此仓库中：可通过 diff 验证。如果在其他仓库或系统中：参见 CROSS-REPO / EXTERNAL-STATE。

**验证派发：**

- **DIFF-VERIFIABLE** → 与 diff 交叉引用（下一节）。
- **CROSS-REPO** → 如果兄弟仓库在磁盘上可达（尝试 `~/Development/<repo>/`、`~/code/<repo>/`、当前仓库的父目录），运行 `[ -f <path> ]` 检查文件是否存在。文件存在 → DONE（引用路径）。文件不存在 → NOT DONE（引用路径）。路径不可达 → UNVERIFIABLE（引用需手动检查的内容）。
- **EXTERNAL-STATE** → UNVERIFIABLE。引用系统及用户必须执行的特定检查。
- **CONTENT-SHAPE 在另一仓库中** → 如果文件存在，在回退到 UNVERIFIABLE 之前运行项目检测到的验证器（见下方"验证器检测"）。使用验证器：通过 → DONE；失败 → NOT DONE（引用验证器输出）。无可用验证器：分类为 UNVERIFIABLE，并引用文件路径和需确认的约定。

**路径具体性规则。** 如果计划项命名了*具体文件系统路径*（绝对路径、`~/...` 或 `<sibling-repo>/<file>`），必须基于 `[ -f <path> ]` 将其分类为 DONE 或 NOT DONE。仅当路径确实抽象（"Cloudflare DNS"、"Supabase 白名单"）或兄弟仓库根目录在此机器上不可达时，UNVERIFIABLE 才有效。"我不想检查"不等于不可达。

**验证器检测。** 在 CONTENT-SHAPE 项回退到 UNVERIFIABLE 之前，扫描目标仓库的 `package.json`，寻找任何匹配 `validate-*`、`lint-wiki`、`check-docs` 或类似模式的脚本。如果找到，使用相关路径参数调用它（例如 `npm run validate-wiki -- <path>`）。对于多目标验证器（例如 `validate-wiki --all`），运行一次并根据输出逐项协调。通过验证器会将项从 UNVERIFIABLE 提升为 DONE；失败则降级为 NOT DONE。

**诚实规则。** 不要因为相关代码已交付就将项分类为 DONE。*处理*交付物的代码不等于交付物本身。交付一个 markdown 提取库不等于交付 markdown 文件。在 DONE 和 UNVERIFIABLE 之间犹豫时，优先选择 UNVERIFIABLE——宁可弹出确认提示，也不要悄悄漏掉交付物。

### 与 diff 交叉引用

运行 `git diff origin/<base>...HEAD` 和 `git log origin/<base>..HEAD --oneline` 来了解已实现的内容。

对每个提取的计划项，执行上一节的验证派发，然后分类：

- **DONE** — 项已交付的清晰证据。对于 DIFF-VERIFIABLE 项，引用 diff 中变更的具体文件；对于 CROSS-REPO 项且兄弟仓库可达时，引用经验证存在的路径。
- **PARTIAL** — 该项存在部分工作但不完整（例如模型已创建但控制器缺失，函数存在但未处理边缘情况）。
- **NOT DONE** — 验证已运行并产生负面证据（文件缺失、diff 中代码不存在、兄弟仓库文件确认不存在）。
- **CHANGED** — 项使用计划描述之外的方法实现了，但目标已达成。注明差异。
- **UNVERIFIABLE** — diff 和任何可达的兄弟仓库检查都无法证明或反驳此项。始终适用于 EXTERNAL-STATE 项，以及兄弟仓库不可达的 CROSS-REPO 项。引用用户必须执行的具体手动验证（例如，"检查 Cloudflare DNS 显示 dashboard.example.com 为仅 DNS 模式"，"确认 domain-hq 仓库中存在 /docs/dashboard.md"）。

**对 DONE 保持保守** — 需要清晰证据。仅被触碰的文件不够；描述的具体功能必须存在。
**对 CHANGED 保持宽松** — 如果目标通过不同手段达成，也算已解决。
**对 UNVERIFIABLE 保持诚实** — 宁可弹出 5 个需要用户手动确认的项，也不要悄悄将它们分类为 DONE。

### 输出格式

```
PLAN COMPLETION AUDIT
═══════════════════════════════
Plan: {plan file path}

## Implementation Items
  [DONE]         Create UserService — src/services/user_service.rb (+142 lines)
  [PARTIAL]      Add validation — model validates but missing controller checks
  [NOT DONE]     Add caching layer — no cache-related changes in diff
  [CHANGED]      "Redis queue" → implemented with Sidekiq instead

## Test Items
  [DONE]         Unit tests for UserService — test/services/user_service_test.rb
  [NOT DONE]    E2E test for signup flow

## Migration Items
  [DONE]         Create users table — db/migrate/20240315_create_users.rb

## Cross-Repo / External Items
  [DONE]         sibling-repo has /docs/dashboard.md — verified at ~/Development/sibling-repo/docs/dashboard.md
  [UNVERIFIABLE] Cloudflare DNS-only on api.example.com — external system, manual check required
  [UNVERIFIABLE] Supabase auth allowlist contains user email — external system, confirm in Supabase dashboard

─────────────────────────────────
COMPLETION: 5/9 DONE, 1 PARTIAL, 1 NOT DONE, 1 CHANGED, 2 UNVERIFIABLE
─────────────────────────────────
```

### 门控逻辑

生成完成清单后，按优先级顺序评估：

1. **任何 NOT DONE 项**（最高优先级——已知的缺失工作）。使用 AskUserQuestion：
   - 展示上述完成清单
   - "{N} 个计划项为 NOT DONE。这些属于原始计划的一部分，但实现中缺失。"
   - RECOMMENDATION：取决于项数量和严重程度。如果是 1-2 个次要项（文档、配置），推荐 B。如果核心功能缺失，推荐 A。
   - 选项：
     A) 停止 — 在交付前实现缺失的项
     B) 仍然交付 — 推迟到后续处理（将在 Step 5.5 中创建 P1 TODO）
     C) 这些项被有意放弃 — 从范围中移除
   - 如果 A：停止。列出缺失项供用户实现。
   - 如果 B：继续。为每个 NOT DONE 项在 Step 5.5 中创建 P1 TODO，标注"从计划推迟：{plan file path}"。
   - 如果 C：继续。在 PR 正文中注明："有意放弃的计划项：{list}。"

2. **任何 UNVERIFIABLE 项**（静默缺口——diff 无法证实也无法证伪）。仅在 NOT DONE 已解决或不存在时触发。

   **逐项确认是强制的。** 不要使用单个 AskUserQuestion 来笼统确认所有 UNVERIFIABLE 项。笼统确认是 VAS-449 中暴露的故障模式（用户点击 A 却未打开任何文件）。相反：

   - 逐个遍历 UNVERIFIABLE 项。
   - 对每个项，使用 AskUserQuestion 并提供该项的*具体*手动检查（例如，"确认：`~/Development/domain-hq/docs/dashboard.md` 是否存在？"，而非"你是否检查了所有项？"）。
   - 每个项的选项：
     Y) 确认已完成 — 引用你验证的内容（自由文本，嵌入 PR 正文）
     N) 未完成 — 阻止交付；视为 NOT DONE 并重新进入第一优先级门控
     D) 有意放弃 — 在 PR 正文中注明："有意放弃的计划项：{item}"
   - 每项 RECOMMENDATION：如果项具体且易于验证，选 Y；如果是关键路径（认证、DNS、交付给其他仓库）且用户表现出犹豫，选 N。

   **退出条件：**
   - 任何 N：停止。暴露缺失项，建议在解决后重新运行 /ship。
   - 全部 Y 或 D：继续。在 PR 正文中嵌入 `## Plan Completion — Manual Verifications` 部分，列出每个 Y 项及用户的自由文本证据，以及每个 D 项标注"有意放弃"。

   **上限。** 如果 UNVERIFIABLE 项超过 5 个，首先以编号列表展示，然后询问用户是否希望 (1) 逐一确认，(2) 停止并缩减范围，或 (3) 明确接受笼统确认（附 VAS-449 故障模式警告）。默认和推荐选项是 (1)。

3. **仅有 PARTIAL 项（无 NOT DONE，无 UNVERIFIABLE）：** 继续在 PR 正文中添加备注。不阻塞。

4. **全部 DONE 或 CHANGED：** 通过。"计划完成度：通过——所有项已解决。"继续。

**未找到计划文件：** 完全跳过。"未检测到计划文件——跳过计划完成度审计。"

**包含在 PR 正文中（Step 8）：** 添加带有清单摘要的 `## Plan Completion` 部分。
>
> 分析完成后，在你的响应的**最后一行**输出一个 JSON 对象（之后不要有其他文本）：
> `{"total_items":N,"done":N,"changed":N,"deferred":N,"unverifiable":N,"summary":"<PR 正文用的 markdown 清单>"}`

**父流程处理：**

1. 将子代理输出的最后一行解析为 JSON。
2. 为 Step 20 指标存储 `done`、`deferred`、`unverifiable`；在 PR 正文中使用 `summary`。
3. 如果 `deferred > 0` 或 `unverifiable > 0` 且无用户覆盖，在继续之前通过相应的 AskUserQuestion 展示这些项（见上方门控逻辑优先级顺序）。
4. 将 `summary` 嵌入 PR 正文的 `## Plan Completion` 部分（Step 19）。如果 `unverifiable > 0` 且用户在 UNVERIFIABLE 门控中选择选项 A，还需嵌入 `## Plan Completion — Manual Verifications`，列出每个用户确认的项。

**如果子代理失败或返回无效 JSON：** 回退到内联运行审计（父流程处理相同的计划提取 + 分类逻辑）。如果内联回退也失败（例如，计划文件不可读、解析器错误），**不要**静默通过——将其作为显式 AskUserQuestion 暴露："计划完成度审计无法运行（{reason}）。选项：(A) 跳过审计并交付——在 PR 正文和 Step 20 指标中记录审计已跳过；(B) 停止并修复审计。"默认和推荐选项是 (B)。静默放行是 VAS-449 暴露的故障模式。

---

## Step 8.1：计划验证

使用 `/qa-only` 技能自动验证计划的测试/验证步骤。

### 1. 检查验证部分

使用 Step 8 中已发现的计划文件，查找验证部分。匹配以下任一标题：`## Verification`、`## Test plan`、`## Testing`、`## How to test`、`## Manual testing`，或包含验证风格项的任何部分（要访问的 URL、要目视检查的内容、要测试的交互）。

**如果未找到验证部分：** 跳过并提示"计划中未找到验证步骤——跳过自动验证。"
**如果 Step 8 中未找到计划文件：** 跳过（已处理）。

### 2. 检查运行中的开发服务器

在调用基于浏览器的验证之前，检查开发服务器是否可达：

```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:3000 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:8080 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:5173 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:4000 2>/dev/null || echo "NO_SERVER"
```

**如果 NO_SERVER：** 跳过并提示"未检测到开发服务器——跳过计划验证。部署后运行 /qa。"

### 3. 内联调用 /qa-only

从磁盘读取 `/qa-only` 技能：

```bash
cat ${CLAUDE_SKILL_DIR}/../qa-only/SKILL.md
```

**如果不可读：** 跳过并提示"无法加载 /qa-only——跳过计划验证。"

按 /qa-only 工作流执行，但做以下修改：
- **跳过前言**（已由 /ship 处理）
- **使用计划的验证部分作为主要测试输入**——将每个验证项视为测试用例
- **使用检测到的开发服务器 URL** 作为基础 URL
- **跳过修复循环**——这是 /ship 期间仅作报告的验证
- **以上计划中的验证项为上限**——不要扩展到通用站点 QA

### 4. 门控逻辑

- **所有验证项通过：** 静默继续。"计划验证：通过。"
- **任何失败：** 使用 AskUserQuestion：
  - 展示带有截图证据的失败项
  - RECOMMENDATION：如果失败表明功能损坏，选择 A。如果仅是外观问题，选择 B。
  - 选项：
    A) 在交付前修复失败项（功能问题推荐）
    B) 仍然交付——已知问题（外观问题可接受）
- **无验证部分 / 无服务器 / 技能不可读：** 跳过（非阻塞）。

### 5. 包含在 PR 正文中

在 PR 正文中添加 `## Verification Results` 部分（Step 19）：
- 如果验证已运行：结果摘要（N 通过，M 失败，K 跳过）
- 如果跳过：跳过原因（无计划、无服务器、无验证部分）

## Prior Learnings（先前经验）

搜索先前会话中的相关经验：

```bash
_CROSS_PROJ=$(~/.claude/skills/gstack/bin/gstack-config get cross_project_learnings 2>/dev/null || echo "unset")
echo "CROSS_PROJECT: $_CROSS_PROJ"
if [ "$_CROSS_PROJ" = "true" ]; then
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 --query "release ship version changelog merge pr" --cross-project 2>/dev/null || true
else
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 --query "release ship version changelog merge pr" 2>/dev/null || true
fi
```

如果 `CROSS_PROJECT` 为 `unset`（首次）：使用 AskUserQuestion：

> gstack 可以从此机器上的其他项目中搜索经验，寻找可能适用于此处的模式。这仅在本地进行（数据不离开你的机器）。
> 推荐独立开发者使用。如果你在多个客户端代码库上工作，且交叉污染是一个顾虑，请跳过。

选项：
- A) 启用跨项目经验搜索（推荐）
- B) 保持经验仅限当前项目范围

如果 A：运行 `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings true`
如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings false`

然后使用适当的标志重新运行搜索。

如果发现了经验，将其纳入你的分析。当审查发现与过去的经验匹配时，展示：

**"Prior learning applied: [key] (confidence N/10, from [date])"**

这使得经验积累可见。用户应当看到 gstack 正在随着时间的推移在代码库上变得更聪明。

## Step 8.2：范围漂移检测

在审查代码质量之前，先检查：**他们是否构建了被要求构建的内容——不多也不少？**

1. 读取 `TODOS.md`（如果存在）。读取 PR 描述（`gh pr view --json body --jq .body 2>/dev/null || true`）。
   读取提交消息（`git log origin/<base>..HEAD --oneline`）。
   **如果不存在 PR：** 依赖提交消息和 TODOS.md 来确定声明的意图——这是常见情况，因为 /review 在 /ship 创建 PR 之前运行。
2. 识别**声明的意图**——这个分支本应完成什么？
3. 运行 `DIFF_BASE=$(git merge-base origin/<base> HEAD) && git diff "$DIFF_BASE" --stat` 并将变更的文件与声明的意图进行比较。

4. 带着怀疑态度评估（如果可用，纳入前一步或相邻部分的计划完成度结果）：

   **SCOPE CREEP（范围蔓延）检测：**
   - 与意图无关的变更文件
   - 计划中未提及的新功能或重构
   - "趁我在这里顺便……"扩大爆炸半径的变更

   **MISSING REQUIREMENTS（遗漏需求）检测：**
   - TODOS.md/PR 描述中的需求未在 diff 中体现
   - 声明需求的测试覆盖缺口
   - 部分实现（已开始但未完成）

5. 输出（在主审查开始之前）：
   ```
   Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
   Intent: <1-line summary of what was requested>
   Delivered: <1-line summary of what the diff actually does>
   [如果漂移：列出每个越界变更]
   [如果遗漏：列出每个未解决的需求]
   ```

6. 这是**信息性的**——不阻塞审查。继续下一步。

---
---
