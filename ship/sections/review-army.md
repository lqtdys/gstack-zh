<!-- AUTO-GENERATED from review-army.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 步骤 9：着陆前审查

审查那些测试无法捕获的结构性问题。

1. 读取 `.claude/skills/review/checklist.md`。如果无法读取该文件，**停止**并报告错误。

2. 运行 `git diff origin/<base>` 以获取完整差异（针对新拉取的基础分支限定功能变更范围）。

3. 分两轮应用审查清单：
   - **第 1 轮（关键）：** SQL 与数据安全、LLM 输出信任边界
   - **第 2 轮（信息性）：** 所有剩余类别

## 置信度校准

每个发现必须包含一个置信度分数（1-10）：

| 分数 | 含义 | 显示规则 |
|------|------|----------|
| 9-10 | 通过阅读特定代码验证。已证明存在具体漏洞或利用方式。 | 正常显示 |
| 7-8 | 高置信度模式匹配。极有可能正确。 | 正常显示 |
| 5-6 | 中等。可能是误报。 | 附带警告显示："中等置信度，请验证这确实是问题" |
| 3-4 | 低置信度。模式可疑但可能没问题。 | 从主报告中抑制。仅在附录中包含。 |
| 1-2 | 猜测。 | 仅在严重度为 P0 时报告。 |

**发现格式：**

\`[严重度] (置信度: N/10) 文件:行 — 描述\`

示例：
\`[P1] (置信度: 9/10) app/models/user.rb:42 — where 子句中通过字符串插值造成 SQL 注入\`
\`[P2] (置信度: 5/10) app/controllers/api/v1/users_controller.rb:18 — 可能存在 N+1 查询，请用生产日志验证\`

### 输出前验证关卡（#1539 — 消除"字段不存在"误报类）

在任何发现被提升报告之前，该关卡要求：

1. **引用触发该发现的具体代码行** — 文件:行号以及触发该发现的代码行的逐字文本。如果发现是"模型 Y 上字段 X 不存在"，则引用类 Y 中该字段应存在的行。如果"dict.get() 可能返回 None"，则引用 dict 初始化代码。如果"条件竞争在 A 和 B 之间"，则同时引用 A 和 B。

2. **如果无法引用触发该发现的代码行，则该发现未经验证。**将其置信度强制设为 4-5（从主报告中抑制）。它仍会进入附录以便审查校准，但用户不会在关键轮输出中看到它。不要通过编造推测性置信度 7+ 来绕过此限制 — 这会破坏关卡。

**框架元提示：**当符号由框架元类、描述符、ORM Meta 内部类或迁移历史生成时（Django `Meta`、Rails `has_many`/`scope`、SQLAlchemy `relationship`/`Column`、TypeORM 装饰器、Sequelize `init`/`belongsTo`、Prisma 生成的客户端），引用元构造（`Meta` 块、迁移、装饰器、模式文件）而不是期望在类主体中找到字面名称。验证是"我阅读了创建此符号的源文件"，而不是"我用 grep 搜索了名称但没找到"。更深层的框架感知验证（模型自省、迁移历史感知检查、ORM 方言检测）故意不在该轻量级关卡的范围内 — 请参阅已推迟的设计文档 `~/.gstack-dev/plans/1539-framework-aware-review.md`。

该关卡消除的误报类（针对 Django Sprint 2.5 #1539 测量）：

| 误报类 | 关卡捕获原因 |
|--------|-------------|
| "字段不存在于模型上" | 需要引用模型类主体或 Meta；字段的缺失变得显而易见 |
| "dict.get() 可能为 None" | 需要引用 dict 初始化（例如 Django 表单的 `cleaned_data` 是 `{}` 初始化的） |
| "save() 可能丢失字段" | 需要引用 ORM 签名或模型定义 |
| "update_fields 可能遗漏 X" | 需要引用字段集；如果 X 不存在，误报是自明的 |

**校准学习：**如果你以置信度 < 7 报告了一个发现并用户确认它确实是问题，那就是校准事件。你的初始置信度太低了。将修正后的模式记录为学习，以便未来审查以更高置信度捕获它。

## 设计审查（条件性，差异范围内）

检查差异是否触及前端文件，使用 `gstack-diff-scope`：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

**如果 `SCOPE_FRONTEND=false`：**静默跳过设计审查。无输出。

**如果 `SCOPE_FRONTEND=true`：**

1. **检查 DESIGN.md。**如果 `DESIGN.md` 或 `design-system.md` 存在于仓库根目录，则读取它。所有发现都根据它进行校准 — DESIGN.md 中认可的模式不会标记。如果没找到，则使用通用设计原则。

2. **读取 `.claude/skills/review/design-checklist.md`。**如果无法读取该文件，则跳过设计审查并注明："未找到设计清单 — 跳过设计审查。"

3. **读取每个变更的前端文件**（完整文件，不只是差异块）。前端文件由清单中列出的模式识别。

4. **将设计清单应用于变更的文件。**对于每个项目：
   - **[HIGH] 机械性 CSS 修复**（`outline: none`、`!important`、`font-size < 16px`）：分类为自动修复
   - **[HIGH/MEDIUM] 需要设计判断**：分类为询问
   - **[LOW] 基于意图的检测**：呈现为"可能 — 请视觉验证或运行 /design-review"

5. **将发现包含**在审查输出的"设计审查"标题下，遵循清单中的输出格式。设计发现与代码审查发现合并到同一个首先修复流中。

6. **将结果记录**到审查就绪仪表板：

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"design-review-lite","timestamp":"时间戳","status":"状态","findings":N,"auto_fixed":M,"commit":"提交"}'
```

