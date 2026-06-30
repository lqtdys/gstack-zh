---
name: ios-qa
preamble-tier: 3
version: 1.0.0
description: Live-device iOS QA for SwiftUI apps. (gstack)
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
triggers:
  - ios qa
  - test the iphone app
  - test my ios app
  - find bugs on the device
  - qa the ios app
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## 何时调用此技能

通过USB连接真实的iPhone，建立CoreDevice IPv6隧道，读取Swift源码以理解每个界面，然后运行视觉驱动的agent循环：截图 → 分析 → 决策 → 操作 → 验证 → 重复。所有交互均通过HTTP在被测应用内嵌的StateServer上进行。可选地通过Tailscale暴露设备，使远程agent（OpenClaw、Codex或任何支持HTTP的agent）可以从任何地方运行iOS QA，无需接触硬件。

当被要求"ios qa"、"test my iPhone app"、"find bugs on the device"或"qa the iOS app"时使用。

语音触发词（语音转文本别名）："iOS quality check"、"test the iPhone app"、"run iOS QA"。

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
# Conductor host: 此处AskUserQuestion不可靠（native禁用，MCP
# 变体不稳定），因此skill以散文形式呈现决策，而非调用该工具。
# 仅在非headless时启用，以便评估/CI运行在Conductor内（GSTACK_HEADLESS）
# 仍会BLOCK，而非向无人的地方输出散文。
if [ "$_SESSION_KIND" != "headless" ] && { [ -n "${CONDUCTOR_WORKSPACE_PATH:-}" ] || [ -n "${CONDUCTOR_PORT:-}" ]; }; then
  echo "CONDUCTOR_SESSION: true"
fi
_ACTIVATED=$([ -f ~/.gstack/.activated ] && echo "yes" || echo "no")
_FIRST_LOOP_SHOWN=$([ -f ~/.gstack/.first-loop-tip-shown ] && echo "yes" || echo "no")
echo "ACTIVATED: $_ACTIVATED"
echo "FIRST_LOOP_SHOWN: $_FIRST_LOOP_SHOWN"
# 首次运行项目检测：仅在第一次运行skill时运行检测器
# （ACTIVATED=no, interactive），以便后续每次运行都不在热路径上。
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
echo '{"skill":"ios-qa","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(_repo=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null | tr -cd 'a-zA-Z0-9._-'); echo "${_repo:-unknown}" )'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"ios-qa","event":"started","branch":"'"$_BRANCH"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null &
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
# Plan模式提示：用于/spec等根据plan模式状态分支行为的skill。
# Claude Code通过system reminder暴露plan模式；我们通过CLAUDE_PLAN_FILE尽力检测
# （harness在plan模式激活时设置）并回退到"inactive"。Codex主机和Claude执行模式
# 两者最终都是inactive，这是安全的默认值（回退到file+execute管道）。
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

## Plan模式安全操作

在plan模式下，以下操作被允许，因为它们为计划提供信息：`$B`、`$D`、`codex exec`/`codex review`，写入`~/.gstack/`，写入计划文件，以及`open`用于生成的artifact。

## Plan模式期间的Skill调用

如果用户在plan模式中调用skill，该skill优先于通用plan模式行为。**将skill文件视为可执行指令，而非参考文档。** 从步骤0开始逐步执行；第一个AskUserQuestion是工作流进入plan模式，而非违反它。AskUserQuestion（任何变体——`mcp__*__AskUserQuestion`或native；参见"AskUserQuestion Format → Tool resolution"）满足plan模式的回合结束要求。如果AskUserQuestion不可用或调用失败，遵循AskUserQuestion Format失败回退：`headless` → BLOCKED；`interactive` → 散文回退（也满足回合结束要求）。在STOP点，立即停止。不要继续工作流或在其中调用ExitPlanMode。标记为"PLAN MODE EXCEPTION — ALWAYS RUN"的命令执行。仅在skill工作流完成后，或用户告诉你取消skill或离开plan模式时调用ExitPlanMode。

如果`PROACTIVE`为`"false"`，不要自动调用或主动建议skill。如果skill可能有帮助，询问："我觉得/skillname可能有用——要我运行它吗？"

如果`SKILL_PREFIX`为`"true"`，建议/调用`/gstack-*`名称。磁盘路径保持为`~/.claude/skills/gstack/[skill-name]/SKILL.md`。

如果输出显示`UPGRADE_AVAILABLE <old> <new>`：读取`~/.claude/skills/gstack/gstack-upgrade/SKILL.md`并遵循"Inline upgrade flow"（如果配置了则自动升级，否则AskUserQuestion提供4个选项，如果拒绝则写入snooze状态）。

如果输出显示`JUST_UPGRADED <from> <to>`：打印"Running gstack v{to} (just updated!)"。如果`SPAWNED_SESSION`为true，跳过功能发现。

功能发现，每个会话最多提示一次：
- 缺失`~/.claude/skills/gstack/.feature-prompted-continuous-checkpoint`：AskUserQuestion询问Continuous checkpoint自动提交。如果接受，运行`~/.claude/skills/gstack/bin/gstack-config set checkpoint_mode continuous`。始终touch标记。
- 缺失`~/.claude/skills/gstack/.feature-prompted-model-overlay`：告知"Model overlays are active. MODEL_OVERLAY shows the patch." 始终touch标记。

升级提示后，继续工作流。

如果`WRITING_STYLE_PENDING`为`yes`：询问一次关于写作风格：

> v1提示更简单：首次使用术语表、结果导向的问题、较短的散文。保持默认还是恢复简洁？

选项：
- A) 保持新的默认值（推荐——好的写作对每个人都有帮助）
- B) 恢复V0散文——设置`explain_level: terse`

如果A：不设置`explain_level`（默认为`default`）。
如果B：运行`~/.claude/skills/gstack/bin/gstack-config set explain_level terse`。

