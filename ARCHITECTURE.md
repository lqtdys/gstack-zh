# 架构

这份文档解释了**为什么** gstack 要这样构建。有关设置和命令，见 CLAUDE.md。有关贡献，见 CONTRIBUTING.md。

## 核心理念

gstack 为 Claude Code 提供持久浏览器和一组自带观点的工作流技能。浏览器是困难的部分 — 其他都是 Markdown。

关键洞察：AI 代理与浏览器交互需要**亚秒级延迟**和**持久状态**。如果每个命令都冷启动浏览器，你每次工具调用要等待 3-5 秒。如果浏览器在命令之间死掉，你丢失 cookie、标签页和登录会话。因此 gstack 运行一个长期存在的 Chromium 守护进程，CLI 通过 localhost HTTP 与之通信。

```
Claude Code                     gstack
─────────                      ──────
                               ┌──────────────────────┐
  Tool call: $B snapshot -i    │  CLI (compiled binary)│
  ─────────────────────────→   │  • reads state file   │
                               │  • POST /command      │
                               │    to localhost:PORT   │
                               └──────────┬───────────┘
                                          │ HTTP
                               ┌──────────▼───────────┐
                               │  Server (Bun.serve)   │
                               │  • dispatches command  │
                               │  • talks to Chromium   │
                               │  • returns plain text  │
                               └──────────┬───────────┘
                                          │ CDP
                               ┌──────────▼───────────┐
                               │  Chromium (headless)   │
                               │  • persistent tabs     │
                               │  • cookies carry over  │
                               │  • 30min idle timeout  │
                               └───────────────────────┘
```

首次调用启动一切（~3秒）。之后每次：约100-200毫秒。

## 为什么用 Bun

Node.js 能用。Bun 在这里更好，原因有三：

1. **编译二进制。** `bun build --compile` 生成单个 ~58MB 可执行文件。运行时无 `node_modules`，无 `npx`，无 PATH 配置。二进制直接运行。这很重要，因为 gstack 安装到 `~/.claude/skills/`，用户不期望在那里管理 Node.js 项目。

2. **原生 SQLite。** Cookie 解密直接读取 Chromium 的 SQLite cookie 数据库。Bun 内置 `new Database()` — 没有 `better-sqlite3`，没有原生插件编译，没有 gyp。少一个在不同机器上可能出问题的东西。

3. **原生 TypeScript。** 服务器在开发时以 `bun run server.ts` 运行。没有编译步骤，没有 `ts-node`，没有需要调试的源码映射。编译二进制用于部署；源码文件用于开发。

4. **内置 HTTP 服务器。** `Bun.serve()` 快速、简单，不需要 Express 或 Fastify。服务器总共处理约 10 个路由。框架会引入额外开销。

瓶颈始终是 Chromium，而不是 CLI 或服务器。Bun 的启动速度（编译二进制约1ms，Node 约100ms）是好，但不是我们选择它的原因。编译二进制和原生 SQLite 才是。

## 守护进程模型

### 为什么不为每个命令启动浏览器？

Playwright 可以以约 2-3 秒启动 Chromium。对于单次截图，这没问题。对于带有 20+ 命令的 QA 会话，则是 40+ 秒的浏览器启动开销。更糟：你丢失命令之间的所有状态。Cookie、localStorage、登录会话、打开的标签页 — 都没了。

守护进程模型意味着：

- **持久状态。** 登录一次，保持登录。打开标签页，保持打开。localStorage 跨命令持久化。
- **亚秒级命令。** 首次调用后，每个命令只是 HTTP POST。约 100-200 毫秒往返包括 Chromium 的工作。
- **自动生命周期。** 服务器在首次使用时自动启动，30 分钟空闲后自动关闭。无需进程管理。

### 状态文件

服务器写入 `.gstack/browse.json`（通过 tmp + rename 原子写入，模式 0o600）：

```json
{ "pid": 12345, "port": 34567, "token": "uuid-v4", "startedAt": "...", "binaryVersion": "abc123" }
```

CLI 读取此文件以查找服务器。如果文件缺失或服务器健康检查失败，CLI 生成一个新服务器。在 Windows 上，基于 PID 的进程检测在 Bun 二进制中不可靠，所以健康检查（GET /health）是所有平台上的主要存活信号。

