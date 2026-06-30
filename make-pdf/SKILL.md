---
name: make-pdf
preamble-tier: 1
version: 1.0.0
description: 将任意 markdown 文件转换为出版级质量的 PDF。(gstack)
triggers:
  - markdown to pdf
  - generate pdf
  - make pdf
  - export pdf
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## When to invoke this skill

精确的 1 英寸边距、智能分页、页码、封面页、页眉、弯引号和破折号、可点击的目录、对角线 DRAFT 水印。不是草稿产物——是成品。当被要求"制作 PDF"、"导出为 PDF"、"把这个 markdown 转为 PDF"或"生成文档"时使用。

语音触发（语音转文字别名）："make this a pdf", "make it a pdf", "export to pdf", "turn this into a pdf", "turn this markdown into a pdf", "generate a pdf", "make a pdf from", "pdf this markdown"。

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -exec rm {} + 2>/dev/null || true
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_PROACTIVE_PROMPTED=$([ -f ~/.gstack/.proactive-prompted ] && echo "yes" || echo "no")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
_SKILL_PREFIX=$(~/.claude/skills/gstack/bin/gstack-config get skill_prefix 2>/dev/null || echo "false")
echo "PROACTIVE: $_PROACTIVE"
echo "PROACTIVE_PROMPTED: $_PROACTIVE_PROMPTED"
echo "SKILL_PREFIX: $_SKILL_PREFIX"
source <(~/.claude/skills/gstack/bin/gstack-repo-mode 2>/dev/null) || true
REPO_MODE=${REPO_MODE:-unknown}
echo "REPO_MODE: $REPO_MODE"
_SESSION_KIND=$(~/.claude/skills/gstack/bin/gstack-session-kind 2>/dev/null || echo "interactive")
case "$_SESSION_KIND" in spawned|headless|interactive) ;; *) _SESSION_KIND="interactive" ;; esac
echo "SESSION_KIND: $_SESSION_KIND"
# Conductor host: AskUserQuestion is unreliable here (native disabled, MCP
# variant flaky), so skills render decisions as prose instead of calling the
# tool. Gated on !headless so an eval/CI run INSIDE Conductor (GSTACK_HEADLESS)
# still BLOCKs rather than rendering prose to nobody.
if [ "$_SESSION_KIND" != "headless" ] && { [ -n "${CONDUCTOR_WORKSPACE_PATH:-}" ] || [ -n "${CONDUCTOR_PORT:-}" ]; }; then
  echo "CONDUCTOR_SESSION: true"
fi
_ACTIVATED=$([ -f ~/.gstack/.activated ] && echo "yes" || echo "no")
_FIRST_LOOP_SHOWN=$([ -f ~/.gstack/.first-loop-tip-shown ] && echo "yes" || echo "no")
echo "ACTIVATED: $_ACTIVATED"
echo "FIRST_LOOP_SHOWN: $_FIRST_LOOP_SHOWN"
# First-run project detection: run the detector ONLY on the first-ever skill run
# (ACTIVATED=no, interactive) so it stays off the hot path for every run after.
_FIRST_TASK=""
if [ "$_ACTIVATED" = "no" ] && [ "$_SESSION_KIND" != "headless" ]; then
  _FIRST_TASK=$(~/.claude/skills/gstack/bin/gstack-first-task-detect 2>/dev/null || true)
fi
echo "FIRST_TASK: $_FIRST_TASK"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
_EXPLAIN_LEVEL=$(~/.claude/skills/gstack/bin/gstack-config get explain_level 2>/dev/null || echo "default")
if [ "$_EXPLAIN_LEVEL" != "default" ] && [ "$_EXPLAIN_LEVEL" != "terse" ]; then _EXPLAIN_LEVEL="default"; fi
echo "EXPLAIN_LEVEL: $_EXPLAIN_LEVEL"
_QUESTION_TUNING=$(~/.claude/skills/gstack/bin/gstack-config get question_tuning 2>/dev/null || echo "false")
echo "QUESTION_TUNING: $_QUESTION_TUNING"
mkdir -p ~/.gstack/analytics
if [ "$_TEL" != "off" ]; then
echo '{"skill":"make-pdf","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(_repo=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null | tr -cd 'a-zA-Z0-9._-'); echo "${_repo:-unknown}")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
for _PF in $(find ~/.gstack/analytics -maxdepth 1 -name '.pending-*' 2>/dev/null); do
  if [ -f "$_PF" ]; then
    if [ "$_TEL" != "off" ] && [ -x "~/.claude/skills/gstack/bin/gstack-telemetry-log" ]; then
      ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true
    fi
    rm -f "$_PF" 2>/dev/null || true
  fi
  break
done
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
_LEARN_FILE="${GSTACK_HOME:-$HOME/.gstack}/projects/${SLUG:-unknown}/learnings.jsonl"
if [ -f "$_LEARN_FILE" ]; then
  _LEARN_COUNT=$(wc -l < "$_LEARN_FILE" 2>/dev/null | tr -d ' ')
  echo "LEARNINGS: $_LEARN_COUNT entries loaded"
  if [ "$_LEARN_COUNT" -gt 5 ] 2>/dev/null; then
    ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 3 2>/dev/null || true
  fi