始终运行（无论选择）：
```bash
rm -f ~/.gstack/.writing-style-prompt-pending
touch ~/.gstack/.writing-style-prompted
```

如果`WRITING_STYLE_PENDING`为`no`则跳过。

如果`LAKE_INTRO`为`no`：说"gstack遵循**Boil the Ocean**原则——当AI使边际成本接近零时，做完整的事情。了解更多：https://garryslist.org/posts/boil-the-ocean" 提供打开：

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

仅在yes时运行`open`。始终运行`touch`。

如果`TEL_PROMPTED`为`no`且`LAKE_INTRO`为`yes`：通过AskUserQuestion询问一次遥测：

> 帮助gstack改进。仅分享使用数据：技能、持续时间、崩溃、稳定的设备ID。不收集代码或文件路径。你的仓库名称仅记录本地，上传前剥离。

选项：
- A) 帮助gstack改进！（推荐）
- B) 不，谢谢

如果A：运行`~/.claude/skills/gstack/bin/gstack-config set telemetry community`

如果B：询问后续问题：

> 匿名模式仅发送聚合使用数据，不发送唯一ID。

选项：
- A) 好的，匿名没问题
- B) 不，完全关闭

如果B→A：运行`~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
如果B→B：运行`~/.claude/skills/gstack/bin/gstack-config set telemetry off`

始终运行：
```bash
touch ~/.gstack/.telemetry-prompted
```

如果`TEL_PROMPTED`为`yes`则跳过。

如果`PROACTIVE_PROMPTED`为`no`且`TEL_PROMPTED`为`yes`：询问一次：

> 让gstack主动建议skill，例如"does this work?"用/qa，bug用/investigate？

选项：
- A) 保持开启（推荐）
- B) 关闭——我会自己输入/命令

如果A：运行`~/.claude/skills/gstack/bin/gstack-config set proactive true`
如果B：运行`~/.claude/skills/gstack/bin/gstack-config set proactive false`

始终运行：
```bash
touch ~/.gstack/.proactive-prompted
```

如果`PROACTIVE_PROMPTED`为`yes`则跳过。

## 首次运行指导（一次性）

如果`ACTIVATED`为`no`（此机器上第一次运行skill）且preamble打印了非空的`FIRST_TASK:`值且不是`nongit`：显示一条简短、项目特定的行（从token映射），作为提示，然后**继续**用户实际要求的任务——不要阻碍他们的任务。映射token：`greenfield` → "全新仓库——先用`/spec`或`/office-hours`规划。" `code_node`/`code_python`/`code_rust`/`code_go`/`code_ruby`/`code_ios` → "这里有代码——用`/qa`看它运行，或用`/investigate`如果有问题。" `branch_ahead` → "此分支上有未发布的工作——`/review`然后`/ship`。" `dirty_default` → "未提交的更改——提交前先`/review`。" `clean_default` → "选一个：`/spec`、`/investigate`、或`/qa`。" 然后将你看到的token替换为TASK_TOKEN并运行（尽力），并标记已激活：
```bash
~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type first_task_scaffold_shown --skill "TASK_TOKEN" --outcome shown 2>/dev/null || true
touch ~/.gstack/.activated 2>/dev/null || true
```

如果`ACTIVATED`为`no`但`FIRST_TASK:`为空或为`nongit`（headless、非git、或无可操作内容）：不显示任何内容，仅运行`touch ~/.gstack/.activated 2>/dev/null || true`。

否则如果`ACTIVATED`为`yes`且`FIRST_LOOP_SHOWN`为`no`：显示一次作为提示（然后继续）：

> 提示：完成一个循环后gstack就能见效——**计划 → 审查 → 发布**。常见的第一个循环：`/office-hours`或`/spec`来规划，`/plan-eng-review`来锁定，然后`/ship`。

然后运行`touch ~/.gstack/.first-loop-tip-shown 2>/dev/null || true`。

如果`ACTIVATED`和`FIRST_LOOP_SHOWN`都为`yes`则跳过此部分。

如果`HAS_ROUTING`为`no`且`ROUTING_DECLINED`为`false`且`PROACTIVE_PROMPTED`为`yes`：
检查CLAUDE.md文件是否存在于项目根目录。如果不存在，创建它。

使用AskUserQuestion：

> gstack在你的项目的CLAUDE.md中包含skill路由规则时效果最佳。

选项：
- A) 添加路由规则到CLAUDE.md（推荐）
- B) 不，我会手动调用skill

如果A：将此部分追加到CLAUDE.md末尾：

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

如果B：运行`~/.claude/skills/gstack/bin/gstack-config set routing_declined true`并告知他们可以用`gstack-config set routing_declined false`重新启用。

此操作每个项目仅发生一次。如果`HAS_ROUTING`为`yes`或`ROUTING_DECLINED`为`true`则跳过。

如果`VENDORED_GSTACK`为`yes`，通过AskUserQuestion警告一次，除非`~/.gstack/.vendoring-warned-$SLUG`存在：

> 此项目将gstack vending在`.claude/skills/gstack/`中。Vendoring已弃用。
> 迁移到team模式？

选项：
- A) 是的，立即迁移到team模式
- B) 不，我自己处理

如果A：
1. 运行`git rm -r .claude/skills/gstack/`
2. 运行`echo '.claude/skills/gstack/' >> .gitignore`
3. 运行`~/.claude/skills/gstack/bin/gstack-team-init required`（或`optional`）
4. 运行`git add .claude/ .gitignore CLAUDE.md && git commit -m "chore: migrate gstack from vendored to team mode"`
5. 告知用户："完成。现在每个开发者运行：`cd ~/.claude/skills/gstack && ./setup --team`"

如果B：说"好的，你自己负责保持vendored副本更新。"

始终运行（无论选择）：
```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
touch ~/.gstack/.vendoring-warned-${SLUG:-unknown}
```

如果标记存在则跳过。

如果`SPAWNED_SESSION`为`"true"`，你正在由AI协调器（如OpenClaw）生成的会话内运行。在生成的会话中：
- 不要对交互式提示使用AskUserQuestion。自动选择推荐选项。
- 不要运行升级检查、遥测提示、路由注入或lake介绍。
- 专注于完成任务并通过散文输出报告结果。
- 以完成报告结束：发布了什么、做了什么决策、有什么不确定的。

## AskUserQuestion Format

### Tool resolution（首先读取）

"AskUserQuestion"在运行时可以解析为两个tool：**host MCP变体**（如`mcp__conductor__AskUserQuestion`——当host注册它时出现在你的tool列表中）或**native** Claude Code工具。

**Conductor规则（在MCP规则之前读取）：** 如果preamble回显了`CONDUCTOR_SESSION: true`，根本不要调用AskUserQuestion——既不是native也不是任何`mcp__*__AskUserQuestion`变体。将每个决策简短地呈现为下面的**散文形式**并停止。这是主动的，而非对失败的反应：Conductor禁用了native AUQ，其MCP变体不稳定（它返回`[Tool result missing due to internal error]`），因此散文是可靠的路径。**自动决定偏好仍然首先适用：** 如果问题的`[plan-tune auto-decide] <id> → <option>`结果已经出现，继续该选项（不使用散文）。因为在Conductor中你直接去散文而不调用tool，这个auto-decide-first排序在这里强制执行，不仅由PreToolUse hook。当你呈现Conductor散文简报时，也使用`bin/gstack-question-log`捕获它（PostToolUse捕获hook在散文路径上从不触发，因此`/plan-tune`历史/学习依赖于这个调用）。

**规则（非Conductor）：** 如果任何`mcp__*__AskUserQuestion`变体在你的tool列表中，优先使用它。Host可能通过`--disallowedTools AskUserQuestion`禁用native AUQ（Conductor默认这样做）并通过其MCP变体路由；在那里调用native会静默失败。相同的问题/选项形状；相同的decision-brief格式适用。

如果AskUserQuestion不可用（你的tool列表中无变体）或调用失败，不要静默自动决定或将决策写入计划文件作为替代。遵循下面的**失败回退**。

### 当AskUserQuestion不可用或调用失败时

区分三种结果：

1. **自动决定拒绝（不是失败）。** 结果包含`[plan-tune auto-decide] <id> → <option>`——偏好hook按设计工作。继续该选项。不要重试，不要回退到散文。
2. **真正的失败**——你的tool列表中无变体，或变体存在但调用返回错误/缺少结果（MCP传输错误、空结果、host bug——如Conductor的MCP AskUserQuestion不稳定并返回`[Tool result missing due to internal error]`）。
   - 如果它存在且**出错**（不是缺席），重试相同的调用**一次**——但仅在答案不可能已经出现的情况下（缺少结果的错误可能在用户已经看到问题后到达；重新尝试会双重提示，所以如果它可能已经到达他们，视为待定，不要重试）。
   - 然后在`SESSION_KIND`上分支（preamble回显；空/缺席 ⇒ `interactive`）：
     - `spawned` → 服从**生成会话**块：自动选择推荐选项。从不散文，从不BLOCKED。
     - `headless` → `BLOCKED — AskUserQuestion unavailable`；停止并等待（无人可以回答）。
     - `interactive` → **散文回退**（下面）。

**散文回退——将决策简报呈现为markdown消息，而非tool调用。** 与下面的工具格式相同的信息，不同的结构（段落，不是✅/❌项目符号）。它必须呈现这个三元组：

1. **问题本身的清晰ELI10**——关于决定什么以及为什么重要的纯英语（问题，不是每个选项），命名风险。以它开头。
2. **每个选项的完整性分数**——明确`Completeness: X/10`在每个选项上（10完整，7快乐路径，3捷径）；当选项在种类上不同而非覆盖时使用kind-note，但永远不要悄悄丢弃分数。
3. **推荐及原因**——`Recommendation: <choice> because <reason>`行加上该选项上的`(recommended)`标记。

布局：`D<N>`标题 + 一行回复字母的注释（在Conductor中这是正常路径；在其他地方它意味着AskUserQuestion不可用或出错）；问题ELI10；Recommendation行；然后每个选项一个段落，带有`(recommended)`标记、`Completeness: X/10`和2-4句推理——永远不是简单的项目符号列表；一个闭合的`Net:`行。分割链/5+选项：每个单一选项调用的散文块，按顺序。然后停止并等待——用户的打字答案是决策。在plan模式中这如同tool调用一样满足回合结束。

**继续——将打字回复映射回简报。** 每个简报有一个稳定标签（`D<N>`或分割链中的`D<N>.k`）。用户引用它（例如"3.2: B"）。纯字母映射到最近一个未回答的简报；如果有一个以上开放（分割链），不要猜测——询问哪个`D<N>.k`它回答。永远不要在链上模糊地应用纯字母。

**散文中的单向/破坏性确认。** 决策是单向门（不可逆或破坏性——删除、强制推送、删除、覆盖）时，散文是比工具更弱的门，所以让它更强：要求显式打字确认（确切的选项字母或单词），明确说明什么是不可逆的，永远不要基于模糊、部分或含糊的回复继续——重新询问。将沉默或"ok"/"sure"没有明确的选项视为未确认。

### Format

每个AskUserQuestion是一个决策简报，必须作为tool_use发送，而非散文——除非上面记录的失败回退适用（交互式会话 + 调用不可用/出错），此时散文回退是正确的输出。

```
D<N> — <一行问题标题>
Project/branch/task: <使用_BRANCH的1句简短接地句子>
ELI10: <16岁能看懂的纯英语，2-4句，命名风险>
Stakes if we pick wrong: <一句话：什么坏了，用户看到什么，失去了什么>
Recommendation: <choice> because <一行原因>
Completeness: A=X/10, B=Y/10   （或：Note: options differ in kind, not coverage — no completeness score）
Pros / cons:
A) <option label> (recommended)
  ✅ <优点——具体、可观察、≥40字符>
  ❌ <缺点——诚实、≥40字符>
