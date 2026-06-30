# Plan Tuning v0 — 设计文档

**状态：** 已批准 v1 实施
**分支：** garrytan/plan-tune-skill
**作者：** Garry Tan（用户），由 Claude Opus 4.7 + OpenAI Codex gpt-5.4 协助评审
**日期：** 2026-04-16

## 本文档是什么

`/plan-tune` v1 的权威记录，包括它是什么、不是什么、我们考虑了什么以及为何做出每个选择。提交到仓库，以便未来的贡献者（以及未来的 Garry）可以在不进行考古的情况下追溯推理。取代两个 `~/.gstack/projects/` 工件（office-hours 设计文档 + CEO 计划），它们是按用户本地记录。

## 功能，一句话总结

gstack 的 40+ 技能不断触发 AskUserQuestion。高级用户会反复以相同方式回答相同问题，却无法告诉 gstack"别再问我这个"。更根本的是，gstack 没有每个用户如何引导工作的模型 —— 范围偏好、风险容忍度、细节偏好、自主性、架构关注 —— 所以每个技能默认值都对所有人折中。`/plan-tune` v1 构建模式 + 观察层：一个类型化问题注册表、每问题的显式偏好、"tune:"行内反馈，以及一个可通过纯英语查看的档案（声明 + 推断维度）。它尚不会基于档案调整技能行为。那在 v2 中，在 v1 证明基质工作正常后。

## 为什么构建较小版本

该功能最初是作为一个完整的自适应基质启动的：心理图谱维度驱动自动决策、盲点指导、LANDED 庆祝 HTML 页面，全部捆绑在一起。四轮审查（office-hours、CEO EXPANSION、DX POLISH、工程审查）通过。然后外部声音（Codex）提出了 20 点批评。按优先级顺序的关键发现：

1. **"基质"是虚假的。** 计划将 5 个技能连接到在序言中读取 profile，但 AskUserQuestion 是一个提示约定，而非中间件。代理可以默默跳过指令。在一个不可强制执行的约定之上构建自动决策是不可靠的。如果没有一个每个 AskUserQuestion 都路由通过的类型化问题注册表，基质声明就是营销。
2. **内部逻辑矛盾。** E4（盲点）+ E6（不匹配）+ 声明维度的 ±0.2 限幅不组合。如果用户自报是限幅的基准事实，E6 的不匹配检测就是在检测噪声。如果行为可以纠正档案，限幅就会抑制 E6 需要的信号。
3. **档案投毒。** 行内"tune: never ask"可能被恶意仓库内容（README、PR 描述、工具输出）发出，代理会忠实地写入它。之前的审查没有发现这个安全缺口。
4. **E5 LANDED 页面在序言中。** `gh pr view` + HTML 写入 + 浏览器打开在每个技能的序言中执行，这是延迟、身份验证失败、速率限制、意外浏览器打开以及注入最热路径的不可预测性。
5. **实施顺序是反向的。** 计划从分类器和分箱开始。正确顺序：先构建集成点（类型化问题注册表），然后基础设施，然后消费者。

在权衡了 Codex 的论点后，我们选择回滚 CEO EXPANSION 并发布以真正类型化注册表为基础的可观察的 v1。心理图谱只有在注册表在生产中证明耐用后才变为行为数据。

## v1 范围（我们现在构建的）