### 端口选择

10000-60000（冲突时最多重试 5 次）。这意味着 10 个 Conductor 工作区可以各自运行自己的浏览守护进程，零配置，零端口冲突。旧方法（扫描 9400-9409）在多工作区设置中经常出问题。

### 版本自动重启

构建将 `git rev-parse HEAD` 写入 `browse/dist/.version`。在每次 CLI 调用时，如果二进制版本与服务器的 `binaryVersion` 不匹配，CLI 杀死旧服务器并启动一个。这完全防止了"过时的二进制"一类 bug — 重建二进制，下一个命令自动拾取。

## 安全模型

### 仅限 localhost

HTTP 服务器绑定到 `127.0.0.1`，不是 `0.0.0.0`。无法从网络到达。

### 双监听器隧道架构 (v1.6.0.0)

当用户运行 `pair-agent --client` 时，守护进程启动 ngrok 隧道，以便远程配对代理可以驱动浏览器。将完整的守护进程表面暴露在即使是随机 ngrok 子域下，也意味着 `/health` 在任何 Origin 欺骗时泄漏 root token，而 `/cookie-picker` 将 token 嵌入任何调用者可以获取的 HTML 中的 token。

修复是 **两个 HTTP 监听器**，不是一个：

- **本地监听器** (`127.0.0.1:LOCAL_PORT`) — 总是绑定。提供引导（`/health` 带 token 传递）、`/cookie-picker`、`/inspector/*`、`/welcome`、`/refs`、侧边栏代理 API 和完整命令表面。从不转发。
- **隧道监听器** (`127.0.0.1:TUNNEL_PORT`) — 在 `/tunnel/start` 时惰性绑定，在 `/tunnel/stop` 时拆除。提供锁定允许列表：`/connect`（配对仪式，未认证 + 速率限制），`/command`（仅限 scoped token，进一步限制为浏览器驱动命令允许列表），和 `/sidebar-chat`。其他所有路由 404。

ngrok 仅转发隧道端口。安全属性来自**物理端口分离**：隧道调用器无法到达 `/health` 或 `/cookie-picker`，因为这些路径不存在于该 TCP 套接字。头部推断（检查 `x-forwarded-for`，检查 Origin）不可靠（ngrok 头部行为会变化；本地代理可以添加这些头部）；套接字分离不会。

| 端点 | 本地监听器 | 隧道监听器 | 备注 |
|---|---|---|---|
| `GET /health` | public（无 token，除非 headed/extension） | 404 | Extension 的 token 引导仅在本地发生 |
| `GET /connect` | public (`{alive:true}`) | public (`{alive:true}`) | 隧道存活探针 |
| `POST /connect` | public（速率限制 300/min） | public（速率限制） | pair-agent 的设置密钥交换 |
| `POST /command` | auth（Bearer root 或 scoped） | auth（仅限 scoped，允许列表中的命令） | root token 在隧道上 = 403 |
| `POST /sidebar-chat` | auth | auth | 让远程代理发布到本地侧边栏 |
| `POST /pair` | root-only | 404 | 配对 mint — 本地操作员动作 |
| `POST /tunnel/{start,stop}` | root-only | 404 | 守护进程配置 |
| `POST /token`、`DELETE /token/:id` | root-only | 404 | Scoped token mint/revoke |
| `GET /cookie-picker`、`GET /cookie-picker/*` | public UI、auth API | 404 | 本地仅限 — 读取本地浏览器数据库 |
| `GET /inspector`、`/inspector/events` 等 | auth | 404 | Extension 回调，仅限本地 |
| `GET /welcome` | public | 404 | GStack 浏览器登录页，仅限本地 |
| `GET /refs` | auth | 404 | Ref map — 内部状态 |
| `GET /activity/stream` | Bearer 或 HttpOnly `gstack_sse` cookie | 404 | SSE。`?token=` 查询参数不再被接受 |
| `GET /inspector/events` | Bearer 或 HttpOnly `gstack_sse` cookie | 404 | SSE。同上 |
| `POST /sse-session` | auth（Bearer） | 404 | 铸造仅查看 30 分钟 SSE 会话 cookie |

