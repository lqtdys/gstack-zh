<!-- AUTO-GENERATED from greptile.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
## 第 10 步：处理 Greptile 评审意见（如果存在 PR）

**将抓取 + 分类任务作为子代理** 调度，使用 Agent 工具并指定 `subagent_type: "general-purpose"`。子代理拉取每一条 Greptile 意见，运行升级检测算法，并对每条评价进行分类。父代理接收结构化列表后处理用户交互和文件编辑。

**子代理提示词：**

> 你正在为 /ship 工作流分类 Greptile 评审意见。请阅读 `.claude/skills/review/greptile-triage.md`，并遵循抓取、过滤、分类和 **升级检测** 步骤。不要修复代码、不要回复意见、不要提交 —— 仅输出报告。
>
> 对于每条评论，分配：`classification`（`valid_actionable`、`already_fixed`、`false_positive`、`suppressed`）、`escalation_tier`（1 或 2）、文件:行号或 [顶级] 标签、正文摘要和永久链接 URL。
>
> 如果不存在 PR、`gh` 命令失败、API 返回错误或评论数量为零，输出：`{"total":0,"comments":[]}` 并停止。
>
> 否则，在输出的 **最后一行** 输出一个 JSON 对象：
> `{"total":N,"comments":[{"classification":"...","escalation_tier":N,"ref":"file:line","summary":"...","permalink":"url"},...]}`

**父代理处理：**

将最后一行为 JSON 进行解析。

如果 `total` 为 0，静默跳过此步骤，继续第 12 步。

否则，打印：`+ {total} 条 Greptile 评论（{valid_actionable} 条有效，{already_fixed} 条已修复，{false_positive} 条误报）`。

对于 `comments` 中的每条评论：

**有效且需处理：** 使用 AskUserQuestion，提供：
- 评论内容（文件:行号或 [顶级] + 正文摘要 + 永久链接 URL）
- `RECOMMENDATION: 选择 A，因为 [单行原因]`
- 选项：A) 立即修复, B) 确认并直接发布, C) 这是误报
- 如果用户选择 A：应用修复，提交修复文件（`git add <fixed-files> && git commit -m \"fix: address Greptile review — <brief description>\"`），使用 greptile-triage.md 中的 **修复回复模板**（包含内联 diff + 解释）回复，并保存到项目级和全局 greptile 历史记录（类型：fix）
- 如果用户选择 C：使用 greptile-triage.md 中的 **误报回复模板**（包含证据 + 建议重新排序）回复，并保存到项目级和全局 greptile 历史记录（类型：fp）

**有效但已修复：** 使用 greptile-triage.md 中的 **已修复回复模板** 回复，无需 AskUserQuestion：
- 包含已完成的操作和修复提交的 SHA
- 保存到项目级和全局 greptile 历史记录（类型：already-fixed）

**误报：** 使用 AskUserQuestion：
- 展示评论及你认为错误的原因（文件:行号或 [顶级] + 正文摘要 + 永久链接 URL）
- 选项：
  - A) 回复 Greptile 解释误报原因（建议用于明显错误的情况）
  - B) 仍然修复（如果很简单）
  - C) 静默忽略
- 如果用户选择 A：使用 greptile-triage.md 中的 **误报回复模板**（包含证据 + 建议重新排序）回复，保存到项目级和全局 greptile 历史记录（类型：fp）

**已抑制：** 静默跳过——这些是之前分类中已知的误报。

**所有评论处理完毕后：** 如果应用了任何修复，第 5 步的测试现在已经过期。继续第 12 步之前 **重新运行测试**（第 5 步）。如果没有应用修复，直接继续第 12 步。

---