1. **类型化问题注册表**（`scripts/question-registry.ts`）。gstack 使用的每个 AskUserQuestion 都声明为 `{id, skill, category, door_type, options[], signal_key?}`。模式治理。
2. **CI 强制。** 扫描测试（门层级）断言 SKILL.md.tmpl 文件中的每个 AskUserQuestion 模式都有匹配的注册表条目。在漂移、重名或重复时 CI 失败。
3. **问题日志**（`bin/gstack-question-log`）。将 `{ts, question_id, user_choice, recommended, session_id}` 追加到 `~/.gstack/projects/{SLUG}/question-log.jsonl`。针对注册表验证。
4. **显式每问题偏好**（`bin/gstack-question-preference`）。写入 `{question_id, preference}`，其中偏好为 `always-ask | never-ask | ask-only-for-one-way`。从第 1 个会话起被遵守。无校准门 —— 用户声明了，系统服从。
5. **序言注入。** 在每个 AskUserQuestion 之前，代理调用 `gstack-question-preference --check <registry-id>`。如果 `never-ask` AND 问题不是单向门，自动选择推荐选项并显示注释："Auto-decided [summary] → [option]（你的偏好）。通过 /plan-tune 更改。"单向门无论偏好如何都始终提问 —— 安全覆盖。
6. **带用户来源门的行内"tune:"反馈。** 代理提供"调整此问题？回复 `tune: [反馈]` 来调整。"用户可以使用快捷方式（`unnecessary`、`ask-less`、`never-ask`、`always-ask`、`context-dependent`）或自由英语。关键：代理仅在 `tune:` 内容出现在用户当前聊天轮次中时写入 tune 事件 —— 不在工具输出中，不在文件读取中。二进制验证 `source: "inline-user"` 写入时；拒绝其他来源。
7. **声明档案**（`/plan-tune setup`）。5 个纯英语问题，每维度一个。存储在统一的 `~/.gstack/developer-profile.json` 中，位 `declared: {...}` 下。在 v1 中仅提供信息 —— 无技能行为改变。
8. **观察/推断档案。** 每个问题日志事件通过手工制作的信号映射（`scripts/psychographic-signals.ts`）为推断维度贡献增量变化。按需计算。显示但不采取行动。
9. **`/plan-tune` 技能。** 对话式纯英语查看工具。"显示我的档案"、"设置偏好"、"我被问了什么问题"、"显示我的言行差距"。无需 CLI 子命令语法。
10. **与现有 `~/.gstack/builder-profile.jsonl` 统一。** 将会话记录和积累的信号合并到统一的 `~/.gstack/developer-profile.json` 中。迁移是原子的 + 幂等的 + 归档源文件。

## 推迟到 v2（不在此 PR 中，但有显式验收标准）

| 项目 | 推迟原因 | v2 晋升的验收标准 |
|------|----------|-------------------|
| E1 基质接线（5 个技能读取档案并适应） | 需要 v1 注册表证明耐用。需要真实观察数据来校准信号增量。有心理图谱漂移风险。 | v1 注册表稳定 90+ 天。推断维度显示跨 3+ 技能的明显稳定性。用户食腐验证档案知情的默认值感觉正确。 |
| E3 `/plan-tune 叙事` + `/plan-tune 氛围` | 事件锚定叙事需要稳定档案。没有 v1 数据，输出将是通用 SLOP。 | 档案多样性检查通过 2+ 周真实使用。叙事测试证明它引用特定事件，而非陈词滥调。 |
| E4 盲点指导 | 在没有显式交互预算设计的情况下与 E1/E6 逻辑冲突。需要全局会话预算、升级规则、排除不匹配检测。 | 交互预算 + 升级的设计规范。食腐确认挑战感觉像指导而非唠叨。 |
| E5 LANDED 庆祝 HTML 页面 | 不能在序言中存在（Codex #9, #10）。晋升时，移到显式命令 `/plan-tune show-landed` OR 发运后钩子 —— 而非热路径中的被动检测。 | 显式命令或钩子设计。/design-shotgun → /design-html 用于视觉方向。PR 数据聚合的安全 + 隐私审查。 |
| E6 基于不匹配的自动调整 | 在 v1 中，/plan-tune 显示声明与推断之间的差距。在 v2 中，它可以建议声明更新。需要双轨档案稳定。 | 来自 v1 的真实不匹配数据显示一致模式。建议 UX 单独设计。 |
| 心理图谱驱动的自动决策 | 在 v1 中零行为改变。只有显式偏好运行。 | 真实使用显示显式偏好涵盖大多数情况。推断档案足够稳定可以信任。 |

## 完全拒绝（Codex 是对的，我们不做这些）

