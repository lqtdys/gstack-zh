# gstack v2 — 最轻量的观点化技能包

## 背景

gstack有一个在外界文档中公认的"臃肿"名声。第三方评论（dev.to，2026年5月）明确说gstack"全开所有角色时会感觉臃肿...在写任何实际代码之前就可能消耗10K+令牌，日常使用令牌消耗快...让即使是直接的任务也感觉迟钝和冗余。"Anthropic自己权威的Skills指南规定"渐进式披露"模式（`SKILL.md`骨架+按需求的`references/`加载）— gstack偏离了这个规范。

数据支持了批评：

- 31个技能，总计生成的SKILL.md语料2.1MB
- 31个中有28个超过40KB软上限（约每个10K令牌）
- ship.md是164KB（约41K令牌）；ship.md.tmpl只有48KB — **115KB是解析器注入的**，这是最具杠杆的压缩目标
- 目录在始终加载的系统提示中：50+技能 × 多段描述、语音触发、主动建议段落

此计划通过两次协调的发布交付gstack v2：v1.45.0.0落地基础架构+低风险的胜利，然后v2.0.0.0在2-4周后发布架构性断裂+营销级别的重新定位。这种分裂来自跨模型审查：Codex认为没有实质性断裂的v2看起来像姿态声明；混合形状给予真正断裂的sections/模式本应获得的大版本号，同时让无风险胜利立即发布。

## 发布形状

```
v1.45.0.0 (基础发布)          v2.0.0.0 (gstack v2发布)
─────────────────────────────           ─────────────────────────────
~1-2周CC工作                   2-4周后，协调发布

第0阶段：评估覆盖度矩阵          第B阶段：sections/模式
  31个技能的闸口+周期性         5个重量级技能
                                   (ship, plan-ceo, office-hours,
第A阶段：构建时压缩                plan-eng, plan-design)
  条件解析器注入
  术语去重                    第C阶段：评估注释
  精简模式实际上压缩            + CI孤儿检查(WARN→FAIL)

目录裁剪（Codex高杠杆胜利）    轻迁移
  单行技能描述                  配套说明 + 自动重新生成
  丢弃语音触发/主动块            到 /gstack-upgrade

硬令牌预算定义                 营销级CHANGELOG
  通过budget回归强制执行         v1 vs v2数字表
                                  README v2横幅
正常发布语音                     "最轻量观点化技能包"
```

## 前提检查（第0A步发现）

1. **这是正确的问题吗？** YES — 经外部验证。臃肿的批评是可引用的，代表真实用户痛苦（令牌成本、迟钝会话）。什么都不做意味着输给Cursor/Codex的"轻触"名声。
2. **什么都不做：** 批评会合流。最近版本（v1.38→v1.44）都添加功能；没有发布走相反方向。没有明确的扭转，名声会固化。
3. **行动风险：** lazy-section模式引入了一个沉默行为丢失作为新的失败类。由评估优先基础架构+机械执行+金丝雀发布缓解（见第B阶段完整性部分）。

## 已经存在的（重用优先审计）

| 资产 | 重用 |
|---|---|
| `scripts/gen-skill-docs.ts` 第439-450行 | 已做字符串替换和按主机抑制；用 `appliesTo` 解析器门扩展（~15 LOC） |
| `scripts/resolvers/types.ts` | 添加 `ResolverEntry` 联合类型 |
| `scripts/resolvers/preamble.ts` | 已做分层门控组合（1-4）；添加按解析器门控 |
| `scripts/jargon-list.json` | 已是单文件；停止内联37× |
| `test/skill-e2e-budget-regression.test.ts`（现有闸口层） | 用每技能硬预算扩展 |
| 来自v1.13.2.0的真实PTY工具 | 重用为行为契约评估（约$0.50/评估） |
| SDK工具 | 重用为廉形状评估（~$0/评估尽可能） |
| `gstack-upgrade/migrations/` | 状态格式迁移模式存在；重用于v2自动重新生成 |
| `~/.gstack/analytics/skill-usage.jsonl` | 已收集；驱动延迟 `gstack budget` CLI |

我们正在赶上Anthropic的规范Skills模式，而不是自己发明一个。

## 梦想状态增量

```
今天                              v1.45.0.0                         v2.0.0.0
──────                              ─────────                         ────────
2.1MB语料                         ~1.3MB语料(-40%)              ~700KB语料(-67%)
ship.md: 164KB                     ship.md: ~80KB(-50%)            ship.md: ~15KB骨架
                                                                     + 5×~5KB sections
28/31超过40KB上限                 ~10/31超过上限                ~3/31超过上限
                                                                     (cso, document-release,
                                                                      design-consultation
                                                                      保持为单体)
目录：多段                       目录：每技能单行              目录：每技能单行
描述，语音触发                    (~70%目录裁剪)                (相同)
无评估覆盖度矩阵                  每个技能：≥1闸口评估          Section级评估
                                  + ≥1周期评估                  注释+CI孤儿检查
第三方中的"臃肿"                  内部测量                      外部测量
声誉                              "压缩，评估保护"               "最轻量观点化技能包"
```

## 第0阶段 — 评估覆盖度矩阵（v1.45.0.0）

**目标：** gstack中的每个技能至少有一个闸口级评估和一个周期级评估，断言必须有行为。评估套件成为设计规范。这是计划中承重的主张 — 必须先来。

**记录的跨模型张力：** Codex认为这是拖延陷阱，形状断言是浅的。用户明确定义了全层覆盖（D9 = A），理由是："评估套件就是设计规范；该承诺是整个计划的承重主张。"我们接受更大前期投资。

**缓解Codex"形状vs质量"批评：** 对于编排/判断技能（plan-ceo、office-hours、autoplan），必须有的不是确定性输出 — 是结构合规（它是否以正确形状调用AskUserQuestion？它是否遵循section顺序？它是否持久化构件？）。评估设计必须捕获结构契约，不是输出内容。当结构评估不可能时，该section明确标注为"依赖判断，非评估保护" — 通过不厌其烦地从切割未受保护的判断散文来尊重Codex的#2批评。

**当前缺乏专用E2E覆盖度的技能**（评估编写目标）：

| 技能 | 闸口评估（目标） | 周期评估（目标） | 估计成本/运行 |
|---|---|---|---|
| qa-only | 仅报告标志触发 | 带禁用修复循环的完整QA流 | $0.30 / $1.50 |
| retro | 每周聚合运行无错误 | 完整retro产生分级输出 | $0.20 / $2.00 |
| document-release | 读取CHANGELOG，生成Diataxis映射 | 完整post-ship文档更新 | $0.30 / $1.80 |
| document-generate | 从提示生成4种文档类型 | E2E生成通过质量门槛 | $0.30 / $2.00 |
| context-save | 持久化状态到预期路径 | 往返恢复保留上下文 | $0.10 / $0.50 |
| context-restore | 读取最新保存，应用到会话 | 跨工作区恢复工作 | $0.10 / $0.50 |
| gstack-upgrade | 检测安装类型，升级运行 | 完整升级+迁移往返 | $0.20 / $1.00 |
| sync-gbrain | 刷新索引无错误 | 完整同步产生可搜索语料 | $0.20 / $1.50 |
| setup-gbrain | 路径1-4检测工作 | 每个路径的端到端设置 | $0.20 / $2.00 |
| setup-browser-cookies | 选择器UI加载无错误 | Cookie导入往返 | $0.20 / $1.00 |
| setup-deploy | 检测配置，写入预期文件 | 完整部署配置设置 | $0.20 / $1.00 |
| design-consultation | DESIGN.md模板渲染 | 完整设计系统生成 | $0.30 / $2.50 |
| design-shotgun | 变体已生成并保存 | 完整多变体探索 | $0.30 / $2.00 |
| open-gstack-browser | 浏览器启动无错误 | 侧边栏附加并显示活动 | $0.20 / $0.80 |
| pair-agent | 设置密钥生成，说明打印 | 与第二个Agent的完整配对流 | $0.20 / $1.50 |
| land-and-deploy | 合并门控检查正确 | 完整合并→部署→金丝雀 | $0.30 / $3.00 |
| canary | post-deploy循环运行，干净退出 | 带警报模拟的完整金丝雀循环 | $0.20 / $1.50 |
| benchmark | 运行并生成分数 | 完整回归检测 | $0.20 / $2.00 |
| plan-devex-review | 模式路由工作 | 带评分的完整DX审查 | $0.40 / $3.00 |
| devex-review | 实时DX审计生成记分卡 | E2E DX测量vs计划基线 | $0.40 / $2.50 |

