<!-- AUTO-GENERATED from release-body.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs (zh-CN variant) -->
## 步骤 2：按文件文档审计

读取每个文档文件，并对照差异进行交叉引用。使用这些通用启发式方法
（适配你所在的项目 — 这些不是 gstack 特有的）：

**README.md：**
- 它是否描述了差异中可见的所有功能和能力？
- 安装/设置说明是否与更改一致？
- 示例、演示和用法描述是否仍然有效？
- 故障排除步骤是否仍然准确？

**ARCHITECTURE.md：**
- ASCII 图表和组件描述是否与当前代码匹配？
- 设计决策和“为什么”的解释是否仍然准确？
- 保守一些 — 仅更新被差异明确矛盾的内容。架构文档描述不太可能频繁更改的内容。

**CONTRIBUTING.md — 新贡献者冒烟测试：**
- 像一个全新贡献者一样走一遍设置说明。
- 列出的命令是否准确？每个步骤都会成功吗？
- 测试分层描述是否与当前测试基础设施匹配？
- 工作流描述（开发设置、操作学习等）是否最新？
- 标记任何会失败或让首次贡献者困惑的地方。

**CLAUDE.md / 项目指令：**
- 项目结构部分是否与实际文件树匹配？
- 列出的命令和脚本是否准确？
- 构建/测试说明是否与 package.json（或等效文件）中的内容匹配？

**任何其他 .md 文件：**
- 读取文件，确定其目的和受众。
- 对照差异进行交叉引用，检查它是否与新功能矛盾。

对于每个文件，将所需更新分类为：
- **自动更新** — 差异明确需要的事实更正：向表中添加项目、更新文件路径、修复计数、更新项目结构树。
- **询问用户** — 叙述性更改、删除部分、安全模型更改、大重写（单个部分中超过约 10 行）、相关性模糊、添加全新部分。

---

## 步骤 3：应用自动更新

使用编辑工具直接进行所有清晰、事实性的更新。

对于每个修改过的文件，输出一行描述**具体更改内容**的摘要 — 不仅仅是"更新了 README.md"，而是"README.md：将 /new-skill 添加到技能表，将技能计数从 9 更新到 10。"

**切勿自动更新：**
- README 介绍或项目定位
- ARCHITETURE 理念或设计原理
- 安全模型描述
- 不要从任何文档中删除整个部分

---

## 步骤 4：询问有风险/可疑的更改

对于步骤 2 中识别的每个有风险或可疑的更新，使用 AskUserQuestion，并提供：
- 上下文：项目名称、分支、哪个文档文件、我们在审查什么
- 具体的文档决策
- `RECOMMENDATION: Choose [X] because [一句话原因]`
- 包含 C) 跳过 — 保持原状的选项

在每个答案后立即应用批准的更改。

---

## 步骤 5：CHANGELOG 语音润色

**关键 — 切勿覆盖 CHANGELOG 条目。**

此步骤润色语音。它不重写、替换或重新生成 CHANGELOG 内容。

发生了一起真实事件，代理在应该保留现有 CHANGELOG 条目时替换了它们。此技能永远不得这样做。

**规则：**
1. 首先阅读整个 CHANGELOG.md。理解已存在的内容。
2. 仅修改现有条目内的措辞。永远不要删除、重新排序或替换条目。
3. 永远不要从头重新生成 CHANGELOG 条目。该条目由 `/ship` 从实际差异和提交历史编写。它是真相的来源。你在润色散文，而不是重写历史。
4. 如果某个条目看起来错误或不完整，使用 AskUserQuestion — 不要静默修复。
5. 使用带有精确 `old_string` 匹配的 Edit 工具 — 永远不要使用 Write 覆盖 CHANGELOG.md。

**如果此分支未修改 CHANGELOG：** 跳过此步骤。

**如果此分支修改了 CHANGELOG**，审查条目的语音：

- **销售测试（Diataxis 评分标准）：** 为每个 CHANGELOG 条目评分 0-3：
  - **1 分** — 回答"发生了什么？"（参考：命名了功能/修复）
  - **1 分** — 为什么我应该关心？（解释：用户影响，消除的痛苦）
  - **1 分** — 回答"我如何使用它？"（how-to：命令、标志或文档链接）
  - 得分 <2 的条目需要重写。得分 3 的条目是黄金标准。
