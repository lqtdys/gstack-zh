# 会话智能层

## 问题

Claude Code 的上下文窗口是短暂的。每个会话都是全新开始。当自动压缩在 ~167K token 触发时，它仅保留通用摘要，但会销毁文件读取、推理链和中间决策。

gstack 已在磁盘上产生有价值的产物：CEO 计划、工程评审、设计评审、QA 报告、学习记录。这些文件包含塑造当前工作的决策、约束和上下文。但 Claude 不知道它们的存在。压缩后，通知每个决策的计划和评审会悄悄从上下文中消失。

生态系统正在致力于此。claude-mem（9K+ stars）捕获工具使用并将上下文注入未来会话。Claude HUD 显示实时 agent 状态。Anthropic 自己的 `claude-progress.txt` 模式使用一个进度文件，agent 在每个会话开始时读取。

没有人解决让**技能产物**在压缩后存活这个具体问题。因为其他人没有 gstack 的产物架构。

## 洞察

gstack 已经将结构化产物写入 `~/.gstack/projects/$SLUG/`：
- CEO 计划：`ceo-plans/`
- 设计评审：`design-reviews/`
- 工程评审：`eng-reviews/`
- 学习记录：`learnings.jsonl`
- 技能使用：`../analytics/skill-usage.jsonl`

缺失的部分不是存储。是感知。前言需要告诉 agent："这些文件存在。它们包含你已经做出的决策。压缩后，重新读取它们。"

## 架构

```
                   ┌─────────────────────────────────────┐
                   │        Claude 上下文窗口            │
                   │   （短暂的，~167K token 限制）       │
                   │                                      │
                   │   压缩触发 ──► 仅保留摘要            │
                   └──────────────┬──────────────────────┘
                                  │
                          启动时 / 压缩后读取
                                  │
                   ┌──────────────▼──────────────────────┐
                   │    ~/.gstack/projects/$SLUG/         │
                   │    （持久的，在一切情况下存活）       │
                   │                                      │
                   │  ceo-plans/         ← /plan-ceo-review
                   │  eng-reviews/       ← /plan-eng-review
                   │  design-reviews/    ← /plan-design-review
                   │  checkpoints/       ← /checkpoint（新增）
                   │  timeline.jsonl     ← 每个技能（新增）
                   │  learnings.jsonl    ← /learn
                   └─────────────────────────────────────┘
                                  │
                          每周汇总
                                  │
                   ┌──────────────▼──────────────────────┐
                   │           /retro                     │
                   │  时间线：3 /review, 2 /ship, ...     │
                   │  健康趋势：编译 8/10 (↑2)            │
                   │  学习应用：本周 4 个                  │
                   └─────────────────────────────────────┘
```

## 功能

### 第 1 层：上下文恢复（前言，所有技能）
前言中约 10 行散文。压缩或上下文降级后，agent 检查 `~/.gstack/projects/$SLUG/` 获取最近的计划、评审和检查点。列出目录，读取最新文件。

成本：接近零。收益：每个技能的计划/评审在压缩后存活。

### 第 2 层：会话时间线（前言，所有技能）
每个技能向 `timeline.jsonl` 追加单行 JSONL 条目：时间戳、技能名、分支、关键结果。`/retro` 渲染它。

使项目的 AI 辅助工作历史可见。"本周：3 /review, 2 /ship, 1 /investigate 横跨 feature-auth 和 fix-billing 分支。"

### 第 3 层：跨会话注入（前言，所有技能）
当新会话在具有近期产物的分支上启动时，前言打印一行："上次会话：实现了 JWT 认证，3/5 任务完成。计划：~/.gstack/projects/$SLUG/checkpoints/latest.md"

agent 在读取任何文件之前就知道上次停在哪里。

### 第 4 层：/checkpoint（可选技能）
工作状态的手动快照：正在做什么、正在编辑的文件、已做出的决策、剩余内容。在离开前、复杂操作前、工作区交接时、或几天后回来时有用。

### 第 5 层：/health（可选技能）
代码质量仪表板：类型检查、lint、测试套件、死代码扫描。综合 0-10 分。随时间追踪。`/retro` 展示趋势。`/ship` 在可配置阈值上门控。

## 复利效应

每个功能独立有用。组合后，它们创造了复利效应：

会话 1：/plan-ceo-review 产生计划。保存到磁盘。
会话 2：agent 在前言后读取计划。不再重复询问决策。
会话 3：/checkpoint 保存进度。时间线显示 2 /review, 1 /ship。
会话 4：压缩在半途触发。agent 重新读取检查点。
           恢复关键决策、类型、剩余工作。继续。
会话 5：/retro 汇总本周。健康趋势：6/10 → 8/10。
           时间线显示 3 个分支上 12 次技能调用。

项目的 AI 历史不再是短暂的。它持续存在、复利增长并使每个未来会话更智能。这就是会话智能层。

## 这不是什么

- 不是替代 Claude 内置压缩（它处理会话状态；我们处理 gstack 产物）
- 不是像 claude-mem 那样的完整内存系统（它通过 SQLite 处理跨会话内存；我们处理结构化技能产物）
- 不是数据库或服务（只是磁盘上的 markdown 文件）

## 研究来源

- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effectivecontext-engineering-for-ai-agents)
- [claude-mem](https://github.com/thedotmack/claude-mem)
- [Claude HUD](https://github.com/jarrodwatts/claude-hud)
- [CodeScene: Agentic AI coding best practices](https://codescene.com/blog/agentic-ai-coding-best-practice-patterns-for-speed-with-quality)
- [Post-compaction recovery via git-persisted state (Beads)](https://dev.to/jeremy_longshore/building-post-compaction-recovery-for-ai-agent-workflows-with-beads-207l)