| 项目 | 拒绝原因 |
|------|----------|
| 作为提示约定的基质（vs 类型化注册表） | Codex #1。代理可以默默跳过指令。在之上构建心理图谱是沙子。 |
| 声明维度的 ±0.2 限幅 | Codex #6。与 E6 不匹配检测产生逻辑矛盾。二选一：可编辑偏好 OR 推断行为。现在：两者独立跟踪（双轨档案）。 |
| 通过解析散文摘要对单向门分类 | Codex #4。安全性取决于措辞。door_type 必须在问题定义站点（注册表）声明，而非推断。 |
| 混合声明 + 覆盖 + 裁决 + 反馈的单一事件模式文件 | Codex #5。不兼容的领域对象。现在拆分为三个文件：question-log.jsonl、question-preferences.json、question-events.jsonl。 |
| /plan-tune 入职的 TTHW 遥测 | Codex #14。与本地优先框架矛盾。仅本地日志记录。 |
| 无用户来源验证的写入行内 tune: | Codex #16。档案投毒攻击。现在：用户来源门不可选。 |

## 架构

```
~/.gstack/
  developer-profile.json            # 统一的：声明 + 推断 + 会话（来自 office-hours）

~/.gstack/projects/{SLUG}/
  question-log.jsonl                # 每个 AskUserQuestion，仅追加，注册表验证
  question-preferences.json         # 显式每问题用户选择
  question-events.jsonl             # tune: 反馈事件，用户来源门控
```

**统一档案模式**（取代 v0.16.2.0 builder-profile.jsonl 和提议的 developer-profile.json）：

```json
{
  "identity": {"email": "..."},
  "declared": {
    "scope_appetite": 0.9,
    "risk_tolerance": 0.7,
    "detail_preference": 0.4,
    "autonomy": 0.5,
    "architecture_care": 0.7
  },
  "inferred": {
    "values": {"scope_appetite": 0.72, "risk_tolerance": 0.58, "...": "..."},
    "sample_size": 47,
    "diversity": {
      "skills_covered": 5,
      "question_ids_covered": 14,
      "days_span": 23
    }
  },
  "gap": {"scope_appetite": 0.18, "...": "..."},
  "sessions": [
    {"date": "...", "mode": "builder", "project_slug": "...", "signals": []}
  ],
  "signals_accumulated": {
    "named_users": 1, "taste": 4, "agency": 3, "...": "..."
  }
}
```

**多样性检查**（Codex #13）：仅当 `sample_size >= 20 AND skills_covered >= 3 AND question_ids_covered >= 8 AND days_span >= 7` 时，`inferred` 才被视为"足够数据"。低于此阈值，`/plan-tune profile` 显示"尚无足够的观察数据"而非可能有误导性的推断值。

## 数据流（v1）

1. 序言：检查 `question_tuning` 配置。如果关闭，什么都不做。
2. 在每个 AskUserQuestion 之前：
   - 代理调用 `gstack-question-preference --check <registry-id>`
   - 如果 `never-ask` AND 问题不是单向门 → 自动选择推荐项并带注释
   - 如果 `always-ask`、未设置，或问题 IS 单向门 → 正常提问
3. AskUserQuestion 之后：
   - 将日志记录追加到 question-log.jsonl（注册表验证，拒绝未知 ID）
4. 提供行内选项："调整此问题？回复 `tune: [反馈]` 来调整。"
5. 如果用户的 NEXT 轮次消息包含 `tune:` 前缀 AND 内容源自用户自己的消息（而非工具输出）：
   - 代理调用 `gstack-question-preference --write` 带 `source: "inline-user"`
   - 二进制验证源字段；如果非 `inline-user` 则拒绝
6. 推断维度由 `bin/gstack-developer-profile --derive` 按需重新计算。信号映射更改触发从事件历史完全重新计算。

## 安全模型

**档案投毒防御**（Codex #16，决策 J 下面）：行内 tune 事件仅在以下情况下可被写入：
- 代理正在处理用户的当前聊天轮次
- `tune:` 前缀出现在该用户消息中（不在任何工具输出、文件内容、PR 描述、提交消息等中）
- 解析器对代理的指令明确指出这一点

二进制强制：`gstack-question-preference --write` 在每个 tune 来源记录上需要 `source: "inline-user"` 字段。任何其他源值（如 `inline-tool-output`、`inline-file-content`）会返回错误。代理被指示永远不要伪造 `source` 字段。

