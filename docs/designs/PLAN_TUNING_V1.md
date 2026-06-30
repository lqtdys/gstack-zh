# Plan Tuning v1 — 设计文档

**状态：** 已批准实施（2026-04-18）
**分支：** garrytan/plan-tune-skill
**作者：** Garry Tan（用户），由 Claude Opus 4.7 + OpenAI Codex gpt-5.4 协助评审
**取代范围：** 在 [PLAN_TUNING_V0.md](./PLAN_TUNING_V0.md)（观察性基质）之上添加写作风格 + LOC 收据层。V0 保持不变。
**相关：** [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md) —— 提取的节奏大修，V1.1 计划。

## 本文档是什么

/plan-tune v1 的权威记录，包括它是什么、不是什么、我们考虑了什么以及为何做出每个选择。提交到仓库，以便未来的贡献者（以及未来的 Garry）可以在不进行考古的情况下追溯推理。取代任何按用户的本地计划工件。

## 致谢

本计划的存在归功于 **[Louise de Sadeleer](https://x.com/LouiseDSadeleer/status/2045139351227478199)**，她作为非技术用户坐完了完整的 gstack 运行，并告诉我们关于它感觉如何的真相。她的具体反馈：

1. "过了一会儿我有点累了，感觉有点死板。" —— *节奏/疲劳*
2. "我就打算说 yes yes yes"（在架构评审期间）。—— *脱离*
3. "我觉得有趣的是他对他生产了多少行代码的强调。AI 当然为他生产了。" —— *LOC 框架*
4. "作为非工程师，这有点复杂难以理解。" —— *术语密度 + 结果框架*

V1 直接解决 #3 和 #4：术语注释 + 结果框架的写作，读起来像真人写给读者的，加上一个可辩护的 LOC 重构。Louise 的 #1 和 #2（节奏/疲劳）需要单独的设计轮次 —— 提取到 [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md) 作为 V1.1 计划。

## 功能，一句话总结

gstack 技能输出就是产品。如果散文对非技术创始人读得不好，他们就会退出评审并点击"yes yes yes"。V1 添加了一个适用于每个 tier ≥ 2 技能的写作风格标准：首次使用时注释术语（从精选的 ~50 项列表）、用结果术语（"如果你的用户发生了什么..."）而非实现术语构建问题、短句、具体名词。想要更紧凑的 V0 散文的高级用户可以设置 `gstack-config set explain_level terse`。二进制开关，无部分模式。另外：README 的"600,000+ 行生产代码"框架 —— 被 Louise 正确地称为 LOC 虚荣 —— 被一个来自 `scc` 支持脚本的真实计算的 2013-vs-2026 比例倍数替换，附带关于公共 vs 私有仓库可见性的诚实注意事项。

## 为什么构建较小版本

V1 在多次审查通过中经历了四次实质性的范围修订。最终范围比任何中间版本都小，因为每次审查通过都发现了真正的问题。

**修订 1 —— 四级经验轴（拒绝）。** 原始提案：在首次运行时询问用户是有经验的工程师、有经验但无独立经验的工程师、非技术但在团队中发运过的，还是完全非技术的。技能按级别适应。在 CEO 审查的前提挑战步骤中被拒绝，因为（a）入职提问恰好在 V1 试图减少它的时刻增加了摩擦，（b）"我是什么级别？"本身对最需要帮助的用户来说就是一个令人困惑的问题，（c）技术专长不是一维的（设计师在 CSS 上是 A 级，在部署上是 D 级），（d）工程师和非技术用户都从相同的写作标准中受益。

**修订 2 —— 默认 ELI10，简洁选择退出（接受）。** 每个技能的输出默认遵循写作标准。想要 V0 散文的高级用户设置 `explain_level: terse`。Codex 第 1 轮发现了关键缺口（静态 markdown 感知、主机感知路径、README 更新机制）—— 全部三个已集成。

**修订 3 —— ELI10 + 审查节奏大修（提议，范围缩小）。** 添加了一个节奏工作流：对发现项排序、自动接受双向门、每阶段最多 3 个 AskUserQuestion 提示、带翻转命令的静默决策块。旨在直接解决 Louise 的 #1 和 #2。工程审查第 2 轮发现了评分公式和路径一致性错误。工程审查第 3 轮 + Codex 第 2 轮揭示了节奏工作流中 10+ 个无法通过计划文本编辑修复的结构性缺口。

**修订 4 —— ELI10 + 仅 LOC（最终）。** 用户选择范围缩小：在 V1 中发布写作风格 + LOC 收据，将节奏推迟到 V1.1 通过 [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md)。这是批准的 V1 范围。

主线：每次审查通过都正确地缩小了雄心，直到剩余范围没有结构性缺口。匹配 CEO 审查技能的 SCOPE REDUCTION 模式，通过工程审查而非早期战略选择到达。

## v1 范围（我们现在构建的）

1. **序言中的写作风格部分**（`scripts/resolvers/preamble.ts`）。六条规则：每次技能调用时首次使用注释术语、结果框架、短句/具体名词/主动语态、决策以用户影响结束、首次使用无条件注释（即使用户粘贴了该术语）、用户轮次覆盖（用户说"be terse" → 跳过该回复）。
2. **通过仓库拥有的列表进行术语边界**（`scripts/jargon-list.json`）。~50 个精选的高频技术术语。不在列表上的术语被认为足够简单。术语在 `gen-skill-docs` 时内联到生成的 SKILL.md 散文中（零运行时成本）。
3. **简洁选择退出**（`gstack-config set explain_level terse`）。二进制：`default` vs `terse`。Terse 完全跳过写作风格块并使用 V0 散文风格。
4. **主机感知序言回显。** `_EXPLAIN_LEVEL=$(${binDir}/gstack-config get explain_level 2>/dev/null || echo "default")`。通过现有 V0 `ctx.paths.binDir` 模式实现主机可移植。
5. **gstack-config 验证。** 在头部记录 `explain_level: default|terse`。白名单值。对未知值发出特定消息警告并默认到 `default`。
6. **README 中的 LOC 重构。** 移除"600,000+ 行生产代码"英雄框架。插入 `<!-- GSTACK-THROUGHPUT-PLACEHOLDER -->` 锚点。构建时脚本用计算出的倍数 + 注意事项替换锚点。
7. **`scc` 支持的吞吐量脚本**（`scripts/garry-output-comparison.ts`）。对于 2013 + 2026 每一年，枚举 Garry 撰写的公共提交，从 `git diff` 提取添加的行，通过 `scc --stdin`（或正则回退）分类。输出 `docs/throughput-2013-vs-2026.json`，包含每语言分解 + 注意事项。
8. **`scc` 作为独立安装脚本**（`scripts/setup-scc.sh`）。不是 `package.json` 依赖（真正可选 —— 95% 的用户从不运行吞吐量）。操作系统检测并运行 `brew install scc` / `apt install scc` / 打印 GitHub releases 链接。
9. **README 更新管道**（`scripts/update-readme-throughput.ts`）。如果存在，读取 `docs/throughput-2013-vs-2026.json`，用计算出的数字替换锚点。如果缺失，写入 `GSTACK-THROUGHPUT-PENDING` 标记，CI 拒绝 —— 强制贡献者在提交前运行脚本。
10. **/retro 在原始 LOC 之上添加逻辑 SLOC + 加权提交。** 原始 LOC 保留用于上下文但在视觉上被降级。
11. **升级迁移**（`gstack-upgrade/migrations/v<VERSION>.sh`）。一次性升级后交互式提示，提供通过 `explain_level: terse` 恢复 V0 散文供偏好它的用户使用。标记文件门控。
12. **文档。** CLAUDE.md 获得写作风格部分（项目约定）。CHANGELOG.md 获得 V1 条目（用户面向的叙事，提及范围缩小 + V1.1 节奏）。README.md 获得写作风格解释部分（~80 字）。CONTRIBUTING.md 获得关于 jargon-list 维护的注释（添加/删除术语的 PR）。
13. **测试。** 6 个新测试文件 + 扩展现有 `gen-skill-docs.test.ts`。除 LLM-judge E2E（周期性）外全部为门层级。
14. **V0 休眠负面测试。** 断言 5D 维度名称和 8 个原型名称不出现在默认模式技能输出中。防止 V0 心理图谱机械泄漏到 V1 中。
15. **V1 和 V1.1 设计文档。** PLAN_TUNING_V1.md（此文件）。PACING_UPDATES_V0.md（V1.1 计划，在 V1 实施期间从提取的附录创建）。TODOS.md P0 条目。

## 推迟

**到 V1.1（显式，有专门的设计文档）：**
- 审查节奏大修（排序、自动接受、每阶段最多 3 个、静默决策块、翻转机制）。理由：见 [PACING_UPDATES_V0.md](./PACING_UPDATES_V0.md) §"为何被提取"。有 10+ 个无法通过纯散文更改修复的结构性缺口。
- 序言首次运行元提示审计（lake 介绍、遥测、主动、路由）。Louise 在首次运行时全部看到它们；它们计入疲劳。V1.1 考虑抑制直到第 N 个会话。

**到 v2（或更晚）：**
- 从问题日志驱动混乱信号检测，提供即时翻译报价。
- 5D 心理图谱驱动的技能适应（V0 E1 项）。
- /plan-tune 叙事 + /plan-tune 氛围（V0 E3 项）。
- 每技能或每主题的解释级别。
- 团队档案。
- 基于 AST 的"已交付功能"指标。

## 完全拒绝（考虑过，不做）

- **四级声明经验轴（A/B/C/D）。** 在 CEO 审查前提挑战期间被拒绝。见上面"为什么构建较小版本"。
- **ELI10 作为新的解析器文件（`scripts/resolvers/eli10-writing.ts`）。** Codex 第 1 轮发现与序言 AskUserQuestion Format 部分中现有的"smart 16-year-old"框架冲突。折叠到现有序言中。
- **运行时抑制写作风格块。** Codex 第 1 轮发现 `gen-skill-docs` 生成静态 Markdown —— 运行时 `EXPLAIN_LEVEL=terse` 无法隐藏已经烘焙的内容。解决方案：条件散文门（散文约定，与 V0 的 `QUESTION_TUNING` 门同类）。
- **默认和简洁之间的中间写作模式。** 修订 3 提议"terse = 无注释但保留结果框架"。Codex 第 2 轮发现与迁移消息的矛盾。二进制胜出：terse = V0 散文，完全停止。
- **运行时可编辑的术语列表。** 修订 3 提议 `~/.gstack/jargon-list.json` 作为用户覆盖。Codex 第 2 轮发现与生成时内联的矛盾。解决：仅仓库拥有，PR 添加/删除，重新生成以生效。
- **`package.json` 中的 `devDependencies.optional` 字段。** 不是真正的 npm/bun 字段。工程审查第 2 轮发现。独立安装脚本代替。
- **在 README 中使用相同字符串作为替换锚点和 CI 拒绝标记。** 工程审查第 2 轮 / Codex 第 2 轮发现这会使管道破坏自己的更新路径。双字符串解决方案：`GSTACK-THROUGHPUT-PLACEHOLDER`（锚点，跨运行保持）vs `GSTACK-THROUGHPUT-PENDING`（显式"构建未运行"标记，CI 拒绝）。
- **"每个技术术语都得到注释"作为验收标准。** Codex 第 2 轮发现与精选列表规则的矛盾。验收重写以匹配规则："出现在 `scripts/jargon-list.json` 上的每个术语都得到注释。"
- **验收标准"每次 /autoplan ≤ 12 个 AskUserQuestion 提示。"** 从 V1 中移除 —— 该目标需要现在在 V1.1 中的节奏大修。

## 架构

```
~/.gstack/
  developer-profile.json           # 从 V0 未变
  config.yaml                       # + explain_level 键（default | terse）

scripts/
  jargon-list.json                  # 新建：~50 个仓库拥有的术语（生成时内联）
  garry-output-comparison.ts        # 新建：scc + git 每年，作者范围
  update-readme-throughput.ts       # 新建：README 锚点替换
  setup-scc.sh                      # 新建：操作系统检测的 scc 安装器
  resolvers/preamble.ts             # 修改：写作风格部分 + EXPLAIN_LEVEL 回显

docs/
  designs/PLAN_TUNING_V1.md         # 新建：此文件
  designs/PACING_UPDATES_V0.md      # 新建：V1.1 计划（提取）
  throughput-2013-vs-2026.json      # 新建：计算出的，已提交

~/.claude/skills/gstack/bin/
  gstack-config                     # 修改：explain_level 头部 + 验证

gstack-upgrade/migrations/
  v<VERSION>.sh                     # 新建：V0 → V1 交互式提示
```

### 数据流

```
用户运行 tier-≥2 技能
       │
       ▼
序言 bash（每次调用）：
  _EXPLAIN_LEVEL=$(${binDir}/gstack-config get explain_level 2>/dev/null || "default")
  echo "EXPLAIN_LEVEL: $_EXPLAIN_LEVEL"
       │
       ▼
生成的 SKILL.md 主体（静态 Markdown，在 gen-skill-docs 时烘焙）：
  - AskUserQuestion Format 部分（现有 V0）
  - Writing Style 部分（新建，条件散文门）
       │
       ├── "如果 EXPLAIN_LEVEL: terse 或用户此轮说'be terse'则跳过"
       ├── 6 条写作规则（术语、结果、短、影响、首次使用、覆盖）
       └── 从 scripts/jargon-list.json 内联的术语列表
       │
       ▼
代理基于运行时 EXPLAIN_LEVEL + 用户轮次信号应用或跳过
       │
       ▼
V0 QUESTION_TUNING + 问题日志 + 偏好未变
       │
       ▼
输出给用户（首次使用注释、结果框架、短句；或如果 terse 则为 V0 散文）
```

### 数据流：吞吐量脚本（构建时）

```
bun run build
   │
   ├── gen:skill-docs（重新生成 SKILL.md 文件，内联术语列表）
   ├── update-readme-throughput（如果存在读取 JSON；替换锚点或写入 PENDING 标记）
   └── 其他步骤（二进制编译等）

单独，按需：
bun run scripts/garry-output-comparison.ts
   │
   ├── scc 预检（如果缺失 → 以 setup-scc.sh 提示退出）
   ├── 对于 2013 + 2026：枚举公共 garrytan/* 仓库中 Garry 撰写的提交
   ├── 对于每个提交：git diff，提取 ADDED 行，通过 scc --stdin 分类
   └── 写入 docs/throughput-2013-vs-2026.json（每语言 + 注意事项）
```

## 安全与隐私

- **无新用户数据。** V1 扩展序言散文 + 配置键。不收集新的个人数据。
- **不运行时读取敏感数据文件。** 术语列表是仓库提交的精选列表。
- **迁移脚本是一次性的。** 标记文件防止重新触发。
- **scc 仅在公共仓库上运行。** 不访问私有工作。

## 决策日志（含利弊）

### 决策 A：四级经验轴 vs. 默认 ELI10 —— 答案：默认 ELI10

**四级轴（拒绝）：** 要求用户在首次运行时自标识为 A/B/C/D。技能按级别适应。
- 优点：显式用户自主权。高级用户获得 V0 行为。
- 缺点：增加入职摩擦。强迫用户给自己贴标签。技术专长不是一维的。工程师和非技术用户都从相同的写作标准中受益。

**默认 ELI10 + 简洁选择退出（选择）：** 每个技能的输出默认遵循写作标准。高级用户设置 `explain_level: terse`。
- 优点：无入职问题。好的写作对所有人有益。高级用户仍有逃生舱。
- 缺点：在升级时静默改变 V0 行为 → 需要迁移提示。

### 决策 B：新解析器文件 vs. 扩展现有序言 —— 答案：扩展现有

**新解析器（拒绝）：** `scripts/resolvers/eli10-writing.ts` 作为单独的生成器。
- 优点：模块化。
- 缺点（Codex #7）：与序言 AskUserQuestion Format 部分中现有的"smart 16-year-old"框架冲突。两个真相来源。

**扩展序言（选择）：** 写作风格部分直接添加到 `scripts/resolvers/preamble.ts` 中 AskUserQuestion Format 下方。
- 优点：一个真相来源。与现有规则组合。
- 缺点：`preamble.ts` 增长。

### 决策 C：运行时抑制 vs. 条件散文门 —— 答案：条件散文门

**运行时抑制（拒绝）：** 序言读取 `explain_level` 触发抑制逻辑。
- 优点：更简单的心理模型。
- 缺点（Codex #1）：`gen-skill-docs` 生成静态 Markdown。一旦烘焙，内容无法被追溯隐藏。运行时抑制是虚构的。

**条件散文门（选择）：** "如果 EXPLAIN_LEVEL: terse 或用户此轮说'be terse'则跳过此块。"散文约定；代理在运行时遵守或不遵守。
- 优点：可测试。匹配 V0 的 `QUESTION_TUNING` 模式。对机制诚实。
- 缺点：依赖代理散文合规（无硬运行时门）。

### 决策 D：术语列表位置 —— 运行时可编辑 vs. 仓库拥有生成时 —— 答案：仓库拥有生成时

**运行时可编辑（拒绝）：** `~/.gstack/jargon-list.json` 覆盖 `scripts/jargon-list.json`。
- 优点：用户可以添加特定于其领域的术语。
- 缺点（Codex #4，第 2 轮）：生成时内联意味着用户编辑需要重新生成。矛盾。

**仓库拥有，生成时内联（选择）：** 仅 `scripts/jargon-list.json`。PR 添加/删除。`bun run gen:skill-docs` 将术语内联到序言散文中。
- 优点：一个真相来源。零运行时成本。与现有构建可组合。
- 缺点：用户不能本地添加术语。缓解：在 CONTRIBUTING.md 中记录；接受 PR。

### 决策 E：V1 中的节奏大修 vs. V1.1 —— 答案：V1.1（提取）

**V1 中的节奏（拒绝）：** 捆绑排序 + 自动接受 + 静默决策 + 每阶段最多 3 个上限 + 翻转机制。
- 优点：直接解决 Louise 的疲劳。
- 缺点（工程审查第 3 轮 + Codex 第 2 轮）：10+ 个无法通过计划文本编辑修复的结构性缺口。会话状态模型未定义。问题日志中缺少 `phase` 字段。注册表不覆盖动态审查发现项。翻转机制没有实现。迁移提示本身就是一个打断。首次运行序言提示也计入。作为散文的节奏无法反转现有的每段提问执行顺序。

**提取到 V1.1（选择）：** 在 V1 中发布 ELI10 + LOC。节奏获得自己的设计轮次和完整审查循环。
- 优点：诚实地发布 V1。从 V1 使用（Louise 的 V1 转录）为 V1.1 提供真实基线数据。匹配 CEO 审查的 SCOPE REDUCTION 模式。
- 缺点：Louise 的疲劳投诉直到 V1.1 才完全解决。缓解：V1 仍通过写作质量改善她的体验；V1.1 随后跟进节奏。

### 决策 F：README 更新机制 —— 单字符串 vs. 双字符串 —— 答案：双字符串

**单字符串（拒绝）：** `<!-- GSTACK-THROUGHPUT-MULTIPLE: N× -->` 作为替换锚点和 CI 拒绝标记。
- 优点：简单。
- 缺点（Codex 第 2 轮）：管道自我破坏 —— CI 拒绝包含标记的提交，但标记就是锚点。

**双字符串（选择）：** `GSTACK-THROUGHPUT-PLACEHOLDER`（锚点，稳定）+ `GSTACK-THROUGHPUT-PENDING`（显式缺失构建标记，CI 拒绝）。
- 优点：锚点持久化；CI 捕获实际故障状态。
- 缺点：两个符号要记住。

## 审查记录

| 审查 | 运行次数 | 状态 | 已纳入的关键发现 |
|---|---|---|---|
| CEO 审查 | 1 | 通过（保持范围） | 前提转向：四级轴 → 默认 ELI10。通过显式用户选择解决跨模型张力。 |
| Codex 审查 | 2 | 发现问题 + 驱动范围缩小 | 第 1 轮：25 个发现，3 个关键阻碍（静态 markdown、主机路径、README 机制）。第 2 轮：修订计划上的 20 个发现，驱动 V1.1 提取。 |
| 工程审查 | 3 | 通过（范围缩小） | 第 1 轮：关键缺口 + 3 个决策（全部 A）。第 2 轮：评分公式错误、路径矛盾、虚假的 `devDependencies.optional` 字段。第 3 轮：识别节奏结构性缺口，驱动提取。 |
| DX 审查 | 1 | 通过（分流） | 3 个关键（文档计划、升级迁移、英雄时刻）。9 个作为静默 DX 决策自动接受。 |

审查报告通过 `gstack-review-log` 持久化在 `~/.gstack/` 中。计划文件以完整历史保留在 `~/.claude/plans/system-instruction-you-are-working-transient-sunbeam.md`。
