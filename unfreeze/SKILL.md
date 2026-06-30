---
name: unfreeze
version: 0.1.0
description: Clear the freeze boundary set by /freeze, allowing edits to all directories again. (gstack)
triggers:
  - unfreeze edits
  - unlock all directories
  - remove edit restrictions
allowed-tools:
  - Bash
  - Read
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->


## 何时调用此技能

当你想扩大编辑范围而不想结束会话时使用。
当被要求 "unfreeze"、"unlock edits"、"remove freeze" 或
"allow all edits" 时使用。

# /unfreeze — 清除 Freeze 边界

移除由 `/freeze` 设置的编辑限制，允许编辑所有目录。

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"unfreeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'\",\"repo\":\"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'\"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 清除边界

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-paths)"
STATE_DIR="$GSTACK_STATE_ROOT"
if [ -f "$STATE_DIR/freeze-dir.txt" ]; then
  PREV=$(cat "$STATE_DIR/freeze-dir.txt")
  rm -f "$STATE_DIR/freeze-dir.txt"
  echo "Freeze boundary cleared (was: $PREV). Edits are now allowed everywhere."
else
  echo "No freeze boundary was set."
fi
```

告知用户结果。注意，`/freeze` 钩子仍在
会话中注册 — 由于状态文件不存在，它们将允许所有操作。要重新 freeze，
请再次运行 `/freeze`。
