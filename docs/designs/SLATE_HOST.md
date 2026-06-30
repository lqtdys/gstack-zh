# Slate 集成 — 调研与设计文档

**日期：** 2026-04-02
**分支：** garrytan/slate-agent-support
**状态：** 调研完成，被 host 配置重构所阻塞
**取代：** 无

## Slate 是什么

Slate 是 Random Labs 专有的编码代理 CLI。
安装方式：`npm i -g @randomlabs/slate` 或 `brew install anthropic/tap/slate`。
许可证：专有。编译后的 Bun 二进制文件 85MB（arm64/x64，darwin/linux/windows）。
npm 包：`@randomlabs/slate@1.0.25`（8.8KB 的薄启动器 + 平台相关可选依赖）。

多模型：动态选择 Claude Sonnet/Opus/Haiku 及其他模型。
为"swarm orchestration"及扩展多小时会话而构建。

## Slate 是 OpenCode 的分支

**通过 85MB Mach-O arm64 二进制文件的字符串分析确认：**

- 内部名称：`name: "opencode"`（二进制文件中的字面字符串）
- 所有 `OPENCODE_*` 环境变量与对应的 `SLATE_*` 并存
- 共享 OpenCode 的 tool/skill 架构、LSP 集成、终端管理
- 自有品牌、API 端点（`api.randomlabs.ai`、`agent-worker-prod.randomlabs.workers.dev`）及配置路径

这对集成有影响：OpenCode 的约定大多适用，但 Slate 在此基础上加入了自己的路径和环境变量。

## Skill 发现（从二进制文件确认）

Slate 扫描全部四个目录家族以查找技能。二进制文件中的错误消息确认：

```
"failed .slate directory scan for skills"
"failed .claude directory scan for skills"
"failed .agents directory scan for skills"
"failed .opencode directory scan for skills"
```

**发现路径（按 Slate 文档中的优先级排序）：**

1. `.slate/skills/<name>/SKILL.md` — 项目级，最高优先级
2. `~/.slate/skills/<name>/SKILL.md` — 全局
3. `.opencode/skills/`、`.agents/skills/` — 兼容性回退
4. `.claude/skills/` — Claude Code 兼容性回退（最低优先级）
5. 通过 `slate.json` 的自定义路径

**Glob 模式：** `**/SKILL.md` 和 `{skill,skills}/**/SKILL.md`

**命令：** 相同的目录结构但在 `commands/` 子目录下：
`/.slate/commands/`、`/.claude/commands/`、`/.agents/commands/`、`/.opencode/commands/`

**Skill frontmatter：** YAML，包含 `name` 和 `description` 字段（依 Slate 文档）。
两个字段均无文档长度限制。

## 项目指令

Slate 同时读取 `CLAUDE.md` 和 `AGENTS.md` 作为项目指令。
两个字面字符串均在二进制文件中确认。对现有 gstack 项目无需修改……
CLAUDE.md 可以直接使用。

## 配置

**配置文件：** `slate.json` / `slate.jsonc`（不是 opencode.json）

**配置选项（来自 Slate 文档）：**
- `privacy`（布尔值）— 禁用遥测/日志记录
- 权限：每个工具（`read`、`edit`、`bash`、`grep`、`webfetch`、`websearch`、`*`）对应 `allow`、`ask`、`deny`
- 模型槽位：`models.main`、`models.subagent`、`models.search`、`models.reasoning`
- MCP 服务器：本地或远程，含自定义命令和 headers
- 自定义命令：`/commands` 带模板

安装脚本不应创建 `slate.json`。用户自行配置权限。

## CLI 标志（无头模式）

```
--stream-json / --output-format stream-json  — JSONL 输出，"与 Anthropic Claude Code SDK 兼容"
--dangerously-skip-permissions               — 跳过所有权限检查（CI/自动化）
--input-format stream-json                   — 程序化输入
-q                                           — 非交互模式
-w <dir>                                     — 工作区目录
--output-format text                         — 纯文本输出（默认）
```

**Stream-JSON 格式：** Slate 文档声称"与 Anthropic Claude Code SDK 兼容"。
尚未经验证。鉴于 OpenCode 背景，可能与 Claude Code 的 NDJSON
事件模式（type: "assistant"、type: "tool_result"、type: "result"）匹配。

**需要验证：** 运行 `slate -q "hello" --stream-json` 并使用有效额度，
在构建会话运行器解析器之前捕获实际的 JSONL 事件。

## 环境变量（来自二进制字符串）