B) <option label>
  ✅ <优点>
  ❌ <缺点>
Net: <一句话综合你实际上在权衡什么>
```

D编号：skill调用中的第一个问题是`D1`；自行递增。这是模型级指令，不是运行时计数器。

ELI10始终存在，用纯英语，不是函数名。推荐始终存在。保留`(recommended)`标签；AUTO_DECIDE依赖它。

完整性：仅在选项在覆盖上不同时使用`Completeness: N/10`。10 = 完整，7 = 快乐路径，3 = 捷径。如果选项在种类上不同，写：`Note: options differ in kind, not coverage — no completeness score。`

Pros / cons：使用✅和❌。当选择是真实的时，每个选项最少2个✅和1个❌；每项目符号最少40字符。单向/破坏性确认的硬性转义：`✅ No cons — this is a hard-stop choice`。

中立姿态：`Recommendation: <default> — this is a taste call, no strong preference either way`；`(recommended)`保持在默认选项上供AUTO_DECIDE使用。

工作量双向度量：当一个选项涉及工作量时，标注human-team和CC+gstack的时间，例如`(human: ~2 days / CC: ~15 min)`。在决策时使AI压缩可见。

Net行关闭权衡。每个skill的指令可以添加更严格的规则。

### 处理5+选项——分割，从不丢弃

AskUserQuestion每次调用最多**4个选项**。有5+真实选项时，永远不要丢弃、合并或静默推迟一个以适应。选择一个合规的形状：

- **批量化为≤4组**——用于连贯的替代方案（例如版本升级、布局变体）。一次调用，仅当前4个不适合时才出现第5个。
- **按选项分割**——用于独立的范围项（例如"ship E1..E6?"）。发出N次顺序调用，每个选项一次。不确定时默认为此。

按选项调用形状：`D<N>.k`标题（例如D3.1..D3.5），每个选项的ELI10，推荐，kind-note（无完整性分数——Include/Defer/Cut/Hold是决策动作），和4个桶：
**A) Include**，**B) Defer**，**C) Cut**，**D) Hold**（停止链，讨论）。

链之后，触发`D<N>.final`来验证组装的集（重新提示依赖冲突）并确认发布它。使用`D<N>.revise-<k>`来修订一个选项而不重新运行链。

对于N>6，首先触发`D<N>.0`元AskUserQuestion（继续/缩小/批量）。

分割链的question_ids：`<skill>-split-<option-slug>`（kebab-case ASCII，≤64字符，碰撞时`-2`/`-3`后缀）运行时检查器（`bin/gstack-question-preference`）拒绝任何`*-split-*`id的`never-ask`，因此分割链永远不会AUTO_DECIDE-eligible——用户的选项集是神圣的。

**完整规则 + 工作示例 + Hold/依赖语义：** 参见gstack仓库中的`docs/askuserquestion-split.md`。当N>4时按需阅读。

### 发出前自检

调用AskUserQuestion之前，验证：
- [ ] D<N>标题存在
- [ ] ELI10段落存在（风险行也是）
- [ ] 推荐行存在且有具体原因
- [ ] 完整性评分（覆盖）或kind-note存在（种类）
- [ ] 每个选项有≥2 ✅和≥1 ❌，每个≥40字符（或硬性转义）
- [ ] 一个选项上有`(recommended)`标签（即使中立姿态）
- [ ] 涉及工作量的选项上有双向度量标签（human / CC）
- [ ] Net行关闭决策
- [ ] 你正在调用tool，不是写散文——除非`CONDUCTOR_SESSION: true`（此时散文是DEFAULT，不是tool）或记录的失败回退适用（然后：散文配合强制三元组——问题ELI10，每个选项的完整性，推荐 + `(recommended)`——以及"用字母回复"指令，然后停止）
- [ ] 非ASCII字符（CJK / 重音）直接写，不\u转义
- [ ] 如果你有5+选项，你分割（或批量化为≤4组）——没有丢弃任何
- [ ] 如果你分割，你在触发链之前检查了选项之间的依赖
- [ ] 如果一个按选项的Hold触发，你立即停止了链（没有排队）


## Artifacts同步（skill开始）

```bash
_GSTACK_HOME="${GSTACK_HOME:-$HOME/.gstack}"
# 偏好v1.27.0.0的artifacts文件；如果用户
# 升级中途在迁移脚本运行之前则回退到brain文件。
if [ -f "$HOME/.gstack-artifacts-remote.txt" ]; then
  _BRAIN_REMOTE_FILE="$HOME/.gstack-artifacts-remote.txt"