替换：TIMESTAMP = ISO 8601 时间戳，STATUS = 0 个发现时为"clean"或"issues_found"，N = 发现总数，M = 自动修复数，COMMIT = `git rev-parse --short HEAD` 的输出。

7. **Codex 设计语音**（可选，自动如果可用）：

```bash
command -v codex >/dev/null 2>&1 && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

如果 Codex 可用，在差异上运行轻量设计检查：

```bash
TMPERR_DRL=$(mktemp /tmp/codex-drl-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo "ERROR: not in a git repo" >&2; exit 1; }
codex exec "审查这个分支上的 git diff。运行 7 个石蕊测试（YES/NO 每个）：1. 品牌/产品在第一屏中是否一眼认出？2. 存在一个强烈的视觉锚点？3. 仅通过扫描标题就能理解页面？4. 每个部分是否只有一个任务？5. 卡片是否真的必要？6. 运动是否改善层次或氛围？7. 如果移除所有装饰阴影，设计是否感觉高级？标记任何硬拒绝：1. 通用 SaaS 卡片网格作为第一印象 2. 带有弱品牌的美丽图像 3. 没有清晰行动的强烈标题 4. 文本后面繁忙图像 5. 部分重复相同的情绪陈述 6. 没有叙事目的的轮播 7. 由堆叠卡片而不是布局制成的应用 UI 仅 5 个最重要的设计发现。引用 文件:line。" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached < /dev/null 2>"$TMPERR_DRL"
```

使用 5 分钟超时（`timeout: 300000`）。命令完成后，读取 stderr：
```bash
cat "$TMPERR_DRL" && rm -f "$TMPERR_DRL"
```

**错误处理：**所有错误都是非阻塞的。在认证失败、超时或空响应时 — 跳过并简要注明继续。

在 `CODEX (design):` 标题下呈现 Codex 输出，与上面的清单发现合并。

   包括任何设计发现与代码审查发现一起。它们遵循下面的相同首先修复流。

## 步骤 9.1：审查军团 — 专家派遣

### 检测技术栈和范围

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null) || true
# 检测技术栈以获取专家上下文
STACK=""
[ -f Gemfile ] && STACK="${STACK}ruby "
[ -f package.json ] && STACK="${STACK}node "
[ -f requirements.txt ] || [ -f pyproject.toml ] && STACK="${STACK}python "
[ -f go.mod ] && STACK="${STACK}go "
[ -f Cargo.toml ] && STACK="${STACK}rust "
echo "STACK: ${STACK:-unknown}"
DIFF_BASE=$(git merge-base origin/<base> HEAD)
DIFF_INS=$(git diff "$DIFF_BASE" --stat | tail -1 | grep -oE '[0-9]+ insertion' | grep -oE '[0-9]+' || echo "0")
DIFF_DEL=$(git diff "$DIFF_BASE" --stat | tail -1 | grep -oE '[0-9]+ deletion' | grep -oE '[0-9]+' || echo "0")
DIFF_LINES=$((DIFF_INS + DIFF_DEL))
echo "DIFF_LINES: $DIFF_LINES"
# 检测测试框架以生成专家测试存根
TEST_FW=""
{ [ -f jest.config.ts ] || [ -f jest.config.js ]; } && TEST_FW="jest"
[ -f vitest.config.ts ] && TEST_FW="vitest"
{ [ -f spec/spec_helper.rb ] || [ -f .rspec ]; } && TEST_FW="rspec"
{ [ -f pytest.ini ] || [ -f conftest.py ]; } && TEST_FW="pytest"
[ -f go.mod ] && TEST_FW="go-test"
echo "TEST_FW: ${TEST_FW:-unknown}"
```

