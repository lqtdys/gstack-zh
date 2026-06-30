# Chrome vs Chromium：为什么我们使用 Playwright 的捆绑 Chromium

## 原始愿景

当我们构建 `$B connect` 时，计划是连接到用户的**真实 Chrome 浏览器** —— 带他们的 cookie、会话、扩展和打开的标签页的那个。不再需要 cookie 导入。该设计需要：

1. 通过 CDP 使用 `chromium.connectOverCDP(wsUrl)` 连接到运行中的 Chrome
2. 优雅地退出 Chrome，用 `--remote-debugging-port=9222` 重新启动
3. 访问用户的真实浏览上下文

这就是为什么 `chrome-launcher.ts` 存在（361 LOC 的二进制文件发现、CDP 端口探测和运行时检测）以及为什么该方法被称为 `connectCDP()`。

## 实际情况

真正由 Playwright 启动的 Chrome 会静默阻止 `--load-extension`。扩展无法加载。我们需要扩展来用于活动源、引用的侧面板。

实现回退到使用 Playwright 捆绑 Chromium 的 `chromium.launchPersistentContext()` —— 它通过 `--load-extension` 和 `--disable-extensions-except` 可靠地加载扩展。但命名保留了：`connectCDP()`、`connectionMode: 'cdp'`、`BROWSE_CDP_URL`、`chrome-launcher.ts`。

原始愿景（访问用户的真实浏览器状态）从未实现。我们每次都会启动一个新的浏览器 —— 功能上与 Playwright 的 Chromium 相同，但附带了 361 行死代码和误导性名称。

## 发现（2026-03-22）

在一次 `/office-hours` 设计会话中，我们追踪架构并发现：

1. `connectCDP()` 不使用 CDP —— 它调用 `launchPersistentContext()`
2. `connectionMode: 'cdp'` 具有误导性 —— 它只是"有头"
3. `chrome-launcher.ts` 是死代码 —— 它唯一的导入在一个不可达的 `attemptReconnect()` 方法中
4. `preExistingTabIds` 是为保护我们从未连接的真正的 Chrome 标签页而设计的
5. `$B handoff`（无头 → 有头）使用了不同的 API（`launch()` + `newContext()`），无法加载扩展，创建了两种不同的"有头"体验

## 修复

### 重命名
- `connectCDP()` → `launchHeaded()`
- `connectionMode: 'cdp'` → `connectionMode: 'headed'`
- `BROWSE_CDP_URL` → `BROWSE_HEADED`

### 删除
- `chrome-launcher.ts`（361 LOC）
- `attemptReconnect()`（死方法）
- `preExistingTabIds`（死概念）
- `reconnecting` 字段（死状态）
- `cdp-connect.test.ts`（已删除代码的测试）

### 收敛
- `$B handoff` 现在使用 `launchPersistentContext()` + 扩展加载（与 `$B connect` 相同）
- 一种有头模式，不是两种
- 切换为您免费提供扩展 + 侧面板

### 门控
- 侧面板聊天在 `--chat` 标志后
- `$B connect`（默认）：活动源 + 引用仅
- `$B connect --chat`：+ 实验性独立聊天代理

## 架构（修复后）

```
浏览器状态：
  无头（默认） ←→ 有头（$B connect 或 $B handoff）
     Playwright            Playwright（相同引擎）
     launch()              launchPersistentContext()
     不可见                可见 + 扩展 + 侧面板

侧面板（正交附加组件，仅在有头模式下）：
  活动标签    — 始终开启，显示实时浏览命令
  引用标签    — 始终开启，显示 @ref 覆盖
  聊天标签    — 通过 --chat 选择加入，实验性独立代理

数据桥接（侧面板 → 工作区）：
  侧面板写入 .context/sidebar-inbox/*.json
  工作区通过 $B inbox 读取
```

## 为什么不用真正的 Chrome？

Playwright 在由命令行启动时阻止真正的 Chrome 使用 `--load-extension`。这是一个 Chrome 安全功能 —— 通过命令行参数加载的扩展在基于 Chromium 的浏览器中受到限制以防止恶意扩展注入。

Playwright 的捆绑 Chromium 没有此限制，因为它专为测试和自动化设计。`ignoreDefaultArgs` 选项让我们绕过了 Playwright 自己的扩展阻止标志。

如果我们想访问用户的真实 cookie / 会话，路径是：
1. Cookie 导入（已通过 `$B cookie-import`）
2. Conductor 会话注入（未来 — 侧面板向工作区代理发送消息）

不重新连接到真正的 Chrome。
