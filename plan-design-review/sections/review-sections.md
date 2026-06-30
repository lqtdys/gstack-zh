<!-- AUTO-GENERATED from review-sections.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -- -->
## Review Sections (7 passes, after scope is agreed) — 审查章节（7轮审查，在范围达成一致后）

**Anti-skip rule（反跳过规则）:** Never condense, abbreviate, or skip any review pass (1-7) regardless of plan type (strategy, spec, code, infra). Every pass in this skill exists for a reason. "This is a strategy doc so design passes don't apply" is always wrong — design gaps are where implementation breaks down. If a pass genuinely has zero findings, say "No issues found" and move on — but you must evaluate it.

> **反跳过规则：** 无论计划类型（策略、规范、代码、基础设施）如何，绝不得压缩、缩写或跳过任何一轮审查（1-7轮）。本技能中的每一轮审查都有其存在的理由。"这是一份策略文档，所以设计审查轮次不适用"这种说法永远是错误的——设计缺陷正是实现崩溃的地方。如果某轮审查确实没有任何发现，说"未发现问题"然后继续——但你必须对该轮进行评估。

**Anti-shortcut clause（反捷径条款）:** The plan file is the OUTPUT of the interactive review, not a substitute for it. Writing every finding into one plan write and calling ExitPlanMode without firing AskUserQuestion is the precise failure mode of the May 2026 transcript bug — the model explored, found issues, and dumped them into a deliverable rather than walking the user through them. If you have ANY non-trivial finding in any review section, the path from finding to ExitPlanMode goes THROUGH AskUserQuestion. Zero findings in every section is the only path to ExitPlanMode that bypasses AskUserQuestion. If you find yourself wanting to write a plan with findings before asking, stop and call AskUserQuestion now — that's the bug, recognize it.

> **反捷径条款：** 计划文件是交互式审查的输出，而不是它的替代品。将所有发现一次性写入计划文件然后在不触发AskUserQuestion的情况下调用ExitPlanMode，正是2026年5月转录bug的精确失败模式——模型探索了、发现了问题，却将它们倾倒在交付物中，而不是引导用户审查这些问题。如果你在任何一个审查章节中有任何非实质性发现，从发现到ExitPlanMode的路径必须经过AskUserQuestion。所有章节零发现才是唯一可以绕过AskUserQuestion直接调用ExitPlanMode的路径。如果你发现自己想在提问之前先写一个有发现的计划，停下来，现在就调用AskUserQuestion——那就是bug，识别它。

## Prior Learnings（先前学习）

Search for relevant learnings from previous sessions:
搜索先前会话中的相关学习：

```bash
_CROSS_PROJ=$(~/.claude/skills/gstack/bin/gstack-config get cross_project_learnings 2>/dev/null || echo "unset")
echo "CROSS_PROJECT: $_CROSS_PROJ"
if [ "$_CROSS_PROJ" = "true" ]; then
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 --cross-project 2>/dev/null || true
else
  ~/.claude/skills/gstack/bin/gstack-learnings-search --limit 10 2>/dev/null || true
fi
```

If `CROSS_PROJECT` is `unset`（如果CROSS_PROJECT为unset（首次使用）: Use AskUserQuestion:
使用AskUserQuestion：

> gstack can search learnings from your other projects on this machine to find
> patterns that might apply here. This stays local (no data leaves your machine).
> Recommended for solo developers. Skip if you work on multiple client codebases
> where cross-contamination would be a concern.

> gstack可以搜索你本机其他项目的学习记录，找出可能适用于当前项目的模式。数据完全保留在本地（不会离开你的机器）。推荐给独立开发者。如果你在多个客户代码库上工作，且存在交叉污染的风险，请跳过此选项。

Options（选项）:
- A) Enable cross-project learnings (recommended) — 启用跨项目学习（推荐）
- B) Keep learnings project-scoped only — 仅在项目范围内保留学习

If A: run `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings true`
如果A：运行 `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings true`
If B: run `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings false`
如果B：运行 `~/.claude/skills/gstack/bin/gstack-config set cross_project_learnings false`

Then re-run the search with the appropriate flag.
然后使用适当的标志重新运行搜索。

If learnings are found, incorporate them into your analysis. When a review finding matches a past learning, display:
如果找到学习记录，将其纳入你的分析。当审查发现与先前的学习匹配时，展示：

**"Prior learning applied: [key] (confidence N/10, from [date])"**
**"已应用先前学习：[键]（置信度 N/10，来自 [日期]）"**

This makes the compounding visible. The user should see that gstack is getting smarter on their codebase over time.
这让复利效应可见。用户应该能看到gstack随着时间推移在他們的代码库上变得更聪明。

### Pass 1: Information Architecture — 第1轮：信息架构
Rate 0-10（评分0-10）: Does the plan define what the user sees first, second, third? 计划是否定义了用户看到的内容的第一、第二、第三顺序？
FIX TO 10（修复至10分）: Add information hierarchy to the plan. Include ASCII diagram of screen/page structure and navigation flow. Apply "constraint worship" — if you can only show 3 things, which 3?
修复至10分：在计划中添加信息层次结构。包含屏幕/页面结构和导航流程的ASCII图。应用"约束崇拜"——如果你只能展示3样东西，展示哪3样？
**STOP.** AskUserQuestion once per issue. Do NOT batch. Recommend + WHY. If no issues, say so and move on. Do NOT proceed until user responds.
**停止。** 每个问题调用一次AskUserQuestion。不要批量处理。提供建议+原因。如果没有问题，说明然后继续。在用户响应之前不要继续。

### Pass 2: Interaction State Coverage — 第2轮：交互状态覆盖
Rate 0-10（评分0-10）: Does the plan specify loading, empty, error, success, partial states? 计划是否指定了加载、空、错误、成功、部分状态？
FIX TO 10（修复至10分）: Add interaction state table to the plan:
修复至10分：在计划中添加交互状态表：
```
  FEATURE              | LOADING | EMPTY | ERROR | SUCCESS | PARTIAL
  ---------------------|---------|-------|-------|---------|--------
  [each UI feature]    | [spec]  | [spec]| [spec]| [spec]  | [spec]
```
For each state: describe what the user SEES, not backend behavior.
对于每个状态：描述用户看到的内容，而不是后端行为。
Empty states are features — specify warmth, primary action, context.
空状态是功能的一部分——指定温度感、主要操作、上下文。
**STOP.** AskUserQuestion once per issue. Do NOT batch. Recommend + WHY.
**停止。** 每个问题调用一次AskUserQuestion。不要批量处理。提供建议+原因。

### Pass 3: User Journey & Emotional Arc — 第3轮：用户旅程与情感弧线
Rate 0-10（评分0-10）: Does the plan consider the user's emotional experience? 计划是否考虑了用户的情感体验？
FIX TO 10（修复至10分）: Add user journey storyboard:
修复至10分：添加用户旅程故事板：
```
  STEP | USER DOES        | USER FEELS      | PLAN SPECIFIES?
  -----|------------------|-----------------|----------------
  1    | Lands on page    | [what emotion?] | [what supports it?]
  ...
```
Apply time-horizon design: 5-sec visceral, 5-min behavioral, 5-year reflective.
应用时间范围设计：5秒的本能层、5分钟的行为层、5年的反思层。
**STOP.** AskUserQuestion once per issue. Do NOT batch. Recommend + WHY.
**停止。** 每个问题调用一次AskUserQuestion。不要批量处理。提供建议+原因。

### Pass 4: AI Slop Risk — 第4轮：AI劣质内容风险
Rate 0-10（评分0-10）: Does the plan describe specific, intentional UI — or generic patterns? 计划描述的是具体的、有意的UI还是通用模式？
FIX TO 10（修复至10分）: Rewrite vague UI descriptions with specific alternatives.
修复至10分：用具体的选择重写模糊的UI描述。