### 读取专家命中率（自适应门禁）

```bash
~/.claude/skills/gstack/bin/gstack-specialist-stats 2>/dev/null || true
```

### 选择专家

基于上面的范围信号，选择要派遣哪些专家。

**始终开启（在每次 50+ 变更行的审查中派遣）：**
1. **测试** — 读取 `~/.claude/skills/gstack/review/specialists/testing.md`
2. **可维护性** — 读取 `~/.claude/skills/gstack/review/specialists/maintainability.md`

**如果 DIFF_LINES < 50：**跳过所有专家。输出："小差异（$DIFF_LINES 行）— 跳过专家。"继续到首先修复流（项目 4）。

**条件性（如果匹配的范围信号为真则派遣）：**
3. **安全** — 如果 SCOPE_AUTH=true，或 SCOPE_BACKEND=true 且 DIFF_LINES > 100。读取 `~/.claude/skills/gstack/review/specialists/security.md`
4. **性能** — 如果 SCOPE_BACKEND=true 或 SCOPE_FRONTEND=true。读取 `~/.claude/skills/gstack/review/specialists/performance.md`
5. **数据迁移** — 如果 SCOPE_MIGRATIONS=true。读取 `~/.claude/skills/gstack/review/specialists/data-migration.md`
6. **API 合约** — 如果 SCOPE_API=true。读取 `~/.claude/skills/gstack/review/specialists/api-contract.md`
7. **设计** — 如果 SCOPE_FRONTEND=true。使用现有的设计审查清单 `~/.claude/skills/gstack/review/design-checklist.md`

### 自适应门禁

基于范围选择后，基于专家命中率应用自适应门禁：

对于通过范围门禁的每个条件性专家，检查上面的 `gstack-specialist-stats` 输出：
- 如果标记为 `[GATE_CANDIDATE]`（10+ 次派遣中 0 发现）：跳过。输出："[expert] 自动门禁（N 次审查中 0 发现）。"
- 如果标记为 `[NEVER_GATE]`：总是派遣无论命中率如何。安全和数据迁移是保险单专家 — 即使沉默时也应该运行。

**强制标志：**如果用户的提示包含 `--security`、`--performance`、`--testing`、`--maintainability`、`--data-migration`、`--api-contract`、`--design` 或 `--all-specialists`，强制包含该专家无论门禁如何。

注意哪些专家被选择、门控和跳过。输出选择：
"派遣 N 个专家：[名称]。跳过：[名称]（未检测到范围）。门控：[名称]（N+ 次审查中 0 发现）。"

---

### 并行派遣专家

对于每个选定的专家，通过 Agent 工具启动一个独立的子代理。
**在单个消息中启动所有选定的专家**（多个 Agent 工具调用）
以便它们并行运行。每个子代理有新上下文 — 没有先前的审查偏差。

**每个专家子代理提示：**

为每个专家构建提示。提示包括：

