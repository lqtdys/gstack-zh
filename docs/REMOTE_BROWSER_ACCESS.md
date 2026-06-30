# 远程浏览器访问 — 如何配对 GStack 浏览器

GStack 浏览器服务器可以与任何能发出 HTTP 请求的 AI 代理共享。
代理获得对真实 Chromium 浏览器的作用域访问：导航页面、读取内容、
点击元素、填写表单、截图。每个代理获得自己的标签页。

本文档是远程代理的参考。快速开始说明由 `$B pair-agent` 生成，其中包含实际的加密凭证。

## 架构

```
你的机器                                  远程代理
─────────────                         ────────────
GStack 浏览器服务器                       任意 AI 代理
  ├── Chromium (Playwright)           (OpenClaw、Hermes、Codex 等)
  ├── 本地监听器 127.0.0.1:LOCAL         │
  │    (bootstrap、CLI、sidebar、cookies)  │
  ├── 隧道监听器 127.0.0.1:TUNNEL ◄───────┤
  │    (pair-agent 专用：/connect、/command、│
  │     /sidebar-chat — 锁定白名单)        │
  ├── ngrok 隧道（仅转发隧道端口）             │
  │     https://xxx.ngrok.dev ─────────────────┘
  └── Token Registry
        ├── Root token（仅本地监听器）
        ├── Setup keys（5 分钟，一次性）
        ├── Session tokens（24h，受限作用域）
        └── SSE cookies（30 分钟，流作用域）
```

### 双监听器架构（v1.6.0.0）

守护进程绑定两个 HTTP 套接字。**本地监听器**仅在 127.0.0.1 上提供完整的命令接口，且从不转发。**隧道监听器**在 `/tunnel/start` 上惰性绑定（在 `/tunnel/stop` 时拆除），具有锁定的路径白名单。ngrok 仅转发隧道端口。

发现你的 ngrok URL 的调用者无法访问 `/health`、`/cookie-picker`、`/inspector/*` 或 `/welcome` —— 这些路径在该 TCP 套接字上不存在。通过隧道发送的 root token 获得 403。隧道监听器仅接受 `/connect`、`/command`（带受限 token + 26 个命令的浏览器白名单）和 `/sidebar-chat`。