### Design Hard Rules（设计硬性规则）

**Classifier — determine rule set before evaluating（分类器——在评估之前确定规则集）:**
- **MARKETING/LANDING PAGE**（营销/落地页） (hero-driven, brand-forward, conversion-focused — 以英雄区为驱动、以品牌为导向、以转化为中心) → apply Landing Page Rules — 应用落地页规则
- **APP UI**（应用UI） (workspace-driven, data-dense, task-focused: dashboards, admin, settings — 以工作区为驱动、数据密集、以任务为中心) → apply App UI Rules — 应用应用UI规则
- **HYBRID**（混合） (marketing shell with app-like sections) → apply Landing Page Rules to hero/marketing sections, App UI Rules to functional sections — 落地页规则应用于英雄区/营销部分，应用UI规则应用于功能部分

**Hard rejection criteria** (instant-fail patterns — flag if ANY apply):
**硬性拒绝标准**（即时失败模式——如果适用任何一项即标记）:
1. Generic SaaS card grid as first impression — 通用SaaS卡片网格作为第一印象
2. Beautiful image with weak brand — 配弱品牌的精美图片
3. Strong headline with no clear action — 没有清晰行动的强力标题
4. Busy imagery behind text — 文字后面的杂乱图像
5. Sections repeating same mood statement — 重复相同情绪陈述的段落
6. Carousel with no narrative purpose — 没有叙事目的轮播
7. App UI made of stacked cards instead of layout — 由堆叠卡片而非布局构成的应用UI

**Litmus checks** (answer YES/NO for each — used for cross-model consensus scoring):
**石蕊测试**（逐个回答YES/NO——用于跨模型共识评分）:
1. Brand/product unmistakable in first screen? — 品牌/产品在第一屏是否一目了然？
2. One strong visual anchor present? — 是否有一个强有力的视觉锚点？
3. Page understandable by scanning headlines only? — 仅扫描标题就能理解页面？
4. Each section has one job? — 每个部分是否只有一个任务？
5. Are cards actually necessary? — 卡片是否真的必要？
6. Does motion improve hierarchy or atmosphere? — 动效是否改善了层次或氛围？
7. Would design feel premium with all decorative shadows removed? — 去掉所有装饰性阴影后设计是否仍感觉高级？

**Landing page rules** (apply when classifier = MARKETING/LANDING):
**落地页规则**（分类器=营销/落地页时应用）:
- First viewport reads as one composition, not a dashboard — 首屏呈现为一个整体构图，而非仪表盘
- Brand-first hierarchy: brand > headline > body > CTA — 品牌优先层次：品牌 > 标题 > 正文 > CTA
- Typography: expressive, purposeful — no default stacks (Inter, Roboto, Arial, system) — 排版：有表现力、有目的性——不使用默认字体栈（Inter、Roboto、Arial、system）
- No flat single-color backgrounds — use gradients, images, subtle patterns — 不使用平面单色背景——使用渐变、图像、微妙图案
- Hero: full-bleed, edge-to-edge, no inset/tiled/rounded variants — 英雄区：全出血、边到边，不使用内嵌/平铺/圆角变体
- Hero budget: brand, one headline, one supporting sentence, one CTA group, one image — 英雄区预算：品牌、一个标题、一个支持句、一个CTA组、一张图片
- No cards in hero. Cards only when card IS the interaction — 英雄区不放卡片。仅当卡片本身就是交互时才使用卡片
- One job per section: one purpose, one headline, one short supporting sentence — 每个部分一个任务：一个目的、一个标题、一个简短支持句
- Motion: 2-3 intentional motions minimum (entrance, scroll-linked, hover/reveal) — 动效：至少2-3个有意图的动效（入场、滚动关联、悬停/揭示）
- Color: define CSS variables, avoid purple-on-white defaults, one accent color default — 颜色：定义CSS变量，避免紫白默认方案，默认一种强调色
- Copy: product language not design commentary. "If deleting 30% improves it, keep deleting" — 文案：用产品语言而非设计评论。"如果删除30%更好，就继续删"
- Beautiful defaults: composition-first, brand as loudest text, two typefaces max, cardless by default, first viewport as poster not document — 优美的默认样式：构图优先、品牌作为最醒目的文字、最多两种字体、默认无卡片、首屏像海报而非文档

**App UI rules** (apply when classifier = APP UI):
**应用UI规则**（分类器=应用UI时应用）:
- Calm surface hierarchy, strong typography, few colors — 平静的表面层次、强有力的排版、少量颜色
- Dense but readable, minimal chrome — 密集但可读、最少装饰
- Organize: primary workspace, navigation, secondary context, one accent — 组织：主工作区、导航、次要上下文、一个强调色
- Avoid: dashboard-card mosaics, thick borders, decorative gradients, ornamental icons — 避免：仪表盘卡片马赛克、粗边框、装饰性渐变、装饰性图标
- Copy: utility language — orientation, status, action. Not mood/brand/aspiration — 文案：实用语言——方向、状态、行动。不是情绪/品牌/愿景
- Cards only when card IS the interaction — 仅当卡片本身就是交互时才使用卡片
- Section headings state what area is or what user can do ("Selected KPIs", "Plan status") — 部分标题说明该区域是什么或用户可以做什么（"已选KPI"、"计划状态"）

**Universal rules** (apply to ALL types):
**通用规则**（应用于所有类型）:
- Define CSS variables for color system — 为颜色系统定义CSS变量
- No default font stacks (Inter, Roboto, Arial, system) — 不使用默认字体栈（Inter、Roboto、Arial、system）
- One job per section — 每个部分一个任务
- "If deleting 30% of the copy improves it, keep deleting" — "如果删除30%的文案更好，就继续删"
- Cards earn their existence — no decorative card grids — 卡片的存在要有意义——不使用装饰性卡片网格
- NEVER use small, low-contrast type (body text < 16px or contrast ratio < 4.5:1 on body text) — **绝不**使用小字号、低对比度文字（正文小于16px或对比度比<4.5:1）
- NEVER put labels inside form fields as the only label (placeholder-as-label pattern — labels must be visible when the field has content) — **绝不**将标签仅放在表单字段内作为占位符（占位符即标签模式——当字段有内容时标签必须可见）
- ALWAYS preserve visited vs unvisited link distinction (visited links must have a different color) — **始终**保留已访问和未访问链接的区别（已访问链接必须有不同的颜色）
- NEVER float headings between paragraphs (heading must be visually closer to the section it introduces than to the preceding section) — **绝不**让标题漂浮在段落之间（标题在视觉上必须比前一个部分更接近它所引入的部分）

**AI Slop blacklist** (the 10 patterns that scream "AI-generated"):
**AI劣质内容黑名单**（10种明显"AI生成"的模式）:
1. Purple/violet/indigo gradient backgrounds or blue-to-purple color schemes — 紫色/蓝紫/靛蓝渐变背景或蓝紫配色方案
2. **The 3-column feature grid:** icon-in-colored-circle + bold title + 2-line description, repeated 3x symmetrically. THE most recognizable AI layout. — **三列功能网格：** 彩色圆圈图标 + 粗体标题 + 2行描述，对称重复3次。最具辨识度的AI布局。
3. Icons in colored circles as section decoration (SaaS starter template look) — 彩色圆圈图标作为装饰（SaaS入门模板风格）
4. Centered everything (`text-align: center` on all headings, descriptions, cards) — 全部居中对齐（所有标题、描述、卡片都使用`text-align: center`）
5. Uniform bubbly border-radius on every element (same large radius on everything) — 每个元素统一的大圆角（所有东西都用同样的大圆角）
6. Decorative blobs, floating circles, wavy SVG dividers (if a section feels empty, it needs better content, not decoration) — 装饰性斑点、漂浮圆圈、波浪形SVG分隔线（如果一个部分感觉空，需要的是更好的内容，而非装饰）
7. Emoji as design elements (rockets in headings, emoji as bullet points) — Emoji作为设计元素（标题中的火箭、作为项目符号的emoji）
8. Colored left-border on cards (`border-left: 3px solid <accent>`) — 卡片的彩色左边框（`border-left: 3px solid <强调色>`）
9. Generic hero copy ("Welcome to [X]", "Unlock the power of...", "Your all-in-one solution for...") — 通用的英雄区文案（"欢迎使用[X]"、"释放...的力量"、"您的全方位解决..."）
10. Cookie-cutter section rhythm (hero → 3 features → testimonials → pricing → CTA, every section same height) — 千篇一律的段落节奏（英雄区 → 3个功能 → 推荐语 → 价格 → CTA，每个部分高度相同）
11. system-ui or `-apple-system` as the PRIMARY display/body font — the "I gave up on typography" signal. Pick a real typeface. — system-ui或`-apple-system`作为主显示/正文字体——"我放弃排版"的信号。选一个真正的字体。