1. 专家清单内容（你已经读取了上面的文件）
2. 技术栈上下文："这是一个 {STACK} 项目。"
3. 过去的学习（如果存在）：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-search --type pitfall --query "{specialist domain}" --limit 5 2>/dev/null || true
```

如果找到学习，包括它们："过去的领域学习：{learnings}"

4. 指示：

"你是一个专家代码审查员。阅读下面的清单，然后运行
`DIFF_BASE=$(git merge-base origin/<base> HEAD) && git diff "$DIFF_BASE"` 以获取完整差异。将清单应用于差异。

对于每个发现，在自己的行上输出一个JSON对象：
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"category","summary":"description","fix":"recommended fix","fingerprint":"path:line:category","specialist":"name"}

必填字段：severity, confidence, path, category, summary, specialist。
可选：line, fix, fingerprint, evidence, test_stub。

如果可以写一个会捕获此问题的测试，将其包含在 `test_stub` 字段中。
使用检测到的测试框架（{TEST_FW}）。写一个最小的骨架 — 描述/it/测试
块带有清晰的意图。对于架构或纯设计发现跳过 test_stub。

如果没有发现：输出 `NO FINDINGS` 什么也不输出。
不要输出任何其他东西 — 没有前言，没有摘要，没有评论。

技术栈上下文：{STACK}
过去学习：{learnings 或 'none'}

清单：
{清单内容}"

**子代理配置：**
- 使用 `subagent_type: "general-purpose"`
- 不要使用 `run_in_background` — 所有专家必须在合并前完成
- 如果任何专家子代理失败或超时，记录失败并继续成功专家的结果。专家是可加的 — 部分结果比没有结果好。

---

### 步骤 9.2：收集和合并发现

在所有专家子代理完成后，收集它们的输出。

**解析发现：**
对于每个专家的输出：
1. 如果输出是 "NO FINDINGS" — 跳过，此专家发现什么都没有
2. 否则，将每行解析为 JSON 对象。跳过不是有效 JSON 的行。
3. 将所有解析的发现收集到一个列表中，用它们专家的名称标记。

**指纹和去重：**
对于每个发现，计算其指纹：
- 如果 `fingerprint` 字段存在，使用它
- 否则：`{path}:{line}:{category}`（如果 line 存在）或 `{path}:{category}`

按指纹分组发现。对于共享相同指纹的发现：
- 保留最高置信度分数的发现
- 标记："MULTI-SPECIALIST CONFIRMED（{specialist1} + {specialist2}）"
- 将置信度 +1（上限为 10）
- 在输出中注明确认的专家

**应用置信度门禁：**
- 置信度 7+：正常显示在发现输出中
- 置信度 5-6：显示附带"中等置信度 — 请验证这实际上是问题"
- 置信度 3-4：移到附录（从主发现中抑制）
- 置信度 1-2：完全抑制

**计算 PR 质量分数：**
合并后，计算质量分数：
`quality_score = max(0, 10 - (critical_count * 2 + informational_count * 0.5))`
上限 10。在末尾的审查结果中记录此。

**输出合并的发现：**
以当前审查的相同格式呈现合并的发现：

```
SPECIALIST REVIEW: N 个发现（X 关键，Y 信息性）来自 Z 个专家

[对于每个发现，按顺序：先是 CRITICAL，然后是 INFORMATIONAL，按置信度降序排列]
[SEVERITY] (confidence: N/10, specialist: name) path:line — summary
  Fix: recommended fix
  [如果 MULTI-SPECIALIST CONFIRMED：显示确认注记]