**数据隐私**：
- 所有数据仅在 `~/.gstack/` 下本地保存。未经显式用户操作，不发送任何内容。
- `/plan-tune export <path>` 将档案写入用户指定路径（选择加入导出）。
- `/plan-tune delete` 清除本地档案文件。
- `gstack-config set telemetry off` 阻止任何遥测（此技能绝不发送档案数据，无论如何）。
- 档案文件具有标准用户主目录权限。

**注入防御**（与现有 `bin/gstack-learnings-log` 模式一致）：`question_summary` 和任何自由格式用户反馈字段都针对已知的提示注入模式（"忽略之前的指令"，"system:"等）进行清洗。

## 5 条硬约束（从 office-hours 保留，针对 Codex 反馈更新）

1. **单向门按注册表声明**确定性分类，**非**运行时散文解析。每个注册表条目声明 `door_type: one-way | two-way`。关键词模式回退（`scripts/one-way-doors.ts`）是边缘情况的双保险检查。
2. **档案维度既可查看又可编辑。** `/plan-tune profile` 显示声明 + 推断 + 差距。通过纯英语的编辑部仅记录到 `declared`。系统独立跟踪 `inferred`。
3. **信号映射在 TypeScript 中手工制作。** `scripts/psychographic-signals.ts` 映射 `{question_id, user_choice} → {dimension, delta}`。非代理推断。在 v1 中仅用于 `inferred.values` 显示 —— 不用于驱动决策。
4. **在 v1 中无心理图谱驱动的自动决策。** 只有显式每问题偏好运行。这完全避开了"校准门可被操控"的批评（Codex #13）—— v1 没有门要通过。
5. **每项目偏好胜过全局偏好。** `~/.gstack/projects/{SLUG}/question-preferences.json` 覆盖任何未来全局偏好文件。全局档案（`~/.gstack/developer-profile.json`）是跨项目多样性的起点。

## 为什么事件驱动 + 双轨

**推断档案为什么事件驱动**：
- 信号映射可在 gstack 版本之间变化。从事件重新计算，无需数据迁移。
- 可审计：`/plan-tune profile --trace autonomy` 显示每个贡献值的事件。
- 面向未来：新维度可从现有历史中派生。

**为什么双轨（声明 + 推断，独立）**（决策 B 下面）：
- 解决 Codex #6 发现的逻辑矛盾。
- `declared` 是用户自主权。用户声明他们是什么。系统对用户驱动的任何内容服从（偏好、声明、覆盖）。
- `inferred` 是观察。系统跟踪行为模式。在 v1 中显示但不采取行动。
- `gap` 是有趣的信号。大差距表明用户的自我描述与他们的行为不匹配 —— 有价值的自我洞察，但不自动纠正。

## 交互模型 —— 处处纯英语

（来自 /plan-devex-review，CLI 语法上的用户纠正）：

`/plan-tune`（无参数）进入对话模式。无需 CLI 子命令语法。

纯语言菜单：
- "展示我的档案"
- "查看我被问过的问题"
- "就某个问题设置偏好"
- "更新我的档案 —— 我改变了主意"
- "展示我的言行差距"
- "关闭"

用户对话式回复。代理解读，确认预期更改，然后写入。例如：
- 用户："我更像是一个把海洋烧开的人，而非 0.5 暗示的那样"
- 代理："明白 —— 将 `declared.scope_appetite` 从 0.5 更新到 0.8？[Y/n]"
- 用户："是"
- 代理写入更新

从自由格式输入对 `declared` 进行任何变异都需要确认步骤（Codex #15 信任边界）。

高级用户可以输入快捷方式（`narrative`、`vibe`、`reset`、`stats`、`enable`、`disable`、`diff`）。两者都不需要。两者都有效。

## 需要创建的文件

### 核心模式
- `scripts/question-registry.ts` —— 类型化注册表。从审核所有 SKILL.md.tmpl AskUserQuestion 调用播种。
- `scripts/one-way-doors.ts` —— 辅助关键词回退。主要：注册表中的 `door_type`。
- `scripts/psychographic-signals.ts` —— 推断计算的手工制作信号映射。

