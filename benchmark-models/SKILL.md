---
name: benchmark-models
preamble-tier: 1
version: 1.0.0
description: Cross-model benchmark for gstack skills. (gstack)
triggers:
  - cross model benchmark
  - compare claude gpt gemini
  - benchmark skill across models
  - which model should I use
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## 何时调用此 Skill

将同一个 prompt 分别通过 Claude、
GPT（通过 Codex CLI）和 Gemini 运行——对比延迟、token 用量、成本，
以及可选的 LLM 裁判评分。回答"这个 skill 到底哪个模型最合适？"
用数据说话，而不是凭感觉。与 /benchmark 不同，后者测量的是网页性能。
当遇到以下场景时使用："benchmark models"、"compare models"、
"which model is best for X"、"cross-model comparison"、"model shootout"。

语音触发（语音转文字别名）："compare models"、"model shootout"、"which model is best"。

## Preamble（首先运行）

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
echo '{"skill":"benchmark-models","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(_repo=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null | tr -cd 'a-zA-Z0-9._-'); echo "${_repo:-unknown}")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"benchmark-models","event":"started","branch":"'"$_BRANCH"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
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

## Plan Mode Safe Operations

在 plan 模式下，以下操作是允许的，因为它们为 plan 提供信息：`$B`、`$D`、`codex exec`/`codex review`、写入 `~/.gstack/`、写入 plan 文件，以及对生成产物执行 `open`。

## Skill 在 Plan 模式下的调用

如果用户在 plan 模式下调用 skill，skill 的优先级高于通用的 plan 模式行为。**将 skill 文件视为可执行的指令，而非参考文档。** 从 Step 0 开始逐步执行；第一次 AskUserQuestion 是工作流进入 plan 模式，而非违规。AskUserQuestion（任何变体 — `mcp__*__AskUserQuestion` 或原生变体；参见"AskUserQuestion 格式 → 工具解析"）满足 plan 模式的 end-of-turn 要求。如果 AskUserQuestion 不可用或调用失败，请遵循 AskUserQuestion 格式的降级处理：`headless` → BLOCKED；`interactive` → 散文降级（同样满足 end-of-turn）。在 STOP 点，立即停止。不要继续工作流或在那里调用 ExitPlanMode。标记为"PLAN MODE EXCEPTION — ALWAYS RUN"的命令会执行。仅在 skill 工作流完成后，或用户告诉你要取消 skill 或离开 plan 模式时，才调用 ExitPlanMode。

如果 `PROACTIVE` 为 `"false"`，不要自动调用或主动建议 skill。如果某个 skill 看起来有用，问："I think /skillname might help here — want me to run it?"

如果 `SKILL_PREFIX` 为 `"true"`，建议/调用 `/gstack-*` 名称。磁盘路径保持为 `~/.claude/skills/gstack/[skill-name]/SKILL.md`。

如果输出显示 `UPGRADE_AVAILABLE <old> <new>`：读取 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` 并遵循"Inline upgrade flow"（如果配置了则自动升级，否则使用 AskUserQuestion 提供 4 个选项，如果拒绝则写入 snooze 状态）。

如果输出显示 `JUST_UPGRADED <from> <to>`：打印 "Running gstack v{to} (just updated!)"。如果 `SPAWNED_SESSION` 为 true，跳过功能发现。

功能发现，每个 session 最多一个 prompt：
- 缺少 `~/.claude/skills/gstack/.feature-prompted-continuous-checkpoint`：询问 AskUserQuestion 是否启用 Continuous checkpoint auto-commits。如果接受，运行 `~/.claude/skills/gstack/bin/gstack-config set checkpoint_mode continuous`。始终创建标记文件。
- 缺少 `~/.claude/skills/gstack/.feature-prompted-model-overlay`：告知"Model overlays 已激活。MODEL_OVERLAY 显示补丁。"始终创建标记文件。

升级提示后，继续工作流。

如果 `WRITING_STYLE_PENDING` 为 `yes`：询问一次关于写作风格的问题：

> v1 prompts 更简单：首次使用时的术语缩写、以结果为导向的问题、更短的篇幅。保持默认还是恢复 terse？

选项：
- A) 保持新的默认值（推荐 — 好的写作对所有人都有帮助）
- B) 恢复 V0 风格 — 设置 `explain_level: terse`

如果 A：保持 `explain_level` 不设置（默认为 `default`）。
如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set explain_level terse`。