### Slate 特有
```
SLATE_API_KEY                              — API key
SLATE_AGENT                                — 代理选择
SLATE_AUTO_SHARE                           — 自动分享设置
SLATE_CLIENT                               — 客户端标识符
SLATE_CONFIG                               — 配置覆盖
SLATE_CONFIG_CONTENT                       — 内联配置
SLATE_CONFIG_DIR                           — 配置目录
SLATE_DANGEROUSLY_SKIP_PERMISSIONS         — 跳过权限
SLATE_DIR                                  — 数据目录覆盖
SLATE_DISABLE_AUTOUPDATE                   — 禁用自动更新
SLATE_DISABLE_CLAUDE_CODE                  — 完全禁用 Claude Code 集成
SLATE_DISABLE_CLAUDE_CODE_PROMPT           — 禁用 Claude Code 提示加载
SLATE_DISABLE_CLAUDE_CODE_SKILLS           — 禁用 .claude/skills/ 加载
SLATE_DISABLE_DEFAULT_PLUGINS              — 禁用默认插件
SLATE_DISABLE_FILETIME_CHECK               — 禁用文件时间检查
SLATE_DISABLE_LSP_DOWNLOAD                 — 禁用 LSP 自动下载
SLATE_DISABLE_MODELS_FETCH                 — 禁用 models 配置获取
SLATE_DISABLE_PROJECT_CONFIG               — 禁用项目级配置
SLATE_DISABLE_PRUNE                        — 禁用会话修剪
SLATE_DISABLE_TERMINAL_TITLE               — 禁用终端标题更新
SLATE_ENABLE_EXA                           — 启用 Exa 搜索
SLATE_ENABLE_EXPERIMENTAL_MODELS           — 启用实验模型
SLATE_EXPERIMENTAL                         — 启用实验功能
SLATE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS — bash 超时覆盖
SLATE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT  — 禁用选择时复制
SLATE_EXPERIMENTAL_DISABLE_FILEWATCHER     — 禁用文件监听
SLATE_EXPERIMENTAL_EXA                     — Exa 搜索（备选标志）
SLATE_EXPERIMENTAL_FILEWATCHER             — 启用文件监听
SLATE_EXPERIMENTAL_ICON_DISCOVERY          — 图标发现
SLATE_EXPERIMENTAL_LSP_TOOL               — LSP 工具
SLATE_EXPERIMENTAL_LSP_TY                 — LSP 类型检查
SLATE_EXPERIMENTAL_MARKDOWN               — markdown 模式
SLATE_EXPERIMENTAL_OUTPUT_TOKEN_MAX       — 输出 token 限制
SLATE_EXPERIMENTAL_OXFMT                  — oxfmt 集成
SLATE_EXPERIMENTAL_PLAN_MODE              — 计划模式
SLATE_FAKE_VCS                            — 测试用假 VCS
SLATE_GIT_BASH_PATH                       — git bash 路径（Windows）
SLATE_MODELS_URL                          — models 配置 URL
SLATE_PERMISSION                          — 权限覆盖
SLATE_SERVER_PASSWORD                     — 服务器认证
SLATE_SERVER_USERNAME                     — 服务器认证
SLATE_TELEMETRY_DISABLED                  — 禁用遥测
SLATE_TEST_HOME                           — 测试 home 目录
SLATE_TOKEN_DIR                           — token 存储目录
```

### OpenCode 遗留（仍可用）
```
OPENCODE_DISABLE_LSP_DOWNLOAD
OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER
OPENCODE_EXPERIMENTAL_FILEWATCHER
OPENCODE_EXPERIMENTAL_ICON_DISCOVERY
OPENCODE_EXPERIMENTAL_LSP_TY
OPENCODE_EXPERIMENTAL_OXFMT
OPENCODE_FAKE_VCS
OPENCODE_GIT_BASH_PATH
OPENCODE_LIBC
OPENCODE_TERMINAL
```

### gstack 集成的关键环境变量

**`SLATE_DISABLE_CLAUDE_CODE_SKILLS`** — 设置后，`.claude/skills/` 加载被禁用。
这使得发布到 `.slate/skills/` 具有实际意义，而不仅仅是优化。
如果没有原生的 `.slate/` 发布，设置此标志后 gstack 技能将消失。

**`SLATE_TEST_HOME`** — 对 E2E 测试有用。可以将 Slate 的 home 目录
重定向到隔离的临时目录，类似于 Codex 测试使用临时 HOME 的方式。

**`SLATE_DANGEROUSLY_SKIP_PERMISSIONS`** — 无头 E2E 测试所必需。

## 模型引用（来自二进制文件）

```
anthropic/claude-sonnet-4.6
anthropic/claude-opus-4
anthropic/claude-haiku-4
anthropic/slate              — Slate 自己的模型路由
openai/gpt-5.3-codex
google/nano-banana
randomlabs/fast-default-alpha
```

## API 端点（来自二进制文件）

```
https://api.randomlabs.ai                          — 主 API
https://api.randomlabs.ai/exaproxy                 — Exa 搜索代理
https://agent-worker-prod.randomlabs.workers.dev   — 生产 worker
https://agent-worker-dev.randomlabs.workers.dev    — 开发 worker
https://dashboard.randomlabs.ai                    — 仪表盘
https://docs.randomlabs.ai                         — 文档
https://randomlabs.ai/config.json                  — 远程配置
```

