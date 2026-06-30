# 设计：设计霰弹枪 — 浏览器到Agent反馈循环

生成日期：2026-03-27
分支：garrytan/agent-design-tools
状态：活文档 — 发现并修复缺陷时更新

## 此功能的作用

设计霰弹枪生成多个AI设计样稿，在用户真实浏览器中以对比画布的形式并排打开，并收集结构化反馈（选择一个最喜爱的、对替代方案评分、留下备注、请求重新生成）。反馈流向编码Agent并据此执行：继续执行已批准的变体，或者生成新变体并重新加载画布。

用户无需离开浏览器标签页。Agent不会提出重复性问题。画布就是反馈机制。

## 核心问题：两个必须通信的世界

```
  ┌─────────────────────┐          ┌──────────────────────┐
  │   用户的浏览器       │          │   编码Agent          │
  │   (真实Chrome)      │          │   (Claude Code /    │
  │                     │          │    Conductor)         │
  │ 对比画布含按钮：     │   ???    │                      │
  │ - 提交              │ ──────── │ 需要知道的：         │
  │ - 重新生成          │          │ - 选中的是什么      │
  │ - 再来点类似的      │          │ - 星级评分           │
  │ - 混音              │          │ - 评论               │
  └─────────────────────┘          │ - 请求重新生成？     │
                                   └──────────────────────┘
```

"???"是难点。用户点击Chrome中的按钮，终端里运行的Agent需要了解此事。这两个完全独立进程之间没有共享内存、没有共享事件总线、没有WebSocket连接。

## 架构：链接机制如何工作

```
  用户的浏览器                      $D serve (Bun HTTP)              Agent
  ═══════════════                   ═══════════════════              ═════
       │                                   │                           │
       │  GET /                            │                           │
       │ ◄─────── 提供画布HTML ────────────►│                           │
       │    (将 __GSTACK_SERVER_URL         │                           │
       │     注入<head>)                   │                           │
       │                                   │                           │
       │  [用户评分、选择、评论]            │                           │
       │                                   │                           │
       │  POST /api/feedback               │                           │
       │ ─────── {preferred:"A",...} ─────►│                           │
       │                                   │                           │
       │  ◄── {received:true} ────────────│                           │
       │                                   │── 写入 feedback.json ──► │
       │  [输入禁用，                       │   (或 feedback-pending    │
       │   "返回Agent"显示]                │    .json 用于重新生成)    │
       │                                   │                           │
       │                                   │                  [Agent轮询
       │                                   │                   每5秒,
       │                                   │                   读取文件]
```

### 三个文件

| 文件 | 写入时机 | 含义 | Agent操作 |
|------|---------|------|----------|
| `feedback.json` | 用户点击"提交" | 最终选定，完成 | 读取并继续 |
| `feedback-pending.json` | 用户点击"重新生成/更多类似" | 想要新选项 | 读取、删除、生成新变体、重新加载画布 |
| `feedback.json`（第2轮+） | 重新生成后点击"提交" | 迭代后的最终选定 | 读取并继续 |

### 状态机

```
  $D serve 启动
       │
       ▼
  ┌──────────┐
  │ 服务中   │◄──────────────────────────────────────┐
  │          │                                        │
  │ 画布已就绪│  POST /api/feedback                    │
  │ ，等待中 │  {regenerated: true}                   │
  │          │──────────────────►┌──────────────┐     │
  │          │                   │ 重新生成中   │     │
  │          │                   │              │     │
  └────┬─────┘                   │ Agent有      │     │
       │                         │ 10分钟来     │     │
       │  POST /api/feedback     │ POST新       │     │
       │  {regenerated: false}   │ 画布HTML     │     │
       │                         └──────┬───────┘     │
       ▼                                │             │
  ┌──────────┐                POST /api/reload        │
  │  完成    │                {html: "/new/board"}    │
  │          │                          │             │
  │ 退出码0  │                          ▼             │
  └──────────┘                   ┌──────────────┐     │
                                 │  重新加载    │─────┘
                                 │              │
                                 │ 画布自动刷新 │
                                 │ (同一标签页) │
                                 └──────────────┘
```

### 端口发现

Agent后台运行 `$D serve` 并从stderr读取端口：

```
SERVE_STARTED: port=54321 html=/path/to/board.html
SERVE_BROWSER_OPENED: url=http://127.0.0.1:54321
```

