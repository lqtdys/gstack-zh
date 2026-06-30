# Greptile 评论分类

获取、筛选和分类 GitHub PR 上 Greptile 评审评论的共享参考文档。`/review`（步骤 2.5）和 `/ship`（步骤 3.75）均引用本文档。

---

## 获取

运行这些命令获取 PR 和评论。两个 API 调用并行执行。

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
PR_NUMBER=$(gh pr view --json number --jq '.number' 2>/dev/null)
```

**如果上述任意命令失败或返回为空：** 直接跳过 Greptile 分类。此集成是附加功能——没有它流程正常运行。

```bash
# 并行获取行级评论和 PR 顶层评论
gh api repos/$REPO/pulls/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | select(.position != null) | {id: .id, path: .path, line: .line, body: .body, html_url: .html_url, source: "line-level"}' > /tmp/greptile_line.json &
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | {id: .id, body: .body, html_url: .html_url, source: "top-level"}' > /tmp/greptile_top.json &
wait
```

**如果 API 报错或两个端点都无 Greptile 评论：** 直接跳过。

`position != null` 过滤条件会自动跳过因强制推送而过时的评论。

---

## 抑制检查

推导项目特定的历史路径：
```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
PROJECT_HISTORY="$HOME/.gstack/projects/$REMOTE_SLUG/greptile-history.md"
```

如存在则读取 `$PROJECT_HISTORY`（按项目存储的抑制记录）。每行记录一次历史分类结果：

```
<日期> | <仓库> | <类型:fp|fix|already-fixed> | <文件模式> | <分类>
```

**分类**（固定集）：`race-condition`、`null-check`、`error-handling`、`style`、`type-safety`、`security`、`performance`、`correctness`、`other`

匹配每条评论，过滤条件为：
- `type == fp`（仅抑制已知的误报，不抑制已修复的真实问题）
- `repo` 匹配当前仓库
- `file-pattern` 匹配评论的文件路径
- `category` 匹配评论中的问题类型

将匹配的评论标记为**已抑制**。

如果历史文件不存在或包含无法解析的行，则跳过这些行继续——绝不因历史文件格式错误而中止流程。

---

## 分类

对于每一条未被抑制的评论：

1. **行级评论：** 读取指定 `path:line` 处的文件及其周围上下文（±10 行）
2. **顶层评论：** 读取完整评论正文
3. 将该评论与完整 diff（`git diff origin/main`）和评审清单交叉比对
4. 分类：
   - **有效且可执行（VALID & ACTIONABLE）** — 当前代码中确实存在的真实 bug、竞态条件、安全漏洞或正确性问题
   - **已修复（VALID BUT ALREADY FIXED）** — 真实问题已在分支后续提交中得到解决。需识别修复提交的 SHA。
   - **误报（FALSE POSITIVE）** — 评论误解了代码、标注了在其他地方已处理的内容，或仅为风格噪音
   - **已抑制（SUPPRESSED）** — 已在上述抑制检查中过滤

---

## 回复 API

回复 Greptile 评论时，根据评论来源使用正确的端点：

**行级评论**（来自 `pulls/$PR/comments`）：
```bash
gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies \
  -f body="<回复内容>"
```

**顶层评论**（来自 `issues/$PR/comments`）：
```bash
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  -f body="<回复内容>"
```

**如果回复 POST 失败**（例如 PR 已关闭、无写入权限）：警告后继续。不要因回复失败而中止流程。

---

## 回复模板

每次回复 Greptile 均使用以下模板。始终包含具体证据——绝不发布模糊回复。

### 层级 1（首次回复）— 友好且含证据

**用户选择修复问题：**

```
**固定修复**于 `<commit-sha>`。

\```diff
- <原问题行>
+ <修复行>
\```

**原因：** <一句话说明问题所在及修复方式>
```

**已修复（问题在分支先前提交中已解决）：**

```
**已修复**于 `<commit-sha>`。

**修复方式：** <1-2 句说明现有提交如何解决了该问题>
```

**误报（评论不正确）：**

```
**不是 Bug。** <一句话直接说明为何不正确>

**证据：**
- <具体代码引用，证明该模式是安全的/正确的>
- <例如，"nil 检查由 `ActiveRecord::FinderMethods#find` 处理，该方法会抛出 RecordNotFound，而非 nil">

**建议重新分级：** 这看起来像是 `<style|noise|misread>` 问题，而非 Greptile 所说的类型。建议降低严重级别。
```

### 层级 2（Greptile 重新标记后）— 强硬，证据充分

当（下方）升级检测识别到同一对话线程中已有先前的 GStack 回复时使用层级 2。包含最大证据以结束讨论。

```
**已经过复查并确认为 [意图性修复/已修复/不是 Bug]。

\```<完整相关 diff 显示变更或安全模式>
\```

**证据链：**
1. <file:line 永久链接，显示安全模式或修复>
2. <涉及的提交 SHA（如有）>
3. <架构原理或设计决策（如有）>

**建议重新分级：** 请重新校准——这实际上是 `<实际类别>` 问题，而非 `<声称的类别>`。[如有帮助，可链接到特定文件变更的永久链接]
```

---

## 升级检测

撰写回复之前，先检查此评论线程中是否已存在之前的 GStack 回复：

1. **行级评论：** 通过 `gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies` 获取回复。检查任何回复正文是否包含 GStack 标记：`**Fixed**`、`**Not a bug.**`、`**Already fixed**`。

2. **顶层评论：** 扫描获取的用户议题评论中，是否存在 Greptile 评论之后发布的且包含 GStack 标记的回复。

3. **如果存在先前的 GStack 回复且 Greptile 在相同文件+分类上再次评论：** 使用层级 2（强硬）模板。

4. **如果不存在先前的 GStack 回复：** 使用层级 1（友好）模板。

如果升级检测失败（API 错误、模糊线程），默认使用层级 1。绝不因为模糊情况而升级。

---

## 严重性评估与重新分级

对评论进行分类时，还需评估 Greptile 暗示的严重性是否与实际情况相符：

- 如果 Greptile 将某内容标记为**安全/正确性/竞态条件**问题，但实际上只是**风格/性能**小问题：在回复中包含 `**建议重新分级：**` 请求修正分类。
- 如果 Greptile 将低严重性的风格问题标为关键问题：在回复中反驳。
- 始终具体说明重新分级的理由——引用代码和行数，而非主观意见。

---

## 历史文件写入

写入前先确保两个目录存在：
```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
mkdir -p "$HOME/.gstack/projects/$REMOTE_SLUG"
mkdir -p ~/.gstack
```

将每条分类结果追加到**两个**文件（按项目抑制用，全局用于复盘）：
- `~/.gstack/projects/$REMOTE_SLUG/greptile-history.md`（按项目的）
- `~/.gstack/greptile-history.md`（全局汇总）

格式：
```
<YYYY-MM-DD> | <owner/repo> | <类型> | <文件模式> | <分类>
```

示例条目：
```
2026-03-13 | garrytan/myapp | fp | app/services/auth_service.rb | race-condition
2026-03-13 | garrytan/myapp | fix | app/models/user.rb | null-check
2026-03-13 | garrytan/myapp | already-fixed | lib/payments.rb | error-handling
```

---

## 输出格式

在输出头部包含 Greptile 摘要：
```
+ N 条 Greptile 评论（X 有效，Y 已修复，Z 误报）
```

对每条评论，显示：
- 分类标签：`[有效]`、`[已修复]`、`[误报]`、`[已抑制]`
- 文件:行引用（行级）或 `[顶层]`（顶层）
- 一句话摘要正文
- 永久链接 URL（`html_url` 字段）
