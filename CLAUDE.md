# gstack 开发指南

## 命令

```bash
bun install          # 安装依赖
bun test             # 运行免费测试（浏览 + 快照 + skill 验证）
bun run test:evals   # 运行付费评估：LLM 评审 + E2E（基于 diff，约 $4/次）
bun run test:evals:all  # 运行所有付费评估，忽略 diff
bun run test:gate    # 仅运行 gate 层测试（CI 默认，阻断合并）
bun run test:periodic  # 仅运行 periodic 层测试（每周 cron / 手动）
bun run test:e2e     # 仅运行 E2E 测试（基于 diff，约 $3.85/次）
bun run test:e2e:all # 运行所有 E2E 测试，忽略 diff
bun run eval:select  # 显示基于当前 diff 将运行哪些测试
bun run dev <cmd>    # 开发模式运行 CLI，例如 bun run dev goto https://example.com
bun run build        # 生成文档 + 编译二进制文件
bun run gen:skill-docs  # 从模板重新生成 SKILL.md 文件
bun run skill:check  # 所有 skill 的健康仪表盘
bun run dev:skill    # 监听模式：变更时自动重新生成 + 验证
bun run eval:list    # 列出 ~/.gstack-dev/evals/ 中的所有评估运行
bun run eval:compare # 比较两次评估运行（自动选择最新的）
bun run eval:summary # 跨所有评估运行聚合统计
bun run slop          # 完整 slop 扫描报告（所有文件）
bun run slop:diff     # 仅限本分支变更的文件中的 slop 发现
```

`test:evals` 需要 `ANTHROPIC_API_KEY`。Codex E2E 测试（`test/codex-e2e.test.ts`）使用 `~/.codex/` 配置中的 Codex 自身认证——无需 `OPENAI_API_KEY` 环境变量。

**Conductor 工作区中的环境键。** `GSTACK_*` 环境垫片（v1.39.2.0+，`lib/conductor-env-shim.ts`）将 `GSTACK_ANTHROPIC_API_KEY` / `GSTACK_OPENAI_API_KEY` 提升为 gstack 的 TS 二进制文件中的规范名称。通过 gstack 入口点运行的测试自动继承此提升。不要将键值回显到 stdout、日志或 shell 历史。历史性的"永远不要将 `env:` 传给 `runAgentSdkTest`"规则已废弃：失败原因是部分环境替换（SDK 的 `Options.env` 会替换子进程的整个环境，所以没有该键的对象会破坏认证）。运行器现在始终传递一个完整的 hermetic 环境，并将每个测试的 `env:` 合并到最后，因此每个测试的覆盖是安全的；对 `process.env.ANTHROPIC_API_KEY` 的隐式变更仍然有效（环境构建器在调用时读取 process.env）。

**Hermetic 本地 E2E（默认）。** 每个 E2E 运行器（`claude -p`、PTY、Agent SDK、`codex`、`gemini`）通过 `test/helpers/hermetic-env.ts` 生成子进程：允许列表擦洗的环境（操作员 `CONDUCTOR_*`、`CLAUDE_*`、`GSTACK_*`、`MCP_*`、`GBRAIN_*` 等变量以及 `GH_TOKEN` 等凭证永远不会到达子进程）、一个全新的种子化 `CLAUDE_CONFIG_DIR`（无操作员的 `~/.claude` CLAUDE.md / MCP 服务器 / skill）、一个临时的 `GSTACK_HOME`，以及 `--strict-mcp-config`。本地评估信号与 CI 一致。使用 `EVALS_HERMETIC=0` 调试真实操作员状态（恢复旧环境并移除 strict-MCP 标志）。每个测试的 `env:` 覆盖合并到最后，因此故意污染（`CONDUCTOR_WORKSPACE_PATH`、每个测试的 `GSTACK_HOME`）继续工作。连线由 `test/hermetic-wiring.test.ts`（静态绊网）和 `test/skill-e2e-hermetic-canary.test.ts` 中的两个 gate 层金丝钉固定。

E2E 测试实时流式传输进度（通过 `--output-format stream-json --verbose` 逐工具）。结果持久化到 `~/.gstack-dev/evals/`，并与上一次运行自动比较。

**基于 diff 的测试选择：** `test:evals` 和 `test:e2e` 基于对基线的 `git diff` 自动选择测试。每个测试在 `test/helpers/touchfiles.ts` 中声明其文件依赖。对全局 touchfiles（session-runner、eval-store、touchfiles.ts 本身）的变更会触发所有测试。使用 `EVALS_ALL=1` 或 `:all` 脚本变体强制运行所有测试。运行 `eval:select` 预览将运行哪些测试。

**双层系统：** 测试在 `E2E_TIERS`（在 `test/helpers/touchfiles.ts` 中）中被分类为 `gate` 或 `periodic`。CI 仅运行 gate 测试（`EVALS_TIER=gate`）；periodic 测试通过 cron 每周运行或手动运行。使用 `EVALS_TIER=gate` 或 `EVALS_TIER=periodic` 进行筛选。添加新的 E2E 测试时，对其进行分类：
1. 安全护栏或确定性功能测试？ -> `gate`
2. 质量基准、Opus 模型测试或非确定性？ -> `periodic`
3. 需要外部服务（Codex、Gemini）？ -> `periodic`

## 测试

```bash
bun test             # 每次提交前运行 — 免费，<2秒
bun run test:evals   # 发布前运行 — 付费，基于 diff（~$4/次）
```

`bun test` 运行 skill 验证、gen-skill-docs 质量检查和浏览集成测试。`bun run test:evals` 通过 `claude -p` 运行 LLM 评审质量评估和 E2E 测试。创建 PR 之前两者都必须通过。

## 项目结构

