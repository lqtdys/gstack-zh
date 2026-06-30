# Spike：用于 plan-tune cathedral 的 Claude Code hook 变更

**状态：** 已完成（2026-05-27）
**涉及面：** D10（PreToolUse 是否允许变更 AUQ 输入？）、D19/Codex（匹配器必须覆盖 MCP 变体）
**下游消费者：** T3、T5、T6、T8

## 本次 Spike 回答的问题

`AskUserQuestion` 上的 PreToolUse hook 能否通过 `updatedInput` 真正替代用户的答案？如果可以，确切的协议是什么？

## 答案

**可以。** `updatedInput` 是支持的机制。来源：https://code.claude.com/docs/en/hooks（2026-04 参考确认）。

## Hook stdin 模式（PreToolUse + PostToolUse）

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/dir",
  "permission_mode": "default",
  "effort": { "level": "medium" },
  "hook_event_name": "PreToolUse",
  "tool_name": "AskUserQuestion",
  "tool_input": { /* 工具特定 */ },
  "tool_use_id": "unique-id-12345"
}
```

在子代理上下文中可选：`agent_id`、`agent_type`。

## PreToolUse hook 用于 `allow + updatedInput` 的 stdout 模式

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "由 plan-tune 偏好自动决定",
    "updatedInput": { /* 浅合并到原始 tool_input */ },
    "additionalContext": "Claude 的可选上下文"
  }
}
```

**permissionDecision 值：**
- `"allow"` — 继续执行，可选择带有 `updatedInput`
- `"deny"` — 阻止（反馈给 Claude，按 D 前缀决策中的 Codex 修正，**不是** 合成答案）
- `"ask"` — 上报给用户
- `"defer"` — 让权限流继续

**`updatedInput` 语义：** 返回对象中存在的字段浅合并到原始 `tool_input` 上。仅在与 `permissionDecision: "allow"` 时有效。这就是让我们为 `never-ask` 偏好替代自动决定答案的原因。

## 匹配器模式

当 `matcher` 字段在 `~/.claude/settings.json` 中**包含正则表达式元字符**时支持 JS-regex 语法。仅包含字母/下划线的匹配器是精确匹配。

要覆盖原生 + MCP `AskUserQuestion`：
```json
"matcher": "(AskUserQuestion|mcp__.*__AskUserQuestion)"
```

Conductor 通过 `--disallowedTools` 禁用原生 `AskUserQuestion`，并通过 `mcp__conductor__AskUserQuestion` 路由 — 我们的 hook 需要该 MCP 后缀才能在那里触发。

## 多 hook 并发注意事项

> 所有匹配的 hook 并行运行，相同的处理器会自动去重。

**对于我们的用例：**
- gstack 在 AUQ 形状的 tool 名上恰好注册一个 PreToolUse hook 和一个 PostToolUse hook。
- 如果用户有他们自己的也在 AskUserQuestion 上返回 `updatedInput` 的 hook，合并顺序未定义。
- 缓解措施：在 `bin/gstack-settings-hook` 安装提示中记录此约束。用户可以在接受前从差异预览中检测冲突。

**`permissionDecision` 优先级（多个 hook 决定时）：**
`deny > ask > allow > defer` — 最严格的获胜。

## 实现 hookSpecificOutput 示例

**自动决定（PreToolUse，`never-ask` 偏好 + 非单向）：**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "plan-tune: never-ask 偏好 on ship-test-failure-triage",
    "updatedInput": {
      "questions": [{ /* 与输入相同，但带自动选择的答案 */ }]
    }
  }
}
```

**直通（无偏好，或单向安全覆盖）：**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "defer"
  }
}
```

**PostToolUse 捕获（始终）：**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse"
  }
}
```
（PostToolUse hook 也可以设置 `additionalContext` 附加到工具结果；我们不需要这个用于 v1 捕获。）

## 工具错误上的 PostToolUse（AUQ 失败回退，OV3:B）— **未验证**

AUQ 失败散文回退添加了一个防御性 PostToolUse hook（`hosts/claude/hooks/auq-error-fallback-hook.ts`），当 AskUserQuestion 调用返回错误/缺失结果时，注入 `additionalContext` 提醒模型按 `SESSION_KIND` 运行散文回退。它使用上面记录的相同 `additionalContext` 机制。

**我们无法在测试台中解决的开放问题：** 当 MCP 工具调用返回传输/缺失结果错误时，Claude Code 是否调用 PostToolUse hook（Conductor bug 表现为 `[Tool result missing due to internal error]`）？上面的文档涵盖*成功*时的 PostToolUse。我们无法按需强制 Conductor 内部 MCP 失败来观察它。

**决策（OV3:B = A）：** 仍然防御性地构建 hook。
- 它在成功时**惰性**（仅在 `isErrorResponse(tool_response)` 为 true 时触发），如果平台在错误路径上从不调用它，也**惰性**。
- `generate-ask-user-format.ts` 中的提示级别回退覆盖了这种情况 — hook 是可靠性*层*，不是机制。
- 其决策逻辑已经过确定性单元测试（`test/auq-error-fallback-hook.test.ts`）：给定合成错误 `tool_response` + 每个 `SESSION_KIND`，它发出正确的指令；给定真实答案时，它延迟。

**推荐的手动/部分 Spike（稍后关闭此间隙）：** 注册一个临时的 PostToolUse hook 记录触发，然后（a）触发正常工具调用（例如失败的 `Bash` 调用）以确认 PostToolUse 在工具错误时触发，（b）重现 Conductor MCP AUQ 失败并检查日志。如果（b）确认触发，将 hook 从"防御性/惰性"提升为"已验证"。在此之前，将运行时层视为尽力而为，将提示级别回退视为保证路径。

## T8 hook 安装程序的 Settings.json 片段

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "(AskUserQuestion|mcp__.*__AskUserQuestion)",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/skills/gstack/hosts/claude/hooks/question-preference-hook",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "(AskUserQuestion|mcp__.*__AskUserQuestion)",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/skills/gstack/hosts/claude/hooks/question-log-hook",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Hook 命令在底层采用 `bun` 调用；Claude Code 的 hook 运行器要求绝对路径（或 `$CLAUDE_PROJECT_DIR` 替换）。hooks 本身是由 bash 包装器 bun 化运行的 TypeScript 文件。

## 推迟到实现的开放问题

1. **推荐选项解析范围。** D2 说首先解析 `(recommended)` 标签。根据 AskUserQuestion Format，标签在选项的 `label` 字段上。实现将需要遍历 `tool_input.questions[*].options[*]` 寻找标签后缀。工作示例：ship/SKILL.md.tmpl 发出类似 `"A) Fix now" (recommended)` 的选项。

2. **自动决定事件标记。** 当 hook 返回 `updatedInput` 时，PostToolUse hook 将看到已解析的输入并记录正常事件。需要在 PostToolUse 负载上添加一个额外字段（例如，`was_auto_decided: true`），hook 可以通过会话状态跟踪来设置它 — 从 PreToolUse 在 `~/.gstack/sessions/<id>/.auto-decided-<tool_use_id>` 中写入标记文件，从 PostToolUse 读取它，读取后删除。

3. **超时行为。** 默认 hook 超时是 60s，但关于超时发生时会发生什么的文档很少。设置显式 `timeout: 5` 以便用户永远不会在 hook 误触发时等待 >5s。回退到直通。

## 参考

- https://code.claude.com/docs/en/hooks（规范的，截至 2026-04 最新）
- 2026-05-27 的 WebSearch 结果
- 现有 `bin/gstack-settings-hook`（仅 SessionStart 实现，将被 T3 模式感知重写取代）