估计增加的成本：**~$5/运行闸口，~$30/运行周期。** 与现有E2E套件结合（~$15/闸口，~$30/周期），总计：~$20/闸口（每次PR），~$60/周期（每周）。可接受。

**评估矩阵位于：** `test/helpers/skill-coverage-matrix.ts` — 单一真实来源，映射每个技能到其闸口+周期评估测试文件。`test/skill-coverage-matrix.test.ts`中的CI检查在任何技能缺少条目时构建失败。

**添加的关键文件：**
- `test/skill-coverage-matrix.ts` — 注册表映射技能→评估路径
- `test/skill-e2e-*.test.ts` — 20个新测试文件（闸口子集从闸口配置开始，周期子集在周期配置中）
- `test/helpers/touchfiles.ts` — 注册新测试用于基于diff的选择

## 第A阶段 — 构建时压缩（v1.45.0.0）

**A.1条件解析器注入** — 扩展 `scripts/gen-skill-docs.ts` 和 `scripts/resolvers/`：

```ts
// scripts/resolvers/types.ts
export type ResolverFn = (ctx: TemplateContext, args?: string[]) => string;
export type ResolverEntry = ResolverFn | {
  resolve: ResolverFn;
  appliesTo?: (ctx: TemplateContext) => boolean;
};
```

```ts
// scripts/resolvers/index.ts — 门控重度解析器
QUESTION_TUNING: {
  resolve: generateQuestionTuning,
  appliesTo: (ctx) => ['plan-ceo-review','plan-eng-review','office-hours'].includes(ctx.skillName),
},
REVIEW_ARMY: {
  resolve: generateReviewArmy,
  appliesTo: (ctx) => ['ship','review'].includes(ctx.skillName),
},
REVIEW_DASHBOARD: {
  resolve: generateReviewDashboard,
  appliesTo: (ctx) => ['ship','plan-ceo-review','plan-eng-review','plan-design-review','plan-devex-review','devex-review'].includes(ctx.skillName),
},
// ... 审计所有21个解析器，按实际使用门控
```

```ts
// scripts/gen-skill-docs.ts（约第444行）— 检查门控
const entry = RESOLVERS[resolverName];
const resolver = typeof entry === 'function' ? entry : entry.resolve;
const gate = typeof entry === 'function' ? undefined : entry.appliesTo;
if (gate && !gate(ctx)) return '';
return args.length > 0 ? resolver(ctx, args) : resolver(ctx);
```

**A.2术语列表去重** — 当前 `scripts/resolvers/preamble/generate-writing-style.ts` 将完整1.8KB术语表内联到37个技能中。用引用替换内联："对于规范术语列表，首次使用时读取 `~/.claude/skills/gstack/scripts/jargon-list.json`。" 总计节省约66KB语料。

**A.3精简模式真正压缩** — 在 `gen-skill-docs.ts` 中读取一次 `~/.gstack/config.yaml`，将 `explainLevel` 传入 `TemplateContext`，让 `generate-writing-style.ts` / `generate-completeness.ts` / `generate-confusion-protocol.ts` / `generate-context-health.ts` 在精简时返回 `''`。今天字节不管配置如何都发送 — 标志只改变运行时模型行为。添加 `--explain-level=terse` 构建标志用于基准测试。

**A.4目录裁剪**（按#6提前至Codex）— 将始终加载的系统提示中的技能描述缩短为每技能一行。语音触发从目录描述移入技能内内容。主动建议段落移到单独的 `~/.claude/skills/gstack/scripts/proactive-suggestions.json`，只在Agent需要路由指导时加载。每技能描述格式：

```
-<技能名>: <单行结果描述，≤80字符> (gstack)
```

估计目录裁剪：~70%（最大单次始终加载减少）。

**A.5 cso/定向压缩**（Codex #9）— cso得到解析器去重+目录裁剪。安全指导散文以单体未压缩保持，直到第B阶段审计证明特定sections可以安全地移到sections/并使用评估覆盖度。不是"豁免"— 只是最后排序。

**A.6硬令牌预算**（Codex #10）— 在 `test/skill-e2e-budget-regression.test.ts` 中定义并强制执行：

| 预算 | v1.44实际 | v1.45目标 | v2.0目标 |
|---|---|---|---|
| 最大系统提示目录令牌 | ~25K | ~8K | ~6K |
| 最大每技能SKILL.md大小 | 164KB (ship) | 100KB | 30KB（重量级技能） |
| 最大语料总计 | 2.1MB | 1.3MB | 700KB |
| 最大首次调用延迟（重量级） | ~立即 | ~立即 | <500ms sections读取 |

CI在任何预算超标时失败。长期通过现有budget回归jsonl跟踪。

## 第B阶段 — 重量级技能的sections/模式（v2.0.0.0）

将5个重量级技能转换为Anthropic规范骨架+ `sections/*.md`：

```
ship/
├── SKILL.md              # 12-15KB决策树骨架 + section清单
├── SKILL.md.tmpl         # 骨架的源
├── sections/
│   ├── manifest.json     # 新：结构化section注册表（Codex #3缓解）
│   ├── version-bump.md
│   ├── changelog.md
│   ├── review-army.md
│   ├── todos-cleanup.md
│   ├── pr-body.md
│   └── ...
```

**沉默行为丢失缓解**（Codex #3）— 分层防御，不仅自检：

1. **Section清单**（`sections/manifest.json`）— 结构化注册表：`{section_file, applies_when, required_for}`。决策树骨架通过ID而非自由散文引用条目。
2. **命令式骨架措辞** — "停止。在计算bump前读取 `sections/version-bump.md`。" 不是"详见..."。
3. **文件顶部section索引表** — 情境→section文件映射。
4. **技能末尾自检** — "确认你读取了决策树指向的每个section。列出来。"（最弱层，保持为回退。）
5. **评估工具 `requiredReads` 声明** — E2E测试断言给定夹具的transcript Read调用中必须出现哪些section。在测试层而非仅提示层的机械执行。
6. **金丝雀群体中的transcript检查** — 发布后首周，记录实际会话读取了哪些section；对标记required的section的Read遗漏警报。

**转换顺序**（一次一个，验证后再下一个）：
1. `ship/` — 最多调用，最大成本，最危险。单独落地，观察1周。
2. `plan-ceo-review/` — 对话式；破坏流程的风险。第二个落地，仔细观察。
3. `office-hours/` — 最对话式。第三个落地（仅当1+2清理干净）。
4. `plan-eng-review/`和`plan-design-review/` — 捆绑，形状相似。

**不要转换**除非后续明确批准：`autoplan`（编排器，已链接技能）、`design-review`（UI流已紧凑）、`qa`（单一目的）、`investigate`（单一目的）。

## 第C阶段 — 评估注释+CI孤儿检查（v2.0.0.0）

