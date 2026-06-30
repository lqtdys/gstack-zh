<!-- AUTO-GENERATED from proposal-and-preview.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 第 3 阶段：完整提案

这是技能的核心。将一切作为一个连贯的包来提议。

**AskUserQuestion Q2 — 呈现完整提案加 SAFE/RISK 分解：**

```
基于[产品上下文]和[研究发现/我的设计知识]：

美学：[方向] — [一行理由]
装饰：[级别] — [为什么这与美学搭配]
布局：[方法] — [为什么这适合产品类型]
色彩：[方法] + 提议的调色板（十六进制值）— [理由]
排版：[3 个字体推荐加角色] — [为什么这些字体]
间距：[基本单位 + 密度] — [理由]
动效：[方法] — [理由]

这个系统之所以连贯，是因为[解释选择如何相互加强]。

安全选择（类别基线——你的用户期望这些）：
  - [2-3 个符合类别惯例的决定，附选择安全的理由]

风险（你的产品获得自己面孔的地方）：
  - [2-3 个有意识地偏离惯例的决定]
  - 对每个风险：它是什么、为什么有效、获得什么、失去什么

安全选择让你在自己的类别中保持可读。风险是你的产品变得难忘的
地方。哪些风险吸引你？想看不同的风险吗？还是要调整其他任何东西？
```

SAFE/RISK 分解至关重要。设计连贯性只是入场费——每个类别的产品都可以是连贯的，但仍然看起来相同。真正的问题是：你在哪里承担创作风险？代理应该始终提议至少 2 个风险，每个都有清晰的理由说明为什么风险值得承担以及用户放弃什么。风险可能包括：该类别中不寻常的字体的使用、其他人不使用的醒目强调色、比正常更紧或更松的间距、打破常规的布局方法、增加个性的动效选择。

**选项：** A) 看起来不错——生成预览页。B) 我想调整[部分]。C) 我想要不同的风险——给我看更野的选项。D) 从头开始，用不同方向。E) 跳过预览，只写 DESIGN.md。

### 你的设计知识（用于信息告知提议——不要显示为表格）

**美学方向**（选适合产品的那个）：
- 残酷极简 — 只有字体和留白。没有装饰。现代主义。
- 极繁混乱 — 密集、分层、图案重。Y2K 遇见当代。
- 复古未来主义 — 复古科技怀旧。CRT 辉光、像素网格、暖色等宽。
- 奢华/精致 — 衬线体、高对比度、宽大留白、贵金属。
- 俏皮/玩具感 — 圆润、弹跳、粗体原色。平易近人且有趣。
- 编辑/杂志 — 强排版层次、不对称网格、引用。
- 粗野/原始 — 暴露结构、系统字体、可见网格、无打磨。
- 装饰艺术 — 几何精确、金属强调、对称、装饰边框。
- 有机/自然 — 大地色调、圆润形式、手绘纹理、颗粒。
- 工业/功能优先 — 功能先行、数据密集、等宽强调、柔和调色板。

**装饰级别：** 极简（排版承担所有工作）/ 有意识（微妙纹理、颗粒或背景处理）/ 表现力（完整创意方向、分层深度、图案）

**布局方法：** 网格克制（严格列、可预测对齐）/ 创意编辑（不对称、重叠、打破网格）/ 混合（应用用网格，营销用创意）

**色彩方法：** 克制（1 个强调色 + 中性色，色彩是稀有且有意义的）/ 平衡（原色 + 次级色，语义色用于层次）/ 表现力（色彩作为主要设计工具，粗体调色板）

**动效方法：** 极简功能（仅有助于理解的过渡）/ 有意识（微妙的进入动画、有意义的状态转换）/ 表现力（完整编排、滚动驱动、俏皮）

**按用途推荐字体：**
- 显示/大标题：Satoshi、General Sans、Instrument Serif、Fraunces、Clash Grotesk、Cabinet Grotesk
- 正文：Instrument Sans、DM Sans、Source Sans 3、Geist、Plus Jakarta Sans、Outfit
- 数据/表格：Geist（tabular-nums）、DM Sans（tabular-nums）、JetBrains Mono、IBM Plex Mono
- 代码：JetBrains Mono、Fira Code、Berkeley Mono、Geist Mono

**字体黑名单**（绝不推荐）：
Papyrus、Comic Sans、Lobster、Impact、Jokerman、Bleeding Cowboys、Permanent Marker、Bradley Hand、Brush Script、Hobo、Trajan、Raleway、Clash Display、Courier New（用于正文）