- 以用户现在可以**做**的事情开头 — 而不是实现细节。
- "你现在可以..." 而不是 "重构了..."
- 标记并重写任何读起来像提交消息的条目。
- 内部/贡献者更改属于单独的 "### For contributors" 部分。
- 自动修复小的语音调整。如果重写会改变含义，使用 AskUserQuestion。

---

## 步骤 6：跨文档一致性 & 可发现性检查

在每个文件单独审计后，进行跨文档一致性检查：

1. README 的功能/能力列表是否与 CLAUDE.md（或项目指令）中描述的一致？
2. ARCHITECTURE 的组件列表是否与 CONTRIBUTING 的项目结构描述匹配？
3. CHANGELOG 的最新版本是否与 VERSION 文件匹配？
4. **可发现性：** 每个文档文件是否都可以从 README.md 或 CLAUDE.md 访问？如果 ARCHITECTURE.md 存在，但 README 和 CLAUDE.md 都没有链接到它，则标记它。每个文档都应该可以从两个入口点文件之一发现。
5. 标记文档之间的任何矛盾。自动修复清晰的事实不一致（例如版本不匹配）。对于叙述性矛盾使用 AskUserQuestion。

---

## 步骤 7：TODOS.md 清理

这是对 `/ship` 步骤 5.5 的补充的第二遍。如果可用，阅读 `review/TODOS-format.md` 了解规范的 TODO 项目格式。

如果 TODOS.md 不存在，跳过此步骤。

1. **已完成但未标记的项目：** 对照差异交叉引用开放的 TODO 项目。如果 TODO 被此分支中的更改明确完成，将其移动到已完成部分，并附上 `**Completed:** vX.Y.Z.W (YYYY-MM-DD)`。保守一些 — 仅标记有清晰差异证据的项目。

2. **需要描述更新的项目：** 如果 TODO 引用了被显著更改的文件或组件，其描述可能已过时。使用 AskUserQuestion 确认 TODO 是否应更新、完成或保持原状。

3. **新的延期工作：** 检查差异中的 `TODO`、`FIXME`、`HACK` 和 `XXX` 注释。对于每个代表有意义延期工作（不是简单的内联注释）的注释，使用 AskUserQuestion 询问是否应将其捕获到 TODOS.md 中。

---

## 步骤 8：版本号提升问题

**关键 — 切勿未经询问就提升版本号。**

1. **如果 VERSION 不存在：** 静默跳过。

2. 检查 VERSION 是否已在此分支上修改：

```bash
git diff <base>...HEAD -- VERSION
```

3. **如果 VERSION 未提升：** 使用 AskUserQuestion：
   - RECOMMENDATION: Choose C (Skip) 因为纯文档更改很少需要版本号提升
   - A) 提升 PATCH (X.Y.Z+1) — 如果文档更改与代码更改一起发布
   - B) 提升 MINOR (X.Y+1.0) — 如果这是一个重要的独立版本
   - C) 跳过 — 不需要版本号提升

4. **如果 VERSION 已提升：** 不要静默跳过。相反，检查提升是否仍然覆盖此分支上的所有更改范围：

   a. 阅读当前 VERSION 的 CHANGELOG 条目。它描述了哪些功能？
   b. 读取完整差异（`git diff <base>...HEAD --stat` 和 `git diff <base>...HEAD --name-only`）。是否存在未在当前版本的 CHANGELOG 条目中提及的重要更改（新功能、新技能、新命令、重大重构）？
   c. **如果 CHANGELOG 条目涵盖了一切：** 跳过 — 输出"VERSION: Already bumped to vX.Y.Z, covers all changes."
   d. **如果有重要的未覆盖更改：** 使用 AskUserQuestion 解释当前版本涵盖的内容与新增内容，并询问：
      - RECOMMENDATION: Choose A 因为新更改值得它们自己的版本
      - A) 提升到下一个 patch (X.Y.Z+1) — 给新更改它们自己的版本
      - B) 保持当前版本 — 将新更改添加到现有 CHANGELOG 条目
      - C) 跳过 — 保持版本不变，稍后处理

   关键洞察：为"功能 A"设置的版本号提升不应静默吸收"功能 B"，如果功能 B 足够重要，值得它自己的版本条目。

