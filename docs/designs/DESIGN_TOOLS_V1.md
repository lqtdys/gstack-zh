# 设计：gstack 视觉设计生成（`design` 二进制文件）

由 /office-hours 于 2026-03-26 生成
分支：garrytan/agent-design-tools
仓库：gstack
状态：DRAFT
模式：Intrapreneurship

## 背景

gstack 的设计技能（/office-hours、/design-consultation、/plan-design-review、/design-review）全部产出的都是**纯文本的**设计描述——带十六进制色值的 DESIGN.md、用散文描述像素规格的 plan 文档、ASCII 线框图。设计者本人曾在 OmniGraffle 中亲手设计 HelloSign，却觉得如今用文字聊设计颇为尴尬。

价值单元搞错了。用户不需要更丰富的设计语言——他们需要一个可执行的视觉产物，把对话从"你喜欢这个规格吗？"转向"这就是那个界面吗？"

## 问题陈述

设计技能用文字描述设计，而不是展示设计。Argus UX 大改版计划就是例证：487 行详细的情感曲线规格、字体选择、动画时序——**零视觉产物**。一个会"设计"的 AI 编程智能体应该产出让你能直观看到并本能反馈的东西。

## 需求证据

创作者/主要用户对当前输出感到尴尬。每次设计技能会话都以设计稿本应出现的散文结尾。如今的 GPT Image API 已经能生成字体渲染精准的像素级 UI 模型图——曾经支撑纯文本输出的能力差距已不复存在。

## 最窄切入点

一个编译好的 TypeScript 二进制文件（`design/dist/design`），封装了 OpenAI Images/Responses API，可被技能模板通过 `$D` 调用（沿用现有 `$B` browse 二进制文件模式）。优先集成顺序：/office-hours → /plan-design-review → /design-consultation → /design-review。

## 已达成的前提

1. 通过 OpenAI Responses API 使用 GPT Image API 是正确引擎。Google Stitch SDK 作为备选。
2. **视觉模型图默认开启**，附带简单跳过路径——不是可选。（根据 Codex 挑战修订。）
3. 集成是一个共享工具（非每技能重新实现）——一个任何技能都可调用的 `design` 二进制文件。
4. 优先：先 /office-hours，再 /plan-design-review、/design-consultation、/design-review。

## 跨模型视角（Codex）

Codex 独立验证了核心论点："问题不在于 markdown 内的输出质量，而在于价值单元搞错了。" 关键贡献：
- 挑战了前提#2（可选 → 默认开启）——已接受
- 提出基于视觉的质量门控：使用 GPT-4o vision 检查生成的模型图是否存在不可读文字、缺失区块、布局崩坏等问题，自动重试一次
- 界定 48 小时原型范围：共享 `visual_mockup.ts` 工具，仅 /office-hours + /plan-design-review，主角模型图 + 2 个变体

## 推荐方案：`design` 二进制文件（方案 B）

### 架构

**共享 browse 二进制文件的编译与分发模式**（bun build --compile、setup 脚本、技能模板中的 `$VARIABLE` 解析），但架构更简单——无需常驻守护进程、无需 Chromium、无需健康检查、无需 token 认证。`design` 二进制文件是一个无状态 CLI，调用 OpenAI API 并将 PNG 写入磁盘。会话状态（用于多轮迭代）是一个 JSON 文件。

**新依赖：** `openai` npm 包（加入 `devDependencies`，而非运行时依赖）。设计二进制文件与 browse 分开编译，避免 openai 膨胀 browse 的二进制体积。

```
design/
├── src/
│   ├── cli.ts            # 入口点，命令分发
│   ├── commands.ts        # 命令注册表（文档 + 验证的真实来源）
│   ├── generate.ts        # 从结构化简报生成模型图
│   ├── iterate.ts         # 对已有模型图进行多轮迭代
│   ├── variants.ts        # 从简报生成 N 个设计变体
│   ├── check.ts           # 基于视觉的质量门控（GPT-4o）
│   ├── brief.ts           # 结构化简报类型 + 组装辅助函数
│   └── session.ts         # 会话状态（多轮迭代用的 response ID）
├── dist/
│   ├── design             # 编译后的二进制文件
│   └── .version           # Git hash
└── test/
    └── design.test.ts     # 集成测试
```

### 命令