**隧道表面拒绝日志。** 隧道监听器上的每个拒绝（`path_not_on_tunnel`、`root_token_on_tunnel`、`missing_scoped_token`、`disallowed_command:*`）都异步记录到 `~/.gstack/security/attempts.jsonl`，带时间戳、来源 IP（来自 `x-forwarded-for`）、路径和方法。全局速率限制为 60 次写入/分钟以防止日志洪水 DoS。与提示注入扫描器共享尝试日志。

**SSE 会话 cookie。** EventSource 无法发送 Authorization 头，所以 extension 在引导时用 root Bearer 一次性 POST `/sse-session`，并接收一个 30 分钟仅查看 cookie（`gstack_sse`、HttpOnly、SameSite=Strict） 。cookie 仅对 `/activity/stream` 和 `/inspector/events` 有效 — 它不是 scoped token，不能用于 `/command`。作用域隔离由模块边界执行：`sse-session-cookie.ts` 没有从 `token-registry.ts` 导入。

**本波的非目标**（跟踪为 #1136）：cookie-import-browser 路径用 `--remote-debugging-port=<random>` 启动 Chrome。在 Windows App-Bound Encryption v20 下，同用户本地进程可以连接到该端口并解密 v20 cookie — 相对于直接读取 SQLite DB 的升级路径（没有 DPAPI 上下文无法解密 v20）。修复方向是 `--remote-debugging-pipe` 而不是 TCP；需要重构 CDP 客户端。

### Bearer 认证