```
gstack/
├── browse/          # 无头浏览器 CLI（Playwright）
│   ├── src/         # CLI + 服务器 + 命令
│   │   ├── commands.ts  # 命令注册表（唯一来源）
│   │   └── snapshot.ts  # SNAPSHOT_FLAGS 元数据数组
│   ├── test/        # 集成测试 + 装置
│   └── dist/        # 编译后的二进制文件
├── hosts/           # 类型化主机配置（每个 AI 代理一个）
│   ├── claude.ts    # 主要主机配置
│   ├── codex.ts, factory.ts, kiro.ts  # 现有主机
│   ├── opencode.ts, slate.ts, cursor.ts, openclaw.ts  # IDE 主机
│   ├── hermes.ts, gbrain.ts  # Agent 运行时主机
│   └── index.ts     # 注册表：导出所有，派生 Host 类型
├── scripts/         # 构建 + DX 工具
│   ├── gen-skill-docs.ts  # 模板 → SKILL.md 生成器（配置驱动）
│   ├── host-config.ts     # HostConfig 接口 + 验证器
│   ├── host-config-export.ts  # 设置脚本的 shell 桥接
│   ├── host-adapters/     # 主机特定适配器（OpenClaw 工具映射）
│   ├── resolvers/   # 模板解析器模块（preamble、design、review、gbrain 等）
│   ├── skill-check.ts     # 健康仪表盘
│   └── dev-skill.ts       # 监听模式
├── test/            # Skill 验证 + 评估测试
│   ├── helpers/     # skill-parser.ts, session-runner.ts, llm-judge.ts, eval-store.ts
│   ├── fixtures/    # 基准真相 JSON、植入缺陷装置、评估基线
│   ├── skill-validation.test.ts  # 第1层：静态验证（免费，<1秒）
│   ├── gen-skill-docs.test.ts    # 第1层：生成器质量（免费，<1秒）
│   ├── skill-llm-eval.test.ts   # 第3层：LLM 即评审（~$0.15/次）
│   └── skill-e2e-*.test.ts       # 第2层：通过 claude -p 的 E2E（~$3.85/次，按类别分割）
├── qa-only/         # /qa-only skill（仅报告 QA，无修复）
├── plan-design-review/  # /plan-design-review skill（仅报告设计审计）
├── design-review/    # /design-review skill（设计审计 + 修复循环）
├── ship/            # 发布工作流 skill
├── review/          # PR 审查 skill
├── plan-ceo-review/ # /plan-ceo-review skill
├── plan-eng-review/ # /plan-eng-review skill
├── autoplan/        # /autoplan skill（自动审查流水线：CEO → 设计 → 工程）
├── benchmark/       # /benchmark skill（性能回归检测）
├── canary/          # /canary skill（部署后监控循环）
├── codex/           # /codex skill（通过 OpenAI Codex CLI 获取多 AI 第二意见）
├── land-and-deploy/ # /land-and-deploy skill（合并 → 部署 → 验证）
├── office-hours/    # /office-hours skill（YC Office Hours — 启动诊断 + 建设者头脑风暴）
├── investigate/     # /investigate skill（系统性根因调试）
├── spec/            # /spec skill（五阶段规范 → GitHub 问题，可选 agent 生成，/ship 自动关闭）
├── retro/           # 回顾 skill（包括 /retro 全局跨项目模式）
├── bin/             # CLI 工具（gstack-repo-mode、gstack-slug、gstack-config 等）
├── document-release/ # /document-release skill（发布后文档更新 + Diataxis 覆盖图）
├── document-generate/ # /document-generate skill（Diataxis 文档生成器：教程/操作指南/参考/说明）
├── cso/             # /cso skill（OWASP Top 10 + STRIDE 安全审计）
├── design-consultation/ # /design-consultation skill（从零构建设计系统）
├── design-shotgun/  # /design-shotgun skill（视觉设计探索）
├── open-gstack-browser/  # /open-gstack-browser skill（启动 GStack Browser）
├── connect-chrome/  # 符号链接 → open-gstack-browser（向后兼容）
├── design/          # 设计二进制 CLI（GPT Image API）
│   ├── src/         # CLI + 命令（generate、variants、compare、serve 等）
│   ├── test/        # 集成测试
│   └── dist/        # 编译后的二进制文件
├── extension/       # Chrome 扩展（侧边栏 + 活动源 + CSS 检查器）
├── lib/             # 共享库（worktree.ts）
├── docs/designs/    # 设计文档
├── setup-deploy/    # /setup-deploy skill（一次性部署配置）
├── .github/         # CI 工作流 + Docker 镜像
│   ├── workflows/   # evals.yml（Ubicloud 上的 E2E）、skill-docs.yml、actionlint.yml
│   └── docker/      # Dockerfile.ci（预装工具链 + Playwright/Chromium）
├── contrib/         # 仅限贡献者的工具（永不安装给最终用户）
│   └── add-host/    # /gstack-contrib-add-host skill
├── setup            # 一次性设置：构建二进制文件 + 符号链接 skill
├── SKILL.md         # 从 SKILL.md.tmpl 生成（不要直接编辑）
├── SKILL.md.tmpl    # 模板：编辑这个，运行 gen:skill-docs
├── ETHOS.md         # 建设者哲学（Boil the Ocean、Search Before Building）
└── package.json     # browse 的构建脚本
```

## SKILL.md 工作流

SKILL.md 文件是**从 `.tmpl` 模板生成**的。更新文档的方法：

1. 编辑 `.tmpl` 文件（例如 `SKILL.md.tmpl` 或 `browse/SKILL.md.tmpl`）
2. 运行 `bun run gen:skill-docs`（或 `bun run build`，它会自动执行）
3. 同时提交 `.tmpl` 和生成的 `.md` 文件

要添加新的 browse 命令：将其添加到 `browse/src/commands.ts` 并重新构建。
要添加快照标志：将其添加到 `browse/src/snapshot.ts` 中的 `SNAPSHOT_FLAGS` 并重新构建。

**Token 上限：** 生成的 SKILL.md 文件在超过 160KB（约 40K token）时触发警告。这是一个"监控功能膨胀"的护栏，而非硬性限制。现代旗舰模型有 200K-1M 的上下文窗口，所以 40K 仅占窗口的 4-20%，提示缓存使较大 skill 的边际成本很小。上限的存在是为了捕获失控的 preamble/解析器增长，而不是对精心调优的大 skill（`ship`、`plan-ceo-review`、`office-hours` 合法打包了 25-35K token 的行为）进行强制压缩。如果你突破 40K，正确的修复通常是：(1) 查看是什么增长了，(2) 如果某个解析器在单个 PR 中增加了 10K+，质疑它应该是内联的还是作为参考文档，(3) 仅作为最后手段压缩精心调优的散文 — 对覆盖审计、审查军团或声音指令的削减有真实的质量代价。

**SKILL.md 文件上的合并冲突：** 永远不要通过接受任一方来解决生成的 SKILL.md 文件上的冲突。相反：(1) 在 `.tmpl` 模板和 `scripts/gen-skill-docs.ts`（来源）上解决冲突，(2) 运行 `bun run gen:skill-docs` 重新生成所有 SKILL.md 文件，(3) 暂存重新生成的文件。接受一方的生成输出会悄悄丢弃另一方的模板变更。

## 平台无关设计

Skill 绝不应硬编码特定于框架的命令、文件模式或目录结构。相反：

1. **读取 CLAUDE.md** 获取项目特定配置（测试命令、评估命令等）
2. **如果缺失，AskUserQuestion** — 让用户告诉你或让 gstack 搜索仓库
3. **将答案持久化到 CLAUDE.md** 以便我们永远不必再次询问

这适用于测试命令、评估命令、部署命令以及任何其他项目特定行为。项目拥有其配置；gstack 读取它。

## 编写 SKILL 模板

SKILL.md.tmpl 文件是**Claude 读取的提示模板**，不是 bash 脚本。每个 bash 代码块在独立的 shell 中运行——变量不在块之间持久化。

规则：
- **使用自然语言表达逻辑和状态。** 不要使用 shell 变量在代码块之间传递状态。相反，告诉 Claude 要记住什么，并在散文中引用它（例如，"在 Step 0 中检测到的基线分支"）。
- **不要硬编码分支名称。** 通过 `gh pr view` 或 `gh repo view` 动态检测 `main`/`master` 等。对面向 PR 的 skill 使用 `{{BASE_BRANCH_DETECT}}`。在散文中使用"基线分支"，在代码块占位符中使用 `<base>`。
- **保持 bash 块自包含。** 每个代码块应独立工作。如果某个块需要来自前一步的上下文，在上方的散文中重述它。
- **用英语表达条件逻辑。** 不要在 bash 中使用嵌套的 `if/elif/else`。写编号的决策步骤："1. 如果是 X，执行 Y。2. 否则，执行 Z。"

## 写作风格（V1）

默认情况下，每个 tier≥2 技能的输出遵循 `scripts/resolvers/preamble.ts` 中的"Writing Style"部分：术语在首次使用时注释（精选列表在 `scripts/jargon-list.json` 中，在 gen-skill-docs 时烘焙），问题以结果术语（"如果你的用户遇到什么会出问题……"）而非实施术语提出，短句子，决策结束时附带用户影响。想要更紧凑 V0 散文的高级用户设置 `gstack-config set explain_level terse`（二元开关，无中间模式）。完整设计原理见 `docs/designs/PLAN_TUNING_VMD`。最初尝试与写作风格并行的审查节奏大修被提取到 V1.1——见 `docs/designs/PACING_UPDATES_VMD`。

## 浏览器交互

当需要与浏览器交互（QA、dogfooding、cookie 设置）时，使用 `/browse` skill 或通过 `$B <command>` 直接运行 browse 二进制文件。**绝不要使用** `mcp__claude-in-chrome__*` 工具——它们速度慢、不可靠，不是本项目使用的。