**过度使用字体**（不要作为主要推荐——仅在用户特别要求时使用）：
Inter、Roboto、Arial、Helvetica、Open Sans、Lato、Montserrat、Poppins、Space Grotesk。

Space Grotesk 在此列表上特别是因为每个 AI 设计工具都将它收敛为"Inter 的安全替代品"。这就是收敛陷阱。将其与 Inter 同样对待：仅在用户按名称要求时才使用。

**抗收敛指令：** 在同一项目的多次生成中，浅色/深色、字体和美学方向之间要有变化。没有明确解释的理由，永远不要提议相同的选择。如果用户的上一会话使用了 Geist + 深色 + 编辑风，这次提议其他东西（或明确承认你因为符合简报而加倍下注）。跨代收敛是 SLOP。

**AI SLOP 反模式**（永远不要在推荐中包含）：
- 紫色/渐变作为默认强调色
- 带彩色圆圈图标的 3 列功能网格
- 所有的东西居中且均匀间距
- 所有元素统一的气泡边框半径
- 作为主要 CTA 模式的渐变按钮
- 通用的库存照片风格的英雄区域
- system-ui / -apple-system 作为主要显示或正文字体（"我放弃排版"信号）
- "为 X 打造" / "为 Y 设计"营销文案模式

### 连贯性验证

当用户覆盖一个部分时，检查其余部分是否仍然连贯。温和地轻推不匹配——永远不要阻止：

- 粗野/极简美学 + 表现力动效 → "注意：粗野美学通常与极简动效搭配。你的组合很不寻常——如果有意为之也可以。要我建议合适的动效，还是保留？"
- 表现力色彩 + 克制装饰 → "粗体调色板加少量装饰可以奏效，但色彩会承载很多重量。要我建议支持调色板的装饰吗？"
- 创意编辑布局 + 数据重产品 → "编辑布局很美但可能与数据密度冲突。要我展示混合方法如何保持两者吗？"
- 始终接受用户的最终选择。永不拒绝继续进行。

---

## 第 4 阶段：深入探讨（仅在用户请求调整时）

当用户想要更改特定部分时，深入了解该部分：

- **字体：** 呈现 3-5 个具体候选加理由，解释每个唤起什么，提供预览页
- **色彩：** 呈现 2-3 个调色板选项加十六进制值，解释色彩理论推理
- **美学：** 简要说明哪些方向适合他们的产品以及为什么
- **布局/间距/动效：** 为他们产品类型呈现带具体权衡的方法

每次深入探讨是一个专注的 AskUserQuestion。用户决定后，重新检查与系统其余部分的连贯性。

---

## 第 5 阶段：设计系统预览（默认开启）

该阶段生成提议设计系统的视觉预览。两条路径，取决于 gstack 设计器是否可用。

### 路径 A：AI 模型（如果 DESIGN_READY）

生成 AI 渲染的模型，展示提议设计系统对该产品真实屏幕的应用。这比 HTML 预览更强大——用户看到他们的产品可能看起来像什么。

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
_DESIGN_DIR="$HOME/.gstack/projects/$SLUG/designs/design-system-$(date +%Y%m%d)"
mkdir -p "$_DESIGN_DIR"
echo "DESIGN_DIR: $_DESIGN_DIR"
```

从第 3 阶段的提案（美学、色彩、字体、间距、布局）和第 1 阶段的产品上下文构建设计简报：

```bash
$D variants --brief "<product name: [name]. Product type: [type]. Aesthetic: [direction]. Colors: primary [hex], secondary [hex], neutrals [range]. Typography: display [font], body [font]. Layout: [approach]. Show a realistic [page type] screen with [specific content for this product].>" --count 3 --output-dir "$_DESIGN_DIR/"
```

对每个变体运行质量检查：

```bash
$D check --image "$_DESIGN_DIR/variant-A.png" --brief "<the original brief>"
```

内联显示每个变体（在每个 PNG 上使用 Read 工具）进行即时预览。

**在呈现给用户之前，自我把关：** 对每个变体，问自己：*"一个人类设计师会羞于在这上面签名吗？"* 如果是，丢弃变体并重新生成。这是硬门槛。平庸的 AI 模型比没有模型更差。尴尬触发包括：紫色渐变英雄、3 列 SaaS 网格、全居中、Inter 正文文本、通用库存照片氛围、system-ui 字体、渐变 CTA 按钮、圆角一切。任何这些 = 拒绝并重新生成。

告诉用户："我已经生成了 3 个视觉方向，将你的设计系统应用到真实的[产品类型]屏幕上。在你浏览器刚打开的比较板中选择你喜欢的元素，也可以跨变体混合搭配。"

### 比较板 + 反馈循环

创建比较板并通过 HTTP 提供：

```bash
$D compare --images "$_DESIGN_DIR/variant-A.png,$_DESIGN_DIR/variant-B.png,$_DESIGN_DIR/variant-C.png" --output "$_DESIGN_DIR/design-board.html" --serve
```

此命令生成板 HTML，在随机端口上启动 HTTP 服务器，并在用户的默认浏览器中打开它。**使用 `&` 在后台运行它**，因为服务器需要在用户与板交互时继续运行。

从 stderr 输出解析板 URL。默认守护进程路径：
`BOARD_URL: http://127.0.0.1:N/boards/<id>/`（已经包含每个板的板路径；
将其用作 AskUserQuestion URL 和重新加载端点的基础）。传统 `--no-daemon` 路径发出 `SERVE_STARTED: port=XXXXX` 并在 `/` 处提供单个板，重新加载在 `/api/reload` 处——仅在外部调用者显式传递 `--no-daemon` 时相关。