### 二进制
- `bin/gstack-question-log` —— 追加日志记录，针对注册表验证。
- `bin/gstack-question-preference` —— 读/写/检查/清除显式偏好。
- `bin/gstack-developer-profile` —— 取代 `bin/gstack-builder-profile`。子命令：`--read`（遗产兼容）、`--derive`、`--gap`、`--profile`。

### 解析器
- `scripts/resolvers/question-tuning.ts` —— 三个生成器：`generateQuestionPreferenceCheck(ctx)`（问题前检查）、`generateQuestionLog(ctx)`（问题后日志）、`generateInlineTuneFeedback(ctx)`（问题后 tune: 提示带用户来源门指令）。

### 技能
- `plan-tune/SKILL.md.tmpl` —— 对话式纯英语查看和偏好工具。

### 测试
- `test/plan-tune.test.ts` —— 注册表完整性、重复 ID 检查、偏好优先级（never-ask + not-one-way → AUTO_DECIDE；never-ask + one-way → ASK_NORMALLY）、用户来源门（拒绝非 inline-user 来源）+ 重新计算、统一档案模式、带 7 会话固定装置的迁移回归。

## 需要修改的文件

- `scripts/resolvers/index.ts` —— 注册 3 个新解析器。
- `scripts/resolvers/preamble.ts` —— `_QUESTION_TUNING` 配置读取；为 tier >= 2 注入 3 个解析器。
- `bin/gstack-builder-profile` —— 遗产 shim 委托给 `bin/gstack-developer-profile --read`。
- 迁移脚本 —— 将现有 builder-profile.jsonl 合并到统一的 developer-profile.json 中。原子、幂等、归档源为 `.migrated-YYYY-MM-DD`。

## v1 中不涉及

显式未更改 —— 无 `{{PROFILE_ADAPTATION}}` 占位符，无基于档案的行为更改：

- `ship/SKILL.md.tmpl`、`review/SKILL.md.tmpl`、`office-hours/SKILL.md.tmpl`、`plan-ceo-review/SKILL.md.tmpl`、`plan-eng-review/SKILL.md.tmpl`

这些技能仅获得序言注入用于日志/偏好检查/tune 反馈。无档案驱动的默认值。v2 工作。

## 决策日志（含每项利弊）

### 决策 A：捆绑全部三个（问题日志 + 灵敏度 + 心理图谱） vs. 发布较小的楔形 —— 初始答案：捆绑；修订：注册表优先观察性

初始用户立场（office-hours）："心理图谱 IS 差异化。发布整个东西，以便反馈循环能够真正调整行为。"这驱动了 CEO EXPANSION。

**捆绑的优点：** 雄心。学习层使其不仅仅是一个配置。没有心理图谱，它就是一个花哨的设置菜单。

**捆绑的缺点（由 Codex 发现）：** 基质不存在。基于提示约定的心理图谱是沙子。E1/E4/E6 组合不连贯。档案投毒未解决。E5 在序言中是一个隐藏的热路径副作用。实施顺序围绕一个不可强制执行的约定构建机制。

**修订答案：** 注册表优先观察性 v1（本文档）。保留雄心作为 v2 目标，附带显式验收标准。发布一个可防御的基础。用户在看到 Codex 的 20 点批评后接受了这一点。

### 决策 B：事件驱动 vs. 存储维度 vs. 混合 —— 答案：事件驱动 + 用户声明锚点（B+C）

**方法 A（存储维度）：** 就地变异。简单。
- 优点：最小数据模型。易于推理。
- 缺点：有损。无历史。信号映射更改需要迁移。对用户档案更改不透明。

**方法 B（事件驱动）：** 存储原始事件，派生维度。
- 优点：可审计。信号映射更改时可重新计算。永远无数据迁移。匹配现有 learnings.jsonl 模式。
- 缺点：更复杂的推导。事件文件随时间增长（压缩推迟到 v2）。