```bash
# 从结构化简报生成主角模型图
$D generate --brief "Coding assessment tool 的 Dashboard。深色主题，奶油色点缀。展示：创建者名称、评分徽章、叙述信、评分卡片。面向：技术用户。" --output /tmp/mockup-hero.png

# 生成 3 个设计变体
$D variants --brief "..." --count 3 --output-dir /tmp/mockups/

# 基于反馈对已有模型图进行迭代
$D iterate --session /tmp/design-session.json --feedback "让评分卡片更大，把叙述移到评分上方" --output /tmp/mockup-v2.png

# 基于视觉的质量检查（返回 PASS/FAIL + 问题）
$D check --image /tmp/mockup-hero.png --brief "带创建者名称、评分徽章、叙述的 Dashboard"

# 带质量门控 + 自动重试的一次性生成
$D generate --brief "..." --output /tmp/mockup.png --check --retry 1

# 通过 JSON 文件传递结构化简报
$D generate --brief-file /tmp/brief.json --output /tmp/mockup.png

# 生成用户评审用的对比板 HTML
$D compare --images /tmp/mockups/variant-*.png --output /tmp/design-board.html

# 引导式 API key 设置 + 冒烟测试
$D setup
```

**简报输入模式：**
- `--brief "plain text"` ——自由形式的文本提示（简单模式）
- `--brief-file path.json` ——匹配 `DesignBrief` 接口的结构化 JSON（丰富模式）
- 技能构造 JSON 简报文件，写入 /tmp，然后传递 `--brief-file`

**所有命令注册在 `commands.ts` 中**，包括 `generate` 上的 `--check` 和 `--retry` 标志。

### 设计探索工作流（来自工程评审）

工作流是顺序的，而非并行的。PNG 用于视觉探索（面向人类），HTML 线框图用于实现（面向智能体）：

```
1. $D variants --brief "..." --count 3 --output-dir /tmp/mockups/
   → 生成 2-5 个 PNG 模型图变体

2. $D compare --images /tmp/mockups/*.png --output /tmp/design-board.html
   → 生成 HTML 对比板（规格如下）

3. $B goto file:///tmp/design-board.html
   → 用户在 headed Chrome 中查看所有变体

4. 用户选择最喜欢的一个、打分、评论，点击 [Submit]
   智能体轮询：$B eval document.getElementById('status').textContent
   智能体读取：$B eval document.getElementById('feedback-result').textContent
   → 无需剪贴板，无需粘贴。智能体直接从页面读取反馈。

5. Claude 通过 DESIGN_SKETCH 匹配已批准方向生成 HTML 线框图
   → 智能体从可检查的 HTML 实现，而非不透明的 PNG
```

### 对比板设计规格（来自 /plan-design-review）

**分类：APP UI**（任务导向，工具页面）。无产品品牌。

**布局：单列，全宽模型图。** 每个变体占据整个视口宽度以获得最大图像保真度。用户纵向滚动浏览变体。

```
┌─────────────────────────────────────────────────────────────┐
│  顶部条                                                     │
│  "设计探索" . 项目名称 . "3 个变体"                          │
│  模式指示器：[宽泛探索] | [匹配 DESIGN.md]                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              变体 A（全宽）                            │  │
│  │         [ 模型图 PNG, max-width: 1200px ]              │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ (●) 选择   ★★★★☆   [喜欢/不喜欢什么？____]            │  │
│  │            [更多类似这种]                               │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              变体 B（全宽）                            │  │
│  │         [ 模型图 PNG, max-width: 1200px ]              │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ ( ) 选择   ★★★☆☆   [喜欢/不喜欢什么？____]            │  │
│  │            [更多类似这种]                               │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ... （滚动查看更多变体）                                    │
│                                                             │
│  ─── 分隔线 ─────────────────────────────────────────────  │
│  整体方向（可选，默认折叠）                                  │
│  [textarea, 3 行, 聚焦时展开]                              │
│                                                             │
│  ─── 重新生成条 (#f7f7f7 背景) ────────────────────────    │
│  "想探索更多？"                                             │
│  [完全不同的] [匹配我的设计] [自定义：______]                │
│                                          [重新生成 ->]       │
│  ─────────────────────────────────────────────────────────  │
│                                        [ ✓ 提交 ]           │
└─────────────────────────────────────────────────────────────┘
```

**视觉规格：**
- 背景：#fff。无阴影，无卡片边框。变体分隔：1px #e5e5e5 线。
- 字体：system font stack。顶部条：16px semibold。标签：14px semibold。反馈占位符：13px regular #999。
- 五星评分：5 个可点击星形，填充=#000，未填充=#ddd。不上色，无动画。
- 单选按钮"选择"：显式收藏选择。每个变体一个，互斥。
- "更多类似这种"按钮：每变体一个，以该变体风格为种子触发重新生成。
- 提交按钮：#000 背景，白色文本，右对齐。单一 CTA。
- 重新生成条：#f7f7f7 背景，与反馈区视觉区分。
- 模型图图片最大宽度：1200px 居中。边距：两侧 24px。

