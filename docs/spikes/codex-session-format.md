# Spike：用于 plan-tune cathedral 的 Codex 会话存储格式

**状态：** 已完成（2026-05-27）
**涉及面：** D5（Codex 导入解析结构化文件，而非正则）
**下游消费者：** T9（gstack-codex-session-import）

## 本次 Spike 回答的问题

Codex 会话的实际磁盘格式是什么，我们如何从中恢复 `AskUserQuestion` 形状的事件用于 `gstack-codex-session-import`？

## 存储布局

```
~/.codex/
├── auth.json                     # Codex 认证（请勿触及）
├── config.toml                   # 用户配置
├── goals_1.sqlite                # ~24KB，内部 goals 数据库（不相关）
├── logs_2.sqlite                 # ~16MB，结构化日志（target=*，见模式）
├── history.jsonl                 # ~9KB，命令历史
└── sessions/
    └── 2026/05/27/
        └── rollout-<iso8601>-<uuid>.jsonl   每个会话转录
```

会话文件：每个 `codex exec` 或交互式会话一个 JSONL。Cwd 路径嵌入在 `session_meta` 事件中。CLI 版本记录。

## 会话 JSONL 事件类型（在 Garry 的机器上测量，2026-05-27）

| type           | 计数 | 含义 |
|----------------|------|------|
| `response_item`|   382 | 模型的响应流（~76%） |
| `event_msg`    |    97 | 高级会话事件（~19%） |
| `turn_context` |     6 | 每轮上下文快照 |
| `session_meta` |     6 | 会话头部（每个会话一个） |

### response_item 子类型

| subtype                  | 计数 | 含义 |
|--------------------------|------|------|
| `function_call`          | 148   | 模型调用了一个工具 |
| `function_call_output`   | 148   | 工具结果返回给模型 |
| `reasoning`              |  44   | 推理摘要 |
| `message`                |  40   | 文本消息（input_text 或 output_text） |
| `web_search_call`        |   2   | 网络搜索工具调用 |

### event_msg 子类型

| subtype           | 计数 | 含义 |
|-------------------|------|------|
| `token_count`     | 55    | 每步 token 统计 |
| `agent_message`   | 22    | agent 的散文输出 |
| `user_message`    |  6    | 用户的散文输入 |
| `task_started`    |  6    | 任务开始（每个顶级任务一个） |
| `task_complete`   |  6    | 任务完成 |
| `web_search_end`  |  2    | 网络搜索完成 |

## 关键发现：Codex 没有 `AskUserQuestion` 工具

Codex 不会在 `response_item` 流中将 AskUserQuestion 暴露为工具调用。在 Codex 上运行的 Gstack 技能将 AskUserQuestion 形状的决策简报作为散文发出在 `agent_message` 事件中（来自序言的 `AskUserQuestion Format`）。用户的答案在下一个 `user_message` 中返回。

**这意味着**从 Codex 会话导入 AUQ 事件在**结构上**不同于从 Claude Code 导入（在 Claude Code 中它们**是**工具调用）：

- **Claude Code：** hook 捕获 `AskUserQuestion` 的结构化 `tool_input`/`tool_output`。问题 + 选项 + 答案全部分开。
- **Codex：** 解析器必须从 `agent_message.text` 正文中提取，检测 D 编号决策简报模式，然后与后续的 `user_message` 匹配以获取答案。

## `gstack-codex-session-import` 的恢复策略

**双层提取：**

1. **标记优先（D18 机制）。** 搜索 `agent_message` 文本中的 `<gstack-qid:foo-bar>` 标记。如果存在，我们有一个确切的 question_id 并可以可靠地恢复。（一旦 T14 将标记添加到前 10 个注册表问题，并且 Codex 开始通过宿主感知序言路径发出它们，就会生效。）

2. **模式回退。** 当没有标记时，解析：
   - `D<N> — <title>` 行（来自 AskUserQuestion Format 的 D 编号）
   - `Recommendation: ...` 行
   - 选项块 `A) ...`、`B) ...` 等
   - 下一个 `user_message` 事件获取所选选项的标签

   仅用它来填充基于哈希的 question_id（与 Claude 上层 1 使用的 `hook-<sha1(skill+text+sorted_options)[:10]>` 相同形状）。标记为 `source: "codex-pattern-fallback"`，**永远不用作偏好键**（按 D18 哈希漂移指导）。

## 我们将从 Codex 导入写入 question-log.jsonl 的模式

每现有的 `bin/gstack-question-log` 模式，扩展：
- `source: "codex-import-marker"`（当找到 qid 标记时）
- `source: "codex-import-pattern"`（当使用回退正则时）
- `codex_session_id`（来自 session_meta 的 UUID）
- `codex_cwd`（来自 session_meta 的工作目录 — 区分项目）
- `codex_ts`（来自事件的时间戳）

## Sqlite logs_2.sqlite 模式

```sql
CREATE TABLE logs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts INTEGER NOT NULL,
  ts_nanos INTEGER NOT NULL,
  level TEXT NOT NULL,
  target TEXT NOT NULL,
  feedback_log_body TEXT,
  module_path TEXT,
  file TEXT,
  line INTEGER,
  thread_id TEXT,
  process_uuid TEXT,
  estimated_bytes INTEGER NOT NULL DEFAULT 0
);
```

`logs_2.sqlite` 是内部遥测，不是会话内容。**不要用于 AUQ 提取。** 会话 JSONL 是权威的。

## 项目短名称派生

从 `session_meta.payload.cwd` — 通过现有 `bin/gstack-slug` 逻辑在 cwd 路径上派生。Conductor 工作树在其 cwd 中有自己的短名称命名约定；该目录已处理。

## 版本安全

`session_meta.payload.cli_version` 记录 Codex CLI 版本（例如 `0.130.0`）。当导入器遇到未知版本时，记录警告到 stderr 但继续 — 模式添加通常在 JSONL 中向后兼容。

如果 `type` 或 `payload.type` 值在未来版本中更改，我们会在导入器的审计日志中看到它们为 `unknown`。在导入器中添加一个受保护的 `KNOWN_VERSIONS = ["0.130.x", "0.131.x", ...]` 常量，并在重新测试时显式提升。

## 实现的开放问题

1. **Codex 究竟在哪里存储"用户的答案"？** 需要用触发决策简报的真实 `codex exec` 运行来测试并检查下一个事件。可能是子类型 `user_message` 的 `event_msg` 或子类型 `message` 的 `response_item`，带有 `role: "user"`。在 T9 实现期间确认。

2. **"其他"的散文提取。** 决策简报散文不在结构上从命名选项中分离"其他"响应。模式回退将需要检测答案中的"Other: <text>"措辞。T10（梦想周期蒸馏）仅在源为 `codex-import-marker` 时触发，因此我们可以信任数据。

3. **Conductor cwd 处理。** Conductor 工作树共享项目状态但有不同的 cwd。导入应该按项目短名称分桶事件，而不是直接按 cwd，以便来自兄弟工作树的事件累积到同一项目视图中。

## 参考

- 现场检查 `~/.codex/sessions/2026/05/*/`
- `sqlite3 ~/.codex/logs_2.sqlite ".schema"`（2026-05-27）
- Codex CLI 0.130.0（Spike 时的当前版本）
- 另见：plan 文件中的 D5 跨模型张力决策。