**侧边栏架构：** 在修改 `sidepanel.js`、`background.js`、`content.js`、`terminal-agent.ts` 或与侧边栏相关的服务器端点之前，阅读 `docs/designs/SIDEBAR_MESSAGE_FLOW.md`。侧边栏有一个主要表面——**Terminal** 窗格（交互式 `claude` PTY）——Activity / Refs / Inspector 作为页脚 `debug` 切换后面的调试覆盖。聊天队列路径在 PTY 被证明后已被移除；`sidebar-agent.ts` 和 `/sidebar-command` / `/sidebar-chat` / `/sidebar-agent/event` 端点已不存在。该文档涵盖了 WS 认证流程、双令牌模型和威胁模型边界——这里的静默故障通常追溯到不理解跨组件流程。

**嵌入者 terminal-agent 所有权**（v1.42.1.0+，基于身份的终止 v1.44.0.0+）。`browse/src/server.ts` 中的 `buildFetchHandler` 接受 `ServerConfig.ownsTerminalAgent?: boolean`（默认 `true`）。当 `true` 时，工厂关闭运行完整的拆卸：通过 `browse/src/terminal-agent-control.ts` 中的 `killAgentByRecord(readAgentRecord(stateDir))` 进行基于身份的终止，以及在 `<stateDir>/terminal-port`、`<stateDir>/terminal-internal-token` 和 `<stateDir>/terminal-agent-pid`（v1.44 中引入的每启动 agent 记录）上进行 `safeUnlinkQuiet`。预先启动自己的 PTY 服务器的嵌入者（例如 gbrowser phoenix 覆盖）必须传递 `false` 以便它们的发现文件在 gstack 拆卸周期中存活。该标志是 `ServerConfig` 中第三个调用方拥有的拆卸门（与 `xvfb?` 和 `proxyBridge?` 一起）；极性反转（显式 bool vs 存在性）并在字段的 JSDoc 中记录。CLI `start()` 始终显式传递 `true`——如果重构删除它，`browse/test/server-embedder-terminal-port.test.ts` 中的静态 grep 测试会在 CI 中失败。v1.44 之前使用 `pkill -f terminal-agent\\.ts`（正则匹配），它会终止同一主机上的兄弟 gstack 会话；如果任何源文件重新引入 `pkill ... terminal-agent` 或 `spawnSync('pkill', ...)`，新的 `browse/test/terminal-agent-pid-identity.test.ts` 静态 grep 绊网会在 CI 中失败。

**WebSocket 认证使用 Sec-WebSocket-Protocol，而非 cookie。** 浏览器无法在 WebSocket 升级上设置 `Authorization`，但可以通过 `new WebSocket(url, [token])` 设置 `Sec-WebSocket-Protocol`。agent 读取它，针对 `validTokens` 进行验证，并且**必须**在升级响应中回显该协议——没有回显，Chromium 会立即关闭连接。`Set-Cookie: gstack_pty=...` 保留为非浏览器调用方的备用方案（跨端口的 `SameSite=Strict` cookie 路径从 chrome-extension 来源无法存活）。

**跨窗格 PTY 注入。** 工具栏的 Cleanup 按钮和 Inspector 的"发送到代码"动作都通过 `window.gstackInjectToTerminal(text)` 将文本管道到活动的 claude PTY，由 `sidepanel-terminal.js` 暴露。没有 `/sidebar-command` POST——活动的 REPL 现在是侧边栏中唯一的执行表面。

**`/health` 绝不能暴露任何 shell-grant 令牌。** 它已经在有头模式下将 `AUTH_TOKEN` 泄露给本地调用者（一个 v1.1+ TODO）。不要通过在那里添加 PTY 会话令牌使情况更糟。PTY 认证仅通过 `POST /pty-session` 流动。