else
  echo "LEARNINGS: 0"
fi
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"make-pdf","event":"started","branch":"'"$_BRANCH"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
_HAS_ROUTING="no"
if [ -f CLAUDE.md ] && grep -q "## Skill routing" CLAUDE.md 2>/dev/null; then
  _HAS_ROUTING="yes"
fi
_ROUTING_DECLINED=$(~/.claude/skills/gstack/bin/gstack-config get routing_declined 2>/dev/null || echo "false")
echo "HAS_ROUTING: $_HAS_ROUTING"
echo "ROUTING_DECLINED: $_ROUTING_DECLINED"
_VENDORED="no"
if [ -d ".claude/skills/gstack" ] && [ ! -L ".claude/skills/gstack" ]; then
  if [ -f ".claude/skills/gstack/VERSION" ] || [ -d ".claude/skills/gstack/.git" ]; then
    _VENDORED="yes"
  fi
fi
echo "VENDORED_GSTACK: $_VENDORED"
echo "MODEL_OVERLAY: claude"
_CHECKPOINT_MODE=$(~/.claude/skills/gstack/bin/gstack-config get checkpoint_mode 2>/dev/null || echo "explicit")
_CHECKPOINT_PUSH=$(~/.claude/skills/gstack/bin/gstack-config get checkpoint_push 2>/dev/null || echo "false")
echo "CHECKPOINT_MODE: $_CHECKPOINT_MODE"
echo "CHECKPOINT_PUSH: $_CHECKPOINT_PUSH"
# Plan-mode hint for skills like /spec that branch behavior on plan-mode state.
# Claude Code exposes plan mode via system reminders; we detect best-effort
# from CLAUDE_PLAN_FILE (set by the harness when plan mode is active) and
# fall back to "inactive". Codex hosts and Claude execution mode both end up
# inactive, which is the safe default (defaults to file+execute pipeline).
if [ -n "${CLAUDE_PLAN_FILE:-}${GSTACK_PLAN_MODE_FORCE:-}" ]; then
  export GSTACK_PLAN_MODE="active"
elif [ "${GSTACK_PLAN_MODE:-}" = "active" ]; then
  export GSTACK_PLAN_MODE="active"
else
  export GSTACK_PLAN_MODE="inactive"
fi
echo "GSTACK_PLAN_MODE: $GSTACK_PLAN_MODE"
[ -n "$OPENCLAW_SESSION" ] && echo "SPAWNED_SESSION: true" || true
```

## MAKE-PDF SETUP (run this check BEFORE any make-pdf command)

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
P=""
[ -n "$MAKE_PDF_BIN" ] && [ -x "$MAKE_PDF_BIN" ] && P="$MAKE_PDF_BIN"
[ -z "$P" ] && [ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/make-pdf/dist/pdf" ] && P="$_ROOT/.claude/skills/gstack/make-pdf/dist/pdf"
[ -z "$P" ] && P="$HOME/.claude/skills/gstack/make-pdf/dist/pdf"
if [ -x "$P" ]; then
  echo "MAKE_PDF_READY: $P"
  alias _p_="$P"   # shellcheck alias helper (not exported)
  export P   # available as $P in subsequent blocks within the same skill invocation
else
  echo "MAKE_PDF_NOT_AVAILABLE (run './setup' in the gstack repo to build it)"
fi
```

如果输出了 `MAKE_PDF_NOT_AVAILABLE`：告诉用户二进制文件尚未构建。让他们在 gstack 仓库中运行 `./setup`，然后重试。

如果输出了 `MAKE_PDF_READY`：`$P` 是该技能剩余部分的二进制路径。请使用 `$P`（而非显式路径），以保持技能正文的可移植性。

核心命令：
- `$P generate <input.md> [output.pdf]` — 将 markdown 渲染为 PDF（80% 的使用场景）
- `$P generate --cover --toc essay.md out.pdf` — 完整出版物版式
- `$P generate --watermark DRAFT memo.md draft.pdf` — 对角线 DRAFT 水印
- `$P preview <input.md>` — 渲染 HTML 并在浏览器中打开（快速迭代）
- `$P setup` — 验证 browse + Chromium + pdftotext 并运行冒烟测试
- `$P --help` — 完整参数参考

输出约定：
- `stdout`：成功时仅输出路径。一行。
- `stderr`：进度信息（`Rendering HTML... Generating PDF...`），除非加了 `--quiet`。
- 退出码：0 成功 / 1 参数错误 / 2 渲染错误 / 3 Paged.js 超时 / 4 browse 不可用。

## Plan Mode 安全操作

在计划模式下允许执行，因为它们为计划提供信息：`$B`、`$D`、`codex exec`/`codex review`、写入 `~/.gstack/`、写入计划文件，以及 `open` 打开生成的产物。

## 计划模式期间的技能调用