始终运行（无论选择哪个）：
```bash
rm -f ~/.gstack/.writing-style-prompt-pending
touch ~/.gstack/.writing-style-prompted
```

如果 `WRITING_STYLE_PENDING` 为 `no` 则跳过。

如果 `LAKE_INTRO` 为 `no`：说 "gstack 遵循 **Boil the Ocean** 原则 — 当 AI 使边际成本接近零时，做完整的事情。了解更多：https://garryslist.org/posts/boil-the-ocean" 提供打开选项：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

仅在用户确认时运行 `open`。始终运行 `touch`。

如果 `TEL_PROMPTED` 为 `no` 且 `LAKE_INTRO` 为 `yes`：通过 AskUserQuestion 询问一次遥测：

> 帮助 gstack 变得更好。仅分享使用数据：skill、持续时间、崩溃、稳定的设备 ID。不发送代码或文件路径。你的仓库名称仅本地记录，上传前会被剥离。

选项：
- A) 帮助 gstack 变得更好！（推荐）
- B) 不，谢谢

如果 A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果 B：追问：

> 匿名模式仅发送聚合使用数据，不包含唯一 ID。

选项：
- A) 好的，匿名可以
- B: 不，谢谢，完全关闭

如果 B→A：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果 B→B：运行 `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

如果 `TEL_PROMPTED` 为 `yes` 则跳过。

如果 `PROACTIVE_PROMPTED` 为 `no` 且 `TEL_PROMPTED` 为 `yes`：询问一次：

> 让 gstack 主动建议 skill，比如对"这个能用吗？"建议 /qa，对 bug 建议 /investigate？

选项：
- A) 保持开启（推荐）
- B) 关闭 — 我会自己输入 /commands

如果 A：运行 `~/.claude/skills/gstack/bin/gstack-config set proactive true`
如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set proactive false`

始终运行：
```bash
touch ~/.gstack/.proactive-prompted
```

如果 `PROACTIVE_PROMPTED` 为 `yes` 则跳过。

## 首次运行指导（一次性）

如果 `ACTIVATED` 为 `no`（此机器上的首次 skill 运行）且 preamble 打印了非空的 `FIRST_TASK:` 值且不是 `nongit`：显示一行简短的项目特定提示（从 token 映射），作为提醒，然后**继续**用户实际要求的任务——**不要**停止他们的任务。Token 映射：`greenfield` → "全新的仓库 — 先用 `/spec` 或 `/office-hours` 规划方向。" `code_node`/`code_python`/`code_rust`/`code_go`/`code_ruby`/`code_ios` → "这里有代码 — 用 `/qa` 看它运行，或者用 `/investigate` 排查问题。" `branch_ahead` → "此分支上有未发布的工作 — `/review` 然后 `/ship`。" `dirty_default` → "有未提交的更改 — 提交前先 `/review`。" `clean_default` → "选择一个：`/spec`、`/investigate` 或 `/qa`。" 然后将你看到的 token 替换为 TASK_TOKEN 并运行（尽力为止），并标记为已激活：
```bash
~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type first_task_scaffold_shown --skill "TASK_TOKEN" --outcome shown 2>/dev/null || true
touch ~/.gstack/.activated 2>/dev/null || true
```

如果 `ACTIVATED` 为 `no` 但 `FIRST_TASK:` 为空或为 `nongit`（headless、非 git 或无操作可做）：不显示任何内容，仅运行 `touch ~/.gstack/.activated 2>/dev/null || true`。

否则如果 `ACTIVATED` 为 `yes` 且 `FIRST_LOOP_SHOWN` 为 `no`：说一次作为提醒（然后继续）：

> 提示：gstack 在完成一次循环时最有价值 — **计划 → 审查 → 交付**。一个常见的首次循环：`/office-hours` 或 `/spec` 来规划，`/plan-eng-review` 来锁定，然后 `/ship`。

然后运行 `touch ~/.gstack/.first-loop-tip-shown 2>/dev/null || true`。

如果 `ACTIVATED` 和 `FIRST_LOOP_SHOWN` 都为 `yes`，则跳过此节。

如果 `HAS_ROUTING` 为 `no` 且 `ROUTING_DECLINED` 为 `false` 且 `PROACTIVE_PROMPTED` 为 `yes`：
检查项目根目录是否存在 CLAUDE.md 文件。如果不存在，创建一个。

使用 AskUserQuestion：