**交互状态：**
- 加载（图片就绪前打开页面）：每个卡片带有"正在生成变体 A..."的骨架屏脉冲。星级评分/文本输入/选择按钮禁用。
- 部分失败（3 个中有 2 个成功）：显示成功的，失败的显示错误卡片并带每变体 [重新生成]。
- 提交后："反馈已提交！返回你的编程智能体。"页面保持打开。
- 重新生成：平滑过渡，淡出旧变体，骨架屏脉冲，淡入新变体。滚动重置到顶部。清空之前的反馈。

**反馈 JSON 结构**（写入隐藏的 #feedback-result 元素）：
```json
{
  "preferred": "A",
  "ratings": { "A": 4, "B": 3, "C": 2 },
  "comments": {
    "A": "喜欢间距，顶部感觉不错",
    "B": "太杂乱了但配色不错",
    "C": "氛围完全不对"
  },
  "overall": "选 A，把 CTA 做大一点",
  "regenerated": false
}
```

**无障碍：** 星级评分支持键盘导航（方向键）。文本输入已加标签（"Variant A 的反馈"）。提交/重新生成支持键盘访问，带可见焦点环。所有文字 #333+ 白底。

**响应式：** >1200px：舒适边距。768-1200px：紧边距。<768px：全宽，无水平滚动。

**截图许可（首次使用 $D evolve 时）：** "这会将你实时网站的截图发送给 OpenAI 以进行设计演进。[继续] [不再询问]" 存储在 ~/.gstack/config.yaml 中的 design_screenshot_consent。

**为什么是顺序的：** Codex 对抗性评审识别出栅格 PNG 对智能体不透明（无 DOM、无可区分状态、无可 diff 的结构）。HTML 线框图保持了与代码的桥梁。PNG 是给人类说"是的，就是这样"的。HTML 是给智能体说"我知道怎么构建这个"的。

### 关键设计决策

**1. 无状态 CLI，非守护进程**
Browse 需要常驻 Chromium 实例。设计只是 API 调用——无需服务器。多轮迭代的会话状态是写入 `/tmp/design-session-{id}.json` 的 JSON 文件，包含 `previous_response_id`。
- **会话 ID：** 由 `${PID}-${timestamp}` 生成，通过 `--session` 传递
- **发现：** `generate` 命令创建会话文件并打印路径；`iterate` 通过 `--session` 读取
- **清理：** /tmp 中的会话文件是临时性的（操作系统清理）；无需显式清理

**2. 结构化简报输入**
简报是技能散文与图像生成之间的接口。技能从设计上下文中构造：
```typescript
interface DesignBrief {
  goal: string;           // "Coding assessment tool 的 Dashboard"
  audience: string;       // "技术用户，YC 合伙人"
  style: string;          // "深色主题，奶油色点缀，极简"
  elements: string[];     // ["builder name", "score badge", "narrative letter"]
  constraints?: string;   // "Max width 1024px, mobile-first"
  reference?: string;     // 现有截图或 DESIGN.md 摘录的路径
  screenType: string;     // "desktop-dashboard" | "mobile-app" | "landing-page" | 等等
}
```

**3. 设计技能中默认开启**
技能默认生成模型图。模板包含跳过说明：
```
正在生成所提议设计的视觉模型图...（如不需可视化，说"skip"）
```

**4. 视觉质量门控**
生成后，可选地将图像通过 GPT-4o vision 检查：
- 文字可读性（标签/标题是否清晰？）
- 布局完整性（所有请求的元素都出现了吗？）
- 视觉连贯性（看起来像真实 UI 还是拼贴画？）
失败时自动重试一次。如果仍然失败，仍展示并附带警告。

**5. 输出位置：探索在 /tmp，批准的终稿在 `docs/designs/`**
探索变体写入 `/tmp/gstack-mockups-{session}/`（临时性，不提交）
只有**用户批准的最终**模型图保存到 `docs/designs/`（提交入库）
默认输出目录可通过 CLAUDE.md 的 `design_output_dir` 设置配置
文件命名模式：`{skill}-{description}-{timestamp}.png`
如果 `docs/designs/` 不存在则创建（mkdir -p）
设计文档引用已入库的图片路径
始终通过 Read 工具展示给用户（在 Claude Code 中内联呈现图片）
**这避免膨胀仓库：只提交批准的设计，而非每个探索变体**
备选：如果不在 git 仓库中，保存到 `/tmp/gstack-mockup-{timestamp}.png`