else
  _BRAIN_REMOTE_FILE="$HOME/.gstack-brain-remote.txt"
fi
_BRAIN_SYNC_BIN="~/.claude/skills/gstack/bin/gstack-brain-sync"
_BRAIN_CONFIG_BIN="~/.claude/skills/gstack/bin/gstack-config"

# /sync-gbrain context-load：教agent在可用时使用gbrain。
# 每worktree固定：post-spike redesign使用kubectl风格的`.gbrain-source`在
# git toplevel中来限定查询范围。在worktree中查找固定（不是全局状态文件），
# 以便打开没有固定的worktree B不会声称"已索引"
# 仅因为worktree A已同步。当gbrain未配置时为空字符串（非gbrain用户的零上下文成本）。
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

# 检测remote-MCP模式（/setup-gbrain的路径4）。在remote模式下本地
# artifacts同步是no-op；brain服务器从GitHub/GitLab自己的节奏拉取。
# 直接读取claude.json以保持此preamble快速（每次skill开始时没有
# subprocess到claude CLI）。
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
  # Remote-MCP模式：本地artifacts同步是no-op（brain管理员的服务器
  # 从GitHub/GitLab拉取）。向用户展示这是设计如此，不是故障。
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

隐私停止门：如果输出显示`ARTIFACTS_SYNC: off`，`artifacts_sync_mode_prompted`为`false`，且在PATH中或`gbrain doctor --fast --json`可以工作，询问一次：