Per Codex #4 — 警告-然后-失败的进展，不是立即严格门控。

```md
<!-- eval: test/skill-e2e-ship-version-bump.test.ts -->
<!-- coverage: 断言队列感知bump在声称版本已占用时选择下一个可用版本 -->
```

注释包括**覆盖语义**（保护什么行为），per Codex #5，不仅是路径。仅有路径会是虚假信心。

`gen-skill-docs.ts` walker中的CI检查：
- v2.0.0.0以WARN模式发布 — 孤儿被记录到PR摘要但构建通过
- v2.1.0.0（或2个发布周期后）：WARN升级为FAIL
- 豁免：`<!-- eval: none — 接受丢失，YYYY-MM-DD由@user审查 -->`

这会避免强制注释无语义的"维护剧场"，并给用户一个过渡窗口。

## 迁移方法（v2.0.0.0，较轻触 per D11）

- v2.0.0.0 CHANGELOG中的配套说明解释sections/格式变化和具体用户影响：fork/复制粘贴的SKILL.md文件需要重新获取；重量级技能的首次调用增加约200-500ms section读取延迟。
- `/gstack-upgrade` 在下次调用时自动重新生成。无交互式迁移提示。
- 厂商安装在第一次v2联系时会话开始显示单行警告（重用现有厂商安装警告模式在skill前言中）。
- `gstack-upgrade --explain-v2` 标志供用户按需获取完整解释。

## Forks/自定义兼容性（Codex #11）

记录在v2.0.0.0配套说明中：

- 任何直接读取/复制/编辑重量级SKILL.md文件的人：文件现在是骨架；行为生活在 `sections/*.md` 中。他们要么把技能当作黑盒（推荐），要么fork整个 `skill/` 目录包括 `sections/`。
- 任何fork中有本地SKILL.md.tmpl编辑的人：模板更小；重新生成时可能有冲突。Fork文档更新迁移指南。
- 任何文档/博客帖子链接到生成的SKILL.md特定行的人：行号会移位；推荐链接到模板+section名称。

## 推出策略（Codex #12）

v1.45.0.0：
- 一次PR落地；现有预算回归测试捕获任何每技能大小回归；评估矩阵CI检查捕获缺少评估的技能。
- Dogfood：1周主动使用横跨Garry的所有工作区然后公告。

v2.0.0.0：
- **金丝雀群体**：通过v2.0.0-rc.1标签先发布给dogfood用户（Garry+活跃Agent）。真实PTY工具记录前5个工作流（`/ship`、`/qa`、`/review`、`/plan-ceo-review`、`/autoplan`）的section Reads；对required section的Read遗漏警报。
- **手动验证**：在标记v2.0.0.0最终版之前手动运行前5个工作流，带有before/after transcript保存为评估基线。
- **回归仪表板**：现有 `bun run eval:summary` 扩展为v1 vs v2每技能令牌+行为合规比较。
- **回退**：revert PR + `bun run gen:skill-docs` 重新生成旧形状。文档在CONTRIBUTING.md中。

## 审查部分发现（部分1-11，浓缩）

| 部分 | 发现 | 状态 |
|---|---|---|
| 1. 架构 | Lazy-section沉默丢失风险；通过上述6层防御缓解 | 发现已在计划中解决 |
| 2. 错误/救援 | gen-skill-docs gate失败响声大；缺失sections回退到骨架；CI孤儿检查响声大 | 发现已解决 |
| 3. 安全 | cso定向去重非整体豁免（Codex #9）；迁移脚本在user-shell信任边界运行，同现有迁移 | 发现已解决 |
| 4. 数据/UX边界情况 | v1→v2肌肉记忆断裂在配套说明中警告；厂商安装获得单行警告；并发dev-符号链会话风险是现有CLAUDE.md警告 | 发现已解决 |
| 5. 代码质量 | 跨gen-skill-docs/types/index约150 LOC附加；~20个新评估测试文件；sections/提取是机械的 | OK |
| 6. 测试 | 第0阶段就是测试计划。覆盖度矩阵CI门控强制每个技能都有评估 | 发现已解决 |
| 7. 性能 | 构建时间 <2× 当前；运行时首次调用为sectioned重量级增加200-500ms；目录裁剪减少每次会话的始终加载提示大小 | 文档化 |
| 8. 可观察性 | budget回归测试已存在；第B阶段金丝雀群体transcript记录；迁移结果记录到 ~/.gstack/analytics/migrations.jsonl | 发现已解决 |
| 9. 部署 | 两发布分裂+警告-then-评估注释+通过revert回退 | 发现已解决 |
| 10. 长期轨道 | 可逆性3/5；sections/模式成为未来技能的模板；延迟TODOS将v2叙事扩展到v2.1+ | OK |
| 11. 设计/UX | README v2横幅+CHANGELOG数字表落在v2.0.0.0；具体数字，gstack语音，无AI敷衍 | OK |

## 不在范围内

- **技能移除。** 用户说"保留所有功能。"qa-only、design-shotgun、pair-agent、open-gstack-browser全部保留。它们得到评估+目录裁剪。
- **技能重命名。** 无 `qa` → `qa-fix` 合并。保持CLI表面稳定。
- **gstack lite/pro安装配置文件。** 延迟到TODOS用于post-v2。
- **gstack budget CLI。** 延迟到TODOS用于post-v2。
- **README中每技能评估覆盖徽章。** 延迟到TODOS。
- **跨工具可移植性测试/演示（Codex/Cursor兼容）。** 延迟到TODOS。
- **调用时的令牌成本预览。** 延迟到TODOS。
- **技能自动加载遥测。** 延迟到TODOS。
- **gstack diff PR注释。** 延迟到TODOS。

## TODOS.md更新（延迟项，建议合并后批量添加）

| TODO | 优先级 | 工作量（人类/CC） | 依赖 |
|---|---|---|---|
| `gstack lite` 安装配置文件（5技能核心） | P2 | 2天/3-4小时 | v2.0.0.0 |
| `gstack pro` 选择加入升级路径 | P2 | 1天/1小时 | gstack lite |
| `gstack budget` CLI（每技能令牌使用遥测） | P2 | 1天/1小时 | v1.45.0.0 |
| `gstack-skills list` + README中每技能评估覆盖徽章 | P3 | 1天/1小时 | 第0阶段 |
| 跨工具可移植性测试/演示（Codex CLI、Cursor） | P3 | 2天/2小时 | v2.0.0.0 |
| 技能调用时的令牌成本预览 | P3 | 1天/1小时 | gstack budget CLI |
| 技能自动加载遥测（无用功能检测） | P3 | 2天/2小时 | v1.45.0.0 |
| `gstack diff` PR注释（每PR预算增量） | P3 | 1天/1小时 | budget回归扩展 |
| Section级评估注释对用户可见（信心信号） | P3 | 半天/30分钟 | 第C阶段 |

## 关键文件