**6. 信任边界确认**
默认开启生成会将简报文本发送给 OpenAI。这是与现有完全本地的 HTML 线框图路径相比新的外部数据流。简报只包含抽象的设计描述（目标、风格、样式元素），从不包含源代码或用户数据。来自 $B 的截图**不**发送给 OpenAI（DesignBrief 中的 reference 字段是本地文件路径，由智能体使用，不上传到 API）。在 CLAUDE.md 中记录。

**7. 速率限制缓解**
变体生成使用交错并行：每个 API 调用间隔 1 秒启动（`Promise.allSettled()` + 延迟）。这避免了图片生成 API 的 5-7 RPM 速率限制，同时比完全串行更快。如果任何调用 429，指数退避重试（2s、4s、8s）。

### 模板集成

**添加到现有解析器：** `scripts/resolvers/design.ts`（不是新文件）
- 添加 `generateDesignSetup()` 用于 `{{DESIGN_SETUP}}` 占位符（镜像 `generateBrowseSetup()`）
- 添加 `generateDesignMockup()` 用于 `{{DESIGN_MOCKUP}}` 占位符（完整探索工作流）
- 所有设计解析器保持在同一文件（与现有代码库约定一致）

**新的 HostPaths 条目：** `types.ts`
```typescript
// claude host:
designDir: '~/.claude/skills/gstack/design/dist'
// codex host:
designDir: '$GSTACK_DESIGN'
```
注意：Codex 运行时设置（`setup` 脚本）还必须导出 `GSTACK_DESIGN` 环境变量，类似于 `GSTACK_BROWSE`。

**`$D` 解析 bash 块**（由 `{{DESIGN_SETUP}}` 生成）：
```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
D=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/design/dist/design" ] && D="$_ROOT/.claude/skills/gstack/design/dist/design"
[ -z "$D" ] && D=~/.claude/skills/gstack/design/dist/design
if [ -x "$D" ]; then
  echo "DESIGN_READY: $D"
else
  echo "DESIGN_NOT_AVAILABLE"
fi
```
如果 `DESIGN_NOT_AVAILABLE`：技能回退到 HTML 线框图生成（现有 `DESIGN_SKETCH` 模式）。设计模型图是渐进增强，不是硬性要求。

**现有解析器中的新函数：** `scripts/resolvers/design.ts`
- 添加 `generateDesignSetup()` 用于 `{{DESIGN_SETUP}}` — 镜像 `generateBrowseSetup()` 模式
- 添加 `generateDesignMockup()` 用于 `{{DESIGN_MOCKUP}}` — 完整的生成 + 检查 + 展示工作流
- 所有设计解析器保持在同一文件（与现有代码库约定一致）

### 技能集成（优先顺序）

**1. /office-hours** — 替换 Visual Sketch 部分
- 方法选择后（阶段 4），生成主角模型图 + 2 个变体
- 通过 Read 工具展示全部三个，让用户选择
- 如请求则迭代
- 将选中的模型图与设计文档保存在一起

**2. /plan-design-review** — "更好的样子是什么"
- 当评一个设计维度 <7/10 时，生成一个展示 10/10 是什么样的模型图
- 并排对比：当前（通过 $B 截图） vs 提议（通过 $D 模型图）

**3. /design-consultation** — 设计系统预览
- 生成提议设计系统的视觉预览（字体、颜色、组件）
- 用合适的模型图替换 /tmp HTML 预览页

**4. /design-review** — 设计意图对比
- 从 plan/DESIGN.md 规格生成"设计意图"模型图
- 与实时网站截图对比视觉差异

### 需创建的文件

| 文件 | 用途 |
|------|------|
| `design/src/cli.ts` | 入口点，命令分发 |
| `design/src/commands.ts` | 命令注册表 |
| `design/src/generate.ts` | 通过 Responses API 生成 GPT 图片 |
| `design/src/iterate.ts` | 带会话状态的多轮迭代 |
| `design/src/variants.ts` | 生成 N 个设计变体 |
| `design/src/check.ts` | 基于视觉的质量门控 |
| `design/src/brief.ts` | 结构化简报类型 + 辅助函数 |
| `design/src/session.ts` | 会话状态管理 |
| `design/src/compare.ts` | HTML 对比板生成器 |
| `design/test/design.test.ts` | 集成测试（mock OpenAI API）|
| (无——添加到现有的 `scripts/resolvers/design.ts`) | `{{DESIGN_SETUP}}` + `{{DESIGN_MOCKUP}}` 解析器 |