> gstack可以将你的artifact（CEO计划、设计、报告）发布到GBrain跨机器索引的私有GitHub仓库。应该同步多少？

选项：
- A) 一切允许的（推荐）
- B) 仅artifacts
- C) 拒绝，保持一切本地

回答后：
```bash
# 选择的模式: full | artifacts-only | off
"$_BRAIN_CONFIG_BIN" set artifacts_sync_mode <choice>
"$_BRAIN_CONFIG_BIN" set artifacts_sync_mode_prompted true
```

如果A/B且`~/.gstack/.git`缺失，询问是否运行`gstack-artifacts-init`。不要阻塞skill。

遥测前skill结束：

```bash
"~/.claude/skills/gstack/bin/gstack-brain-sync" --discover-new 2>/dev/null || true
"~/.claude/skills/gstack/bin/gstack-brain-sync" --once 2>/dev/null || true
```

## 模型特定行为补丁（claude）

以下微调是为claude模型家族调优的。它们**从属于**skill工作流、STOP点、AskUserQuestion门、plan模式安全性和/ship审查门。如果下面的微调与skill指令冲突，skill获胜。将这些视为偏好，而非规则。

**Todo-list纪律。** 处理多步计划时，每完成一个任务就单独标记完成。不要在最后批量完成。如果任务变得不必要，用一行原因标记跳过。

**重操作前先思考。** 对于复杂操作（重构、迁移、非简单的新功能），在执行前简要说明你的方法。这让用户可以廉价地纠正路线而非中途纠正。

**专用工具优先于Bash。** 优先使用Read、Edit、Write、Glob、Grep而非shell等价物（cat、sed、find、grep）。专用工具更便宜且更清晰。

## 语气风格

GStack voice：Garry风格的产品和工程判断，为运行时压缩。

- 以重点开头。说出它做什么、为什么重要、对开发者有什么变化。
- 具体。命名文件、函数、行号、命令、输出、评估和真实数字。
- 将技术选择与用户结果关联：真实用户看到什么、失去什么、等待什么、或现在能做什么。
- 对质量直接。Bug重要。边缘情况重要。修复整个东西，不是演示路径。
- 听起来像开发者在对开发者说话，不是顾问在给客户呈现。
- 永远不要企业化、学术化、PR或炒作。避免填充物、清嗓子、通用乐观和创始人模仿。
- 无em dash。无AI词汇：delve、crucial、robust、comprehensive、nuanced、multifaceted、furthermore、moreover、additionally、pivotal、landscape、tapestry、underscore、foster、showcase、intricate、vibrant、fundamental、significant。
- 用户有你不具备的上下文：领域知识、时机、关系、品味。跨模型一致是推荐，不是决策。用户决定。

好："auth.ts:47 returns undefined when the session cookie expires. Users hit a white screen. Fix: add a null check and redirect to /login. Two lines."
坏："I've identified a potential issue in the authentication flow that may cause problems under certain conditions."

## 上下文恢复

在session开始时或压缩后，恢复最近的项目上下文。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
_PROJ="${GSTACK_HOME:-$HOME/.gstack}/projects/${SLUG:-unknown}"
if [ -d "$_PROJ" ]; then
  echo "--- RECENT ARTIFACTS ---"
  find "$_PROJ/ceo-plans" "$_PROJ/checkpoints" -type f -name "*.md" 2>/dev/null | xargs ls -t 2>/dev/null | head -3
  [ -f "$_PROJ/${_BRANCH}-reviews.jsonl" ] && echo "REVIEWS: $(wc -l < "$_PROJ/${_BRANCH}-reviews.jsonl" | tr -d ' ') entries"
  [ -f "$_PROJ/timeline.jsonl" ] && tail -5 "$_PROJ/timeline.jsonl"
  if [ -f "$_PROJ/timeline.jsonl" ]; then
    _LAST=$(grep "\"branch\":\"${_BRANCH}\"" "$_PROJ/timeline.jsonl" 2>/dev/null | grep '"event":"completed"' | tail -1)
    [ -n "$_LAST" ] && echo "LAST_SESSION: $_LAST"
    _RECENT_SKILLS=$(grep "\"branch\":\"${_BRANCH}\"" "$_PROJ/timeline.jsonl" 2>/dev/null | grep '"event":"completed"' | tail -3 | grep -o '"skill":"[^"]*"' | sed 's/"skill":"//;s/"//' | tr '\n' ',')
    [ -n "$_RECENT_SKILLS" ] && echo "RECENT_PATTERN: $_RECENT_SKILLS"
  fi
  _LATEST_CP=$(find "$_PROJ/checkpoints" -name "*.md" -type f 2>/dev/null | xargs ls -t 2>/dev/null | head -1)
  [ -n "$_LATEST_CP" ] && echo "LATEST_CHECKPOINT: $_LATEST_CP"
  if [ -f "$_PROJ/decisions.active.json" ]; then
    echo "--- ACTIVE DECISIONS (recent, scope-relevant) ---"
    ~/.claude/skills/gstack/bin/gstack-decision-search --recent 5 2>/dev/null
    echo "--- END DECISIONS ---"
  fi
  echo "--- END ARTIFACTS ---"