---

## 步骤 9：提交和输出

**空检查优先：** 运行 `git status`（切勿使用 `-uall`）。如果任何之前的步骤都没有修改任何文档文件，输出"All documentation is up to date."并退出而不提交。

**提交：**

1. 按名称暂存修改的文档文件（切勿使用 `git add -A` 或 `git add .`）。
2. 创建一个提交：

```bash
git commit -m "$(cat <<'EOF'
docs: update project documentation for vX.Y.Z.W

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

3. 推送到当前分支：

```bash
git push
```

**PR/MR 正文更新（幂等的，竞态安全）：**

1. 将现有 PR/MR 正文读取到 PID 唯一的 tempfile 中（使用步骤 0 中检测到的平台）：

**如果是 GitHub：**
```bash
gh pr view --json body -q .body > /tmp/gstack-pr-body-$$.md
```

**如果是 GitLab：**
```bash
glab mr view -F json 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin).get('description',''))" > /tmp/gstack-pr-body-$$.md
```

2. 如果 tempfile 已包含 `## Documentation` 部分，用更新的内容替换该部分。如果它不包含，在末尾追加一个 `## Documentation` 部分。

3. 文档部分应包括：

   a. **文档差异预览** — 对于每个修改的文件，描述具体更改了什么（例如，"README.md：将 /document-release 添加到技能表，将技能计数从 9 更新到 10"）。

   b. **文档债务** — 如果步骤 1.5 的覆盖图发现了差距，追加一个 `### Document Debt` 部分，列出：
      - 关键差距：新的公共表面，零文档覆盖
      - 常见差距：仅有参考覆盖的功能（无 how-to 或教程）
      - 过时图表：实体名称偏离代码的架构图
      - 每个项目应包含缺少什么的单行描述以及哪个 Diataxis 象限会填补它（例如，"⚠️ `/new-skill` — 在 AGENTS.md 中有参考但在 README 中无 how-to 示例"）。

   如果存在任何文档债务项目，建议向 PR 添加 `docs-debt` 标签。

4. 在写入点进行编辑前扫描，然后将更新后的正文写回。正文已经在 tempfile（`/tmp/gstack-pr-body-$$.md`）中；在编辑之前扫描该文件，以便扫描的字节就是发送的字节：

```bash
REDACT_VIS=$(~/.claude/skills/gstack/bin/gstack-config get redact_repo_visibility 2>/dev/null)
[ -z "$REDACT_VIS" ] && REDACT_VIS=$(gh repo view --json visibility -q .visibility 2>/dev/null | tr 'A-Z' 'a-z')
~/.claude/skills/gstack/bin/gstack-redact --from-file /tmp/gstack-pr-body-$$.md --repo-visibility "${REDACT_VIS:-unknown}" --json
# exit 3 (HIGH) → 不要编辑，轮换+编辑；exit 2 (MEDIUM) → 按发现确认。
```

**如果是 GitHub：**
```bash
gh pr edit --body-file /tmp/gstack-pr-body-$$.md
```

**如果是 GitLab：**
使用 Read 工具读取 `/tmp/gstack-pr-body-$$.md` 的内容，然后通过 heredoc 将其传递给 `glab mr update` 以避免 shell 元字符问题：
```bash
glab mr update -d "$(cat <<'MRBODY'
<将文件内容粘贴在此>
MRBODY
)"
```

5. 清理 tempfile：

```bash
rm -f /tmp/gstack-pr-body-$$.md
```

6. 如果 `gh pr view` / `glab mr view` 失败（没有 PR/MR 存在）：跳过并提示"No PR/MR found — skipping body update."
7. 如果 `gh pr edit` / `glab mr update` 失败：警告"Could not update PR/MR body — documentation changes are in the commit."并继续。

**PR/MR 标题同步（幂等的，始终开启）：**

PR 标题必须始终以 `v<VERSION>` 开头 — 与 `/ship` 相同的规则。如果步骤 8 在 `/ship` 已经创建 PR 后提升了版本，标题现在已过时。此子步骤修复它。

1. 读取当前 VERSION：