| 路径 | 变更 | 阶段 |
|---|---|---|
| `scripts/gen-skill-docs.ts` | 添加解析器门控检查（约第444行）；从config读取explain_level；添加CI孤儿walker | A, C |
| `scripts/resolvers/types.ts` | 添加 `ResolverEntry` 联合类型 | A |
| `scripts/resolvers/index.ts` | 用 `appliesTo` 谓词包装重度解析器（审计所有21个） | A |
| `scripts/resolvers/preamble/generate-writing-style.ts` | 替换内联术语；精简时返回 `''` | A |
| `scripts/resolvers/preamble/generate-completeness.ts` | 精简时返回 `''` | A |
| `scripts/resolvers/preamble/generate-confusion-protocol.ts` | 精简时返回 `''` | A |
| `scripts/resolvers/preamble/generate-context-health.ts` | 精简时返回 `''` | A |
| `scripts/skill-catalog.ts`（新或在gen-skill-docs中） | 单行目录生成器+语音触发JSON分裂器 | A.4 |
| `scripts/proactive-suggestions.json`（新） | 语音触发+主动建议，按需加载 | A.4 |
| `test/skill-coverage-matrix.ts`（新） | 单一真实来源评估注册表 | 第0阶段 |
| `test/skill-coverage-matrix.test.ts`（新） | CI门控：每个技能有条目 | 第0阶段 |
| `test/skill-e2e-*.test.ts`（~20新文件） | 当前缺乏覆盖度的新评估 | 第0阶段 |
| `test/helpers/touchfiles.ts` | 注册新测试用于基于diff的选择 | 第0阶段 |
| `ship/SKILL.md.tmpl` → `ship/sections/manifest.json` + `ship/sections/*.md` | 骨架提取 | B |
| `plan-ceo-review/SKILL.md.tmpl` → sections/ | 骨架提取 | B |
| `office-hours/SKILL.md.tmpl` → sections/ | 骨架提取 | B |
| `plan-eng-review/SKILL.md.tmpl` → sections/ | 骨架提取 | B |
| `plan-design-review/SKILL.md.tmpl` → sections/ | 骨架提取 | B |
| `gstack-upgrade/migrations/v2.0.0.0.sh`（新） | 自动重新生成+厂商安装警告 | B |
| `CHANGELOG.md` | v1.45.0.0条目（正常），v2.0.0.0条目（营销级附数字表） | A, B |
| `README.md` | v2.0.0.0横幅；"最轻量观点化技能包"定位 | B |
| `CONTRIBUTING.md` | 文档化sections/模式+回退程序 | B |

## 验证

**v1.45.0.0：**
1. `bun run gen:skill-docs` 成功无错误
2. `bun test` 通过（skill-validation、gen-skill-docs.test.ts、浏览集成、新skill-coverage-matrix.test.ts）
3. `bun run test:evals` 通过 — 所有新闸口评估绿灯；现有评估无回归
4. `bun run test:evals:periodic` 通过 — 所有新周期评估绿灯
5. Catalog系统提示大小测量：目标≤8K令牌（vs ~25K当前）。在PR正文中捕获before/after。
6. 每SKILL.md字节数总计：目标≤1.3MB（vs 2.1MB）。捕获在PR正文中。
7. 最重3个技能低于100KB。
8. 手动烟雾：在新Claude Code会话中调用 `/ship`、`/plan-ceo-review`、`/office-hours`；确认无行为丢失。保存transcript作为v1.45基线。

**v2.0.0.0：**
1. 所有v1.45检查通过
2. Section化技能：总计语料≤700KB；重量级骨架各≤30KB
3. `test/skill-e2e-ship-section-loading.test.ts`（新）：断言 `/ship` 按决策树读取预期sections
4. 金丝雀群体：在v2.0.0-rc.1进行1周dogfood并transcript记录；零Read遗漏标记required sections
5. 前5个工作流手动验证；transcript与v1.45基线比较
6. 迁移：在v1.45安装上的 `gstack-upgrade` 成功无提示重新生成；厂商安装警告出现一次
7. CHANGELOG数字表匹配实测现实
8. WARN模式孤儿检查：PR摘要显示孤儿列表；构建通过

## 跨模型协议已融入

来自Codex审查接受并融入的项目：

- #4 警告-then-失败的评估注释（第C阶段）
- #5 注释评论中的覆盖语义，不仅是路径
- #6 目录裁剪提前到第A阶段（原来埋在sections/之后）
- #9 cso得到解析器去重+目录裁剪（非整体豁免）
- #10 硬令牌预算定义+强制执行（A.6）
- #11 Forks/自定义兼容性文档化（迁移部分）
- #12 带金丝雀群体+手动前5工作流验证的推出策略（推出部分）

来自Codex审查被用户明确拒绝的项目（D9、D10）：
- #1 评估优先范围：用户保持全层覆盖。用结构评估指导（非输出内容）缓解孤儿/判断技能。
- #7 v2.0.0.0 vs v1.x：用户选择HYBRID。v1.45吸收低风险胜利；v2.0.0.0携带真正断裂的sections/变更。

用户接受Codex而非原始选择的项目：
- #8 迁移方法：用户从硬切（D7）移到较轻触（D11），一旦v1.45吸收了低风险工作。

## 实施任务

综合自审查发现。每个任务源自上述特定部分/发现。T1→T8落到v1.45.0.0；T9-T16落到v2.0.0.0。

- [ ] **T1 (P1, 人类：约3天 / CC：约7小时)** — 第0阶段/覆盖度矩阵 — 为缺乏覆盖度的20个技能编写闸口+周期评估
  - 发现来自：第0阶段部分
  - 文件：`test/skill-coverage-matrix.ts`, `test/skill-coverage-matrix.test.ts`, ~20个新 `test/skill-e2e-*.test.ts`, `test/helpers/touchfiles.ts`
  - 验证：`bun test test/skill-coverage-matrix.test.ts`和 `bun run test:evals` 两者通过新评估
- [ ] **T2 (P1, 人类：约1天 / CC：约1小时)** — A.1条件解析器注入 — 添加 `appliesTo` 门控
  - 发现来自：第A阶段部分，Codex #10（架构前的测量）
  - 文件：`scripts/resolvers/types.ts`, `scripts/gen-skill-docs.ts:444`, `scripts/resolvers/index.ts`
  - 验证：`bun run gen:skill-docs` 产生更小的SKILL.md文件；`bun test` 通过
- [ ] **T3 (P1, 人类：约半天 / CC：约30分钟)** — A.2 + A.3术语去重+精简模式gen时间压缩
  - 发现来自：第A阶段部分
  - 文件：`scripts/resolvers/preamble/generate-writing-style.ts`, `generate-completeness.ts`, `generate-confusion-protocol.ts`, `generate-context-health.ts`
  - 验证：术语列表不再出现内联在生成的SKILL.md中；`gstack-config set explain_level terse && bun run gen:skill-docs` 产生更短文件
- [ ] **T4 (P1, 人类：约1天 / CC：约2小时)** — A.4目录裁剪 — 单行技能描述；语音触发+主动段落移到JSON
  - 发现来自：Codex #6（最高杠杆），第A.4阶段
  - 文件：`scripts/skill-catalog.ts`（新），`scripts/proactive-suggestions.json`（新），每技能SKILL.md.tmpl前言用于单行描述字段
  - 验证：目录系统提示大小<8K令牌；语音触发调用仍然工作
- [ ] **T5 (P1, 人类：约半天 / CC：约30分钟)** — A.6 budget回归中的硬令牌预算
  - 发现来自：Codex #10
  - 文件：`test/skill-e2e-budget-regression.test.ts`
  - 验证：当人为膨胀的测试SKILL.md超过预算时budget回归失败
- [ ] **T6 (P1, 人类：约1天 / CC：约1小时)** — A.5 cso解析器去重+目录裁剪（非更广压缩）
  - 发现来自：Codex #9
  - 文件：`cso/SKILL.md.tmpl`（无结构变化，仅解析器门控审计）
  - 验证：cso SKILL.md大小下降20-30%；cso E2E评估仍然通过
- [ ] **T7 (P1, 人类：约1天 / CC：约1小时)** — 原子重新生成所有SKILL.md + 测量
  - 发现来自：第A阶段
  - 文件：所有 `*/SKILL.md` 重新生成
  - 验证：PR正文包含before/after语料大小，前10技能大小，目录大小；budget回归确认满足目标
