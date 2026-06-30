# Sidebar 流程

GStack 浏览器侧边栏实际的工作原理。在修改
`sidepanel.js`、`background.js`、`content.js`、`terminal-agent.ts` 或
sidebar 相关的服务端端点之前，请先阅读本文。

侧边栏只有一个主表面 —— **Terminal** 窗格，一个可交互的
`claude` PTY。Activity / Refs / Inspector 作为调试覆盖层在脚部的 `debug` 切换后面存活。聊天队列路径（一次性 `claude -p`，
sidebar-agent.ts）在 PTY 证明可行后已被移除 —— Terminal 窗格
严格更强大。

## 组件

```
┌─────────────────┐     ┌──────────────┐     ┌──────────────────┐
│  sidepanel.js + │────▶│  server.ts   │────▶│terminal-agent.ts │
│  -terminal.js   │     │  (compiled)  │     │  (non-compiled)  │
│  (xterm.js)     │     │              │     │  PTY listener    │
└─────────────────┘     └──────────────┘     └──────────────────┘
        ▲                       │                      │
        │  ws://127.0.0.1:<termPort>/ws (Sec-WebSocket-Protocol auth)
        └───────────────────────┼──────────────────────▶│ Bun.spawn(claude)
                                │                      │  terminal: {data}
                                │                      ▼
                                │              ┌──────────────────┐
                                │              │  claude PTY      │
                                │              └──────────────────┘
            POST /pty-session   │
            (Bearer AUTH_TOKEN) │
                                ▼
                       ┌──────────────────┐
                       │ pty-session-     │
                       │ cookie.ts        │
                       │ (in-memory token │
                       │  registry)       │
                       └──────────────────┘
                                │
                                │ POST /internal/grant (loopback)
                                ▼
                       ┌──────────────────┐
                       │  validTokens Set │
                       │  in agent memory │
                       └──────────────────┘
```

编译后的 browse 服务器不能 `posix_spawn` 外部可执行文件 ——
`terminal-agent.ts` 作为单独的未编译 `bun run` 进程运行，并
拥有 `claude` 子进程。

## 启动 + 第一次击键时间线

```
T+0ms     CLI 运行 `$B connect`
            ├── Server 启动（编译后）
            └── 生成 terminal-agent.ts 通过 `bun run`

T+500ms   terminal-agent.ts 引导
            ├── Bun.serve on 127.0.0.1:0（随机端口）
            ├── 写入 <stateDir>/terminal-port（服务器读取它用于 /health）
            ├── 写入 <stateDir>/terminal-internal-token（loopback 握手）
            └── 探测 claude → 写入 claude-available.json

T+1-3s    扩展加载，侧边栏打开
            ├── sidepanel-terminal.js：setState(IDLE)，显示 "Starting Claude Code..."
            └── tryAutoConnect() 轮询直到 window.gstackServerPort + token 被设置

T+ready   tryAutoConnect 调用 connect()
            ├── POST /pty-session（Authorization: Bearer ***）
            │   └── server 铸造 session token，推送到 agent 通过 loopback
            │   └── 响应 {terminalPort, ptySessionToken}
            ├── GET /claude-available（preflight）
            ├── new WebSocket(`ws://127.0.0.1:<terminalPort>/ws`,
            │                 [`gstack-pty.<token>`])
            │   └── Browser 发送 Sec-WebSocket-Protocol + Origin
            │   └── Agent 在升级前验证 Origin AND token
            │   └── Agent 回传协议（REQUIRED —— browser
            │       没有它就关闭连接）
            ├── 打开后：send {type:"resize"} 然后一个单个 \n 字节
            └── Agent 消息处理器看到字节 → spawnClaude()