**方法 C（混合 —— 用户声明锚点，事件细化）：** 初始档案由用户提供；事件在 ±0.2 内细化。
- 优点：第一天价值。用户自主。校准锚点而非从零开始。
- 缺点：±0.2 限幅与不匹配检测产生逻辑冲突（Codex #6 发现了这一点）。

**选择：B+C 组合，删除了 ±0.2 限幅。** 底层事件驱动，声明档案作为一等独立字段。无障版。声明和推断作为独立值存在。它们之间的差距被显示但在 v1 中不自动纠正。

### 决策 C：单向门分类 —— 运行时散文解析 vs. 注册表声明 —— 答案：注册表声明（经 Codex 后）

**运行时散文解析（原始）：** `isOneWayDoor(skill, category, summary)` 加关键词模式。
- 优点：技能作者的摩擦力最小。无需维护模式。
- 缺点（Codex #4）：安全性取决于措辞。措辞温和的破坏性问题可能被错误分类。对安全门不可接受。

**注册表声明（修订）：** 每个注册表条目声明 `door_type`。
- 优点：确定性。可审计。CI 可强制（所有问题必须声明）。
- 缺点：维护负担。每个新技能问题必须分类。

**选择：注册表声明为主，关键词模式回退。** 模式治理是安全的代价。

### 决策 D：行内 tune 反馈语法 —— 结构化关键词 vs. 自由英语 —— 答案：结构化 + 自由格式回退

**仅结构化关键词：** `tune: unnecessary | ask-less | never-ask | always-ask | context-dependent`。
- 优点：无歧义。干净的档案数据。
- 缺点：用户必须记住。

**仅自由格式：** 代理解析用户说的任何内容。
- 优点：自然。无需学习语法。
- 缺点：不一致的档案数据。难以调试为什么 tune 没有生效。

**选择：两者兼具。** 高级用户的快捷方式已记录；代理接受并规范自由英语。纯英语交互是默认；结构化关键词是可选快路径。

### 决策 E：/plan-tune 的 CLI 子命令结构 —— 答案：纯英语对话式（无需子命令语法）

**`/plan-tune profile`、`/plan-tune profile set autonomy 0.4` 等。**（原始）：
- 优点：高级用户快。通过 --help 自文档化。
- 缺点：用户必须记住。每次调用感觉像 CLI 会话，而非对话。

**纯英语对话式（经用户纠正后修订）：** `/plan-tune` 进入菜单。用户用自然语言说出他们想要什么。
- 优点：零记忆。感觉像在跟教练交流，而非 shell。
- 缺点：高级用户较慢。需要良好的代理解读。

**选择：对话式 + 可选快捷方式。** 两者都不需要。大多数用户永远不会看到快捷方式。变异声明档案前需要确认步骤（防止代理误解的安全措施 —— Codex #15 信任边界）。

### 决策 F：LANDED 庆祝 —— 被动序言检测 vs. 显式命令 vs. 发运后钩子 —— 答案：推迟到 v2；晋升时，不在序言中

**序言中的被动检测（原始）：** 每个技能的序言运行 `gh pr view` 来检测最近的合并。
- 优点：无论用户运行哪个技能都适用。用户不需要做任何特别的事情。
- 缺点（Codex #9延迟、授权失败、速率限制、意外浏览器打开、注入到每个技能序言中的不可预测性。热路径中的副作用。

**显式命令（`/plan-tune show-landed`）：** 用户选择加入。
- 优点：无热路径副作用。用户控制何时看到它。
- 缺点：需要用户发现。"当你挣得它时给你惊喜"的魔力丢失了。

**发运后钩子（`/ship` 在 PR 创建后触发检测）：** 与 /ship 绑定。
- 优点：时机自然。无序言成本。
- 缺点：/ship 不总是着陆事件（手动合并、团队成员合并等）。

**选择：完全推迟。** v2 将正确设计此功能。晋升时，它离开序言。用户接受了 Codex 的论点，即序言中的庆祝页对于一个已经有风险的功能是战略错位。

### 决策 G：校准门 —— 20 个事件 vs. 多样性检查 —— 答案：多样性检查

