# 领域技能

代理为自己编写的按站点笔记。跨会话累积：一旦代理
弄清楚关于某网站的非显而易见的知识，它就保存一个技能，未来
在该 host 上的会话将让该笔记注入到其提示上下文中。

这是 gstack 从 [browser-use/browser-harness](https://github.com/browser-use/browser-harness) 的借用。
gstack 复制的是按站点笔记模式，**不是**运行时自修改
模式。技能是加载到提示中的 markdown 文本；它们不是可执行
代码。

## 代理如何使用

```bash
# 代理在成功完成任务后记录下它关于某网站的发现。
# host 自动取自活动标签页（无需代理参数）。
echo "# LinkedIn Apply Button

The Apply button on /jobs/view pages is inside an iframe with a class
matching 'jobs-apply-button-iframe'. Use \$B frame --url 'apply' first,
then snapshot." | $B domain-skill save

# 查看已保存的内容
$B domain-skill list

# 读取特定 host 技能的正文
$B domain-skill show linkedin.com

# 在 $EDITOR 中交互编辑
$B domain-skill edit linkedin.com

# 将活动的按项目技能提升为全局（跨项目）
$B domain-skill promote-to-global linkedin.com

# 回退最近的编辑
$B domain-skill rollback linkedin.com

# 删除（墓碑 — 可通过回退恢复）
$B domain-skill rm linkedin.com
```

## 状态机

```
  ┌──────────────┐  3 次成功使用        ┌────────┐  promote-to-global   ┌────────┐
  │ quarantined  │ ─────────────────▶  │ active │ ──────────────────▶  │ global │
  │ （按项目）   │  （无分类器标记）     │(项目内)│  （手动命令）        │        │
  └──────────────┘                      └────────┘                      └────────┘
         ▲                                   │
         │  使用期间分类器标记               │  回退（版本日志）
         └───────────────────────────────────┘
```

新保存的技能以 **quarantined** 状态落地，不会在提示中自动触发。
在该 host 上使用 3 次且 L4 ML 分类器未标记技能内容后，
技能自动提升为 **active** 在该项目中。活动技能在该 hostname 的每个新 sidebar-agent 会话中触发。

要使技能跨项目触发（例如，"我希望我的 LinkedIn 技能
在我工作的每个 gstack 项目上都生效"），需显式运行
`$B domain-skill promote-to-global <host>`。这是有意设计为 opt-in（Codex T4
外部语音审查）：无差别跨项目累积会在不相关工作间泄漏上下文。

## 存储

技能保存在两个位置：

- **按项目：** `~/.gstack/projects/<slug>/learnings.jsonl` — 与 `/learn` 技能
  使用的同一 JSONL 文件。领域技能是 `type:"domain"` 的行。
- **全局：** `~/.gstack/global-domain-skills.jsonl` — 仅包含 `state:"global"` 的行。

两个文件都是仅追加 JSONL。删除使用墓碑；空闲压缩器
会定期重写文件。容错解析器在读取时丢弃不完整的尾部行，
因此写入中途崩溃不会毒害后续读取。

## 安全模型

技能是代理编写的内容，被加载到未来的提示上下文中。这使
它们成为经典的代理到代理提示注入载体。本计划明确地
用多层来解决这个问题：

| 层级 | 内容 | 位置 |
|-------|------|-------|
| L1-L3 | Datamarking、隐藏元素剥离、ARIA 正则、URL 阻止列表 | `content-security.ts`（编译二进制） |
| L4 | TestSavantAI ONNX 分类器 | `security-classifier.ts`（sidebar-agent，未编译） |
| L4b | Claude Haiku transcript 分类器 | `security-classifier.ts`（sidebar-agent） |
| L5 | Canary token 泄漏检测 | `security.ts` |

L1-L3 检查在 **保存时** 运行（在守护进程中）。L4 ML 分类器在
**加载时** 运行（在 sidebar-agent 中），因此每个加载技能到其
提示中的会话也会重新验证内容。这捕获取了只有在分类器模型
更新后才会显现的问题。

保存命令从**活动标签页的顶级 origin** 派生 hostname，
而非从代理参数。这关闭了 Codex 标记的 confused-deputy Bug：
恶意页面重定向链可能欺骗代理去
毒害其他域名。

## 错误参考

| 错误 | 原因 | 操作 |
|-------|-------|--------|
| `Save blocked: classifier flagged content as potential injection` | 保存时 L4 得分 ≥ 0.85 | 重写技能，移除指令性散文；重试。 |
| `Save blocked: <L1-L3 message>` | 保存时 URL 阻止列表匹配或 ARIA 注入 | 审查技能正文是否有可疑模式。 |
| `Save failed: empty body` | 通过 stdin 或 `--from-file` 无内容 | 将 markdown 管道到 `$B domain-skill save`，或传递 `--from-file <path>`。 |
| `Cannot save domain-skill: no top-level URL on active tab` | 标签页是 `about:blank` 或 `chrome://...` | 先 `$B goto <target-site>`，然后保存。 |
| `Cannot promote: skill is in state "quarantined"` | 技能尚未自动提升 | 在该项目中使用直到 3 次成功运行无分类器标记。 |
| `Cannot rollback: <host> has fewer than 2 versions` | 仅存在一个版本 | 使用 `$B domain-skill rm` 改为删除。 |

## 遥测

当遥测启用时（默认 `community` 模式，除非手动关闭），
以下事件会被写入 `~/.gstack/analytics/browse-telemetry.jsonl`：

- `domain_skill_saved {host, scope, state, bytes}`
- `domain_skill_save_blocked {host, reason}`
- `domain_skill_fired {host, source, version}`
- `domain_skill_state_changed {host, from_state, to_state}`（计划中）

仅 hostname — 无正文内容，无代理文本。完全禁用请使用
`gstack-config set telemetry off` 或 `GSTACK_TELEMETRY_OFF=1`。
