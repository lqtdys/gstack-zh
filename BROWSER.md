# Browser — 完整参考

gstack 的浏览器功能面，一文打尽。无头 Chromium 守护进程，约 70+
命令，基于 ref 的元素选择，可固化的浏览器技能，带 Chrome 侧边栏的真实浏览器模式，
侧边栏内嵌 Claude PTY，ngrok 对等代理流程，以及分层提示注入防御——
这一切都封装在一个编译后的 CLI 中，向 stdout 输出纯文本。每次调用约 100-200ms。零上下文 token 开销。

如果你在过去一两个版本中使用过 gstack，生产力循环才是新的头条：
`/scrape <意图>` 驱动页面一次，`/skillify` 将流程固化为确定性的 Playwright 脚本，
下一次对相同意图的 `/scrape` 调用在 ~200ms 内完成，而非约 30 秒的代理重复探索。

---

## 快速开始

```bash
# 一次性：构建二进制文件（browse/dist/browse，约 58MB）
bun install && bun run build

# 配置一次 $B，以后不用再管
B=./browse/dist/browse           # 或 ~/.claude/skills/gstack/browse/dist/browse

# 驱动页面
$B goto https://news.ycombinator.com
$B snapshot -i                   # @e refs，后续可用于点击/填充/检查
$B click @e30                    # 点击快照中的 ref 30
$B text                          # 获取干净的页面文本
$B screenshot /tmp/hn.png

# 固化重复流程
/scrape latest hacker news stories
/skillify                        # 写入 ~/.gstack/browser-skills/hn-front/...
/scrape hacker news front page   # 第二次调用：~200ms，通过已固化技能运行

# 实时观看 Claude 工作
$B connect                       # 带 Side Panel 扩展的 headed 版 Chromium
```

---

## 目录

