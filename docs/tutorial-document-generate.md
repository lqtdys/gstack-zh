# 教程：90 秒内为功能生成文档

你将针对一个已有项目运行 `/document-generate`，观察它在正确的位置编写教程 / 操作指南 / 参考 / 解释四类文档，最终产出一份可放入 PR 的覆盖度地图。学完后，你将掌握四个关键步骤：范围界定、代码考古、分区规划、文档编写。

## 你需要准备什么

- gstack 已安装（`git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`）
- Claude Code 运行在至少包含一项公开接口（CLI 命令、导出函数、配置项、技能或 API 端点）的项目中
- 大约 90 秒

你不需要提前创建 `docs/` 目录——如果缺失，技能会自动创建。你也不需要了解 Diataxis 术语——技能会为你标记输出分类。

## 第一步：在任何项目中调用技能

在你要编写文档的项目中打开 Claude Code，输入：

```
/document-generate
```

技能会询问一个关于输出目标的问题：

```
A) 写入现有文件的行内文档（README、ARCHITECTURE 等）
B) 创建独立的文档文件（例如 docs/ 目录）
C) 两者兼有——现有文件中的行内摘要 + 独立文件中的深度文档

推荐：选 C，因为它能最大化可发现性和深度。
```

选择 C。你将获得一份 README 指引以及一整套独立的文档。

## 第二步：观察代码考古过程

技能会静默约 30 秒，同时读取代码库。这是有意为之——第一步的"代码考古"阶段是整个流程中最重要的步骤。技能在读取：

- 完整的仓库结构
- README、ARCHITECTURE、CONTRIBUTING、CLAUDE.md（入口文件）
- 你要编写文档的功能实现文件（完整文件，不仅仅是签名）
- 测试文件（揭示边缘场景和预期行为）
- 以 `// NOTE:`、`// DESIGN:`、`// WHY:` 标记的内联注释

完成后，你会看到类似这样的输出：

```
已研究 47 个文件，识别出 12 个公开接口项、8 个概念和 4 个设计决策。
```

这个数字说明技能真正阅读了代码，而不是从文件名猜测。

## 第三步：查看 Diataxis 分区规划

技能会打印一份分区规划，显示为哪些实体编写哪些象限的文档：

```
文档计划：
  [实体]                [教程] [操作指南] [参考] [解释]
  WidgetService         ✅ 新建  ✅ 新建    ✅ 新建  ✅ 新建
  --verbose 标志        ❌      ✅ 新建    ✅ 行内  ❌
  Bayesian scheduler    ❌      ❌        ✅ 新建  ✅ 新建
```

并非每个实体都需要全部四个象限。CLI 标志获得参考 + 操作指南。内部模块获得参考 + 解释。面向用户的功能获得全部四个。技能会根据实体类型进行选择。

如果规划中的文档超过 5 个，技能会在继续前要求你确认。否则直接执行。

## 第四步：阅读生成的第一篇文档

参考文档会最先落地，因为它们确定术语。你会看到类似这样的输出：

```
已生成：docs/reference-widget-service.md
```

打开该文件。它有严格的结构：一段式介绍、带有类型和默认值的完整 API 列表、2-3 个可运行的示例，以及一个关联部分链接到即将生成的操作指南和教程。

这就是 Diataxis 中参考文档的样子：事实性、穷举性、无叙事。如果你发现自己想解释某个选项*为什么*存在，那属于解释文档的内容，技能随后会编写。

## 第五步：查看解释、操作指南和教程的生成

快速连续地（每个约 5-10 秒），技能会编写剩余的象限：

```
已生成：docs/explanation-widget-architecture.md
已生成：docs/howto-create-a-custom-widget.md
已生成：docs/tutorial-build-your-first-widget.md
```

逐个打开。注意它们不会互相重复：

- **解释** 以问题开头，然后阐述方法，最后是权衡和考虑过的替代方案
- **操作指南** 包含先决条件、带精确命令的编号步骤、验证部分和排错部分
- **教程** 让你在不超过 3 个步骤内获得可用的结果，以"你构建了什么"结尾

技能强制实施这些结构。如果某篇操作指南缺少验证部分，第八步的质量自检会在提交前捕获它。

## 第六步：检查交叉链接

每篇文档都链接到其它文档。参考文档的关联部分：链接到操作指南和教程。操作指南的关联部分：链接到参考。教程的"你构建了什么"部分：链接到参考以深入探索。

运行 grep 验证无断链：

```bash
grep -rE '\]\([^)]*\.md\)' docs/ | head -10
```

每个链接的文件都应该存在。技能的第七步"跨文档链接与可发现性"会在提交前检查这一点。

## 第七步：在 PR 正文中查看覆盖率摘要

如果你在有开放 PR 的功能分支上，技能会用 `## Documentation Generated` 表格更新 PR 正文：

```
## 文档生成

| 文件 | 象限 | 描述 |
|------|------|------|
| docs/tutorial-build-your-first-widget.md | 教程 | 从安装到首个可用 widget 的走查 |
| docs/reference-widget-service.md | 参考 | 完整的 widget API，含类型、默认值、示例 |
| docs/explanation-widget-architecture.md | 解释 | 为什么 widget 是隔离的服务 |
| docs/howto-create-a-custom-widget.md | 操作指南 | 创建和注册自定义 widget |
```

审阅 PR 的人看到表格后，就能立即知道产出了哪类覆盖度。

## 你构建了什么

你现在拥有了四份服务四位不同读者的文档：

- 项目新手可以阅读 `tutorial-*.md` 并获得可运行的成果
- 有经验的用户可以阅读 `howto-*.md` 来完成特定任务
- API 调用者可以阅读 `reference-*.md` 获取精确的签名
- 代码审阅者可以阅读 `explanation-*.md` 来理解设计

每篇都足够简短以便维护。每篇都有单一职责。PR 正文显示了哪些象限已被覆盖。如果你之后运行 `/document-release`，Diataxis 覆盖度地图会将此实体报告为完全覆盖（4/4 象限）。

## 接下来做什么

- **如果有缺口** /document-release 标记但未填写：再次运行 `/document-generate`，专门针对这些实体。
- **如果你想了解为什么存在四个象限：** 阅读 [explanation-diataxis-in-gstack.md](./explanation-diataxis-in-gstack.md)。
- **如果你想为一个已上线的特定功能编写文档**（而非整个项目）：阅读 [howto-document-a-shipped-feature.md](./howto-document-a-shipped-feature.md)。
- **技能本身的参考：** [`document-generate/SKILL.md`](../document-generate/SKILL.md)。