每个服务器会话生成一个随机 UUID token，以模式 0o600（仅所有者读取）写入状态文件。每个改变浏览器状态的 HTTP 请求必须包含 `Authorization: Bearer ***。如果 token 不匹配，服务器返回 401。

这阻止了同一台机器上的其他进程与浏览服务器通信。cookie picker UI（`/cookie-picker`）和健康检查（`/health`）在本地监听器上免除 — 它们是 127.0.0.1 绑定并且不执行命令。在隧道监听器上除了 `/connect`，都不免除。

### Cookie 安全

Cookie 是 gstack 处理的最敏感数据。设计：

1. **Keychain 访问需要用户批准。** 浏览器的第一次 cookie 导入触发 macOS Keychain 对话框。用户必须点击"允许"或"始终允许。"gstack 从不静默访问凭证。

2. **解密在进程内发生。** Cookie 值在内存在解密（PBKDF2 + AES-128-CBC），加载到 Playwright 上下文中，从不以明文写入磁盘。cookie picker UI 从不显示 cookie 值 — 只有域名和计数。

3. **数据库只读。** gstack 复制 Chromium cookie DB 到临时文件（避免与运行中的浏览器发生 SQLite 锁冲突）并以只读方式打开。它从不修改真实浏览器的 cookie 数据库。

4. **密钥缓存按会话。** Keychain 密码 + 派生的 AES 密钥在服务器生命周期内缓存内存中。当服务器关闭（空闲超时或显式停止），缓存消失。

5. **日志中无 cookie 值。** 控制台、网络和对话框日志从未包含 cookie 值。`cookies` 命令输出 cookie 元数据（域名、名称、过期时间），但值被截断。

### Shell 注入预防

浏览器注册表（Comet、Chrome、Arc、Brave、Edge）是硬编码的。数据库路径从已知常量构建，从不从用户输入建构。Keychain 访问使用带有显式参数数组的 `Bun.spawn()`，而不是 shell 字符串插值。

### Unicode 清理在服务器出口 (v1.38.0.0)

CDP 收获的页面内容可能包含孤立的 UTF-16 代理半体（来自页面上的 JavaScript 字符串处理的高或低代理）。当这些到达 `JSON.stringify` 时，Bun 发出类似 `\uD800` 样式的转义序列，下游消费者的 `JSON.parse` 接受，但 Anthropic API 以 400 拒绝 — 将一个奇怪的页面变成会话终止错误。防御是单点，应用于发送派生字符串的每个服务器出口。

| 出口路径 | 模块 | 清理位置 |
|---|---|---|
| `POST /command` (HTTP) | `browse/src/server.ts` | `handleCommandInternal` 包装器（清理 `handleCommandInternalImpl` 的结果） |
| `POST /command/batch` | `browse/src/server.ts` | 相同的包装器 — 批量消费者继承 |
| `GET /activity/stream` (SSE) | `browse/src/server.ts` | `sanitizeReplacer` 传递给 `JSON.stringify` |
| `GET /inspector/events` (SSE) | `browse/src/server.ts` | `sanitizeReplacer` 传递给 `JSON.stringify` |

`sanitizeReplacer` 是一个 `JSON.stringify` replacer 函数，在编码期间清理每个字符串值。在这里，后字符串化正则表达式不起作用 — `JSON.stringify` 在正则表达式能匹配之前已将 `\uD800` 转换为字面转义序列 `"\\ud800"`，所以 replacer 必须在编码管道内运行。纯字符串助手 `sanitizeLoneSurrogates` 直接用于 `text/plain` 响应。

**架构不变式。** 每个发送页面内容派生字符串的新 SSE/WebSocket 写入器或 HTTP 响应都必须经过两条路径之一：`JSON.stringify(payload, sanitizeReplacer)` 用于对象负载，或 `sanitizeLoneSurrogates(body)` 用于文本体。绕过两者的新表面将会与系统失去同步。`server.ts` 中两个 SSE 生产者的内联注释都说明了；`browse/test/server-sanitize-surrogates.test.ts` 用 bug-repro + 不变式测试固定接线（`handleCommandInternalImpl` 重命名、中央清理行、replacer 存在、SSE 生产者使用 replacer）。

### 提示注入防御（侧边栏代理）

Chrome 侧边栏代理有工具（Bash、Read、Glob、Grep、WebFetch）并读取恶意网页，所以它是 gstack 中最暴露于提示注入的部分。防御是分层而非单点。

1. **L1-L3 内容安全（`browse/src/content-security.ts`）。** 在每个页面内容命令和每个工具输出上运行：数据标记、隐藏元素剥离、ARIA 正则表达式、URL 阻止列表和信任边界包络封装。在服务器和代理两侧应用。

2. **L4 ML 分类器 — TestSavantAI（`browse/src/security-classifier.ts`）。** 一个 22MB BERT-small ONNX 模型（int8 量化），与代理捆绑。在本地运行，无网络。在 Claude 看到每个用户消息和每个 Read/Glob/Grep/WebFetch 工具输出之前扫描。通过 `GSTACK_SECURITY_ENSEMBLE=deberta` 选择加入 721MB DeBERTa-v3 集成。

3. **L4b 记录分类器。** Claude Haiku 通行证，查看完整对话形状（用户消息、工具调用、工具输出），而不仅仅是文本。由 `LOG_ONLY: 0.40` 控制，因此大部分干净流量跳过付费调用。

4. **L5 canary token（`browse/src/security.ts`）。** 在会话开始时注入系统提示中的随机 token。跨 `text_delta` 和 `input_json_delta` 流式传输的滚动缓冲区检测捕获 token（如果它出现在 Claude 的输出、工具参数、URL 或文件写入中的任何位置）。确定性 BLOCK — 如果 token 泄漏，攻击者说服 Claude 泄露了系统提示，会话结束。

5. **L6 集成合并器（`combineVerdict`）。** BLOCK 需要两个 ML 分类器 >= `WARN` (0.75) 的协议，而不是单次置信度。这是 Stack Overflow 写作误报缓解。在工具输出扫描上，单层高置信度直接 BLOCK — 内容不是用户创作的，所以 FP 顾虑不适用。

**关键约束：** `security-classifier.ts` 仅在侧边栏代理进程中运行，从不在编译的浏览二进制中运行。`@huggingface/transformers` v4 需要 `onnxruntime-node`，它从 Bun 编译的临时提取目录失败 `dlopen`。只有纯字符串部分（canary 注入/检查、判定合并器、攻击日志、状态）在 `security.ts` 中，`server.ts` 可以安全地导入。

**Env 旋钮：** `GSTACK_SECURITY_OFF=1` 是一个真正的终止开关（跳过 ML 扫描，canary 仍注入）。模型缓存位于 `~/.gstack/models/testsavant-small/`（112MB，首次运行）和 `~/.gstack/models/deberta-v3-injection/`（721MB，仅在集成启用时）。攻击日志位于 `~/.gstack/security/attempts.jsonl`（加盐 sha256 + 域，在 10MB 处轮换，每 5 代）。每台设备盐值在 `~/.gstack/security/device-salt`（0600），在进程中缓存以在 FS 不可写环境中存活。

**可见性。** 侧边栏头部显示盾牌图标（绿色/琥珀色/红色），通过 `/sidebar-chat` 轮询。在 canary 泄漏或 BLOCK 判定时会出现居中横幅，带有准确的层级分数。`bin/gstack-security-dashboard` 聚合本地尝试；`supabase/functions/community-pulse` 聚合选择加入的跨用户遥测。

## Ref 系统

Ref（`@e1`、`@e2`、`@c1`）是代理在不编写 CSS 选择器或 XPath 的情况下寻址页面元素的方式。

### 工作原理

```
1. Agent runs: $B snapshot -i
2. Server calls Playwright's page.accessibility.snapshot()
3. Parser walks the ARIA tree, assigns sequential refs: @e1, @e2, @e3...
4. For each ref, builds a Playwright Locator: getByRole(role, { name }).nth(index)
5. Stores Map<string, RefEntry> on the BrowserManager instance (role + name + Locator)
6. Returns the annotated tree as plain text