**"20 个事件"（原始）：** 简单计数。
- 优点：实现简单。
- 缺点（Codex #13）：可被操控。对 ONE 问题回复 20 行内"unnecessary"不应校准五个维度。

**多样性检查（修订）：** `sample_size >= 20 AND skills_covered >= 3 AND question_ids_covered >= 8 AND days_span >= 7`。
- 优点：档案在接受前实际上贯穿于整个系统。
- 缺点：略复杂。

**选择：多样性检查。** 在 v1 中仅用于"显示足够数据"阈值。在 v2 中将成为心理图谱驱动的自动决策的门。

### 决策 H：实施顺序 —— 分类器优先 vs. 集成点优先 —— 答案：集成点优先（注册表 + CI lint）

**分类器优先（原始）：** 构建分层工具，然后解析器，然后技能模板。
- 优点：原子构建块。可在集成前单元测试。
- 缺点（Codex #19）：围绕一个不可强制执行的约定构建机制。如果约定不成立，所有工作都浪费了。

**集成点优先（修订）：** 首先构建类型化注册表 + CI lint。在构建基础设施之前证明集成工作。
- 优点：基础已证明。基础设施有可靠的东西可以依赖。
- 缺点：需要审核 gstack 中现有的每个 AskUserQuestion —— 大量的预工作。

**选择：集成点优先。** Codex 的论点是决定性的。审核正是要点 —— 它迫使我们在构建适应之前编目我们实际拥有的东西。

### 决策 I：用于 TTHW 的遥测 —— 选择加入遥测 vs. 仅本地 —— 答案：仅本地

**选择加入遥测（原始，在 DX 审查中建议）：** 通过遥测事件测量 TTHW。
- 优点：定量测量所有用户的入职体验。
- 缺点（Codex #14）：与本地优先 OSS 框架矛盾。为此技能专门添加遥测表面。

**仅本地（修订）：** 日志记录是本地的。尊重现有 `telemetry` 配置；技能不添加新的遥测通道。
- 优点：与 gstack 的本地优先精神一致。
- 缺点：无入职时间的聚合视图。

**选择：仅本地。** 如果以后我们需要 TTHW 数据，我们将它作为一个 gstack 范围的遥测事件添加到现有选择加入之后，而非技能特定的。

### 决策 J：档案投毒防御 —— 无防御 vs. 确认门 vs. 用户来源门 —— 答案：用户来源门

**无防御（原始 —— Codex 抓住）：** 代理写入它看到的任何 tune 事件。
- 优点：最简单的。无额外的信任检查。
- 缺点（Codex #16）：恶意仓库内容、PR 描述、工具输出可以注入 `tune: never ask` 并投毒档案。这是一个真实的攻击表面。

**确认门：** 每次 tune 写入提示"确认？[Y/n]"。
- 优点：通用防御。
- 缺点：每次合法使用的摩擦。

**用户来源门：** 代理仅在 `tune:` 前缀出现在用户当前轮次的聊天消息中（而非工具输出，而非文件内容）时写入 tune 事件。二进制验证 `source: "inline-user"`。
- 优点：在不影响合法使用的情况下阻止攻击。
- 缺点：依赖代理正确识别源。二进制级验证是强制执行。

**选择：用户来源门。** 匹配威胁模型（自动化输入中的恶意内容）而不降低正常流。

## 成功标准

- `bun test` 通过包括新的 `test/plan-tune.test.ts`。
- 每个 SKILL.md.tmpl 中的每个 AskUserQuestion 调用都有一个注册表条目。CI lint 强制。
- 从 `~/.gstack/builder-profile.jsonl` 迁移保留 100% 的会话 + signals_accumulated。带 7 会话固定装置的回归测试。
- 单向门注册表声明条目：100% 的破坏性操作、架构分叉、范围增加 > 1 天 CC 努力、安全/合规选择被分类为 `one-way`。
- 用户来源门测试：尝试使用 `source: "inline-tool-output"` 写入 tune 事件被拒绝。
- 内部测试：Garry 使用 `/plan-tune` 达 2+ 周。报告：
  - `tune: never-ask` 感觉自然输入还是被忽略
  - 注册表维护（添加新问题）感觉是合理的纪律还是模式官僚主义
  - 推断维度跨会话稳定还是嘈杂
  - 纯英语交互感觉像在与教练争论还是在与聊天机器人争论