Source: [OpenAI "Designing Delightful Frontends with GPT-5.4"](https://developers.openai.com/blog/designing-delightful-frontends-with-gpt-5.4) (Mar 2026) + gstack design methodology.
来源：[OpenAI "Designing Delightful Frontends with GPT-5.4"](https://developers.openai.com/blog/designing-delightful-frontends-with-gpt-5.4)（2026年3月）+ gstack设计方法论。
- "Cards with icons" → what differentiates these from every SaaS template? — "卡片+图标" → 这和每个SaaS模板有什么区别？
- "Hero section" → what makes this hero feel like THIS product? — "英雄区" → 是什么让这个英雄区像**这个**产品？
- "Clean, modern UI" → meaningless. Replace with actual design decisions. — "简洁现代UI" → 毫无意义。用实际的设计决策替换。
- "Dashboard with widgets" → what makes this NOT every other dashboard? — "带组件的仪表盘" → 是什么让它不同于所有其他仪表盘？
If visual mockups were generated in Step 0.5, evaluate them against the AI slop blacklist above. Read each mockup image using the Read tool. Does the mockup fall into generic patterns (3-column grid, centered hero, stock-photo feel)? If so, flag it and offer to regenerate with more specific direction via `$D iterate --feedback "..."`.
如果在步骤0.5中生成了视觉模型，请根据上面的AI劣质内容黑名单对其进行评估。使用Read工具读取每个模型图片。模型是否陷入通用模式（三列网格、居中英雄区、图库照片感）？如果是，标记它，并通过`$D iterate --feedback "..."`提供更具体的方向来重新生成。
**STOP.** AskUserQuestion once per issue. Do NOT batch. Recommend + WHY.
**停止。** 每个问题调用一次AskUserQuestion。不要批量处理。提供建议+原因。

### Pass 5: Design System Alignment — 第5轮：设计系统对齐
Rate 0-10（评分0-10）: Does the plan align with DESIGN.md? 计划是否与DESIGN.md对齐？
FIX TO 10（修复至10分）: If DESIGN.md exists, annotate with specific tokens/components. If no DESIGN.md, flag the gap and recommend `/design-consultation`.
修复至10分：如果存在DESIGN.md，用具体的标记/组件进行注释。如果没有DESIGN.md，标记这个差距并推荐`/design-consultation`。
Flag any new component — does it fit the existing vocabulary?
标记任何新组件——它是否符合现有的词汇体系？
**STOP.** AskUserQuestion once per issue. Do NOT batch. Recommend + WHY.
**停止。** 每个问题调用一次AskUserQuestion。不要批量处理。提供建议+原因。

### Pass 6: Responsive & Accessibility — 第6轮：响应式与无障碍
Rate 0-10（评分0-10）: Does the plan specify mobile/tablet, keyboard nav, screen readers? 计划是否指定了移动端/平板、键盘导航、屏幕阅读器支持？
FIX TO 10（修复至10分）: Add responsive specs per viewport — not "stacked on mobile" but intentional layout changes. Add a11y: keyboard nav patterns, ARIA landmarks, touch target sizes (44px min), color contrast requirements.
修复至10分：为每个视口添加响应式规格——不是"移动端堆叠"而是有意的布局变化。添加无障碍：键盘导航模式、ARIA地标、触摸目标尺寸（最小44px）、颜色对比度要求。
**STOP.** AskUserQuestion once per issue. Do NOT batch. Recommend + WHY.
**停止。** 每个问题调用一次AskUserQuestion。不要批量处理。提供建议+原因。

### Pass 7: Unresolved Design Decisions — 第7轮：未解决的设计决策
Surface ambiguities that will haunt implementation:
揭示困扰实现的模糊性：
```
  DECISION NEEDED              | IF DEFERRED, WHAT HAPPENS
  -----------------------------|---------------------------
  What does empty state look like? | Engineer ships "No items found."
  空状态长什么样？              | 工程师交付出"未找到项目。"
  Mobile nav pattern?          | Desktop nav hides behind hamburger
  移动端导航模式？              | 桌面端导航藏在汉堡菜单后面
  ...
```
If visual mockups were generated in Step 0.5, reference them as evidence when surfacing unresolved decisions. A mockup makes decisions concrete — e.g., "Your approved mockup shows a sidebar nav, but the plan doesn't specify mobile behavior. What happens to this sidebar on 375px?"
如果在步骤0.5中生成了视觉模型，在揭示未解决决策时引用它们作为证据。模型使决策具体化——例如，"你批准的模型显示侧边栏导航，但计划没有指定移动端行为。在375px下这个侧边栏会怎样？"
Each decision = one AskUserQuestion with recommendation + WHY + alternatives. Edit the plan with each decision as it's made.
每个决策 = 一个AskUserQuestion调用，提供建议+原因+备选方案。在每次决策做出时编辑计划。

### Post-Pass: Update Mockups (if generated) — 审后：更新模型（如果已生成）

If mockups were generated in Step 0.5 and review passes changed significant design decisions (information architecture restructure, new states, layout changes), offer to regenerate (one-shot, not a loop):
如果在步骤0.5中生成了模型，且审查轮次更改了重大设计决策（信息架构重组、新状态、布局变化），提供一次性重新生成（不是循环）：

AskUserQuestion: "The review passes changed [list major design changes]. Want me to regenerate mockups to reflect the updated plan? This ensures the visual reference matches what we're actually building."
AskUserQuestion："审查轮次更改了[列出重大设计更改]。你想让我重新生成模型以反映更新后的计划吗？这确保视觉参考匹配我们实际构建的内容。"

If yes, use `$D iterate` with feedback summarizing the changes, or `$D variants` with an updated brief. Save to the same `$_DESIGN_DIR` directory.
如果是，使用`$D iterate`并附带总结更改的反馈，或使用`$D variants`并附带更新的摘要。保存到相同的`$_DESIGN_DIR`目录。

## CRITICAL RULE — How to ask questions（关键规则——如何提问）
Follow the AskUserQuestion format from the Preamble above. Additional rules for plan design reviews:
遵循上述前言中的AskUserQuestion格式。计划设计审查的附加规则：
* **One issue = one AskUserQuestion call.** Never combine multiple issues into one question.
* **一个问题 = 一个AskUserQuestion调用。** 永远不要将多个问题合并为一个问题。
* Describe the design gap concretely — what's missing, what the user will experience if it's not specified.
* 具体描述设计缺陷——缺了什么，如果不指定用户会体验到什么。
* Present 2-3 options. For each: effort to specify now, risk if deferred.
* 提供2-3个选项。对于每个：现在指定的工作量，如果推迟的风险。
* **Map to Design Principles above.** One sentence connecting your recommendation to a specific principle.
* **映射到上面的设计原则。** 用一句话将你的建议与特定原则联系起来。
* Label with issue NUMBER + option LETTER (e.g., "3A", "3B").
* 用问题编号+选项字母标记（例如"3A"、"3B"）。
* **Zero findings:** if a section has zero findings, state "No issues, moving on" and proceed. Otherwise, use AskUserQuestion for each gap — a gap with an "obvious fix" is still a gap and still needs user approval before any change lands in the plan.
* **零发现：** 如果一个部分没有发现，说明"没有问题，继续"并继续。否则，对每个缺陷使用AskUserQuestion——有"明显修复"的缺陷仍然是缺陷，在任何更改落地到计划之前仍然需要用户批准。
* **NEVER use AskUserQuestion to ask which variant the user prefers.** Always create a comparison board first (`$D compare --serve`) and open it in the browser. The board has rating controls, comments, remix/regenerate buttons, and structured feedback output. Use AskUserQuestion ONLY to notify the user the board is open and wait for them to finish — not to present variants inline and ask "which do you prefer?" That is a degraded experience.
* **绝不使用AskUserQuestion询问用户更喜欢哪个变体。** 总是先创建一个比较面板（`$D compare --serve`）并在浏览器中打开。面板有评分控件、评论、重新混合/重新生成按钮，以及结构化的反馈输出。仅使用AskUserQuestion通知用户面板已打开并等待他们完成——不要以内联方式展示变体并问"你更喜欢哪个？"那是一种退化的体验。

## Required Outputs（必需输出）

### "NOT in scope" section（"不在范围"部分）
Design decisions considered and explicitly deferred, with one-line rationale each.
考虑过并明确推迟的设计决策，每项都有一个单行理由。

### "What already exists" section（"已存在"部分）
Existing DESIGN.md, UI patterns, and components that the plan should reuse.
现有的DESIGN.md、UI模式和组件，计划应复用。

### TODOS.md updates（TODOS.md更新）
After all review passes are complete, present each potential TODO as its own individual AskUserQuestion. Never batch TODOs — one per question. Never silently skip this step.
在所有审查轮次完成后，将每个潜在的TODO作为单独的AskUserQuestion呈现。永远不要批量处理TODO——每个问题一个。永远不要默默跳过这一步。

For design debt: missing a11y, unresolved responsive behavior, deferred empty states. Each TODO gets:
对于设计债务：缺失的无障碍、未解决的响应式行为、推迟的空状态。每个TODO获得：
* **What:** One-line description of the work. — **是什么：** 工作的单行描述。
* **Why:** The concrete problem it solves or value it unlocks. — **为什么：** 它解决的具体问题或解锁的价值。
* **Pros:** What you gain by doing this work. — **优点：** 做这项工作你获得什么。
* **Cons:** Cost, complexity, or risks of doing it. — **缺点：** 成本、复杂性或风险。
* **Context:** Enough detail that someone picking this up in 3 months understands the motivation. — **上下文：** 足够的细节，让3个月后接手的人理解动机。
* **Depends on / blocked by:** Any prerequisites. — **依赖/阻塞：** 任何先决条件。

Then present options: **A)** Add to TODOS.md **B)** Skip — not valuable enough **C)** Build it now in this PR instead of deferring.
然后提供选项：**A)** 添加到TODOS.md **B)** 跳过——价值不够大 **C)** 现在在这个PR中构建而不是推迟。