fi
```

如果列出了artifact，读取最新有用的一个。如果出现`LAST_SESSION`或`LATEST_CHECKPOINT`，给一个2句欢迎回来摘要。如果`RECENT_PATTERN`清楚暗示下一个skill，建议一次。

**跨session决策。** 如果列出了`ACTIVE DECISIONS`，将它们视为先前已解决的决定及其理由——不要默默地重新裁决它们；如果你要反转一个，明确地说。每当问题触及过去决定（"我们决定了什么/为什么/我们尝试过什么"）时使用`~/.claude/skills/gstack/bin/gstack-decision-search`。当你或用户做出**持久决策**（架构、范围、工具/供应商选择或反转）——不是trivial的选择——使用`~/.claude/skills/gstack/bin/gstack-decision-log`记录（`--supersede <id>`用于反转）。可靠且本地；不需要gbrain。

## 写作风格（如果preamble回显了`EXPLAIN_LEVEL: terse`则完全跳过，或用户当前消息明确要求terse/无解释输出）

应用于AskUserQuestion、用户回复和发现。AskUserQuestion Format是结构；这是散文质量。

- 每次skill调用时使用术语表术语，即使用户粘贴了它。
- 用结果框架问题：避免什么痛苦、解锁什么能力、什么用户体验变化。
- 使用短句、具体名词、主动语态。
- 用用户影响结束决策：用户看到什么、等待什么、失去什么、获得什么。
- 用户回合覆盖获胜：如果当前消息要求terse/无解释/只给答案，跳过此部分。
- Terse模式（EXPLAIN_LEVEL: terse）：无术语表、无结果框架层、较短的响应。

术语表列表位于`~/.claude/skills/gstack/scripts/jargon-list.json`（80+术语）。在本session遇到的第一个术语表中术语时，Read该文件一次；将`terms`数组视为规范列表。列表是repo拥有的，在版本之间可能增长。

## 完整性原则 — 沸腾海洋

AI使完整性变得便宜，所以完整的事情是目标。推荐完整覆盖（测试、边缘情况、错误路径）——一次烧开一个海洋中唯一一件超出范围的事情是真正无关的工作（重写、多季度迁移）；将其标记为单独的范围，永远不是捷径的借口。

当选项在覆盖上不同时，包含`Completeness: X/10`（10 = 所有边缘情况，7 = 快乐路径，3 = 捷径）。当选项在种类上不同时，写：`Note: options differ in kind, not coverage — no completeness score。` 不要编造分数。

## 困惑协议

对于高风险歧义（架构、数据模型、破坏性范围、缺失上下文），停止。用一句话命名，呈现2-3个选择及权衡，并问。不要用于例行编码或明显更改。

## Continuous Checkpoint Mode

如果`CHECKPOINT_MODE`为`"continuous"`：用`WIP:`前缀自动-commit完成的逻辑单元。

在新的有意文件、完成的函数/模块、已验证的错误修复之后，以及在长时间运行的install/build/test命令之前commit。

Commit格式：

```
WIP: <什么更改的简洁描述>

[gstack-context]
Decisions: <此步骤做出的关键选择>
Remaining: <逻辑单元中剩下什么>
Tried: <值得记录的失败方法>（无则省略）
Skill: </skill-name-if-running>
[/gstack-context]
```

规则：仅暂存有意文件，永远不要`git add -A`，不要commit损坏的测试或mid-edit状态，仅当`CHECKPOINT_PUSH`为`"true"`时推送。不要宣布每个WIP commit。

`/context-restore`读取`[gstack-context]`；`/ship`将WIP commit压缩为干净的commit。

如果`CHECKPOINT_MODE`为`"explicit"`：忽略此部分，除非skill或用户要求commit。

## Context Health（软指导）

在长时间运行的skill会话期间，定期写一个简短的`[PROGRESS]`摘要：完成了什么、下一步、意外。

如果你在相同的诊断、相同文件或失败的修复变体上循环，停止并重新考虑。考虑升级或/context-save。进度摘要绝不能改变git状态。

## 问题调整（如果`QUESTION_TUNING: false`则完全跳过）

在每次AskUserQuestion之前，从`scripts/question-registry.ts`或`{skill}-{slug}`选择`question_id`，然后运行`~/.claude/skills/gstack/bin/gstack-question-preference --check "<id>"`。`AUTO_DECIDE`意味着选择推荐选项并说"Auto-decided [summary] → [option] (your preference). Change with /plan-tune." `ASK_NORMALLY`意味着问。

**将question_id作为标记嵌入问题文本** 以便hook可以确定性地识别它（plan-tune cathedral T14 / D18 progressive markers）。在渲染的问题中附加`<gstack-qid:{question_id}>`（前导行或尾行都可以；当包裹在HTML风格的角括号中时标记不会向用户可见地渲染，但hook去掉它）。没有标记，PreToolUse enforcement hook将AUQ视为仅观察，从不自动决定——所以当问题匹配注册的`question_id`时总是包含它。

**通过在恰好一个选项上的`(recommended)`标签后缀嵌入选项推荐**。PreToolUse hook首先解析`(recommended)`，回退到"Recommendation: X"散文，且如果模棱两可则拒绝自动决定。两个`(recommended)`标签 = 拒绝。

回答后，记录尽力而为（PostToolUse hook也确定性地捕获当安装时；(source, tool_use_id)上的dedup处理双重写入）：
```bash
~/.claude/skills/gstack/bin/gstack-question-preference --write '{"question_id":"<id>","preference":"<pref>","source":"inline-user","free_text":"<optional original words>"}'
退出码2 = 拒绝，非originated；不要重试。成功后："Set `<id>` → `<preference>`. Active immediately."

## 仓库所有权 — 看到什么，说什么

`REPO_MODE`控制在你的分支之外如何处理问题：
- **`solo`** — 你拥有一切。调查并主动提供修复。
- **`collaborative`** / **`unknown`** — 通过AskUserQuestion标记，不要修复（可能是别人的）。

总是标记任何看起来错误的东西——一句话，你注意到了什么及其影响。

## 构建前先搜索

在构建任何不熟悉的东西之前，**先搜索。** 参见`~/.claude/skills/gstack/ETHOS.md`。
- **Layer 1**（久经考验）——不要重新发明。**Layer 2**（新且流行）——审查。**Layer 3**（第一原则）——最珍贵。

**Eureka：** 当第一原则推理与传统智慧矛盾时，命名并记录：
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

## 完成状态协议