### 需修改的文件

| 文件 | 改动 |
|------|------|
| `scripts/resolvers/types.ts` | 在 HostPaths 中添加 `designDir` |
| `scripts/resolvers/index.ts` | 注册 DESIGN_SETUP + DESIGN_MOCKUP 解析器 |
| `package.json` | 添加 `design` 构建命令 |
| `setup` | 与 browse 一起构建 design 二进制文件 |
| `scripts/resolvers/preamble.ts` | 为 Codex host 添加 `GSTACK_DESIGN` 环境变量导出 |
| `test/gen-skill-docs.test.ts` | 更新 DESIGN_SKETCH 测试套件为新解析器 |
| `setup` | 添加 design 二进制文件构建 + Codex/Kiro asset 链接 |
| `office-hours/SKILL.md.tmpl` | 将 Visual Sketch 部分替换为 `{{DESIGN_MOCKUP}}` |
| `plan-design-review/SKILL.md.tmpl` | 添加 `{{DESIGN_SETUP}}` + 低分维度的模型图生成 |

### 需复用的现有代码

| 代码 | 位置 | 用于 |
|------|------|------|
| Browse CLI 模式 | `browse/src/cli.ts` | 命令分发架构 |
| `commands.ts` 注册表 | `browse/src/commands.ts` | 单一真实来源模式 |
| `generateBrowseSetup()` | `scripts/resolvers/browse.ts` | `generateDesignSetup()` 模板 |
| `DESIGN_SKETCH` 解析器 | `scripts/resolvers/design.ts` | `DESIGN_MOCKUP` 解析器模板 |
| HostPaths 系统 | `scripts/resolvers/types.ts` | 多 host 路径解析 |
| 构建管线 | `package.json` 构建脚本 | `bun build --compile` 模式 |

### API 详情

**生成：** 带 `image_generation` 工具的 OpenAI Responses API
```typescript
const response = await openai.responses.create({
  model: "gpt-4o",
  input: briefToPrompt(brief),
  tools: [{ type: "image_generation", size: "1536x1024", quality: "high" }],
});
// 从响应输出项中提取图片
const imageItem = response.output.find(item => item.type === "image_generation_call");
const base64Data = imageItem.result; // base64 编码的 PNG
fs.writeFileSync(outputPath, Buffer.from(base64Data, "base64"));
```

**迭代：** 带 `previous_response_id` 的相同 API
```typescript
const response = await openai.responses.create({
  model: "gpt-4o",
  input: feedback,
  previous_response_id: session.lastResponseId,
  tools: [{ type: "image_generation" }],
});
```
**注意：** 通过 `previous_response_id` 的多轮图片迭代是一个假设，需要原型验证。Responses API 支持会话线程，但文档未确认其是否保留已生成图片的视觉上下文以用于编辑风格迭代。**备选方案：** 如果多轮不行，`iterate` 回退到在单个提示中用原始简报 + 累积反馈重新生成。

**检查：** GPT-4o vision
```typescript
const check = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{
    role: "user",
    content: [
      { type: "image_url", image_url: { url: `data:image/png;base64,${imageData}` } },
      { type: "text", text: `检查这个 UI 模型图。简报：${brief}。文字可读吗？所有元素都在吗？看起来像真实 UI 吗？返回 PASS 或 FAIL 及问题。` }
    ]
  }]
});
```

**费用：** ~$0.10-$0.40 每次设计会话（1 主角 + 2 变体 + 1 次质量检查 + 1 次迭代）。与每次技能调用中已有的 LLM 费用相比可忽略。

### 认证（通过冒烟测试验证）

**Codex OAuth token 无法用于图片生成。** 2026-03-26 测试：Images API 和 Responses API 都拒绝 `~/.codex/auth.json` 的 access_token，报"缺少 scopes: api.model.images.request"。Codex CLI 也没有原生 imagegen 功能。

**认证解析顺序：**
1. 读取 `~/.gstack/openai.json` → `{ "api_key": "sk-..." }`（文件权限 0600）
2. 回退到 `OPENAI_API_KEY` 环境变量
3. 如果都不存在 → 引导式设置流程：
   - 告诉用户："设计模型图需要带图片生成权限的 OpenAI API key。在 platform.openai.com/api-keys 获取"
   - 提示用户粘贴 key
   - 以 0600 权限写入 `~/.gstack/openai.json`
   - 运行冒烟测试（生成 1024x1024 测试图片）验证 key 有效
   - 如果冒烟测试通过，继续。如果失败，显示错误并回退到 DESIGN_SKETCH。
