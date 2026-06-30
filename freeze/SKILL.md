---
name: freeze
version: 0.1.0
description: Restrict file edits to a specific directory for the session. (gstack)
triggers:
  - freeze edits to directory
  - lock editing scope
  - restrict file changes
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
hooks:
  PreToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "bash $HOME/.claude/skills/gstack/freeze/bin/check-freeze.sh"
          statusMessage: "Checking freeze boundary..."
    - matcher: "Write"
      hooks:
        - type: command
          command: "bash $HOME/.claude/skills/gstack/freeze/bin/check-freeze.sh"
          statusMessage: "Checking freeze boundary..."
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## 何时调用此技能

阻止 Edit 和
Write 在允许路径之外的操作。调试时使用，防止意外\"修复\"不相关的代码，
或当你想将更改限定在一个模块内时。
当被要求 \"freeze\"、\"restrict edits\"、\"only edit this folder\"、
或 \"lock down edits\" 时调用。

# /freeze — 将编辑限定在目录内

将文件编辑锁定到特定目录。任何 Edit 或 Write 操作，如果目标文件
在允许路径之外，将被**阻止**（而非仅警告）。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"freeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'\","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'\""}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

询问用户要将编辑限定到哪个目录。使用 AskUserQuestion：

- 问题: "Should I restrict edits to which directory? Files outside this path will be blocked from editing."
- 文本输入（非多选）— 用户输入路径。

用户提供目录路径后：

1. 解析为绝对路径：
```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
echo "$FREEZE_DIR"
```

2. 确保末尾斜杠并保存到 freeze 状态文件：
```bash
FREEZE_DIR="${FREEZE_DIR%/}/"
eval "$(~/.claude/skills/gstack/bin/gstack-paths)"
STATE_DIR="$GSTACK_STATE_ROOT"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
echo "Freeze boundary set: $FREEZE_DIR"
```

告知用户: "Edits are now restricted to `<path>/`. Any Edit or Write
outside this directory will be blocked. To change the boundary, run `/freeze`
again. To remove it, run `/unfreeze` or end the session."

## 工作原理

钩子从 Edit/Write 工具输入 JSON 中读取 `file_path`，然后检查
该路径是否以 freeze 目录开头。如果不是，则返回
`permissionDecision: "deny"` 以阻止操作。

freeze 边界通过状态文件在整个会话中持久化。钩子脚本在每次 Edit/Write 调用时读取它。

## 注意事项

- freeze 目录末尾的 `/` 防止 `/src` 匹配到 `/src-old`
- Freeze 仅适用于 Edit 和 Write 工具 — Read、Bash、Glob、Grep 不受影响
- 这仅用于防止意外编辑，而非安全边界 — 像 `sed` 这样的 Bash 命令仍可修改边界外的文件
- 要停用，运行 `/unfreeze` 或结束对话