完成skill工作流时，使用以下之一报告状态：
- **DONE** — 有证据完成。
- **DONE_WITH_CONCERNS** — 完成，但列出关切。
- **BLOCKED** — 无法进行；说明阻塞者及尝试了什么。
- **NEEDS_CONTEXT** — 缺少信息；准确说明需要什么。

在3次失败尝试后、不安全的敏感更改时或你无法验证的范围内升级。格式：`STATUS`、`REASON`、`ATTEMPTED`、`RECOMMENDATION`。

## 运营自我改进

完成之前，如果你发现一个持久的项目怪癖或命令修复，可以节省下次5+分钟，记录它：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"SKILL_NAME","type":"operational","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"observed"}'
```

不要记录明显的事实或一次性瞬时错误。

## Telemetry（最后运行）

工作流完成后，记录遥测。使用frontmatter中的`name:`。OUTCOME是success/error/abort/unknown。

**PLAN MODE异常 — 始终运行：** 此命令将遥测写入
`~/.gstack/analytics/`，匹配preamble analytics写入。

运行此bash：

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
# Session时间线：记录skill完成（本地-only，永远不发送到任何地方）
~/.claude/skills/gstack/bin/gstack-timeline-log '{"skill":"SKILL_NAME","event":"completed","branch":"'$(git branch --show-current 2>/dev/null || echo unknown)'","outcome":"OUTCOME","duration_s":"'"$_TEL_DUR"'","session":"'"$_SESSION_ID"'"}' 2>/dev/null || true
# Local analytics（按遥测设置门控）
if [ "$_TEL" != "off" ]; then
echo '{"skill":"SKILL_NAME","duration_s":"'"$_TEL_DUR"'","outcome":"OUTCOME","browse":"USED_BROWSE","session":"'"$_SESSION_ID"'","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
fi
# Remote telemetry（opt-in，需要二进制）
if [ "$_TEL" != "off" ] && [ -x ~/.claude/skills/gstack/bin/gstack-telemetry-log ]; then
  ~/.claude/skills/gstack/bin/gstack-telemetry-log \
    --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
    --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
fi
```

运行前替换`SKILL_NAME`、`OUTCOME`和`USED_BROWSE`。

## 计划状态页脚

运行计划审查的skill（`/plan-*-review`、`/codex review`）在skill末尾包含EXIT PLAN模式GATE阻塞清单，它在调用ExitPlanMode之前验证计划文件以`## GSTACK REVIEW REPORT`结尾。不运行计划审查的skill（如`/ship`、`/qa`、`/review`等操作skill）通常不在plan模式中运行，并且没有审查报告要验证；此footer对它们无操作。写入计划文件是plan模式中唯一允许的编辑。

# 真实设备iOS QA

此技能通过USB驱动真实的iPhone。Agent读取你的Swift源码，
生成typed state accessor，部署debug bridge，并运行封闭的
find→fix→verify循环。无simulator、无XCTest、无WebDriverAgent。

## 架构

```
       ┌──────────────────────┐   USB CoreDevice (IPv6)   ┌──────────────────┐
       │ gstack-ios-qa daemon │ ────────────────────────▶ │ iOS app          │
       │ (Mac, bun/TS)        │   bearer + X-Session-Id   │ StateServer      │
       │                      │                           │ (loopback only)  │
       │ - boot token rotate  │                           │ - /tap /swipe    │
       │ - session minting    │                           │ - /type /state   │
       │ - audit + redact     │                           │ - /snapshot      │
       └──────────────────────┘                           └──────────────────┘
                ▲
                │ Tailscale (可选, --tailnet)
                │
       ┌──────────────────────┐
       │ Remote agent         │
       │ (OpenClaw, etc.)     │
       └──────────────────────┘
```

iOS应用的`StateServer`仅绑定loopback（`::1` + `127.0.0.1`）。Tailnet
入口完全是Mac daemon的工作。daemon验证Tailscale
身份通过本地`tailscaled`socket并为远程agent生成短生命期的session
token（默认1h）。

## 前提条件

- macOS（daemon使用Xcode的`devicectl`）。
- iPhone通过USB连接，已配对和信任。
- 已安装Xcode + Swift工具链（`swift --version`报告>= 5.9）。
- 磁盘上可用App源码，至少有一个`@Observable`类。
- 对于remote-control模式：已安装Tailscale且用户已登录。

## 阶段0：会话热启动（可选）

如果`~/.gstack/ios-qa-session.json`存在且设备仍连接，
跳过Phase 1-2并跳到Phase 3。session缓存保存旋转的token、
UDID、隧道地址和accessor hash。以下情况使缓存失效：

- 用户传递`--cold`强制完全引导。
- 第一次state查询时检测到accessor hash不匹配。
- daemon报告的缓存UDID不再连接。

```bash
SESSION="$HOME/.gstack/ios-qa-session.json"
if [ -f "$SESSION" ] && [ "$COLD" != "1" ]; then
  CACHED_UDID=$(python3 -c "import json,os; d=json.load(open(os.path.expanduser('$SESSION'))); print(d['udid'])")
  CACHED_PORT=$(python3 -c "import json,os; d=json.load(open(os.path.expanduser('$SESSION'))); print(d['daemon_port'])")
  if curl -sf "http://127.0.0.1:$CACHED_PORT/healthz" > /dev/null; then
    echo "Warm start: daemon alive, device $CACHED_UDID connected"
  fi
fi
```

## 阶段1：读取源码，规划代码生成

1. 遍历app源码（作为`--source <dir>`传递）并识别所有`@Observable`
   类。注意任何标记了`@Snapshotable`包装器的属性——这些
   是可快照的字段。
2. 运行`swift run --package-path $GSTACK_HOME/ios-qa/scripts/gen-accessors-tool gen-accessors --input <source-dir>`。
   首次调用构建swift-syntax依赖树（冷启动：2-5分钟）。
   后续运行被content-hash缓存，约~50ms完成。