如果用户在计划模式下调用技能，该技能优先于通用计划模式行为。**将技能文件视为可执行指令，而非参考文档。** 从 Step 0 开始逐步执行；第一个 AskUserQuestion 表示工作流进入计划模式，而非违反计划模式。AskUserQuestion（任何变体 — `mcp__*__AskUserQuestion` 或原生版本；参见 "AskUserQuestion Format → Tool resolution"）满足计划模式的回合结束要求。如果 AskUserQuestion 不可用或调用失败，按照 AskUserQuestion Format 故障回退处理：`headless` → BLOCKED；`interactive` → 散文式回退（同样满足回合结束要求）。在 STOP 点立即停止。不要继续工作流或在此处调用 ExitPlanMode。标记为 "PLAN MODE EXCEPTION — ALWAYS RUN" 的命令始终执行。仅在技能工作流完成后、或用户要求取消技能或退出计划模式时，方可调用 ExitPlanMode。

如果 `PROACTIVE` 为 `"false"`，不要自动调用或主动建议技能。如果觉得某个技能有用，可以问："我觉得 /skillname 可能有用——要我运行吗？"

如果 `SKILL_PREFIX` 为 `"true"`，使用 `/gstack-*` 名称调用/建议。磁盘路径仍为 `~/.claude/skills/gstack/[skill-name]/SKILL.md`。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并按"内联升级流程"操作（如已配置则自动升级，否则 AskUserQuestion 提供 4 个选项，如拒绝则写入休眠状态）。

如果输出显示 `JUST_UPGRADE <from> <to>`：打印 "Running gstack v{to} (just updated!)"。如果 `SPAWNED_SESSION` 为 true，跳过功能发现。

功能发现，每个会话最多一次提示：
- 缺少 `~/.claude/skills/gstack/.feature-prompted-continuous-checkpoint`：AskUserQuestion 询问 Continuous checkpoint 自动提交。如果接受，运行 `~/.claude/skills/gstack/bin/gstack-config set checkpoint_mode continuous`。始终 touch 标记文件。
- 缺少 `~/.claude/skills/gstack/.feature-prompted-model-overlay`：告知"Model overlays 已激活。MODEL_OVERLAY 显示补丁。"始终 touch 标记文件。

升级提示后，继续工作流。

如果 `WRITING_STYLE_PENDING` 为 `yes`：询问一次写作风格：

> v1 提示更简洁：首次使用术语注释、结果导向的问题、简短的散文。保持默认还是恢复简约风格？

选项：
- A) 保持新的默认值（推荐——好写作对每个人都有帮助）
- B) 恢复 V0 散文风格——设置 `explain_level: terse`

如果 A：不设置 `explain_level`（默认即为 `default`）。
如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set explain_level terse`。

始终运行（无论选择哪个）：
```bash
rm -f ~/.gstack/.writing-style-prompt-pending
touch ~/.gstack/.writing-style-prompted
```

如果 `WRITING_STYLE_PENDING` 为 `no`，则跳过。

如果 `LAKE_INTRO` 为 `no`：说明"gstack 遵循**Boil the Ocean**原则——当 AI 使边际成本接近零时，就把事情做完整。了解更多：https://garryslist.org/posts/boil-the-ocean"。提供打开方式：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

仅在用户同意时运行 `open`。始终运行 `touch`。

如果 `TEL_PROMPTED` 为 `no` 且 `LAKE_INTRO` 为 `yes`：通过 AskUserQuestion 询问一次遥测：

> 帮助 gstack 改进。仅分享使用数据：技能名称、持续时间、崩溃、稳定的设备 ID。不收集代码或文件路径。你的仓库名仅本地记录，上传前会被剥离。

选项：
- A) 帮助 gstack 改进!（推荐）
- B) 不用了，谢谢

如果 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果 B：追加询问：

> 匿名模式仅发送聚合使用数据，不收集唯一 ID。

选项：
- A) 好的，匿名可以
- B) 不用了，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

如果 `TEL_PROMPTED` 为 `yes` 则跳过。

如果 `PROACTIVE_PROMPTED` 为 `no` 且 `TEL_PROMPTED` 为 `yes`：询问一次：

> 让 gstack 主动推荐技能，比如运行 /qa 来检查"这个能用吗？"或者 /investigate 来排查 bug？

选项：
- A) 保持开启（推荐）
- B) 关闭——我会自己输入 /commands

如果 A：运行 `~/.claude/skills/gstack/bin/gstack-config set proactive true`
如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set proactive false`

始终运行：
```bash
touch ~/.gstack/.proactive-prompted
```

如果 `PROACTIVE_PROMPTED` 为 `yes` 则跳过。

## 首次运行引导（一次性）