## Implementation Tasks（实施任务）

Before closing this review, synthesize the findings above into a flat list of build-actionable tasks. Each task derives from a specific finding — no padding.
在结束此审查之前，将上面的发现综合成一个扁平的可操作构建任务列表。每个任务源于一个具体的发现——无填充。
Emit the markdown section AND write a JSONL artifact that `/autoplan` can aggregate across phases.
输出markdown部分并写入一个JSONL构件，`/autoplan`可以跨阶段聚合。

### Markdown section (always emit)（Markdown部分（始终输出））

```markdown
## Implementation Tasks（实施任务）
Synthesized from this review's findings. Each task derives from a specific finding above. Run with Claude Code or Codex; checkbox as you ship.
从本次审查的综合发现综合而来。每个任务源于上面的具体发现。使用Claude Code或Codex运行；发货时勾选。

- [ ] **T1 (P1, human: ~2h / CC: ~15min)** — <component> — <imperative title>
  - Surfaced by: <section name> — <specific finding text or line reference>
  - Files: <paths to touch>
  - Verify: <test command or manual check>
- [ ] **T2 (P2, human: ~30min / CC: ~5min)** — ...
```

Rules（规则）:
- P1 blocks ship; P2 should land same branch; P3 is a follow-up TODO. — P1阻止发货；P2应在同一分支落地；P3是后续TODO。
- If a finding produced no actionable task, do not invent one. — 如果一个发现没有产生可操作的任务，不要发明一个。
- If a section had zero findings, emit `_No new tasks from <section>._` — 如果一个部分没有发现，输出`_没有新任务来自<部分>。_`
- Effort uses the AI-compression table from CLAUDE.md. — 工作量使用CLAUDE.md中的AI压缩表。

### JSONL artifact (always write, even if zero tasks)（JSONL构件（始终写入，即使零任务））

`/autoplan` reads this file to aggregate across phases. Build each line with `jq -nc` so titles and source findings containing quotes, newlines, or backslashes serialize cleanly — never use hand-rolled `echo` / `printf`.
`/autoplan`读取此文件以跨阶段聚合。使用`jq -nc`构建每一行，以便包含引号、换行符或反斜杠的标题和来源发现干净地序列化——永远不要使用手动拼接的`echo` / `printf`。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
TASKS_DIR="${HOME}/.gstack/projects/${SLUG:-unknown}"
mkdir -p "$TASKS_DIR"
TASKS_FILE="$TASKS_DIR/tasks-design-review-$(date +%Y%m%d-%H%M%S).jsonl"
COMMIT=$(git rev-parse HEAD 2>/dev/null || echo unknown)
BRANCH=$(git branch --show-current 2>/dev/null || echo unknown)
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-$$"

# Repeat ONE jq invocation per task identified during this review.
# 对于本次审查中识别的每个任务，重复执行一次jq调用。
# Substitute the placeholders inline with shell variables you set per task:
# 使用每个任务设置的shell变量内联替换占位符：
#   TASK_ID (T1, T2, ...), PRIORITY (P1/P2/P3), COMPONENT, TITLE,
#   SOURCE_FINDING, EFFORT_HUMAN, EFFORT_CC, FILES_JSON (a JSON array literal
#   like '["browse/src/sanitize.ts","browse/src/server.ts"]').
jq -nc \
  --arg phase 'design-review' \
  --arg run_id "$RUN_ID" \
  --arg branch "$BRANCH" \
  --arg commit "$COMMIT" \
  --arg id "$TASK_ID" \
  --arg priority "$PRIORITY" \
  --arg component "$COMPONENT" \
  --arg effort_human "$EFFORT_HUMAN" \
  --arg effort_cc "$EFFORT_CC" \
  --arg title "$TITLE" \
  --arg source_finding "$SOURCE_FINDING" \
  --argjson files "$FILES_JSON" \
  '{phase:$phase, run_id:$run_id, branch:$branch, commit:$commit, id:$id, priority:$priority, component:$component, files:$files, effort_human:$effort_human, effort_cc:$effort_cc, title:$title, source_finding:$source_finding}' \
  >> "$TASKS_FILE"