Later:
7. Agent runs: $B click @e3
8. Server resolves @e3 → Locator → locator.click()
```

### 为什么是在 DOM 外侧的 Locators，而非 DOM 突变

明显的方法是注入 `data-ref="@e1"` 属性到 DOM 中。这在这些上面会坏：

- **CSP（内容安全策略）。** 许多生产现场阻止脚本 DOM 修改。
- **React/Vue/Svelte 水合。** 框架 reconciliation 可以剥离注入的属性。
- **Shadow DOM。** 无法从外部到达 shadow root 内部。

Playwright Locators 在 DOM 外部。它们使用可访问性树（Chromium 内部维护）和 `getByRole()` 查询。没有 DOM 突变，没有 CSP 问题，没有框架冲突。

### Ref 生命周期

Ref 在导航上清除（主框架上的 `framenavigated` 事件）。这是正确的 — 导航后，所有 Locators 都过时了。代理必须再次运行 `snapshot` 以获取新鲜 ref。这是刻意的设计：过时的 ref 应该大声失败，而不是点到错误的元素。

### Ref 过时检测

SPA 可以在不触发 `framenavigated` 的情况下改变 DOM（例如 React 路由转换、标签页切换、模态打开）。这使得尽管页面 URL 没有改变，ref 仍会过时。为了捕捉这一点，`resolveRef()` 在使用任何 ref 之前执行异步 `count()` 检查：

```
resolveRef(@e3) → entry = refMap.get("e3")
                → count = await entry.locator.count()
                → if count === 0: throw "Ref @e3 is stale — element no longer exists. Run 'snapshot' to get fresh refs."
                → if count > 0: return { locator }
