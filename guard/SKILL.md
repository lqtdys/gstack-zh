---
name: guard
version: 0.1.0
description: "Full safety mode: destructive command warnings + directory-scoped edits. (gstack)"
triggers:
  - full safety mode
  - guard against mistakes
  - maximum safety
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash $HOME/.claude/skills/gstack/careful/bin/check-careful.sh"
          statusMessage: "Checking for destructive commands..."
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

合并 /careful（在 rm -rf、DROP TABLE、force-push 等操作前警告）和
/freeze（阻止在指定目录之外的编辑）。当你接触生产环境或调试线上系统时，
用于最大安全性。当被要求 "guard mode"、
"full safety"、"lock it down" 或 "maximum safety" 时使用。

# /guard — 全面安全模式

同时启用破坏性命令警告和目录范围编辑限制。
这是 `/careful` + `/freeze` 的组合，集成在一个命令中。

**依赖说明：** 此 skill 引用了同级 `/careful`
和 `/freeze` skill 目录中的钩子脚本。两者都必须已安装（它们由 gstack 安装脚本一起安装）。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"guard","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'\","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'\""}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 设置

询问用户要将编辑限定到哪个目录。使用 AskUserQuestion：

- 问题: "Guard mode: which directory should edits be restricted to? Destructive command warnings are always on. Files outside the chosen path will be blocked from editing."
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

告知用户：
- "**Guard mode 已激活。** 以下两项保护现已运行："
- "1. **破坏性命令警告** — rm -rf、DROP TABLE、force-push 等操作在执行前会发出警告（可覆盖）"
- "2. **编辑边界** — 文件编辑限定在 `<path>/` 内。此目录外的编辑将被阻止。"
- "要移除编辑边界，请运行 `/unfreeze`。要停用所有保护，请结束对话。"

## 保护范围

完整的破坏性命令模式列表和安全例外，请参阅 `/careful`。
编辑边界强制执行的工作原理，请参阅 `/freeze`。