```

If `jq` is not installed, fall back to skipping the JSONL write and warn the user to install jq for autoplan aggregation. Never hand-roll JSONL.
如果未安装`jq`，回退跳过JSONL写入，并警告用户安装jq以进行autoplan聚合。永远不要手动拼接JSONL。

If zero tasks were identified in this review, still touch the JSONL file (`: > "$TASKS_FILE"`) so the aggregator sees that the phase produced output this run (an empty file means "ran, no findings" — distinct from "didn't run").
如果本次审查中未识别到任何任务，仍然触碰JSONL文件（`: > "$TASKS_FILE"`），以便聚合器看到此阶段在本次运行中产生了输出（空文件表示"已运行，无发现"——与"未运行"不同）。


### Completion Summary（完成总结）
```
  +====================================================================+
  |         DESIGN PLAN REVIEW — COMPLETION SUMMARY                    |
  |         设计计划审查 — 完成总结                                     |
  +====================================================================+
  | System Audit         | [DESIGN.md status, UI scope]                |
  | 系统审计              | [DESIGN.md 状态, UI 范围]                    |
  | Step 0               | [initial rating, focus areas]               |
  | 步骤0                 | [初始评分, 重点关注区域]                     |
  | Pass 1  (Info Arch)  | ___/10 → ___/10 after fixes                |
  | 第1轮  (信息架构)     | ___/10 → 修复后 ___/10                       |
  | Pass 2  (States)     | ___/10 → ___/10 after fixes                |
  | 第2轮  (状态)         | ___/10 → 修复后 ___/10                       |
  | Pass 3  (Journey)    | ___/10 → ___/10 after fixes                |
  | 第3轮  (旅程)         | ___/10 → 修复后 ___/10                       |
  | Pass 4  (AI Slop)    | ___/10 → ___/10 after fixes                |
  | 第4轮  (AI劣质内容)   | ___/10 → 修复后 ___/10                       |
  | Pass 5  (Design Sys) | ___/10 → ___/10 after fixes                |
  | 第5轮  (设计系统)     | ___/10 → 修复后 ___/10                       |
  | Pass 6  (Responsive) | ___/10 → ___/10 after fixes                |
  | 第6轮  (响应式)       | ___/10 → 修复后 ___/10                       |
  | Pass 7  (Decisions)  | ___ resolved, ___ deferred                 |
  | 第7轮  (决策)         | ___ 已解决, ___ 已推迟                      |
  +--------------------------------------------------------------------+
  | NOT in scope         | written (___ items)                         |
  | 不在范围内            | 已写入 (___ 项)                              |
  | What already exists  | written                                     |
  | 已存在的内容          | 已写入                                       |
  | TODOS.md updates     | ___ items proposed                          |
  | TODOS.md更新          | ___ 项已提出                                 |
  | Approved Mockups     | ___ generated, ___ approved                 |
  | 已批准的模型          | ___ 已生成, ___ 已批准                       |
  | Decisions made       | ___ added to plan                           |
  | 已做出的决策          | ___ 已添加到计划                             |
  | Decisions deferred   | ___ (listed below)                          |
  | 已推迟的决策          | ___（列在下面）                              |
  | Overall design score | ___/10 → ___/10                             |
  | 整体设计评分          | ___/10 → ___/10                             |
  +====================================================================+
```

If all passes 8+（如果所有轮次评分都≥8）: "Plan is design-complete. Run /design-review after implementation for visual QA." — "计划设计已完成。实施后运行 /design-review 进行视觉QA。"
If any below 8（如果有任何一轮低于8）: note what's unresolved and why (user chose to defer) — 说明什么是未解决的以及为什么（用户选择推迟）。

### Unresolved Decisions（未解决的决策）
If any AskUserQuestion goes unanswered, note it here. Never silently default to an option.
如果有任何AskUserQuestion未获回答，在此注明。永远不要默默默认一个选项。

### Approved Mockups（已批准的模型）

If visual mockups were generated during this review, add to the plan file:
如果在此审查期间生成了视觉模型，添加到计划文件：

```
## Approved Mockups（已批准的模型）

| Screen/Section | Mockup Path | Direction | Notes |
|----------------|-------------|-----------|-------|
| [screen name]  | ~/.gstack/projects/$SLUG/designs/[folder]/[filename].png | [brief description] | [constraints from review] |
| [屏幕名称]     | ~/.gstack/projects/$SLUG/designs/[folder]/[filename].png | [简要描述]           | [审查约束]                 |
```

Include the full path to each approved mockup (the variant the user chose), a one-line description of the direction, and any constraints. The implementer reads this to know exactly which visual to build from. These persist across conversations and workspaces. If no mockups were generated, omit this section.
包含每个已批准模型的完整路径（用户选择的变体）、方向的单行描述以及任何约束。实施者阅读此内容以了解要构建的确切视觉效果。这些在对话和工作区之间持久存在。如果未生成模型，则省略此部分。

## Review Log（审查日志）

After producing the Completion Summary above, persist the review result.
生成上面的完成总结后，持久化审查结果。

**PLAN MODE EXCEPTION — ALWAYS RUN:** This command writes review metadata to `~/.gstack/` (user config directory, not project files). The skill preamble already writes to `~/.gstack/sessions/` and `~/.gstack/analytics/` — this is the same pattern. The review dashboard depends on this data. Skipping this command breaks the review readiness dashboard in /ship.
**计划模式例外——始终运行：** 此命令将审查元数据写入`~/.gstack/`（用户配置目录，非项目文件）。技能前言已经写入`~/.gstack/sessions/`和`~/.gstack/analytics/`——这是相同的模式。审查面板依赖此数据。跳过此命令会破坏/ship中的审查准备度面板。

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-design-review","timestamp":"TIMESTAMP","status":"STATUS","initial_score":N,"overall_score":N,"unresolved":N,"decisions_made":N,"commit":"COMMIT"}'
```

Substitute values from the Completion Summary（从完成总结中替换值）:
- **TIMESTAMP（时间戳）**: current ISO 8601 datetime — 当前ISO 8601日期时间
- **STATUS（状态）**: "clean" if overall score 8+ AND 0 unresolved; otherwise "issues_open" — 如果整体评分≥8且0个未解决则为"clean"；否则为"issues_open"
- **initial_score（初始评分）**: initial overall design score before fixes (0-10) — 修复前的初始整体设计评分（0-10）
- **overall_score（整体评分）**: final overall design score after fixes (0-10) — 修复后的最终整体设计评分（0-10）
- **unresolved（未解决）**: number of unresolved design decisions — 未解决的设计决策数量
- **decisions_mades（已做出的决策）**: number of design decisions added to the plan — 添加到计划的设计决策数量
- **COMMIT（提交）**: output of `git rev-parse --short HEAD` — `git rev-parse --short HEAD`的输出

## Review Readiness Dashboard（审查准备度面板）

After completing the review, read the review log and config to display the dashboard.
完成审查后，读取审查日志和配置以显示面板。

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

Parse the output. Find the most recent entry for each skill (plan-ceo-review, plan-eng-review, review, plan-design-review, design-review-lite, adversarial-review, codex-review, codex-plan-review). Ignore entries with timestamps older than 7 days. For the Eng Review row, show whichever is more recent between `review` (diff-scoped pre-landing review) and `plan-eng-review` (plan-stage architecture review). Append "(DIFF)" or "(PLAN)" to the status to distinguish. For the Adversarial row, show whichever is more recent between `adversarial-review` (new auto-scaled) and `codex-review` (legacy). For Design Review, show whichever is more recent between `plan-design-review` (full visual audit) and `design-review-lite` (code-level check). Append "(FULL)" or "(LITE)" to the status to distinguish. For the Outside Voice row, show the most recent `codex-plan-review` entry — this captures outside voices from both /plan-ceo-review and /plan-eng-review.
解析输出。找到每个技能的最新条目。忽略时间戳超过7天的条目。对于工程审查行，显示`review`（差异范围的落地前审查）和`plan-eng-review`（计划阶段架构审查）中较新的一个。对于对抗行，显示`adversarial-review`（新自动扩展）和`codex-review`（旧版）中较新的一个。对于设计审查，显示`plan-design-review`（完整视觉审计）和`design-review-lite`（代码级别检查）中较新的一个。对于外部声音行，显示最新的`codex-plan-review`条目。

**Source attribution（来源归属）:** If the most recent entry for a skill has a `"via"` field, append it to the status label in parentheses.
如果技能的最新条目有`"via"`字段，将其附加到括号中的状态标签。