如果 `ACTIVATED` 为 `no`（本机器首次运行技能）且 preamble 输出的 `FIRST_TASK:` 值非空且不是 `nongit`：根据标记映射显示一条简短的项目特定提示，作为提醒，然后继续执行用户实际要求的任务——不要中断任务。标记映射规则：`greenfield` → "新先用 `/spec` 或 `/office-hours` 规划形状。"`code_node`/`code_python`/`code_rust`/`code_go`/`code_ruby`/`code_ios` → "这里有代码——`/qa` 看运行效果，有问题用 `/investigate`。"`branch_ahead` → "此分支有未发布的工作——先 `/review` 再 `/ship`。"`dirty_default` → "有未提交的改动——提交前先 `/review`。"`clean_default` → "选一个：`/spec`、`/investigate` 或 `/qa`。"然后将你看到的标记替换为 TASK_TOKEN 并运行（尽力而为），标记已激活：
```bash
~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type first_task_scaffold_shown --skill "TASK_TOKEN" --outcome shown 2>/dev/null || true
touch ~/.gstack/.activated 2>/dev/null || true
```

如果 `ACTIVATED` 为 `no` 但 `FIRST_TASK:` 为空或为 `nongit`（headless、非 git、或不可操作）：不显示任何内容，仅运行 `touch ~/.gstack/.activated 2>/dev/null || true`。

否则如果 `ACTIVATED` 为 `yes` 且 `FIRST_LOOP_SHOWN` 为 `no`：提示一次作为提醒（然后继续）：

> 提示：gstack 在完成一个循环时效果最好——**plan → review → ship**。常见的第一个循环：`/office-hours` 或 `/spec` 规划形状，`/plan-eng-review` 锁定方案，然后 `/ship`。

然后运行 `touch ~/.gstack/.first-loop-tip-shown 2>/dev/null || true`。

如果 `ACTIVATED` 和 `FIRST_LOOP_SHOWN` 都为 `yes`，则跳过此部分。

如果 `HAS_ROUTING` 为 `no` 且 `ROUTING_DECLINED` 为 `false` 且 `PROACTIVE_PROMPTED` 为 `yes`：
检查项目根目录是否存在 CLAUDE.md 文件。如果不存在，创建它。

使用 AskUserQuestion：

> gstack 在项目的 CLAUDE.md 中包含技能路由规则时效果最好。

选项：
- A) 添加路由规则到 CLAUDE.md（推荐）
- B) 不用了，我手动调用技能

如果 A：将以下部分追加到 CLAUDE.md 末尾：

```markdown

## Skill routing

When the user's request matches an available skill, invoke it via the Skill tool. When in doubt, invoke the skill.

Key routing rules:
- Product ideas/brainstorming → invoke /office-hours
- Strategy/scope → invoke /plan-ceo-review
- Architecture → invoke /plan-eng-review
- Design system/plan review → invoke /design-consultation or /plan-design-review
- Full review pipeline → invoke /autoplan
- Bugs/errors → invoke /investigate
- QA/testing site behavior → invoke /qa or /qa-only
- Code review/diff check → invoke /review
- Visual polish → invoke /design-review
- Ship/deploy/PR → invoke /ship or /land-and-deploy
- Save progress → invoke /context-save
- Resume context → invoke /context-restore
- Author a backlog-ready spec/issue → invoke /spec
```

然后提交更改：`git add CLAUDE.md && git commit -m "chore: add gstack skill routing rules to CLAUDE.md"`

如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set routing_declined true`，并告知用户可用 `gstack-config set routing_declined false` 重新启用。

这仅在每个项目发生一次。如果 `HAS_ROUTING` 为 `yes` 或 `ROUTING_DECLINED` 为 `true` 则跳过。

如果 `VENDORED_GSTACK` 为 `yes`，通过 AskUserQuestion 警告一次，除非 `~/.gstack/.vendoring-warned-$SLUG` 存在：

> 此项目在 `.claude/skills/gstack/` 中有 gstack vendored 副本。Vendoring 已弃用。
> 是否迁移到 team 模式？

选项：
- A) 是，立即迁移到 team 模式
- B) 不用了，我自己处理

如果 A：
1. 运行 `git rm -r .claude/skills/gstack/`
2. 运行 `echo '.claude/skills/gstack/' >> .gitignore`
3. 运行 `~/.claude/skills/gstack/bin/gstack-team-init required`（或 `optional`）
4. 运行 `git add .claude/ .gitignore CLAUDE.md && git commit -m "chore: migrate gstack from vendored to team mode"`
5. 告知用户："完成。每位开发者现在运行：`cd ~/.claude/skills/gstack && ./setup --team`"

如果 B：说明"OK，你需要自己负责保持 vendored 副本更新。"

始终运行（无论选择哪个）：
```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
touch ~/.gstack/.vendoring-warned-${SLUG:-unknown}
```

如果标记文件存在，则跳过。

如果 `SPAWNED_SESSION` 为 `"true"`，你正在由 AI 编排器（如 OpenClaw）生成的会话中运行。在生成的会话中：
- 不要对交互式提示使用 AskUserQuestion。自动选择推荐选项。
- 不要运行升级检查、遥测提示、路由注入或 lake 入门引导。
- 专注于完成任务并通过散文输出报告结果。
- 以完成报告收尾：发布了什么、做了哪些决策、有哪些不确定事项。
- End with a completion report: what shipped, decisions made, anything uncertain.

## Artifacts Sync (skill start)

```bash
_GSTACK_HOME="${GSTACK_HOME:-$HOME/.gstack}"
# Prefer the v1.27.0.0 artifacts file; fall back to brain file for users
# upgrading mid-stream before the migration script runs.
if [ -f "$HOME/.gstack-artifacts-remote.txt" ]; then
  _BRAIN_REMOTE_FILE="$HOME/.gstack-artifacts-remote.txt"
