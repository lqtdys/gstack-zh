# 设计：slop-scan 在 /review 和 /ship 中的集成

状态：已推迟
创建日期：2026-04-09
依赖项：slop-diff 脚本（scripts/slop-diff.ts，已合入）

## 问题

slop-scan 的发现只有在手动运行 `bun run slop:diff` 时才可见。
它们应该在代码审查和发布过程中自动浮出水面，与 SQL 安全和信任边界检查的做法一样。

## 集成点

### /review（第 4 步，清单通过后）

在 critical/informational 清单通过后运行 `bun run slop:diff`。将新发现与其他审查输出一起内联显示：

```
Pre-Landing Review: 3 issues (1 critical, 2 informational)

AI Slop: +2 new findings, -0 removed
  browse/src/new-feature.ts
    defensive.empty-catch: 2 locations
      line 42: empty catch, boundary=filesystem
      line 87: empty catch, boundary=process
```

分类：INFORMATIONAL（永不阻塞合并，仅呈现该模式）。

Fix-First 启发式方法适用：如果发现在文件操作周围有空 catch 块，
则用 `safeUnlink()` 自动修复。如果是在扩展代码中的 catch-and-log，则跳过
（按 CLAUDE.md 指南，这是正确的模式）。

### /ship（第 3.5 步，pre-landing 审查 + PR 正文）

与 /review 相同的集成。此外，在 PR 正文中显示单行摘要：

```markdown
## Pre-Landing Review
- 2 issues auto-fixed, 0 needs input
- AI Slop: +0 new / -3 removed ✓
```

### Review 就绪仪表盘

不要添加一行。Slop 是对 diff 的诊断，不是一次"运行"的独立审查。
它出现在 Eng Review 输出中，而非作为自己的仪表盘条目。

## 哪些需要自动修复 vs 哪些需要跳过

遵循 CLAUDE.md "Slop-scan" 章节。摘要如下：

**自动修复（真正的质量改进）：**
- `fs.unlinkSync` 周围的空 catch → 替换为 `safeUnlink()`
- `process.kill` 周围的空 catch → 替换为 `safeKill()`
- 无外层 try 的 `return await` → 移除 `await`
- URL 解析周围的无类型 catch → 添加 `instanceof TypeError` 检查

**跳过（slop-scan 标记的正确模式）：**
- 在 fire-and-forget 浏览器操作（page.close、bringToFront）上的 `.catch(() => {})`
- Chrome 扩展代码中的 catch-and-log（未捕获的错误会崩溃扩展）
- 在关闭/紧急路径中的 `safeUnlinkQuiet`（吞掉所有错误是正确的）
- 委托给活动会话的传递包装器（API 稳定性层）

## 实现说明

- `scripts/slop-diff.ts` 已处理繁重的工作（基于 worktree 的 base 比较、不敏感于行号的指纹、优雅的回退）
- review/ship 技能运行 bash 块。集成方式是：运行脚本、解析输出、包含在审查发现中
- 如果 slop-scan 未安装（`npx slop-scan` 失败），则静默跳过
- 脚本总是退出 0（诊断性的，永不阻塞）

## 工作量估算

| 任务 | 人工 | CC+gstack |
|------|-------|-----------|
| 添加到 review/SKILL.md.tmpl | 2 小时 | 10 分钟 |
| 添加到 ship/SKILL.md.tmpl | 2 小时 | 10 分钟 |
| 添加到 review/checklist.md | 1 小时 | 5 分钟 |
| 用实际 PR 测试 | 2 小时 | 15 分钟 |
| 重新生成 SKILL.md 文件 | — | 1 分钟 |