```

这快速失败（~5ms 开销），而不是让 Playwright 的 30 秒操作超时过期在缺失元素上。`RefEntry` 存储 `role` 和 `name` 元数据以及 Locator，以便错误消息可以告诉代理那个元素是什么。

### 光标交互 ref (@c)

`-C` 标志找到可点击但不在 ARIA 树中的元素 — 用 `cursor: pointer` 样式化的东西，带 `onclick` 属性的元素，或自定义 `tabindex`。这些获取 `@c1`、`@c2` 引用，在单独的命名空间中。这捕捉了框架渲染为 `<div>` 但实际上是按钮的自定义组件。

## 日志架构

三个环形缓冲区（各 50,000 条目，O(1) 推送）：

```
Browser events → CircularBuffer (in-memory) → Async flush to .gstack/*.log
```

控制台消息、网络请求和对话框事件各有自己的缓冲区。刷新每秒发生一次 — 服务器仅追加自上次刷新以来的新条目。这意味着：

- HTTP 请求处理从不被磁盘 I/O 阻塞
- 日志在服务器崩溃时存活（最多丢失 1 秒数据）
- 内存有界（50K 条目 × 3 缓冲区）
- 磁盘文件仅追加，外部工具可读

`console`、`network` 和 `dialog` 命令从内存缓冲区读取，而不是磁盘。磁盘文件用于事后调试。

## SKILL.md 模板系统

### 问题

SKILL.md 文件告诉 Claude 如何使用浏览器命令。如果文档列出不存在的标志，或遗漏添加的命令，代理会遇到错误。手动维护的文档总是偏离代码。

### 解决方案

```
SKILL.md.tmpl          (人类编写的散文 + 占位符)
       ↓
gen-skill-docs.ts      (读取源码元数据)
       ↓
SKILL.md               (提交的，自动生成的部分)
```

模板包含需要人类判断的工作流、技巧和示例。占位符在构建时从源码填充：

| 占位符 | 来源 | 它生成什么 |
|-------------|--------|-------------------|
| `{{COMMAND_REFERENCE}}` | `commands.ts` | 分组的命令表 |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` | 带示例的标志参考 |
| `{{PREAMBLE}}` | `gen-skill-docs.ts` | 启动块：更新检查、会话跟踪、贡献者模式、AskUserQuestion 格式 |
| `{{BROWSE_SETUP}}` | `gen-skill-docs.ts` | 二进制发现 + 设置说明 |
| `{{BASE_BRANCH_DETECT}}` | `gen-skill-docs.ts` | PR 目标技能的动态基本分支检测（ship、review、qa、plan-ceo-review） |
| `{{QA_METHODOLOGY}}` | `gen-skill-docs.ts` | /qa 和 /qa-only 的共享 QA 方法论块 |
| `{{DESIGN_METHODOLOGY}}` | `gen-skill-docs.ts` | /plan-design-review 和 /design-review 的共享设计审计方法论 |
| `{{REVIEW_DASHBOARD}}` | `gen-skill-docs.ts` | /ship 预飞的审查准备度仪表盘 |
| `{{TEST_BOOTSTRAP}}` | `gen-skill-docs.ts` | 测试框架检测、引导、/qa、/ship、/design-review 的 CI/CD 设置 |
| `{{CODEX_PLAN_REVIEW}}` | `gen-skill-docs.ts` | /plan-ceo-review 和 /plan-eng-review 的可选跨模型计划审查（Codex 或 Claude 子代理回退） |
| `{{DESIGN_SETUP}}` | `resolvers/design.ts` | `$D` 设计二进制的发现模式，镜像 `{{BROWSE_SETUP}}` |
| `{{DESIGN_SHOTGUN_LOOP}}` | `resolvers/design.ts` | /design-shotgun、/plan-design-review、/design-consultation 的共享比较板反馈循环 |
| `{{UX_PRINCIPLES}}` | `resolvers/design.ts` | 用户行为（扫描、满意、善意、主干测试），用于 /design-html、/design-shotgun、/design-review、/plan-design-review |
| `{{GBRAIN_CONTEXT_LOAD}}` | `resolvers/gbrain.ts` | 脑优先上下文搜索，含关键词提取、健康感知和数据-研究路由。注入到 10 个脑感知技能中。在非物质主机上抑制。 |
| `{{GBRAIN_SAVE_RESULTS}}` | `resolvers/gbrain.ts` | 节后脑持久化，含实体丰富、节流技能和按技能保存说明。8 种技能特定的保存格式。 |

这在结构上是可靠的 — 如果命令存在于代码，就出现在文档。如果不存在，就无法出现。

### 序言

每个技能都以一个 `{{PREAMBLE}}` 块开始，在技能自身逻辑之前运行。它在单个 bash 命令中处理五件事：

1. **更新检查** — 调用 `gstack-update-check`，报告是否有可用升级。
2. **会话跟踪** — 触碰 `~/.gstack/sessions/$PPID` 并计数活跃会话（最近 2 小时内修改的文件）。当 3+ 运行时，所有技能进入"ELI16 模式" — 每个问题根据是因为用户正在 juggling 窗口重新引导用户上下文。
3. **运维自我改进** — 在每个技能会话结束时，代理反思失败（CLI 错误、错误方法、项目怪癖）并将运维学习记录到项目的 JSONL 文件中供未来会话使用。
4. **AskUserQuestion 格式** — 通用格式：上下文、问题、`RECOMMENDATION: 选择 X 因为 ___`、编号选项。在所有技能中保持一致。
5. **先搜索再构建** — 在构建基础设施或不熟悉的模式之前，先搜索。知识三层：久经考验（第1层）、新潮流行（第2层）、第一原理（第3层）。当第一原理推理显示传统智慧错误时，代理命名"顿悟时刻"并记录它。参见 `ETHOS.md` 获取完整的构建者哲学。

### 为什么提交，不在运行时生成？

三个原因：

1. **Claude 在技能加载时读取 SKILL.md。** 当用户调用 `/browse` 时没有构建步骤。文件必须已经存在且正确。
2. **CI 可以验证新鲜度。** `gen:skill-docs --dry-run` + `git diff --exit-code` 在合并之前捕获过时的文档。
3. **Git blame 起作用。** 你可以看到命令是在哪个提交中何时添加的。

### 模板测试等级

| 等级 | 什么 | 成本 | 速度 |
|------|------|------|-------|
| 1 — 静态验证 | 解析 SKILL.md 中的每个 `$B` 命令，针对注册表验证 | 免费 | <2 秒 |
| 2 — E2E via `claude -p` | 生成真实 Claude 会话，运行每个技能，检查错误 | ~$3.85 | ~20min |
| 3 — LLM 作为评审 | Sonnet 对清晰度/完整性/可操作性评分文档 | ~$0.15 | ~30s |

等级 1 运行在每个 `bun test` 上。等级 2+3 受限于 `EVALS=1`。思路：免费捕获 95% 的问题，仅在判断时使用 LLM。

## 命令分派

命令按副作用分类：

- **READ**（text、html、links、console、cookie 等）：无改变。可重试。返回页面状态。
- **WRITE**（goto、click、fill、press 等）：改变页面状态。非幂等。
- **META**（snapshot、screenshot、tab、chain 等）：不适合简单读/写的服务器级操作。

这不只是组织。服务器用于分派：

```typescript
if (READ_COMMANDS.has(cmd))  → handleReadCommand(cmd, args, bm)
if (WRITE_COMMANDS.has(cmd)) → handleWriteCommand(cmd, args, bm)
if (META_COMMANDS.has(cmd))  → handleMetaCommand(cmd, args, bm, shutdown)
```

`help` 命令返回所有三组，以便代理可以自我发现可用命令。

## 错误哲学

错误是为 AI 代理，不是为人类。每个错误消息都必须是可操作的：

- "元素未找到" → "元素未找到或不可交互。运行 `snapshot -i` 查看可用元素。"
- "选择器匹配多个元素" → "选择器匹配多个元素。请从 `snapshot` 使用 @ref。"
- 超时 → "导航 30 秒后超时。页面可能很慢或 URL 可能错误。"

Playwright 的原生错误通过 `wrapError()` 重写以剥离内部堆栈跟踪并添加指导。代理应该能读错误并知道接下来该做什么，无需人类干预。

### 崩溃恢复

服务器不尝试自我修复。如果 Chromium 崩溃（`browser.on('disconnected')`），服务器立即退出。CLI 在下一个命令上检测死掉的守护进程并自动重启。这尝试重新连接到半死的浏览进程更简单可靠。

## E2E 测试基础设施

### 会话运行器（`test/helpers/session-runner.ts`）

E2E 测试将 `claude -p` 作为完全独立的子进程生成 — 不是通过 Agent SDK，它无法嵌套在 Claude Code 会话内。运行器：

1. 将提示写入临时文件（避免 shell 转义问题）
2. 生成 `sh -c 'cat prompt | claude -p --output-format stream-json --verbose'`
3. 从 stdout 流式传输 NDJSON 进行实时进度
4. 与可配置超时竞赛
5. 解析完整 NDJSON 转录为结构化结果

`parseNDJSON()` 函数是纯的 — 无 I/O，无副作用 — 使其可独立测试。

### 可观测性数据流

```
  skill-e2e-*.test.ts
        │
        │ generates runId, passes testName + runId to each call
        │
  ┌─────┼──────────────────────────────┐
  │     │                              │
  │  runSkillTest()              evalCollector
  │  (session-runner.ts)         (eval-store.ts)
  │     │                              │
  │  per tool call:              per addTest():
  │  ┌──┼──────────┐              savePartial()
  │  │  │          │                   │
  │  ▼  ▼          ▼                   ▼
  │ [HB] [PL]    [NJ]          _partial-e2e.json
  │  │    │        │             (atomic overwrite)
  │  │    │        │
  │  ▼    ▼        ▼
  │ e2e-  prog-  {name}
  │ live  ress   .ndjson
  │ .json .log
  │
  │  on failure:
  │  {name}-failure.json
  │
  │  ALL files in ~/.gstack-dev/
  │  Run dir: e2e-runs/{runId}/
  │
  │         eval-watch.ts
  │              │
  │        ┌─────┴─────┐
  │     read HB     read partial
  │        └─────┬─────┘
  │              ▼
  │        render dashboard
  │        (stale >10min? warn)
```

**分工：** session-runner 拥有心跳（当前测试状态），eval-store 拥有部分结果（已完成的测试状态）。观察器同时读取两者。两个组件都不知道另一个 — 它们仅通过文件系统共享数据。

**都非致命：** 所有的可观测性 I/O 都包装在 try/catch 中。写入失败从不会导致测试失败。测试本身是可观察性尽力的真实来源。

**机器可读诊断：** 每个测试结果包含 `exit_reason`（success、timeout、error_max_turns、error_api、exit_code_N）、`timeout_at_turn` 和 `last_tool_call`。这使 `jq` 查询成为可能：
```bash
jq '.tests[] | select(.exit_reason == "timeout") | .last_tool_call' ~/.gstack-dev/evals/_partial-e2e.json
```

### 评估持久化（`test/helpers/eval-store.ts`）

`EvalCollector` 累积测试结果，并以两种方式写入：

1. **增量式：** `savePartial()` 在每个测试后写入 `_partial-e2e.json`（原子性：写 `.tmp`，`fs.renameSync`）。在 kill 后存活。
2. **最终：** `finalize()` 写入带时间戳的评估文件（如 `e2e-20260314-143022.json`）。部分文件永远不会被清理 — 它与最终文件一起保留用于可观测性。

`eval:compare` 比较两个评估运行。`eval:summary` 跨所有 `~/.gstack-dev/evals/` 运行聚合统计。

### 测试等级

| 等级 | 什么 | 成本 | 速度 |
|------|------|------|-------|
| 1 — 静态验证 | 解析 `$B` 命令、针对注册表验证、可观测性单元测试 | 免费 | <5 秒 |
| 2 — E2E via `claude -p` | 生成真实 Claude 会话、运行每个技能、扫描错误 | ~$3.85 | ~20min |
| 3 — LLM 作为评审 | Sonnet 对清晰度/完整性/可操作性评分文档 | ~$0.15 | ~30 秒 |

等级 1 运行在每个 `bun test` 上。等级 2+3 受限于 `EVALS=1`。思路：免费捕获 95% 的问题，仅在判断和集成测试时使用 LLM。

## 这里故意没有的东西

- **没有 WebSocket 流。** HTTP 请求/响应更简单、可调试、足够快。流式传输会增加复杂性收益微乎其微。
- **没有 MCP 协议。** MCP 在每个请求上增加 JSON schema 开销并需要持久连接。纯 HTTP + 纯文本输出对 token 更轻，更容易调试。
- **没有多用户支持。** 每工作区一台服务器，一个用户。token 认证是纵深防御，不是多租户。
- **没有 Windows/Linux cookie 解密。** macOS Keychain 是唯一支持的凭证存储。Linux（GNOME Keyring/kwallet）和 Windows（DPAPI）在架构上是可能的但未实现。
- **没有 iframe 自动发现。** `$B frame` 支持跨帧交互（CSS 选择器、@ref、`--name`、`--url` 匹配），但 ref 系统不会在 `snapshot` 期间自动爬行 iframe。你必须先显式地进入帧上下文。