Note（注意）: `autoplan-voices` and `design-outside-voices` entries are audit-trail-only (forensic data for cross-model consensus analysis). They do not appear in the dashboard and are not checked by any consumer.
说明：`autoplan-voices`和`design-outside-voices`条目仅用于审计追踪（跨模型共识分析的取证数据）。它们不出现在面板中，也不被任何消费者检查。

Display（展示）:

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
|                    审查准备度面板                                    |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
| 审查            | 运行次数 | 最近运行时间        | 状态      | 是否必需   |
|-----------------|------|---------------------|-----------|----------|
| Eng Review      |  1   | 2026-03-16 15:00    | CLEAR     | YES      |
| 工程审查        |      |                     | 通过       | 是        |
| CEO Review      |  0   | —                   | —         | no       |
| CEO审查         |      |                     | —         | 否        |
| Design Review   |  0   | —                   | —         | no       |
| 设计审查        |      |                     | —         | 否        |
| Adversarial     |  0   | —                   | —         | no       |
| 对抗审查        |      |                     | —         | 否        |
| Outside Voice   |  0   | —                   | —         | no       |
| 外部声音        |      |                     | —         | 否        |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed                                |
| 判定：已清除 — 工程审查已通过                                       |
+====================================================================+
```

**Review tiers（审查层级）:**
- **Eng Review (required by default):** The only review that gates shipping. Covers architecture, code quality, tests, performance. Can be disabled globally with `gstack-config set skip_eng_review true` (the "don't bother me" setting). — **工程审查（默认为必需）**：唯一一个控制发货的审查。涵盖架构、代码质量、测试、性能。可以使用`gstack-config set skip_eng_review true`全局禁用（"别烦我"设置）。
- **CEO Review (optional):** Use your judgment. Recommend it for big product/business changes, new user-facing features, or scope decisions. Skip for bug fixes, refactors, infra, and cleanup. — **CEO审查（可选）**：自行判断。推荐用于重大产品/业务变更、新的面向用户的功能或决策范围。对于错误修复、重构、基础设施和清理可跳过。
- **Design Review (optional):** Use your judgment. Recommend it for UI/UX changes. Skip for backend-only, infra, or prompt-only changes. — **设计审查（可选）**：自行判断。推荐用于UI/UX变更。对于仅后端、基础设施或仅提示的变更可跳过。
- **Adversarial Review (automatic):** Always-on for every review. Every diff gets both Claude adversarial subagent and Codex adversarial challenge. Large diffs (200+ lines) additionally get Codex structured review with P1 gate. No configuration needed. — **对抗审查（自动）**：对每次审查始终开启。每个差异都同时获得Claude对抗子代理和Codex对抗挑战。大差异（200+行）额外获得带有P1门的Codex结构化审查。无需配置。
- **Outside Voice (optional):** Independent plan review from a different AI model. Offered after all review sections complete in /plan-ceo-review and /plan-eng-review. Falls back to Claude subagent if Codex is unavailable. Never gates shipping. — **外部声音（可选）**：来自不同AI模型的独立计划审查。在/plan-ceo-review和/plan-eng-review的所有审查部分完成后提供。如果Codex不可用，回退到Claude子代理。绝不控制发货。

**Verdict logic（判定逻辑）:**
- **CLEARED（已清除）**: Eng Review has >= 1 entry within 7 days from either `review` or `plan-eng-review` with status "clean" (or `skip_eng_review` is `true`) — 工程审查在7天内有≥1条来自`review`或`plan-eng-review`的状态为"clean"的条目（或`skip_eng_review`为`true`）
- **NOT CLEARED（未清除）**: Eng Review missing, stale (>7 days), or has open issues — 工程审查缺失、过期（>7天）或有未解决的问题
- CEO, Design, and Codex reviews are shown for context but never block shipping — CEO、设计和Codex审查仅为上下文显示，绝不阻止发货
- If `skip_eng_review` config is `true`, Eng Review shows "SKIPPED (global)" and verdict is CLEARED — 如果`skip_eng_review`配置为`true`，工程审查显示"SKIPPED (global)"，判定为已清除

**Staleness detection（过期检测）:** After displaying the dashboard, check if any existing reviews may be stale: 显示面板后，检查是否有任何现有审查可能已过期：
- Parse the `---HEAD---` section from the bash output to get the current HEAD commit hash — 解析bash输出中的`---HEAD---`部分以获取当前HEAD提交哈希
- For each review entry that has a `commit` field: compare it against the current HEAD. If different, count elapsed commits: `git rev-list --count STORED_COMMIT..HEAD`. Display: "Note: {skill} review from {date} may be stale — {N} commits since review" — 对于有`commit`字段的每个审查条目：将其与当前HEAD比较。如果不同，计算经过的提交数：`git rev-list --count STORED_COMMIT..HEAD`。显示："注意：来自{date}的{skill}审查可能已过期——审查以来有{N}次提交"
- For entries without a `commit` field (legacy entries): display "Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection" — 对于没有`commit`字段的条目（旧条目）：显示"注意：来自{date}的{skill}审查没有提交跟踪——考虑重新运行以进行准确的过期检测"
- If all reviews match the current HEAD, do not display any staleness notes — 如果所有审查与当前HEAD匹配，不显示任何过期提示

## Plan File Review Report（计划文件审查报告）

After displaying the Review Readiness Dashboard in conversation output, also update the **plan file** itself so review status is visible to anyone reading the plan.
在对话输出中显示审查准备度面板后，还更新**计划文件**本身，以便审查状态对任何阅读计划的人可见。

### Detect the plan file（检测计划文件）

1. Check if there is an active plan file in this conversation (the host provides plan file paths in system messages — look for plan file references in the conversation context). — 检查此对话中是否有活动计划文件（宿主在系统消息中提供计划文件路径——在对话上下文中查找计划文件引用）。
2. If not found, skip this section silently — not every review runs in plan mode. — 如果未找到，默默跳过此部分——并非每次审查都在计划模式下运行。

### Generate the report（生成报告）

Read the review log output you already have from the Review Readiness Dashboard step above. Parse each JSONL entry. Each skill logs different fields:
读取你已从上一步审查准备度面板步骤获得的审查日志输出。解析每个JSONL条目。每个技能记录不同的字段：

- **plan-ceo-review**: `status`, `unresolved`, `critical_gaps`, `mode`, `scope_proposed`, `scope_accepted`, `scope_deferred`, `commit`
  → Findings: "{scope_proposed} proposals, {scope_accepted} accepted, {scope_deferred} deferred" → 发现："{scope_proposed}个提议，{scope_accepted}个已接受，{scope_deferred}个已推迟"
  → If scope fields are 0 or missing (HOLD/REDUCTION mode): "mode: {mode}, {critical_gaps} critical gaps" → 如果作用域字段为0或缺失（HOLD/REDUCTION模式）："模式：{mode}，{critical_gaps}个关键缺陷"
- **plan-eng-review**: `status`, `unresolved`, `critical_gaps`, `issues_found`, `mode`, `commit`
  → Findings: "{issues_found} issues, {critical_gaps} critical gaps" → 发现："{issues_found}个问题，{critical_gaps}个关键缺陷"
- **plan-design-review**: `status`, `initial_score`, `overall_score`, `unresolved`, `decisions_made`, `commit`
  → Findings: "score: {initial_score}/10 → {overall_score}/10, {decisions_made} decisions" → 发现："评分：{initial_score}/10 → {overall_score}/10，{decisions_made}个决策"
- **plan-devex-review**: `status`, `initial_score`, `overall_score`, `product_type`, `tthw_current`, `tthw_target`, `mode`, `persona`, `competitive_tier`, `unresolved`, `commit`
  → Findings: "score: {initial_score}/10 → {overall_score}/10, TTHW: {tthw_current} → {tthw_target}" → 发现："评分：{initial_score}/10 → {overall_score}/10，TTHW：{tthw_current} → {tthw_target}"
- **devex-review**: `status`, `overall_score`, `product_type`, `tthw_measured`, `dimensions_tested`, `dimensions_inferred`, `boomerang`, `commit`
  → Findings: "score: {overall_score}/10, TTHW: {tthw_measured}, {dimensions_tested} tested/{dimensions_inferred} inferred" → 发现："评分：{overall_score}/10，TTHW：{tthw_measured}，{dimensions_tested}个已测试/{dimensions_inferred}个已推断"
- **codex-review**: `status`, `gate`, `findings`, `findings_fixed`
  → Findings: "{findings} findings, {findings_fixed}/{findings} fixed" → 发现："{findings}个发现，{findings_fixed}/{findings}已修复"

All fields needed for the Findings column are now present in the JSONL entries.
Findings列所需的所有字段现在都存在于JSONL条目中。
For the review you just completed, you may use richer details from your own Completion Summary. For prior reviews, use the JSONL fields directly — they contain all required data.
对于你刚刚完成的审查，你可以使用自己完成总结中更丰富的细节。对于先前的审查，直接使用JSONL字段——它们包含所有必需的数据。

Produce this markdown table:
生成此markdown表：

```markdown
## GSTACK REVIEW REPORT（GStack审查报告）

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| 审查     | 触发器   | 原因  | 运行次数 | 状态     | 发现       |
| CEO Review | `/plan-ceo-review` | Scope & strategy — 范围与策略 | {runs} | {status} | {findings} |
| Codex Review | `/codex review` | Independent 2nd opinion — 独立第二意见 | {runs} | {status} | {findings} |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) — 架构与测试（必需） | {runs} | {status} | {findings} |
| Design Review | `/plan-design-review` | UI/UX gaps — UI/UX缺陷 | {runs} | {status} | {findings} |
| DX Review | `/plan-devex-review` | Developer experience gaps — 开发者体验缺陷 | {runs} | {status} | {findings} |
```

Below the table, add these lines. **CODEX** and **CROSS-MODEL** are optional (omit when empty); **VERDICT** is always present:
在表格下方添加这些行。**CODEX**和**CROSS-MODEL**是可选的（为空时省略）；**VERDICT**始终存在：

- **CODEX:** (only if codex-review ran) — one-line summary of codex fixes — **CODEX：**（仅当运行了codex-review时）——codex修复的单行总结
- **CROSS-MODEL:** (only if both Claude and Codex reviews exist) — overlap analysis — **跨模型：**（仅当同时存在Claude和Codex审查时）——重叠分析
- **VERDICT:** list reviews that are CLEAR (e.g., "CEO + ENG CLEARED — ready to implement"). — **判定：** 列出已清除的审查（例如，"CEO + ENG CLEARED — 已准备好实施"）。
  If Eng Review is not CLEAR and not skipped globally, append "eng review required". — 如果工程审查未清除且未全局跳过，添加"需要工程审查"。

**Unresolved-decisions status (MANDATORY — never omitted; the report's final non-whitespace line).** After VERDICT, end the report with exactly one: the exact unbolded line `NO UNRESOLVED DECISIONS` (a bolded one does NOT count), OR a `**UNRESOLVED DECISIONS:**` header + one bullet per open item.
**未解决决策状态（强制——永远不省略；报告最后一个非空白行）。** 在判定之后，以准确的一行结束报告：准确的未加粗行`没有未解决的决策`（加粗的不算），或一个`**未解决的决策：**`标题 + 每个开放项目一个项目符号。
This avoids double-counting: list THIS review's open items from context; for prior reviews sum `unresolved` over the latest fresh row per skill (dashboard 7-day window) after you DROP the current skill's row; emit the sentinel only when both are zero.
这避免了双重计数：从上下文列出本次审查的开放项目；对于先前的审查，在你DROP当前技能行后，对每个技能最新的新鲜行求和`unresolved`（面板7天窗口）；仅当两者都为零时才发出哨兵。

### Write to the plan file（写入计划文件）

**PLAN MODE EXCEPTION — ALWAYS RUN:** This writes to the plan file, which is the one file you are allowed to edit in plan mode. The plan file review report is part of the plan's living status.
**计划模式例外——始终运行：** 这写入计划文件，这是你在计划模式下被允许编辑的唯一文件。计划文件审查报告是计划动态状态的一部分。

The report must always be the LAST section of the plan file — never mid-file.
报告必须是计划文件的最后部分——永远不在文件中间。
Use a single delete-then-append flow:
使用单个删除然后追加流程：

1. Read the plan file (Read tool) to see its full current content. Search the read output for a `## GSTACK REVIEW REPORT` heading anywhere in the file. — 读取计划文件（Read工具）以查看其完整当前内容。在读取输出中搜索文件中任何地方的`## GSTACK REVIEW REPORT`标题。
2. If found, use the Edit tool to DELETE the entire existing section. Match from `## GSTACK REVIEW REPORT` through either the next `## ` heading or end of file, whichever comes first. Replace with the empty string. This applies regardless of where the section currently lives — mid-file deletion is intentional, not a special case. If the Edit fails (e.g., concurrent edit changed the content), re-read the plan file and retry once. — 如果找到，使用Edit工具删除整个现有部分。从`## GSTACK REVIEW REPORT`到下一个`## `标题或文件结尾，以先到者为准。替换为空字符串。这适用于该部分当前所在的任何位置——文件中间删除是故意的，不是特殊情况。如果Edit失败（例如，并发编辑更改了内容），重新读取计划文件并重试一次。
3. After the delete (or skipped, if no section existed), append the new `## GSTACK REVIEW REPORT` section at the END of the file. Use the Edit tool to match the file's current last paragraph and add the section after it, or use Write to re-emit the whole file with the section at the end. — 删除后（或跳过，如果没有部分存在），在文件末尾追加新的`## GSTACK REVIEW REPORT`部分。使用Edit工具匹配文件的当前最后一并在其后添加该部分，或使用Write重新发出整个文件并将该部分放在末尾。
4. Verify with the Read tool that `## GSTACK REVIEW REPORT` is the last `## ` heading in the file before continuing. If it isn't, repeat steps 2-3 once. — 用Read工具验证在继续之前`## GSTACK REVIEW REPORT`是文件中的最后一个`## `标题。如果不是，重复步骤2-3一次。

