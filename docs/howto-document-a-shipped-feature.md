# 如何记录你刚刚交付的功能

这是交付后的工作流程：你合并了一个 PR，文档已经过时，你希望一次性生成覆盖度报告并填补缺口。先运行 `/document-release` 进行审计，然后运行 `/document-generate` 来填补发现的缺口。

## 前置条件

- 已安装 gstack（完成 `./setup`；通过 `which gstack` 或在 Claude Code 中输入 `/` 查看已列出的技能来验证）
- 包含你交付功能的分支已检出
- GitHub 或 GitLab 上存在 PR（推荐 — 工作流会更新 PR 正文，添加覆盖度报告）

如果 PR 尚未创建，请先运行 `/ship` 创建一个；`/document-release` 设计为针对 PR 运行。

## 步骤

### 1. 审计当前覆盖度

运行：

```
/document-release
```

该技能会遍历你的 diff，对比基础分支，提取新的新增公开界面（技能、CLI 标志、配置选项、API 端点、新模块），并在 Diataxis 四个象限中对每个实体进行评分。你会看到类似这样的覆盖度报告：

```
覆盖度报告：
  [实体]             [参考？] [操作指南？] [教程？] [讲解？]
  /new-skill       ✅ AGENTS.md  ❌        ❌          ❌
  --new-flag       ✅ README     ✅ README  ❌          ❌
  FooProcessor     ❌            ❌        ❌          ❌
```

覆盖度为零的条目是**关键缺口**。仅有参考文档覆盖的条目是**常见缺口**。两者都会作为 `### Documentation Debt`（文档债务）子条目出现在 PR 正文中，以便评审者查看。

如果 `/document-release` 报告所有条目均已覆盖，则表示你已完成。跳过本指南的剩余步骤。

### 2. 阅读 PR 正文中的文档债务部分

打开你的 PR（该技能会打印出 URL）。滚动到 `## Documentation` → `### Documentation Debt`。每个条目都用能填补它的 Diataxis 象限进行了标记：

```
### Documentation Debt

- ⚠️ /new-skill — 在 AGENTS.md 中有参考，但 README 中没有操作指南示例。Diataxis 象限：操作指南。
- ⚠️ FooProcessor — 零覆盖。Diataxis 象限：参考、讲解。
```

这是下一步的输入。每一行都告诉你缺失了什么以及哪个象限能填补它。

### 3. 用 /document-generate 填补缺口

运行：

```
/document-generate
```

当技能询问范围时，告诉它债务部分中标记的特定实体。该技能会读取代码库（第一步的考古阶段是强制性的），按 Diataxis 象限分组，并写入缺失的文档。

你也可以让技能自动发现：如果 `/document-release` 明确向你传递了缺口信息（链式调用时会这样做），`/document-generate` 已经知道要写什么。

### 4. 验证缺口已关闭

重新运行 `/document-release`：

```
/document-release
```

覆盖度报告现在应该显示之前标记的实体在之前的空象限中有绿色对勾。PR 正文的文档债务部分应该为空，或仅包含你有意推迟的条目。

## 验证

打开你的 PR 并确认：

1. PR 正文包含带有文档差异预览的 `## Documentation` 部分。
2. `### Documentation Debt` 子部分列出零个关键缺口（或仅包含你有意推迟的条目）。
3. 生成的 `docs/` 文件中每个文档都能干净打开并交叉链接到同类文档（参考 → 操作指南 → 教程 → 讲解）。
4. 运行 `grep -rE '\]\([^)]*\.md\)' docs/` 并验证没有链接指向不存在的文件。

如果全部通过，你的 PR 即可合并，且文档完整。

## 故障排除

**`/document-release` 报告 "No public surface changes detected."（未检测到公开界面变更）。**
该 diff 是内部的（重构、测试、基础设施）。无需文档。跳过即可合并。

**缺口上的 Diataxis 象限标记与你的预期不符。**
该技能使用实体分类法来决定哪些象限相关（CLI 标志需要参考 + 操作指南；内部模块需要参考 + 讲解；用户需要全部四个）。如果你不同意，可以在生成后手动编辑文档。审计是一个指南，不是约束。

**`/document-generate` 写了一个需要 8 步才能得到工作成果的教程。**
教程应在 3 步内达到工作成果。重新运行技能并要求其压缩，或手动编辑。第 8 步质量自检能捕获其中一些但不是全部。

**你想记录一个功能，但 PR 尚未创建。**
先运行 `/ship` 创建 PR，然后使用本工作流。没有 PR 时，`/document-release` 仍可审计，但会跳过 PR 正文更新。

**生成的参考文档包含了虚构的 API 签名。**
请提交 bug。技能的步骤 1 考古阶段应该端到端读取实现文件，而不仅仅是签名，专门为了防止这种情况。包含生成的文本和实际代码，以便我们追踪为何考古阶段遗漏了它。

## 相关

- **教程：首次使用 `/document-generate`：** [tutorial-document-generate.md](./tutorial-document-generate.md)
- **为什么 gstack 使用 Diataxis 框架：** [explanation-diataxis-in-gstack.md](./explanation-diataxis-in-gstack.md)
- **审计技能的参考文档：** [`document-release/SKILL.md`](../document-release/SKILL.md)
- **生成技能的参考文档：** [`document-generate/SKILL.md`](../document-generate/SKILL.md)