- [ ] **T8 (P2, 人类：约半天 / CC：约30分钟)** — v1.45.0.0 CHANGELOG条目（正常语音；记录第0阶段+第A阶段已落地）
  - 发现来自：发布形状部分
  - 文件：`CHANGELOG.md`, `VERSION`
  - 验证：CHANGELOG整洁的lint；逆时序排列保留；条目覆盖diff

- [ ] **T9 (P1, 人类：约2天 / CC：约3小时)** — 第B.1阶段转换ship/为骨架+sections/
  - 发现来自：第B阶段部分
  - 文件：`ship/SKILL.md.tmpl` → 骨架；`ship/sections/manifest.json` + `ship/sections/*.md`
  - 验证：新 `test/skill-e2e-ship-section-loading.test.ts` 断言按决策树的预期Reads；现有ship评估通过；ship.md骨架<15KB
- [ ] **T10 (P1, 人类：约1天 / CC：约1小时)** — ship/ 金丝雀群体（在v2.0.0-rc.1进行1周dogfood）
  - 发现来自：推出策略部分，Codex #12
  - 文件：`test/helpers/transcript-section-logger.ts`（新）
  - 验证：dogfood transcript中对标记required sections的零Read遗漏
- [ ] **T11 (P1, 人类：约2天 / CC：约3小时)** — 第B.2阶段转换plan-ceo-review/（after ship/ proven）
  - 发现来自：第B阶段部分
  - 文件：`plan-ceo-review/SKILL.md.tmpl` + `plan-ceo-review/sections/`
  - 验证：section-loading测试绿灯；plan-ceo评估通过
- [ ] **T12 (P2, 人类：约3天 / CC：约4小时)** — 第B.3+B.4阶段转换office-hours/ + plan-eng-review/ + plan-design-review/
  - 发现来自：第B阶段部分
  - 文件：各自的 `SKILL.md.tmpl` + `sections/` 目录
  - 验证：section-loading测试绿灯；各自评估通过
- [ ] **T13 (P1, 人类：约1天 / CC：约1小时)** — 第C阶段评估注释+WARN模式CI孤儿检查
  - 发现来自：第C阶段部分，Codex #4 + #5
  - 文件：`scripts/gen-skill-docs.ts`（孤儿walker），所有 `sections/*.md`（带覆盖语义注释）
  - 验证：孤儿检查正确报告在PR摘要中；构建在WARN模式仍然通过
- [ ] **T14 (P1, 人类：约半天 / CC：约30分钟)** — `gstack-upgrade/migrations/v2.0.0.0.sh` 较轻触自动重新生成
  - 发现来自：迁移方法部分
  - 文件：`gstack-upgrade/migrations/v2.0.0.0.sh`
  - 验证：从v1.45安装升级产生干净的v2状态无提示；厂商安装获得单行警告
- [ ] **T15 (P1, 人类：约半天 / CC：约1小时)** — v2.0.0.0 营销级CHANGELOG附v1 vs v2数字表
  - 发现来自：D5，发布形状，Codex #7（记录真正断裂）
  - 文件：`CHANGELOG.md`, `VERSION`, `README.md`（v2横幅）
  - 验证：数字表匹配实测语料；配套说明记录具体断裂（sections/格式变化、首次调用延迟、厂商安装弃用）；定位过去式臃肿名声
- [ ] **T16 (P2, 人类：约1天 / CC：约1小时)** — 批量添加9个延迟TODOS到TODOS.md（gstack lite、gstack budget等）
  - 发现来自：TODOS.md更新部分
  - 文件：`TODOS.md`
  - 验证：TODOS格式匹配 `.claude/skills/review/TODOS-format.md`

## 失败模式注册表