```

## 身份验证：WebSocket 不能发送 Authorization 头部

浏览器 WebSocket 客户端不能设置 `Authorization`。它们可以设置
`Sec-WebSocket-Protocol` 通过第二个参数 `new WebSocket(url,
protocols)`。我们利用这一点：

1. `POST /pty-session`（auth：Bearer AUTH_TOKEN）→ server 铸造一个
   短期 session token，通过 loopback 推送到 agent，
   在 JSON 主体中返回。
2. 扩展调用 `new WebSocket(url, ['gstack-pty.<token>'])`。
3. Agent 读取 `Sec-WebSocket-Protocol`，剥离 `gstack-pty.`，验证
   对抗 `validTokens`，回传协议。回传是强制性的 ——
   没有它在 Chromium 收到升级响应时关闭连接。

对于非浏览器调用者（curl、集成测试）还会返回一个 `Set-Cookie: gstack_pty=...` header。cookie 路径是 v1 原始
设计但 `SameSite=Strict` cookie 不能从 chrome-extension origin 的
server.ts:34567 → agent:<random> 的跨端口跳转中存活。
协议-token 路径是浏览器实际使用的。

### 双-token 模型

| Token | 存活于 | 用于 | 生存期 |
|-------|--------|------|--------|
| `AUTH_TOKEN` | `<stateDir>/browse.json`；server.ts 内存中 | `/pty-session` POST（mint cookie + token） | server 生存期 |
| `gstack-pty.<...>`（Sec-WebSocket-Protocol） | 仅浏览器内存中；agent `validTokens` Set | `/ws` 升级认证 | 30 分钟，WS 关闭自动撤销 |
| `INTERNAL_TOKEN` | `<stateDir>/terminal-internal-token`；agent 内存中 | server → agent loopback `/internal/grant` | agent 生存期 |

`AUTH_TOKEN` **永远**不直接有效于 `/ws`。Session token
**永远**不有效于 `/pty-session` 或 `/command`。严格分离
防止 SSE 或 page-content token 泄漏升级到 shell
访问。

## 威胁模型

Terminal 窗格刻意绕过提示注入安全堆栈 —— 用户直接输入到 claude，没有不受信任的页面内容在循环中。信任来源是键盘，与任何本地终端相同。

该信任假设依赖于三个传输保证：

1. **仅本地监听器。** `terminal-agent.ts` 仅绑定 `127.0.0.1`。
   双-监听器隧道表面（server.ts `TUNNEL_PATHS`）不
   包括 `/pty-session` 或 `/terminal/*`，因此隧道通过默认拒绝
   返回 404。
2. **Origin 门。** `/ws` 升级需要
   `Origin: chrome-extension://<id>`。本地网页不能对 shell 发起
   跨站点 WebSocket 劫持因为其 Origin 是常规的 `http(s)://...`。
3. **Session token 认证。** 仅由经过认证的 `/pty-session` POST 铸造，
   限定于单个 WS，关闭时自动撤销。

三个中丢弃任何一个，整个标签页都不安全。

## 生命周期

- **热自动连接。** 侧边栏打开 → tryAutoConnect 轮询引导全局变量并连接一旦它们被设置。无需击键。
- **每个 WS 一个 PTY。** 关闭 WebSocket SIGINTs claude，3 秒后
  SIGKILL。Session token 被撤销所以被盗的 token
  不能重放。
- **关闭时不自动重新连接。** 用户看到"会话结束，点击
  开始新会话。"自动重新连接会在每次重新加载时燃烧一个新的
  claude 会话。v1.1 可能添加基于 tab/session
  id 的会话恢复（参见 TODOS）。
- **随时手动重启。** `↻ Restart` 按钮始终存在于总是可见的终端工具栏中 —— 在会话中期工作，不仅仅是从
  ENDED 状态。

## 快速操作工具栏

三个浏览器操作按钮位于 Terminal 窗格顶部 Restart 按钮旁：

| 按钮 | 行为 |
|------|------|
| 🧹 Cleanup | `window.gstackInjectToTerminal(prompt)` —— 将"移除 ads/banners"指令管道到运行中的 PTY。终端中的 claude 看到它并执行。 |
| 📸 Screenshot | `POST /command screenshot` —— 直接的 browse-server 调用，无 PTY 介入。 |
| 🍪 Cookies | 导航到 `/cookie-picker` 页面。 |

Inspector 的"Send to Code"按钮使用相同的 `gstackInjectToTerminal`
路径将 CSS inspector 数据转发到 claude 中。

## 调试表面（Activity / Refs / Inspector）

在脚部的 `debug` 切换后面。SSE 驱动，独立于
Terminal 窗格：

- **Activity** —— 通过 `/activity/stream` SSE 流传输每个 browse 命令。
- **Refs** —— REST：`GET /refs` —— 当前页面的 `@ref` 元素标签。
- **Inspector** —— 基于 CDP 的元素选择器；在 `/inspector/events` 上 SSE。

调试条关闭后，Terminal 窗格重新可见。
xterm.js 在其容器从 `display:none` 翻转到 `display:flex` 时不自动重绘，所以 sidepanel-terminal.js 在 `#tab-terminal` 的 class 属性上运行 `MutationObserver` 并在 `.active` 返回时强制 fit + refresh。

## 文件

| 组件 | 文件 | 运行于 |
|------|------|---------|
| Sidebar UI shell | `extension/sidepanel.html` + `sidepanel.js` + `sidepanel.css` | Chrome side panel |
| Terminal UI | `extension/sidepanel-terminal.js` + `extension/lib/xterm.js` | Chrome side panel |
| Service worker | `extension/background.js` | Chrome background |
| Content script | `extension/content.js` | 页面上下文 |
| HTTP server | `browse/src/server.ts` | Bun（编译二进制） |
| PTY agent | `browse/src/terminal-agent.ts` | Bun（未编译） |
| PTY token store | `browse/src/pty-session-cookie.ts` | Bun（编译后，在 server.ts 中） |
| CLI entry | `browse/src/cli.ts` | Bun（编译二进制） |
| State file | `<stateDir>/browse.json` | 文件系统 |
| Terminal port | `<stateDir>/terminal-port` | 文件系统 |
| Internal token | `<stateDir>/terminal-internal-token` | 文件系统 |
| Claude probe | `<stateDir>/claude-available.json` | 文件系统 |
| Active tab | `<stateDir>/active-tab.json` | 文件系统（claude 读取） |