> gstack 在你的项目的 CLAUDE.md 包含 skill routing 规则时效果最好。

选项：
- A) 添加 routing 规则到 CLAUDE.md（推荐）
- B: 不，谢谢，我会手动调用 skill

如果 A：将此部分追加到 CLAUDE.md 末尾：

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

如果 B：运行 `~/.claude/skills/gstack/bin/gstack-config set routing_declined true` 并告诉他们可以用 `gstack-config set routing_declined false` 重新启用。

每个项目仅执行一次。如果 `HAS_ROUTING` 为 `yes` 或 `ROUTING_DECLINED` 为 `true` 则跳过。

如果 `VENDORED_GSTACK` 为 `yes`，除非 `~/.gstack/.vendoring-warned-$SLUG` 存在，否则通过 AskUserQuestion 警告一次：

> 此项目将 gstack 内置在 `.claude/skills/gstack/` 中。内置已弃用。
> 是否迁移到 team 模式？

选项：
- A) 是的，现在迁移到 team 模式
- B: 不，我自己处理

如果 A：
1. 运行 `git rm -r .claude/skills/gstack/`
2. 运行 `echo '.claude/skills/gstack/' >> .gitignore`
3. 运行 `~/.claude/skills/gstack/bin/gstack-team-init required`（或 `optional`）
4. 运行 `git add .claude/ .gitignore CLAUDE.md && git commit -m "chore: migrate gstack from vendored to team mode"`
5. 告诉用户："完成。每个开发者现在运行：`cd ~/.claude/skills/gstack && ./setup --team`"

如果 B：说 "OK，你自己负责保持内置副本最新。"

始终运行（无论选择哪个）：
```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
touch ~/.gstack/.vendoring-warned-${SLUG:-unknown}
```

如果标记文件存在，则跳过。

如果 `SPAWNED_SESSION` 为 `"true"`，你正在由 AI 编排器（如 OpenClaw）启动的 session 中运行。在生成的 session 中：
- 不要对交互式提示使用 AskUserQuestion。自动选择推荐选项。
- 不要运行升级检查、遥测提示、routing 注入或 lake 介绍。
- 专注于完成任务并通过散文输出报告结果。
- 以完成报告结束：交付了什么、做了什么决策、有什么不确定。

## 产物同步（skill 启动时）

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

Privacy stop-gate：如果输出显示 `ARTIFACTS_SYNC: off`，且 `artifacts_sync_mode_prompted` 为 `false`，且 gbrain 在 PATH 上或 `gbrain doctor --fast --json` 可用，询问一次：

> gstack 能否将你的产物（CEO 计划、设计、报告）发布到一个 GBrain 跨机器索引的私有 GitHub 仓库？应该同步多少？

选项：
- A) 全部白名单内容（推荐）
- B) 仅产物
- C) 拒绝，保持所有内容本地存储

回答后：

```bash
# Chosen mode: full | artifacts-only | off
"$_BRAIN_CONFIG_BIN" set artifacts_sync_mode <choice>
"$_BRAIN_CONFIG_BIN" set artifacts_sync_mode_prompted true
```

如果 A/B 且 `~/.gstack/.git` 丢失，询问是否运行 `gstack-artifacts-init`。不要阻塞 skill。

skill 结束前，遥测之前：

```bash
"~/.claude/skills/gstack/bin/gstack-brain-sync" --discover-new 2>/dev/null || true
"~/.claude/skills/gstack/bin/gstack-brain-sync" --once 2>/dev/null || true
```

## 模型特定行为补丁（claude）

以下调整针对 claude 模型系列进行了优化。它们**从属于** skill 工作流、STOP 点、AskUserQuestion 门控、plan 模式安全以及 /ship review 门控。如果下面的调整与 skill 指令冲突，skill 优先。将这些视为偏好，而非规则。

**Todo-list 纪律。** 在多步骤计划中工作时，完成每个任务时单独标记为已完成。不要在最后批量完成。如果某个任务被证明是不必要的，用一行原因标记为跳过。

**操作前先思考。** 对于复杂操作（重构、迁移、非平凡的新功能），在执行之前简要说明你的方法。这允许用户以低成本修正方向，而不是在飞行中途改出。

**专用工具优先于 Bash。** 优先使用 Read、Edit、Write、Glob、Grep 而非 shell 等效命令（cat、sed、find、grep）。专用工具更经济、更清晰。

## 语气