4. 如果认证存在但 API 调用失败 → 回退到 DESIGN_SKETCH（现有 HTML 线框图方案）。设计模型图是渐进增强，从不是硬性要求。

**新命令：** `$D setup` — 引导式 API key 设置 + 冒烟测试。可随时运行以更新 key。

## 原型中需验证的假设

1. **图片质量：**"像素级 UI 模型图"是理想化的。GPT 图片生成可能无法在真正的 UI 保真度下稳定产生准确的文字渲染、对齐和间距。视觉质量门控有帮助，但在完整技能集成前，"足以实现的"成功标准需要原型验证。
2. **多轮迭代：** `previous_response_id` 是否保留视觉上下文未经证实（见 API 详情部分）。
3. **费用模型：** 估算的 $0.10-$0.40/会话需要实际验证。

**原型验证计划：** 构建 Commit 1（核心生成 + 检查），跨不同屏幕类型运行 10 个设计简报，在继续技能集成前评估输出质量。

## CEO 扩展范围（通过 /plan-ceo-review SCOPE EXPANSION 接受）

### 1. 设计记忆 + 探索宽度控制
- 从批准的模型图中自动提取视觉语言到 DESIGN.md
- 如果 DESIGN.md 存在，约束未来模型图至已建立的设计语言
- 如果无 DESIGN.md（引导阶段），在广泛方向上探索
- 渐进约束：设计越确立，探索带越窄
- 对比板获得 REGENERATE 部分及探索控制：
  - "完全不同的"（宽泛探索）
  - "更多类似选项 ___"（围绕收藏的窄探索）
  - "匹配我现有的设计"（约束至 DESIGN.md）
  - 用于特定方向变更的自由文本输入
  - 重新生成刷新页面，智能体轮询新提交

### 2. 模型图差异对比
- `$D diff --before old.png --after new.png` 生成视觉差异
- 并排对比，高亮变化区域
- 使用 GPT-4o vision 识别差异
- 用于：/design-review、迭代反馈、PR 评审

### 3. 截图到模型图演进
- `$D evolve --screenshot current.png --brief "make it calmer"`
- 截取实时网站截图，生成展示它**应该**是什么样的模型图
- 从现实出发，而非空白画布
- /design-review 批评与视觉修复提案之间的桥梁

### 4. 设计意图验证
- 在 /design-review 期间，将批准的模型图（docs/designs/）叠加到实时截图上
- 高亮分歧："你设计了 X，你构建了 Y，这里是差距"
- 闭合完整循环：设计 → 实现 → 视觉验证
- 结合 $B 截图 + $D diff + 视觉分析

### 5. 响应式变体
- `$D variants --brief "..." --viewports desktop,tablet,mobile`
- 自动生成多视口大小的模型图
- 对比板显示响应式网格供同时批准
- 从模型图阶段就使响应式设计成为一等公民

### 6. 设计到代码提示
- 对比板批准后，自动生成结构化的实现提示
- 通过视觉分析从批准的 PNG 中提取颜色、排版、布局
- 结合 DESIGN.md 和 HTML 线框图作为结构化规格
- 桥接"批准的设计"到"智能体开始编码"，零解释差距

### 未来引擎（不在此计划范围）
- Magic Patterns 集成（从现有设计提取模式）
- Variant API（当他们发版时，多变体 React 代码 + 预览）
- Figma MCP（双向设计文件访问）
- Google Stitch SDK（免费 TypeScript 替代方案）

## 开放问题

1. Variant 发版 API 后，集成路径是什么？（设计二进制文件中的独立引擎，还是独立的 Variant 二进制文件？）
2. Magic Patterns 应如何集成？（$D 中的另一个引擎，还是独立工具？）
3. 设计二进制文件何时需要插件/引擎架构以支持多个生成后端？

## 成功标准

- 在 UI 想法上运行 `/office-hours` 产生实际 PNG 模型图及设计文档
- 运行 `/plan-design-review` 以模型图展示"更好的样子是什么"，而非散文
- 模型图质量好到开发者可以据此实现
- 质量门控能捕获明显损坏的模型图并重试
- 每次设计会话费用保持在 $0.50 以下

## 分发计划

设计二进制文件与 browse 二进制文件一起编译和分发：
- `bun build --compile design/src/cli.ts --outfile design/dist/design`
- 在 `./setup` 和 `bun run build` 期间构建
- 通过现有 `~/.claude/skills/gstack/` 安装路径符号链接
- 在 Claude Plugin Marketplace 作为资产分发