Agent从stderr中解析 `port=XXXXX` 端口号。后续当用户请求重新生成时，需要此端口来POST `/api/reload`。如果Agent丢失了端口号，则无法重新加载画布。

### 为什么用127.0.0.1而不使用localhost

`localhost` 在某些系统上可能解析为IPv6 `::1`，而 Bun.serve() 仅在IPv4上监听。更重要的是，`localhost` 会携带开发者一直在处理的每个域名的所有开发Cookie。在拥有大量活跃会话的机器上，这会超过Bun默认的头部大小限制（HTTP 431错误）。`127.0.0.1` 避免了这两个问题。

## 每个边界情况与陷阱

### 1. 僵尸表单问题

**问题：** 用户提交反馈，POST成功，服务器退出。但HTML页面仍在Chrome中打开。页面看起来仍可交互。用户可能编辑反馈并再次点击"提交"。由于服务器已停止，什么也不会发生。

**修复：** POST成功后，画布JS：
- 禁用所有输入（按钮、单选框、文本区域、星级评分）
- 完全隐藏"重新生成"栏
- 用"反馈已收到！请返回编码Agent。"替换"提交"按钮
- 显示"想做更多更改？在设计Agent中再次运行 `/design-shotgun`。"
- 页面变为只读的提交记录

**实现位置：** `compare.ts:showPostSubmitState()`（第484行）

### 2. 死服务器问题

**问题：** 在用户仍打开画布的情况下，服务器超时（默认10分钟）或崩溃。用户点击"提交"。fetch() 静默失败。

**修复：** `postFeedback()` 函数有一个 `.catch()` 处理器。网络故障时：
- 显示红色错误横幅："连接断开"
- 显示收集到的反馈JSON在可复制的 `<pre>` 块中
- 用户可以直接复制粘贴到编码Agent中

**实现位置：** `compare.ts:showPostFailure()`（第546行）

### 3. 陈旧的重新生成旋转器

**问题：** 用户点击"重新生成"。画布显示旋转器并每2秒轮询 `/api/progress`。Agent崩溃或生成新变体耗时过长。旋转器永远旋转。

**修复：** 进度轮询有硬性的5分钟超时（150次轮询 x 2秒间隔）。5分钟后：
- 旋转器被替换为："出问题了。"
- 显示"请在编码Agent中再次运行 `/design-shotgun`。"
- 轮询停止。页面变为信息性的。

**实现位置：** `compare.ts:startProgressPolling()`（第511行）

### 4. file:// URL问题（原始BUG）

**问题：** 技能模板最初使用 `$B goto file:///path/to/board.html`。但 `browse/src/url-validation.ts:71` 出于安全原因阻止 `file://` URL。回退方案 `open file://...` 会打开用户的macOS浏览器，但 `$B eval` 轮询的是Playwright的无头浏览器（不同进程，从未加载页面）。Agent无限轮询空DOM。

**修复：** `$D serve` 通过HTTP提供服务。切勿对画布使用 `file://`。`$D compare` 上的 `--serve` 标志将画布生成和HTTP服务结合在一个命令中。

**证据：** 见 `.context/attachments/image-v2.png` — 真实用户遇到过的确切Bug。Agent正确诊断了：(1) `$B goto` 拒绝 `file://` URL，(2) 即使使用浏览器守护进程也没有轮询循环。

### 5. 双击竞态

**问题：** 用户快速连续点击"提交"两次。两个POST请求到达服务器。第一个将状态设为"done"并在100ms后安排 exit(0)。第二个在100ms窗口内到达。

**当前状态：** 未完全保护。`handleFeedback()` 函数在处理前不检查状态是否已是"done"。第二个POST会成功并写入第二个 `feedback.json`（无害，相同数据）。退出仍在100ms后触发。

**风险：** 低。画布在第一个成功POST响应时禁用所有输入，因此第二次点击需要在~1ms内到达。并且两次写入包含相同的反馈数据。

**潜在修复：** 在 `handleFeedback()` 顶部添加 `if (state === 'done') return Response.json({error: 'already submitted'}, {status: 409})`。

### 6. 端口协调问题

**问题：** Agent后台运行 `$D serve` 并从stderr解析 `port=54321`。Agent后续需要此端口在重新生成期间POST `/api/reload`。如果Agent丢失上下文（对话压缩、上下文窗口填满），可能忘记端口号。