| 代码路径 | 失败模式 | 已救援？ | 测试？ | 用户看到 | 记录 |
|---|---|---|---|---|---|
| gen-skill-docs.ts gate检查 | 解析器 `appliesTo` 抛出 | Y — try/catch记录+跳过解析器 | Y（test/gen-skill-docs.test.ts扩展） | "解析器X出错，已跳过"在构建输出中 | stderr |
| 运行时sections/读取 | section文件缺失 | Y — Agent回退到仅骨架行为 | Y（test/skill-e2e-ship-section-loading.test.ts） | Agent散文中的警告 | 会话transcript |
| CI孤儿walker | sections/*.md缺少评估注释 | WARN模式v2.0；FAIL v2.1+ | Y（test/skill-coverage-matrix.test.ts） | PR摘要列出孤儿 | PR注释 |
| 迁移脚本v2.0.0.0.sh | 受损安装上重新生成失败 | Y — 脚本中止，打印修复步骤 | Y（迁移测试） | 清晰错误+修复步骤 | ~/.gstack/analytics/migrations.jsonl |
| 目录单行生成器 | 前言中缺少单行描述的技能  | Y — gen-skill-docs响亮地构建失败 | Y（gen-skill-docs.test.ts扩展） | 构建错误 | stderr |
| 金丝雀section-Read记录器 | 重量级技能缺记录器 | Y — 静默跳过，差距在仪表板可见 | Y（transcript-logger测试） | 无直接；在金丝雀仪表板浮出水面 | ~/.gstack/analytics/section-reads.jsonl |

无关键缺口 — 每个失败模式都有救援、测试和可见性。

## 图表

系统架构（构建流水线）：
```
  CONFIG (~/.gstack/config.yaml)
     |
     v
  +-----------------+      +--------------------+
  | gen-skill-docs  | <--- | resolvers/*.ts     |
  | （带门控）       |      | （带appliesTo）     |
  +-----------------+      +--------------------+
     |
     v
  +--------------------------+
  | 每技能SKILL.md.tmpl       |
  | + sections/manifest.json |（仅重量级，v2）
  | + sections/*.md          |（仅重量级，v2）
  +--------------------------+
     |
     v
  +--------------------+         +--------------------------+
  | 生成SKILL.md       | <-----> | scripts/jargon-list.json |
  | （骨架用于         |         | （引用，非内联）          |
  |  重量级v2）        |         +--------------------------+
  +--------------------+
     |
     v
  +-------------------+      +----------------------+
  | 目录（系统        | <--- | proactive-suggestions|
  | 提示，每技能      |      | .json（按需加载，     |
  | 单行）            |      |  仅在需要时）         |
  +-------------------+      +----------------------+
```

Section-Read流（v2运行时）：
```
  用户 /ship
     |
     v
  +-----------------------+
  | ship/SKILL.md         |
  | （12-15KB骨架）        |
  | 读取：                 |
  |  - manifest.json      |
  |  - 决策树             |
  +-----------------------+
     |
     v  Agent遍历决策树，识别适用哪些sections
     |
     +-----> 读取 sections/version-bump.md   （如果是bump）
     +-----> 读取 sections/changelog.md      （如果是编写条目）
     +-----> 读取 sections/review-army.md    （如果是post-ship审查）
     +-----> ...仅适用的sections
     |
     v
  +-------------------------+
  | 技能末尾自检            |
  | "列出我读取的sections"  |
  +-------------------------+
     |
     v  金丝雀群体：transcript-section-logger比较
     |  实际Reads vs清单的required_for声明
     |  遗漏警报
```

## 陈旧图表审计

此计划影响的CLAUDE.md/ARCHITECTURE.md中的ASCII图：

| 图表 | 文件 | 在v2后仍然准确吗？ |
|---|---|---|
| Sidebar消息流 | `docs/designs/SIDEBAR_MESSAGE_FLOW.md` | YES（无关子系统） |
| 双监听器隧道架构 | `ARCHITECTURE.md` | YES（无关） |
| 服务器出口处的Unicode消毒 | `ARCHITECTURE.md` | YES（无关） |
| （无技能构建流水线） | — | 上面新图是新创建的，不是更新 |

无陈旧图表需要修复。

## 完成摘要

```
+====================================================================+
|            大型计划审查 — 完成摘要                                  |
+====================================================================+
| 模式选择             | 范围扩展                                    |
| 系统审计             | 臃肿在外部文档中；之前设计文档                  |
|                      | 无关；budget回归基础设施已存在                  |
| Step 0               | 扩展+方法C+评估优先+                         |
|                      | 混合v1.45/v2.0分裂+较轻轻迁移                  |
| 第1部分（架构）       | 1发现 — 沉默丢失风险，6层缓解                   |
| 第2部分（错误）       | 6种失败模式映射，0关键缺口                      |
| 第3部分（安全）       | cso定向去重（Codex #9吸收）                   |
| 第4部分（数据/UX）    | v1→v2肌肉记忆已警告，厂商已记录                |
| 第5部分（质量）       | ~150 LOC附加，机械提取                         |
| 第6部分（测试）        | 第0阶段就是测试计划                           |
| 第7部分（性能）        | <2×构建时间；v2 +200-500ms首次调用             |
| 第8部分（观察）        | budget回归+金丝雀+migrations.log              |
| 第9部分（部署）        | 2发布分裂+警告-then-fail+回退                 |
| 第10部分（未来）       | 可逆性3/5；sections/成为模板                 |
| 第11部分（设计）       | README横幅+数字表                           |
+--------------------------------------------------------------------+
| 不在范围内           | 已编写（9项延迟）                              |
| 已存在的             | 已编写（9个重用点）                            |
| 梦想状态增量         | 已编写（今天/v1.45/v2.0）                      |
| 错误/救援注册表       | 6种模式，0关键缺口                             |
| 失败模式             | 覆盖在注册表中                                 |
| TODOS.md更新         | 9项，合并后批量添加                             |
| 范围建议             | 3个浮出水面，1个接受（发布定位）                  |
| CEO计划              | 这个计划就是CEO计划                           |
| 外部声音             | 运行过（codex）；3个张力浮出水面                  |
| Lake Score           | 11/11推荐选择了完整选项                         |
| 图表产出             | 2（构建流水线，section-Read流）                  |
| 陈旧图表发现         | 0                                              |
| 未解决的决定         | 0                                              |
+================================================================----+
```

## 工程审查补充（来自/plan-eng-review会话）

### 已锁定的架构决策

- **D1（清单格式）：** `sections/manifest.json` 是每个重量级技能的结构化注册表（JSON，对gen-skill-docs CI检查机器可读）。SKILL.md骨架是markdown标题+命令式散文块（"停止。如果X，读取 `sections/Y.md`"）。匹配Anthropic文档化的 `references/` 风格。无自创DSL。
- **D2（漂移控制）：** `sections/*.md.tmpl` 是真实来源；`sections/*.md` 是生成的。gen-skill-docs遍历 `<skill>/sections/*.tmpl` 并使用与SKILL.md相同的解析器流水线写入 `<skill>/sections/*.md`。成本：`scripts/gen-skill-docs.ts` 中约30 LOC。消除 `test/ship-version-sync.test.ts` 已遭受的漂移类（TODOS:1120）。
- **D3（CI成本上限）：** `EVALS_BUDGET_HARD_CAP=$30` env var由 `test/skill-e2e-budget-regression.test.ts` 强制执行；单次运行超过则构建失败。Section-loading测试（第B阶段）使用minimal-bash夹具（约$0.30每个），因为它们断言**结构行为**（是否读取了正确文件？）非输出质量。

### 相邻TODOS浮出水面（信息性，非阻塞）

- **TODOS:161** — 为浏览器技能计划的"会话开始时的解析器注入"（P2）。与此计划的 `appliesTo` 谓词有架构重叠。决策：暂时保持独立 — 浏览器技能解析器注入是运行时（会话开始主机名匹配）；我们的 `appliesTo` 是构建时（gen-skill-docs.ts）。不同生命周期，不同关注点。仅在浏览器技能工作需要相同谓词形状时重新讨论。
- **TODOS:1120** — `test/ship-version-sync.test.ts` 重新实现ship/SKILL.md.tmpl Step 12 bash。D2（sections/*.md.tmpl流水线）是结构性修复。第B阶段工作使此TODO过时；当ship/提取落地时标记为已解决。
- **TODOS:1136** — ship/SKILL.md.tmpl Step 12第409行中的 `git show` 回退。第B阶段涉及此；将 `git rev-parse --verify` 修复捆绑到version-bump section提取中。

### 测试计划构件

测试计划写入 `~/.gstack/projects/garrytan-gstack/garrytan-garrytan-slim-skill-tokens-eng-review-test-plan-<timestamp>.md`。`/qa` 和 `/qa-only` 以此作为主测试输入。覆盖：每阶段测试覆盖度目标、section-loading测试的夹具设计、CI预算强制执行检查、迁移往返测试。

### 失败模式补充

从§失败模式添加（已完整；新行）：

| 代码路径 | 失败模式 | 已救援？ | 测试？ | 用户看到 | 记录 |
|---|---|---|---|---|---|
| sections/*.md.tmpl生成器 | 模板引用缺失解析器 | Y — gen-skill-docs响亮地构建失败 | Y（gen-skill-docs.test.ts扩展） | 构建错误 | stderr |
| 清单↔文件系统一致性 | 清单引用不存在的section文件 | Y — CI检查失败 | Y（新 `test/section-manifest-consistency.test.ts`） | 构建错误 | PR摘要 |
| 清单↔文件系统一致性 | section文件存在但不在清单中（孤儿） | WARN v2.0；FAIL v2.1+ | Y（相同测试） | PR摘要 | PR注释 |
| 预算上限超出 | 单个测试或聚合超过 `EVALS_BUDGET_HARD_CAP` | Y — CI失败 | Y（budget回归扩展） | 带成本明细的构建错误 | stderr |

仍然0关键缺口。所有新失败模式有救援+测试+可见性。

### 执行排序（顺序v1.45，集成分支v2.0）

v1.45在单个分支中**顺序**运行，T1→T8。在codex的第二遍批评标记后，并行化地图被重新考虑。T2（gen-skill-docs.ts TemplateContext变化）和T4（catalog前言添加）几乎肯定在编译时互相触及 — 两个分支各自通过，在集成时失败。顺序落地更干净，避免三方合并惊喜。AI压缩使顺序的墙壁时钟成本可接受。

| 步骤 | 触摸模块 | 依赖 |
|---|---|---|
| T1第0阶段评估（~20文件） | `test/skill-e2e-*.test.ts`, `test/skill-coverage-matrix.ts`, `test/helpers/touchfiles.ts` | — |
| T2条件解析器门控 | `scripts/gen-skill-docs.ts`, `scripts/resolvers/types.ts`, `scripts/resolvers/index.ts` | T1 |
| T3术语去重+精简压缩 | `scripts/resolvers/preamble/*` | T2 |
| T4目录裁剪 | `scripts/skill-catalog.ts`, `scripts/proactive-suggestions.json`, 所有SKILL.md.tmpl前言 | T2 |
| T5硬令牌预算+覆盖路径 | `test/skill-e2e-budget-regression.test.ts`（每套件上限+`EVALS_BUDGET_OVERRIDE_REASON`） | T1 |
| T6 cso定向去重 | `cso/SKILL.md.tmpl` | T2, T3 |
| T7原子重新生成所有SKILL.md | 所有 `*/SKILL.md` | T1-T6 |
| T8 v1.45 CHANGELOG | `CHANGELOG.md`, `VERSION` | T7 |
| **— v1.45.0.0发货边界 —** | | |
| T9 ship/ sections/提取 | `ship/SKILL.md.tmpl`, `ship/sections/*`, gen-skill-docs（sections流水线+TemplateContext契约） | T8 + sections-流水线（T2/D2） |
| T10 ship/ 金丝雀群体 | `test/helpers/transcript-section-logger.ts` | T9 |
| T11 plan-ceo-review sections/ | `plan-ceo-review/SKILL.md.tmpl` + sections | T10（ship/已证明） |
| T12 office-hours + plan-eng + plan-design sections/ | 各自目录 | T11 |
| T13第C阶段评估注释+3层孤儿检查 | gen-skill-docs.ts孤儿walker, 所有sections/*.md | T9-T12 |
| T14迁移脚本 | `gstack-upgrade/migrations/v2.0.0.0.sh` | T13 |
| T15 v2.0.0.0 CHANGELOG + README横幅 | `CHANGELOG.md`, `README.md`, `VERSION` | T14 |
| T16 TODOS批量添加 | `TODOS.md` | — 随时 |

**执行推荐：** v1.45（T1→T8）和v2.0（T9→T15）都用单工作树顺序。T16随时落地。CC加速来自每步压缩（每步约1小时vs人类天），非并行分支。

## Codex咨询补充（第二遍，post eng-review）

### 大教堂对等评估套件（第0阶段新增，扩展至"11"）

用户说"像11一样做，不只是10。最大化还要加码。"最大化范围：

- **所有31个技能**获得黄金基线transcript（不仅是前5）
- **每技能多夹具**（3-5个代表性调用路径每个）
- **定量+定性评分：** LLM-as-judge相似度分数（1-10）AND transcript-diff高光（添加/删除sections，缺失细微差别）
- **令牌效率比测量：** quality-per-token = judge_score / tokens_consumed（强制v2可测量地更高效，不只是更小）
- **"质量预算"与"令牌预算"并列：** 两者在CI中强制执行。一个压缩到一半大小但从9/10质量降到6/10的v2技能会失败门控。
- **并排PR注释：** 每个触及重量级技能的PR在PR摘要中自动发布v1.45基线与当前的并行比较
- **公开基准页面：** `gstack.benchmarks.md`（新），持续更新。可引用："v2平均对等分数：9.2/10，平均令牌减少：67%。"
- **持续监控：** 对main每周运行对等套件；如果任何技能漂流低于基线（Discord webhook或类似）
- **基线捕获脚本，** `test/helpers/capture-parity-baseline.ts` — 在v1.44 HEAD运行一次，锁定黄金transcript在第A阶段任何工作开始之前

工作量：人类约3-4天 / CC约6-8小时一次性 + ~$30/周持续用于持续监控。成本合理 — 这是唯一捕获section-loading和预算测试都错过的"看上去绿色，感觉更差"的沉默回归机制。在T1之前添加新任务T0a（基线捕获）和T0b（对等评估工具）。

### 吸收的codex咨询改进（无需进一步用户决策）

1. **sections流水线的TemplateContext契约（codex D2批评）：** T9需要显式规范。Section生成使用与SKILL.md生成相同的 `TemplateContext` — 相同 `skillName`，相同主机抑制，相同 `explainLevel`，相同层门控。在代码注释中记录+由 `test/template-context-parity.test.ts`（新）断言。
2. **3层孤儿分类（codex孤儿语义批评）：** CI检查（T13）区分：
   - **生成孤儿**（`sections/foo.md` 存在，无 `sections/foo.md.tmpl`）→ 立即失败，每个发布
   - **清单孤儿**（`sections/foo.md.tmpl` 存在，不在 `manifest.json` 中）→ 在v2.0警告，v2.1+失败
   - **手工编辑的生成文件**（`sections/foo.md` 与重新生成的输出偏离）→ 立即失败，带"此文件是生成的，请编辑 `.tmpl`"消息
3. **预算上限覆盖路径（codex D3批评）：** `EVALS_BUDGET_HARD_CAP=$30` 成为默认；每套件上限通过 `EVALS_BUDGET_HARD_CAP_GATE=$25`，`EVALS_BUDGET_HARD_CAP_PERIODIC=$70`；覆盖路径 `EVALS_BUDGET_OVERRIDE_REASON="<text>"` env需要超过上限（CI打印原因在构建输出用于审计跟踪）；通过现有分析（`~/.gstack/analytics/skill-usage.jsonl`聚合器）的每日org级花费警报。
4. **清单作为被动数据（codex D1批评）：** `manifest.json` 字段仅为ID、文件路径和人类可读触发文本。无 `applies_when` 谓词。技能骨架的决策树散文是唯一决定"何时读取X"的地方。避免发明第四种条件语言与层门控+ `appliesTo` + `requiredReads` 并列。
5. **T7作为集成分支流（codex并行化批评，现在已经过时而顺序）：** 顺序执行使T7只是"在单个v1.45分支内的原子重新生成。"集成分支舞蹈不需要。批评意图（无三方合并惊喜）通过折叠到顺序来尊重。

### 新失败模式（注册表补充）

| 代码路径 | 失败模式 | 已救援？ | 测试？ | 用户看到 | 记录 |
|---|---|---|---|---|---|
| Sections流水线TemplateContext | sections以不同ctx生成（如错误的skillName） | Y — 对等测试失败 | Y（`test/template-context-parity.test.ts`） | 构建错误 | stderr |
| 手工编辑的生成section | 用户直接编辑 `sections/foo.md` 而非 `.tmpl` | Y — CI失败带显式消息 | Y（孤儿检查3层分类） | "此文件是生成，请编辑 `.tmpl`" | PR摘要 |
| 质量预算超出 | v2技能压缩但在LLM-judge对等中下降>2分 | Y — CI失败 | Y（对等评估套件） | "v2 X.md从v1.45基线的9.2降到6.4" | PR注释带diff |
| 预算上限覆盖审计 | EVALS_BUDGET_OVERRIDE_REASON使用 | N（有意逃生阀） | Y（审计日志测试） | 原因打印在CI输出，记录到spend-audit jsonl | analytics/spend-overrides.jsonl |
| Main上的对等基线漂流 | 每周持续监控检测回归 | Y — Discord警报名片 | Y（持续监控测试）| 团队频道中的警报 | analytics/parity-drift.jsonl |

仍然0关键缺口。

## v2发布文案规范（来自/plan-devex-review）

这些草稿成为v2.0.0.0发布语音的真实来源。T15按原样实施它们（除非ship时间研讨产生可测量的更好方案，那么同步更新计划和实施）。

### JUST_UPGRADED通知（Persona A — 现有用户升级）

由 `gstack-update-check` 显示 `JUST_UPGRADED v1.x v2.0.0.0` 触发。用persona-A感知文案替换通用的v1"运行gstack v{to}（刚刚更新！）"，既命名感知的速度胜利又表示"你的肌肉记忆仍然工作。"

```
运行gstack v2.0.0.0（刚刚更新！）— 你的会话现在轻了约67%。
重量级技能仅加载它们需要的sections；目录降到了每技能
单行。一切仍然同样工作 — 你的 /ship、/qa、/review 命令
没变。运行 `/gstack-upgrade --explain-v2` 获取完整迁移
故事，或者继续工作。
```

语音规则遵守：领先胜利（"轻67%"）；具体数字；保证工作流不变（"一切仍然同样工作"）；逃生口（`--explain-v2`）。无破折号。瞄准5秒阅读。

实施：更新 `~/.claude/skills/gstack/gstack-upgrade/SKILL.md.tmpl` 内联升级流带v2感知消息；现有 `JUST_UPGRADED <from> <to>` 检测在skill前言中触发。

### CHANGELOG数字表（Persona A的神奇时刻 + Persona B的评估证据）

列入CHANGELOG.md的 `## [v2.0.0.0]` 条目，紧跟标题。比较v1.44实测（基线由 `test/helpers/capture-parity-baseline.ts` 在第A阶段开始之前捕获）vs v2.0.0.0测量。数字必须真实，非估计；在T15期间替换占位符。

| 指标 | v1.44.1（基线） | v2.0.0.0（测量） | Δ |
|---|---|---|---|
| 每SKILL.md总计 | 2.1 MB | ~700 KB | **−67%** |
| ship.md（最重） | 164 KB | ~15 KB骨架 + 5×~5 KB sections | **−76%首次读取** |
| plan-ceo-review.md | 131 KB | ~12 KB骨架 + sections按需 | **−68%首次读取** |
| office-hours.md | 111 KB | ~10 KB骨架 + sections按需 | **−71%首次读取** |
| 目录令牌（始终加载系统提示） | ~25K令牌 | ~6K令牌 | **−76%** |
| 每调用令牌（典型/ship会话） | ~41K | ~14K骨架 + 按需sections | **~60%下降** |
| 评估覆盖度（有E2E保护的技能） | ~16 of 31 | **31 of 31 + 对等基线** | 质量门控启用 |
| 对等分数vs v1.44基线（LLM judge，所有31技能） | — | **≥9.0/10最低** | （CI强制执行；见对等评估套件） |

表下方，一段gstack语音："v1是最大最重的观点化技能包。v2是最轻量的。压缩不是免费的 — 每个技能都随附闸口级和周期级E2E评估，以及持续的对等监控捕获沉默质量回归。上面的数字是针对 `test/helpers/parity-baseline-v1.44.1/` 实测并由 `bun run eval:parity` 复现的。"

### README v2横幅

位置：README.md顶部，现有Karpathy引文正上方，在"当我听到Karpathy说这个..."之前。在发布后停留60天，然后在Quick start区域折叠成一行"v2发布于2026年5月"条目。

```markdown
> **gstack v2.0.0.0 — 最轻量的观点化技能包（2026年5月）**
>
> 重量级技能现在仅加载它们需要的sections。总计SKILL.md
> 语料从2.1 MB降至~700 KB。每个技能都随附E2E评估
> 保护和持续的对等监控与v1.44基线对比。
> 参见 [v2.0.0.0发布说明](CHANGELOG.md) 了解每技能的数字和
> 迁移故事。现有用户：`/gstack-upgrade` 自动重新生成。
```

语音规则遵守：领先定位（"最轻量的观点化技能包"）；具体数字（2.1 MB → 700 KB）；严谨证明（评估保护+对等监控）；迁移路径明确。无破折号。瞄准10秒阅读。

### 实施说明（用于T15）

- 在实际v1.44基线数字锁到 `test/helpers/parity-baseline-v1.44.1/` **在第A阶段重新生成开始之前**。"v1 vs v2"增量仅在v1.44以相同单位（令牌计数通过tiktoken，字节计数通过wc -c，评估覆盖度通过test/skill-coverage-matrix.ts）测量时准确。
- 如果测量的v2数字不如上述草稿令人印象深刻（如ship.md最终25 KB而非15 KB），更新草稿反映现实。绝不编造数字；营销级发货时刻在读者找到一个能用 `wc -c` 反驳的数字时立即死亡。
- JUST_UPGRADED通知通过现有 `gstack-upgrade`检测自动触发 — 无需新机制。
- README横幅放置在现有Karpathy引文上方是有意的：persona B（新评估者在Karpathy框架之前看到v2胜利，锚定"这是2026年5月最新的gstack"）。

## GSTACK审查报告

| 审查 | 触发 | 为什么 | 运行 | 状态 | 发现 |
|---|---|---|---|---|---|
| CEO审查 | `/plan-ceo-review` | 范围和策略 | 1 | 清理 | SCOPE_EXPANSION模式；3范围建议（1接受：v2发布定位；2延迟：gstack lite, gstack budget）；11/11部分审查；0关键缺口 |
| Codex审查 | `/codex review` | 独立第二意见（外部声音） | 1 | issues_found | 12项挑战浮出；7项吸收进计划（#4,#5,#6,#9,#10,#11,#12）；3项浮为用户决定（#1用户保持原始选择,#7混合分裂采用,#8用户接受codex） |
| 工程审查 | `/plan-eng-review` | 架构和测试（必需） | 1 | 清理 | 3架构决策锁定（D1 JSON清单, D2 sections/*.md.tmpl流水线, D3 CI成本上限）；4新失败模式添加（全救援+测试）；测试计划构件写入；并行化地图产生（v1.45中3条并行，v2.0中顺序）；0关键缺口；0未解决决定 |
| Codex咨询（第二遍） | `/codex`（咨询工程审查补充） | D1/D2/D3+并行化独立挑战 | 1 | issues_found | 工程审查补充的7个附加发现；5吸收（TemplateContext契约, 3层孤儿分类, 预算上限覆盖路径, 清单作为被动数据非谓词, T7作为集成流过时而顺序）；2浮为用户决定（注意架构风险→大教堂对等评估套件以"11"添加；并行化按codex批评折叠为v1.45顺序） |
| 设计审查 | `/plan-design-review` | UI/UX缺口 | 0 | — | 无需（无显式UI范围；仅README/CHANGELOG） |
| DX审查 | `/plan-devex-review` | 开发者体验差距 | 1 | CLEAN | DX POLISH模式；产品类型 = Claude Code Skill；2 persona同等跟踪（现有用户升级者+新用户评估者）；初始7.9/10 → 9.0/10在发布文案规范补充到计划之后（JUST_UPGRADED通知、CHANGELOG数字表、README v2横幅全部起草为T15交付件）；所有8遍评估；技能DX清单通过 |

**CODEX：** 第一遍（CEO）：12发现，7吸收，3个跨模型用户决定，2融入任务。第二遍（post eng-review）：7个关于新D1/D2/D3补充的发现，5吸收，2用户决定。两遍作为审计轨迹保留。19个codex发现总计 → 12无摩擦吸收，5个用户决定跨两遍，2个生活质量改进融入任务。DX审查跳过了新鲜codex遍（3个先前两遍已覆盖结构盲点；剩余DX工作是文案工艺，其中codex增加的价值少于用户品味）。

**跨模型：** 强烈同意（a）分段（目录裁剪早，sections/晚），（b）测量优先（硬令牌预算+覆盖审计跟踪），（c）forks/推出策略缺口。所有遍解决张力：评估优先范围（用户保持），v2 vs v1.x（HYBRID采用），迁移重量（较轻触采用），并行化（用户接受codex的顺序批评），注意架构风险（用户扩展范围到大教堂对等评估套件覆盖所有31技能带质量预算与令牌预算），发布文案构件（用户起草所有三个在计划中vs延迟到T15实施）。

**未解决：** 所有5个审查中0未解决决定。

**裁定：** CEO + ENG + CODEX×2 + DX清理 — 准备实施。混合v1.45/v2.0分裂降低臃肿名声修复的风险；sections/*.md.tmpl流水线（D2）防止漂移；带覆盖审计的CI成本上限（D3+codex吸收改进）防止逃跑评估支出；大教堂对等评估套件（codex第二遍）捕获section-loading+预算测试单独会错过的沉默注意架构回归；顺序v1.5执行（codex吸收）以墙壁时钟换取集成安全；v2发布文案规范（DX审查）使营销级发货时刻落在两个persona A（现有升级者）和persona B（新评估者）上。计划现已可执行。