```bash
V=$(cat VERSION 2>/dev/null | tr -d '[:space:]')
```

如果 `VERSION` 不存在或为空，完全跳过此子步骤。

2. 读取当前 PR/MR 标题：

**如果是 GitHub：**
```bash
CURRENT_TITLE=$(gh pr view --json title -q .title 2>/dev/null || true)
```

**如果是 GitLab：**
```bash
CURRENT_TITLE=$(glab mr view -F json 2>/dev/null | jq -r .title 2>/dev/null || true)
```

如果 `CURRENT_TITLE` 为空（没有打开的 PR/MR），跳过并提示"No PR/MR found — skipping title sync."

3. 使用共享助手计算修正后的标题（单一真相来源 — 与 `/ship` 使用的相同）：

```bash
NEW_TITLE=$(~/.claude/skills/gstack/bin/gstack-pr-title-rewrite.sh "$V" "$CURRENT_TITLE")
```

助手处理三种情况：标题已经正确（无操作），标题有不同的 `v<X.Y.Z.W>` 前缀（替换它），或标题没有版本前缀（在前面添加一个）。

4. 如果 `NEW_TITLE` 与 `CURRENT_TITLE` 不同，更新它：

**如果是 GitHub：**
```bash
gh pr edit --title "$NEW_TITLE"
```

**如果是 GitLab：**
```bash
glab mr update -t "$NEW_TITLE"
```

5. 如果编辑命令失败：警告"Could not update PR/MR title — documentation changes are still in the commit."并继续。不要因标题同步失败而阻塞。

**结构化文档健康摘要（最终输出）：**

输出一个可扫描的摘要，显示每个文档文件的状态：

```
Documentation health:
  README.md       [status] ([details])
  ARCHITECTURE.md [status] ([details])
  CONTRIBUTING.md [status] ([details])
  CHANGELOG.md    [status] ([details])
  TODOS.md        [status] ([details])
  VERSION         [status] ([details])
```

其中 status 是以下之一：
- Updated — 描述更改了什么
- Current — 无需更改
- Voice polished — 调整了措辞
- Not bumped — 用户选择跳过
- Already bumped — 版本由 /ship 设置
- Skipped — 文件不存在

如果步骤 1.5 的覆盖图发现了任何差距，追加：

```
Documentation coverage:
  [entity]         [reference] [how-to] [tutorial] [explanation]
  /new-skill       ✅          ❌       ❌         ❌
  --new-flag       ✅          ✅       ❌         ❌

Diagram drift:
  ARCHITECTURE.md: "FooProcessor" renamed to "BarProcessor" in code — diagram may be stale
```

如果所有覆盖都完整且没有图表漂移，输出："Coverage: all shipped features have adequate documentation."

---

## Codex 文档审查（默认开启）

在上述文档更新写入后，运行一个独立的跨模型检查，对照实际发布的内容检查文档。这是 /document-release 的标准部分，不是可选的。用户仅在明确要求时才关闭它（`gstack-config set codex_reviews disabled`）。

**预检 — 决定文档审查是否及如何运行：**