**当前状态：** 端口只能输出到stderr一次。Agent必须记住它。没有写入端口文件到磁盘。

**潜在修复：** 启动时在画布HTML旁边写入 `serve.pid` 或 `serve.port` 文件。Agent可随时读取：
```bash
cat "$_DESIGN_DIR/serve.port"  # → 54321
```

### 7. 反馈文件清理问题

**问题：** 重新生成轮次的 `feedback-pending.json` 残留在磁盘上。如果Agent在读取前崩溃，下一个 `$D serve` 会话会找到陈旧文件。

**当前状态：** 解析器模板中的轮询循环要求在读取后删除 `feedback-pending.json`。但这取决于Agent完美遵循说明。陈旧文件可能混淆新会话。

**潜在修复：** `$D serve` 启动时检查并删除陈旧的反馈文件。或者：给文件加时间戳（`feedback-pending-1711555200.json`）。

### 8. 顺序生成规则

**问题：** 底层的OpenAI GPT Image API对并发图像生成请求进行速率限制。当3个 `$D generate` 调用并行运行时，1个成功，2个被中止。

**修复：** 技能模板必须明确说明："一次生成一个样稿。不要并行化 `$D generate` 调用。"这是提示级指令，不是代码级锁。设计二进制不强制执行顺序执行。

**风险：** Agent被训练为并行化独立工作。没有明确指令，它们会尝试同时运行3次生成。这会浪费API调用和金钱。

### 9. AskUserQuestion冗余

**问题：** 用户通过画布提交反馈（在JSON中提供了首选变体、评分、评论）后，Agent再次询问："你更喜欢哪个变体？"这很烦人。画布的全部目的就是避免这种情况。

**修复：** 技能模板必须说："不要使用 AskUserQuestion 询问用户偏好。读取 `feedback.json`，其中包含他们的选择。仅使用 AskUserQuestion 确认你理解正确，而非重新询问。"

### 10. CORS问题

**问题：** 如果画布HTML引用外部资源（字体、来自CDN的图片），浏览器会发送 `Origin: http://127.0.0.1:PORT` 的请求。大多数CDN允许此操作，但有些可能阻止。

**当前状态：** 服务器不设置CORS头。画布HTML是自包含的（图片base64编码，样式内联），因此实践中未成为问题。

**风险：** 当前设计低。如果画布加载外部资源则会有影响。

### 11. 大负荷问题

**问题：** 对 `/api/feedback` 的POST体没有大小限制。如果画布发送多MB负载，`req.json()` 会将其全部解析到内存中。

**当前状态：** 实际上，反馈JSON约500字节到~2KB。风险是理论上的，不是实践中的。画布JS构造固定形状的JSON对象。

### 12. fs.writeFileSync错误

**问题：** `serve.ts:138` 中 `feedback.json` 的写入使用 `fs.writeFileSync()` 且无 try/catch。如果磁盘已满或目录只读，这会抛出异常并崩溃服务器。用户看到旋转器永远旋转（服务器已死，但画布不知道）。

**风险：** 实践低（画布HTML刚刚写入同一目录，证明可写）。但用 try/catch 和500响应会更干净。

## 完整流程（逐步说明）

### 快乐路径：首次即选中

```
1. Agent运行：$D compare --images "A.png,B.png,C.png" --output board.html --serve &
2. $D serve 在随机端口（如54321）上启动 Bun.serve()
3. $D serve 在用户浏览器中打开 http://127.0.0.1:54321
4. $D serve 输出到stderr：SERVE_STARTED: port=54321 html=/path/board.html
5. $D serve 写入注入 __GSTACK_SERVER_URL 的画布HTML
6. 用户看到并排3个变体的对比画布
7. 用户选择选项B，评分 A:3/5, B:5/5, C:2/5
8. 用户在整体反馈中写"B的间距更好，就选它"
9. 用户点击"提交"
10. 画布JS POST到 http://127.0.0.1:54321/api/feedback
    请求体：{"preferred":"B","ratings":{"A":3,"B":5,"C":2},"overall":"B的间距更好","regenerated":false}
11. 服务器将 feedback.json 写入磁盘（与board.html同目录）
12. 服务器将反馈JSON输出到stdout
13. 服务器响应 {received:true, action:"submitted"}
14. 画布禁用所有输入，显示"返回您的编码Agent"
15. 服务器在100ms后以退出码0退出
16. Agent的轮询循环找到 feedback.json
17. Agent读取、总结给用户、继续执行
```

