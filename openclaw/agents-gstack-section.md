## 编码任务（gstack）

### 规则（不可协商）

1. **始终生成，绝不重定向。** 当用户要求使用任何 gstack 技能时，
   始终通过 sessions_spawn 生成 Claude Code 会话。永远不要告诉用户
   自己打开 Claude Code。永远不要说"这需要在 Claude Code 中运行。"
   永远不要说"你需要为此打开 Claude Code。" 直接做。

2. **解析仓库。** 如果用户命名了一个仓库或项目，将工作
   目录设置为该仓库路径。如果仓库路径未知，问哪个
   仓库——不要推给用户说自己打开 Claude Code。

3. **Autoplan 端到端运行。** 对于 /autoplan 具体来说：生成会话，
   让它运行完整的审查管道（CEO → 设计 → 工程），当它
   完成后，在这里的聊天中报告计划。将计划写入内存以便
   用户以后可以找到。用户永远不需要离开 Telegram。

### 调度路由

当被要求做编码工作时，选择调度层级：

**简单：** "修正这个拼写错误，" "更新那个配置，" 单文件更改
→ sessions_spawn(runtime: "acp", prompt: "<仅任务>")

**中等：** 多文件功能、重构、技能编辑
→ sessions_spawn(runtime: "acp", prompt: "<gstack-lite 内容>\n\n<任务>")

**重：** 需要特定 gstack 方法论
→ sessions_spawn(runtime: "acp", prompt: "加载 gstack。运行 /qa https://...")
  技能：/cso、/review、/qa、/ship、/investigate、/design-review、/benchmark、/gstack-upgrade

**完整：** 构建完整功能、多日范围，需要规划 + 审查
→ sessions_spawn(runtime: "acp", prompt: "<gstack-full 内容>\n\n<任务>")
  Claude Code 运行：/autoplan → 实现 → /ship → 报告回来

**规划：** 用户想规划一个 Claude Code 项目、设计功能、或设计
  任何代码编写前的东西
→ sessions_spawn(runtime: "acp", prompt: "<gstack-plan 内容>\n\n<任务>")
  Claude Code 运行：/office-hours → /autoplan → 保存计划文件 → 报告回来
  将计划链接持久化到内存/知识存储。
  当用户准备好实现时，生成一个新的完整会话指向该计划。

### 决策启发

- 能在 <10 行代码中完成？→ **简单**
- 是否涉及多文件但方法明显？→ **中等**
- 用户是否命名了特定技能（/cso、/review、/qa）？→ **重**
- "升级 gstack"、"更新 gstack" → **带 `Run /gstack-upgrade` 的重**
- 它是一个功能、项目或目标（不是任务）？→ **完整**
- 用户想在不实现的情况下规划什么？→ **规划**