else
  _BRAIN_REMOTE_FILE="$HOME/.gstack-brain-remote.txt"
fi
_BRAIN_SYNC_BIN="~/.claude/skills/gstack/bin/gstack-brain-sync"
_BRAIN_CONFIG_BIN="~/.claude/skills/gstack/bin/gstack-config"

# /sync-gbrain context-load: teach the agent to use gbrain when it's available.
# Per-worktree pin: post-spike redesign uses kubectl-style `.gbrain-source` in the
# git toplevel to scope queries. Look for the pin in the worktree (not a global
# state file) so that opening worktree B without a pin doesn't claim "indexed"
# just because worktree A was synced. Empty string when gbrain is not
# configured (zero context cost for non-gbrain users).
_GBRAIN_CONFIG="$HOME/.gbrain/config.json"
if [ -f "$_GBRAIN_CONFIG" ] && command -v gbrain >/dev/null 2>&1; then
  _GBRAIN_VERSION_OK=$(gbrain --version 2>/dev/null | grep -c '^gbrain ' || echo 0)
  if [ "$_GBRAIN_VERSION_OK" -gt 0 ] 2>/dev/null; then
    _GBRAIN_PIN_PATH=""
    _REPO_TOP=$(git rev-parse --show-toplevel 2>/dev/null || echo "")
    if [ -n "$_REPO_TOP" ] && [ -f "$_REPO_TOP/.gbrain-source" ]; then
      _GBRAIN_PIN_PATH="$_REPO_TOP/.gbrain-source"
    fi
    if [ -n "$_GBRAIN_PIN_PATH" ]; then
      echo "GBrain configured. Prefer \`gbrain search\`/\`gbrain query\` over Grep for"
      echo "semantic questions; use \`gbrain code-def\`/\`code-refs\`/\`code-callers\` for"
      echo "symbol-aware code lookup. See \"## GBrain Search Guidance\" in CLAUDE.md."
      echo "Run /sync-gbrain to refresh."
    else
      echo "GBrain configured but this worktree isn't pinned yet. Run \`/sync-gbrain --full\`"
      echo "before relying on \`gbrain search\` for code questions in this worktree."
      echo "Falls back to Grep until pinned."
    fi
  fi
fi

_BRAIN_SYNC_MODE=$("$_BRAIN_CONFIG_BIN" get artifacts_sync_mode 2>/dev/null || echo off)

# Detect remote-MCP mode (Path 4 of /setup-gbrain). Local artifacts sync is
# a no-op in remote mode; the brain server pulls from GitHub/GitLab on its
# own cadence. Read claude.json directly to keep this preamble fast (no
# subprocess to claude CLI on every skill start).
_GBRAIN_MCP_MODE="none"
if command -v jq >/dev/null 2>&1 && [ -f "$HOME/.claude.json" ]; then
  _GBRAIN_MCP_TYPE=$(jq -r '.mcpServers.gbrain.type // .mcpServers.gbrain.transport // empty' "$HOME/.claude.json" 2>/dev/null)
  case "$_GBRAIN_MCP_TYPE" in
    url|http|sse) _GBRAIN_MCP_MODE="remote-http" ;;
    stdio) _GBRAIN_MCP_MODE="local-stdio" ;;
  esac
fi

if [ -f "$_BRAIN_REMOTE_FILE" ] && [ ! -d "$_GSTACK_HOME/.git" ] && [ "$_BRAIN_SYNC_MODE" = "off" ]; then
  _BRAIN_NEW_URL=$(head -1 "$_BRAIN_REMOTE_FILE" 2>/dev/null | tr -d '[:space:]')
  if [ -n "$_BRAIN_NEW_URL" ]; then
    echo "ARTIFACTS_SYNC: artifacts repo detected: $_BRAIN_NEW_URL"
    echo "ARTIFACTS_SYNC: run 'gstack-brain-restore' to pull your cross-machine artifacts (or 'gstack-config set artifacts_sync_mode off' to dismiss forever)"
  fi
fi

if [ -d "$_GSTACK_HOME/.git" ] && [ "$_BRAIN_SYNC_MODE" != "off" ]; then
  _BRAIN_LAST_PULL_FILE="$_GSTACK_HOME/.brain-last-pull"
  _BRAIN_NOW=$(date +%s)
  _BRAIN_DO_PULL=1
  if [ -f "$_BRAIN_LAST_PULL_FILE" ]; then
    _BRAIN_LAST=$(cat "$_BRAIN_LAST_PULL_FILE" 2>/dev/null || echo 0)
    _BRAIN_AGE=$(( _BRAIN_NOW - _BRAIN_LAST ))
    [ "$_BRAIN_AGE" -lt 86400 ] && _BRAIN_DO_PULL=0
  fi
  if [ "$_BRAIN_DO_PULL" = "1" ]; then
    ( cd "$_GSTACK_HOME" && git fetch origin >/dev/null 2>&1 && git merge --ff-only "origin/$(git rev-parse --abbrev-ref HEAD)" >/dev/null 2>&1 ) || true
    echo "$_BRAIN_NOW" > "$_BRAIN_LAST_PULL_FILE"
  fi
  "$_BRAIN_SYNC_BIN" --once 2>/dev/null || true