**传输层安全**（v1.6.0.0+）。当 `pair-agent` 启动 ngrok 隧道时，守护进程绑定两个 HTTP 监听器：本地监听器（127.0.0.1，完整命令表面，永不转发）和隧道监听器（锁定允许列表：`/connect`，`/command` + 令牌作用域 + 26 命令浏览器驱动允许列表，`/sidebar-chat`）。ngrok 仅转发隧道端口。隧道上的根令牌返回 403。SSE 端点使用通过 `POST /sse-session` 铸造的 30 分钟 HttpOnly `gstack_sse` cookie（对 `/command` 永远无效）。隧道表面拒绝通过 `tunnel-denial-log.ts` 记录到 `~/.gstack/security/attempts.jsonl`。在编辑 `server.ts`、`sse-session-cookie.ts` 或 `tunnel-denial-log.ts` 之前，阅读 [ARCHITECTURE.md](ARCHITECTURE.md#dual-listener-tunnel-architecture-v1600)——模块边界（从 `token-registry.ts` 到 `sse-session-cookie.ts` 无导入）对作用域隔离至关重要。

**服务器出口处的 Unicode 清理**（v1.38.0.0+）。每个发送页面内容派生字符串的服务器出口必须针对对象负载使用 `JSON.stringify(payload, sanitizeReplacer)`，或针对文本体使用 `sanitizeLoneSurrogates(body)`。来自 CDP 页面内容的孤立的 UTF-16 代理对会以 `\uD800` 风格转义到达 Anthropic API 并触发 400。今天在四个出口点接线：`handleCommandInternal`（HTTP + 批次，通过围绕 `handleCommandInternalImpl` 的清理包装器）和两个 SSE 生产者（`/activity/stream`、`/inspector/events`）。字符串化后的正则表达式是空操作——`JSON.stringify` 在正则表达式可能匹配之前已经转义了代理，所以替换器必须在编码管道内部运行。在 `server.ts` 中添加新的 SSE/WebSocket 写入器或 HTTP 响应之前，阅读 [ARCHITECTURE.md](ARCHITECTURE.md#unicode-sanitization-at-server-egress-v13800)。`browse/test/server-sanitize-surrogates.test.ts` 通过不变测试固定接线，因此绕过会在 CI 中失败。

**SSE 端点助手**（v1.51.0.0+）。`server.ts` 中的新 SSE 端点必须通过 `browse/src/sse-helpers.ts` 中的 `createSseEndpoint(req, config)` 路由。该助手拥有清理契约（abort + enqueue-throw + heartbeat-throw，全部幂等）并在每个 JSON.stringify 上烘焙 `sanitizeLoneSurrogates`，因此新订阅者不会意外回退任一不变量。当 TCP 连接在未触发 `req.signal.abort` 的情况下死亡时（Chromium MV3 service-worker 悬挂，中间代理半关闭），内联 `ReadableStream` 接线会泄露订阅者。`/activity/stream`、`/inspector/events` 和 `/memory`（SSE 合格）都通过它路由。`browse/test/sse-helpers.test.ts` 固定清理契约。

**CDP 会话生命周期**（v1.51.0.0+）。在 `browse/src/cdp-bridge.ts` 外部直接调用 `page.context().newCDPSession(page)` 会通过 `browse/test/cdp-session-cleanup.test.ts` 中的静态 grep 绊网使 CI 失败。对一次性 CDP 工作使用 `withCdpSession(page, async (s) => {...})`（try/finally 分离），或对绑定到页面生命周期的缓存会话使用 `getOrCreateCdpSession(page, cache)`（通过 `Map<page, session>` 关闭分离）。三个站点已迁移：cdp-bridge 帧事件、写命令存档捕获、cdp-inspector。这些助手防止了成功路径分离发生但错误路径分离被遗漏的每会话泄漏类。

**设置符号链接加固**（v1.38.0.0+）。`setup` 中的每个链接站点必须通过在 `IS_WINDOWS` 检测附近的 `_link_or_copy SRC DST` 助手路由。在没有开发者模式的 Windows 上，普通的 `ln -snf` 会产生冻结的文件副本，在 `git pull` 时不刷新——每个主机适配器上都会出现静默陈旧。助手在 Unix 上保留 `ln -snf` 并在 Windows 上切换到 `cp -R` / `cp -f`。`test/setup-windows-fallback.test.ts` 强制实施静态不变量：助手主体外部的单个原始 `ln` 调用会使 CI 失败。Windows 用户会从 `_print_windows_copy_note_once` 获得一行注释，提醒他们在每次 `git pull` 后重新运行 `./setup`。

**侧边栏安全堆栈**（分层防御对抗提示注入）：

| 层 | 模块 | 存在于 |
|----|------|--------|
| L1-L3 | `content-security.ts` | 服务器和 agent — 数据标记、隐藏元素剥离、ARIA 正则、URL 阻止列表、信封包装 |
| L4 | `security-classifier.ts`（TestSavantAI ONNX） | **仅 sidebar-agent** |
| L4b | `security-classifier.ts`（Claude Haiku 转录） | **仅 sidebar-agent** |
| L5 | `security.ts`（金丝雀） | 两者 — 注入已编译，检查 agent |
| L6 | `security.ts`（combineVerdict 集成） | 两者 |

**关键约束：** `security-classifier.ts` **不能**从编译的 browse 二进制导入。`@huggingface/transformers` v4 需要 `onnxruntime-node`，它无法从 Bun 编译的临时提取目录 `dlopen`。只有 `security.ts`（纯字符串操作——金丝雀、判决组合器、攻击日志、状态）对 `server.ts` 是安全的。完整架构决策见 `~/.gstack/projects/garrytan-gstack/ceo-plans/2026-04-19-prompt-injection-guard.md` §"Pre-Impl Gate 1 Outcome"。

**阈值**（在 `security.ts` 中）：
- `BLOCK: 0.85` — 如果跨确认将导致 BLOCK 的单一层分数
- `WARN: 0.75` — 跨确认阈值。当 L4 和 L4b 都 >= 0.75 → BLOCK
- `LOG_ONLY: 0.40` — 门控转录分类器（当所有层 < 0.40 时跳过 Haiku）
- `SOLO_CONTENT_BLOCK: 0.92` — 无标签内容分类器（testsavant、deberita）的单一层阈值。故意高于 `BLOCK`，因为这些层无法区分"这是注入"和"这看起来像是针对用户的钓鱼"。转录分类器保留单独的标签门控路径在 `BLOCK`（0.85）。

**集成规则：** 仅当 ML 内容分类器和转录分类器都报告 >= WARN 时才 BLOCK。单层高置信度降级为 WARN——这是 Stack Overflow 编写指令的 FP 缓解。金丝雀泄漏始终 BLOCK（确定性）。

**环境旋钮：**
- `GSTACK_SECURITY_OFF=1` — 紧急终止开关。即使已预热，分类器也保持关闭。金丝雀仍然注入；仅跳过 ML 扫描。
- `GSTACK_SECURITY_ENSEMBLE=deberta` — 选择加入 DeBERTa-v3 集成。添加 ProtectAI DeBERTa-v3-base-injection-onnx 作为用于跨模型一致的 L4c 分类器。首次运行下载 721MB。启用集成后，BLOCK 需要 2/3 ML 分类器在 >= WARN 时一致（testsavant、deberita、转录）。无集成（默认），BLOCK 需要 testsavant + 转录在 >= WARN。
- 分类器模型缓存：`~/.gstack/models/testsavant-small/`（112MB，仅首次运行）加上 `~/.gstack/models/deberta-v3-injection/`（721MB，仅在启用集成时）
- 攻击日志：`~/.gstack/security/attempts.jsonl`（加盐 sha256 + 仅域名，在 10MB 时轮换，5 代）
- 每设备盐：`~/.gstack/security/device-salt`（0600）
- 会话状态：`~/.gstack/security/session-state.json`（跨进程，原子性）

## 开发符号链接意识

开发 gstack 时，`.claude/skills/gstack` 可能是返回此工作目录的符号链接（gitignored）。这意味着 skill 变更**立即生效**——非常适合快速迭代，在大重构期间很危险，因为半写的 skill 可能破坏同时使用 gstack 的其他 Claude Code 会话。

**每次会话检查一次：** 运行 `ls -la .claude/skills/gstack` 查看它是符号链接还是真实副本。如果它是到你工作目录的符号链接，请注意：
- 模板变更 + `bun run gen:skill-docs` 会立即影响所有 gstack 调用
- 对 SKILL.md.tmpl 文件的破坏性变更可能会破坏并发的 gstack 会话
- 在大重构期间，移除符号链接（`rm .claude/skills/gstack`）以便使用 `~/.claude/skills/gstack/` 处的全局安装

## 编辑脱敏（PII / 秘密 / 法律内容）

共享脱敏引擎在内容到达外部汇点（codex 调度、GitHub issue/PR 正文、推送的提交）之前捕获凭证、PII 和/或法律/有害内容。它是一个**护栏，而非气密的执行**——`git push --no-verify`、直接 `gh issue create` 和 `GSTACK_REDACT_PREPUSH=skip` 都会绕过它。它捕获事故和疏忽（99% 的情况）。不要声称它阻止了顽固的泄露者（CHANGELOG 中那样做的人会败给敌对截图者）。

- **引擎 + 分类法：** `lib/redact-patterns.ts`（唯一来源——3 层；HIGH = 真正秘密的凭证，阻止，MEDIUM = PII/法律/内部 + 高 FP 凭证形状，通过 AskUserQuestion 确认，LOW = 仅供参考）和 `lib/redact-engine.ts`（纯 `scan()` + `applyRedactions()`）。校准很重要：狼来了的闸门会被忽略，所以上下文可变形状（Stripe `pk_live_`、Google `AIza`、JWT、env `*_KEY=`）位于 MEDIUM。
- **CLI：** `bin/gstack-redact`（退出 0 干净 / 2 MEDIUM / 3 HIGH；`--json`、`--auto-redact`、`--repo-visibility`、`--from-file`）。`bin/gstack-redact-prepush` 是选择加入的 git 钩子。
- **Skill 文档从** `scripts/resolvers/redact-doc.ts`（`{{REDACT_TAXONOMY_TABLE}}`、`{{REDACT_INVOCATION_BLOCK:<sink>}}`）生成，因此 /spec、/cso、/ship、/document-release、/document-generate 永远不会偏离引擎。
- **在汇点扫描：** 始终扫描将要发送的确切字节——写入临时文件，扫描该文件，将**同一文件**传递给 `gh`/`git`。永远不要扫描字符串然后重新渲染（这会重新打开扫描与发送的间隙）。
- **可见性（无层级提升）：** 每次运行解析一次，顺序 = 本地配置（`gstack-config get redact_repo_visibility`，~/.gstack 所以永不提交）→ gh → glab → unknown(=public-strict)。公共仓库获得**更严格**的每个发现确认（无批量确认，无静默继续）；MEDIUM 永远不会自动提升为 HIGH。
- **工具属性围栏：** 将 Codex/Greptile/eval 输出包装在 ` ```codex-review ` / ` ```greptile ` 围栏中，以便这些工具引用的示例凭证 WARN 降级而非阻止。围栏内的实时格式凭证仍会阻止。
- **配置键：** `redact_repo_visibility`（public|private|未知，gh/glab 无法读取的仓库的本地覆盖），`redact_prepush_hook`（true|false）。故意没有禁用 HIGH 阻止的键。
- **审计：** /spec 语义传递将无内容记录（类别 + 正文 sha256，无规范文本）附加到 `~/.gstack/security/semantic-reviews.jsonl`（0600）。

## 提交风格

**始终 bisect 提交。** 每个提交应为单一逻辑变更。当你进行了多次变更（例如，重命名 + 重写 + 新测试），在推送之前将它们拆分为单独的提交。每个提交应独立可理解和可还原。

良好的 bisection 示例：
- 重命名/移动与行为变更分离
- 测试基础设施（touchfiles、助手）与测试实现分离
- 模板变更与生成的文件重新生成分离
- 机械重构与新功能分离

当用户说"bisect 提交"或"bisect 并推送"时，将已暂存/未暂存的变更拆分为逻辑提交并推送。

## Slop 扫描：AI 代码质量，而非 AI 代码隐藏

我们使用 [slop-scan](https://github.com/benvinegar/slop-scan) 来捕获 AI 生成的代码确实比人类写的差的模式。我们**不是**试图伪装成人类代码。我们是 AI 编码的，并以此为荣。目标是代码质量。

```bash
npx slop-scan scan .          # 人类可读的报告
npx slop-scan scan . --json   # 机器可读，用于 diff
```

配置：仓库根部的 `slop-scan.config.json`（当前排除 `**/vendor/**`）。

### 需要修复的（真正的质量改进）

- **文件操作周围的空 catch** — 使用 `safeUnlink()`（忽略 ENOENT，重新抛出 EPERM/EIO）。清理中被吞下的 EPERM 意味着静默数据丢失。
- **进程终止周围的空 catch** — 使用 `safeKill()`（忽略 ESRCH，重新抛出 EPERM）。被吞下的 EPERM 意味着你认为你终止了某物但实际上没有。
- **冗余的 `return await`** — 当没有封闭的 try 块时删除。节省一个微任务，表明意图。
- **类型化异常捕获** — `catch (err) { if (!(err instanceof TypeError)) throw err }` 在 try 块进行 URL 解析或 DOM 工作时确实比 `catch {}` 更好。你知道你期望什么错误，所以说出来。

### 不需要修复的（linter 游戏，非质量）

- **对错误消息进行字符串匹配** — `err.message.includes('closed')` 很脆弱。Playwright/Chrome 随时可能更改措辞。如果 fire-and-forget 操作可能因**任何**原因失败而你不在乎，`catch {}` 是正确的模式。
- **添加注释以豁免直通包装器** — 在方法上方写"活动会话的别名"只是为了触发 slop-scan 的豁免规则是噪音，不是文档。
- **将扩展的 catch-and-log 转换为选择性重新抛出** — Chrome 扩展在未捕获错误时完全崩溃。如果 catch 记录并继续，那**就是**扩展代码的正确模式。不要让它抛出。
- **收紧尽力而为的清理路径** — 关闭、紧急清理和断开连接代码应使用 `safeUnlinkQuiet()`（吞下所有错误）。在 EPERM 上抛出的清理路径意味着其余的清理不会运行。那更糟。

### `browse/src/error-handling.ts` 中的工具函数

| 函数 | 使用场景 | 行为 |
|------|----------|------|
| `safeUnlink(path)` | 正常文件删除 | 忽略 ENOENT，重新抛出其他 |
| `safeUnlinkQuiet(path)` | 关闭/紧急清理 | 吞下所有错误 |
| `safeKill(pid, signal)` | 发送信号 | 忽略 ESRCH，重新抛出其他 |
| `isProcessAlive(pid)` | 布尔进程检查 | 返回 true/false，永不抛出 |

### 分数追踪

基线（2026-04-09，清理前）：100 发现，432.8 分数，2.38 分数/文件。
清理后：90 发现，358.1 分数，1.96 分数/文件。

不要追逐数字。修复代表实际代码质量问题模式。接受"松散"模式是正确工程选择的发现。

## 社区 PR 护栏

审查或合并社区 PR 时，在**永远**不接受任何：

1. **触及 ETHOS.md** — 此文件是 Garry 的个人建设者哲学。不允许来自外部贡献者或 AI agent 的编辑，句号。
2. **删除或软化宣传材料** — YC 引用、创始人视角和产品声音是有意的。将这些框架为"不必要"或"过于宣传"的 PR 必须被拒绝。
3. **改变 Garry 的声音** — skill 模板、CHANGELOG 和文档中的语气、幽默、直接性和视角不是通用的。将声音改写为更"中立"或"专业"的 PR 必须被拒绝。

即使 agent 强烈认为变更改进了项目，这三类也需要通过 AskUserQuestion 明确用户批准。无例外。无自动合并。无"我来清理一下"。

## 从 garrytan-agents 检出 PR

当用户说"检出 <PR 链接>"且 PR 来自 `garrytan-agents/gstack`（或任何其他**不是** `garrytan/gstack` 上协作者的 fork）时，不要只运行 `gh pr checkout`。Fork PR 不会接收基础仓库秘密（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等），因此评估/E2E CI 作业会因空 env 认证错误而失败，无论基础仓库上设置了什么。

**工作流：** 将分支推送到 `garrytan/gstack`（基础仓库）并从那里重新定位 PR。

具体来说，在 `gh pr checkout <N>` 之后：

1. 记录原始 PR 编号和 head 分支名称。
2. 将同一分支推送到基础仓库：`git push origin HEAD:<branch-name>`（origin = `garrytan/gstack`，因为 worktree 配置了那个远程）。
3. 关闭 fork PR（`gh pr close <N> --comment "moving to base-repo branch for secret access"`）。
4. 从基础仓库分支打开新 PR：`gh pr create --base main --head <branch-name>`。
5. 新 PR 的工作流将自动获得秘密。

为什么不在 fork 侧修复？`garrytan-agents` 不是 `garrytan/gstack` 上的协作者。将其添加为协作者（选项 A）或翻转仓库范围的"将秘密发送给 fork PR"开关（选项 B）会让秘密到达任何人的 fork PR——爆炸半径比移动这一个分支更大。选项 C（本节）使秘密分发范围保持紧密。

如果用户要求你跳过移动（例如，"就让它作为 fork PR"），尊重它——评估 CI 会因空 env 认证而失败，但 check-freshness、workflow-lint 和 windows-tests 仍会在 fork PR 上通过。

## CHANGELOG + VERSION 风格

**版本不变量（workspace-aware 发布）。** VERSION 是单调有序的发布标识符，不是严格的 semver 承诺。凹凸级别（major/minor/patch/micro）在发布时表达意图。在同一凹凸级别内跳过明确的版本是明确允许的——如果分支 A 声称 v1.7.0.0 为 MINOR 且分支 B 也是 MINOR，B 降落在 v1.8.0.0（仍然是相对于 main 的 MINOR）。下游消费者绝不能将"MINOR = 仅功能，PATCH = 仅修复"依赖为严格契约。这就是为什么 `bin/gstack-next-version` 在选定的凹凸级别内前进，而不是在碰撞发生时重新选择级别。

**规模感知的凹凸 — 用常识。** 当 diff 很大时，凹凸 MINOR（或 MAJOR），而不是 PATCH。PATCH 用于错误修复和小添加；MINOR 用于实质性新能力或实质性减少；MAJOR 用于破坏性变更。粗略的检验标准（不要当作规则，当作气味检查）：

- **PATCH (X.Y.Z+1.0)**：错误修复、文档调整、小的添加性变更、添加的单个测试/文件。净 diff 在 ~500 行以下，无新的面向用户的能力。
- **MINOR (X.Y+1.0.0)**：新能力发布（skill、工具、命令、大重构）、实质性代码减少（压缩、迁移）或多文件协调变更。净 diff 在 ~2000 行以上添加/删除，或你会放在推文中的用户可见功能。
- **MAJOR (X+1.0.0.0)**：对公共表面的破坏性变更（CLI 标志重命名、skill 移除、配置格式变更）或大到足以成为博客文章头条的发布。

如果你在纠结"10K 添加 + 24K 删除真的是 PATCH 吗？"——不是。凹凸 MINOR。同样适用于"这添加了一个全新的测试工具，有 6 个新 E2E 测试 + 助手实用程序"——MINOR。凹凸级别是与用户关于这是什么类型的发布的沟通；不要低估它。

当合并 origin/main 带来更高的 VERSION 时，根据你分支工作的**规模**重新评估凹凸级别，而不仅仅是 main 是否向前移动。如果 main 凹凸了 MINOR 且你的分支也是实质性变更，你再次凹凸 MINOR（例如，main 在 v1.14.0.0，你的分支降落在 v1.15.0.0）。

**VERSION 和 CHANGELOG 是分支范围的。** 每个发布的功能分支都获得自己的版本凹凸和 CHANGELOG 条目。条目描述**这个**分支添加了什么——不是 main 上已经有什么。

**CHANGELOG 条目是 main 和发布分支之间的 diff — 用户升级时得到的。** 不是分支如何到达那里。到达条目的读者应该学习他们现在能做什么以前不能做的；他们不应该学习分支的内部版本凹凸、我们在分支中期捕获并修复的错误、我们运行的计划审查，或我们压缩的提交。那是分支开发叙事。它属于 PR 描述和提交消息，不属于 CHANGELOG。

**永远不要在 CHANGELOG 条目中引用分支内部版本。** 如果你的分支在开发期间将 VERSION 从 v1.5.0.0 → v1.5.1.0 → v1.6.0.0 凹凸，并且只有最终的 v1.6.0.0 发布到 main，条目必须读作 v1.5.1.0 从未存在过。具体来说，永远不要写：
- "v1.5.1.0 有一个 v1.6.0.0 修复的错误" — 读者不知道 v1.5.1.0；它是一个分支内部工件。
- "v1.5.1.0 的发布头条被破坏了，因为……" — 同样的原因。从 main 的角度来看，v1.5.1.0 从未发布。
- "修复前测试编码了破坏的行为" — 那是贡献者的胜利，不是用户利益。
- "两次手术编辑，都在调度路径中" — 补丁的微观叙事。

相反，描述发布的系统："Browser-skills 端到端运行，具有预期的 tab 访问语义。" 如果发布系统的某个属性值得指出（例如，"skill 生成获得每missive 的 tab 访问；pair-agent 隧道令牌需要所有权"），将其记录为属性，不是修复。发布的系统是用户得到的；通往该系统的路径对他们不可见。

**何时编写 CHANGELOG 条目：**
- 在 `/ship` 时（Step 13），而不是在开发期间或分支中期。
- 条目涵盖此分支对基础分支的**所有**提交。
- 永远不要将新工作折叠到来自先前版本（已在 main 上发布）的现有 CHANGELOG 条目中。如果 main 有 v0.10.0.0 且你的分支添加功能，凹凸到 v0.10.1.0 并创建新条目——不要编辑 v0.10.0.0 条目。

**编写前的关键问题：**
1. 我在哪个分支上？**这个**分支改变了什么？
2. 基础分支版本已经发布了吗？（如果是，凹凸并创建新条目。）
3. 此分支上的现有条目是否已经涵盖了早期工作？（如果是，用最终版本的一个统一条目替换它。）

**合并 main 并不意味着采用 main 的版本。** 当你将 origin/main 合并到功能分支时，main 可能带来新的 CHANGELOG 条目和更高的 VERSION。你的分支仍然需要**自己的**凹凸。如果 main 在 v0.13.8.0 且你的分支添加功能，凹凸到 v0.13.9.0 并创建新条目。永远不要将你已经发布到 main 的条目中。你的条目放在上面，因为你的分支接下来发布。

**合并 main 后，始终检查：**
- CHANGELOG 是否有你自己分支的条目，与 main 的条目分开？
- VERSION 是否高于 main 的 VERSION？
- 你的条目是否是 CHANGELOG 中最上面的条目（高于 main 的最新条目）？
如果任何答案为否，在继续之前修复它。

**任何移动、添加或删除条目的 CHANGELOG 编辑后，** 立即运行 `grep "^## \[" CHANGELOG.md` 验证无重复和合理的逆时序。版本号之间的间隙是可以的。在 main 上没有先前的 v1.5.2.0 或 v1.5.3.0 条目的情况下在 v1.6.4.0 发布的分支是正确的——那些是从未降落的分支内部版本号。不要用占位符条目回填间隙。

**永远不要孤立分支内部版本。** 如果你的分支在开发期间将 VERSION 凹凸了几次（比如 v1.5.1.0 → v1.5.2.0 → v1.6.4.0）且那些早期条目从未发布到 main，最终发布将所有这些折叠到最终版本（v1.6.4.0）的单个条目中。折叠它们——删除旧条目并将它们的内容移动到最终条目，相应地重新调整版本表列。读者看到一个发布，不是分支日记。间隙是可以的（v1.6.3.0 → v1.6.4.0，中间在 main 上没有 v1.5.x 是正确的）。

CHANGELOG.md 是**给用户的**，不是给贡献者。像产品发布说明一样写它：

- 以用户现在可以**做什么**开头，他们以前不能做的。推销功能。
- 使用简单语言，不是实施细节。"你现在可以……" 不是 "重构了……"
- **永远不要提及 TODOS.md、内部跟踪、评估基础设施或贡献者面对的详细信息。** 这些对用户是不可见的，对他们毫无意义。
- 将贡献者/内部变更放在底部的单独"Just for contributors"部分。
- 每个条目都应该让某人想"哦不错，我想试试。"
- 无术语：说"每个问题现在告诉你你在哪个项目和分支上" 不是 "通过 preamble 解析器在 skill 模板中标准化 AskUserQuestion 格式"。

**仅记录 main 和此变更之间发布的内容。** 读者不在乎我们如何到达这里。始终排除在 CHANGELOG 之外：

- 分支重新同步、与 main 的合并提交、变基活动。
- 计划批准、审查结果（CEO / 工程 / 设计 / 外部声音 / codex 发现）、AskUserQuestion 决策、范围协商。
- "工作已排队"、"计划已批准"、"进行中"、"稍后将发布" — CHANGELOG 记录的是**已发布**的内容，不是**可能发布**的内容。
- 当没有面向用户的实际工作时，版本凹凸整理。

如果基础分支版本和此版本之间的 diff 没有面向用户的变更（仅合并、仅 CHANGELOG 编辑、仅占位符工作），诚实的条目是一句话："用于分支领先纪律的版本凹凸。尚无面向用户的变更。" 停止。不要填充。不要解释最终发布的计划。不要叙述分支的历史。当真正的工作降落时，条目将在 `/ship` 时替换此条目。

### 发布摘要格式（每个 `## [X.Y.Z]` 条目）

`CHANGELOG.md` 中的每个版本条目必须以 GStack/Garry 声音的发布摘要部分开始，一个视口大小的散文 + 表格，像裁决一样落地，不是营销。项目化变更日志（子部分、要点、文件）位于该摘要**下方**，由 `### Itemized changes` 标题分隔。

发布摘要部分由人类、自动更新代理和任何决定升级的人阅读。项目化列表供需要确切知道什么变更的代理使用。

每个 `## [X.Y.Z]` 条目顶部的结构：

1. **两行粗体标题**（总共 10-14 个词）。应该像裁决一样落地，不营销。听起来像今天发布并关心它是否有效的人。
2. **引言段落**（3-5 个句子）。发布了什么，用户发生了什么变更。具体、具体，无 AI 词汇，无 em dash，无炒作。
3. **一个"The X numbers that matter"部分**，包含：
   - 一个简短的设置段落，命名数字的来源（真实生产部署**或**可重现的基准，命名要运行的文件/命令）。
   - 一个包含 3-6 个关键指标的表，带有 BEFORE / AFTER / Δ 列。
   - 第二个可选表，用于按类别细分（如果相关）。
   - 1-2 句话以最具体的用户术语解释最引人注目的数字。
4. 一个**"What this means for [受众]"**结束段落（2-4 个句子），将指标与真实工作流变更联系起来。以要做什么结束。

发布摘要的声音规则：
- 无 em dash（使用逗号、句号、"..."）。
- 无 AI 词汇（delve、robust、comprehensive、nuanced、fundamental 等）或禁止短语（"here's the kicker"、"the bottom line" 等）。
- 真实数字、真实文件名、真实命令。不是"快" 而是 "~30s on 30K pages"。
- 短段落，将一句重拳与 2-3 句连贯混合。
- 连接到用户结果："agent 少了 ~3x 的阅读" 击败 "improved precision"。
- 对质量直接。"设计良好" 或 "这是一团糟。" 不绕弯。

来源材料：
- CHANGELOG 先前条目用于先前上下文。
- 基准文件或 `/retro` 输出用于头条数字。
- 最近的提交（`git log <prev-version>..HEAD --oneline`）用于发布的内容。
- 不要编造数字。如果指标不在基准或生产数据中，不要包含它。如果被问到，说"尚无测量"。

目标长度：摘要约 250-350 个单词。应渲染为一个视口。

### 项目化变更（在发布摘要下方）

写 `### Itemized changes` 并继续详细的子部分（Added、Changed、Fixed、Just for contributors）。上述用户面对的声音指导规则之上，加上：

- **始终归功于社区贡献。** 当条目包含社区 PR 的工作时，用 `Contributed by @username` 命名贡献者。贡献者做了真正的工作。每次公开感谢他们，无例外。

## AI 努力压缩

在估计或讨论努力时，始终同时展示人类团队和 CC+gstack 时间：

| 任务类型 | 人类团队 | CC+gstack | 压缩比 |
|----------|----------|-----------|--------|
| 脚手架 / 脚手架 | 2 天 | 15 分钟 | ~100x |
| 测试编写 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| 错误修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构 / 设计 | 2 天 | 4 小时 | ~5x |
| 研究 / 探索 | 1 天 | 3 小时 | ~3x |

完整性很便宜。当完整实现可达成时，不要推荐捷径。沸腾海洋——完整的事情是目标；仅真正无关的多季度迁移是独立范围，从来不是捷径的借口。参见 skill preamble 中的完整性原则了解完整哲学。

## 先搜索，后构建

在设计涉及并发、不熟悉模式、基础设施或运行时/框架可能有内置的任何解决方案之前：

1. 搜索"{runtime} {thing} built-in"
2. 搜索"{thing} best practice {current year}"
3. 检查官方运行时/框架文档

三层知识：久经考验（第1层）、新颖且流行（第2层）、第一原理（第3层）。最重视第3层。参见 ETHOS.md 了解完整的建设者哲学。

## 本地计划

贡献者可以将长期远景文档和设计文档存储在 `~/.gstack-dev/plans/` 中。这些是仅本地的（不提交）。审查 TODOS.md 时，检查 `plans/` 中可能准备好提升到 TODOS 或实现的计划。

## E2E 评估失败归因协议

当 E2E 评估在 `/ship` 或其他工作流期间失败时，**永远不要在没有证明的情况下声称"与我们的变更无关"**。这些系统有不可见的耦合——prelude 文本变更影响 agent 行为，新助手变更时序，重新生成的 SKILL.md 改变提示上下文。

**在将失败归因为"预先存在"之前需要做的：**
1. 在 main（或基线分支）上运行相同的评估并显示它也在那里失败
2. 如果它在 main 上通过但在分支上是你的变更。追踪归因。
3. 如果你无法在 main 上运行，说"未验证——可能相关也可能不相关"并将其标记为 PR 正文中的风险

没有收据的"预先存在"是懒惰的主张。证明它，或者不要这样说。

## 长期运行的任务：不要放弃

在运行评估、E2E 测试或任何长期运行的后台任务时，**轮询直到完成**。使用 `sleep 180 && echo "ready"` + `TaskOutput` 每 3 分钟循环一次。永远不要切换到阻塞模式并在轮询超时时放弃。永远不要说"我会在完成时收到通知"并停止检查——继续循环，直到任务完成或用户告诉你停止。

完整的 E2E 套件可能需要 30-45 分钟。那是 10-15 个轮询循环。全部完成。在每个检查时报告进度（哪些测试通过了，哪些在运行，到目前为止的任何失败）。用户想看运行完成，不是承诺你稍后会检查。

## 作为 agent 运行评估：始终分离（SIGTERM 证明）

当**你（agent/工具）**启动长期评估/基准运行时，通过 `bin/gstack-detach` 运行它——**绝不**作为普通后台 Bash 任务。普通后台任务存在于工具进程组中，因此 SIGTERM（"礼貌退出"）、停止的监控器或中断会在飞行中杀死运行（观察到：`script "test:gate"` 在运行约 40 分钟后被信号 SIGTERM 终止）。在 macOS 上，运行也可能死于空闲睡眠。`gstack-detach` 修复两者：一个全新的会话（逃脱组 SIGTERM）包裹在 `caffeinate -i` 中（阻止空闲睡眠）。

- 使用 `eval:bg*` 脚本（`eval:bg`、`eval:bg:all`、`eval:bg:gate`、`eval:bg:periodic`）——它们用 `gstack-detach` 包装评估命令，带有机器范围的 `gstack-evals` 锁（并发 worktree 序列化，而不是饱和共享模型 API）、每层看门狗和**运行范围**的日志，在 `~/.gstack-dev/eval-runs/` 下（无共享 `/tmp` 冲突）。每个打印其日志路径。或调用 `gstack-detach [--lock NAME] [--timeout SECS] [--label LBL] -- <cmd>` 直接用于任何长期 agent 作业。首先导出 `ANTHROPIC_API_KEY`（永远不要在 argv 中传递键）。
- 然后**轮询打印的日志文件**，使用死亡感知观察者在保证的 `### gstack-detach EXIT=<code> ###` 哨兵上中断（成功**和**失败都被标记，所以寂静永远不会被误认为成功）。分离的运行在你的观察者被收割后仍然存活，所以重新检查日志始终有效。
- 为什么需要锁：一个带有几个 Conductor worktree 的共享开发盒会限制模型 API，如果两个评估套件同时运行（每个 15 路并发），这会使 E2E 测试大量超时。锁使第二个运行等待，而不是碰撞。
- 人类在自己的终端中前台运行 `bun run test:evals` 不需要这个——Ctrl-C 是有意的。分离仅用于 agent 启动的运行。

## E2E 测试装置：提取，不要复制

**永远不要将完整的 SKILL.md 文件复制到 E2E 测试装置中。** SKILL.md 文件有 1500-2000 行。当 `claude -p` 读取那么大的文件时，上下文膨胀导致超时、脆弱的限制和比必要长 5-10 倍的测试。

相反，仅提取测试实际需要的部分：

```typescript
// 坏 — agent 读取 1900 行，在无关部分上消耗令牌
fs.copyFileSync(path.join(ROOT, 'ship', 'SKILL.md'), path.join(dir, 'ship-SKILL.md'));

// 好 — agent 读取 ~60 行，在 38 秒内完成而不是超时
const full = fs.readFileSync(path.join(ROOT, 'ship', 'SKILL.md'), 'utf-8');
const start = full.indexOf('## Review Readiness Dashboard');
const end = full.indexOf('\n---\n', start);
fs.writeFileSync(path.join(dir, 'ship-SKILL.md'), full.slice(start, end > start ? end : undefined));
```

还有，在运行有针对性的 E2E 测试来调试失败时：
- 在**前台**运行（`bun test ...`），不要用 `&` 和 `tee` 后台运行
- 永远不要 `pkill` 正在运行的评估进程并重新开始——你丢失结果并浪费金钱
- 一次干净的运行胜过三次杀死并重新开始的运行

## 发布原生 OpenClaw skill 到 ClawHub

原生 OpenClaw skill 位于 `openclaw/skills/gstack-openclaw-*/SKILL.md`。这些是手工制作的方法论 skill（不是由流水线生成）发布到 ClawHub，以便任何 OpenClaw 用户可以安装它们。

**发布：** 命令是 `clawhub publish`（不是 `clawhub skill publish`）：

```bash
clawhub publish openclaw/skills/gstack-openclaw-office-hours \
  --slug gstack-openclaw-office-hours --name "gstack Office Hours" \
  --version 1.0.0 --changelog "description of changes"
```

对每个 skill 重复：`gstack-openclaw-ceo-review`、`gstack-openclaw-investigate`、`gstack-openclaw-retro`。每次更新时凹凸 `--version`。

**认证：** `clawhub login`（打开浏览器进行 GitHub 认证）。`clawhub whoami` 验证。

**更新：** 相同的 `clawhub publish` 命令，带有更高的 `--version` 和 `--changelog`。

**验证：** `clawhub search gstack` 确认它们是活跃的。

## 部署到活动的 skill

活动的 skill 位于 `~/.claude/skills/gstack/`。变更后：

1. 推送你的分支
2. 在 skill 目录中获取并重置：`cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main`
3. 重建：`cd ~/.claude/skills/gstack && bun run build`

**如果你使用 gbrain：** 第 2 步中的 `git reset --hard` 恢复了脑感知（`GBRAIN_CONTEXT_LOAD` / `GBRAIN_SAVE_RESULTS`）块，这些块由 `gstack-config gbrain-refresh` 渲染到安装中（那些生成的块因设计与 `main` 不同）。部署后，重新运行 `gstack-config gbrain-refresh` 以恢复它们跨越你项目的所有 Claude 会话。它是幂等的。

或直接复制二进制文件：
- `cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse`
- `cp design/dist/design ~/.claude/skills/gstack/design/dist/design`

## Skill 路由

当用户的请求匹配可用的 skill 时，通过 Skill 工具调用它。有疑问时，调用 skill。

关键路由规则：
- 产品想法/头脑风暴 → 调用 /office-hours
- 战略/范围 → 调用 /plan-ceo-review
- 架构 → 调用 /plan-eng-review
- 设计系统/计划审查 → 调用 /design-consultation 或 /plan-design-review
- 完整审查流水线 → 调用 /autoplan
- 错误/错误 → 调用 /investigate
- QA/测试站点行为 → 调用 /qa 或 /qa-only
- 代码审查/diff 检查 → 调用 /review
- 视觉润色 → 调用 /design-review
- 发布/部署/PR → 调用 /ship 或 /land-and-deploy
- 保存进度 → 调用 /context-save
- 恢复上下文 → 调用 /context-restore

## 跨会话决策记忆

持久决策及其原理被捕获在仅追加的、事件溯源的存储中，位于 `~/.gstack/projects/<slug>/decisions.jsonl`，这样你和用户都不会重新争论一个已确定的调用或跨会话丢失"为什么"。这是可靠的、纯文件的路径：它在 gbrain 关闭时有效。（gbrain 语义召回是可选的增强层，永远不会是依赖。）

- **重新浮现**活动决策之前重新决定：`bin/gstack-decision-search`（`--recent N`、`--scope repo|branch|issue`、`--query KW`、`--all`、`--json`）。添加 `--semantic`（与 `--query` 一起）以在 gbrain 启动时从 gbrain 内存中附加相关命中；当 gbrain 关闭时，它降级为可靠的文件结果。会话启动已通过 Context Recovery 浮现了范围相关的活动决策。如果列出了决策，以其原理将其视为确定的；如果你即将明确反转它。
- **捕获**你或用户做出持久决策时：`bin/gstack-decision-log '{"decision":"...","rationale":"...","scope":"repo|branch|issue","source":"user|skill|agent","confidence":1-10}'`。用 `--supersede <id>` 反转先前的调用；用 `--redact <id>` 驱逐意外的秘密；用 `--compact` 将日志重写到活动集。非交互式（永远不提示）、注入清理和 HIGH 秘密阻止在写入时。
- **持久意味着：** 架构选择、范围削减、工具/供应商选择或先前调用的反转。不是轮次级别的编辑、措辞调整或任何容易重新推导的东西。捕获在源头策划——仅记录持久决策，否则存储变得嘈杂。

## GBrain 搜索指导（由 /sync-gbrain 配置）
<!-- gstack-gbrain-search-guidance:start -->

GBrain 在此机器上设置并同步。当问题是语义性的或者你还不知道确切标识符时，agent 应该优先使用 gbrain 而不是 Grep。

**此 worktree 固定到 worktree 范围的代码源**，通过仓库根部的 `.gbrain-source` 文件（kubectl 风格的上下文）。从此 worktree 下任何地方的 `gbrain code-def`、`code-refs`、`code-callers`、`code-callees` 或 `query` 调用默认路由到该源——无需 `--source` 标志。同一仓库的 Conductor 兄弟 worktree 每个都有自己的固定和自己的索引页面，所以语义结果匹配此 worktree 磁盘上的实际代码。

可通过 `gbrain` CLI 访问两个索引语料库：
- 此 worktree 的代码（通过 `.gbrain-source` 自动固定）。
- `~/.gstack/` 策划的内存（通过现有联合流水线注册为 `gstack-brain-<user>` 源）。

何时优先使用 gbrain：
- "X 在哪里处理？" / 语义意图，尚无确切字符串：
    `gbrain search "<terms>"` 或 `gbrain query "<question>"`
- "符号 Y 在哪里定义？" / 基于符号的代码问题：
    `gbrain code-def <symbol>` 或 `gbrain code-refs <symbol>`
- "什么调用了 Y？" / "Y 依赖于什么？":
    `gbrain code-callers <symbol>` / `gbrain code-callees <symbol>`
- "我们上次决定了什么？" / 过去的计划、回顾、学习：
    `gbrain search "<terms>" --source gstack-brain-<user>`

对于已知的精确字符串、正则表达式、多行模式和文件 glob，Grep 仍然是正确的。在有意义的代码变更后运行 `/sync-gbrain`；对于跨所有 worktree 的持续自动同步，每台机器运行一次 `gbrain autopilot --install`——gbrain 的守护进程按计划处理增量刷新。

安全：在 `gbrain autopilot` 活跃时不要运行 `/sync-gbrain`——当它检测到正在运行的 autopilot 时，编排器拒绝破坏性源操作，以避免与其竞速（#1734）。优先使用 `gbrain sources add --path <dir>`（无 `--url`）注册用户仓库：URL 管理的源可以自动重新克隆，它们的同步代码行走需要显式 `--allow-reclone` 选择加入。

<!-- gstack-gbrain-search-guidance:end -->

## 升级迁移

当变更修改了磁盘上状态（目录结构、配置格式、陈旧文件）的方式可能破坏现有用户安装时，将迁移脚本添加到 `gstack-upgrade/migrations/`。阅读 CONTRIBUTING.md 的"Upgrade migrations"部分了解格式和测试要求。升级 skill 在 `/gstack-upgrade` 期间、`./setup` 之后自动运行这些。

## 编译后的二进制文件 — 绝不提交 browse/dist/ 或 design/dist/

`browse/dist/` 和 `design/dist/` 目录包含编译的 Bun 二进制文件（`browse`、`find-browse`、`design`，每个约 58MB）。这些仅是 Mach-O arm64——它们在 Linux、Windows 或 Intel Mac 上不工作。`./setup` 脚本已经从源码为每个平台构建，所以检入的二进制文件是多余的。它们由于历史错误被 git 跟踪，应该最终用 `git rm --cached` 删除。

**永远不要暂存或提交这些文件。** 它们在 `git status` 中显示为已修改，因为尽管有 `.gitignore` 它们仍被跟踪——忽略它们。暂存文件时，始终使用特定文件名（`git add file1 file2`）——永远不要用 `git add .` 或 `git add -A`，这会意外包含二进制文件。

## 前缀设置

设置使用在顶层创建真实目录（不是符号链接），内部有 SKILL.md 符号链接（例如，`qa/SKILL.md -> gstack/qa/SKILL.md`）。这确保 Claude 将它们发现为顶级 skill，而不是嵌套在 `gstack/` 下。名称是短的（`qa`）或命名空间化的（`gstack-qa`），由 `~/.gstack/config.yaml` 中的 `skill_prefix` 控制。传递 `--no-prefix` 或 `--prefix` 跳过交互式提示。

**注意：** 将 gstack vendor 到项目仓库中已弃用。使用全局安装 + `./setup --team`。有关团队模式说明，请参阅 README.md。

**对于计划审查：** 当审查修改 skill 模板或 gen-skill-docs 流水线的计划时，考虑变更是否应该在上线前单独测试（特别是如果用户在其他窗口中活跃使用 gstack）。