直接、具体、builder 对 builder。指出文件、函数、命令和用户可见的影响。不要填充词。

不使用破折号。不使用 AI 词汇：delve、crucial、robust、comprehensive、nuanced、multifaceted。永远不要公司化或学术化。短段落。以该做什么结尾。

用户有你没有的上下文。跨模型一致的答案是建议，而非决策。用户做决策。

## 完成状态协议

完成 skill 工作流时，使用以下状态之一报告状态：
- **DONE** — 有证据地完成。
- **DONE_WITH_CONCERNS** — 完成，但列出顾虑。
- **BLOCKED** — 无法继续；说明阻塞原因和已尝试的方法。
- **NEEDS_CONTEXT** — 缺少信息；准确说明需要什么。

在 3 次失败尝试后、不确定的安全敏感更改时，或无法验证的范围上升级。格式：`STATUS`、`REASON`、`ATTEMPTED`、`RECOMMENDATION`。

## 操作自我改进

完成前，如果你发现了可以节省下次 5 分钟以上的持久项目特性或命令修复，记录下来：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"SKILL_NAME","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

不要记录显而易见的事实或一次性短暂错误。

## 遥测（最后运行）

工作流完成后，记录遥测。使用 frontmatter 中的 skill `name:`。OUTCOME 为 success/error/abort/unknown。

**PLAN MODE EXCEPTION — ALWAYS RUN：** 此命令将遥测写入
`~/.gstack/analytics/`，与 preamble 分析写入匹配。

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

运行前替换 `SKILL_NAME`、`OUTCOME` 和 `USED_BROWSE`。

## Plan Status Footer

运行 plan review 的 skill（`/plan-*-review`、`/codex review`）在 skill 末尾包含 EXIT PLAN MODE GATE 阻塞清单，该清单在调用 ExitPlanMode 之前验证 plan 文件以 `## GSTACK REVIEW REPORT` 结尾。不运行 plan review 的 skill（如 `/ship`、`/qa`、`/review` 等操作类 skill）通常不在 plan 模式下运行，也没有需要验证的 review 报告；此 footer 对它们来说是 no-op。写入 plan 文件是 plan 模式中唯一允许的编辑操作。

# /benchmark-models — 跨模型 Skill Benchmark

你正在运行 `/benchmark-models` 工作流。用交互式流程包装 `gstack-model-benchmark` 二进制文件，选择 prompt、确认 provider、预览认证并运行 benchmark。

与 `/benchmark` 不同 — 那个 skill 测量网页性能（Core Web Vitals、加载时间）。此 skill 测量 AI 模型在 gstack skills 或任意 prompt 上的性能。

---

## Step 0：定位二进制文件

```bash
BIN="$HOME/.claude/skills/gstack/bin/gstack-model-benchmark"
[ -x "$BIN" ] || BIN=".claude/skills/gstack/bin/gstack-model-benchmark"
[ -x "$BIN" ] || { echo "ERROR: gstack-model-benchmark not found. Run ./setup in the gstack install dir." >&2; exit 1; }
echo "BIN: $BIN"
```

如果找不到，停止并告诉用户重新安装 gstack。

---

## Step 1：选择 prompt

使用带有 preamble 格式的 AskUserQuestion：
- **Re-ground：** 当前项目 + 分支。
- **Simplify：** "跨模型 benchmark 将同一个 prompt 通过 2-3 个 AI 模型运行，并展示它们在速度、成本和输出质量方面的对比。我们应该使用什么 prompt？"
- **RECOMMENDATION：** A，因为针对真实 skill 进行 benchmark 能暴露工具使用的差异，而不仅仅是原始生成能力。
- **选项：**
  - A) 对我的一个 gstack skill 进行 benchmark（接下来选哪个 skill）。完整度：10/10。
  - B) 使用内联 prompt — 下一轮输入。完整度：8/10。
  - C) 指向磁盘上的 prompt 文件 — 下一轮指定路径。完整度：8/10。

如果 A：列出具有 SKILL.md 文件的顶级 gstack skill（通过 `find . -maxdepth 2 -name SKILL.md -not -path './.*'`），要求用户通过第二个 AskUserQuestion 选择一个。使用选定的 SKILL.md 路径作为 prompt 文件。

如果 B：要求用户提供内联 prompt。通过 `--prompt "<text>"` 逐字使用。

如果 C：要求提供路径。验证其存在。作为位置参数使用。

---

## Step 2：选择 providers

