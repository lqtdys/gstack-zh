## 第 18 步：文档同步（通过子代理，在 PR 创建之前）

**将 /document-release 分派为子代理**，使用 Agent 工具并设置 `subagent_type: "general-purpose"`。子代理获得全新的上下文窗口——前面 17 步的零污染。它还运行**完整的** `/document-release` 工作流（带有 CHANGELOG 覆盖保护、文档排除、风险变更门控、命名暂存、竞态安全的 PR 主体编辑），而不是较弱的重新实现。

**排序：** 此步骤在第 17 步（推送）之后、第 19 步（创建 PR）之前运行。PR 从最终 HEAD 一次性创建，`## Documentation` 部分嵌入初始主体中。不需要创建-再编辑的往复。

**子代理提示：**

> 你正在代码推送后执行 /document-release 工作流。读取完整的 skill 文件 `${HOME}/.claude/skills/gstack/document-release/SKILL.md` 并从头到尾执行其完整工作流，包括 CHANGELOG 覆盖保护、文档排除、风险变更门控和命名暂存。不要尝试编辑 PR 主体——尚不存在 PR。分支：`<branch>`，基准：`<base>`。
>
> 完成工作流后，在响应的**最后一行**输出单个 JSON 对象（其后无其他文本）：
> `{"files_updated":["README.md","CLAUDE.md",...],"commit_sha":"abc1234","pushed":true,"documentation_section":"<用于 PR 主体的 ## Documentation 部分的 markdown 块>"}`
>
> 如果无需更新文档文件，输出：
> `{"files_updated":[],"commit_sha":null,"pushed":false,"documentation_section":null}`

**父处理：**

1. 将子代理输出的**最后一行**解析为 JSON。
2. 存储 `documentation_section`——第 19 步将其嵌入 PR 主体中（若为 null 则省略该部分）。
3. 如果 `files_updated` 非空，打印：`Documentation synced: {files_updated.length} files updated, committed as {commit_sha}`。
4. 如果 `files_updated` 为空，打印：`Documentation is current — no updates needed.`

**如果子代理失败或返回无效 JSON：** 打印警告并继续到第 19 步，不带 `## Documentation` 部分。不要因子代理失败而阻塞 /ship。用户可以 PR 落地后手动运行 `/document-release`。

---

## 第 19 步：创建 PR/MR

**幂等性检查：** 检查此分支是否已存在 PR/MR。

**如果是 GitHub：**
```bash
gh pr view --json url,number,state -q 'if .state == "OPEN" then "PR #\(.number): \(.url)" else "NO_PR" end' 2>/dev/null || echo "NO_PR"
```

**如果是 GitLab：**
```bash
glab mr view -F json 2>/dev/null | jq -r 'if .state == "opened" then "MR_EXISTS" else "NO_MR" end' 2>/dev/null || echo "NO_MR"
```

如果**存在开启的 PR/MR**：使用 `gh pr edit --body-file "$PR_BODY_FILE"`（GitHub）或 `glab mr update -d ...`（GitLab）**更新** PR 主体。始终使用此运行的新鲜结果（测试输出、覆盖率审计、评审发现、对抗评审、TODOS 摘要、第 18 步的 documentation_section）从头重新生成 PR 主体。永远不要复用先前运行的陈旧 PR 主体内容。**在编辑前运行与创建路径（第 19 步）相同的 redaction 扫描-at-sink（PR 主体 + 标题）——扫描临时文件，然后从中 `gh pr edit --body-file`。**

**始终将 PR 标题更新为以 `v$NEW_VERSION` 开头。** PR 标题使用工作空间感知的格式 `v<NEW_VERSION> <type>: <summary>`——版本始终在前，无例外，无"自定义标题有意保留"的逃生舱。共享助手 `bin/gstack-pr-title-rewrite.sh` 是该规则的唯一真相源。

1. 读取当前标题：`CURRENT=$(gh pr view --json title -q .title)`（或 `glab mr view -F json | jq -r .title`）。
2. 计算修正后的标题：`NEW_TITLE=$(~/.claude/skills/gstack/bin/gstack-pr-title-rewrite.sh "$NEW_VERSION" "$CURRENT")`。助手处理三种情况：标题已正确（无操作）、标题有另一个 `v<X.Y.Z.W>` 前缀（替换）、标题无前缀（前置）。
3. 如果 `NEW_TITLE` 与 `CURRENT` 不同，运行 `gh pr edit --title "$NEW_TITLE"`（或 `glab mr update -t "$NEW_TITLE"`）。
4. **自检：** 重新获取标题并断言它以 `v$NEW_VERSION ` 开头。如果不这样做，重试编辑一次。如果仍失败，向用户展示失败。