fi

if [ "$_GBRAIN_MCP_MODE" = "remote-http" ]; then
  # Remote-MCP mode: local artifacts sync is a no-op (brain admin's server
  # pulls from GitHub/GitLab). Show the user this is by design, not broken.
  _GBRAIN_HOST=$(jq -r '.mcpServers.gbrain.url // empty' "$HOME/.claude.json" 2>/dev/null | sed -E 's|^https?://([^/:]+).*|\1|')
  echo "ARTIFACTS_SYNC: remote-mode (managed by brain server ${_GBRAIN_HOST:-remote})"
elif [ -d "$_GSTACK_HOME/.git" ] && [ "$_BRAIN_SYNC_MODE" != "off" ]; then
  _BRAIN_QUEUE_DEPTH=0
  [ -f "$_GSTACK_HOME/.brain-queue.jsonl" ] && _BRAIN_QUEUE_DEPTH=$(wc -l < "$_GSTACK_HOME/.brain-queue.jsonl" | tr -d ' ')
  _BRAIN_LAST_PUSH="never"
  [ -f "$_GSTACK_HOME/.brain-last-push" ] && _BRAIN_LAST_PUSH=$(cat "$_GSTACK_HOME/.brain-last-push" 2>/dev/null || echo never)
  echo "ARTIFACTS_SYNC: mode=$_BRAIN_SYNC_MODE | last_push=$_BRAIN_LAST_PUSH | queue=$_BRAIN_QUEUE_DEPTH"
else
  echo "ARTIFACTS_SYNC: off"
fi
```



隐私停止门：如果输出显示 `ARTIFACTS_SYNC: off`、`artifacts_sync_mode_prompted` 为 `false`，且 gbrain 在 PATH 中或 `gbrain doctor --fast --json` 可用，询问一次：

> gstack 可以将你的产物（CEO计划、设计稿、报告）发布到一个私有 GitHub 仓库，GBrain 会在多设备间索引。希望同步多少？

选项：
- A) 全部允许列表中的内容（推荐）
- B) 仅产物
- C) 拒绝，保持所有内容本地存储

回答后：

```bash
# 选择的模式：full | artifacts-only | off
"$_BRAIN_CONFIG_BIN" set artifacts_sync_mode <choice>
"$_BRAIN_CONFIG_BIN" set artifacts_sync_mode_prompted true
```

如果 A/B 且 `~/.gstack/.git` 缺失，询问是否运行 `gstack-artifacts-init`。不要阻塞技能执行。

技能结束、遥测之前：
```bash
"~/.claude/skills/gstack/bin/gstack-brain-sync" --discover-new 2>/dev/null || true
"~/.claude/skills/gstack/bin/gstack-brain-sync" --once 2>/dev/null || true
```


## 模型特定行为补丁（claude）

以下微调用于 claude 模型系列。它们**从属于**技能工作流、STOP 点、AskUserQuestion 门、计划模式安全逻辑，以及 /ship 审查门。如果以下微调与技能指令冲突，以技能为准。将这些视为偏好而非规则。

**Todo 清单纪律。** 执行多步计划时，每完成一个任务就单独标记完成。不要在最后批量完成。如果某个任务最终是不必要的，标记跳过并附一行原因。

**重操作前思考。** 对于复杂操作（重构、迁移、非平凡的新功能），先简要说明你的方案再执行。这样用户可以低成本地中途修正。

**专用工具优于 Bash。** 优先使用 Read、Edit、Write、Glob、Grep 而非 shell 等效命令（cat、sed、find、grep）。专用工具更便宜且更清晰。

## 语气（Voice）

直接、具体，builder 对 builder。列出文件名、函数名、命令和影响用户可见的效果。不废话。

不使用破折号。不使用 AI 词汇：delve、crucial、robust、comprehensive、nuanced、multifaceted。不学术、不公司化。短段落。以可执行的建议结束。

用户有你没有的上下文。跨模型的建议是推荐而非决定。用户做决定。

## 完成状态协议

完成技能工作流时，使用以下状态之一：
- **DONE** — 已完成，有证据。
- **DONE_WITH_CONCERNS** — 已完成，但需列出问题。
- **BLOCKED** — 无法继续；说明阻塞原因及已尝试的方法。
- **NEEDS_CONTEXT** — 缺少信息；明确说明需要什么。

在 3 次尝试失败、不确定的安全相关更改，或无法确认范围时升级。格式：`STATUS`、`REASON`、`ATTEMPTED`、`RECOMMENDATION`。

## 运营自我改进

完成前，如果你发现了可持续的项目快捷方式或命令修复，可节省下次 5 分钟以上的时间，记录：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"SKILL_NAME","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

不要记录显而易见的事实或一次性瞬时错误。

## 遥测（最后运行）

工作流完成后，记录遥测。使用 frontmatter 中的技能 `name:`。OUTCOME 是 success/error/abort/unknown。

**计划模式例外——始终运行：** 该命令向 `~/.gstack/analytics/` 写入遥测，与 preamble 分析写入匹配。

运行此 bash：

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
# Session timeline: record skill completion (local-only, never sent anywhere)
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"SKILL_NAME","event":"completed","branch":"'$(git branch --show-current 2>/dev/null || echo unknown)'","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
# Local analytics (gated on telemetry setting)
if [ "$_TEL" != "off" ]; then
echo '{"skill":"SKILL_NAME","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","browse":"USED_BROWSE","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
# Remote telemetry (opt-in, requires binary)
if [ "$_TEL" != "off" ] && [ -x ~/.claude/skills/gstack/bin/gstack-telemetry-log ]; then
  ~/.claude/skills/gstack/bin/gstack-telemetry-log \
    --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
    --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
fi
```