```bash
# Codex 预检：一个块（此处源的函数不会持久化到后面的块）。
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

分支基于回显的 `CODEX_MODE`：
- **`disabled`** — 用户关闭了 Codex 审查（`codex_reviews=disabled`）。完全跳过此部分；不要回退到 Claude 子代理 — 禁用意味着无额外审查步骤。打印："Codex review skipped (codex_reviews disabled). Re-enable: `gstack-config set codex_reviews enabled`."
- **`not_installed`** — Codex CLI 不存在。打印："Codex not installed — using Claude subagent. Install for cross-model coverage: `npm install -g @openai/codex`."回退到 Claude 子代理路径。
- **`not_authed`** — 已安装但无凭据。打印："Codex installed but not authenticated — using Claude subagent. Run `codex login` or set `$CODEX_API_KEY`."回退到 Claude 子代理路径。
- **`ready`** — 运行下面的 Codex 检查。

当模式为 `ready`、`not_installed` 或 `not_authed` 时，打印一行以保持关闭开关可发现："Running the Codex doc review automatically (standard step). Disable: `gstack-config set codex_reviews disabled`."

**确定发布差异范围（D3 — 重用该方法，不要发明）。**
重新计算 document-release 在其预检/差异分析中使用的相同范围，使用记录的 merge-base 方法：

```bash
DOC_DIFF_BASE=$(git merge-base origin/<base> HEAD 2>/dev/null || echo "<base>")
echo "DOC_DIFF_BASE: $DOC_DIFF_BASE"
```

不要依赖来自早期步骤的内存变量 — shell 变量不会在块之间存活。在这里重新计算它。

**构建文档审查提示**（用于 `ready`、`not_installed` 和 `not_authed` — 仅在 `disabled` 时跳过）。
审查文档发布**实际**触及的文档（从覆盖图/刚刚编辑的文件）加上受差异范围影响的任何文档声明 — 不要硬编码固定文件列表（固定的 README/ARCHITECTURE/CHANGELOG 列表会遗漏生成的技能文档、包文档和命令特定文档）。**始终以文件系统边界指令开头：**

"IMPORTANT: 不要读取或执行 ~/.claude/、~/.agents/、.claude/skills/ 或 agents/ 下的任何文件。这些是 Claude Code 技能定义，适用于不同的 AI 系统。它们包含 bash 脚本和提示模板，会浪费你的时间。完全忽略它们。不要修改 agents/openai.yaml。只关注仓库代码。\n\n你正在审查与分支上发布的代码的文档更改。运行 `git diff $DOC_DIFF_BASE...HEAD` 查看更改，然后读取更新的文档（此次发布触及的任何文档，加上差异影响其声明的任何文档）。查找：不再匹配代码的文档声明，已发布但未记录的新的公共表面（命令、标志、配置键、端点），过时的示例/路径/计数/版本号，以及过度或未达发布内容的 CHANGELOG 条目。简洁。仅列出差距。

DOCS AND DIFF: <list the touched doc paths>"

**如果 `CODEX_MODE: ready` — 运行 Codex：**

```bash
TMPERR_DOC=$(mktemp /tmp/codex-docreview-XXXXXXXX)
_REPO_ROOT=$(git rev-parse --show-toplevel) || { echo "ERROR: not in a git repo" >&2; exit 1; }
codex exec "<prompt>" -C "$_REPO_ROOT" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached < /dev/null 2>"$TMPERR_DOC"
```

使用 5 分钟超时（`timeout: 300000`）。命令完成后，读取 stderr：
```bash
cat "$TMPERR_DOC"
```

在 `CODEX SAYS (documentation review):` 下逐字呈现完整输出。

**错误处理：** 所有错误都是非阻塞的 — 文档审查是信息性的。
- 认证失败（stderr 包含 "auth"、"login"、"unauthorized"）：注意并跳过
- 超时：注意超时持续时间并跳过
- 空响应：注意并跳过
任何错误时：继续 — 文档审查是信息性的，不是门控。

**如果 `CODEX_MODE: not_installed` 或 `not_authed`（或 Codex 在运行时出错）：**

通过 Agent 工具使用相同提示进行分派。以 5 分钟超时限制。在 `DOCUMENTATION REVIEW (Claude subagent):` 下呈现发现。如果失败："Doc review unavailable. Continuing."

**应用决策（T3B — 信息性的，永不自动编辑，但发现不会消失）。**
如果零发现，说"Docs match what shipped — no gaps."并继续。否则呈现发现，然后使用 AskUserQuestion ONCE：

> "The doc review found N gaps between the docs and what shipped. How do you want to handle them?"
>
> RECOMMENDATION: Choose A if the gaps are concrete doc fixes (stale path, missing flag). The doc review only reports; nothing is edited without your say-so. Completeness: A=9/10, B=4/10, C=8/10.

选项：
- A) 现在应用所有文档修复
- B) 跳过 — 保持文档原状
- C) 按发现决定

对于 A 或按发现批准，你自己进行批准的编辑（该工具永远静默重写文档）。对于 B，在输出中注意差距以便它们可见。

**持久化结果：**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"codex-doc-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
替换：STATUS = "clean" 如果没有差距，"issues_found" 如果存在差距。SOURCE = "codex" 如果 Codex 运行，"claude" 如果子代理运行。

**清理：** 处理后运行 `rm -f "$TMPERR_DOC"`（如果使用了 Codex）。