### 重新生成路径：用户想要不同选项

```
1-6.  同上
7.  用户点击"完全不同"选项
8.  用户点击"重新生成"
9.  画布JS POST到 /api/feedback
    请求体：{"regenerated":true,"regenerateAction":"different","preferred":"","ratings":{},...}
10. 服务器将 feedback-pending.json 写入磁盘
11. 服务器状态 → "regenerating"
12. 服务器响应 {received:true, action:"regenerate"}
13. 画布显示旋转器："正在生成新设计..."
14. 画布开始每2秒轮询 GET /api/progress

    与此同时，在Agent中：
15. Agent的轮询循环找到 feedback-pending.json
16. Agent读取它、删除它
17. Agent运行：$D variants --brief "完全不同的方向" --count 3
    （一次一个，非并行）
18. Agent运行：$D compare --images "new-A.png,new-B.png,new-C.png" --output board-v2.html
19. Agent POST：curl -X POST http://127.0.0.1:54321/api/reload -d '{"html":"/path/board-v2.html"}'
20. 服务器将 htmlContent 切换到新画布
21. 服务器状态 → "serving"（从reloading）
22. 画布的下一个 /api/progress 轮询返回 {"status":"serving"}
23. 画布自动刷新：window.location.reload()
24. 用户看到包含3个新变体的新画布
25. 用户选择一个、点击"提交" → 从步骤10开始的快乐路径
```

### "更多类似"路径

```
与重新生成相同，除了：
- regenerateAction 是 "more_like_B"（引用变体）
- Agent使用 $D iterate --image B.png --brief "更多类似这样的，保持间距"
  代替 $D variants
```

### 回退路径：$D serve 失败

```
1. Agent尝试 $D compare --serve，失败（二进制缺失、端口错误等）
2. Agent回退到：open file:///path/board.html
3. Agent使用 AskUserQuestion："我已打开设计画布。
   您更喜欢哪个变体？有什么反馈？"
4. 用户在文本中响应
5. Agent继续处理文本反馈（无结构化JSON）
```

## 实现此功能的文件

| 文件 | 角色 |
|------|------|
| `design/src/serve.ts` | HTTP服务器、状态机、文件写入、浏览器启动 |
| `design/src/compare.ts` | 画布HTML生成、评分/选择/重新生成JS、POST逻辑、提交后生命周期 |
| `design/src/cli.ts` | CLI入口，连接 `serve` 和 `compare --serve` 命令 |
| `design/src/commands.ts` | 命令注册表，定义 `serve` 和 `compare` 及其参数 |
| `scripts/resolvers/design.ts` | `generateDesignShotgunLoop()` — 输出轮询循环和重新加载指令的模板解析器 |
| `design-shotgun/SKILL.md.tmpl` | 编排完整流程的技能模板：上下文收集、变体生成、`{{DESIGN_SHOTGUN_LOOP}}`、反馈确认 |
| `design/test/serve.test.ts` | HTTP端点和状态转换的单元测试 |
| `design/test/feedback-roundtrip.test.ts` | E2E测试：浏览器点击 → JS fetch → HTTP POST → 磁盘上的文件 |
| `browse/test/compare-board.test.ts` | 对比画布UI的DOM级测试 |

## 仍然可能出错的地方

### 已知风险（按可能性排序）

1. **Agent不遵循顺序生成规则** — 大多数LLM想要并行化。二进制中没有强制执行，这是可能被忽略的提示级指令。

2. **Agent丢失端口号** — 上下文压缩丢弃stderr输出。Agent无法重新加载画布。缓解措施：将端口写入文件。

3. **陈旧的反馈文件** — 崩溃会话残留的 `feedback-pending.json` 使下次运行混淆。缓解措施：启动时清理。

4. **fs.writeFileSync崩溃** — 反馈文件写入时没有try/catch。磁盘满时静默服务器死亡。用户看到无限旋转器。

5. **进度轮询漂移** — `setInterval(fn, 2000)` 持续5分钟。实际上JavaScript定时器足够准确。但如果浏览器标签页后台运行，Chrome可能将间隔限制为每分钟一次。

### 运行良好的特性

1. **双通道反馈** — stdout用于前台模式，文件用于后台模式。两者始终活跃。Agent可使用任一工作模式。