1. [它是什么](#它是什么)
2. [生产力循环 — `/scrape` + `/skillify`](#生产力循环)
3. [架构](#架构)
4. [命令参考](#命令参考)
5. [快照系统 + 基于 ref 的选择](#快照系统)
6. [浏览器技能运行时](#浏览器技能运行时)
7. [领域技能（每站点代理笔记）](#领域技能)
8. [真实浏览器模式（`$B connect`）](#真实浏览器模式) — 包括 [`--headed` + `--proxy` + `--navigate` (v1.28.0.0)](#headed-模式--代理--浏览器原生下载-v12800)
9. [侧边栏 + 侧边代理](#侧边栏--侧边代理)
10. [对等代理 — 通过 ngrok 隧道的远程代理](#对等代理)
11. [认证 + 令牌](#认证)
12. [提示注入安全栈（L1–L6）](#安全栈)
13. [截图、PDF、视觉检查](#截图pdf视觉)
14. [本地 HTML — `goto file://` vs `load-html`](#本地-html)
15. [批处理端点](#批处理端点)
16. [控制台、网络、对话框捕获](#捕获)
17. [JS 执行 — `js` + `eval`](#js-执行)
18. [标签页、框架、状态、监视、收件箱](#标签页框架状态)
19. [CDP 应急出口 + CSS 检查器](#cdp)
20. [性能 + 规模](#性能)
21. [多工作区隔离](#多工作区)
22. [环境变量](#环境变量)
23. [源码映射](#源码映射)
24. [开发 + 测试](#开发)
25. [交叉引用](#交叉引用)
26. [致谢](#致谢)

---

## 它是什么

一个编译后的 CLI 二进制文件，通过 HTTP 与持久化的本地 Chromium 守护进程通信。
该 CLI 是一个薄客户端——它读取状态文件，发送命令，将响应输出到 stdout。
守护进程通过 [Playwright](https://playwright.dev/) 完成实际工作。

早期作为 Chrome MCP 服务器的一切现在都通过纯 stdout 完成。
没有 JSON-schema 框架，没有协议协商，没有持久 WebSocket——Claude 的 Bash 工具已经存在，我们直接用就行。

三种逐步升级的模式：

- **无头模式**（默认）。守护进程运行 Chromium，没有可见窗口。最快、最便宜，
  `/qa`、`/design-review`、`/benchmark` 等技能默认使用此模式。
- **通过 `$B connect` 的 headed 模式**。同一守护进程，但 Chromium 可见（重新品牌化为
  "GStack Browser"），并自动加载 Side Panel 扩展。你实时观看每个命令执行。
- **通过隧道的对等代理**。守护进程绑定第二个 ngrok 转发的监听器。远程代理
  （Codex、OpenClaw、Hermes，任何会说 HTTP 的）通过 26 命令白名单和
  作用域、一次性令牌驱动你的本地浏览器。

---

## 生产力循环

v1.19.0.0 的发布头条。两个 gstack 技能封装了浏览器技能运行时，
这样第二次你让 Claude 抓取页面时，它在 ~200ms 内完成。

### `/scrape <意图>`

拉取页面数据的单一入口。内部有三条路径：

1. **匹配路径（~200ms）** — 代理运行 `$B skill list`，将意图与每个技能的 `triggers:` 数组
   + `description` + `host` 进行语义匹配，如果存在高置信度匹配则运行 `$B skill run <name>`。
2. **原型路径（~30s）** — 无匹配，代理使用 `$B goto`、`$B text`、`$B html`、`$B links`
   等驱动页面，返回 JSON，并附加一行"请说 `/skillify`"的建议。
3. **变更意图拒绝** — 诸如 *submit*、*click*、*fill* 之类的动词路由到 `/automate`
   （Phase 2b，`TODOS.md` 中的 P0）。`/scrape` 按契约是只读的。

### `/skillify`

将最近一次成功的 `/scrape` 原型固化为磁盘上的永久浏览器技能。十一步，三个锁定契约：

- **D1 — 来源守卫。** 回溯 ≤10 个代理轮次，寻找一个边界清晰的 `/scrape` 结果。
  如果冷启动则拒绝并给出特定消息。不从聊天片段静默合成。
- **D2 — 合成输入切片。** 仅提取生成用户接受的 JSON 的最终尝试 `$B` 调用，
  加上用户的意图字符串。丢弃失败的丢弃器、丢弃聊天、丢弃早于本次会话的内容。
- **D3 — 原子写入。** 将所有内容暂存到 `~/.gstack/.tmp/skillify-<spawnId>/`，
  针对临时目录运行 `$B skill test`，仅在测试通过 + 用户批准后才重命名到最终层路径。
  测试失败或用户拒绝：`rm -rf` 整个临时目录。列表中永远不会出现半写的技能。

变更流程的兄弟技能 `/automate` 在 `TODOS.md` 中拆分为 P0 并在下一个分支发布——
同样的 skillify 机制，在未固化运行时每个变更步骤都有确认门控。

完整设计 + 决策记录见 [`docs/designs/BROWSER_SKILLS_V1.md`](docs/designs/BROWSER_SKILLS_V1.md)。

---

## 架构

```
┌─────────────────────────────────────────────────────────────────┐
│  Claude Code                                                    │
│                                                                 │
│  $B goto https://staging.myapp.com                              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────┐    HTTP POST     ┌──────────────┐                 │
│  │ browse   │ ──────────────── │ Bun HTTP     │                 │
│  │ CLI      │  127.0.0.1:rand  │ daemon       │                 │
│  │          │  Bearer token    │              │                 │
│  │ compiled │ ◄──────────────  │  Playwright  │──── Chromium    │
│  │ binary   │  plain text      │  API calls   │    (headless    │
│  └──────────┘                  └──────────────┘     or headed)  │
│   ~1ms startup                  persistent daemon               │
│                                 auto-starts on first call       │
│                                 auto-stops after 30 min idle    │
└─────────────────────────────────────────────────────────────────┘
```

### 守护进程生命周期

1. **首次调用。** CLI 检查 `<project>/.gstack/browse.json` 寻找运行中的服务器。
   未找到——在后台生成 `bun run browse/src/server.ts`。守护进程通过 Playwright 启动
   无头 Chromium，选择随机端口（10000–60000），生成 bearer 令牌，写入状态文件
   （chmod 600），开始接受请求。约 3 秒。
2. **后续调用。** CLI 读取状态文件，发送带 bearer 令牌的 HTTP POST，输出响应。
   往返约 100-200ms。
3. **空闲关闭。** 30 分钟无命令后，守护进程关闭并清理状态文件。下次调用重新启动。
4. **崩溃恢复。** 如果 Chromium 崩溃，守护进程立即退出——不要自愈，不要隐藏故障。
   CLI 在下次调用时检测到死掉的守护进程并启动新的。

### 多工作区隔离

每个项目根目录（通过 `git rev-parse --show-toplevel` 检测）获得自己的守护进程、端口、
状态文件、Cookies 和日志。无工作区间碰撞。状态文件位于 `<project>/.gstack/browse.json`。

| 工作区 | 状态文件 | 端口 |
|--------|-----------|------|
| `/code/project-a` | `/code/project-a/.gstack/browse.json` | 随机 (10000–60000) |
| `/code/project-b` | `/code/project-b/.gstack/browse.json` | 随机 (10000–60000) |

---

## 命令参考

约 70 条命令，涵盖读写和元操作。选择器接受 CSS、来自 `snapshot` 的 `@e` refs
或来自 `snapshot -C` 的 `@c` refs。完整表格：

### 读取

| 命令 | 描述 |
|------|------|
| `text [sel]` | 干净页面文本（或限定于选择器范围） |
| `html [sel]` | innerHTML，或完整页面 HTML（无选择器时） |
| `links` | 所有链接，格式为 `text → href` |
| `forms` | 表单字段 JSON |
| `accessibility` | 完整 ARIA 树 |
| `media [--images\|--videos\|--audio] [sel]` | 带 URL、尺寸、类型的媒体元素 |
| `data [--jsonld\|--og\|--meta\|--twitter]` | 结构化数据：JSON-LD、OG、Twitter Cards、meta 标签 |

### 检查

| 命令 | 描述 |
|------|------|
| `js <expr> [--out <file>] [--raw]` | 在页面上下文中运行内联 JS 表达式，返回字符串。`--out <file>` 时结果写入磁盘而非返回（`data:*;base64,...` 结果解码为原始字节，除非 `--raw`）。`--out` 使调用变为 WRITE（需要 `write` 作用域，隧道上永不允许）。 |
| `eval <file> [--out <file>] [--raw]` | 从文件运行 JS（路径在 /tmp 或 cwd 下；与 `js` 相同的沙箱）。`--out`/`--raw` 与 `js` 一致。 |
| `css <sel> <prop>` | 计算的 CSS 值 |
| `attrs <sel\|@ref>` | 元素属性 JSON |
| `is <prop> <sel\|@ref>` | 状态检查：visible、hidden、enabled、disabled、checked、editable、focused |
| `console [--clear\|--errors]` | 捕获的控制台消息 |
| `network [--clear]` | 捕获的网络请求 |
| `dialog [--clear]` | 捕获的对话框消息 |
| `cookies` | 所有 cookies JSON |
| `storage` / `storage set <key> <val>` | 读取 localStorage + sessionStorage；设置 localStorage |
| `perf` | 页面加载时间指标 |
| `inspect [sel] [--all] [--history]` | 通过 CDP 深度 CSS — 完整规则级联、盒模型、计算样式 |
| `ux-audit` | 用于行为分析的页面结构：站点 ID、导航、标题、文本块、交互元素 |
| `cdp <Domain.method> [json-params]` | 原始 CDP 方法派发（默认拒绝；白名单在 `cdp-allowlist.ts` 中） |

### 导航

| 命令 | 描述 |
|------|------|
| `goto <url>` | 导航到 URL（`http://`、`https://`、`file://`） |
| `load-html <file>` | 在内存中加载本地 HTML（无 `file://` URL；在视口缩放变化后保留） |
| `back`、`forward`、`reload` | 标准导航 |
| `url` | 当前页面 URL |
| `wait <sel\|--networkidle\|--load>` | 等待元素、网络空闲或页面加载（15s 超时） |

### 交互

| 命令 | 描述 |
|------|------|
| `click <sel\|@ref>` | 点击元素 |
| `fill <sel> <val>` | 填充输入框 |
| `select <sel> <val>` | 选择下拉选项（值、标签或可见文本） |
| `hover <sel>` | 悬停元素 |
| `type <text>` | 在聚焦元素中输入 |
| `press <key>` | Playwright 键盘按键（区分大小写：Enter、Tab、ArrowUp、Shift+Enter、Control+A、...） |
| `scroll [sel\|@ref]` | 滚动元素到视图，或无选择器时跳到页面底部 |
| `viewport [<WxH>] [--scale <n>]` | 设置视口大小 + 可选 `deviceScaleFactor` 1-3（视网膜截图） |
| `upload <sel> <file> [...]` | 上传文件 |
| `dialog-accept [text]` | 自动接受下一个 alert/confirm/prompt；text 用于 prompts |
| `dialog-dismiss` | 自动关闭下一个对话框 |

### 样式 + 清理

| 命令 | 描述 |
|------|------|
| `style <sel> <prop> <val>` | 修改 CSS 属性（支持撤销） |
| `style --undo [N]` | 撤销最近 N 次样式变更 |
| `cleanup [--ads\|--cookies\|--sticky\|--social\|--all]` | 移除页面杂乱 |
| `prettyscreenshot [--scroll-to <sel\|text>] [--cleanup] [--hide <sel>...] [path]` | 带可选清理、滚动、隐藏的干净截图 |

### 视觉

| 命令 | 描述 |
|------|------|
| `screenshot [--selector <css>] [--viewport] [--clip x,y,w,h] [--base64] [sel\|@ref] [path]` | 五种模式：整页、视口、元素裁剪、区域裁剪、base64 |
| `pdf [path] [--format letter\|a4\|legal] [...]` | 带完整布局的 PDF：格式、宽度/高度、边距、页眉/页脚模板、页码、--tagged 可访问性、--toc 等待 Paged.js |
| `responsive [prefix]` | 三张截图：手机 (375x812)、平板 (768x1024)、桌面 (1280x720) |
| `diff <url1> <url2>` | 两个 URL 之间的文本 diff |

### Cookies + 请求头

| 命令 | 描述 |
|------|------|
| `cookie <name>=<value>` | 设置当前页面域的 cookie |
| `cookie-import <json>` | 从 JSON 文件导入 cookies |
| `cookie-import-browser [browser] [--domain d]` | 从已安装 Chromium 浏览器导入（交互式选择器，或 `--domain` 直接导入） |
| `header <name>:<value>` | 设置自定义请求头（敏感值自动脱敏） |
| `useragent <string>` | 设置 User-Agent（触发上下文重建，使 refs 失效） |

### 标签页 + 框架

| 命令 | 描述 |
|------|------|
| `tabs` | 列出打开的标签页 |
| `tab <id>` | 切换到标签页 |
| `newtab [url] [--json]` | 打开新标签页；`--json` 返回 `{tabId, url}` 供程序化使用 |
| `closetab [id]` | 关闭标签页 |
| `tab-each <command> [args...]` | 将命令扇出到每个打开的标签页；返回 JSON |
| `frame <sel\|@ref\|--name n\|--url pattern\|main>` | 切换到 iframe 上下文（或返回主框架）；清除 refs |

### 提取

| 命令 | 描述 |
|------|------|
| `download <url\|@ref> [path] [--base64]` | 使用浏览器 cookies 下载 URL 或媒体元素 |
| `scrape <images\|videos\|media> [--selector] [--dir] [--limit]` | 批量下载页面所有媒体；写入 `manifest.json` |
| `archive [path]` | 通过 CDP 将完整页面保存为 MHTML |

### 快照

| 命令 | 描述 |
|------|------|
| `snapshot [-i] [-c] [-d N] [-s sel] [-D] [-a] [-o path] [-C]` | 带 `@e` refs 的 AX 树；`-i` 仅交互、`-c` 紧凑、`-d N` 深度、`-s` 范围、`-D` 与上次 diff、`-a` 标注截图、`-$ -C` 光标交互 `@c` refs |

### 服务器生命周期

| 命令 | 描述 |
|------|------|
| `status` | 守护进程健康 + 模式（无头 / headed / cdp） |
| `stop` | 关闭守护进程 |
| `restart` | 重启守护进程 |
| `connect` | 启动带 Side Panel 扩展的 headed 版 GStack Browser |
| `disconnect` | 关闭 headed Chrome，返回无头模式 |
| `focus [@ref]` | 将 headed Chrome 带到前台（macOS）；`@ref` 还会滚动到视图 |
| `state save\|load <name>` | 保存或加载浏览器状态（cookies + URLs） |
| `memory [--json]` | 快照 Bun 堆 + 每标签 JS 堆 + Chromium 进程树 + 有限缓冲区大小。程序化消费者用 `--json`；文本模式渲染排序前 10 标签 + "and N more" 尾部。 |

### 交接

| 命令 | 描述 |
|------|------|
| `handoff [reason]` | 打开可见 Chrome 到当前页面，供用户接管（CAPTCHA、MFA、复杂认证） |
| `resume` | 用户接管后重新快照，将控制权返还 AI |

### 元 + 链

| 命令 | 描述 |
|------|------|
| `chain`（通过 stdin 的 JSON） | 运行命令序列。将 `[["cmd","arg1",...],...]` 管道到 `$B chain`。首个错误时停止。 |
| `inbox [--clear]` | 列出侧边栏侦察收件箱中的消息 |
| `watch [stop]` | 被动观察——用户浏览时定期快照；`stop` 返回摘要 |

### 浏览器技能运行时

| 命令 | 描述 |
|------|------|
| `skill list` | 列出所有浏览器技能及解析后的层（project > global > bundled） |
| `skill show <name>` | 打印 SKILL.md |
| `skill run <name> [--arg k=v...] [--timeout=Ns]` | 用每 spawn 作用域令牌生成技能脚本 |
| `skill test <name>` | 针对捆绑 fixtures 运行技能的 `script.test.ts` |
| `skill rm <name> [--global]` | 对用户层技能做墓碑标记 |

### 领域技能

| 命令 | 描述 |
|------|------|
| `domain-skill save\|list\|show\|edit\|promote-to-global\|rollback\|rm <host?>` | 每站点代理笔记（从活动标签页派生主机名）。生命周期：隔离 → 活跃（N=3 次成功使用且无分类器标记后） → 全局（显式提升） |

别名：`setcontent`、`set-content`、`setContent` → `load-html`（在作用域检查前规范化，
因此只读作用域令牌无法使用别名运行写命令）。

---

## 快照系统浏览器的关键创新是**基于 ref 的元素选择**，构建于 Playwright 的 AX 树 API 之上。
无 DOM 变异。无注入脚本。只用 Playwright 原生 AX API。

### `@ref` 如何工作

1. `page.locator(scope).ariaSnapshot()` 返回类似 YAML 的 AX 树。
2. 快照解析器为每个元素分配 refs（`@e1`、`@e2`……）。
3. 对每个 ref，构建一个 Playwright `Locator`（使用 `getByRole` + nth-child）。
4. ref→Locator 映射存储在 `BrowserManager` 上。
5. 后续命令如 `click @e3` 查找 Locator 并调用 `locator.click()`。

### Ref 过期检测

SPA 可以在不导航的情况下改变 DOM（React router、标签切换、模态框）。
当这种情况发生时，从前一次 `snapshot` 收集的 refs 可能指向已不存在的元素。
`resolveRef()` 在任何 ref 使用前运行异步 `count()` 检查——如果元素计数为 0，
立即抛出并告诉代理重新运行 `快照`。快速失败（约 5ms），
而非等待 Playwright 的 30 秒操作超时。

### 扩展快照功能

- **`--diff`（`-D`）。** 将每个快照存储为基线。下次 `-D` 调用时，返回统一 diff 显示变化。
  用于验证操作（点击、填充等）是否真正生效。
- **`--annotate`（`-a`）。** 在每个 ref 的边界框注入临时覆盖 div，拍摄带 ref 标签可见的截图，
  然后移除覆盖。用 `-o <path>` 控制输出。
- **`--cursor-interactive`（`-C`）。** 扫描非 ARIA 交互元素（
  带 `cursor:pointer`、`onclick`、`tabindex>=0` 的 div），使用 `page.evaluate`。
  分配 `@c1`、`@c2`... refs，使用确定性的 `nth-child` CSS 选择器。
  这些是 ARIA 树遗漏但用户仍可点击的元素。

---

## 浏览器技能运行时

按任务目录，将重复的浏览器流程固化为确定性的 Playwright 脚本。复利层。

### 浏览器技能的解剖

```
browser-skills/<name>/
├── SKILL.md                        # frontmatter + 散文契约
├── script.ts                       # 确定性 Playwright-via-browse-client 逻辑
├── _lib/browse-client.ts           # 供应商副本 SDK（~3KB，与规范逐字节一致）
├── fixtures/<host>-<date>.html     # 捕获页面用于 fixture-replay 测试
└── script.test.ts                  # 针对 fixture 的解析器测试（无需守护进程）
```

捆绑的参考技能是 `browser-skills/hackernews-frontpage/`：抓取 HN 首页，
返回 30 条故事 JSON。试试：

```bash
$B skill list                            # 显示 hackernews-frontpage (bundled)
$B skill show hackernews-frontpage
$B skill run hackernews-frontpage        # ~200ms 内 30 条故事 JSON
$B skill test hackernews-frontpage       # 针对 fixture 运行 script.test.ts
```

### 三层存储

`$B skill list` 按优先级遍历所有三层；首次命中即赢。解析后的层
打印在每个技能名称旁：

| 层 | 路径 | 何时 |
|----|------|------|
| **Project** | `<project>/.gstack/browser-skills/<name>/` | 项目特定技能（提交或 gitignored） |
| **Global** | `~/.gstack/browser-skills/<name>/` | 每用户技能，所有项目 |
| **Bundled** | `<gstack-install>/browser-skills/<name>/` | 随 gstack 附带，只读 |

### 信任模型

两条正交轴——守护进程端能力 + 进程端环境——独立配置。

| 轴 | 机制 | 默认 |
|----|------|------|
| **守护进程端能力** | 绑定到读+写作用域（浏览器驱动命令减去 admin：`eval`、`js`、`cookies`、`storage`）的每 spawn 作用域令牌。单一用途的 clientId 编码技能名称 + spawn id。spawn 退出时撤销。 | 始终作用域——绝不是守护进程根令牌 |
| **进程端环境** | `trusted: true` frontmatter 传递 `process.env` 减去 `GSTACK_TOKEN`。`trusted: false`（默认）仅保留最小白名单（LANG、LC_ALL、TERM、TZ）和模式剥离秘密（TOKEN/KEY/SECRET/PASSWORD、AWS_*、ANTHROPIC_*、OPENAI_*、GITHUB_* 等） | 不可信（需显式选择） |

`GSTACK_PORT` 和 `GSTACK_SKILL_TOKEN` 最后注入，因此父进程无法覆盖。

### 输出协议

stdout = JSON。stderr = 流式日志。退出 0 / 非零。默认 60s 超时，
通过 `--timeout=Ns` 覆盖。最大 stdout 1MB（截断 + 非零退出如果超出）。
符合 `gh` / `kubectl` / `docker` 惯例。

### SDK 分发如何工作

每个技能在 `_lib/browse-client.ts` 附带自己的 `browse-client.ts` 副本，
与规范的 `browse/src/browse-client.ts` 逐字节一致。`/skillify` 在每个生成的脚本
旁复制规范 SDK。每个技能完全自包含：将目录复制到任何地方都能运行。
版本漂移不可能——SDK 冻结在技能编写时的版本。

### 原子写入纪律（`/skillify` D3）

`browse/src/browser-skill-write.ts` 提供三个原语：

- `stageSkill(opts)` — 将文件写入 `~/.gstack/.tmp/skillify-<spawnId>/<name>/`，权限受限。
- `commitSkill(opts)` — 原子 `fs.renameSync` 到最终层路径。拒绝跟随符号链接暂存目录（`lstat` 检查），
  拒绝覆盖现有技能，对层根运行 `realpath` 规范。
- `discardStaged(stagedDir)` — `rm -rf` 暂存目录 + 每 spawn 包装器。幂等。测试失败或批准拒绝时调用。

不存在"几乎发布"状态。测试通过 + 用户批准 = 原子重命名。
测试失败或用户拒绝 = 暂存消失。

完整设计原理见 [`docs/designs/BROWSER_SKILLS_V1.md`](docs/designs/BROWSER_SKILLS_V1.md)。

---

## 领域技能

与浏览器技能不同的心智模型：代理编写的关于站点的*笔记*（非确定性脚本）。每个主机名一个。生命周期：

1. `domain-skill save <host>` — 代理写入关于站点的笔记（例如，
   "GitHub：PR 创建需要非staff的 `--draft` 标志"、"X.com：时间线使用光标分页，不是页码"）。
   默认状态：**隔离**。
2. **N=3** 次成功使用且 L4 提示注入分类器未标记笔记后，自动提升为**活跃**。
3. `domain-skill promote-to-global <host>` 将其提升到全局层（机器级，所有项目）。
4. `domain-skill rollback <host>` 降级；`domain-skill rm <host>` 做墓碑标记。

分类器标记由 L4 提示注入扫描自动设置；代理不手动设置。

存储：
- 每项目：`<project>/.gstack/domain-skills/<host>.md`
- 全局：`~/.gstack/domain-skills/<host>.md`

---

## 真实浏览器模式

`$B connect` 启动 **GStack Browser** — 一个由 Playwright 控制、重新品牌化的 Chromium，
自动加载 Side Panel 扩展并应用反机器人隐身补丁。你在可见窗口中实时观看每个命令执行。

```bash
$B connect              # 启动 headed 版 GStack Browser
$B goto https://app.com # 在可见窗口中导航
$B snapshot -i          # 真实页面的 refs
$B click @e3            # 在真实窗口中点击
$B focus                # 将窗口带到前台（macOS）
$B status               # 显示 Mode: cdp
$B disconnect           # 返回无头模式
```

窗口顶部有微妙的金色闪光线，右下角有浮动的 "gstack" 药丸，
让你始终知道哪个 Chrome 窗口正在被控制。

### "GStack Browser" 是什么意思

不是你的日常 Chrome——而是一个 Playwright 管理的 Chromium，在 Dock 和菜单栏中
有自定义品牌（`.app` 名称、Dock 图标、托盘，**不是** UA 字符串），
始终开启的 Layer C 反机器人隐身（大多数 JS 可观察的自动化特征被掩盖，
因此许多受反机器人保护的站点可以干净加载），一个报告底层 Chromium 版本的标准 Chrome User-Agent，
以及通过 `launchPersistentContext` 预加载的 gstack 扩展。UA 不再携带 `GStackBrowser` 后缀——
该品牌字符串本身就是一个高熵特征，因此浏览器现在报告标准的 `Chrome/<version>` UA。
最深层的 CDP 协议检测仍然可以通过（Google 仍然可以触发验证码；参见 `TODOS.md` 中的 CDP 补丁项）。
你带有标签页和书签的日常 Chrome 不受影响。

### 何时使用 headed 模式

- **QA 测试**，你想观看 Claude 点击你的应用
- **设计评审**，你需要看到 Claude 看到的确切内容
- **调试**，无头行为与真实 Chrome 不同时
- **演示**，分享屏幕时
- **对等代理**会话（远程代理驱动你的本地浏览器）

### CDP 感知技能

在真实浏览器模式下，`/qa` 和 `/design-review` 自动跳过 cookie 导入提示和无头变通方案——headed 浏览器已经拥有你登录的会话。

### Headed 模式 + 代理 + 浏览器原生下载 (v1.28.0.0)

三个协调的标志，用于阻止无头浏览器、指纹识别 Playwright 默认值或位于认证上游代理后面的站点：

```bash
# 可见 Chromium。在 Linux 容器无 DISPLAY 时自动启动 Xvfb。
$B --headed goto https://example.com

# SOCKS5 带认证——Chromium 无法提示 SOCKS5 凭据，因此 $B 运行本地 127.0.0.1 桥处理认证握手。
$B --proxy socks5://user:pass@residential.proxy.host:1080 goto https://example.com

# HTTP/HTTPS 代理直接传递给 Chromium。
$B --proxy http://corp-proxy:3128 goto https://example.com

# 浏览器原生下载，用于 Content-Disposition、重定向链、page.request.fetch() 失效的反机器人 CDN。
$B download "https://protected.example.com/file" /tmp/file.bin --navigate

# 组合。
$B --headed --proxy socks5://user:pass@host:1080 \
   download "https://protected.example.com/file" /tmp/file.bin --navigate
```

**凭据策略。** 通过 URL（`socks5://user:pass@host`）或环境变量 `BROWSE_PROXY_USER` / `BROWSE_PROXY_PASS` 传递凭据——
切勿同时使用。`$B` 在两者都设置时拒绝并给出明确提示；
静默覆盖会制造"在我机器上能跑"的调试陷阱。

**守护进程纪律。** `--proxy` 和 `--headed` 是守护进程启动配置。
一个以配置 A 运行的新调用遇到配置 B 时，以退出 1 和 `browse disconnect` 提示退出，
而非静默重启并丢失标签状态、cookies 或会话。

**隐身范围（Layer C，始终开启）。** 每个上下文——无头 `launch`、
`--headed`/`--proxy`、`handoff`，以及 `useragent`/`viewport --scale`
重建（`recreateContext`）——都获得完整的 Layer C 掩码，无需选择加入标志。
Layer C 掩盖 `navigator.webdriver`，恢复 `window.chrome.*` 形状
（`runtime`、`app`、`csi`、`loadTimes`），将 `Notification.permission`
与 Permissions API 对齐，报告每次安装的
`hardwareConcurrency`/`deviceMemory`（来自主机配置文件），扫描已知的
Selenium/Phantom/Nightmare/Playwright 全局变量，并安装一个
`Function.prototype.toString` 代理，使每个被修补的 getter 在深度 3 递归检查下
报告 `[native code]`。它仍然不伪造 `navigator.plugins` 或 `navigator.languages`——
现代指纹检查这些的一致性，固定值的合成会*更*像 bot，而不是 less。
ChromeDriver 的 `cdc_`/`__webdriver` 运行时工件和 Permissions 通知特征也在每条路径上清理。

`GSTACK_STEALTH=extended`（也接受 `1` 或 `true`；默认关闭）在顶部叠加六个更
激进的补丁——WebGL 渲染器欺骗、伪造的 `navigator.plugins` PluginArray、
`navigator.mediaDevices`。该模式主动撒谎并可能破坏反映这些属性的站点；
仅当默认触发检测时使用。对于带 C++ 补丁的 gbrowser 构建，
`GSTACK_*` 主机配置文件 env（GPU 厂商/渲染器、UA-CH 平台/型号、
硬件）发出 Pack 1 `--gstack-gpu-vendor` / `--gstack-gpu-renderer` /
`--gstack-ua-platform` / `--gstack-ua-model` / `--gstack-hw-concurrency` /
`--gstack-device-memory` 开关，将 GPU/UA-CH/硬件欺骗推送到原生代码，
且 `GSTACK_CDP_STEALTH=on`（或 `1`/`true`）发出 Pack 2
`--gstack-suppress-prepare-stack-trace` 开关（关闭 Cloudflare 的
`Error.prepareStackTrace` 金丝雀）。在标准 Playwright Chromium 上，
所有这些开关都是安全的 no-op。

`launchHeaded` / `handoff` 还通过 `ignoreDefaultArgs`（`STEALTH_IGNORE_DEFAULT_ARGS`）
剥离 Playwright 的自动化特征启动默认值：
`--enable-automation`（"Chrome 正由自动化测试软件控制"信息栏）、`--disable-extensions`、
`--disable-component-extensions-with-background-pages`、
`--disable-popup-blocking`、`--disable-component-update` 和
`--disable-default-apps`。

**容器支持。** `--headed` 在 Linux 上无 `DISPLAY` 时遍历显示范围（`:99`、`:100`……）
直到 `xdpyinfo` 报告空闲槽，然后生成 Xvfb。断开清理验证记录 PID 的
`/proc/<pid>/cmdline` 匹配 `Xvfb` 且启动时间匹配，然后才发送任何信号——
无 PID 复用脚枪。当设置 `WAYLAND_DISPLAY` 时完全跳过生成（Chromium 原生使用 Wayland）。
标准 Debian/Ubuntu 容器开箱即用；最小镜像（alpine、distroless）
可能需要字体/dbus/gtk 库才能让 headed Chromium 渲染。

**故障模式。** SOCKS5 上游被拒绝或不可达——在 3 次重试（5s 预算）后以脱敏错误快速失败。
上游中途掉落——桥仅杀死受影响的客户端连接；无可能破坏浏览器流量的传输重试。

---

## 侧边栏 + 侧边代理

随 GStack Browser 附带的 Chrome 扩展在 Side Panel 中显示每个浏览命令的实时活动流，
页面上的 `@ref` 覆盖，以及侧边栏内嵌的交互式 Claude PTY。

### 终端面板（头条）

Side Panel 的主要表面是**终端面板**——一个你可以直接从侧边栏输入的实时 `claude -p` PTY。
Activity / Refs / Inspector 是 `debug` 开关后面的调试覆盖层。WebSocket 认证使用
`Sec-WebSocket-Protocol`（浏览器无法在 WebSocket 升级上设置 `Authorization`），
且 PTY 会话令牌是通过 `POST /pty-session` 铸造的 30 分钟 HttpOnly cookie。

工具栏的 Cleanup 按钮和 Inspector 的 "Send to Code" 动作都通过
`window.gstackInjectToTerminal(text)`（由 `sidepanel-terminal.js` 暴露）
将文本管道到实时 Claude PTY。没有单独的 `/sidebar-command` POST——
实时 REPL 是唯一的执行面。

### 活动流

每个浏览命令的滚动流——名称、参数、持续时间、状态、错误。
在 Claude 工作时实时显示。由 SSE（`/activity/stream`）支持，
接受 Bearer 令牌或 HttpOnly `gstack_sse` 会话 cookie
（通过 `POST /sse-session` 铸造的 30 分钟流作用域 cookie）。

### Refs 标签

在 `$B snapshot` 之后，显示当前 `@ref` 列表（角色 + 名称），
让你看到 Claude 正在定位什么。

### CSS 检查器

由 `$B inspect`（基于 CDP）驱动。点击页面上的任何元素查看完整的 CSS 规则级联、
计算样式、盒模型和修改历史。"Send to Code" 按钮将描述注入 Claude PTY。

### 侧边栏架构

| 组件 | 位于 | 备注 |
|------|------|------|
| Side Panel UI | `extension/sidepanel.js`、`sidepanel-terminal.js` | Chrome 扩展表面 |
| Background SW | `extension/background.js` | 管理标签事件、端口管理 |
| Content script | `extension/content.js` | 页面覆盖、`gstack` 药丸 |
| Terminal agent | `browse/src/terminal-agent.ts` | PTY 生成、生命周期、认证 |
| Sidebar 工具函数 | `browse/src/sidebar-utils.ts` | URL 清理、辅助函数 |

在修改其中任何一个之前，先阅读 `CLAUDE.md` 中"侧边栏架构"下的注释块——
这里的静默失败通常源于不理解跨组件流程。

### 如果你想在日常 Chrome 中安装扩展（非 Playwright 控制的）

```bash
bin/gstack-extension    # 打开 chrome://extensions，复制路径到剪贴板
```

或者手动操作：`chrome://extensions` → 切换 Developer mode → 已解压的扩展程序 →
导航到 `~/.claude/skills/gstack/extension` → 固定扩展 →
输入 `$B status` 中的端口。

---

## 对等代理

远程 AI 代理（Codex、OpenClaw、Hermes，任何会说 HTTP 的）可以通过 ngrok 隧道
驱动你的本地浏览器。整个流程由 26 命令白名单、作用域令牌和拒绝日志门控。

### 工作原理

```bash
/pair-agent                     # 生成设置密钥，打印连接说明
# 将说明复制到远程代理
# 远程代理运行：
#   POST <tunnel-url>/connect 带设置密钥 → 获得作用域令牌（24h，单一客户端）
#   POST <tunnel-url>/command 带令牌 → 运行允许的命令
```

### 双监听器架构（v1.6.0.0+）

当 `pair-agent` 激活时，守护进程绑定**两个 HTTP 监听器**：

- **本地监听器**（`127.0.0.1:LOCAL_PORT`）。完整命令表面。绝不被 ngrok 转发。
  由你的 Claude Code、Side Panel、你机器上的任何东西使用。
- **隧道监听器**（`127.0.0.1:TUNNEL_PORT`）。锁定白名单——
  `/connect`、`/command`（作用域令牌 + 26 命令浏览器驱动白名单）、
  `/sidebar-chat`。ngrok 仅转发此端口。

通过隧道发送的根令牌返回 403。SSE 端点使用 30 分钟 HttpOnly `gstack_sse` cookie
（绝不对 `/command` 有效）。

### 26 命令隧道白名单

定义在 `browse/src/server.ts` 中的 `TUNNEL_COMMANDS`。纯门函数
`canDispatchOverTunnel(command)` 为单元测试导出。设置：

```
goto, click, text, screenshot, html, links, forms, accessibility,
attrs, media, data, scroll, press, type, select, wait, eval,
newtab, tabs, back, forward, reload, snapshot, fill, url, closetab
```

明显缺席：`pair`、`unpair`、`cookies`、`setup`、`launch`、`restart`、
`stop`、`tunnel-start`、`token-mint`、`state`、`connect`、`disconnect`。
尝试这些的远程代理获得 403 加拒绝日志中的新条目。

### 隧道拒绝日志

`~/.gstack/security/attempts.jsonl` — 仅追加，仅源加盐 SHA-256
+ 域（无原始 IP，无完整请求体），在 10MB 时轮转，5 代。
每设备盐位于 `~/.gstack/security/device-salt`（模式 0600）。

完整操作符指南见 [`docs/REMOTE_BROWSER_ACCESS.md`](docs/REMOTE_BROWSER_ACCESS.md)。

### 标签所有权

作用域令牌默认为 `tabPolicy: 'own-only'`。配对代理可以 `newtab` 创建自己的标签
并自由驱动该标签，但不能在另一个调用者拥有的标签上 `goto`、`fill`
或 `click`。`tabs` 列出所有标签元数据（一个已接受的权衡——参见 ARCHITECTURE.md），
但未拥有标签的 `text`/`html`/`snapshot` 内容被所有权检查阻止。

---

## 认证

三种令牌类型，三种生命周期，三种作用域。

| 令牌 | 生成者 | 生命周期 | 作用域 |
|------|--------|----------|--------|
| **Root token** | 守护进程启动（随机 UUID） | 守护进程进程生命周期 | 完整命令表面，仅本地监听器——隧道上 403 |
| **Setup key** | `POST /pair` | 5 分钟，一次性使用 | 单一兑换：在 `/connect` 出示，获得作用域令牌 |
| **Scoped token** | `POST /connect`（带设置密钥） | 24 小时 | 每客户端，白名单绑定，可选标签作用域 |

根令牌以 chmod 600 写入 `<project>/.gstack/browse.json`。
每个改变浏览器状态的命令必须包含 `Authorization: Bearer ***`。

### SSE 会话 cookie（v1.6.0.0+）

SSE 端点（`/activity/stream`、`/inspector/events`）接受 Bearer 令牌
或通过 `POST /ses-session` 铸造的 30 分钟 HttpOnly `gstack_sse` cookie。
`?token=<ROOT>` 查询参数认证不再支持。
这允许 Chrome 订阅活动流而无需将根令牌放入扩展存储。

### PTY 会话 cookie

终端面板使用单独的会话 cookie `gstack_pty`，通过 `POST /pty-session` 铸造。
不同作用域——可以生成 / 驱动实时 `claude` PTY，不能派生任意 `/command` 调用。
`/health` 端点必须不暴露此令牌。

### 令牌注册中心

`browse/src/token-registry.ts` 处理所有三种类型的铸币/验证/撤销，
外加每令牌速率限制。设置密钥是一次性的；作用域令牌有滑动 24h 窗口；
根令牌在每次守护进程启动时轮换。

---

## 安全栈

分层防御提示注入。每个用户在每条用户消息和每条可能携带不受信任内容
（Read、Glob、Grep、WebFetch、来自 `$B` 的页面文本）的工具输出上同步运行每个层。

| 层 | 模块 | 位于 |
|----|------|------|
| **L1** 数据标记 | `content-security.ts` | 服务器 + 侧边代理两端 |
| **L2** 隐藏元素剥离 | `content-security.ts` | 两端 |
| **L3** ARIA + URL 白名单 + 信封包装 | `content-security.ts` | 两端 |
| **L4** TestSavantAI ML 分类器（22MB ONNX） | `security-classifier.ts` | 仅侧边代理* |
| **L4b** Claude Haiku 转录检查 | `security-classifier.ts` | 仅侧边代理 |
| **L5** Canary 令牌（会话泄露检测） | `security.ts` | 两端——注入在编译版中，代理检查 |
| **L6** `combineVerdict` 集成 | `security.ts` | 两端 |

\* `security-classifier.ts` 不能从编译后的 browse 二进制导入——
`@huggingface/transformers` v4 需要 `onnxruntime-node`，后者在 Bun 编译的临时提取目录中
无法 `dlopen`。编译后的二进制仅运行 L1–L3、L5、L6。

### 阈值

- `BLOCK: 0.85` — 如果交叉确认会导致 BLOCK 的单一层分数
- `WARN: 0.75` — 交叉确认阈值。当 L4 和 L4b 都 >= 0.75 → BLOCK
- `LOG_ONLY: 0.40` — 门控转录分类器（当所有层 < 0.40 时跳过 Haiku）
- `SOLO_CONTENT_BLOCK: 0.92` — 无标签内容分类器的单层阈值

### 集成规则

仅当 ML 内容分类器和转录分类器都报告 >= WARN 时才 BLOCK。
单层高置信度降级为 WARN——这是 Stack Overflow 指令写作的误报缓解。
**Canary 泄漏始终 BLOCK（确定性）。**

### 环境旋钮

- `GSTACK_SECURITY_OFF=1` — 紧急关闭开关。分类器保持关闭，
  即使已预热。Canary 仍然注入；仅跳过 ML 扫描。
- `GSTACK_SECURITY_ENSEMBLE=deberta` — 选择加入 DeBERTa-v3 集成。
  添加 ProtectAI DeBERTa-v3-base-injection-onnx 作为 L4c 分类器。
  721MB 首次运行下载。启用集成后，BLOCK 需要 2/3 ML 分类器在 >= WARN 处同意。
- 分类器模型缓存：`~/.gstack/models/testsavant-small/`（112MB，仅首次运行）
  加上 `~/.gstack/models/deberta-v3-injection/`（721MB，仅在启用集成时）。
- 攻击日志：`~/.gstack/security/attempts.jsonl`（加盐 SHA-256 + 仅域，
  在 10MB 时轮转，5 代）。
- 每设备盐：`~/.gstack/security/device-salt`（0600）。
- 会话状态：`~/.gstack/security/session-state.json`（跨进程，原子）。

侧边栏头部的盾牌图标显示实时状态。完整威胁模型见
ARCHITECTURE.md § "提示注入防御"。

---

## 截图、PDF、视觉

### 截图模式

| 模式 | 语法 | Playwright API |
|------|--------|----------------|
| 整页（默认） | `screenshot [path]` | `page.screenshot({ fullPage: true })` |
| 仅视口 | `screenshot --viewport [path]` | `page.screenshot({ fullPage: false })` |
| 元素裁剪（标志） | `screenshot --selector <css> [path]` | `locator.screenshot()` |
| 元素裁剪（位置） | `screenshot "#sel" [path]` 或 `screenshot @e3 [path]` | `locator.screenshot()` |
| 区域裁剪 | `screenshot --clip x,y,w,h [path]` | `page.screenshot({ clip })` |

元素裁剪接受 CSS 选择器（`.class`、`#id`、`[attr]`）或 `@e`/`@c` refs。
**像 `button` 这样的标签选择器不被位置启发式捕获**——使用 `--selector` 标志形式。

`--base64` 返回 `data:image/png;base64,...` 而非写入磁盘——
与 `--selector`、`--clip`、`--viewport` 组合。

互斥：`--clip` + 选择器、`--viewport` + `--clip`，
以及 `--selector` + 位置选择器都会抛出。

### 视网膜截图 — `viewport --scale`

`viewport --scale <n>` 设置 Playwright 的 `deviceScaleFactor`（上下文级，
1-3 上限）：

```bash
$B viewport 480x600 --scale 2
$B load-html /tmp/card.html
$B screenshot /tmp/card.png --selector .card
# .card 在 400x200 CSS 像素 → card.png 是 800x400 像素
```

单独的 `--scale N`（无 `WxH`）保持当前视口大小。
缩放变化触发上下文重建，使 `@e`/`@c` refs 失效——之后重新运行
`snapshot`。通过 `load-html` 加载的 HTML 通过内存中回放保留。
在 headed 模式中被拒绝（真实浏览器控制缩放）。

### PDF 生成

`pdf` 接受完整的 Playwright 表面外加几个添加：

- **布局：** `--format letter|a4|legal`、`--width <dim>`、`--height <dim>`、
  `--margins <dim>`、`--margin-top/right/bottom/left <dim>`
- **结构：** `--toc`（如果加载了 Paged.js 则等待）、`--outline`、
  `--tagged`（PDF/A 可访问性）、`--print-background`、
  `--prefer-css-page-size`
- **品牌：** `--header-template <html>`、`--footer-template <html>`、
  `--page-numbers`
- **标签：** `--tab-id <N>` 渲染特定标签
- **大型负载：** `--from-file <payload.json>`（避免 shell argv 限制）

### 响应式截图

`responsive [prefix]` — 一次调用三张截图：手机 (375x812)、
平板 (768x1024)、桌面 (1280x720)。保存为 `{prefix}-mobile.png` 等。

### `prettyscreenshot`

在一次调用中组合清理 + 滚动 + 元素隐藏：

```bash
$B prettyscreenshot --cleanup --scroll-to "hero section" --hide ".cookie-banner" /tmp/clean.png
```

---

## 本地 HTML

两种渲染不在 Web 服务器上的 HTML 的方式：

| 方式 | 何时 | 之后 URL | 相对资源 |
|------|------|-----------|----------|
| `goto file://<abs-path>` | 文件已在磁盘上 | `file:///...` | 相对于文件的目录解析 |
| `goto file://./<rel>`、`goto file://~/<rel>` | 智能解析为绝对路径 | `file:///...` | 同上 |
| `load-html <file>` | 内存中生成的 HTML，无需父目录上下文 | `about:blank` | 断开（仅自包含 HTML） |

两者都通过受限目录策略限定于 cwd 或 `$TMPDIR` 下的文件，与 `eval` 的受限目录策略相同。
`file://` URL 保留查询字符串和片段（SPA 路由工作）。

`load-html` 有扩展名白名单（`.html`、`.htm`、`.xhtml`、`.svg`）和
魔数嗅探，拒绝被重命名为 HTML 的二进制文件。50MB 大小上限
（通过 `GSTACK_BROWSE_MAX_HTML_BYTES` 覆盖）。

`load-html` 内容在后续 `viewport --scale` 调用中通过内存中回放保留
（TabSession 跟踪加载的 HTML + waitUntil）。
回放在纯内存中——HTML 从不通过 `state save` 持久化到磁盘，
以避免泄漏秘密或客户数据。

---

## 批处理端点

`POST /batch` 在单个 HTTP 请求中发送多个命令。
消除每命令往返延迟——对 ngrok 上的远程代理至关重要，每次 HTTP 调用花费 2-5s。

```json
POST /batch
Authorization: Bearer ***

{
  "commands": [
    {"command": "text", "tabId": 1},
    {"command": "text", "tabId": 2},
    {"command": "snapshot", "args": ["-i"], "tabId": 3},
    {"command": "click", "args": ["@e5"], "tabId": 4}
  ]
}
}
```

每个命令通过 `handleCommandInternal` 路由——完整安全管道
（作用域检查、域验证、标签所有权、内容包装）对每个命令强制执行。
每命令错误隔离：一个失败不会中止批处理。
每批最多 50 命令。嵌套批处理被拒绝。速率限制：
1 批 = 1 针对每代理限制的请求。

模式：代理抓取 20 个页面打开 20 个标签（单独 `newtab` 或
批处理），然后 `POST /batch` 带 20 个 `text` 命令 → 20 个页面内容在
总共 ~2-3 秒 vs ~40-100 秒串行。

---

## 捕获

控制台、网络和对话框事件流入 O(1) 环形缓冲区（各 50,000 容量），
通过 `Bun.write()` 异步刷新到磁盘：

- 控制台：`.gstack/browse-console.log`
- 网络：`.gstack/browse-network.log`
- 对话框：`.gstack/browse-dialog.log`

`console`、`network` 和 `dialog` 命令从内存缓冲区读取（非磁盘），
因此即使磁盘慢也能实时捕获。

对话框（alert、confirm、prompt）默认自动接受以防止浏览器锁定。
`dialog-accept <text>` 控制 prompt 响应文本。

---

## JS 执行

`js` 运行内联表达式。`eval` 运行 JS 文件。两者在**相同的 JS 沙箱**
中运行——唯一的区别是内联 vs 文件。两者都支持 `await`——
包含 `await` 的表达式被自动包装在异步上下文中：

```bash
$B js "await fetch('/api/data').then(r => r.json())"   # 自动包装
$B js "document.title"                                  # 无需包装
$B eval my-script.js                                    # 带 await 的文件
```

对于 `eval` 文件，单行文件直接返回表达式值。
多行文件在使用 `await` 时需要显式 `return`。包含字面标记 "await" 的注释不触发包装。

路径安全：`eval` 拒绝 cwd 或 `/tmp` 外的路径。`js` 完全不读文件。

---

## 标签页、框架、状态

### 标签页

```bash
$B tabs                          # 列出所有打开标签
$B tab 3                         # 切换到标签 3
$B newtab https://example.com    # 打开新标签，切换到它
$B newtab --json                 # 程序化：返回 {"tabId":N,"url":...}
$B closetab                      # 关闭当前
$B closetab 2                    # 关闭标签 2
$B tab-each "text"               # 将 "text" 扇出到每个标签，返回 JSON
```

`tab-each <command>` 将命令扇出到每个打开的标签并返回 JSON 数组——
适用于"给我所有打开标签的文本"。

### 框架

```bash
$B frame "#stripe-iframe"        # 通过选择器切换到 iframe
$B frame @e7                     # 通过 ref
$B frame --name "checkout"       # 通过 name 属性
$B frame --url "stripe.com"      # 通过 URL 模式匹配
$B frame main                    # 返回顶层框架
```

切换时 refs 被清除（iframe 有自己的 AX 树）。

### 状态保存/加载

```bash
$B state save my-session         # 将 cookies + URLs 保存到 .gstack/browse-state-my-session.json
$B state load my-session         # 恢复
```

内存中的 `load-html` 内容有意**不**持久化（避免向磁盘泄漏秘密）。

### 监视

```bash
$B watch                         # 被动观察：用户浏览时每 5s 快照
$B watch stop                    # 返回变化摘要
```

当你手动驱动浏览器并希望 Claude 最终看到你做了什么而不不断发送 `snapshot` 调用时很有用。

### 收件箱

```bash
$B inbox                         # 列出侧边栏侦察的消息
$B inbox --clear                 # 读取后清空
```

侧边栏侦察（Chrome 扩展可以生成的后台进程）在用户注意到某些想要关注的东西时
为 Claude 投递笔记。存储在 `.gstack/browser-scout.jsonl` 中。

---

## CDP

### `$B cdp` — 原始 Chrome DevTools Protocol 派发

默认拒绝。只有 `browse/src/cdp-allowlist.ts`
中枚举的方法（`CDP_ALLOWLIST` 常量）可达；任何其他方法返回 403。
每个白名单条目声明作用域（标签 vs 浏览器）和输出（受信任 vs
不受信任）。不受信任的方法（数据泄露形状，例如
`Network.getResponseBody`）获得 UNTRUSTED-envelope 包装输出。

```bash
$B cdp Page.getLayoutMetrics
$B cdp Network.enable
$B cdp Accessibility.getFullAXTree --json '{"max_depth":5}'
```

发现允许的方法：阅读 `browse/src/cdp-allowlist.ts`。

### `$B inspect` — 基于 CDP 的 CSS 检查器

```bash
$B inspect ".header"                # 标题的完整规则级联
$B inspect ".header" --all          # 包含用户代理规则
$B inspect ".header" --history      # 显示修改历史
```

返回带特异性的匹配规则级联、计算样式、盒模型和
（带 `--history`）通过 `$B style` 自页面加载以来的每次 CSS 修改。
由 `browse/src/cdp-inspector.ts` 中每页的持久 CDP 会话驱动。

### `$B ux-audit`

```bash
$B ux-audit
```

返回带站点身份、导航、标题（上限 50）、文本块、
交互元素（上限 200）的 JSON——用于行为分析的页面结构而不倾倒整个 HTML。
`/qa` 和 `/design-review` 用于廉价覆盖率地图。

---

## 性能

| 工具 | 首次调用 | 后续调用 | 每调用上下文开销 |
|------|-----------|----------|----------------|
| Chrome MCP | ~5s | ~2-5s | ~2000 tokens（schema + 协议） |
| Playwright MCP | ~3s | ~1-3s | ~1500 tokens（schema + 协议） |
| **gstack browse** | **~3s** | **~100-200ms** | **0 tokens**（纯文本 stdout） |
| **gstack browse + 固化技能** | **~3s** | **~200ms** | **0 tokens**（单一技能调用） |

在 20 命令浏览器会话中，MCP 工具仅协议框架就消耗 30,000–40,000 tokens。
gstack 消耗零。固化技能路径将 20 命令会话降到单一 `$B skill run` 调用。

### 为什么用 CLI 而非 MCP

MCP 对远程服务工作良好。对本地浏览器自动化，它只增加纯开销：

- **上下文膨胀** — 每次 MCP 调用包含完整 JSON schemas。一个简单的
  "获取页面文本" 花费的上下文 token 是应有的 10 倍。
- **连接脆弱** — 持久 WebSocket/stdio 连接断开且无法重连。
- **不必要的抽象** — Claude 已有 Bash 工具。一个向 stdout 输出的 CLI
  是最简单的可能接口。

gstack 跳过所有这些。编译后的二进制文件。纯文本进，纯文本出。
无协议。无 schema。无连接管理。

---

## 多工作区

每个项目根目录（通过 `git rev-parse --show-toplevel` 检测）获得自己的
守护进程、端口、状态文件、cookies 和日志。无工作区间碰撞。

| 工作区 | 状态文件 | 端口 |
|--------|-----------|------|
| `/code/project-a` | `/code/project-a/.gstack/browse.json` | 随机 (10000–60000) |
| `/code/project-b` | `/code/project-b/.gstack/browse.json` | 随机 (10000–60000) |

浏览器技能三层查找遍历 project → global → bundled，因此 project 层技能
在 `/code/project-a/.gstack/browser-skills/foo/` 阴影了全局的
`~/.gstack/browser-skills/foo/`，但仅在 project-a 内。

---

## 环境变量

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `BROWSE_PORT` | 0 (随机 10000–60000) | HTTP 服务器的固定端口（调试覆盖） |
| `BROWSE_IDLE_TIMEOUT` | 1800000 (30 分钟) | 空闲关闭超时（毫秒） |
| `BROWSE_STATE_FILE` | `.gstack/browse.json` | 状态文件路径 |
| `BROWSE_SERVER_SCRIPT` | 自动检测 | `server.ts` 路径 |
| `BROWSE_CDP_URL` | (无) | 设置为 `channel:chrome` 进入真实浏览器模式 |
| `BROWSE_CDP_PORT` | 0 | CDP 端口（内部使用） |
| `BROWSE_HEADLESS_SKIP` | 0 | 完全跳过 Chromium 启动（仅测试 harness） |
| `BROWSE_TUNNEL` | 0 | 激活双监听器隧道架构（需要 `NGROK_AUTHTOKEN`） |
| `BROWSE_TUNNEL_LOCAL_ONLY` | 0 | 仅在本地测试——在本地绑定两个监听器，无需 ngrok |
| `GSTACK_BROWSE_MAX_HTML_BYTES` | 52428800 (50MB) | `load-html` 大小上限 |
| `GSTACK_SECURITY_OFF` | 未设置 | 紧急关闭开关——禁用 ML 分类器 |
| `GSTACK_SECURITY_ENSEMBLE` | 未设置 | 设为 `deberta` 启用 3 分类器集成（721MB 下载） |
| `GSTACK_STEALTH` | 未设置 | 设为 `extended`（也接受 `1`/`true`）在 Layer C 上叠加六个激进补丁（WebGL 欺骗、伪造插件、mediaDevices）。主动撒谎；可能破坏站点。 |
| `GSTACK_CDP_STEALTH` | 未设置 | 设为 `on`/`1`/`true` 发出 `--gstack-suppress-prepare-stack-trace`（仅 gbrowser Pack 2 / B11 C++ 补丁；标准 Chromium 上无操作） |
| `GSTACK_GPU_VENDOR`、`GSTACK_GPU_RENDERER`、`GSTACK_GPU_CHIPSET` | 未设置 | 每次安装的 GPU 欺骗，喂给 Pack 1 WebGL/UA-CH C++ 补丁。由 gbd 从主机配置文件设置；仅存在时作为 `--gstack-gpu-vendor` / `--gstack-gpu-renderer` / `--gstack-ua-model` 命令行开关发出。 |
| `GSTACK_PLATFORM` | 未设置 | 主机平台分类（`MacARM`/`MacIntel` → `macOS`、`Win32` → `Windows`、`Linux*` → `Linux`）作为 `--gstack-ua-platform` 发出 |
| `GSTACK_HW_CONCURRENCY`、`GSTACK_DEVICE_MEMORY` | 主机配置文件（回退 8） | 每次安装的 `hardwareConcurrency`/`deviceMemory`，由 Layer C 报告并作为 `--gstack-hw-concurrency` / `--gstack-device-memory` 发出，用于 worker-navigator C++ 补丁 |

---

## 源码映射

```
browse/
├── src/
│   ├── cli.ts                   # 薄客户端——读取状态，发送 HTTP，输出
│   ├── server.ts                # Bun HTTP 守护进程——路由命令，双监听器
│   ├── browser-manager.ts       # Chromium 生命周期、标签、ref 映射、崩溃检测
│   ├── socks-bridge.ts          # 本地 127.0.0.1 SOCKS5 桥，处理 Chromium 无法表达的认证握手
│   ├── proxy-config.ts          # --proxy URL 解析 + 凭据解析（URL vs env，两者都快速失败）
│   ├── proxy-redact.ts          # 凭据脱敏助手，用于任何暴露给日志/错误的代理 URL
│   ├── xvfb.ts                  # Xvfb 自动启动 + 孤儿清理，带 PID + 启动时间验证
│   ├── stealth.ts               # Layer C：webdriver 掩码 + window.chrome.* + Notification/Permissions + 每次安装的硬件 + toString 代理 + 自动化全局扫描；buildGStackLaunchArgs（GSTACK_* 命令行开关）；GSTACK_STEALTH=extended 选择加入
│   ├── browse-client.ts         # 规范 SDK——技能作为 _lib/browse-client.ts 导入的内容
│   ├── snapshot.ts              # AX 树 → @e/@c refs → Locator 映射；-D/-a/-C 处理
│   ├── read-commands.ts         # 非变更：text、html、links、js、css、is、dialog、...
│   ├── write-commands.ts        # 变更：goto、click、fill、upload、dialog-accept、...
│   ├── meta-commands.ts         # state、watch、inbox、frame、ux-audit、chain、diff、...
│   ├── browser-skills.ts        # 3 层遍历 + frontmatter 解析器 + 墓碑
│   ├── browser-skill-commands.ts # $B skill list/show/run/test/rm + spawnSkill
│   ├── browser-skill-write.ts   # D3 原子阶段/提交/丢弃助手，用于 /skillify
│   ├── skill-token.ts           # mintSkillToken / revokeSkillToken（每 spawn，作用域）
│   ├── domain-skills.ts         # 每站点代理笔记（状态机：隔离→活跃→全局）
│   ├── domain-skill-commands.ts # $B domain-skill save/list/show/edit/promote/rollback/rm
│   ├── cdp-allowlist.ts         # 默认拒绝 CDP 方法白名单
│   ├── cdp-bridge.ts            # CDP 会话生命周期桥
│   ├── cdp-commands.ts          # $B cdp 派发器
│   ├── cdp-inspector.ts         # $B inspect ——每页持久 CDP 会话
│   ├── activity.ts              # ActivityEntry、CircularBuffer、SSE 订阅者、隐私过滤
│   ├── buffers.ts               # 控制台/网络/对话框环形缓冲区（O(1) 环）
│   ├── tab-session.ts           # 每标签会话状态（load-html 回放、ref 映射作用域）
│   ├── token-registry.ts        # 铸币/验证/撤销根 + 设置密钥 + 作用域令牌
│   ├── sse-session-cookie.ts    # 30 分钟 HttpOnly cookie，用于 /activity/stream + /inspector/events
│   ├── pty-session-cookie.ts    # 单独作用域：实时 Claude PTY 认证
│   ├── tunnel-denial-log.ts     # ~/.gstack/security/attempts.jsonl 写入器（加盐）
│   ├── path-security.ts         # validateOutputPath / validateReadPath / validateTempPath
│   ├── url-validation.ts        # goto 的 URL 安全检查
│   ├── content-security.ts      # L1-L3：数据标记、隐藏条、ARIA、URL 白名单、信封
│   ├── security.ts              # L5 canary + L6 verdict 组合器 + 阈值
│   ├── security-classifier.ts   # L4 ML 分类器（TestSavant + 可选 DeBERTa 集成）
│   ├── terminal-agent.ts        # Side Panel Claude PTY 管理器（认证 + 生命周期）
│   ├── sidebar-utils.ts         # Sidebar URL 清理 + 助手
│   ├── cookie-import-browser.ts # 从真实 Chromium 浏览器解密 + 导入 cookies
│   ├── cookie-picker-routes.ts  # /cookie-picker/* 的 HTTP 路由
│   ├── cookie-picker-ui.ts      # cookie 选择器的自包含 HTML/CSS/JS
│   ├── network-capture.ts       # $B network 的网络请求捕获
│   ├── media-extract.ts         # $B media 的媒体元素提取
│   ├── project-slug.ts          # 状态路径的项目 slug 推导
│   ├── error-handling.ts        # safeUnlink / safeKill / isProcessAlive
│   ├── platform.ts              # OS 检测（macOS、Linux、Windows）
│   ├── telemetry.ts             # 匿名选择加入使用遥测
│   ├── find-browse.ts           # 定位运行中的守护进程或引导
│   └── config.ts                # 配置解析（env / 文件）
├── test/                        # 集成测试 + HTML fixtures
└── dist/
    └── browse                   # 编译后的二进制文件（~58MB，Bun --compile）

browser-skills/
└── hackernews-frontpage/        # 捆绑的参考技能
    ├── SKILL.md
    ├── script.ts
    ├── _lib/browse-client.ts
    ├── fixtures/hn-2026-04-26.html
    └── script.test.ts

scrape/SKILL.md.tmpl             # /scrape gstack 技能——匹配或原型入口点
skillify/SKILL.md.tmpl           # /skillify gstack 技能——将最后 /scrape 固化为永久技能
```

---

## 开发

### 先决条件

- [Bun](https://bun.sh/) v1.0+
- Playwright 的 Chromium（由 `bun install` 自动安装）

### 快速开始

```bash
bun install                      # 安装依赖 + Playwright Chromium
bun test                         # 所有集成测试（仅 browse 约 3 秒）
bun run dev <cmd>                # 从源码运行 CLI（无编译）
bun run build                    # 编译到 browse/dist/browse
```

### 开发模式 vs 编译后的二进制文件

在开发期间使用 `bun run dev` 而非编译后的二进制文件。它直接用 Bun 运行
`browse/src/cli.ts`，即时反馈：

```bash
bun run dev goto https://example.com
bun run dev text
bun run dev snapshot -i
bun run dev click @e3
```

编译后的二进制文件（`bun run build`）仅用于分发。它使用 Bun 的
`--compile` 标志在 `browse/dist/browse` 生成一个约 58MB 的单一可执行文件。

### 运行测试

```bash
bash
bun test                                    # 所有测试
bun test browse/test/commands               # 命令集成测试
bun test browse/test/snapshot               # 快照测试
bun test browse/test/cookie-import-browser  # cookie 导入单元测试
bun test browse/test/browser-skill-write    # D3 原子写入助手测试
bun test browse/test/tunnel-gate-unit       # canDispatchOverTunnel 纯测试
```

测试启动本地 HTTP 服务器（`browse/test/test-server.ts`），提供来自
`browse/test/fixtures/` 的 HTML fixtures，然后针对这些页面执行 CLI。

### 添加新命令

1. 在 `read-commands.ts`（非变更）或 `write-commands.ts`
   （变更），或 `meta-commands.ts`（服务器 / 生命周期）中添加处理程序。
2. 在 `server.ts` 中注册路由。
3. 在 `browse/src/commands.ts` 中添加 `COMMAND_DESCRIPTIONS` 条目（带
   清晰的 `description` 和 `usage`——`gen-skill-docs` 验证套件强制执行
   `description` 中无 `|` 字符）。
4. 在 `browse/test/commands.test.ts` 中添加带 HTML fixture 的测试用例
   （如需要）。
5. 运行 `bun test` 验证。
6. 运行 `bun run build` 编译。
7. 运行 `bun run gen:skill-docs` 重新生成 SKILL.md（命令出现
   在下游命令参考表中）。

### 添加新浏览器技能

手写技能：复制 `browser-skills/hackernews-frontpage/`，
更新 SKILL.md frontmatter，针对目标站点重写 `script.ts`，
重新捕获 fixture，更新解析器测试。`bun test` 验证
SKILL.md 契约（兄弟 SDK 字节一致性，frontmatter schema）。

代理编写的技能：通过 `/scrape <intent>` 驱动页面一次，
说 `/skillify`，在批准门控中接受建议的名称。测试通过后
技能到达 `~/.gstack/browser-skills/<name>/`。

### 部署到活动技能

活动技能位于 `~/.claude/skills/gstack/`。更改后：

```bash
cd ~/.claude/skills/gstack
git fetch origin && git reset --hard origin/main
bun run build
```

或直接复制二进制文件：

```bash
cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse
```

---

## 交叉引用

- [`ARCHITECTURE.md`](ARCHITECTURE.md) — 系统级架构，双监听器隧道设计，提示注入防御威胁模型
- [`CLAUDE.md`](CLAUDE.md) — 项目级指令，侧边栏架构笔记，安全栈约束
- [`docs/REMOTE_BROWSER_ACCESS.md`](docs/REMOTE_BROWSER_ACCESS.md) — `/pair-agent` 操作符指南（设置密钥、作用域令牌、拒绝日志）
- [`docs/designs/BROWSER_SKILLS_V1.md`](docs/designs/BROWSER_SKILLS_V1.md) — 浏览器技能运行时的设计文档（Phase 1 + 2a + 路线图）
- [`scrape/SKILL.md`](scrape/SKILL.md) — `/scrape` 技能：匹配或原型数据提取
- [`skillify/SKILL.md`](skillify/SKILL.md) — `/skillify` 技能：将最后 `/scrape` 固化为永久技能
- [`TODOS.md`](TODOS.md) — `/automate`（Phase 2b P0）、Phase 3 解析器注入、Phase 4 eval + 沙箱

---

## 致谢

浏览器自动化层基于 [Playwright](https://playwright.dev/)。
Playwright 的 AX 树 API、定位器系统和无头 Chromium 管理
使基于 ref 的交互成为可能。
快照系统——将 `@ref` 标签分配给 AX 树节点并将它们映射回
Playwright Locators——完全构建在 Playwright 的原语之上。
感谢 Playwright 团队构建了如此坚实的基础。

提示注入 L4 层使用
[TestSavantAI/distilbert-v1.1-32](https://huggingface.co/TestSavantAI/distilbert-v1.1-32)
（112MB ONNX），可选集成层使用
[ProtectAI/deberta-v3-base-prompt-injection-v2](https://huggingface.co/protectai/deberta-v3-base-prompt-injection-v2)
（721MB ONNX）——两者都通过 `@huggingface/transformers` 在本地运行。

CDP 应急出口由直接受 Codex 启发的白名单门控——
v1.4 设计阶段的 T2 外部声音审查：默认拒绝加显式白名单，
而非默认允许加黑名单。