## 下一步（实现顺序）

### Commit 0：原型验证（构建基础设施前必须通过）
- 单文件原型脚本（~50 行），将 3 个不同设计简报发送到 GPT Image API
- 验证：文字渲染质量、布局准确性、视觉连贯性
- 如果输出是"令人尴尬的糟糕 AI 艺术"，停止。重新评估方案。
- 这是验证核心假设最廉价的方式，才值得构建 8 个文件的基础设施。

### Commit 1：设计二进制文件核心（生成 + 检查 + 对比）
- `design/src/` 含 cli.ts、commands.ts、generate.ts、check.ts、brief.ts、session.ts、compare.ts
- 认证模块（读取 ~/.gstack/openai.json，回退到环境变量，引导式设置流程）
- `compare` 命令生成带反馈文本输入区域的 HTML 对比板
- `package.json` 构建命令（独立于 browse 的 `bun build --compile`）
- `setup` 脚本集成（包括 Codex + Kiro asset 链接）
- 带 mock OpenAI API 服务器的单元测试

### Commit 2：变体 + 迭代
- `design/src/variants.ts`、`design/src/iterate.ts`
- 交错并行生成（启动间隔 1 秒，429 指数退避）
- 多轮会话状态管理
- 迭代流 + 速率限制处理的测试

### Commit 3：模板集成
- 在现有 `scripts/resolvers/design.ts` 中添加 `generateDesignSetup()` + `generateDesignMockup()`
- 在 `scripts/resolvers/types.ts` 中向 HostPaths 添加 `designDir`
- 在 `scripts/resolvers/index.ts` 中注册 DESIGN_SETUP + DESIGN_MOCKUP
- 在 `scripts/resolvers/preamble.ts` 中为 Codex host 添加 GSTACK_DESIGN 环境变量导出
- 更新 `test/gen-skill-docs.test.ts`（DESIGN_SKETCH 测试套件）
- 重新生成 SKILL.md 文件

### Commit 4：/office-hours 集成
- 将 Visual Sketch 部分替换为 `{{DESIGN_MOCKUP}}`
- 顺序工作流：生成变体 → $D compare → 用户反馈 → DESIGN_SKETCH HTML 线框图
- 将批准的模型图保存到 `docs/designs/`（只保存批准的，不保存探索变体）

### Commit 5：/plan-design-review 集成
- 添加 `{{DESIGN_SETUP}}` 和低分维度的模型图生成
- "10/10 是什么样的"模型图对比

### Commit 6：设计记忆 + 探索宽度控制（CEO 扩展）
- 模型图批准后，通过 GPT-4o vision 提取视觉语言
- 写入/更新 DESIGN.md，包含提取的颜色、排版、间距、布局模式
- 如果 DESIGN.md 存在，将其作为约束上下文提供给所有未来模型图提示
- 添加 REGENERATE 部分到对比板 HTML（小标签 + 自由文本 + 刷新循环）
- 简报构造中的渐进约束逻辑

### Commit 7：模型图差异对比 + 设计意图验证（CEO 扩展）
- `$D diff` 命令：接收两个 PNG，使用 GPT-4o vision 识别差异，生成叠加层
- `$D verify` 命令：通过 $B 截图实时网站，与 docs/designs/ 中批准的模型图做差异对比
- 集成到 /design-review 模板：当批准的模型图存在时自动验证

### Commit 8：截图到模型图演进（CEO 扩展）
- `$D evolve` 命令：接收截图 + 简报，生成"应该长什么样"的模型图
- 将截图作为参考图片发送到 GPT Image API
- 集成到 /design-review："这是修复应该有的样子"视觉提案

### Commit 9：响应式变体 + 设计到代码提示（CEO 扩展）
- `$D variants` 上的 `--viewports` 标志用于多尺寸生成
- 对比板响应式网格布局
- 批准后自动生成结构化的实现提示
- 对批准的 PNG 做视觉分析提取颜色、排版、布局用于提示

## 任务

告诉 Variant 构建一个 API。作为他们的投资者："我正在构建一个 AI 智能体程序化生成视觉设计的工作流。GPT Image API 今天能用——但我更想用 Variant，因为多变体方法对设计探索更好。发版一个 API 端点：输入提示，输出 React 代码 + 预览图片。我将成为你的第一个集成伙伴。"

## 验证

