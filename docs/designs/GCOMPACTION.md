# GCOMPACTION.md — 设计与架构（TABLED）

**批准时的目标路径：** `docs/designs/GCOMPACTION.md`

这是 `gstack compact` 的保存设计产物。下方第一个 `---` 分隔符之前的所有内容会被原样提取到 `docs/designs/GCOMPACTION.md` 中（计划批准时）。分隔符之后是归档研究（office hours + 深度竞品分析 + 工程评审笔记 + codex 评审 + 研究发现），这些信息为设计提供了依据。

---

## 状态：TABLED（2026-04-17）— 等待 Anthropic `updatedBuiltinToolOutput` API

**搁置原因。** v1 架构假设 Claude Code 的 `PostToolUse` 钩子可以替换内置工具（Bash、Read、Grep、Glob、WebFetch）进入模型上下文的工具输出。2026-04-17 的研究确认目前这不可能。

**证据：**

1. **官方文档**（https://code.claude.com/docs/en/hooks）：为 `PostToolUse` 记录的唯一 output-replace 字段是 `hookSpecificOutput.updatedMCPToolOutput`，文档明确说明：*"仅限 MCP 工具：用提供值替换工具输出。"* 内置工具无等效字段。
2. **Anthropic issue [#36843](https://github.com/anthropics/claude-code/issues/36843)**（OPEN）：Anthropic 自己承认这个差距。*"PostToolUse 钩子可以通过 `updatedMCPToolOutput` 替换 MCP 工具输出，但内置工具（WebFetch、WebSearch、Bash、Read 等）没有等效物……它们只能通过 `decision: block`（注入原因字符串）或 `additionalContext` 添加警告。原始恶意内容仍到达模型。"*
3. **RTK 机制**（源码审阅于 `src/hooks/init.rs:906-912` 和 `hooks/claude/rtk-rewrite.sh:83-100`）：RTK 不是 PostToolUse 压缩器。它是一个重写 `tool_input.command` 的 **PreToolUse** Bash 匹配器（例如 `git status` → `rtk git status`）。被包装的命令自己产生紧凑的 stdout。RTK README 确认：*"钩子只在 Bash 工具调用上运行。Claude Code 内置工具如 Read、Grep 和 Glob 不通过 Bash 钩子，因此不会被自动重写。"* RTK 是按架构约束仅支持 Bash，而非选择。
4. **tokenjuice 机制**（源码审阅于 `src/core/claude-code.ts:160, 491, 540-549`）：tokenjuice 确实注册了带 `matcher: "Bash"` 的 `PostToolUse`，但没有真正的 output-replace API 可用——它劫持 `decision: "block"` + `reason` 来注入压缩文本。这是否实际减少了模型上下文 token 还是仅叠加 UI 输出存在争议。tokenjuice 也是仅 Bash。
5. **Read/Grep/Glob 在 Claude Code 进程中执行**并完全绕过钩子。楔子（ii）"原生工具覆盖"从第一天起在架构上就不可能，无论替换 API 是否存在。

**后果。** 两个楔子都以原始形式死亡：
- 楔子（i）"条件 LLM 验证器"——技术上仍可能，但仅用于 Bash，通过 PreToolUse 命令包装（RTK 的机制）。一旦我们也仅 Bash，验证器就不再是差异化因素。
- 楔子（ii）"原生工具覆盖"——今天不可能。Read/Grep/Glob 不触发钩子。即使它们触发了，也没有用于非 MCP 工具的输出替换字段。

**决定。** 完全搁置 `gstack compact`。跟踪 Anthropic issue #36843 以等待 `updatedBuiltinToolOutput`（或等效物）的到来。当该 API 发版时，本文档 + 下方 15 个锁定决策 + 底部的研究档案成为新实现 sprint 的解锁产物。

**如果取消搁置：** 从下方的"plan-eng-review 期间锁定的决策"块开始——大多数仍有效。然后针对新发版的 API 重新验证钩子参考，更新架构数据流图以使用任何真实输出替换字段，并在编码前对修订后的计划重新运行 `/codex review`。

**我们不做的事：**
- 不发版 Bash-only PreToolUse 包装器。那是 RTK 的产品；他们有 28K star 和 3 年的规则伤疤。没有楔子。
- 不发版 `decision: block` + `reason` hack。无文档行为，Anthropic 可能破坏它，而且模型可能仍会在压缩叠加旁看到原始输出——上下文节省存在争议。
- 不发版单独的 B-series 基准。没有工作的压缩器，就没有可基准测试的东西。

**搁置成本：** ~0。未写代码。文档 + 研究 + 决策保持为准备解锁的产物。

---

## plan-eng-review 期间锁定的决策（2026-04-17）

如果/当 Anthropic 发版内置工具 output-replace API 时为取消搁置 sprint 保留。

工程评审期间每个决策的完整摘要。完整理由保留在下方的各部分中；如果其他任何内容漂移，此块是真实来源。

**范围（第 0 节）：**
1. **Claude-first v1。** 仅在 Claude Code 上发版 compact + rules + verifier。Codex + OpenClaw 在 v1.1 落地，在主 host 证明楔子之后。减少约 2 天 host 集成并降低发布风险。原始"楔子（ii）原生工具覆盖"声明仅适用于 v1 的 Claude Code；我们在 v1.1 前不做跨 host 声明。
2. **13 规则启动库。** v1 发版 tests（jest/vitest/pytest/cargo-test/go-test/rspec）+ git（diff/log/status）+ install（npm/pnpm/pip/cargo）。build/lint/log 家族推迟到 v1.1，由真实用户的 `gstack compact discover` 遥测驱动。
3. **验证器在 v1.0 默认开启。** `failureCompaction` 触发（exit≠0 AND >50% 缩减）开箱即用。验证器就是楔子——默认关闭会隐藏差异化功能。触发边界已保持预期触发率 ≤10% 工具调用。

**架构（第 1 节）：**
4. **Haiku 输出的精确行匹配消毒。** 用 `\n` 分割原始输出，将行放入集合，仅追加 Haiku 中在该集合中逐字出现的行。最紧密的对抗契约；提示注入尝试无法潜入新文本。
5. **分层 failureCompaction 信号。** 优先使用信封中的 `exitCode`；如果 host 省略它，回退到输出上的 `/FAIL|Error|Traceback|panic/` 正则。在 `meta.failureSignal`（"exit" | "pattern" | "none"）中记录哪个信号触发。预实现任务 #1 仍经验性地验证 Claude Code 的信封，但如果它不提供，系统不再损坏。
6. **深度合并规则解析。** 用户/项目规则继承它们不覆盖的内置字段。逃生舱：规则文件中的 `"extends": null` 触发完全替换语义。匹配 eslint/tsconfig/.gitignore 的心智模型——覆盖一部分而不丢失其余部分。

**代码质量（第 2 节）：**
7. **每规则正则超时，无 RE2 依赖。** 通过 50ms AbortSignal 预算运行每个规则的正则；超时时跳过规则并记录 `meta.regexTimedOut: [ruleId]`。避免 WASM 依赖并保持规则作者语法不受约束。
8. **预编译规则包。** `gstack compact install` 和 `gstack compact reload` 产生 `~/.gstack/compact/rules.bundle.json`（深度合并，正则编译元数据缓存）。钩子读取那个单文件而不是解析 N 个源文件。
9. **mtime 漂移时自动重载。** 钩子在启动时统计规则源文件；如果任何源文件比包新，内联重建后再应用。每次调用增加 ~0.5ms 但消除了"我编辑了规则但什么都没变"的脚枪。
10. **扩展的 v1 编辑集。** Tee 文件编辑：AWS key、GitHub token（`ghp_/gho_/ghs_/ghu_`）、GitLab token（`glpat-`）、Slack webhook、通用 JWT（三个 base64 段）、通用 bearer token、SSH 私钥头（`-----BEGIN * PRIVATE KEY-----`）。信用卡 / SSN / 每 key 环境对推迟到 v2 的完整 DLP 层。

**测试（第 3 节）：**
11. **P-series 门控子集。** v1 gate-tier P 测试：P1（二进制垃圾）、P3（空输出）、P6（RTK-killer 关键栈帧）、P8（secrets to tee）、P15（钩子超时）、P18（prompt injection）、P26（畸形用户规则 JSON）、P28（正则 DoS）、P30（Haiku 幻觉）。剩余 21 个 P-case 随已发版 bug 增长为 R 系列。
12. **fixture 版本戳。** 每个 golden fixture 有 `toolVersion:` frontmatter。CI 在 fixture toolVersion ≠ 当前安装版本时警告。不再基于日历的轮换。
13. **B-series 真实世界基准测试平台（硬 v1 门控）。** 新组件 `compact/benchmark/` 扫描 `~/.claude/projects/**/*.jsonl`，排名最嘈杂的工具调用，将它们聚类为命名场景，针对它们重放压缩器，并报告按规则家族的缩减。v1 直到 B-series 在作者自己的 30 天语料库上显示 ≥15% 缩减 AND 在植入 bug 上零关键行丢失时才能发版。仅本地；不上传。社区共享语料库是 v2。

**性能（第 4 节）：**
14. **修订的延迟预算。** macOS ARM 上 Bun 冷启动为 15-25ms；原始 10ms p50 目标不现实。新预算：macOS ARM <30ms p50 / <80ms p99，Linux <20ms p50 / <60ms p99（验证器关闭）。验证器触发预算保持 <600ms p50 / <2s p99。守护进程模式是一个 v2 选项，取决于 B-series 显示冷启动伤害会话总节省。
15. **面向行的流式管线。** Readline over stdin → filter → group → dedupe → 环形缓冲尾部截断 → stdout。任何单行 >1MB 触发 P9（截断到 1KB，带 `[... truncated ...]` 标记）。无论总输出大小如何，内存上限 64MB。

上面的每一行都是实现中的 `MUST`。偏差需要新的工程评审。

---

## 摘要

`gstack compact` 被设计为在工具输出到达 AI 编程智能体的上下文窗口之前减少其噪声的 `PostToolUse` 钩子。确定性 JSON 规则会缩小嘈杂的测试运行器、构建日志、git diff 和包安装。条件 Claude Haiku 验证器会在过度压缩风险高时充当安全网。

**当前状态：TABLED。** 见上方"状态"部分。架构依赖于截至 2026-04-17 不存在的 Claude Code API（`updatedBuiltinToolOutput` 或内置工具的等效物）。Anthropic issue #36843 跟踪该差距。

**预期目标（为取消搁置 sprint 保留）：** 每次长会话减少 15-30% 工具输出 token，任务失败率零增加。

**原始楔子（vs RTK，28K star 现有者）——均被研究无效化：**
1. ~~**条件 LLM 验证器。~~** 技术上仍可通过 PreToolUse 命令包装实现，但仅用于 Bash。一旦我们也仅 Bash 就不再是差异化因素。如果内置工具 API 到达则重新考虑。
2. ~~**原生工具覆盖。~~** 今天架构上不可能。Read/Grep/Glob 在 Claude Code 进程中执行且不触发钩子。即使对于确实触发 `PostToolUse` 的工具，也没有用于非 MCP 工具的输出替换字段。

**原始定位（现已过时）：** *"RTK 快。gstack compact 快 AND 安全，覆盖你工具箱中的每个工具，不只是 Bash。"*

## 非目标

- 总结用户消息或先前的智能体轮次（Claude 自己的 Compaction API 拥有那个）。
- 压缩智能体响应输出（caveman 的层）。
- 缓存工具调用以避免重新执行（token-optimizer-mcp 的层）。
- 充当通用日志分析器。
- 替换智能体关于何时用 `GSTACK_RAW=1` 重新运行命令自己的判断。

## 为什么值得构建

**问题是可测量的，不是假设的。**

- [Chroma 研究 (2025)](https://research.trychroma.com/context-rot) 测试了 18 个前沿模型。每个模型都随上下文增长而退化。退化在窗口限制之前就开始了——200K 模型在 50K 就开始退化。
- 编码智能体是最坏情况：累积上下文 + 高干扰密度 + 长任务范围。工具输出被明确命名为主要噪声源。
- 市场已投票：Anthropic 发版了 Opus 4.6 Compaction API；OpenAI 发版了 compaction 指南；Google ADK 发版了上下文压缩；LangChain 发版了自主压缩；sst/opencode 内置了 compaction。混合确定性 + LLM 模式是行业共识。

**现有领域（gstack compact 加入和差异化的地方）：**

| 项目 | Stars | License | 层 | 威胁 | 备注 |
|------|-------|---------|-----|------|------|
| **RTK (rtk-ai/rtk)** | **28K** | Apache-2.0 | 工具输出 | 主要基准 | 纯 Rust，仅 Bash，零 LLM |
| caveman | 34.8K | MIT | 输出 token | 不同轴 | 简洁系统提示；与我们配对 |
| claude-token-efficient | 4.3K | MIT | 响应冗余度 | 不同轴 | 单 CLAUDE.md |
| token-optimizer-mcp | 49 | MIT | MCP 缓存 | 不同轴 | 防止调用而非压缩输出 |
| tokenjuice | ~12 | MIT | 工具输出 | 太新 | 2 天前创建；启发了我们的 JSON 信封 |
| 6-Layer Token Savings Stack | — | Public gist | Recipe | Zero | 文档；验证了堆叠压缩论点 |

RTK 是唯一直接竞争对手。其他所有工具压缩不同的 token 源。

**License 兼容性：** 每个引用的项目都是宽松许可的（MIT 或 Apache-2.0），与 gstack 的 MIT 兼容。无 AGPL、GPL 或 other copyleft 依赖。见下方的"License & attribution"部分了解净室策略。

## 架构

### 数据流

```
┌─────────────────────────────────────────────────────────────────┐
│  Host (Claude Code / Codex / OpenClaw)                          │
│  ─────────────────────────────────────────                      │
│  1. 智能体请求工具调用：Bash|Read|Grep|Glob|MCP                  │
│  2. Host 执行工具                                                │
│  3. Host 调用 PostToolUse 钩子，传入：{tool, input, output}      │
└────────────────────┬────────────────────────────────────────────┘
                     │ stdin (JSON envelope)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  gstack-compact hook 二进制文件                                  │
│  ───────────────────────────                                    │
│  a. 解析信封                                                     │
│  b. 按 (tool, command, pattern) 匹配规则                         │
│  c. 应用规则原语：filter / group / truncate / dedupe             │
│  d. 记录缩减元数据                                               │
│  e. 评估验证器触发器                                             │
│  f. 如果触发：调用 Haiku，追加保留的行                           │
│  g. 在失败退出码时：tee 原始到 ~/.gstack/compact/tee/...         │
│  h. 向 stdout 发送 JSON envelope                                │
└────────────────────┬────────────────────────────────────────────┘
                     │ stdout (JSON envelope)
                     ▼
              Host 将压缩后的输出替换到智能体上下文中
```

### 规则解析

三层层次结构（最高优先级获胜），与 tokenjuice 和 gstack 现有的 host-config-export 模型相同模式：

1. 内置规则：随 gstack 发版的 `compact/rules/`
2. 用户规则：`~/.config/gstack/compact-rules/`
3. 项目规则：`.gstack/compact-rules/`

规则按规则 ID 匹配工具调用。ID 为 `tests/jest` 的项目规则完全覆盖内置 `tests/jest`。不合并——替换语义，保持推理简单。

### JSON envelope 契约（从 tokenjuice 采纳）

输入：
```json
{
  "tool": "Bash",
  "command": "bun test test/billing.test.ts",
  "argv": ["bun", "test", "test/billing.test.ts"],
  "combinedText": "...",
  "exitCode": 1,
  "cwd": "/Users/garry/proj",
  "host": "claude-code"
}
```

输出：
```json
{
  "reduced": "compacted output with [gstack-compact: N → M lines, rule: X] header",
  "meta": {
    "rule": "tests/jest",
    "linesBefore": 247,
    "linesAfter": 18,
    "bytesBefore": 18234,
    "bytesAfter": 892,
    "verifierFired": false,
    "teeFile": null,
    "durationMs": 8
  }
}
```

### 规则 schema

紧凑、最小。磁盘上总规则负载必须保持 <5KB（来自 claude-token-efficient 的教训：规则文件本身每次会话都消耗 token）。

```json
{
  "id": "tests/jest",
  "family": "test-results",
  "description": "Jest/Vitest output — preserve failures and summary counts",
  "match": {
    "tools": ["Bash"],
    "commands": ["jest", "vitest", "bun test"],
    "patterns": ["jest", "vitest", "PASS", "FAIL"]
  },
  "primitives": {
    "filter": {
      "strip": ["\\x1b\\[[0-9;]*m", "^\\s*at .+node_modules"],
      "keep": ["FAIL", "PASS", "Error:", "Expected:", "Received:", "✓", "✗", "Tests:"]
    },
    "group": {
      "by": "error-kind",
      "header": "Errors grouped by type:"
    },
    "truncate": {
      "headLines": 5,
      "tailLines": 15,
      "onFailure": { "headLines": 20, "tailLines": 30 }
    },
    "dedupe": {
      "pattern": "^\\s*$",
      "format": "[... {count} blank lines ...]"
    }
  },
  "tee": {
    "onExit": "nonzero",
    "maxBytes": 1048576
  },
  "counters": [
    { "name": "failed", "pattern": "^FAIL\\s", "flags": "m" },
    { "name": "passed", "pattern": "^PASS\\s", "flags": "m" }
  ]
}
```

四个原语——`filter`、`group`、`truncate`、`dedupe`——直接从 RTK 的技术分类法移植（每个严肃压缩器都需要的唯一东西）。任何规则都可以组合四个原语中任意子集；省略的原语是 no-op。

### 验证器层（分层，opt-in）

验证器是一个廉价的 Haiku 调用，仅在特定触发器下触发。从不在每次工具调用上触发。

**触发器矩阵（用户可配置）：**

| 触发器 | 默认 | 条件 |
|--------|------|------|
| `failureCompaction` | **ON** | exit code ≠ 0 AND reduction >50%（诊断处于风险中） |
| `aggressiveReduction` | off | reduction >80% AND original >200 lines |
| `largeNoMatch` | off | 无规则匹配 AND output >500 lines |
| `userOptIn` | on (env-gated) | `GSTACK_COMPACT_VERIFY=1` 强制该调用使用验证器 |

默认配置仅发版 `failureCompaction`——最高杠杆情况（智能体正在调试；规则可能已过滤关键栈帧）。

**Haiku 的工作（有界）：**

```
这是原始输出（截断到前 2000 行）和一个压缩版本。
返回原始中缺失于压缩版本中的重要行，
或返回 `NONE` 如果没有关键内容缺失。
```

验证器从不重写压缩输出。它只在标题下追加缺失的行：

```
[gstack-compact: 247 → 18 lines, rule: tests/jest]
[gstack-verify: 2 additional lines preserved by Haiku]
  TypeError: Cannot read property 'foo' of undefined
    at parseConfig (src/config.ts:42:18)
```

**为什么用 Haiku 而不是 Sonnet：** ~1/12 成本，~500ms vs ~2s，而且任务是简单的子字符串分类，不是推理。

**验证器配置（`compact/rules/_verifier.json`）：**

```json
{
  "verifier": {
    "enabled": true,
    "model": "claude-haiku-4-5-20251001",
    "maxInputLines": 2000,
    "triggers": {
      "aggressiveReduction": { "enabled": false, "thresholdPct": 80, "minLines": 200 },
      "failureCompaction":   { "enabled": true,  "minReductionPct": 50 },
      "largeNoMatch":        { "enabled": false, "minLines": 500 },
      "userOptIn":           { "enabled": true, "envVar": "GSTACK_COMPACT_VERIFY" }
    },
    "fallback": "passthrough"
  }
}
```

**故障模式（验证器严格是加法性的——从不破坏基线）：**

- 无 `ANTHROPIC_API_KEY` → 跳过验证器，使用纯规则输出。
- Haiku 调用超时（>5s）→ 跳过验证器，使用纯规则输出。
- Haiku 返回畸形 JSON → 跳过，使用纯规则输出。
- Haiku 返回 prompt injection 尝试 → 消毒：仅追加原始 raw output 中存在的行。
- Haiku 返回幻觉行（不存在于原始输出中）→ 丢弃。

### Tee 模式（从 RTK 采纳）

在任何 exit code ≠ 0 的命令上，完整的未过滤输出被写入 `~/.gstack/compact/tee/{timestamp}_{cmd-slug}.log`。压缩输出包含 tee 文件指针：

```
[gstack-compact: 247 → 18 lines, rule: tests/jet, tee: ~/.gstack/compact/tee/20260416-143022_bun-test.log]
```

智能体可以直接读取 tee 文件，如果它需要完整的栈追踪。这用更干净的设计替换了早期的 `onFailure.preserveFull` 机制：压缩输出始终保持小；原始输出始终只需一次 `cat`。

**Tee 安全：**

- 文件模式 `0600`——非全局可读。
- 内置 secret-regex 在写入前编辑 AWS key、bearer token 和常见凭证模式。
- 写入失败（只读文件系统、权限拒绝）优雅降级：仍发送压缩输出，记录 `meta.teeFailed: true`。
- Tee 文件 7 天后自动过期（钩子启动时清理）。

### Host 集成矩阵

| Host | 钩子类型 | 支持匹配器 | 配置路径 |
|------|----------|------------|----------|
| Claude Code | `PostToolUse` | Bash, Read, Grep, Glob, Edit, Write, WebFetch, WebSearch, mcp__* | `~/.claude/settings.json` |
| Codex (v1.1) | `PostToolUse` 等效 | Bash（主要）；工具子集待定——经验验证是 v1.1 前置条件 | `~/.codex/hooks.json` |
| OpenClaw (v1.1) | 原生钩子 API | Bash + MCP | OpenClaw 配置 |

**v1 is Claude-first.** 楔子（ii）——原生工具覆盖——通过 [hooks 参考](https://code.claude.com/docs/en/hooks) 在 Claude Code 上确认。Codex 和 OpenClaw 集成仅在主 host 通过 B-series 基准数据证明楔子后在 v1.1 发版。v1 的 CHANGELOG 明确说明仅 Claude 的范围。

### 配置表面

用户配置（`~/.config/gstack/compact.toml`）：

```toml
[compact]
enabled = true
level = "normal"                            # minimal | normal | aggressive（caveman 模式）
exclude_commands = ["curl", "playwright"]   # RTK 模式

[compact.bundle]
auto_reload_on_mtime_drift = true           # 如果源规则文件更新则钩子重建包
bundle_path = "~/.gstack/compact/rules.bundle.json"

[compact.regex]
per_rule_timeout_ms = 50                    # 每正则 AbortSignal 预算；超时 → 跳过规则

[compact.verifier]
enabled = true
trigger_failure_compaction = true
trigger_aggressive_reduction = false
trigger_large_no_match = false
failure_signal_fallback = true              # 缺少 exitCode 时使用 /FAIL|Error|Traceback|panic/
sanitization = "exact-line-match"           # 仅追加原始输出中逐字存在的行

[compact.tee]
on_exit = "nonzero"
max_bytes = 1048576
redact_patterns = ["aws", "github", "gitlab", "slack", "jwt", "bearer", "ssh-private-key"]
cleanup_days = 7

[compact.benchmark]
local_only = true                           # 硬编码；配置是文档性的，不能更改
transcript_root = "~/.claude/projects"
output_dir = "~/.gstack/compact/benchmark"
scenario_cap = 20                           # 按总输出量排名的 top-N 聚类
```

**强度级别（caveman 模式）：**

- **minimal：** 仅 `filter` + `dedupe`；无截断。最安全。
- **normal：** `filter` + `dedupe` + `truncate`。默认。
- **aggressive：** 添加 `group`；更多节省，更多边缘案例风险。

### CLI 表面

| 命令 | 用途 | 来源 |
|------|------|------|
| `gstack compact install <host>` | 在 host 配置中注册 PostToolUse 钩子；构建 `rules.bundle.json` | 新 |
| `gstack compact uninstall <host>` | 幂等移除 | 新 |
| `gstack compact reload` | 编辑用户/项目规则后重建 `rules.bundle.json` | 新 |
| `gstack compact doctor` | 检测漂移 / 损坏的钩子配置，提供修复 | tokenjuice |
| `gstack compact gain` | 显示随时间节省的 token/美元（按规则细分） | RTK |
| `gstack compact discover` | 查找无匹配规则的命令，按噪声量排名 | RTK |
| `gstack compact verify <rule-id>` | 在 fixture 上 dry-run 验证器 | 新 |
| `gstack compact list-rules` | 显示深度合并后的有效规则集（内置 + 用户 + 项目） | 新 |
| `gstack compact test <rule-id> <fixture>` | 将规则应用于 fixture 并显示 diff | 新 |
| `gstack compact benchmark` | 在本地 transcript 语料库上运行 B-series 测试平台（见基准部分） | 新 |

逃生舱：`GSTACK_RAW=1` 环境变量绕过钩子，对该命令的整个持续时间（与 tokenjuice 的 `--raw` 标志相同模式）。如果任何源规则文件的 mtime 比包文件更新，钩子也会自动重载包。

## 文件布局

```

compact/
├── SKILL.md.tmpl              # 模板；通过 `bun run gen:skill-docs` 重新生成
├── src/
│   ├── hook.ts                # 入口点；读取 stdin，写入 stdout；mtime 检查包
│   ├── engine.ts              # 规则匹配 + 缩减元数据
│   ├── apply.ts               # 原语应用（面向行的流式管线）
│   ├── merge.ts               # 内置/用户/项目规则的深度合并；遵守 `extends: null`
│   ├── bundle.ts              # 编译源规则 → rules.bundle.json（install/reload）
│   ├── primitives/
│   │   ├── filter.ts
│   │   ├── group.ts
│   │   ├── truncate.ts        # 环形缓冲尾部；对任意输入大小安全
│   │   └── dedupe.ts
│   ├── regex-sandbox.ts       # AbortSignal 约束的正则执行（每规则 50ms 预算）
│   ├── verifier.ts            # Haiku 集成（触发器 + failure-signal 回退 + 消毒）
│   ├── sanitize.ts            # 验证器输出的精确行匹配过滤
│   ├── tee.ts                 # 带 secret 编辑 + 7 天清理的原始输出归档
│   ├── redact.ts              # secret 模式集（AWS/GitHub/GitLab/Slack/JWT/bearer/SSH）
│   ├── envelope.ts            # JSON I/O 契约解析 + 验证
│   ├── doctor.ts              # 钩子漂移检测 + 修复
│   ├── analytics.ts           # 针对本地元数据的 gain + discover 查询
│   └── cli.ts                 # argv 分发；每个子命令一个薄分发
├── benchmark/                 # B-series 测试平台（硬 v1 门控）
│   └── src/
│       ├── scanner.ts         # 遍历 ~/.claude/projects/**/*.jsonl；配对 tool_use × tool_result
│       ├── sizer.ts           # 每次调用的 token（ceil(len/4) 启发式）；排名重尾
│       ├── cluster.ts         # 按（工具，命令模式）对高杠杆调用分组
│       ├── scenarios.ts       # 发射 B1-Bn 真实世界场景 fixture
│       ├── replay.ts          # 针对场景运行压缩器；测量缩减
│       ├── pathology.ts       # 在真实场景上叠加植入 bug 的 P-case
│       └── report.ts          # 仪表盘：每场景 before/after + 整体缩减
├── rules/                     # v1 内置 JSON 规则库（13 规则）
│   ├── tests/
│   │   ├── jest.json
│   │   ├── vitest.json
│   │   ├── pytest.json
│   │   ├── cargo-test.json
│   │   ├── go-test.json
│   │   └── rspec.json
│   ├── install/
│   │   ├── npm.json
│   │   ├── pnpm.json
│   │   ├── pip.json
│   │   └── cargo.json
│   ├── git/
│   │   ├── diff.json
│   │   ├── log.json
│   │   └── status.json
│   ├── _verifier.json         # 验证器配置（严格来说不是规则）
│   └── _HOLD/                 # v1.1 规则家族（在 v1 不发版；保留供参考）
│       ├── build/
│       ├── lint/
│       └── log/
└── test/
    ├── unit/
    ├── golden/
    ├── fuzz/                  # P-series — 仅 v1 门控子集（P1/P3/P6/P8/P15/P18/P26/P28/P30）
    ├── cross-host/            # v1：仅 claude-code.test.ts；codex/openclaw 存根文件
    ├── adversarial/           # R-series — 随已发版 bug 增长
    ├── benchmark/             # B-series 场景 fixture + 预期缩减范围
    ├── fixtures/              # 版本戳记的 golden 输入（toolVersion: frontmatter）
    └── evals/

```

## 测试策略

测试计划按设计全面。进入一个有 28K star 现有者拥有三年正则伤疤的空间，我们的楔子（Haiku 验证器 + 原生工具覆盖）引入了新的故障表面，意味着我们在"压缩器让我的智能体变笨"病毒式传播上只有一次机会。对此零容忍。

### 测试层级

| 层级 | 成本 | 频率 | 阻止合并 |
|------|------|------|----------|
| 单元 | free, <1s | 每次 PR | yes |
| Golden file（带 `toolVersion:` frontmatter） | free, <1s | 每次 PR | yes |
| 规则 schema 验证 | free, <1s | 每次 PR | yes |
| Fuzz（P-series 门控子集：P1/P3/P6/P8/P15/P18/P26/P28/P30） | free, <10s | 每次 PR | yes |
| 跨 host E2E — v1 仅 Claude Code | free, ~1min | 每次 PR（gate tier） | yes |
| 带验证器的 E2E（mock Haiku） | free, ~15s | 每次 PR | yes |
| 带验证器的 E2E（真实 Haiku） | paid, ~$0.10/run | 触碰验证器文件的 PR | yes |
| **B-series 基准（真实世界场景）** | **free, ~2min** | **预发版门控** | **yes（v1 硬门控）** |
| Token 节省 eval（E1-E4 合成） | paid, ~$4/run | 每周定期 | no（信息性） |
| 对抗性回归（R-series） | free, <5s | 每次 PR | yes |
| 工具版本漂移警告 | free, <1s | 每次 PR | 仅警告 |

测试文件布局：

```

compact/test/
├── unit/
│   ├── engine.test.ts         # 规则匹配 + 原语应用
│   ├── primitives.test.ts     # filter / group / truncate / dedupe
│   ├── envelope.test.ts       # JSON 输入/输出契约
│   ├── triggers.test.ts       # 验证器触发器评估
│   └── verifier.test.ts       # Haiku 调用（mock）
├── golden/
│   ├── tests/                 # 每个测试运行器一个 fixture
│   │   ├── jest-success.input.txt
│   │   ├── jest-success.expected.txt
│   │   ├── jest-fail.input.txt
│   │   ├── jest-fail.expected.txt
│   │   └── ...（vitest, pytest, cargo-test, go-test, rspec）
│   ├── install/
│   ├── git/
│   ├── build/
│   ├── lint/
│   └── log/
├── fuzz/
│   └── pathological.test.ts   # P-series
├── cross-host/
│   ├── claude-code.test.ts
│   ├── codex.test.ts
│   └── openclaw.test.ts
├── adversarial/
│   └── regression.test.ts     # R-series；过去必须永不复发的 bug
├── fixtures/
│   └── {tool}/                # 共享原始输出 fixture
└── evals/
    └── token-savings.eval.ts  # 定期层级；测量真实缩减

```

### G-series：好情况（必须产生预期缩减）

| ID | 场景 | 预期缩减 |
|----|------|----------|
| G1 | `jest` 47 通过测试，干净运行 | 150+ 行 → ≤10 行 |
| G2 | `jest` 47 测试含 2 失败 | 200+ 行 → 保留两个失败 + 摘要 |
| G3 | `vitest` 运行带 `--reporter=verbose` | 300+ 行 → ≤15 行 |
| G4 | `pytest` 收集然后运行 | 保留失败追溯 |
| G5 | `cargo test` 有一个 panic | panic 位置逐字保留 |
| G6 | `go test -v` 带 200 子测试通过 | 折叠为 `PASS: 200 subtests` |
| G7 | `git diff` 在 500 行上下文中有 2 hunk | 保留 hunk，丢弃上下文 |
| G8 | `git log -50` | 保留 SHA + subject + author，丢弃 body |
| G9 | `git status` 有 30 已修改文件 | 按目录分组 |
| G10 | `pnpm install` 全新安装 | 最终计数 + 警告；丢弃已解析包 |
| G11 | `pip install -r requirements.txt` | 丢弃下载进度；保留最终安装列表 + 错误 |
| G12 | `cargo build` 成功 | 丢弃编译进度；保留最终目标 |
| G13 | `docker build` 成功 | 丢弃 layer pull；保留最终 image digest |
| G14 | `tsc --noEmit` 干净 | 压缩为 `tsc: 0 errors` |
| G15 | `tsc --noEmit` 带 3 错误 | 保留所有 3 错误及位置 |
| G16 | `eslint .` 干净 | 压缩为 `eslint: 0 problems` |
| G17 | `eslint .` 有违规 | 按规则分组；保留位置 + 修复建议 |
| G18 | `docker logs -f` 带 1000 行重复行 | dedupe 带计数：`[last message repeated 973 times]` |
| G19 | `kubectl get pods -A` | 按命名空间分组 |
| G20 | `ls -la` 深层树 | 按目录分组（RTK 模式） |
| G21 | `find . -type f` 10K 文件 | 按扩展名分组带计数 |
| G22 | `grep -r "foo" .` 带 500 命中 | 上限 50；后缀 `[... 450 more matches; use --ripgrep for full]` |
| G23 | `curl -v https://api.example.com` | 剥离 verbose 头；保留响应 body |
| G24 | `aws ec2 describe-instances` 50 实例 | 列式摘要 |

### P-series：病态情况（绝不能破坏智能体）

如果我们弄错其中任何一个，就变成"灾难性回归"。

| ID | 场景 | 要求行为 |
|----|------|----------|
| P1 | 输出中的二进制垃圾（非 UTF8 字节） | 原样传递；不崩溃 |
| P2 | ANSI 转义爆炸（10K+ 代码） | 干净剥离，不卡住正则 |
| P3 | 空输出 (`""`) | 传递空；**不要**注入头 |
| P4 | Stdstdout+stderr 交错 | 规则在两个流上匹配 |
| P5 | 截断输出（SIGPIPE 中途） | 不误压缩部分输出 |
| P6 | **失败测试，关键栈帧在第 4/200 行** | 必须不过滤该帧（RTK-killer 案例） |
| P7 | Exit 0 但输出中有 `ERROR:` | 规则不能仅信任 exit code |
| P8 | 输出包含 AWS key / bearer token / password | Tee 文件必须不能全局可读；在压缩输出中编辑 |
| P9 | 单行压缩 JS 错误（40KB 一行） | 截断到前 1KB；追加 `[... truncated ...]` |
| P10 | Unicode（emoji、RTL、组合字符、CJK） | 字节省略；不分割代码点 |
| P11 | 两个规则匹配同一命令 | 确定性优先级：最长 `match.commands` 前缀获胜；平局 → 规则 ID 字母顺序 |
| P12 | 规则的压缩输出匹配另一个规则的图案 | 不递归应用；钩子每次工具调用运行一次 |
| P13 | 命令包含引号中嵌入的换行符 | 规则不误解析 argv |
| P14 | 并发工具调用（并行 Bash 调用） | 钩子中无可变共享状态；每次调用隔离 |
| P15 | 钩子执行 >5s | 传递原始；发出 `meta.timedOut: true` |
| P16 | Haiku API 离线/限流 | 静默跳过验证器；使用纯规则输出 |
| P17 | Haiku 返回畸形 JSON | 跳过验证器；不要将原始响应喂给智能体 |
| P18 | Haiku 响应包含 prompt injection（`"Ignore all prior instructions..."`） | 消毒：仅追加原始输出的子字符串匹配行 |
| P19 | 1M 行输出 | 流处理，内存上限 64MB；截断带清晰标记 |
| P20 | 快速连发：50 工具调用 / 秒 | 钩子延迟保持 <15ms p99 |
| P21 | 命令包含 shell 重定向（`cmd >file 2>&1`） | 匹配底层命令名，非重定向包装器 |
| P22 | 命令字符串中深层嵌套的引号/转义 | 稳健 argv 解析器；无 shell 注入可能 |
| P23 | NULL 字节在输出中 | 安全剥离；不截断 |
| P24 | 命令退出然后写入更多到 stderr | 钩子接收最终组合输出；优雅处理 |
| P25 | 只读文件系统 / 无 tee 写入权限 | 优雅降级；仍发出压缩输出；记录 `meta.teeFailed: true` |
| P26 | 用户的规则 JSON 畸形 | 跳过该规则；发出 stderr 警告；不破坏钩子 |
| P27 | 规则引用不存在的原语字段 | 忽略未知字段；应用规则其余部分 |
| P28 | 规则正则灾难性回溯 | RE2 兼容引擎（无回溯）或每规则超时 |
| P29 | Exit code 137（OOM kill） | 规则视为通用失败；保留完整输出 |
| P30 | Haiku 返回不在原始输出中的行（幻觉） | 丢弃幻觉行；仅保留子字符串匹配 |

### CH-series：跨 host E2E

在每个支持的 host 上运行每个场景。相同输入，相同预期输出。如果 host 不支持匹配器，测试标记为 `skip-on-{host}`，带注释链接上游限制。

| ID | 场景 | Host |
|----|------|------|
| CH1 | 通过 `gstack compact install <host>` 安装钩子 | Claude Code, Codex, OpenClaw |
| CH2 | 卸载钩子是幂等的 | 所有 |
| CH3 | 重新安装不重复条目 | 所有 |
| CH4 | 钩子与用户的其他 PostToolUse 钩子共存 | 所有 |
| CH5 | 钩子在 Bash 工具上触发 | 所有 |
| CH6 | 钩子在 Read 工具上触发 | Claude Code（已确认）；Codex/OpenClaw verify-then-require |
| CH7 | 钩子在 Grep 工具上触发 | 同 CH6 |
| CH8 | 钩子在 Glob 工具上触发 | 同 CH6 |
| CH9 | 钩子在 MCP 工具上触发（`mcp__*` 匹配器） | Claude Code；在其他上验证 |
| CH10 | 配置优先级：project > user > built-in | 所有 |
| CH11 | `GSTACK_RAW=1` 环境变量绕过钩子 | 所有 |
| CH12 | 规则 ID 替换工作（项目规则覆盖内置） | 所有 |
| CH13 | `gstack compact doctor` 在每个 host 上检测漂移 | 所有 |
| CH14 | 钩子错误不崩溃智能体会话 | 所有 |

实现说明：跨 host 测试重用 `golden/` 树中的 fixture 语料库；harness 将每个 fixture 包装在 host 特定的钩子调用信封中，并断言输出跨 host 字节级相同（除了 `host` 字段）。

### V-series：验证器测试（付费）

| ID | 场景 | 预期 |
|----|------|------|
| V1 | 规则将 200 行测试输出减为 5 行，exit=1 | 验证器触发（失败 + >50% 追加），追加任何缺失的关键行 |
| V2 | 规则将 10 行输出减为 9 行，exit=1 | 验证器**不触发**（缩减太小） |
| V3 | 规则将 200 行输出减为 5 行，exit=0 | 验证器**不触发**（成功路径，默认配置） |
| V4 | `aggressiveReduction` 触发启用，300 行 → 20 行，exit=0 | 验证器触发 |
| V5 | `GSTACK_COMPACT_VERIFY=1` 环境变量设置 | 验证器对该调用触发一次 |
| V6 | `ANTHROPIC_API_KEY` 缺失 | 验证器静默跳过；返回纯规则输出 |
| V7 | 验证器 mock 返回 "NONE" | 输出与纯规则路径相同 |
| V8 | 验证器 mock 返回 prompt injection | 注入被丢弃；仅追加子字符串匹配行 |
| V9 | 验证器 mock 超时 >5s | 跳过；`meta.verifierTimedOut: true` |
| V10 | 验证器 mock 返回 500 错误 | 跳过；返回规则输出 |

### R-series：对抗性回归

v1 发版后捕获的每个 bug 获得一个永久 R 系列测试。从空开始；随伤疤增长。模板：

```

{R{N}: {commit-sha} — {1-line summary}
Scenario: {reproducer}
Fix: {PR link}
}

```

### 性能预算（在 CI 中执行；针对真实 Bun 冷启动修订）

| 指标 | 目标 | 硬限制 |
|------|------|--------|
| 钩子开销 macOS ARM（验证器禁用） | <30ms p50 | <80ms p99 |
| 钩子开销 Linux（验证器禁用） | <20ms p50 | <60ms p99 |
| 钩子开销（验证器触发） | <600ms p50 | <2s p99 |
| 包反序列化（rules.bundle.json） | <2ms | <10ms |
| mtime 漂移检查（源文件 stat） | <0.5ms | <3ms |
| 单正则执行预算（每规则） | <5ms | <50ms（硬中止） |
| 每次钩子调用内存（行流式） | <16MB typical | <64MB max |
| 磁盘上总规则负载大小（源文件） | <5KB | <15KB |
| 磁盘上编译包大小 | <25KB | <80KB |

守护进程模式是一个 v2 优化。如果 B-series 基准在作者语料库上显示冷启动显著伤害会话总节省（例如总钩子开销 >5% 节省 token 的挂钟时间），提升至 v1.1。

### B-series 真实世界基准测试平台（硬 v1 门控）

**为什么存在。** 每个竞争压缩器都发版带有手工挑选的 fixture 数字。B-series 证明压缩器在用户**实际**编码会话上工作，在他们启用钩子之前。它既是发版门控，也是营销产物。

**架构**（`compact/benchmark/src/` 中的组件）：

```

┌──────────────────────────────────────────────────────────────┐
│  1. SCAN     scanner.ts 遍历 ~/.claude/projects/**/*.jsonl   │
│              → 配对 tool_use × tool_result 块                │
│              → 发出 {tool, command, outputBytes, lineCount, │
│                estimatedTokens, sessionId, timestamp}        │
├──────────────────────────────────────────────────────────────┤
│  2. RANK     sizer.ts 按 estimatedTokens desc 排序语料库      │
│              → cluster.ts 按 (tool, command-pattern) 分组    │
│              → 识别重尾：哪 10% 的调用产生了 80% 的 token？ │
├──────────────────────────────────────────────────────────────┤
│  3. SCENARIO scenarios.ts 发出 fixture 文件：                 │
│              B1_bun_test_heavy.jsonl                         │
│              B2_git_diff_huge.jsonl                          │
│              B3_tsc_errors_production.jsonl                  │
│              B4_pnpm_install_fresh.jsonl ... （每个高杠杆   │
│              聚类一个，最多 ~20 场景）                        │
├──────────────────────────────────────────────────────────────┤
│  4. REPLAY   replay.ts 针对每个场景运行压缩器，              │
│              测量 token 缩减 + 丢弃行的 diff                  │
│              → 按规则缩减数字                                │
│              → 每场景 before/after token 计数                │
├──────────────────────────────────────────────────────────────┤
│  5. PATHOLOGY pathology.ts 注入植入的关键行                  │
│              （失败测试 fixture 中 200 行的第 4 行）到真实   │
│              B-场景。确认验证器恢复它们。                     │
│              真实数据 + 真实威胁 = 真实证明。                │
├──────────────────────────────────────────────────────────────┤
│  6. REPORT   report.ts 发出 HTML + JSON 仪表盘到             │
│              ~/.gstack/compact/benchmark/latest/              │
│              "在你自己的 30 天 Claude Code 数据上，gstack    │
│              compact 会在 Y 个场景中节省 X 个 token。"       │
└──────────────────────────────────────────────────────────────┘

```

**v1 发版门控（硬）：**
- ≥15% 总 token 缩减，跨作者自己 30 天 transcript 集聚合的场景语料库。
- 植入 bug 场景上零关键行丢失（每个植入栈帧必须被规则或验证器之一保留）。
- 新规则下无场景回归到 <5% 缩减（捕获过度压缩边缘案例）。

**隐私（不可协商）：**
- 仅本地读取 `~/.claude/projects/**/*.jsonl`。从不上传。从不分享。从不对遥测记录场景。
- 输出文件位于 `~/.gstack/compact/benchmark/`，模式 `0600`。
- 命令打印确认横幅：*"扫描 ~/.claude/projects/ 中的本地 transcript（仅本地；此机器上无物离开）。"*
- 任何未来社区语料库都是利用在 OSS 项目上手工贡献的、经秘密扫描的 fixture 构建的独立 v2 工作流。

**从 analyze_transcripts 移植的端口（TypeScript 重新实现；非子进程调用）：**
- JSONL 解析 + tool_use/tool_result 配对模式（来自 `event_extractor.rb`）。
- Token 估算 `ceil(len/4)`（相同字符比率启发式；足以排名）。
- 事件类型分类法（`bash_command`、`file_read`、`test_run`、`error_encountered`）用于场景聚类。
- fixture 生成模式用于病理层。

**我们不移植的内容：** 行为评分、pgvector 嵌入、决策交换图表、速度指标、Rails/ActivRecord 层。超出范围；不是我们要测量的。

### 合成 token 节省 eval（E-series，定期/仅信息性）

从原始计划保留但现在仅信息性，因为 B-series 是真实门控。

- **E1：** 中型 TypeScript 项目上模拟 30 分钟编码会话。测量启用/禁用 gstack compact 的总 token。目标：≥15% 追加。
- **E2：** 相同会话在 `level=aggressive`。目标：≥25% 追加，零测试失败增加。
- **E3：** 相同会话仅在 `failureCompaction` 上发验证器。验证器触发率 ≤10% 工具调用。
- **E4：** 对抗性——在测试输出中注入植入 bug，确认验证器恢复关键栈帧。

### 测试语料库来源

对于每个规则家族，捕获 3+ 个真实输出：

1. 在真实项目上运行工具（gstack 本身用于 TS；热门 OSS 用于 Rust/Go/Python）。
2. 将 stdout+stderr+exit code 捕获到带 `toolVersion:` frontmatter 的 fixture 文件（例如 `jest@29.7.0`）。
3. 手写预期的压缩输出一次。
4. Golden file 测试：规则应用必须产生字节级相同输出。
5. CI 漂移警告：如果安装的工具版本与 fixture 的 `toolVersion:` 不同，CI 警告（不失败）。漂移警告仪表盘在预发版时检查。

来源：
- tokenjuice 的 fixture 目录模式（`tests/fixtures/`）
- RTK 的每命令示例（它们 README 列出真实 before/after 指标；独立验证）
- gstack 自己的测试 output（吃自己的狗粮）
- 来自 `~/.gstack/compact/tee/` 的真实失败归档（一旦志愿者贡献）
- **B-series 真实场景是缩减测量的主要语料库。**

## 模式采纳表

从竞争格局借用的具体模式：

| 从 | 采纳为 | 为什么 |
|----|--------|--------|
| RTK | 4 缩减原语（filter/group/truncate/dedupe）作为 JSON 规则动词 | 严肃压缩器的标配 |
| RTK | `gstack compact tee` 用于故障模式原始保存 | 优于原始的 `onFailure.preserveFull` 设计 |
| RTK | `gstack compact gain` + `gstack compact discover` | 信任 + 持续改进 |
| RTK | `exclude_commands` 每用户阻止列表 | 必须有的配置 |
| tokenjuice | hook I/O 的 JSON envelope 契约 | 干净的机器适配器 |
| tokenjuice | `gstack compact doctor` | 钩子漂移；自我修复重要 |
| caveman | 强度级别（minimal/normal/aggressive） | 用户可调的安全/节省旋钮 |
| claude-token-efficient | 规则文件大小预算（<5KB 总计） | 不膨胀上下文 |

## 推出计划

**所有阶段 TABLED 等待 Anthropic `updatedBuiltinToolOutput` API。** 见本文档顶部的"状态"部分。下面的推出是 API 发版且此设计取消搁置时的预期顺序。

### 取消搁置检查清单（API 到达时按顺序执行）

1. **确认新 API 的形状。** 阅读更新的 Claude Code hooks 参考。捕获包含新输出替换字段的真实信封，用于 Bash、Read、Grep、Glob。记录在 `docs/designs/GCOMPACTION_envelope.md` 中。
2. **重新验证楔子。** 新 API 是否覆盖 Read/Grep/Glob（它们现在触发 `PostToolUse`），还是仅 Bash/WebFetch？如果仅 Bash，楔子（ii）仍死，产品需要新卖点再实现。
3. 对修订后的计划重新运行 `/plan-eng-review`（带新 API）。15 个锁定决策中的大多数应继续沿用；调整架构数据流和任何信封相关决策。
4. 对修订后的计划重新运行 `/codex review`。先前 BLOCK 结论关于钩子替换的顾虑一旦 API 存在就消失；剩余的严重问题（B-series 隐私、正则 DoS、JSON-envelope 流式）仍适用。
5. 执行下面的原始推出。

### 原始推出（为取消搁置保留）

每个层级在通过所有 gate-tier 测试后阻塞下一个。Claude-first——Codex 和 OpenClaw 仅在主 host 证明楔子后在 v1.1 落地。

1. **v0.0（1 天）：** 规则引擎 + 4 原语 + 面向行的流式管线 + 深度合并 + 包编译器 + envelope 契约 + 仅对 `tests/*` 家族的 golden 测试。暂无 host 集成。在离线 fixture 上测量节省。
2. **v0.1（1 天）：** Claude Code 钩子集成 + `gstack compact install` + mtime 自动重载。发版 opt-in；默认关闭。邀请 10 个 gstack 高级用户尝试；收集反馈。
3. **v0.5（1 天）：** B-series 基准测试平台（`compact/benchmark/`）。发版 `gstack compact benchmark` 让用户在自己的数据上测量。从一开始就匿名收集吃狗粮者的缩减数字（无物上传）。
4. **v1.0（1 天）：** 验证器层带 `failureCompaction` 默认开启 + 精确行匹配消毒 + 分层 exitCode/pattern 回退 + 扩展 tee 编辑集。**硬发版门控：** 作者 30 天本地语料库上的 B-series 显示 ≥15% 总缩减 AND 植入 bug 上零关键行丢失。发版 CHANGELOG 条目，以楔子框架为头条（v1 仅 Claude Code）。
5. **v1.1（+1 天）：** Codex + OpenClaw 钩子集成。跨 host E2E 套件全绿。build/lint/log 规则家族随 `gstack compact discover` 驱动的优先级落地。
6. **v1.2+：** 扩展规则家族、社区规则贡献工作流、社区语料库基准（手工编写公共 fixture，与本地 B-series 分离）。

## 风险分析

| 风险 | 严重性 | 缓解措施 |
|------|--------|----------|
| RTK 添加 LLM 验证器作为回应 | 低 | 创作者公开宣称零依赖 Rust。先发版，构建模式库。 |
| Platform compaction 吞并我们（Claude Code 中的 Anthropic Compaction API） | 中 | 我们在不同层操作（每工具输出 vs 整个上下文）。定位为互补。 |
| 规则丢弃关键内容 → "压缩器让我的智能体变笨" | 高 | B-series 真实世界基准作为硬发版门控；tee 模式始终可用；验证器默认开启处理失败；精确行匹配消毒。 |
| Haiku 成本蔓延（触发器比预期更多触发） | 中 | E3 eval + B-series 触发率指标；成本在 `gstack compact gain` 中可见；在 v1.1 中如率 >10% 则添加每会话速率上限。 |
| 规则维护债务（jest/vitest output 格式变更） | 中 | `toolVersion:` fixture frontmatter + CI 漂移警告；社区规则 PR；`discover` 标记绕过命令。 |
| 规则文件膨胀上下文 | 低 | CI 强制 <5KB 源 + <25KB 编译包预算；在 schema 验证时每规则大小警告。 |
| 正则 DoS 阻塞智能体 | 中 | 每规则 50ms AbortSignal 预算；超时记录到 `meta.regexTimedOut`；重复失败时隔离陈旧规则。 |
| 包陈旧默默地破坏用户编辑 | 低 | 每次钩子调用 mtime 检查自动重建；`gstack compact reload` 是备份不是要求。 |
| 基准泄露用户私有数据 | 高 | 仅本地构建：无网络调用，模式 0600 输出，运行时明确的横幅。在 v1 发版前进行隐私评审。 |

## 开放问题

1. ~~Codex 的 PostToolUse 钩子是否支持 Read/Grep/Glob 匹配器？~~（推迟到 v1.1 — v1 仅 Claude。）
2. ~~OpenClaw 的钩子 API 是否专门支持 PostToolUse？~~（推迟到 v1.1。）
3. 验证器模型应是固定的，还是像 gstack 的其他 AI 调用一样版本跟踪？（倾向于固定 `claude-haiku-4-5-20251001` 并在 CHANGELOG 中显式升级。）
4. ~~Tee 文件的内置 secret 编辑正则集~~**（已解决：扩展集——AWS/GitHub/GitLab/Slack/JWT/bearer/SSH-private-key。见决策 #10。）**
5. `gstack compact discover` 应通过 Haiku 提议自动生成规则吗？（推迟到 v2；技能蠕变风险。）
6. **新：** Claude Code 的 PostToolUse envelope 是否包含 `exitCode`？（仍需按预实现任务 #1 经验验证；无论如何系统现在有分层回退。）
7. **新：** B-series 的正确场景数量上限是什么？Cluster.ts 根据重尾形状可产生 5-50 场景。计划：按总输出量将 top 20 聚类作为上限。

## 预实现任务（编码前必须完成）

1. **经验验证 Claude Code 的 PostToolUse envelope 内容。** 发版一个 noop 钩子；确认 `exitCode`、`command`、`argv`、`combinedText` 都存在。这是楔子（ii）原生工具覆盖和 failureCompaction 触发器的支点。输出：`docs/designs/GCOMPACTION_envelope.md`，含 Bash + Read + Grep + Glob 的真实捕获信封。
2. **阅读 RTK 的规则定义**（`ARCHITECTURE.md`、`src/rules/`）并写 1 段摘要，说明  4 原语中哪个它们处理最好。告知我们的 v1 规则集。这是 Search Before Building 层。
3. **将 analyze_transcripts JSONL 解析器移植到 TypeScript。** `compact/benchmark/src/scanner.ts`。写一个 quick-look 输出，列出作者 `~/.claude/projects/` 上 top-50 最嘈杂的工具调用。在构建重放循环之前确认测试平台前提。这是 B-series 基础。
4. **先写 CHANGELOG 条目。** 目标句子：*"Claude Code 上智能体工具箱中的每个工具现在都产生更少噪声——测试运行器、git diff、包安装——带有一个智能 Haiku 安全网，在我们规则过度压缩时恢复关键栈帧，以及一个在你的实际 30 天编码会话上证明节省的本地基准。Codex + OpenClaw 在 v1.1 落地。"* 如果我们不能诚实写那个句子，楔子还不存在。
5. **发版仅规则的 v0**（无 Haiku 验证器，无基准）。用当前 gstack eval + 早期 B-series 原型测量真实 token 节省。如果在本地语料库上 <10%，整个前提比声称的更弱——在添加验证器之前迭代规则。

## License & attribution

gstack 在 MIT 下发版。为了对下游用户保持 license 干净，本项目对从竞争景观借用的一切遵循严格的净室策略：

- 上面引用的每个项目都是**宽松许可的**（MIT 或 Apache-2.0）。无 AGPL、GPL、SSPL 或其他 copyleft 暴露。
  - RTK (rtk-ai/rtk)：**Apache-2.0** — MIT 兼容；Apache patent grant 是我们的加分项。
  - tokenjuice、caveman、claude-token-efficient、token-optimizer-mcp、sst/opencode：**MIT**。
- **模式，不是代码。** 我们阅读这些项目以了解它们解决了什么以及为什么。我们在 `compact/src/` 中用 TypeScript 独立实现。我们不复制源文件，不逐行翻译源文件，也不字面采用测试 fixture。
- **署名。** 如果模式被直接借用（RTK 的 4 原语、tokenjuice 的 JSON envelope、caveman 的强度级别、claude-token-efficient 的规则文件大小预算），我们在注释中和上方的"模式采纳表"中署名来源。项目的 `README` 和 `NOTICE` 文件（如果我们添加）列出灵感。
- **Fixture 来源。** Golden-file fixture 来自在真实项目上运行真实工具——它们是我们自己的捕获，不是从 RTK 或 tokenjuice 导入的。这保持测试语料库无 license 纠缠内容。
- **禁止来源。** 在添加任何新的引用项目之前，运行 `gh api repos/OWNER/REPO --jq '.license'` 并验证 license key 是以下之一：`mit`、`apache-2.0`、`bsd-2-clause`、`bsd-3-clause`、`isc`、`cc0-1.0`、`unlicense`。如果项目无 license 字段，视为"all rights reserved"并不要从中提取。拒绝 `agpl-3.0`、`gpl-*`、`sspl-*`，以及任何自定义或 source-available license。

CI 执行：`scripts/check-references.ts` 脚本解析 `docs/designs/GCOMPACTION.md` 中的 GitHub URL 并重新运行 license 检查，如果任何引用项目的 license 移出允许列表则失败。

## 参考

- [RTK (Rust Token Killer) — rtk-ai/rtk](https://github.com/rtk-ai/rtk)
- [RTK issue #538 — native-tool gap](https://github.com/rtk-ai/rtk/issues/538)
- [tokenjuice — vincentkoc/tokenjuice](https://github.com/vincentkoc/tokenjuice)
- [caveman — juliusbrussee/caveman](https://github.com/juliusbrussee/caveman)
- [claude-token-efficient — drona23](https://github.com/drona23/claude-token-efficient)
- [token-optimizer-mcp — ooples](https://github.com/ooples/token-optimizer-mcp)
- [6-Layer Token Savings Stack — doobidoo gist](https://gist.github.com/doobidoo/e5500be6b59e47cadc39e0b7c5cd9871)
- [Claude Code hooks 参考](https://code.claude.com/docs/en/hooks)
- [Chroma context rot 研究](https://research.trychroma.com/context-rot)
- [Morph: 为什么 LLMs 随上下文增长退化](https://www.morphllm.com/context-rot)
- [Anthropic Opus 4.6 Compaction API — InfoQ](https://www.infoq.com/news/2026/03/opus-4-6-context-compaction/)
- [OpenAI compaction 文档](https://developers.openai.com/api/docs/guides/compaction)
- [Google ADK 上下文压缩](https://google.github.io/adk-docs/context/compaction/)
- [LangChain 自主上下文压缩](https://blog.langchain.com/autonomous-context-compression/)
- [sst/opencode 上下文管理](https://deepwiki.com/sst/opencode/2.4-context-management-and-compaction)
- [DEV: 确定性 vs. LLM 评估器 — 2026 权衡研究](https://dev.to/anshd_12/deterministic-vs-llm-evaluators-a-2026-technical-trade-off-study-11h)
- [MadPlay: RTK 80% token 缩减实验](https://madplay.github.io/en/post/rtk-reduce-ai-coding-agent-token-usage)
- [Esteban Estrada: RTK 70% Claude Code 缩减](https://codestz.dev/experiments/rtk-rust-token-killer)

**GCOMPACTION.md 规范部分结束。** 计划批准时，上面所有内容被原样复制到 `docs/designs/GCOMPACTION.md` 作为**搁置的设计产物**。不写代码；不安装钩子；不添加 CHANGELOG 条目。文档存在是为了让未来 sprint 在 Anthropic 发版内置工具 output-replace API 时能快速解锁。