Do NOT replace the section in place. The "replace mid-file" path is what allowed prior versions to leave the report mid-file when an older report already lived there — the user then sees a plan whose review report is not at the bottom and (correctly) rejects it.
不要在原地替换该部分。"在文件中替换"路径是允许先前版本在已有旧报告时将报告留在文件中间的原因——用户会看到审查报告不在底部的计划并（正确地）拒绝它。

## Capture Learnings（捕获学习记录）

If you discovered a non-obvious pattern, pitfall, or architectural insight during this session, log it for future sessions:
如果你在此会话中发现了一个非显而易见的模式、陷阱或架构洞察，为未来会话记录它：

```bash
~/.claude/skills/gstack/bin/gstack-learnings-log '{"skill":"plan-design-review","type":"TYPE","key":"SHORT_KEY","insight":"DESCRIPTION","confidence":N,"source":"SOURCE","files":["path/to/relevant/file"]}'
```

**Types（类型）:** `pattern`（reusable approach — 可复用方法）, `pitfall`（what NOT to do — 不该做什么）, `preference`（user stated — 用户陈述）, `architecture`（structural decision — 结构决策）, `tool`（library/framework insight — 库/框架洞察）, `operational`（project environment/CLI/workflow knowledge — 项目环境/CLI/工作流知识）。

**Sources（来源）:** `observed`（you found this in the code — 你在代码中发现）, `user-stated`（user told you — 用户告诉你）, `inferred`（AI deduction — AI推理）, `cross-model`（both Claude and Codex agree — Claude和Codex都同意）。