替换 `SKILL_NAME`、`OUTCOME` 和 `USED_BROWSE` 后运行。

## 计划模式状态页脚

运行计划审查技能（`/plan-*-review`、`/codex review`）的技能在末尾包含"退出计划模式门"阻塞检查清单，验证计划文件以 `## GSTACK REVIEW REPORT` 结尾后才调用 ExitPlanMode。不运行计划审查的技能（如 `/ship`、`/qa`、`/review` 等操作技能）通常不在计划模式下运行，也没有审查报告要验证；此页脚对其无影响。写入计划文件是计划模式下唯一允许的编辑。

# make-pdf: 从 markdown 生成出版级 PDF

将 `.md` 文件转换为像 Faber & Faber 文章集的 PDF：1 英寸边距、左对齐正文、全篇 Helvetica 字体、弯引号和破折号、可选封面页和点击式目录、需要时加对角线 DRAFT 水印。从 PDF 复制粘贴得到的是干净的文字，绝不是"S a i l i n g"。

在 Linux 上，需要安装 `fonts-liberation` 以确保正确渲染——Helvetica 和 Arial 默认不存在，Liberation Sans 是标准的度量兼容回退字体。CI 和 Docker 构建通过 Dockerfile.ci 自动安装它。

Emoji 需要彩色 emoji 字体。macOS（Apple Color Emoji）和 Windows（Segoe UI Emoji）自带一个；大多数 Linux 发行版和容器不自带，所以 emoji 会渲染成空框（▯）。`./setup` 在 Linux 上自动安装 `fonts-noto-color-emoji`（apt/dnf/pacman/apk，尽力而为），打印 CSS 会依次回退到 Apple / Segoe / Noto emoji 字体家族。设置 `GSTACK_SKIP_FONTS=1` 可跳过安装（无 sudo 的 CI、托管或离线机器）。

## 核心模式

### 80% 场景——备忘录/信函

一条命令，无参数。默认生成带页眉 + 页码 + CONFIDENTIAL 页脚的整洁 PDF。

```bash
$P generate letter.md                 # writes /tmp/letter.pdf
$P generate letter.md letter.pdf      # explicit output path
```

### 出版物模式——封面 + 目录 + 章节分页

```bash
$P generate --cover --toc --author "Garry Tan" --title "On Horizons" \
  essay.md essay.pdf
```

markdown 中每个顶层 H1 会开始新的一页。对于恰好有多个 H1 的备忘录，使用 `--no-chapter-breaks` 禁用此功能。

### 草稿阶段水印

```bash
$P generate --watermark DRAFT memo.md draft.pdf
```

每页都有对角线 10% 不透明度的 DRAFT 水印。草稿定稿后，删除此参数重新生成。

### 通过预览快速迭代

```bash
$P preview essay.md
```

使用相同的打印 CSS 渲染 HTML 并在浏览器中打开。编辑 markdown 时刷新即可。准备好之前跳过 PDF 往返。

### 无品牌（无 CONFIDENTIAL 页脚）

```bash
$P generate --no-confidential memo.md memo.pdf
```

### 图表——mermaid 和 excalidraw 围栏渲染为图片

markdown 中第 0 列的 ` ```mermaid ` 或 ` ```excalidraw ` 围栏会渲染为清晰的矢量图，完全离线（vendored bundle，无 CDN）。缩进的围栏（列表内）按设计保持为纯代码块。损坏的围栏会产生可见的红色诊断块，带有解析错误——绝不是静默的原始代码。

围栏 info-string 选项：

```
```mermaid title="Auth flow"        ← caption + aria-label
```mermaid render=false             ← keep it as a code block (today's behavior)
```mermaid page=landscape           ← force this diagram onto a landscape page
```mermaid page=portrait            ← veto auto-landscape for this diagram
```