参见 [ARCHITECTURE.md](../ARCHITECTURE.md#dual-listener-tunnel-architecture-v1600) 获取完整的端点表。

## 连接流程

1. **用户运行** `$B pair-agent`（或 Claude Code 中的 `/pair-agent`）
2. **服务器创建一个**一次性 setup key（5 分钟后过期）
3. **用户复制**指令块到另一个代理的聊天中
4. **远程代理运行** 带 setup key 的 `POST /connect`
5. **服务器返回**一个受限的 session token（默认 24h）
6. **远程代理通过** 带 `newtab` 的 `POST /command` 创建自己的标签页
7. **远程代理使用** 带其 session token + tabId 的 `POST /command` 进行浏览

## API 参考

### 身份验证

所有命令端点需要 Bearer token：

```
Authorization: Bearer ***
```

`/connect` 无需身份验证（速率限制）—— 它是远程代理交换 setup key 以获取受限 session token 的方式。`/health` 在本地监听器上无需身份验证（bootstrap），但在隧道监听器上不存在（404）。

SSE 端点（`/activity/stream`、`/inspector/events`）接受 Bearer token 或 HttpOnly `gstack_sse` cookie（通过 `POST /sse-session` 生成，30 分钟 TTL，仅流作用域 —— 不能用于 `/command`）。自 v1.6.0.0 起，不再接受 `?token=<ROOT>` 查询字符串身份验证。

### 端点

#### POST /connect
用 setup key 交换 session token。无需身份验证。限制为 300/分钟（洪水防御 —— setup key 是 24 个随机字节，无法暴力破解）。

```json
Request:  {"setup_key": "gsk_setup_..."}
Response: {"token": "gsk_sess_...", "expires": "ISO8601", "scopes": ["read","write"], "agent": "agent-name"}
```

#### POST /command
发送浏览器命令。需要 Bearer 身份验证。

```json
Request:  {"command": "goto", "args": ["https://example.com"], "tabId": 1}
Response: (纯文本命令结果)
```

#### GET /health
服务器状态。无需身份验证。返回状态、标签页、模式、运行时间。

### 命令

#### 导航
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `goto` | `["URL"]` | 导航到 URL |
| `back` | `[]` | 后退 |
| `forward` | `[]` | 前进 |
| `reload` | `[]` | 重新加载页面 |

#### 读取内容
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `snapshot` | `["-i"]` | 带 @ref 标签的交互式快照（最有用） |
| `text` | `[]` | 完整页面文本 |
| `html` | `["selector?"]` | 元素或完整页面的 HTML |
| `links` | `[]` | 页面上所有链接 |
| `screenshot` | `["/tmp/s.png"]` | 截图 |
| `url` | `[]` | 当前 URL |

#### 交互
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `click` | `["@e3"]` | 点击元素（使用快照中的 @ref） |
| `fill` | `["@e5", "text"]` | 填写表单字段 |
| `select` | `["@e7", "option"]` | 选择下拉值 |
| `type` | `["text"]` | 键入文本（键盘） |
| `press` | `["Enter"]` | 按键 |
| `scroll` | `["down"]` | 滚动页面 |

#### 标签页
| 命令 | 参数 | 描述 |
|---------|------|-------------|
| `newtab` | `["URL?"]` | 创建新标签页（写入前必需） |
| `tabs` | `[]` | 列出所有标签页 |
| `closetab` | `["id?"]` | 关闭标签页 |

## 快照 → @ref 模式

这是最强大的浏览模式。不需要编写 CSS 选择器：

1. 运行 `snapshot -i` 获取带标签元素的交互式快照
2. 快照返回如下文本：
   ```
   [页面标题]
   @e1 [链接] "首页"
   @e2 [按钮] "登录"
   @e3 [输入] "搜索..."
   ```
3. 直接在命令中使用 `@e` 引用：`click @e2`、`fill @e3 "查询内容"`

这就是快照系统的工作原理，比猜测 CSS 选择器可靠得多。始终先 `snapshot -i`，然后使用引用。

## 作用域

| 作用域 | 允许的操作 |
|-------|---------------|
| `read` | snapshot、text、html、links、screenshot、url、tabs、console 等。 |
| `write` | goto、click、fill、scroll、newtab、closetab 等。 |
| `admin` | eval、js、cookies、storage、cookie-import、useragent 等。 |
| `meta` | tab、diff、frame、responsive、watch |

默认 token 获得 `read` + `write`。管理操作需要配对时使用 `--admin` 标志。

## 标签页隔离

每个代理拥有它创建的标签页。规则：
- **读取：** 任何代理都可以读取任何标签页（snapshot、text、screenshot）
- **写入：** 只有标签页所有者可以写入（click、fill、goto 等）
- **未拥有的标签页：** 预先存在的标签页仅 root 可写入
- **第一步：** 在尝试交互前始终先 `newtab`

## 错误代码

| 代码 | 含义 | 处理方法 |
|------|---------|------------|
| 401 | token 无效、过期或被撤销 | 请用户再次运行 /pair-agent |
| 403 | 命令不在作用域内，或标签页不属于你 | 使用 newtab，或请求 --admin |
| 429 | 超过速率限制（>10 req/s） | 等待 Retry-After 标头 |

## 安全模型

- **物理端口分离。** 本地监听器和隧道监听器是单独的 TCP 套接字。ngrok 仅转发隧道端口。隧道调用者完全无法访问引导端点（404，错误端口）。
- **隧道命令白名单。** 隧道上的 `/command` 仅接受 26 个浏览器驱动命令（goto、click、fill、snapshot、text、newtab、tabs、back、forward、reload、closetab 等）。服务器管理命令（tunnel、pair、token、useragent、js）在隧道上被拒绝。
- **root token 在隧道上被阻止。** 在隧道监听器上发送 root token 的请求返回 403 并附带配对提示。只有受限的 session token 可在隧道上工作。
- **Setup key** 5 分钟过期，只能使用一次。
- **Session token** 24 小时过期（可配置）。
- Root token 永远不会出现在指令块或连接字符串中。
- **管理作用域**（JS 执行、Cookie 访问）默认被拒绝。
- Token 可以立即撤销：`$B tunnel revoke agent-name`
- **SSE 身份验证** 使用 30 分钟的 HttpOnly SameSite=Strict cookie，仅流作用域（绝不能用于 `/command`）。
- **路径遍历防护** 在 `/welcome` 上 —— `GSTACK_SLUG` 必须匹配 `^[a-z0-9_-]+$`，否则回退到内置模板。
- **SSRF 防护** 在 `goto`、`download` 和 scrape 路径上 —— 验证 URL 目标对私有地址范围的黑名单。
- **隧道表面拒绝日志记录。** 隧道监听器上的每次拒绝（`path_not_on_tunnel`、`root_token_on_tunnel`、`missing_scoped_token`、`disallowed_command:*`）都会附加到 `~/.gstack/security/attempts.jsonl`，附带时间戳、来源 IP、路径、方法。速率限制为 60 次写入/分钟。
- 所有代理活动都有归属记录（clientId）。

**已知非目标（追踪为 #1136）：** 在 Windows 上，cookie-import-browser 路径会用 `--remote-debugging-port=<随机>` 启动 Chrome。通过 App-Bound Encryption v20，同一用户的本地进程可以连接到该端口并窃取解密的 v20 cookie —— 相对于直接读取 SQLite DB 的提权路径。修复方向是用 `--remote-debugging-pipe` 替代 TCP。

## 同一机器捷径

如果两个代理在同一台机器上，跳过复制粘贴：

```bash
$B pair-agent --local openclaw    # 写入 ~/.openclaw/skills/gstack/browse-remote.json
$B pair-agent --local codex       # 写入 ~/.codex/skills/gstack/browse-remote.json
$B pair-agent --local cursor      # 写入 ~/.cursor/skills/gstack/browse-remote.json
```

无需隧道。直接使用 localhost。

## ngrok 隧道设置

适用于不同机器上的远程代理：

1. 在 [ngrok.com](https://ngrok.com) 注册（免费层可用）
2. 从仪表板复制你的 auth token
3. 保存：`echo 'NGROK_AUTHTOKEN=your_token' > ~/.gstack/ngrok.env`
4. 可选，申请稳定域名：`echo 'NGROK_DOMAIN=your-name.ngrok-free.dev' >> ~/.gstack/ngrok.env`
5. 使用隧道启动：`BROWSE_TUNNEL=1 $B restart`
6. 运行 `$B pair-agent` —— 它会自动使用隧道 URL