**Confidence（置信度）:** 1-10. Be honest. An observed pattern you verified in the code is 8-9. An inference you're not sure about is 4-5. A user preference they explicitly stated is 10.
1-10。诚实评估。在代码中验证的观察模式为8-9。你不确定的推断为4-5。用户明确陈述的偏好为10。

**files:** Include the specific file paths this learning references. This enables staleness detection: if those files are later deleted, the learning can be flagged.
**文件：** 包含此学习引用的具体文件路径。这支持过期检测：如果这些文件后来被删除，该学习可被标记为过期。

**Only log genuine discoveries.** Don't log obvious things. Don't log things the user already knows. A good test: would this insight save time in a future session? If yes, log it.
**仅记录真正的发现。** 不要记录显而易见的东西。不要记录用户已经知道的东西。一个好的测试：这个洞察会在未来会话中节省时间吗？如果是，记录它。



## Brain Calibration Write-Back (Phase 2 / gated)（大脑校准回写（第2阶段 / 门控））

When the skill makes a typed prediction worth tracking（当技能做出一个值得追踪的类型化预测时） (scope decision, TTHW target, architectural bet, wedge commitment), it MAY write a `kind=bet` take to the brain so a calibration profile builds over time.
它可以向大脑写入一个`kind=bet`的看法，以便随时间建立校准档案。

**Gated on two things（两个门控条件）:**
1. Brain trust policy for the active endpoint is `personal`（活动端点的大脑信任策略为`personal`） (check via `~/.claude/skills/gstack/bin/gstack-config get brain_trust_policy@<endpoint-hash>`). Shared brains skip write-back to avoid polluting team calibration. — 共享大脑跳过回写以避免污染团队校准。
2. Feature flag `BRAIN_CALIBRATION_WRITEBACK` is set（功能标志`BRAIN_CALIBRATION_WRITEBACK`已设置） (today: false; flips to true when upstream gbrain v0.42+ ships `takes_add` MCP op). （今天：false；当上游gbrain v0.42+发布`takes_add` MCP操作时翻转为true。）

When both gates pass, the write-back path uses `mcp__gbrain__takes_add` to record a take with weight 0.5 (per SKILL_CALIBRATION_WEIGHTS). If the MCP op is unavailable, fall back to `mcp__gbrain__put_page` with a gstack:takes fence block (documented but uglier path).
当两个门控都通过时，回写路径使用`mcp__gbrain__takes_add`记录一个权重为0.5的看法（根据SKILL_CALIBRATION_WEIGHTS）。如果MCP操作不可用，回退到使用gstack:takes围栏块的`mcp__gbrain__put_page`（已记录但较不优雅的路径）。

Mandatory take frontmatter shape:
必需的看法前置元数据形状：
```yaml
kind: bet
holder: <user identity from whoami>
claim: <one-line prediction the skill is making> — <技能做出的单行预测>
weight: 0.5
since_date: <today's date>
expected_resolution: <date in 1-3 months depending on skill> — <1-3个月内取决于技能的日期>
source_skill: plan-design-review
```

After write, invalidate the affected digests so the next preflight reflects the new state:
写入后，使受影响的摘要失效，以便下一次预检反映新状态：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate brand --project "$SLUG" 2>/dev/null || true
```


## Brain Cache Background Refresh（大脑缓存后台刷新）

After the skill's work completes (and telemetry has logged), kick a background refresh of any cache digest that's getting close to its TTL. This is non-blocking — the user doesn't wait. Next invocation benefits from the warm cache.
在技能工作完成（并且遥测已记录）后，启动任何接近其TTL的缓存摘要的后台刷新。这是非阻塞的——用户不需要等待。下一次调用从温暖的缓存中受益。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
(~/.claude/skills/gstack/bin/gstack-brain-cache refresh --project "$SLUG" 2>/dev/null &) || true
```


## Next Steps — Review Chaining（下一步——审查链）

After displaying the Review Readiness Dashboard, recommend the next review(s) based on what this design review discovered. Read the dashboard output to see which reviews have already been run and whether they are stale.
显示审查准备度面板后，根据此设计审查的发现推荐下一步审查。读取面板输出以查看哪些审查已经运行以及它们是否已过期。

**Recommend /plan-eng-review if eng review is not skipped globally（如果工程审查未全局跳过，推荐 /plan-eng-review）** — check the dashboard output for `skip_eng_review`. If it is `true`, eng review is opted out — do not recommend it. Otherwise, eng review is the required shipping gate. If this design review added significant interaction specifications, new user flows, or changed the information architecture, emphasize that eng review needs to validate the architectural implications. If an eng review already exists but the commit hash shows it predates this design review, note that it may be stale and should be re-run.
检查面板输出中的`skip_eng_review`。如果为`true`，工程审查被选择退出——不要推荐它。否则，工程审查是必需的发货物门口。如果此设计审查添加了重要的交互规范、新的用户流或更改了信息架构，强调工程审查需要验证架构影响。如果已存在工程审查但提交哈希显示它早于此设计审查，注意它可能已过期，应重新运行。

**Consider recommending /plan-ceo-review（考虑推荐 /plan-ceo-review）** — but only if this design review revealed fundamental product direction gaps. Specifically: if the overall design score started below 4/10, if the information architecture had major structural problems, or if the review surfaced questions about whether the right problem is being solved. AND no CEO review exists in the dashboard. This is a selective recommendation — most design reviews should NOT trigger a CEO review.
但仅当此设计审查揭示了根本性的产品方向缺陷时。具体：如果整体设计评分开始低于4/10，如果信息架构有重大结构问题，或者如果审查引出了关于是否解决了正确问题的问题。并且面板中没有CEO审查。这是一个选择性推荐——大多数设计审查不应触发CEO审查。

**If both are needed, recommend eng review first** (required gate). — **如果两者都需要，先推荐工程审查**（必需门）。

**Recommend design exploration skills when appropriate（适当时推荐设计探索技能）** — /design-shotgun and /design-html produce design artifacts (mockups, HTML previews), not application code. They belong in plan mode alongside reviews. If this design review found visual issues that would benefit from exploring new directions, recommend /design-shotgun. If approved mockups exist and need to be turned into working HTML, recommend /design-html.
/design-shotgun和/design-html生成设计产物（模型、HTML预览），而非应用代码。它们属于计划模式中的审查。如果此设计审查发现了将从探索新方向中受益的视觉问题，推荐/design-shotgun。如果存在已批准的模型并且需要将其转化为可用的HTML，推荐/design-html。

Use AskUserQuestion to present the next step. Include only applicable options:
使用AskUserQuestion呈现下一步。仅包括适用的选项：
- **A)** Run /plan-eng-review next (required gate) — 运行 /plan-eng-review 下一个（必需门）
- **B)** Run /plan-ceo-review (only if fundamental product gaps found) — 运行 /plan-ceo-review（仅当发现根本性产品缺陷时）
- **C)** Run /design-shotgun — explore visual design variants for issues found — 运行 /design-shotgun — 为发现的问题探索视觉设计变体
- **D)** Run /design-html — generate Pretext-native HTML from approved mockups — 运行 /design-html — 从批准的模型生成Pretext原生HTML
- **E)** Skip — I'll handle next steps manually — 跳过——我将手动处理下一步

## Formatting Rules（格式规则）
* NUMBER issues (1, 2, 3...) and LETTERS for options (A, B, C...). — 编号问题（1、2、3...）和选项字母（A、B、C...）。
* Label with NUMBER + LETTER (e.g., "3A", "3B"). — 用编号+字母标记（例如"3A"、"3B"）。
* One sentence max per option. — 每个选项最多一句话。
* After each pass, pause and wait for feedback. — 每轮之后暂停并等待反馈。
* Rate before and after each pass for scannability. — 在每轮前后对可扫性进行评分。

