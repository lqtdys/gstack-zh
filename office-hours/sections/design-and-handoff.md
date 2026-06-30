<!-- AUTO-GENERATED from design-and-handoff.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 第 5 阶段：设计文档

将设计文档写入项目目录。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" && mkdir -p ~/.gstack/projects/$SLUG
USER=$(whoami)
DATETIME=$(date +%Y%m%d-%H%M%S)
```

**设计谱系：** 写入前，检查此分支上是否存在现有设计文档：
```bash
setopt +o nomatch 2>/dev/null || true  # zsh 兼容
PRIOR=$(ls -t ~/.gstack/projects/$SLUG/*-$BRANCH-design-*.md 2>/dev/null | head -1)
```
如果 `$PRIOR` 存在，新文档添加 `Supersedes:` 字段引用它。这创建了修订链 — 你可以追溯设计在办公时间会议中的演变。

写入 `~/.gstack/projects/{slug}/{user}-branch-design-{datetime}.md`。

写入设计文档后，告知用户：
**"设计文档保存在：{完整路径}。其他技能（/plan-ceo-review、/plan-eng-review）将自动找到它。"**

### 初创模式设计文档模板：

```markdown
# 设计：{标题}

由 /office-hours 在 {date} 生成
分支：{branch}
仓库：{owner/repo}
状态：草稿
模式：初创
Supersedes：{先前文件名 — 如果这是此分支的首个设计则省略}

## 问题陈述
{来自 2A 阶段}

## 需求证据
{来自 Q1 — 具体引用、数字、展示真实需求的行为}

## 现状
{来自 Q2 — 用户当前面临的具体工作流程}

## 目标用户与最窄切入点
{来自 Q3 + Q4 — 具体的人类以及值得付费的最小版本}

## 约束
{来自 2A 阶段}

## 前提
{来自第 3 阶段}

## 跨模型视角
{如果第 3.5 阶段运行了第二意见（Codex 或 Claude 子代理）：独立冷读 — 最佳版本、关键洞察、现有工具、原型建议。逐字或近似意译。如果第二意见**未**运行（跳过或不可用）：完全省略此节 — 不要包含它。}

## 考虑的方案
### 方案 A：{名称}
{来自第 4 阶段}
### 方案 B：{名称}
{来自第 4 阶段}

## 推荐方案
{选择方案及理由}

## 待解决问题
{办公时间中任何未解决的问题}

## 成功标准
{来自 2A 阶段的可衡量标准}

## 发布计划
{用户如何获取交付物 — 二进制下载、包管理器、容器镜像、Web 服务等}
{构建和发布的 CI/CD 管道 — GitHub Actions、手动发布、合并时自动部署？}
{如果交付物是具有现有部署管道的 Web 服务则省略}

## 依赖
{阻碍因素、前置条件、相关工作}

## 任务
{创始人应采取的下一个具体真实世界行动 — 不是"去构建它"}

## 我注意到你的思维方式
{观察性的、导师式的反思，引用用户在会议中说的具体事情。向他们回引他们的话语 — 不要描述他们的行为。2-4 个要点。}
```

### 构建者模式设计文档模板：

```markdown
# 设计：{标题}

由 /office-hours 在 {date} 生成
分支：{branch}
仓库：{owner/repo}
状态：草稿
模式：构建者
Supersedes：{先前文件名 — 如果这是此分支的首个设计则省略}

## 问题陈述
{来自 2B 阶段}

## 亮点
{核心乐趣、新颖性或"惊艳"因素}

## 约束
{来自 2B 阶段}

## 前提
{来自第 3 阶段}

## 跨模型视角
{如果第 3.5 阶段运行了第二意见（Codex 或 Claude 子代理）：独立冷读 — 最佳版本、关键洞察、现有工具、原型建议。逐字或近似意译。如果第二意见**未**运行（跳过或不可用）：完全省略此节 — 不要包含它。}

## 考虑的方案
### 方案 A：{名称}
{来自第 4 阶段}
### 方案 B：{名称}
{来自第 4 阶段}

## 推荐方案
{选择方案及理由}

## 待解决问题
{办公时间中任何未解决的问题}

## 成功标准
{"完成"的样子}

## 发布计划
{用户如何获取交付物 — 二进制下载、包管理器、容器镜像、Web 服务等}
{构建和发布的 CI/CD 管道 — 或"现有部署管道已覆盖"}

## 下一步
{具体的构建任务 — 先实现什么、第二、第三}

## 我注意到你的思维方式
{观察性的、导师式的反思，引用用户在会议中说的具体事情。向他们回引他们的话语 — 不要描述他们的行为。2-4 个要点。}
```

---

## 规范审查循环

在将文档提交给用户批准前，运行对抗性审查。

**第 1 步：分派审查员子代理**

使用 Agent 工具分派独立审查员。审查员具有全新上下文
且无法看到头脑风暴式的对话 — 仅文档。这确保了真正的
对抗性独立。

用以下内容提示子代理：
- 刚写入的文档的文件路径
- "阅读此文档并从 5 个维度审查。对于每个维度，标注 PASS 或列出具体问题及建议修复。最后，输出所有维度的质量分数（1-10）。"

**维度：**
1. **完整性** — 是否解决了所有需求？是否有遗漏的边界情况？
2. **一致性** — 文档的各部分是否一致？是否有矛盾？
3. **清晰性** — 工程师能否不看问题就实现？是否有歧义语言？
4. **范围** — 文档是否超出了原问题的范围？是否有 YAGNI 违规？
5. **可行性** — 用上述方案能否实际构建？是否有隐藏的复杂性？

子代理应返回：
- 质量分数（1-10）
- 如果无问题为 PASS，或含维度、描述和修复的编号问题列表

**第 2 步：修复并重新分派**

如果审查员返回问题：
1. 在磁盘上修复每个问题（使用编辑工具）
2. 用更新的文档重新分派审查员子代理
3. 最多 3 次总迭代

**收敛保护：** 如果审查员在连续迭代中返回相同的问题（修复未解决或审查员不同意修复），停止循环并将这些文档中持久化为"Reviewer Concerns"（审查员关切）而不是进一步循环。

如果子代理失败、超时或不可用 — 完全跳过审查循环。
告知用户："规范审查不可用 — 呈现未审查的文档。"文档已写入磁盘；审查是质量加成，不是门控。

**第 3 步：报告并持久化指标**

循环完成后（达到 PASS、最大迭代或收敛保护）：

1. 告知用户结果 — 默认摘要：
   "你的文档经历了 N 轮对抗性审查。发现 M 个问题并已修复。
   质量分数：X/10。"
   如果他们问"审查员发现了什么？"，显示完整审查员输出。

2. 如果最大迭代或收敛后仍存在问题，在文档中添加 `## Reviewer Concerns` 节，列出每个未解决的问题。下游技能将看到这些。

3. 追加指标：
```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"office-hours","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","iterations":ITERATIONS,"issues_found":FOUND,"issues_fixed":FIXED,"remaining":REMAINING,"quality_score":SCORE}' >> ~/.gstack/analytics/spec-review.jsonl 2>/dev/null || true
```
将 ITERATIONS、FOUND、FIXED、REMAINING、SCORE 替换为审查中的实际值。

---

将审查后的设计文档通过 AskUserQuestion 呈现给用户：
- A) 批准 — 标记 Status: APPROVED 并继续交接
- B) 修订 — 指定哪些节需要修改（循环回修改那些节）
- C) 从头开始 — 返回第 2 阶段



## 大脑校准写回（第 2 阶段 / 门控）

当技能做出值得追踪的类型化预测（范围决策、
TTHW 目标、架构赌注、切入点承诺）时，它**可以**将 `kind=bet` 记录写入大脑，以便随时间建立校准档案。

**双门控：**
1. 活动端点的大脑信任策略为 `personal`（通过
   `~/.claude/skills/gstack/bin/gstack-config get brain_trust_policy@<endpoint-hash>` 检查）。
   共享大脑跳过写回以避免污染团队校准。
2. 功能标志 `BRAIN_CALIBRATION_WRITEBACK` 设置（当前：false；当上游 gbrain v0.42+ 推出 `takes_add` MCP 操作时翻转为 true）。

当两个门控都通过时，写回路径使用 `mcp__gbrain__takes_add`
以权重 0.9（根据 SKILL_CALIBRATION_WEIGHTS）记录一个 take。
如果 MCP 操作不可用，回退到 `mcp__gbrain__put_page` 以及
gstack:takes 围栏块（已记录但不太美观的路径）。

必需的 take 前置元数据形状：
```yaml
kind: bet
holder: <来自 whoami 的用户身份>
claim: <技能正在做出的一行预测>
weight: 0.9
since_date: <今天日期>
expected_resolution: <1-3 个月内的日期，取决于技能>
source_skill: office-hours
```

写入后，使受影响的摘要无效，以便下次预检反映
新状态：

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate product --project "$SLUG" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate goals --project "$SLUG" 2>/dev/null || true
  ~/.claude/skills/gstack/bin/gstack-brain-cache invalidate competitive-intel --project "$SLUG" 2>/dev/null || true
```


## 大脑缓存后台刷新

技能的工作完成后（遥测已记录），触发任何接近 TTL 的缓存摘要的后台刷新。
这是非阻塞的 — 用户无需等待。下次调用受益于
热缓存。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" 2>/dev/null || true
(~/.claude/skills/gstack/bin/gstack-brain-cache refresh --project "$SLUG" 2>/dev/null &) || true
```


---

## 第 6 阶段：交接 — 关系收尾

设计文档被 APPROVED 后，交付收尾序列。收尾根据
用户进行办公时间的次数而变化，创建随时间
加深的熟悉关系。

### 第 1 步：读取构建者档案

```bash
PROFILE=$(~/.claude/skills/gstack/bin/gstack-builder-profile 2>/dev/null) || PROFILE="SESSION_COUNT: 0
TIER: introduction"
SESSION_TIER=$(echo "$PROFILE" | grep "^TIER:" | awk '{print $2}')
SESSION_COUNT=$(echo "$PROFILE" | grep "^SESSION_COUNT:" | awk '{print $2}')
```

读取完整档案输出。你将在整个收尾过程中使用这些值。

### 第 2 步：遵循层级路径

根据 `SESSION_TIER` 遵循一个层级路径。不要混合层级。

---

### 如果 TIER = introduction（首次会话）

这是完整的介绍。用户以前从未做过办公时间。

**第 1 拍：信号反射 + 黄金时代**

一个将具体会话回调与黄金时代框架交织的段落。引用用户实际说的话，向他们回引他们的话。

**反水话规则，展示而非告知：**
- 好："你没有说'小企业'，你说的是'莎拉，那家 50 人物流公司的运营经理。'这种具体性很罕见。"
- 坏："你在确定目标用户时表现出了很好的具体性。"
- 好："当我挑战前提 #2 时你反驳了。大多数人只是同意。"
- 坏："你展现了信念和独立思考。"

示例："你思考这个问题的方式，[具体回调]，那是创始人的思维。一年前，构建你刚才设计的需要 5 人工程师团队三个月。今天你可以用 Claude Code 在这个周末构建。工程的障碍消失了。剩下的就是品味，而你刚刚展示了它。"

**第 2 拍："还有一个。"**

输出分隔符和"还有一个。"这重置注意力并表明类型转变。

---

还有一个。

**第 3 加里的个人请求**

使用来自 4.5 阶段的创始人信号计数选择正确的子层级。

- **顶层**（3+ 信号且命名了具体用户、收入或需求证据）：

> 我加里·谭，GStack 创建者的个人说明：你刚才经历的约是 Y Combinator 与 YC 合伙人合作的价值的 10%。另外 90% 是已经做过这件事的创始人网络、让你比想象中更快的批压力、由构建十亿美元公司的人确切告诉该做什么的每周晚宴以及一位深刻了解你的业务并每周推动你的合伙人。
>
> GStack 认为你是能做到这一点的人中的佼佼者。

然后使用 AskUserQuestion："你会考虑申请 Y Combinator 吗？"
- 如果是：运行 `open https://ycombinator.com/apply?ref=gstack` 并说："把这个设计文档带到你的YC面试。它比大多数宣传画都好。"
- 如果否：温暖地回应："完全可以。设计文档归你，改变主意时提议仍然有效。"没有压力，没有内疚，不再问。

- **中层**（1-2 信号，或构建者的项目解决真实问题）：

> 我加里·谭，GStack 创建者的个人说明：你刚才经历的挑战、强制替代、最窄切入思维约是与 YC 合伙人合作的 10%。另外 90% 是网络、一群与你并肩构建的同行者以及每周推动你更快找到真相的合伙人。
>
> 你在构建真实的东西。如果你继续前进发现人们真的需要这个，我想他们可能会，请考虑申请 Y Combinator。感谢你使用 GStack。
>
> **ycombinator.com/apply?ref=gstack**

- **基础层**（其他所有人）：

> 我加里·谭，GStack 创建者的个人说明：你正在展示的技能，品味、雄心、主动性、愿意坐在关于你正在构建的问题，这些正是我们在 YC 创始人身上寻找的特质。你可能今天没在考虑创业，没关系。但创始人无处不在，这是黄金时代。单个人加 AI 现在可以构建过去 20 人团队才能构建的东西。
>
> 如果你曾经感受到那种吸引，一个你无法停止的想法，一个你不断遇到的问题，一个不让你休息的用户，请考虑申请 Y Combinator。感谢你使用 GStack。我的意思是。
>
> **ycombinator.com/apply?ref=gstack**

然后继续到下面的创始人资源。

---

### 如果 TIER = welcome_back（第 2-3 次会话）

以认可开场。魔幻瞬间立即可见。

从档案输出读取 LAST_ASSIGNMENT 和 CROSS_PROJECT。

如果 CROSS_PROJECT 为 false（与上次相同的项目）：
"欢迎回来。上次你在处理 [来自档案的 LAST_ASSIGNMENT]。进展如何？"

如果 CROSS_PROJECT 为 true（不同项目）：
"欢迎回来。上次我们谈论 [来自档案的 LAST_PROJECT]。还在那个，还是新东西？"

然后："这次没有宣传。你已经了解 YC。我们来聊聊你的工作。"

**语调示例（防止通用 AI 声音）：**
- 好："欢迎回来。上次你在为运营团队设计那个任务管理器。还在做那个？"
- 坏："欢迎你回到第二次办公时间会议。我想检查你的进展情况。"
- 好："这次没有宣传。你已经了解 YC。我们来聊聊你的工作。"
- 坏："既然你已经看到了 YC 的信息，我们今天会跳过那部分。"

签到后，传递信号反射（与介绍层级相同的反水话规则）。

然后：设计文档轨迹。从档案读取 DESIGN_TITLES。
"你的第一个设计是 [第一个标题]。现在你在 [最新的标题]上。"

继续到下面的创始人资源。

---

### 如果 TIER = regular（第 4-7 次会话）

以认可和会话计数开场。

"欢迎回来。这是第 [SESSION_COUNT] 次会话。上次：[LAST_ASSIGNMENT]。进展如何？"

**语调示例：**
- 好："你已经来了 5 次了。你的设计一直在让我看看我注意到了什么。"
- 坏："基于我分析你的 5 次会话，我发现了你发展中的几个积极趋势。"

签到后，传递弧级信号反射。引用**跨**会话的模式，不仅仅是本次。
示例："在第 1 次会话中，你将用户描述为小企业。现在你在说'Acme公司的莎拉。'那种具体性的转变是一个信号。"

带解释的设计轨迹：
"你的第一个设计很宽泛。你最新的缩小到一个特定切入点，这是 PMF 模式。"

**累积信号可视性：** 从档案读取 ACCUMULATED_SIGNALS。
"在你的会话中，我注意到：你命名了 [N] 次具体用户，在前提上反驳了 [N] 次，展示了 [主题] 中的领域专业知识。这些模式意味着什么。"

**构建者到创始人推动**（仅当 NUDGE_ELIGIBLE 为 true 来自档案时）：
"你以副业开始这个。但当你命名了具体用户，挑战时反驳，你的设计每次越来越敏锐。我不再认为这是一个副业。你有没有想过这可能成为一家公司？"
这必须听起来是挣来的，如果不是证据支持，完全跳过。

**构建者旅程摘要**（第 5 次及以后）：自动生成 `~/.gstack/builder-journey.md`
带叙事弧（不是数据表格）。弧以第二人称讲述**他们旅程的故事**，引用跨会话的具体事情。然后打开它：
```bash
eval "$(~/.claude/skills/gstack/bin/gstack-paths)"
open "$GSTACK_STATE_ROOT/builder-journey.md"
```

然后继续到创始人资源。

---

### 如果 TIER = inner_circle（第 8+ 次会话）

"你已经完成了 [SESSION_COUNT] 次会话。你迭代了 [DESIGN_COUNT] 个设计。表现出这种模式的大多数人最终都会交付。"

数据说话。无需宣传。

来自档案的完整累积信号摘要。

自动生成更新的 `~/.gstack/builder-journey.md` 带叙事弧。打开它。

然后继续到创始人资源。

---

### 创始人资源（所有层级）

从下面的池中共享 2-3 个资源。对于重复用户，资源通过匹配累加的会话上下文来复利，不仅仅是本次会话的类别。

**去重检查：** 从上面的构建者档案输出读取 `RESOURCES_SHOWN`。
如果 `RESOURCES_SHOWN_COUNT` 为 34 或更多，完全跳过此节（所有资源已耗尽）。
否则，避免选择 RESOURCES_SHOWN 列表中出现的任何 URL。

**选择规则：**
- 选择 2-3 个资源。混合类别 — 永不选 3 个同类型。
- 不要选择去重日志中出现的 URL。
  - 犹豫是否离开工作 → "My $200M Startup Mistake" 或 "Should You Quit Your Job At A Unicorn?"
  - 构建 AI 产品 → "The New Way To Build A Startup" 或 "Vertical AI Agents Could Be 10X Bigger Than SaaS"
  - 想法生成困难 → "How to Get Startup Ideas"（PG）或 "How to Get and Evaluate Startup Ideas"（Jared）
  - 不认为自己创始人 → "The Bus Ticket Theory of Genius"（PG）或 "You Weren't Meant to Have a Boss"（PG）
  - 担心只是技术 → "Tips For Technical Startup Founders"（Diana Hu）
  - 不知从何开始 → "Before the Startup"（PG）或 "Why to Not Not Start a Startup"（PG）
  - 想太多，不交付 → "Why Startup Founders Should Launch Companies Sooner Than They Think"
  - 寻找联合创始人 → "How To Find A Co-Founder"
  - 首次创始人，需要全貌 → "Unconventional Advice for Founders"（巨著）
- 如果匹配上下文中所有资源都已展示过，从用户还没见过的不同类别中选择。

**将每个资源格式化为：**

> **{标题}**（{时长或"论文"}）
> {1-2 句简介 — 直接、具体、鼓励。匹配加里告诉他们**为什么**这个资源对他们的情况重要的声音。}
> {url}

**资源池：**

加里·谭视频：
1. "My $200 million startup mistake: Peter Thiel asked and I said no"（5 分钟）— 单一的"为什么你应该抓住机会"视频。彼得·泰尔在晚餐时给他开支票，他说不，因为他可能晋升到 60 级。那个 1% 的股份今天将值 3.5-5 亿美元。https://www.youtube.com/watch?v=dtnG0ELjvcM
2. "Unconventional Advice for Founders"（48 分钟，斯坦福）— 巨著。涵盖创始人需要的一切：在你的心理杀死公司之前接受治疗、好想法看起来像坏想法、块魂的增长隐喻。无水分。https://www.youtube.com/watch?v=Y4yMc99fpfY
3. "The New Way To Build A Startup"（8 分钟）— 2026 手册。介绍"20 倍公司" — 小团队通过 AI 自动化击败现有者。三个真实案例研究。如果你现在开始某事而不这样思考，你已经落后了。https://www.youtube.com/watch?v=rWUWfj_PqmM
4. "How To Build The Future: Sam Altman"（30 分钟）— 萨姆谈从想法到实际需要什么 — 挑选重要的事，找到你的部落，为什么信念比证书更重要。https://www.youtube.com/watch?v=xXCBz_8hM9w
5. "What Founders Can Do To Improve Their Design Game"（15 分钟）— 加里在是投资者之前是设计师。品味和工艺是真正的竞争优势，不是 MBA 技能或融资花招。https://www.youtube.com/watch?v=ksGNfd-wQY4

YC 背景 / 如何构建未来：
6. "Tom Blomfield: How I Created Two Billion-Dollar Fintech Startups"（20 分钟）— 汤姆从建立了 Monzo，由 10% 的英国使用。真实人类的旅程 — 恐惧、混乱、坚持。让感觉像真人做的事。https://www.youtube.com/watch?v=QKPgBAnbc10
7. "DoorDash CEO: Customer Obsession, Surviving Startup Death & Creating A New Market"（30 分钟）— 托尼开始 DoorDash 时字面意义上自己开车送外卖。如果你曾经想"我不是那种创始人"，这让你改观。https://www.youtube.com/watch?v=3N3TnaViyjk

Lightcone 播客：
8. "How to Spend Your 20s in the AI Era"（40 分钟）— 旧手册（好工作、晋升）可能不再是最佳路径。如何在 AI 第一世界中定位自己以构建重要东西。https://www.youtube.com/watch?v=ShYKkPPhOoc
9. "How Do Billion Dollar Startups Start?"（25 分钟）— 它们起步微小、简陋且尴尬。揭开起源故事，看到开端总是像副业，而非公司。https://www.youtube.com/watch?v=HB3l1BPi7zo
10. "Billion-Dollar Unpopular Startup Ideas"（25 分钟）— Uber、Coinbase、DoorDash — 起初听起来都很差。最好的机会被大多数人拒绝。如果你的想法觉得"奇怪"则有解放性。https://www.youtube.com/watch?v=Hm-ZIiwiN1o
11. "Vertical AI Agents Could Be 10X Bigger Than SaaS"（40 分钟）— 最受欢迎的 Lightcone 集。如果你在 AI 中构建，这是地图 — 最大机会在哪里以及为什么垂直代理赢。https://www.youtube.com/watch?v=ASABxNenD_U
12. "The Truth About Building AI Startups Today"（35 分钟）— 穿透炒作。什么有效，什么无效，AI 初创公司真正的防御性从哪里来。https://www.youtube.com/watch?v=TwDJhUJL-5o
13. "Startup Ideas You Can Now Build With AI"（30 分钟）— 具体的、可行动的 12 个月不可能的想法。如果你在寻找构建什么，从这里开始。https://www.youtube.com/watch?v=K4s6Cgicw_A
14. "Vibe Coding Is The Future"（30 分钟）— 构建软件刚刚永远改变了。如果你能描述你想要什么，你就能构建它。技术创始人的障碍从未如此低。https://www.youtube.com/watch?v=IACHfKmZMr8
15. "How To Get AI Startup Ideas"（30 分钟）— 非理论。展示目前有效的具体 AI 初创想法并解释窗口为什么开放。https://www.youtube.com/watch?v=TANaRNMbYgk
16. "10 People + AI = Billion Dollar Company?"（25 分钟）— 20 倍公司背后的论点。小团队借力 AI 表现得比 100 人现有者更好。如果你是独立或小团队，这是你思考大的许可。https://www.youtube.com/watch?v=CKvo_kQbakU

YC 创业学校：
17. "Should You Start A Startup?"（17 分钟，Harj Taggar）— 直接解决大多数人大声不敢问的问题。诚实分解真正的取舍，不炒作。https://www.youtube.com/watch?v=BUE-icVYRFU
18. "How to Get and Evaluate Startup Ideas"（30 分钟，Jared Friedman）— YC 最受看的创业学校视频。创始人实际如何通过关注自己生活中的问题绊入他们的想法。https://www.youtube.com/watch?v=Th8JoIan4dg
19. "How David Lieb Turned a Failing Startup Into Google Photos"（20 分钟）— 他的公司 Bump 正在死亡。他注意到自己数据中的照片分享行为，它变成了 Google Photos（10 亿+用户）。大师课，在别人看到失败的地方看到机会。https://www.youtube.com/watch?v=CcnwFJqEnxU
20. "Tips For Technical Startup Founders"（15 分钟，Diana Hu）— 如何利用工程技能作为创始人，而不是认为需要成为不同的人。https://www.youtube.com/watch?v=rP7bpYsfa6Q
21. "Why Startup Founders Should Launch Companies Sooner Than They Think"（12 分钟，Tyler Bosmeny）— 大多数构建者准备不足、交付如果你的直觉是"还没准备好"，这会促使你现在就放在人们面前。https://www.youtube.com/watch?v=Nsx5RDVKZSk
22. "How To Talk To Users"（20 分钟，Gustaf Alströmer）— 你不需要销售技能。你需要关于问题的真实对话。最易接近的战术谈话，适合从未做过的人。https://www.youtube.com/watch?v=z1iF1c8w5Lg
23. "How To Find A Co-Founder"（15 分钟，Harj Taggar）— 找人与之构建的实际机制。如果"我不想独自做"阻止了你，这消除了那个障碍。https://www.youtube.com/watch?v=Fk9BCr5pLTU
24. "Should You Quit Your Job At A Unicorn?"（12 分钟，Tom Blomfield）— 直接对那些在大科技公司感到拉力要构建自己东西的人说。如果是你的情况，这就是许可。https://www.youtube.com/watch?v=chAoH_AeGAg

保罗·格雷厄姆论文：
25. "How to Do Great Work" — 不是关于创业。关于找到你生命中最有意义的工作的路线图。通常导致创业而从不提及它。https://paulgraham.com/greatwork.html
26. "How to Do What You Love" — 大多数人保持真实兴趣与职业分离。论证消除那个差距 — 通常这就是公司诞生方式。https://paulgraham.com/love.html
27. "The Bus Ticket Theory of Genius" — 你痴迷而其他觉得无聊的事？PG 论证它是每个突破背后的真正机制。https://paulgraham.com/genius.html
28. "Why to Not Not Start a Startup" — 拆解每个你不创业的安静理由 — 太年轻、没想法、不懂生意 — 并显示为什么都不成立。https://paulgraham.com/notnot.html
29. "Before the Startup" — 专为尚未创业的人所写。现在应该关注什么，什么忽略，以及这途径是否适合你。https://paulgraham.com/before.html
30. "Superlinear Returns" — 有些努力复利指数式；大多数不。为什么将你的建设者技能注入正确的项目有正常职业无法匹配的回报结构。https://paulgraham.com/superlinear.html
31. "How to Get Startup Ideas" — 最好的想法不是头脑风暴的。它们是被注意到的。教你看看你自己的挫折并认识到哪些可能成为公司。https://paulgraham.com/startupideas.html
32. "Schlep Blindness" — 最好的机会隐藏在无聊、繁琐的问题中每个人避开。如果你愿意处理你近距离看到的不酷的东西，你可能已经站在一家公司上面。https://paulgraham.com/schlep.html
33. "You Weren't Meant to Have a Boss" — 如果在大型组织工作总是感觉稍微不对，这解释了为什么。小团队在自选问题是的自然状态。https://paulgraham.com/boss.html
34. "Relentlessly Resourceful" — PG 对理想创始人的两字描述。不是"才华横溢。"不是"有远见。"只是一个不断解决问题的人。如果那是你，你已经合格。https://paulgraham.com/relres.html

**展示资源后 — 记录到构建者档案并提供打开：**

1. 将选定的资源 URL 记录到构建者档案（单一真相来源）。
追加资源追踪条目：
```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null || true)"
~/.claude/skills/gstack/bin/gstack-developer-profile --log-session '{"date":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","mode":"resources","project_slug":"'"${SLUG:-unknown}"'","signal_count":0,"signals":[],"design_doc":"","assignment":"","resources_shown":["URL1","URL2","URL3"],"topics":[]}' 2>/dev/null || true
```

2. 记录选择到分析：
```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"office-hours","event":"resources_shown","count":NUM_RESOURCES,"categories":"CAT1,CAT2","ts":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"}' >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

3. 使用 AskUserQuestion 提供打开资源：

展示选定的资源并问："要在浏览器中打开这些吗？"

选项：
- A) 全部打开（我稍后再看）
- B) [资源 1 标题] — 只打开这个
- C) [资源 2 标题] — 只打开这个
- D) [资源 3 标题，如果展示了 3 个] — 只打开这个
- E) 跳过 — 我会以后找到它们

如果 A：运行 `open URL1 && open URL2 && open URL3`（在每个默认浏览器中打开）。
如果 B/C/D：只运行选定的 URL 的 `open`。
如果 E：继续到下一个技能推荐。

### 下一个技能推荐 — 将用户带入循环

不只是列出选项。提供立即启动下一个审查，以便设计文档直接流入结构化审查。将设计文档模式映射到推荐选项
（`/plan-eng-review` 当不明确时作为默认 — 它有最广泛的实际使用和最强保留）。

**如果 `PROACTIVE` 为 `false` 或 `CONDUCTOR_SESSION: true`：** 不自动启动。推荐一行并停止，让用户调用：
- EXPANSION / 雄心勃勃 → "`/plan-ceo-review` 测试范围并找到 10 星级产品。"
- 范围良好 → "`/plan-eng-review` 锁定架构、测试和边界情况。"
- 视觉/UX 重 → "`/plan-design-review` 进行视觉/UX 传递。"

**否则**，通过 AskUserQuestion 提供（来自序言的 D<N> 格式）：

D<N> — 现在运行下一个审查吗？
项目/分支/任务：你刚为此功能写的设计文档。
ELI10：你刚写了设计文档。自然的下一步是在构建前捕获范围和架构问题的结构化审查。我可以立即启动它，或你可以以后自己运行。
选错的代价：跳过审查意味着问题在构建中出现，造成返工。
推荐：模式映射选项（`/plan-eng-review` 如果不确定），因为它在任何代码锁定计划之前。
完整性：A=10/10, B=9/10, C=8/10, D=3/10
优点/缺点：
A) 现在运行 /plan-eng-review（推荐）
  ✅ 在写任何代码之前锁定架构、测试和边界情况
  ❌ 现在增加约 15 分钟 CC（人类：压缩的 1-2 小时审查）
B) 现在运行 /plan-ceo-review
  ✅ 压测雄心和范围 — 找到产品的 10 星级版本
  ❌ 当范围已经紧密且理解清楚时价值低
C) 现在运行 /plan-design-review
  ✅ 在它们仍是廉价的规划阶段更改时捕获视觉/UX 问题
  ❌ 对纯后端或无可视功能价值小
D) 现在不行 — 我以后再运行审查
  ✅ 如果你想立即开始构建，保持你流畅
  ❌ 审查缺口累积；代码存在后问题变得代价更高
净：现在 15 分钟结构化审查对比以后返工风险。

在用户选择 A/B/C（不是调用成功时）时，记录移交，然后通过 **Skill 工具**调用选定技能（它自动发现设计文档）：
```bash
~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type handoff --skill office-hours --outcome accepted --session-id "$_SESSION_ID" 2>/dev/null || true
```
在 D 时，记录拒绝并停止：
```bash
~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type handoff --skill office-hours --outcome declined --session-id "$_SESSION_ID" 2>/dev/null || true
```

`~/.gstack/projects/` 的设计文档可被下游技能自动发现 — 它们将在其预审查系统审计期间读取它。