2. **自包含HTML** — 画布内联了所有CSS、JS和base64编码的图片。无外部依赖。离线工作。

3. **同标签页重新生成** — 用户停留在一个标签页。画布通过 `/api/progress` 轮询+ `window.location.reload()` 自动刷新。无标签页爆炸。

4. **优雅降级** — POST失败显示可复制JSON。进度超时显示清晰的错误消息。无静默失败。

5. **提交后生命周期** — 提交后画布变为只读。无僵尸表单。清晰的"下一步做什么"消息。

## 测试覆盖

### 已测试内容

| 流程 | 测试 | 文件 |
|------|------|------|
| 提交 → 磁盘上feedback.json | 浏览器点击 → 文件 | `feedback-roundtrip.test.ts` |
| 提交后UI锁定 | 输入禁用，显示成功 | `feedback-roundtrip.test.ts` |
| 重新生成 → feedback-pending.json | 选项+重新生成点击 → 文件 | `feedback-roundtrip.test.ts` |
| "更多类似"→ 特定动作 | more_like_B在JSON中 | `feedback-roundtrip.test.ts` |
| 重新生成后旋转器 | DOM显示加载文本 | `feedback-roundtrip.test.ts` |
| 完整重新生成 → 重新加载 → 提交 | 2轮往返 | `feedback-roundtrip.test.ts` |
| 服务器在随机端口启动 | 端口0绑定 | `serve.test.ts` |
| 服务器URL的HTML注入 | __GSTACK_SERVER_URL检查 | `serve.test.ts` |
| 拒绝无效JSON | 400响应 | `serve.test.ts` |
| HTML文件验证 | 缺失时退出码1 | `serve.test.ts` |
| 超时行为 | 超时后退出码1 | `serve.test.ts` |
| 画布DOM结构 | 单选框、星级、选项 | `compare-board.test.ts` |

### 未测试内容

| 缺口 | 风险 | 优先级 |
|-----|------|----------|
| 双击提交竞态 | 低 — 第一次响应时输入禁用 | P3 |
| 进度轮询超时（150次迭代） | 中 — 测试中等待5分钟太长 | P2 |
| 重新生成期间服务器崩溃 | 中 — 用户看到无限旋转器 | P2 |
| POST期间网络超时 | 低 — localhost速度很快 | P3 |
| 后台Chrome标签页节流间隔 | 中 — 可能将5分钟超时延长到30+分钟 | P2 |
| 大反馈负载 | 低 — 画布构造固定形状JSON | P3 |
| 并发会话（两个画布，一个服务器） | 低 — 每个 $D serve 有自己的端口 | P3 |
| 先前会话的陈旧反馈文件 | 中 — 可能混淆新轮询循环 | P2 |

## 潜在改进

### 短期（此分支）

1. **将端口写入文件** — `serve.ts` 启动时将 `serve.port` 写入磁盘。Agent可随时读取。5行。
2. **启动时清理陈旧文件** — `serve.ts` 开始前删除 `feedback*.json`。3行。
3. **保护双击** — 在 `handleFeedback()` 顶部检查 `state === 'done'`。2行。
4. **try/catch文件写入** — 将 `fs.writeFileSync` 包装在try/catch中，失败时返回500。5行。

### 中期（后续跟进）

5. **WebSocket替代轮询** — 用WebSocket连接替换 `setInterval` + `GET /api/progress`。画布在新HTML就绪时获得即时通知。消除轮询漂移和后台标签页节流。serve.ts中约50行+ compare.ts中约20行。

6. **Agent的端口文件** — 启动时将 `{"port": 54321, "pid": 12345, "html": "/path/board.html"}` 写入 `$_DESIGN_DIR/serve.json`。Agent读取此内容而非解析stderr。使系统对上下文丢失更健壮。

7. **反馈模式验证** — 写入前根据JSON模式验证POST体。尽早捕获格式错误的反馈，而非让Agent下游困惑。

### 长期（设计方向）

8. **持久设计服务器** — 不必每次会话都启动 `$D serve`，而是运行一个长生命周期的设计守护进程（类似浏览器守护进程）。多个画布共享一个服务器。消除冷启动。但增加守护进程生命周期管理复杂性。

9. **实时协作** — 两个Agent（或一个Agent+一个人）同时在同一画布上工作。服务器通过WebSocket广播状态变化。需要解决反馈冲突。