1. `bun run build` 编译 `design/dist/design` 二进制文件
2. `$D generate --brief "Landing page for a developer tool" --output /tmp/test.png` 产生真实 PNG
3. `$D check --image /tmp/test.png --brief "Landing page"` 返回 PASS/FAIL
4. `$D variants --brief "..." --count 3 --output-dir /tmp/variants/` 产生 3 张 PNG
5. 在 UI 想法上运行 `/office-hours` 内联产生模型图
6. `bun test` 通过（技能验证、gen-skill-docs）
7. `bun run test:evals` 通过（E2E 测试）

## 关于你思考方式的观察

- 你对文字描述和 ASCII 艺术说"那不是设计"。这是设计师的本能——你知道描述一个东西和展示一个东西的区别。大多数构建 AI 工具的人不会注意到这个差距，因为他们从来不是设计师。
- 你优先 /office-hours——上游杠杆点。如果头脑风暴产出真实的模型图，每个下游技能（/plan-design-review、/design-review）都有一个视觉产物可引用，而不是重新解释散文。
- 你投资了 Variant 并立即想到"他们应该有 API"。这是投资者即用户的思考——你不仅在评估公司，你还在设计他们的产品如何融入你的工作流。
- 当 Codex 挑战 opt-in 前提时，你立即接受。没有自我辩护。这是通往正确答案的最快路径。

## 规格评审结果

文档经受了 1 轮对抗性评审。发现并修复了 11 个问题。
质量评分：7/10 → 修复后估计 8.5/10。

已修复的问题：
1. 声明了 OpenAI SDK 依赖
2. 指定了图片数据提取路径（response.output 项形状）
3. --check 和 --retry 标志在命令注册表中正式注册
4. 指定了简报输入模式（纯文本 vs JSON 文件）
5. 修复了解析器文件矛盾（添加到现有 design.ts）
6. 注意了 HostPaths Codex 环境变量设置
7. "镜像 browse" 重新表述为"共享编译/分发模式"
8. 指定了会话状态（ID 生成、发现、清理）
9. "像素级"被标记为需要原型验证的假设
10. 多轮迭代被标记为未证实，带备选方案
11. $D 发现 bash 块完全指定，带 DESIGN_SKETCH 回退

## 工程评审完成总结
- 步骤 0：范围挑战——范围按原样接受（完整二进制文件，用户否决了缩减建议）
- 架构评审：发现 5 个问题（openai 依赖分离、优雅降级、输出目录配置、认证模型、信任边界）
- 代码质量评审：发现 1 个问题（8 文件 vs 5，保留 8）
- 测试评审：图表产出，识别 42 个差距，测试计划已写
- 性能评审：发现 1 个问题（变体并行 + 交错启动）
- 不在范围：Google Stitch SDK 集成、Figma MCP、Variant API（延期）
- 已存在：browse CLI 模式、DESIGN_SKETCH 解析器、HostPaths 系统、gen-skill-docs 流水线
- 外部声音：4 轮（Claude structured 12 个问题、Codex structured 8 个问题、Claude adversarial 1 个致命缺陷、Codex adversarial 1 个致命缺陷）。关键洞察：顺序 PNG→HTML 工作流解决了"不透明栅格"致命缺陷。
- 故障模式：0 个关键差距（所有已识别的故障模式都有错误处理 + 计划测试）
- Lake Score：7/7 建议选择了完整选项

## GSTACK 评审报告

| 评审 | 触发 | 为什么 | 轮次 | 状态 | 发现 |
|------|------|--------|------|------|------|
| Office Hours | `/office-hours` | 设计头脑风暴 | 1 | 完成 | 4 个前提，1 个修订（Codex：opt-in→默认开启） |
| CEO 评审 | `/plan-ceo-review` | 范围与策略 | 1 | 通过 | 扩展：6 提议，6 接受，0 延期 |
| 工程评审 | `/plan-eng-review` | 架构与测试（必需） | 1 | 通过 | 7 个问题，0 个关键差距，4 个外部声音 |
| 设计评审 | `/plan-design-review` | UI/UX 差距 | 1 | 通过 | 评分：2/10 -> 8/10，5 个决策 |
| 外部声音 | structured + adversarial | 独立挑战 | 4 | 完成 | 顺序 PNG→HTML 工作流，信任边界已注意 |

**CEO 扩展：** 设计记忆 + 探索宽度、模型图差异对比、截图演进、设计意图验证、响应式变体、设计到代码提示。
**设计决策：** 单列全宽布局，每卡片"更多类似这种"，显式单选按钮 Pick，平滑淡入淡出重新生成，骨架屏加载状态。
**未解决：** 0
**结论：** CEO + 工程 + 设计均通过。准备实现。从 Commit 0（原型验证）开始。