Brew tap：`anthropic/tap/slate`（值得注意的是在 Anthropic 的 tap 而非 Random Labs 下）

## npm 包结构

```
@randomlabs/slate (8.8 kB，薄启动器)
├── bin/slate           — Node.js 启动器（在 node_modules 中查找平台二进制）
├── bin/slate1          — Bun 启动器（相同逻辑，import.meta.filename）
├── postinstall.mjs     — 验证平台二进制存在，必要时创建符号链接
└── package.json        — 声明各平台的 optionalDependencies

平台包（每个 85MB）：
├── @randomlabs/slate-darwin-arm64
├── @randomlabs/slate-darwin-x64
├── @randomlabs/slate-linux-arm64
├── @randomlabs/slate-linux-x64
├── @randomlabs/slate-linux-x64-musl
├── @randomlabs/slate-linux-arm64-musl
├── @randomlabs/slate-linux-x64-baseline
├── @randomlabs/slate-linux-x64-baseline-musl
├── @randomlabs/slate-darwin-x64-baseline
├── @randomlabs/slate-windows-x64
└── @randomlabs/slate-windows-x64-baseline
```

二进制覆盖：`SLATE_BIN_PATH` 环境变量跳过所有发现过程，直接运行指定的二进制。

## 目前已可用

gstack 技能已通过 `.claude/skills/` 回退路径在 Slate 中可用。
基本功能无需更改。为 Claude Code 安装 gstack 的用户
如果也使用 Slate，会发现他们的技能在两个代理中都可用。

## 一等公民支持所增加的功能

1. **可靠性** — `.slate/skills/` 是 Slate 的最高优先级路径。不受
   `SLATE_DISABLE_CLAUDE_CODE_SKILLS` 影响。
2. **优化的 frontmatter** — 去除 Slate 不使用的 Claude 特有字段
   （allowed-tools、hooks、version）。只保留 `name` 和 `description`。
3. **安装脚本** — 自动检测 `slate` 二进制文件，将技能安装到 `~/.slate/skills/`。
4. **E2E 测试** — 验证技能在由 Slate 直接调用时是否工作。

## 被阻塞：Host 配置重构

Codex 的外部语音审查指出，在 Claude、Codex、Factory 之后添加 Slate 作为第 4 个
host 是"路径别名的 host 爆炸"。当前架构有：

- 在 `type Host = 'claude' | 'codex' | 'factory'` 中硬编码的 host 名称
- `transformFrontmatter()` 中按 host 分支，逻辑几乎重复
- 在 `EXTERNAL_HOST_CONFIG` 中按 host 配置，模式相似
- 安装脚本中按 host 的函数（`create_codex_runtime_root`、`link_codex_skill_dirs`）
- Host 名称在 `bin/gstack-platform-detect`、`bin/gstack-uninstall`、`bin/dev-setup` 中重复

添加 Slate 意味着所有这些模式又要复制一遍。重构为数据驱动的 host
（配置对象代替 if/else 分支）将使 Slate 集成变得简单，并且让未来的 host（任何新的
OpenCode 分支、任何新的 agent）零工作量。

## 计划中缺失的（由 Codex 识别）

- `lib/worktree.ts` 只复制 `.agents/`，不复制 `.slate/` — worktree 中的 E2E 测试
  将没有 Slate 技能
- `bin/gstack-uninstall` 不知道 `.slate/`
- `bin/dev-setup` 没有为开发者开发模式连接 `.slate/`
- `bin/gstack-platform-detect` 无法检测 Slate
- E2E 测试应设置 `SLATE_DISABLE_CLAUDE_CODE_SKILLS=1` 以证明 `.slate/` 路径
  确实有效（而非仅回退到 `.claude/`）

## 会话运行器设计（供后续使用）

当 JSONL 格式被验证后，会话运行器应：

- 生成：`slate -q "<prompt>" --stream-json --dangerously-skip-permissions -w <dir>`
- 解析：Claude Code SDK 兼容的 NDJSON（假定，需验证）
- 技能：安装到测试 fixture 中的 `.slate/skills/`（非 `.claude/skills/`）
- 认证：使用 `SLATE_API_KEY` 或已有的 `~/.slate/` 凭据
- 隔离：使用 `SLATE_TEST_HOME` 进行 home 目录隔离
- 超时：默认 300s（与 Codex 相同）

```typescript
export interface SlateResult {
  output: string;
  toolCalls: string[];
  tokens: number;
  exitCode: number;
  durationMs: number;
  sessionId: string | null;
  rawLines: string[];
  stderr: string;
}
```

## 文档参考

- Slate 文档：https://docs.randomlabs.ai
- 快速入门：https://docs.randomlabs.ai/en/getting-started/quickstart
- 技能：https://docs.randomlabs.ai/en/using-slate/skills
- 配置：https://docs.randomlabs.ai/en/using-slate/configuration
- 快捷键：https://docs.randomlabs.ai/en/using-slate/hotkey_reference