**主要等待：带板 URL 的 AskUserQuestion**

板服务开始后，使用 AskUserQuestion 等待用户。包括板 URL 以便他们可以点击它：

"我已经打开了带设计变体的比较板：
<BOARD_URL> — 评分、留评论、混合搭配你喜欢的元素，完成后点击提交。告诉我你何时提交了反馈（或在这里粘贴你的偏好）。如果你在板上点击了重新生成或混合搭配，告诉我，我会生成新的变体。"

用 stderr 解析的 URL 替换 `<BOARD_URL>`（守护进程路径发出 `BOARD_URL: http://127.0.0.1:N/boards/<id>/`）。

**不要使用 AskUserQuestion 询问用户喜欢哪个变体。** 比较板就是选择器。AskUserQuestion 只是阻塞等待机制。

**AskUserQuestion 用户响应后：**

检查板 HTML 旁边的反馈文件：
- `$_DESIGN_DIR/feedback.json` — 用户点击提交时写入（最终选择）
- `$_DESIGN_DIR/feedback-pending.json` — 用户点击重新生成/混合搭配/更多此类型时写入

```bash
if [ -f "$_DESIGN_DIR/feedback.json" ]; then
  echo "SUBMIT_RECEIVED"
  cat "$_DESIGN_DIR/feedback.json"
elif [ -f "$_DESIGN_DIR/feedback-pending.json" ]; then
  echo "REGENERATE_RECEIVED"
  cat "$_DESIGN_DIR/feedback-pending.json"
  rm "$_DESIGN_DIR/feedback-pending.json"
else
  echo "NO_FEEDBACK_FILE"
fi
```

反馈 JSON 有这个结构：
```json
{
  "preferred": "A",
  "ratings": { "A": 4, "B": 3, "C": 2 },
  "comments": { "A": "Love the spacing" },
  "overall": "Go with A, bigger CTA",
  "regenerated": false
}
```

**如果找到 `feedback.json`：** 用户在板上点击了提交。
从 JSON 读取 `preferred`、`ratings`、`comments`、`overall`。继续
使用批准的变体。

**如果找到 `feedback-pending.json`：** 用户在板上点击了重新生成/混合搭配。
1. 从 JSON 读取 `regenerateAction`（`"different"`、`"match"`、`"more_like_B"`、
   `"remix"` 或自定义文本）
2. 如果 `regenerateAction` 是 `"remix"`，读取 `remixSpec`（例如 `{"layout":"A","colors":"B"}`）
3. 使用 `$D iterate` 或 `$D variants` 使用更新的简报生成新变体
4. 创建新板：`$D compare --images "..." --output "$_DESIGN_DIR/design-board.html"`
5. 在用户浏览器中重新加载板（同一标签页）— 守护进程模式下 URL 是每个板的，因此使用 `<BOARD_URL>`（来自 `BOARD_URL:` stderr 行）作为基础：
   `curl -s -X POST "${BOARD_URL}api/reload" -H 'Content-Type: application/json' -d '{"html":"$_DESIGN_DIR/design-board.html"}'`
   在 `--no-daemon` 下重新加载端点是传统端口上的 `/api/reload`；仅当调用者显式选择退出守护进程时此路径才重要。