这在第 12 步的队列漂移检测重新碰撞陈旧版本时保持标题真实，并在没有创建 PR 的格式时强制执行格式。

打印现有 URL 并继续到第 20 步。

如果不存在 PR/MR：使用第 0 步检测到的平台创建 pull request（GitHub）或 merge request（GitLab）。

PR/MR 主体应包含以下部分：

```
## Summary
<总结所有正在发布的变更。运行 `git log <base>..HEAD --oneline` 以枚举每次提交。
排除 VERSION/CHANGELOG 元数据提交（这是此 PR 的簿记，不是实质性变更）。
将剩余提交分组为逻辑部分（例如，"**Performance**"、"**Dead Code Removal**"、
"**Infrastructure**"）。每次实质性提交必须出现在至少一个部分中。
如果某次提交的工作未反映在摘要中，你遗漏了它。>

## Test Coverage
<第 7 步的覆盖率图表，或"All new code paths have test coverage.">
<如果第 7 步运行过："Tests: {before} → {after} (+{delta} new)">

## Pre-Landing Review
<第 9 步代码评审的发现，或"No issues found.">

## Design Review
<如果设计评审运行过："Design Review (lite): N findings — M auto-fixed, K skipped. AI Slop: clean/N issues.">
<如果无前端文件变更："No frontend files changed — design review skipped.">

## Eval Results
<如果 eval 运行过：套件名称、通过/失败计数、成本仪表板总结。如果跳过："No prompt-related files changed — evals skipped.">

## Greptile Review
<如果找到 Greptile 评论：带 [FIXED] / [FALSE POSITIVE] / [ALREADY FIXED] 标签的项目符号列表 + 每条评论一行总结>
<如果未找到 Greptile 评论："No Greptile comments.">
<如果在第 10 步期间不存在 PR：完全省略此部分>

## Scope Drift
<如果范围漂移运行过："Scope Check: CLEAN" 或漂移/蠕变发现列表>
<如果无范围漂移：省略此部分>

## Plan Completion
<如果找到计划文件：第 8 步的完成清单总结>
<如果无计划文件："No plan file detected.">
<如果计划项目推迟：列出推迟的项目>

## Linked Spec
<自动检测：通过以下方式查找与此分支匹配的 /spec 档案：
  eval "$(~/.claude/skills/gstack/bin/gstack-paths)"
  eval "$(~/.claude/skills/gstack/bin/gstack-slug)"
  CURRENT_BRANCH=$(git branch --show-current)
  SPEC_ARCHIVES="$GSTACK_STATE_ROOT/projects/$SLUG/specs"
  # Find newest archive whose spec_branch frontmatter matches current branch (or one of its
  # parents — if spec spawned worktree spec/<slug>-$$, the spawned worktree IS where /ship runs).
  SPEC_FILE=$(grep -l "^spec_branch: $CURRENT_BRANCH$" "$SPEC_ARCHIVES"/*.md 2>/dev/null | head -1)
  [ -z "$SPEC_FILE" ] && exit  # no spec; omit this section entirely
  SPEC_ISSUE=$(grep "^spec_issue_number:" "$SPEC_FILE" | cut -d' ' -f2)
  [ -z "$SPEC_ISSUE" ] && exit  # spec archive exists but no issue number; omit

  # CONDITIONAL Closes #N (codex F4): only add when Plan Completion above is "complete".
  # If the plan completion gate from Step 8 reports any deferred or failed items, emit:
  #   "Linked to #$SPEC_ISSUE (partial delivery — NOT auto-closing; close manually after follow-up)"
  # If Plan Completion is fully complete, emit:
  #   "Closes #$SPEC_ISSUE"
  # and include the Closes #N line in the PR body so GitHub auto-closes on merge.>

<Format:
  Closes #<N>

  This PR delivers the spec at <archive path relative to repo root>.
  Spec filed: <spec_filed_at from frontmatter>>

<If partial delivery, emit instead:
  Linked to #<N> (partial delivery — not auto-closing).
  Deferred items: <list from Plan Completion>.
  Close #<N> manually after follow-up lands.>

<如果无 /spec 档案匹配此分支：完全省略此部分。>

## Verification Results
<如果验证运行过：第 8.1 步的总结（N PASS, M FAIL, K SKIPPED）>
<如果跳过：原因（无计划、无服务器、无验证部分）>
<如果不适用：省略此部分>

## TODOS
<如果项目标记为完成：带版本的已完成项目符号列表>
<如果无项目完成："No TODO items completed in this PR.">
<如果 TODOS.md 已创建或重组：注明>
<如果 TODOS.md 不存在且用户跳过：省略此部分>

## Documentation
<在此逐字嵌入第 18 步子代理返回的 `documentation_section` 字符串。>
<如果第 18 步返回 `documentation_section: null`（无文档更新），完全省略此部分。>

## Test plan
- [x] 所有 Rails 测试通过（N runs, 0 failures）
- [x] 所有 Vitest 测试通过（N tests）

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

#### 扫描编辑（PR 主体 +标题）——在创建 AND 编辑之前运行

公共仓库上的 PR 主体是全世界可读的。在发送前扫描-at-sink：
将组合主体写入临时文件，使用共享引擎扫描该文件，
并将同一文件传递给 `gh`/`glab`。将任何 Codex / Greptile / eval 输出
部分用 tool-attributed 围栏包裹（` ```codex-review ` / ` ```greptile `）以便引擎
降级这些工具引用的示例凭据的警告而不是阻塞
PR（围栏内的实时格式凭据仍然阻塞）。

```bash
REDACT_VIS=$(~/.claude/skills/gstack/bin/gstack-config get redact_repo_visibility 2>/dev/null)
[ -z "$REDACT_VIS" ] && REDACT_VIS=$(gh repo view --json visibility -q .visibility 2>/dev/null | tr 'A-Z' 'a-z')
REDACT_VIS="${REDACT_VIS:-unknown}"
PR_BODY_FILE=$(mktemp)
cat > "$PR_BODY_FILE" <<'PR_BODY_EOF'
<PR body from above>
PR_BODY_EOF
~/.claude/skills/gstack/bin/gstack-redact --from-file "$PR_BODY_FILE" --repo-visibility "$REDACT_VIS" --self-email "$(git config user.email 2>/dev/null)" --json
case $? in
  3) echo "BLOCKED — credential in PR body. Rotate + redact, do not create the PR."; exit 1 ;;
  2) echo "MEDIUM findings — confirm per finding (sterner on public) before proceeding." ;;