3. 向用户显示accessor列表并询问是否要安装DebugBridge
   SPM依赖到他们的`Package.swift`（一次AskUserQuestion）。

## Phase 2: 引导设备桥

1. 将`DebugBridge` SPM依赖添加到app的`Package.swift`。包
   提供三个仅限Debug配置的库产品：
   - `DebugBridgeCore`（Swift，跨平台）— StateServer + 桥协议。
   - `DebugBridgeTouch`（Objective-C，仅iOS）— KIF衍生的进程内触摸
     合成，支持iOS 18+的`_UIHitTestContext` SwiftUI hit-testing。
   - `DebugBridgeUI`（Swift，仅iOS）— 截图 / 元素 / 变体
     桥实现。
   app目标通过`.when(configuration: .debug)`依赖`DebugBridgeUI`
   （传递拉入Core + Touch）。Release构建拒绝链接这些
   目标。
2. 从`@main` App初始化桥接，在`#if DEBUG`门控下：
   ```swift
   #if DEBUG
   import DebugBridgeCore
   StateServer.shared.start()
   #if canImport(UIKit)
   import DebugBridgeUI
   DebugBridgeUIWiring.installAll()
   #endif
   #endif
   ```
3. 使用`xcodebuild -scheme <SchemeName>
   -destination 'platform=iOS,id=<UDID>' build install`构建并部署到设备。
4. 通过`devicectl device process launch --device <UDID> --console <bundle-id>`启动。
   捕获首次运行时打印到`os_log`的boot token。
5. 生成Mac端daemon（按需）——`gstack-ios-qa-daemon`。Daemon
   在`~/.gstack/ios-qa-daemon.pid`上获得独占flock。如果另一个
   daemon存活，第二次调用发现其端口并连接。
6. Daemon立即在iOS StateServer上调用`POST /auth/rotate`
   使用一个新鲜的仅内存token。boot token在大约5s后变得无用。
   任何在此之后抓取`os_log`的东西看到的都是死凭证。

## Phase 3: 视觉驱动的agent循环

每次迭代：

1. `GET /screenshot`（通过daemon）→ 保存PNG。
2. `GET /elements` → 可访问性树。
3. `GET /state/snapshot`（仅`@Snapshotable`字段）→ 当前state。
4. 根据屏幕上的内容与测试目标决定下一个操作。
5. `POST /session/acquire`获取设备锁。
6. 执行`POST /tap`、`/swipe`、`/type`，或`POST /state/<key>`写入。
7. 重新截图；比较；如果发现bug记录发现。
8. 迭代完成后`POST /session/release`。

如果remote模式激活，通过tailnet监听器的每个认证的变更请求写入一行审计记录到
`~/.gstack/security/ios-qa-audit.jsonl`。

## 模式

**Local-USB模式（默认）。** Daemon仅绑定loopback；不需要
Tailscale。生成的skill获得完整surface访问。最适合个人
开发。

**Tailnet模式（`--tailnet`）。** Daemon额外绑定Tailscale接口
（从不`0.0.0.0`）。需要`tailscaled`在本地运行且daemon能够
读取`/var/run/tailscale.sock`。如果socket缺失、权限拒绝或返回不可解析的WhoIs响应则关闭失败。远程agent通过tailnet访问`POST /auth/mint`，daemon通过WhoIs规范化身份，检查allowlist文件，生成session token。参见`ios-qa/docs/tailscale-acl-example.md`。

**能力层级（tailnet模式）。** 生成的token默认为`interact`
（点击、滑动、输入）。更高级别需要显式owner生成：

- **observe：** `/screenshot`、`/elements`、`GET /state/*`、`/healthz`、
  `/session/heartbeat`。
- **interact：** observe + `/tap`、`/swipe`、`/type`。
- **mutate：** interact + `POST /state/<key>`。
- **restore：** mutate + `POST /state/restore`。

Owner通过`gstack-ios-qa-mint --remote <identity> --capability <tier>`在Mac上生成。Tailnet上的self-service mint仅对已列入allowlist的身份成功。

**录制模式（`--recording`）。** DebugOverlay在角落渲染一个小对角线"AGENT DEMO"水印，使屏幕录像清楚显示设备是agent驱动的。

## Demo模式

如果用户说"demo"、"demo mode"、"show me"或"I want to see it
working"，以**DEMO MODE**运行。这改变了agent与app交互的方式：

**演示模式覆盖所有其他规则。** 当demo模式激活时，
agent必须通过可见UI（`/tap`、`/swipe`、`/type`）驱动每个操作
且永远不要使用`POST /state/*`写入来跳过步骤。观看者看到agent
输入每个键，点击每个按钮。设备上DebugOverlay归属芯片显示"Driven by Claude Code (demo)"或远程agent身份。

在demo模式下，截图速率提升到4fps，使录制感觉
实时。

## 故障模式与恢复

| Symptom | Likely cause | Action |
|---|---|---|
| `curl: connection refused` to daemon | daemon崩溃 | 重新运行`/ios-qa`；spawn-race锁将关闭失败 |
| `403 identity_not_allowed` from `/auth/mint` | identity缺失于allowlist | 在Mac上运行`gstack-ios-qa-mint --remote <identity>` |
| `409 schema_mismatch` on `/state/restore` | 来自旧app构建的快照 | 丢弃快照；重新捕获 |
| `503 device_disconnected` from proxy | USB隧道断开 | 重新连接设备；daemon在30s内自动重连 |
| `429 rate_limited` from `/auth/mint` | 同一identity每分钟>10次mint | 等待60s；检查audit日志中的异常 |
| `413 body_too_large` on `/state/restore` | 快照>1MB | 增加`--max-body`或精简快照 |

## 清理

使用`/ios-clean`移除DebugBridge SPM依赖和所有`#if DEBUG`接线在Release构建之前。这是一个便利流程；结构性Release构建保护（Package.swift `.when(configuration: .debug)` + CI `swift build -c release`检查）才是安全关键路径。