6. 板自动刷新。**再次 AskUserQuestion** 使用相同的板 URL 等待下一轮反馈。重复直到 `feedback.json` 出现。

**如果 `NO_FEEDBACK_FILE`：** 用户直接在 AskUserQuestion 响应中键入他们的偏好，而不是使用板。他们的文本响应作为反馈。

**轮询回退：** 仅当 `$D serve` 失败时使用轮询（无可用端口）。在这种情况下，使用 Read 工具内联显示每个变体（以便用户可以看见它们），然后使用 AskUserQuestion：
"比较板服务器启动失败。我已经在上面显示了变体。你喜欢哪个？有任何反馈吗？"

**收到反馈（任何路径）后：** 输出一个清晰的确认摘要：

"从你的反馈中我理解的是：
首选：变体 [X]
评分：[列表]
你的笔记：[评论]
方向：[overall]

对吗？"

使用 AskUserQuestion 在继续前验证。

**保存批准的选择：**
```bash
echo '{"approved_variant":"<V>","feedback":"<FB>","date":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","screen":"<SCREEN>","branch":"'$(git branch --show-current 2>/dev/null)'"}' > "$_DESIGN_DIR/approved.json"
```

用户选择方向后：

- 使用 `$D extract --image "$_DESIGN_DIR/variant-<CHOSEN>.png"` 分析批准的模型并从（将填充第 6 阶段中 DESIGN.md 的）提取设计标记（色彩、字体、排版、间距）。这将设计系统建立在实际批准的视觉上，而不仅仅是文字描述。
- 如果用户想进一步迭代：`$D iterate --feedback "<user's feedback>" --output "$_DESIGN_DIR/refined.png"`

**计划模式 vs 实施模式：**
- **如果在计划模式：** 将批准的模型路径（完整 `$_DESIGN_DIR` 路径）和提取的标记添加到计划文件的"## 批准的设计方向"部分下。当计划实施时，设计系统被写入 DESIGN.md。
- **如果不在计划模式：** 直接进行到第 6 阶段并使用提取的标记编写 DESIGN.md。

### 路径 B：HTML 预览页（回退，如果 DESIGN_NOT_AVAILABLE）

生成精美的 HTML 预览页并在用户浏览器中打开。此页面是技能产生的第一个视觉产物——它应看起来漂亮。

```bash
PREVIEW_FILE="/tmp/design-consultation-preview-$(date +%s).html"
```

将预览 HTML 写入 `$PREVIEW_FILE`，然后打开：

```bash
open "$PREVIEW_FILE"
```

### 预览页要求（仅路径 B）

代理编写一个**独立的单 HTML 文件**（无框架依赖），满足：

1. 通过 `<link>` 标签从 Google Fonts（或 Bunny Fonts）**加载提议字体**
2. 各处**使用提议色彩调色板**——吃自己的设计系统狗粮
3. **显示产品名称**（不是"Lorem Ipsum"）作为英雄大标题
4. **字体示例部分：**
   - 每个字体候选以其提议角色显示（英雄大标题、正文段落、按钮标签、数据表格行）
   - 如果一个角色有多个候选则并排对比
   - 使用匹配产品的真实内容（例如，公民科技 → 政府数据示例）
5. **色彩调色板部分：**
   - 带十六进制值和名称的色板
   - 以调色板渲染的样本 UI 组件：按钮（主要、次要、幽灵）、卡片、表单输入、警报（成功、警告、错误、信息）
   - 显示对比的背景/文本色彩组合
6. **真实产品模型** — 这是让预览页强大的东西。基于第 1 阶段的项目类型，使用完整设计系统渲染 2-3 个真实页面布局：
   - **仪表板/网络应用：** 带指标的示例数据表格、侧边导航、用户头像的头部、统计卡片
   - **营销站点：** 带真实文案的英雄部分、功能亮点、推荐块、CTA
   - **设置/管理：** 带标记输入的表单、开关、下拉菜单、保存按钮
   - **认证/入门：** 带社交按钮的登录表单、品牌、输入验证状态
   - 使用产品名称、符合领域的真实内容、提议的间距/布局/边框半径。用户应在写任何代码之前（粗略地）看到他们的产品。