esac
# Also scan the title (short, single-line):
printf '%s' "v$NEW_VERSION <type>: <summary>" | ~/.claude/skills/gstack/bin/gstack-redact --repo-visibility "$REDACT_VIS" --json
```

HIGH 阻塞（退出 3，无跳过）。MEDIUM → AskUserQuestion（PII 子集提供 `--auto-redact`）。相同扫描在 `gh pr edit --body` 路径之前运行（第 17 步）。

**如果为 GitHub：** 从扫描的文件创建（扫描的字节 = 发送的字节）：

```bash
# PR title MUST start with v$NEW_VERSION — enforced on every run, no exceptions.
# (See Step 19 idempotency block + bin/gstack-pr-title-rewrite.sh for the rule.)
gh pr create --base <base> --title "v$NEW_VERSION <type>: <summary>" --body-file "$PR_BODY_FILE"
rm -f "$PR_BODY_FILE"
```

**如果为 GitLab：**

```bash
# MR title MUST start with v$NEW_VERSION — enforced on every run, no exceptions.
# (See Step 19 idempotency block + bin/gstack-pr-title-rewrite.sh for the rule.)
glab mr create -b <base> -t "v$NEW_VERSION <type>: <summary>" -d "$(cat <<'EOF'
<MR body from above>
EOF
)"
```

**如果两个 CLI 都不可用：**
打印分支名称、远程 URL，并指示用户通过 Web UI 手动创建 PR/MR。不要停下——代码已推送就绪。

**输出 PR/MR URL**——然后继续到第 20 步。

---
