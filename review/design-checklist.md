# 设计审核检查清单（精简版）

> **DESIGN_METHODOLOGY 的子集** — 在此处添加条目时，也请更新 `scripts/gen-skill-docs.ts` 中的 `generateDesignMethodology()`，反之亦然。

## 说明

此检查清单适用于 **差异中的源代码** — 而非渲染输出。读取每个变更的前端文件（完整文件，不仅仅是差异片段）并标记反模式。

**触发条件：** 仅当差异触及前端文件时运行此检查清单。使用 `gstack-diff-scope` 检测：

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

如果 `SCOPE_FRONTEND=false`，则静默跳过整个设计审核。

**DESIGN.md 校准：** 如果仓库根目录存在 `DESIGN.md` 或 `design-system.md`，请先读取。所有发现都对照项目声明的设计系统进行校准。DESIGN.md 中明确认可的图案**不**标记。如果不存在 DESIGN.md，则使用通用设计原则。

---

## 置信度层级

每个条目都标注了检测置信度级别：

- **[HIGH]** — 可通过 grep/模式匹配可靠地检测。发现具有决定性。

- **[MEDIUM]** — 可通过模式聚合或启发式方法检测。标记为发现但预计会有一定的误报。

- **[LOW]** — 需要理解视觉意图。呈现为：「可能的问题 — 请目视验证或运行 /design-review。」

---

## 分类

**AUTO-FIX**（仅限机械性 CSS 修复 — HIGH 置信度，无需设计判断）：

- `outline: none` 无替代方案 → 添加 `outline: revert` 或 `&:focus-visible { outline: 2px solid currentColor; }`

- 新 CSS 中使用 `!important` → 移除并修复特异性

- 正文文本 `font-size` < 16px → 提升到 16px

**ASK**（其他所有情况 — 需要设计判断）：

- 所有 AI slop 发现、排版结构、间距选择、交互状态缺失、DESIGN.md 违规

**LOW 置信度项目** → 呈现为「可能：[描述]。请目视验证或运行 /design-review。」绝不自动修复。

---

## 输出格式

```
设计审核：N 个问题（X 可自动修复，Y 需要输入，Z 可能）

**自动已修复：**
- [文件:行号] 问题 → 应用的修复

**需要输入：**
- [文件:行号] 问题描述
  建议修复：建议的修复方案

**可能（请目视验证）：**
- [文件:行号] 可能的问题 — 请使用 /design-review 验证
```

可选：`test_stub` — 使用此项目的测试框架，为此次发现的测试骨架代码。

如果未发现问题：`设计审核：未发现问题。`

如果未变更前端文件：静默跳过，无输出。

---

## 分类

### 1. AI Slop 检测（6 项）— 最高优先级

这些是 AI 生成 UI 的 tell-tale 迹象，任何受尊敬的工作室的设计师都不会发布的产品。

- **[MEDIUM]** 紫色/紫罗兰/靛蓝渐变背景或蓝到紫的颜色方案。查找 `linear-gradient`，值在 `#6366f1`–`#8b5cf6` 范围内，或解析为紫色/紫罗兰色的 CSS 自定义属性。

- **[LOW]** 三列功能网格：图标在彩色圆圈内 + 粗体标题 + 2 行描述，对称重复 3 次。查找网格/flex 容器，恰好包含 3 个子元素，每个子元素包含圆形元素 + 标题 + 段落。

- **[LOW]** 将图标放在彩色圆圈内作为装饰。查找 `border-radius: 50%` + 用作图标装饰容器的背景色的元素。

- **[HIGH]** 全部居中：所有标题、描述和卡片都用了 `text-align: center`。grep `text-align: center` 的密度 — 如果 >60% 的文本容器使用居中对齐，标记它。

- **[MEDIUM]** 统一的气泡状圆角：每个元素都应用了相同的大半径（16px+）。聚合 `border-radius` 值 — 如果 >80% 使用相同值 ≥16px，标记它。

- **[MEDIUM]** 通用英雄文案：「欢迎来到 [X]」、「释放……的力量」、「您的一站式解决方案」、「彻底改变您的工作流」。grep HTML/JSX 内容中这些模式。

### 2. 排版（4 项）

- **[HIGH]** 正文文本 `font-size` < 16px。grep `font-size` 声明，针对 `body`、`p`、`.text` 或基本样式。低于 16px（或当基础为 16px 时为 1rem）的值会被标记。

- **[HIGH]** 差异中引入了超过 3 种字体族。计算不同的 `font-family` 声明。如果变更文件中出现超过 3 种唯一的字体族，标记它。

- **[HIGH]** 标题层级跳过：`h1` 后面直接跟 `h3`，同一文件/组件中没有 `h2`。检查 HTML/JSX 中的标题标签。

- **[HIGH]** 禁用字体：Papyrus、Comic Sans、Lobster、Impact、Jokerman。grep `font-family` 中这些名称。

### 3. 间距和布局（4 项）

- **[MEDIUM]** 不在 DESIGN.md 指定间距比例上的任意间距值，当 DESIGN.md 指定了间距比例时。检查 `margin`、`padding`、`gap` 值与声明比例的一致性。仅当 DESIGN.md 定义了比例时才标记。

- **[MEDIUM]** 没有响应式处理的固定宽度：容器上设置 `width: NNNpx` 但没有 `max-width` 或 `@media` 断点。存在移动端水平滚动的风险。

- **[MEDIUM]** 文本容器缺少 `max-width`：正文文本或段落容器没有 `max-width`，允许行超过 75 个字符。在文本包装器中检查 `max-width`。

- **[HIGH]** 新 CSS 规则中使用 `!important`。grep 新增行中的 `!important`。几乎总是应该正确修复的特异性逃生舱。

### 4. 交互状态（3 项）

- **[MEDIUM]** 交互元素（按钮、链接、输入框）缺少悬停/焦点状态。检查新增交互元素样式是否存在 `:hover` 和 `:focus` 或 `:focus-visible` 伪类。

- **[HIGH]** `outline: none` 或 `outline: 0` 无替代焦点指示器。grep `outline:\s*none` 或 `outline:\s*0`。这会移除键盘无障碍访问。

- **[LOW]** 交互元素的触摸目标 < 44px。检查按钮和链接的 `min-height`/`min-width`/`padding`。需要从多个属性计算有效尺寸 — 仅从代码的低置信度。

### 5. DESIGN.md 违规（3 项，条件性）

仅在存在 `DESIGN.md` 或 `design-system.md` 时应用：

- **[MEDIUM]** 颜色不在声明调色板中。对照 DESIGN.md 中定义的比较变更 CSS 中的颜色值。

- **[MEDIUM]** 字体不在声明排版部分中。对照 DESIGN.md 的字体列表比较 `font-family` 值。

- **[MEDIUM]** 间距值超出声明范围。对照 DESIGN.md 的间距范围比较 `margin`/`padding`/`gap` 值。

---

## 抑制

不标记：

- DESIGN.md 中明确记录为有意选择的图案

- 第三方/供应商 CSS 文件（node_modules、vendor 目录）

- CSS reset 或 normalize 样式表

- 测试固定文件

- 生成/压缩的 CSS