PR Quality Score: X/10
```

这些发现流入首先修复流（项目 4）以及清单轮（步骤 9）。
首先修复启发式方法同样适用 — 专家发现遵循相同的 AUTO-FIX vs ASK 分类。

**编译每专家统计：**
合并发现后，编译一个 `specialists` 对象用于审查日志持久化。
对于每个专家（testing, maintainability, security, performance, data-migration, api-contract, design, red-team）：
- 如果 dispatched：`{"dispatched": true, "findings": N, "critical": N, "informational": N}`
- 如果 scope 跳过：`{"dispatched": false, "reason": "scope"}`
- 如果 gated：`{"dispatched": false, "reason": "gated"}`
- 如果不适用（例如 red-team 未激活）：从对象中省略

包含设计专家即使它使用 `design-checklist.md` 而不是专家模式文件。
记住这些统计 — 你将需要它们在步骤 5.8 的审查日志条目。

---

### 红队派遣（条件性）

**激活：**仅当 DIFF_LINES > 200 或任何专家产生了关键发现。

如果激活，通过 Agent 工具再派遣一个子代理（前台，不是后台）。

红队子代理接收：
1. 来自 `~/.claude/skills/gstack/review/specialists/red-team.md` 的红队清单
2. 来自步骤 9.2 的合并专家发现（所以它知道已经捕获了什么）
3. git diff 命令

提示："你是一个红队审查员。代码已经由 N 个专家审查过他们找到了以下问题：{merged findings summary}。你的工作是找到他们**遗漏的**。阅读清单，运行 `DIFF_BASE=$(git merge-base origin/<base> HEAD) && git diff "$DIFF_BASE"`，并寻找差距。
输出发现作为 JSON 对象（与专家相同的模式）。专注于跨切面关注点、集成边界问题和专家清单不覆盖的故障模式。"

如果红队发现其他问题，将它们合并到发现列表之前首先修复流（项目 4）。红队发现标记为 `"specialist":"red-team"`。

如果红队返回 NO FINDINGS，注明："红队审查：没有发现其他问题。"
如果红队子代理失败或超时，静默跳过并继续。

### 步骤 9.3：跨审查发现去重

在分类发现之前，检查是否有任何发现被用户在同一分支的先前审查中跳过。

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

解析输出：仅在 `---CONFIG---` 之前的行是 JSONL 条目（输出还包含 `---CONFIG---` 和 `---HEAD---` 页脚部分，这些不是 JSONL — 忽略那些）。

对于每个具有 `findings` 数组的 JSONL 条目：
1. 收集所有指纹其中 `action: "skipped"`
2. 注记该条目的 `commit` 字段

如果跳过的指纹存在，获取自该审查以来更改的文件列表：

```bash
git diff --name-only <prior-review-commit> HEAD
```

对于每个当前发现（来自清单轮（步骤 9）和专家审查（步骤 9.1-9.2）），检查：
- 它的指纹是否匹配先前跳过的发现？
- 发现的文件路径是否**不在**更改文件集中？

如果两个条件都为真：抑制该发现。它被有意跳过的且相关代码已没有改变。

输出："从先前审查抑制 N 个发现（先前被用户跳过）"

**只抑制 `skipped` 发现 — 永远不要 `fixed` 或 `auto-fixed`**（那些可能回退和应该重新检查）。

如果不存在先前审查或没有有 `findings` 数组的，静默跳过此步骤。

输出摘要头部：`Pre-Landing Review: N issues (X critical, Y informational)`

4. **将每个发现从清单轮和专家审查（步骤 9.1-步骤 9.2）分类为 AUTO-FIX 或 ASK** per 首先修复启发式方法在 checklist.md 中。关键发现倾向于 ASK；信息性发现倾向于 AUTO-FIX。

5. **自动修复所有 AUTO-FIX 项目。**应用每个修复。每个修复输出一行：
   `[AUTO-FIXED] [file:line] Problem → what you did`

6. **如果 ASK 项目仍然存在，**在一个 AskUserQuestion 中呈现它们：
   - 列出每个带有编号、严重度、问题、推荐修复
   - 每项目选项：A) 修复  B) 跳过
   - 总体推荐
   - 如果 3 个或更少 ASK 项目，你可以使用单个 AskUserQuestion 调用

7. **在所有修复（自动 + 用户批准）后：**
   - 如果应用了任何修复：通过名称提交修复的文件（`git add <fixed-files> && git commit -m "fix: pre-landing review fixes"`），然后**停止**并告诉用户运行 `/ship` 再次重新测试。
   - 如果未应用修复（所有 ASK 项目跳过，或没问题发现）：继续到步骤 12。

8. 输出摘要：`Pre-Landing Review: N issues — M auto-fixed, K asked (J fixed, L skipped)`

   如果没问题发现：`Pre-Landing Review: No issues found.`

9. 将审查结果持久化到审查日志：
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"review","timestamp":"TIMESTAMP","status":"STATUS","issues_found":N,"critical":N,"informational":N,"quality_score":SCORE,"specialists":SPECIALISTS_JSON,"findings":FINDINGS_JSON,"commit":"'"$(git rev-parse --short HEAD)'","via":"ship"}'
```
替换 TIMESTAMP（ISO 8601）、STATUS（如果没有问题则为"clean"，否则为"issues_found"），
和来自上面总结计数的 N 值。`via:"ship"` 区分于独立 `/review` 运行。
- `quality_score` = 步骤 9.2 计算的 PR 质量分数（例如 7.5）。如果专家被跳过（小差异），使用 `10.0`
- `specialists` = 步骤 9.2 编译的每专家统计对象。每个被考虑的专家都获得一个条目：`{"dispatched":true/false,"findings":N,"critical":N,"informational":N}` 如果 dispatched，或 `{"dispatched":false,"reason":"scope|gated"}` 如果跳过。示例：`{"testing":{"dispatched":true,"findings":2,"critical":0,"informational":2},"security":{"dispatched":false,"reason":"scope"}}`
- `findings` = 每发现记录的数组。对于每个发现（来自清单轮和专家），包括：`{"fingerprint":"path:line:category","severity":"CRITICAL|INFORMATIONAL","action":"ACTION"}`。ACTION 是 `"auto-fixed"`、`"fixed"`（用户批准）或 `"skipped"`（用户选择 Skip）。

保存审查输出 — 它进入步骤 19 的 PR 正文 |

---