```bash
"$BIN" --prompt "unused, dry-run" --models claude,gpt,gemini --dry-run
```

显示 dry-run 结果。"Adapter availability"部分会告诉用户哪些 provider 将实际运行（OK）vs 跳过（NOT READY — 包含修复提示）。

如果全部三个都显示 NOT READY：停止并给出明确的提示 — 没有至少一个已认证的 provider，benchmark 无法运行。建议 `claude login`、`codex login` 或 `gemini login` / `export GOOGLE_API_KEY`。

如果至少有一个为 OK：AskUserQuestion：
- **Simplify：** "我们应该包含哪些模型？上面的 dry-run 显示了哪些已认证。未认证的会被干净跳过 — 它们不会中止批量运行。"
- **RECOMMENDATION：** A（所有已认证的 provider），因为运行尽可能多的 provider 能给出最丰富的对比。
- **选项：**
  - A) 所有已认证的 provider。完整度：10/10。
  - B) 仅 Claude。完整度：6/10（无跨模型信号 — 使用 /ship 的 review 进行单独的 claude benchmark）。
  - C) 选两个 — 下一轮指定。完整度：8/10。

---

## Step 3：决定是否使用 judge

```bash
[ -n "$ANTHROPIC_API_KEY" ] || grep -q 'ANTHROPIC' "$HOME/.claude/.credentials.json" 2>/dev/null && echo "JUDGE_AVAILABLE" || echo "JUDGE_UNAVAILABLE"
```

如果 judge 可用，AskUserQuestion：
- **Simplify：** "质量裁判使用 Anthropic 的 Claude 作为平局打破者，在 0-10 的范围内为每个模型的输出评分。每次运行增加约 $0.05。如果你关心输出质量（不仅仅是延迟和成本），推荐使用。"
- **RECOMMENDATION：** A — 整个目的就是比较质量，而不仅仅是速度。
- **选项：**
  - A) 启用 judge（增加约 $0.05）。完整度：10/10。
  - B) 跳过 judge — 仅速度/成本/tokens。完整度：7/10。

如果 judge 不可用，跳过此问题并省略 `--judge` 标志。

---

## Step 4：运行 benchmark

从 Step 1、2、3 的决策构建命令：

```bash
"$BIN" <prompt-spec> --models <picked-models> [--judge] --output table
```

其中 `<prompt-spec>` 是 `--prompt "<text>"`（Step 1B）或文件路径（Step 1A 或 1C），`<picked-models>` 是来自 Step 2 的逗号分隔列表。

输出到达时流式传输。这很慢 — 每个 provider 都会完整运行 prompt。根据 prompt 复杂度和是否开启 `--judge`，预计 30 秒到 5 分钟。

---

## Step 5：解读结果

表格打印后，为用户总结：
- **最快** — 延迟最低的 provider。
- **最便宜** — 成本最低的 provider。
- **最高质量**（如果 `--judge` 运行了）— 得分最高的 provider。
- **最佳整体** — 使用判断。如果 judge 运行了：按质量加权。否则：指出用户需要做的权衡。

如果任何 provider 遇到错误（auth/timeout/rate_limit），指出并给出修复路径。

---

## Step 6：提供保存结果的选项

AskUserQuestion：
- **Simplify：** "将此 benchmark 保存为 JSON，以便与未来的运行进行比较？"
- **RECOMMENDATION：** A — 随着 provider 更新模型，skill 性能会漂移；保存的基线能捕获质量回归。
- **选项：**
  - A) 保存到 `~/.gstack/benchmarks/<date>-<skill-or-prompt-slug>.json`。完整度：10/10。
  - B) 仅打印，不保存。完整度：5/10（丢失趋势数据）。

如果 A：用 `--output json` 重新运行并 tee 到带日期的文件。打印路径，以便用户与未来的运行做 diff。

---

## 重要规则

- **绝不**在 Step 2 的 dry-run 之前运行真正的 benchmark。用户需要在花费 API 调用前看到认证状态。
- **绝不**硬编码模型名称。始终从用户的 Step 2 选择中传入 provider — 二进制文件会处理其余部分。
- **绝不**自动包含 `--judge`。它增加真实成本；用户必须主动选择加入。
- **如果零个 provider 已认证，停止。** 不要尝试 benchmark — 它不会产生有用的输出。
- **成本可见。** 每次运行都会在表格中显示每个 provider 的成本。用户应在下一次运行前看到它。