## 实施顺序

1. 审核 gstack 每个 SKILL.md.tmpl 中的每个 `AskUserQuestion` 调用。构建初始 `scripts/question-registry.ts`，包含 ID、类别、door_types、选项。这是基础；其他一切都基于它。
2. 编写 `test/plan-tune.test.ts` 注册表完整性测试（门层级）。验证它捕获漂移 —— 临时删除一个注册表条目，确认 CI 失败。
3. 用关键词模式回退分类器播种 `scripts/one-way-doors.ts`。
4. 用初始 `{question_id, user_choice} → {dimension, delta}` 映射播种 `scripts/psychographic-signals.ts`。数字是暂定的 —— v1 发布，v2 重新校准。
5. 用原型定义播种 `scripts/archetypes.ts`（未来的 v2 `/plan-tune vibe` 引用）。
6. `bin/gstack-question-log` —— 针对注册表验证，拒绝未知 ID。
7. `bin/gstack-question-preference` —— 所有子命令 + 测试。
8. `bin/gstack-developer-profile` —— `--read`（遗产）、`--derive`、`--gap`、`--profile`。
9. 迁移脚本 —— builder-profile.jsonl → 统一的 developer-profile.json。原子、幂等、归档源。带固定装置的回归测试。
10. `scripts/resolvers/question-tuning.ts` —— 三个生成器（偏好检查、日志、带用户来源门指令的行内 tune）。
11. 在 `scripts/resolvers/index.ts` 中注册 3 个解析器。
12. 更新 `scripts/resolvers/preamble.ts` —— `_QUESTION_TUNING` 配置读取；有条件地注入 tier >= 2 技能。
13. `plan-tune/SKILL.md.tmpl` —— 对话式纯英语技能。
14. `bun run gen:skill-docs` —— 所有 SKILL.md 文件重新生成；验证每个保持低于 100KB token 上限。
15. `bun test` —— 所有 45+ 测试用例绿色。
16. 内部吃 2+ 周。收集真实问题日志 + 偏好数据。根据成功标准衡量。
17. `/ship` v1。吃后 v2 范围讨论。

## 开放问题（v2 范围决定，推迟到真实数据）

1. 确切的信号映射增量变化。v1 以初始猜测发布；v2 从观察到的数据重新校准。
2. 当 `inferred` 和 `declared` 差距变大时，我们自动建议更新 `declared`？还是仅显示？
3. 当信号映射版本更改时，我们自动重新计算还是提示用户？默认：带差异显示的自动重新计算。
4. 跨项目档案继承 vs. 隔离。v1 是每项目偏好 + 全局档案；v2 可能添加显式跨项目学习选择加入。
5. /plan-tune 是否应支持一种"团队档案"模式，其中共享的开发者档案通知协作？v2+。

## 已纳入的审查

- **/office-hours（2026-04-16，1 次会话）：** 设置 5 条硬约束，选择事件驱动 + 用户声明的架构。
- **/plan-ceo-review（2026-04-16，EXPANSION 模式）：** 6 个扩展接受，在 Codex 审查后回滚。
- **/plan-devex-review（2026-04-16，POLISH 模式）：** 纯英语交互模型；这个存活到 v1。
- **/plan-eng-review（2026-04-16）：** 测试计划和完整性检查；被注册表优先重写部分取代。
- **/codex（2026-04-16，gpt-5.4 高推理）：** 20 点批评驱动了回滚。Claude 审查错过的 15+ 合法发现。

## 致谢与注意事项

本计划在迭代 AI 协作循环中开发，历时约 6 小时规划。作者（Garry Tan）指导每个范围决策；AI 声音（Claude Opus 4.7 和 OpenAI Codex gpt-5.4）挑战并完善计划。没有 Codex 的外部声音，会发布一个更大且更难辩护的计划。跨模型审查在高风险架构变更上的价值和可衡量。