A ` ```excalidraw ` 围栏包含一个完整的 .excalidraw 场景文件（excalidraw.com 保存的格式）。**从英文创作**新图表是 `/diagram` 的工作——它输出一个可编辑的三元组（source、.excalidraw、SVG/PNG）并与本技能配合：在 markdown 中嵌入 `.mmd` 源而非 PNG。

### 图片——正确缩放，绝不裁切

本地图片自动内联（相对路径根据 markdown 文件解析）。每张图片都被限制在内容框内——永远不裁切。超大照片降采样到打印分辨率（300dpi），文件体积保持小而视觉效果无损。

远程（http/https）图片**默认被阻止并显示可见占位符**——离线姿态；传入 `--allow-network` 以获取。解析到 markdown 文件目录之外的图片（即使通过符号链接）仍会内联，但会大声警告；`--strict` 使其成为致命错误。超过 64MB 的文件或非常规文件（fifo、设备）会降级为占位符，而非挂起运行。

每张图片的指令，紧跟在图片之后：

```
![chart](data.png){width=full}      ← stretch to content-box width
![chart](data.png){width=50%}       ← percentage or 3in/8cm/200px
![wide](arch.png){page=landscape}   ← give it its own landscape page
![wide](shot.png){page=portrait}    ← veto auto-landscape
```

宽幅、小文本的图表图片自动提升到横向页面（保守策略：宽高比 ≥ 1.8，宽度超过内容框约 2.5 倍，且 alt 文本为图表类词汇——diagram/architecture/flowchart/chart/graph）。提升的页面垂直居中。当启发式猜测错误时，`{page=portrait}` 可否决；误报只需添加 `{page=landscape}`。

### 其他格式——单文件 HTML 和 Word

```bash
$P generate readme.md out.html --to html    # ONE self-contained file: inline
                                            # SVG diagrams, data-URI images,
                                            # zero network refs, screen-readable
$P generate readme.md out.docx --to docx    # Word: content fidelity (headings,
                                            # tables, code, diagrams as PNG) —
                                            # layout is Word's, not ours
```

`--to` 是输出格式。`--format` 是另一个东西（`--page-size` 的别名）——不要混淆。

### CI 模式——缺失资源时报错退出

```bash
$P generate docs.md --strict     # missing, remote, out-of-tree, oversized,
                                 # and non-regular-file images exit non-zero
                                 # instead of warn + placeholder
```

## 常用参数

```
Page layout:
  --margins <dim>            1in (default) | 72pt | 2.54cm | 25mm
  --page-size letter|a4|legal

Structure:
  --cover                    Cover page (title, author, date, hairline rule)
  --toc                      Clickable TOC with page numbers
  --no-chapter-breaks        Don't start a new page at every H1

Branding:
  --watermark <text>         Diagonal watermark ("DRAFT", "CONFIDENTIAL")
  --header-template <html>   Custom running header
  --footer-template <html>   Custom footer (mutex with --page-numbers)
  --no-confidential          Suppress the CONFIDENTIAL right-footer

Output:
  --to pdf|html|docx         Output format (default: pdf). html = single
                             self-contained file; docx = content fidelity.
  --strict                   Missing, remote, out-of-tree, oversized, or
                             non-regular-file images fail the run (CI mode).
  --page-numbers             "N of M" footer (default on)
  --tagged                   Accessible PDF (default on)
  --outline                  PDF bookmarks from headings (default on)
  --quiet                    Suppress progress on stderr
  --verbose                  Per-stage timings

Network:
  --allow-network            Fetch external images. Off by default: remote
                             images render as a visible blocked placeholder
                             (no tracking pixels fetch at print time).

Metadata:
  --title "..."              Document title (defaults to first H1)
  --author "..."             Author for cover + PDF metadata
  --date "..."               Date for cover (defaults to today)
```

## Claude 何时运行

留意 markdown 转为 PDF 的意图。以下任何模式 → 运行 `$P generate`：

- "Can you make this markdown a PDF"
- "Export it as a PDF"
- "Turn this letter into a PDF"
- "I need a PDF of the essay"
- "Print this as a PDF for me"

如果用户打开了 `.md` 文件并说"make it look nice"，建议 `$P generate --cover --toc` 并先询问再运行。

## 调试

- 输出看起来为空/空白 → 检查 browse daemon 是否在运行：`$B status`。
- 复制粘贴时文字碎片化 → highlight.js 输出（Phase 4）。该标志存在后用 `--no-syntax` 重试。目前删除围栏代码块并重新生成。
- Paged.js 超时 → markdown 中可能没有标题。去掉 `--toc`。
- 输出中出现 "[remote image blocked]" 占位符 → 添加 `--allow-network`（理解你正在授予该 markdown 文件从其图片 URL 获取的权限）。
- 生成的 PDF 太高/太宽 → `--page-size a4` 或 `--margins 0.75in`。

## 输出约定

```
stdout: /tmp/letter.pdf          ← just the path, one line
stderr: Rendering HTML...        ← progress spinner (unless --quiet)
        Generating PDF...
        Done in 1.5s. 43 words · 22KB · /tmp/letter.pdf

exit code: 0 success / 1 bad args / 2 render error / 3 Paged.js timeout
           / 4 browse unavailable
```

捕获路径：`PDF=$($P generate letter.md)` — 然后使用 `$PDF`。