7. 使用 CSS 自定义属性和 JS 切换按钮的**明暗模式切换**
8. **干净、专业的布局** — 预览页本身就是技能的品味信号
9. **响应式** — 在任何屏幕宽度上都好看

页面应让用户想"噢不错，他们考虑过这个。"它通过展示产品的可能感觉来推销设计系统，而不仅仅是列出十六进制代码和字体名称。

如果 `open` 失败（无头环境），告诉用户：*"我把预览写到 [路径]——在浏览器中打开以查看字体和色彩渲染。"*

如果用户说跳过预览，直接进行到第 6 阶段。

---

## 第 6 阶段：编写 DESIGN.md 并确认

如果在第 5 阶段使用了 `$D extract`（路径 A），使用提取的标记作为 DESIGN.md 值的主要来源——色彩、字体和排版基于批准的模型而非仅文字描述。将提取的标记与第 3 阶段的提议合并（提议提供原理和上下文；提取提供精确值）。

**如果在计划模式：** 将 DESIGN.md 内容写入计划文件作为"## 提议的 DESIGN.md"部分。不要写实际文件——那在实施时发生。

**如果不在计划模式：** 将 `DESIGN.md` 写入仓库根目录，结构如下：

```markdown
# 设计系统 — [项目名称]

## 产品上下文
- **这是什么：** [1-2 句描述]
- **为谁：** [目标用户]
- **市场/行业：** [类别、同类]
- **项目类型：** [网络应用/仪表板/营销站点/编辑/内部工具]

## 美学方向
- **方向：** [名称]
- **装饰级别：** [极简/有意识/表现力]
- **情绪：** [1-2 句产品应有的感觉描述]
- **参考站点：** [URL，如果有研究]

## 排版
- **显示/大标题：** [字体名称] — [理由]
- **正文：** [字体名称] — [理由]
- **UI/标签：** [字体名称或"与正文相同"]
- **数据/表格：** [字体名称] — [理由，必须支持 tabular-nums]
- **代码：** [字体名称]
- **加载：** [CDN URL 或自托管策略]
- **比例：** [模块化比例，每个级别有特定的 px/rem 值]

## 色彩
- **方法：** [克制/平衡/表现力]
- **强调色：** [十六进制] — [代表什么、用途]
- **次级色：** [十六进制] — [用途]
- **中性色：** [暖灰/冷灰，从最浅到最深的十六进制范围]
- **语义色：** 成功[十六进制]、警告[十六进制]、错误[十六进制]、信息[十六进制]
- **暗色模式：** [策略——重新设计表面、降低饱和度 10-20%]

## 间距
- **基本单位：** [4px 或 8px]
- **密度：** [紧凑/舒适/宽敞]
- **比例：** 2xs(2) xs(4) sm(8) md(16) lg(24) xl(32) 2xl(48) 3xl(64)

## 布局
- **方法：** [网格克制/创意编辑/混合]
- **网格：** [每个断点的列数]
- **最大内容宽度：** [值]
- **边框半径：** [分层比例——例如 sm:4px、md:8px、lg:12px、full:9999px]

## 动效
- **方法：** [极简功能/有意识/表现力]
- **缓动：** 进入(ease-out) 退出(ease-in) 移动(ease-in-out)
- **时长：** 微(50-100ms) 短(150-250ms) 中(250-400ms) 长(400-700ms)

## 决策日志
| 日期 | 决策 | 理由 |
|------|------|-----------|
| [今天] | 创建初始设计系统 | 由 /design-consultation 基于[产品上下文/研究]创建 |
```

**更新 CLAUDE.md**（或创建一个如果不存在）——追加此节：

```markdown
## 设计系统
在做任何视觉或 UI 决策之前始终读取 DESIGN.md。
所有字体选择、色彩、间距和美学方向都在那里定义。
没有明确用户批准不要偏离。
在 QA 模式下，标记任何不匹配 DESIGN.md 的代码。
```

**AskUserQuestion Q-final — 显示摘要并确认：**

列出所有决定。标记任何使用代理默认值而未经用户显式确认的内容（用户应该知道他们在发布什么）。选项：
- A) 发布——写 DESIGN.md 和 CLAUDE.md
- B) 我想更改某些内容（指定什么）
- C) 从头开始

发布 DESIGN.md 后，如果会话产生了屏幕级模型或页面布局
（不仅仅是系统级令牌），建议：
"想把这个设计系统当作可工作的 Pretext 原生 HTML 吗？运行 /design-html。"

---
