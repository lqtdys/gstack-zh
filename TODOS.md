# 待办事项

## 下一优先级

### P1: #1882 — 可移植的技能安装前缀（非 `gstack` 安装目录静默失败）

**内容：** 每个生成的 SKILL.md 都硬编码了字面路径 `~/.claude/skills/gstack/...`，用于其 `bin/`/assets 调用（每次调用的遥测/配置前置块加上约 9 个解析器）。`setup` 会为任何目录名称连接顶层技能符号链接，所以安装在 `~/.claude/skills/<其他>` 下会让所有内部 `bin` 引用指向不存在的 `~/.claude/skills/gstack/` 路径 — 在技能调用时**静默失败**。使发出的引用具备可移植性：在运行时解析安装根目录（前置块脚本已在 `scripts/resolvers/preamble/generate-preamble-bash.ts` 中定义了 `GSTACK_ROOT`/`GSTACK_BIN`，但字面量没有使用它们）并发出 `$GSTACK_BIN` 相对路径代替硬编码前缀。

**原因：** 记录为 #1882。从 2026 年 6 月修复浪潮（决策 A）中分离出来，因为实现显示这是一个主机配置/设计变更，而非修复浪潮补丁。紧急的一半 — 在 CC 2.1.162 上损坏的保护/冻结/谨慎前置钩子 — 已在该浪潮中修复（#1871），使用字面 `$HOME` 锚定路径，因为前置钩子在任何运行变量存在之前运行，无法使用 `$GSTACK_BIN`。所以 #1882 现在纯粹是前置块可移植性工作。

**优点：** 解锁在任何目录名称下的安装；消除一整类调用时静默失败。
**缺点：** 触及仓库中最承重的每个技能前置块；一个静默错误会破坏所有 52 个技能。高爆炸半径 — 需要独立聚焦的 PR。

**上下文 / 起始位置：**
- 重新接线 `ctx.paths.binDir`（以及浏览/设计目录路径）+ 发出字面量的约 9 个解析器（`testing.ts`、`review.ts`、`design.ts`、`browse.ts`、`redact-doc.ts`、`tasks-section.ts`、`preamble/generate-*.ts`），改为使用前置块定义的 `$GSTACK_ROOT`/`$GSTACK_BIN`。
- 确保 `GSTACK_ROOT`/`GSTACK_BIN` 在每个技能的前置块中首次使用**之前**已定义（验证遥测前置块首次 bin 调用在定义之后）。
- **测试冲突（已验证）：** `test/gen-skill-docs.test.ts:1942` 和兄弟 ship 断言当前*断言*生成的 Claude 输出 `.toContain('~/.claude/skills/gstack')` 作为 Codex 主机路径泄漏的防护栏。这些必须重写以匹配新的可移植方案。
- 重新生成所有 52 个 SKILL.md（`bun run scripts/gen-skill-docs.ts --host all`）；永远不要手动编辑生成文件。二分：解析器/主机配置变更提交，然后是 52 文件重新生成提交。
- 冒烟测试从非 `gstack` 安装目录调用技能以证明修复有效。
- 与 #349（`$CLAUDE_CONFIG_DIR` / `~/.claude` 路径问题）的兄弟项。

## 测试基础设施

### Eval 工具：实时进度 + 增量结果持久化（消除静默一小时）

**优先级：** P1

**内容：** `bun run test:evals` 在其整个运行期间明显静默，并且在完成之前不持久化任何内容。使 E2E 工具做到 (1) 每次测试开始和结束时将单行进度记录追加到知名心跳文件（例如 `~/.gstack-dev/evals/.current-run.jsonl`），(2) 增量写入每个测试的 eval-store 结果而非仅在运行结束时写入，(3) 将每次测试的通过/失败行刷新到 stderr 并无缓冲，以便 `bun test --concurrent` 巨型文件缓冲无法隐藏 50 分钟的合法进度。

**原因：** 在 v1.57.11.0 发布期间，diff 选择的 eval 运行（54 个测试）在大约 50 分钟时被杀死，尸体与健康运行数小时内没有任何区别：日志零测试行（在五个巨型 `skill-e2e-*.test.ts` 文件中文件间缓冲），`~/.gstack-dev/evals/` 零新文件（结果仅在完成时持久化），唯一可用的活跃信号（`pgrep "bun test --max-concurrency"`）在每个兄弟空闲套件分片上误报。观察运行的代理或人类没有诚实的信号。

**优点：** 死亡运行在分钟内而不是数小时内被检测；部分结果在杀死中存活（在第 40/54 个测试死亡的 50 分钟运行保留 40 个结果并可以恢复）；`eval:watch` 获得真实数据源。

**缺点：** 触及 `test/helpers/session-runner.ts` + `eval-store.ts`（全局触达文件 — 变更触发下一次 diff 选择运行上的所有 eval 测试）；增量写入需要 PARTIAL 标记，以便 `eval:compare` 不会将死亡运行视为完整基线。

**上下文：** 2026-06-12 期间 v1.57.11.0 /ship 的根因分析。运行本身在进度上（54 个 E2E 测试在并发 15 下约 50 分钟是标称值）；故障完全是可观察性。相关的：已有的 `project_e2e_harness_observability` 注释（stream-json 推理 + 失败时丢弃工具跟踪 — 同一模块，一起修复）。从 `test/helpers/session-runner.ts`（每测试生命周期）和 `test/helpers/eval-store.ts`（持久化时间）开始。

**依赖/阻塞：** 无。新行为分类到现有的两层系统下；心跳文件必须在 `--concurrent` 下安全（追加一次，每事件一行 JSON）。

### ✅ 已完成（v1.3.1.0）：重新基准化 parity 套件（v1.44.1 → v1.53.0.0）

**内容：** `test/parity-suite.test.ts` 对照冻结的 `test/fixtures/parity-baseline-v1.44.1.json` 检查每个技能 SKILL.md 的大小。五个规划技能超过了 1.05 倍上限：`plan-ceo-review`（1.052）、`plan-eng-review`（1.062）、`plan-design-review`（1.068）、`investigate`（1.053）、`office-hours`（1.065）— 增长来自 brain-aware 规划发布（v1.49–v1.52）加上 v1.53 编辑保护。

**已解决：** HEAD 通过 `bun run scripts/capture-baseline.ts --tag v1.53.0.0` 捕获新基线并将测试重新指向 `test/fixtures/parity-baseline-v1.53.0.0.json`。每技能 1.05 比率保持，所以未来的膨胀仍被捕获 — 只有陈旧的锚点移动。镜像了之前的 `skill-size-budget` 重新基准化（v1.44.1 → v1.47.0.0）。历史 v1.44.1 / v1.46.0.0 / v1.47.0.0 基线保留在 `test/fixtures/` 中用于 v1→v2 审计跟踪。捕获的技能字节与 `origin/main` 完全匹配（重新基准化分支未改变任何 SKILL.md）。`bun test` 再次变绿。

## Token 缩减后续（Phase B，通过 /plan-eng-review 在 plan-ceo-review carve 上提交）

### P3：将总是加载的 `{{PREAMBLE}}` 引用块切割为按需文档

**内容：** 每技能部分切割（`/ship` v1.54，`/plan-ceo-review` v1.56）产生真实但有界的胜利（切割技能 -42% 到 -59%），因为共享的 `{{PREAMBLE}}`（每个 tier-3/4 技能约 40-50KB）是主要的总是加载成本并保持内联。将很少需要的前置块参考块（AskUserQuestion 拆分规则和 CJK / 孤替代转义参考）移动到文档的部分式文档中，仅当代理遇到这些边缘情况时才读取，保持热路径（语音、完整性原则、推荐格式）内联。

**原因：** 最高的 ROI 剩余 token 目标。一次前置切割立即帮助每个 tier≥2 技能，而非一个技能一个 PR。eng-review 在 plan-ceo 切割上标记称每技能切割保持适度正是因为前置块主导总是加载表面。

**优点：** 一次变更减少整个技能包的总是加载成本。
**缺点：** 前置块是承重的且共享的；错误切割会回归每个技能。需要与部分切割使用的相同的并集-parity + 每推送新鲜度保护，应用于整个语料库。

**上下文：** 基于 v2 部分管道（`scripts/resolvers/sections.ts`，`{{SECTION:id}}` / `{{SECTION_INDEX}}`）。前置块源是 `scripts/resolvers/preamble.ts`。测量哪些子块是冷的（转义参考、拆分规则）对比热的（语音、推荐格式）再切割。在一个技能上验证，然后推广到整个语料库。

**工作量估算：** L（人类团队）→ M（CC+gstack）
**优先级：** P3
**依赖/阻塞：** 部分管道（已发布 v1.54）。无硬性阻塞。

## gbrowser 内存后续（通过 /plan-eng-review + /codex 在 v1.49 泄漏修复 PR 上提交）

这四个项目来自泄漏调查，该调查发布了 `$B memory` 诊断 + 四个泄漏修复。它们被有意推迟（已 14 个提交 / ~12 个文件）；每个都可以独立发布。

### P2：MV3 扩展服务工作者内存配置

**内容：** `/memory` 端点快照枚举页面但不会举 gstack 内置扩展的服务工作者目标。长期运行的 MV3 服务工作者可以通过保留的 DOM 快照、永远不关闭的消息端口、重新装备的警报和无界增长的缓存来泄漏。诊断应使用 `Target.getTargets` 过滤 `service_worker` 并将每个包含在 `tabs[]` 中（或兄弟 `serviceWorkers[]` 数组），包含相同的 `Performance.getMetrics` 数据。

**原因：** Codex 在 eng-review 上的外部声音评论标记了这类泄漏（扩展是 gbrowser 进程树的一部分但对今天快照不可见）。在我们呈现它之前，SW 泄漏仅在父进程 RSS 中出现，没有每目标归因。

**优点：** 关闭单最可能的未来泄漏源（我们自己的扩展）的每目标归因差距。
**缺点：** 扩展 SW 生命周期与页面生命周期不对称；自动附加 + 过滤是另一段 CDP 管道。

**上下文：** Codex 发现 eng-review 外部声音 #4。不在 v1.49 PR 范围内；有意推迟以保持 PR 为四个最高置信度泄漏修复。

**优先级：** P2。**工作量：** M。

---

### P2：`$B memory` 中的原生 + GPU 内存分解

**内容：** `$B memory` 显示 Bun RSS + 每标签 JS 堆 + Chromium 进程树（PIDs + 类型 + CPU 时间）但每进程 RSS 不存在 — `SystemInfo.getProcessInfo` 不暴露 RSS 且 eng 审查（D2 USE_CDP）明确选择了 CDP 而不是 shell 到 `ps`。诚实的下一步是呈现 CDP 对其他内存类别的给予：每目标 `Memory.getDOMCounters`（节点 + 监听器计数），`SystemInfo.getInfo` 用于 GPU 内存，`Memory.getAllTimeSamplingProfile` 用于采样原生估计。

**原因：** Codex 的外部声音审查标记了 `Performance.getMetrics` 遗漏原生内存、GPU 内存、视频缓冲区、Skia、网络缓存、扩展进程 RSS 和浏览器进程 RSS — 所有 160 GB 泄漏实际存在的类别。错过泄漏类别的诊断低估了自己。

**优点：** 每进程类别分解关闭了"活动监视器说 160 GB"和诊断显示之间的差距。
**缺点：** 每个 CDP 方法都有自己的怪癖；这是一个真正的实现过程，不是一行添加。

**上下文：** Codex 发现 eng-review 外部声音 #5。不在 v1.49 PR 范围内；有意推迟。

**优先级：** P2。**工作量：** M。

---

### P3：Network.loadingFinished 的单上下文 CDP 监听器

**内容：** `wirePageEvents` 附加一个 `page.on('requestfinished')` 监听器每页面。D10 修复移除了该监听器体内的物化泄漏但保持了每页面监听器架构（每个标签附加 7 个监听器 — close、framenavigated、dialog、console、request、response、requestfinished）。D10 的延伸目标是将每页面 `requestfinished` 监听器替换为通过 `Target.setAutoAttach({autoAttach: true, waitForDebuggerOnStart: false, flatten: true})` 和浏览器范围 `Network.loadingFinished` 事件处理器的单上下文 CDP 监听器。

**原因：** 从 N 到 1 监听器用于请求大小捕获是从结构上正确的架构并移除每标签内存压力的一部分。物化修复已经解决了急性泄漏；这是防止同类类似泄漏的架构清理。

**优点：** 浏览器一个监听器而不是一标签一个。
**缺点：** `Target.setAutoAttach` 管道比直接每页面监听器代码更多；在已落地物化修复之上的边际内存增益很小。

**上下文：** D10 延伸目标在 eng-review。最小风险修复在 v1.49 发布（用 `await req.sizes()` 替换 `await res.body()`，保留每页面监听器）；这是架构后续。

**优先级：** P3。**工作量：** M-L。

---

### P3：真正的 Chromium 峰值 RSS 复现器（period 层）

**内容：** 门层复现器（`browse/test/memory-leak-reproducer.test.ts`）固定了不变量：在 `requestfinished` 事件突发期间从不调用 `res.body()`。它使用假页面；它不会启动真正的 Chromium，也不会测量真正并发获取突发期间的峰值 Bun RSS。period 层后续应：启动一个真正的无头 Chromium，导航到并发获取 500 个混合响应（小 JSON、100 KB 图片、10 MB 分块、gzip 压缩 2 MB）的固定页面，每 100 ms 在突发期间采样 `process.memoryUsage().heapUsed`，断言 `peak_heap < baseline 以上 200 MB` 且 `post-gc_heap < baseline 以上 30 MB`。还包括增长到 >4 GB 的单标签 WebGL 变体验证每标签 RSS 提示触发。

**原因：** Codex 标记称泄漏的真正故障模式是并发突发下的瞬态放大，不是保留泄漏 — 稳态堆测试错过了它。假页面门层测试捕获监听器架构回归；period 的真正浏览器测试捕获实际峰值-RSS 类。

**优点：** 用硬数字关闭"我们是否真正演示了 OOM 已修复"的问题。供给 ANGLE_B_NUMBERS CHANGELOG 发布摘要表。
**缺点：** period 层每次运行花费数分钟 CI 时间和金钱；真正的浏览器内存测试固有地不稳定。

**上下文：** Codex 外部声音发现 eng-review；D7 ANGLE_B_NUMBERS CHANGELOG 框架在 /ship 之前需要此复现器的数字。

**优先级：** P3。**工作量：** M。

---

## design daemon：后续（v1.45.0.0 通过 /ship 审查团提交）

### ✅ 已完成（v1.45.0.0）：收紧 daemon 测试覆盖

**在提交 `6b037c55` 中解决（同一 PR）：** 所有 5 个测试差距在落地前填补。之后每文件总数：serve 16、daemon 34、daemon-discovery 23、feedback-roundtrip-daemon 4 = 77（从初始发布 +10）。具体：
- 空闲关闭实际触发（基于生成，观察到 daemon 进程退出，状态文件移除）
- 裸 GET 轮询不重置空闲（在后台锤击 `/api/progress`，daemon 仍然空闲超时）
- 具有活跃面板的空闲扩展，然后在 MAX_EXTENSIONS 后强制关闭（`DESIGN_DAEMON_EXTENSION_MS=1500` + `MAX_EXTENSIONS=2`）
- 并发 `ensureDaemon()` 竞赛收敛到一个 daemon（锁胜出）
- 陈旧锁回收（死 PID 成功，活无关 PID 拒绝）
- `POST /api/boards` 和 `POST /boards/<id>/api/reload` 的畸形 JSON + 非对象 + 数组体 + 缺少 html 否定情况

### P3：来自 /ship 审查的次要可维护性微调

- `design/src/cli.ts` 和 `design/src/serve.ts` 都有一个小的 `openBrowser` 辅助函数，具有相同的 darwin/linux/else 分支。提取共享的 `design/src/open-browser.ts`。
- `design/src/daemon-client.ts:320`（`AbortSignal.timeout(2000)`）和 `:357`（`delay(50)`）使用裸数字字面量，而兄弟超时是命名常量。提升为 `SHUTDOWN_POST_TIMEOUT_MS` 和 `ALIVE_POLL_INTERVAL_MS`。
- `design/src/daemon-state.ts:21` `serverPath` 字段被写入（`daemon.ts:541`）但从未被生产代码读取。移除或记录取证意图。

### P3：从 v1.45.0.0 计划推迟的 Daemon 范围

最初列在计划的"TODOs surfaced for later"部分：

- 每 daemon 范围鉴权令牌（仅在隧道/共享用例出现时相关）
- `~/.gstack/projects/$SLUG/designs/history/` 中可选的持久面板历史，以便已提交的面板在 daemon 重启后存活
- 从 browse 提升的 Windows 分支（V1 daemon 是 macOS + Linux；Windows 用户回退到传统 `--no-daemon` 每进程服务器）
- `$D board list` / `$D board stop <id>` 每面板操作 CLI（V1 仅有 `$D daemon status` / `stop`）
- 跨 worktree daemon attach（同一仓库的导体兄弟 worktree 当前各自生成自己的 daemon — 与 browse 匹配；如果引起摩擦则重新考虑）

---

## browse server：终端代理清理后续（v1.41 通过 /plan-eng-review 提交）

### ✅ 已完成（v1.44.0.0）：基于身份的终端代理杀死（用 PID 替换 pkill 正则）

**已解决：** 打包到 v1.44.0.0 长寿命侧边栏 PR 中作为提交 0。`browse/src/terminal-agent-control.ts` 是 `readAgentRecord`、`writeAgentRecord`、`clearAgentRecord` 和 `killAgentByRecord` 的新主页。代理在启动时写入 `<stateDir>/terminal-agent-pid`（JSON `{pid, gen, startedAt}`），在 SIGTERM/SIGINT 时清除它。`cli.ts` 和 `server.ts` 都通过 `killAgentByRecord` 而不是 `pkill -f terminal-agent\\.ts`。新的 `browse/test/terminal-agent-pid-identity.test.ts` 是静态 grep 绊线，如果 `pkill ... terminal-agent` 或 `spawnSync('pkill', ...)` 重新出现在任何源文件则 CI 失败。

---

### P3：shutdown() 读取模块级 `config`，不是 `cfg.config`（组合差距）

**内容：** `browse/src/server.ts:shutdown()` 读取 `path.dirname(config.stateFile)`，其中 `config` 是导入时解析的模块级值，不是传入 `buildFetchHandler` 的 `cfg.config`。同样的差距适用于 server.ts:1298 的 `cleanSingletonLocks(resolveChromiumProfile())` — 应读取 `cfg.chromiumProfile`。

**原因：** 今天的嵌入者恰好与 CLI 共享状态目录解析（两者都通过 `resolveConfig()` 针对相同环境），所以这不咬人。但如果嵌入者传递不同的 `cfg.config`（例如，指向临时目录的测试工具），关闭将操作错误路径。`ownsTerminalAgent` 标志暴露了问题而没有修复它。

**优点：** 正确关闭嵌入者组合故事。与 `cfg.chromiumProfile` 配对，给出一致的"此工厂拆除尊重 cfg"契约。

**缺点：** 先前存在的 — 不是回归。今天两个调用站点（1285 用于终端文件，1298 用于 chromium 锁）。将 `cfg.config` 和 `cfg.chromiumProfile` 穿入正确的闭包是直接的但比 v1.41 修复更广。

**上下文：** 由 Codex 和 Claude 子代理在 /plan-eng-review 双声音中标记。记录为 v1.41 计划中的范围外；与发送给 gbrowser 团队的 `chromiumProfile` PR 正文注释相同形状。

**依赖：** 无。

---

### P3：如果出现第 4 个调用者拥有的拆除门，所有权对象重构

**内容：** 今天 `ServerConfig` 有三个调用者拥有的拆除门：`xvfb?`（存在 ⇒ 不关闭）、`proxyBridge?`（相同）和 `ownsTerminalAgent`（显式布尔值）。如果出现第 4 个门，折叠为 `cfg.callerOwns?: Set<'terminalAgent' | 'xvfb' | 'proxyBridge' | ...>` 或类似。

**原因：**三个独立标志低于重构阈值 — 每个字段具有清晰、独特的语义且 JSDoc 声音一致。第四个倾斜成本平衡：每字段表面变得嘈杂，"这个工厂拥有什么？"变成你必须问三个或四个分散字段而不是一个显式集合的问题。

**优点：** "gstack 拆除什么"的真相来源单一。未来调用者拥有的资源的简单扩展表面。测试中断言更容易（"集合应包含 X，不是 Y"）。

**缺点：** 今天过早。`ownsTerminalAgent` JSDDoc 中的极性反转注释只有一点伤害 — 这是一个异常，不是模式。现在重构为所有权对象会触及每个嵌入者。

**上下文：** Claude 子代理在 /plan-ceo-review 双声音（自动计划）期间推荐。触发：在此相同 `ServerConfig` 形状中的第 4 个调用者拥有的拆除门。

**依赖：** 激励重构的第 4 个门。

---

## /sync-gbrain 内存阶段性能后续

### P2：调查 `gbrain import` 在大暂存目录上的性能

**内容：** 在 5131 个文件的暂存目录上的冷运行时间 >仅 `gbrain import` 中 10 分钟（在 gstack 的准备阶段之后，现在在删除每文件 gitleaks 后 <10s）。在 501 个文件上它花了 10s。缩放比线性更差且瓶颈在 gbrain 内部，不在 gstack 编排器内。

**原因：** 内存摄取的准备阶段现在很快，剩余的冷运行成本完全在 gbrain 用户上。拥有大语料库（5K+ 文件）的用户当前在首次摄取上支付约 15-30 分钟。`~/git/gbrain/src/core/import-file.ts` 中的可能罪魁祸首：

- N+1 SQL 查询：`engine.getPage(slug)` 用于每个文件的 content_hash 检查（行 242 + 478）— 应批处理为单个查询
- 即使对于未更改的内容也会触发的每页自动链接协调
- 没有批处理事务的 FTS / 向量索引更新

**优点：** 位于 gbrain 中（更清晰的分离）。在 gbrain 中修复也惠及其他 gbrain 调用者（`gbrain sync`、MCP `put_page` 工作流）。仅从批处理查询可能有 10-50x 加速。

**缺点：** 跨仓库变更，需要 gbrain 测试覆盖新的批处理路径。不在 gstack 关键路径上；gstack 的架构已正确。

**上下文：** 2026-05-10 在真实语料库上验证。准备阶段 gstack 端带 `--scan-secrets` 关闭在 <10s 内运行。同一暂存目录上的完整 gbrain import 占用 100% CPU >10 分钟。两个观察都来自 `bin/gstack-memory-ingest.ts:ingestPass` 快速到达 `runGbrainImport` 调用，然后子进程占用大部分挂钟时间。

**依赖：** 无 — gstack 的批处理摄取架构（`docs/designs/SYNC_GBRAIN_BATCH_INGEST.md` 中的 D1-D8）已正确发布。

---

### P3：在准备批处理级别缓存"自上次导入以来无变化"

**内容：** 即使在准备阶段快速（5135 个文件 <10s），在真正的无操作运行上遍历和 mtime 状态每个文件添加几秒并创建虚假暂存目录。在状态文件中缓存每个源的最新 mtime；如果没有源目录具有更新的 mtime，跳过遍历 + 暂存 + 导入。

**原因：** 大多数 `/sync-gbrain` 调用没有新内容要摄取。最快路径是"什么也不做，快。" `gbrain doctor` 仍应报告状态，但实际摄取流水线可以在 last_full_walk 最近且没有源树 mtime 移动时短路。

**优点：** 微不足道的实现（在 `ingestPass` 中约 20 行）。使增量快速路径真正达到原始计划中"<30s"。

**缺点：** 增加缓存无效化表面。如果用户编辑文件但父目录的 mtime 不更新（在 macOS APFS 上罕见），更改会错过。缓解：仅在 last_full_walk 最近（例如 <1 分钟前）时短路。

**上下文：** 在 2026-05-10 性能测试后 `--scan-secrets` 变为选择加入后提交。优先级低于上述 gbrain 端性能问题。

---

## 浏览器技能后续（阶段 2-4）

### P1：浏览器技能阶段 2 — `/scrape` 和 `/skillify` 技能模板

**内容：** 浏览器技能设计的阶段 2a（`docs/designs/BROWSER_SKILLS_V1.md`）。两个新的 gstack 技能：`/scrape <intent>`（只读）是拉取页面数据的单一入口点 — 首次调用通过 `$B` 原型的原型，后续匹配意图的调用路由到约 200ms 的编码浏览器技能。`/skillify` 将最近成功的原型编码为磁盘上的永久浏览器技能：从代理自己的上下文（仅最终尝试的 $B 调用）综合 `script.ts` + `script.test.ts` + 固定装置，在临时目录中运行测试，在提交前询问，原子重命名为 `~/.gstack/browser-skills/<name>/`。突变流兄弟 `/automate` 分离为自己的 P0（下面）— 相同的 skillify 模式，不同的信任配置文件。

**原因：** 阶段 1 发布了运行时 — 人类可以编写 gstack 运行的确定性浏览器脚本。阶段 2a 解锁生产力增益：一次通过 20+ $B 命令获得流程正确的代理说 `/skillify` 并且脚本在以后变成 200ms 调用。相同的 skillify 模式加里的文章描述，应用于最易受确定性压缩的只读浏览器活动（抓取）。突变操作作为 `/automate` 下一个发布，因为故障模式（意外写入）需要更强的门。

**优点：** 100x 生产力增益在这里。关闭循环：代理原型、编码，然后在未来会话中达到编码技能而不是重新探索。替换原始的"自创作 `$B` 命令"P1 — 相同的用户可见目标，没有守护进程内隔离问题（技能脚本作为独立 Bun 进程运行，从不导入到守护进程）。综合问题（Codex 发现 #6）通过从代理自己的对话上下文重新提示（设计文档中的选项 b）解决，按 `/plan-eng-review` D2 限制为最终尝试的 $B 调用。

**缺点：** **Bun 运行时分发**（Codex 发现 #7）。阶段 1 避开了这一点，因为捆绑的参考技能在 gstack 安装内发布。用户创作的技能落在没有 Bun 的机器上，除非我们随同发布运行时、编译为自包含二进制或使用 Node + 现有 `cli.ts` 模式。推迟到阶段 4 — `/skillify` 记录了 gstack 已安装的假设（这意味着 Bun 在 PATH 上）。

**上下文：** 阶段 1 架构（3 层查找、范围令牌、兄弟 SDK、前置契约）已锁定并由捆绑的 `hackernews-frontpage` 参考技能练习。阶段 2a 通过两个技能模板 + 一个新辅助函数（`browse/src/browser-skill-write.ts` 用于按 `/plan-eng-review` D3 原子临时目录然后重命名）将 `/scrape` 和 `/skillify` 插入该运行时 — 无新存储原语。

**工作量：** M（人类：约 1 周 / CC：约 1 天）
**优先级：** P1（此分支 — `garrytan/browserharness` 发布为 v1.19.0.0）
**依赖：** 阶段 1 已发布（此分支）。

---

### P2：浏览器技能阶段 3 — 在会话开始时的解析器注入

**内容：** 镜像 `browse/src/server.ts:722-743` 的域技能解析器。当侧边栏代理会话在具有匹配浏览器技能的主机上启动时，注入列表块告诉代理该主机存在哪些技能以及如何调用它们（`$B skill run <name> --arg ...`）。通过现有 L1-L6 安全堆栈进行 UNTRUSTED 包装。添加 `gstack-config browser_skillify_prompts` 旋钮（默认 `off`），在 `/qa`、`/design-review` 等中当活动订阅显示在单个主机上 ≥N 个命令且该主机+意图尚无技能时控制结束任务提示。

**原因：** 没有解析器，浏览器技能仅在用户明确键入 `$B skill run <name>` 时工作。使用解析器，代理自动发现当前主机的现有技能并达到它们而不是重新探索。与域技能相同的复合模式。

**优点：** 关闭可发现性差距。不会知道技能存在的代理现在在其系统提示中自动看到它。结束任务提示（通过旋钮选择加入）在 skillify 最有价值的时刻触发。

**缺点：** 解析器块位于系统提示中并与其他解析器块竞争提示预算。需要仔细门控以便它不会在每个具有技能的主机上触发 — 仅当技能与当前任务合理相关时。v1.8.0.0 域技能通过仅为主机名的活动标签触发来处理这一点；这里相同模式。

**工作量：** S（人类：约 3 天 / CC：约 4 小时）
**优先级：** P2
**依赖：** 阶段 2。

---

### P2：浏览器技能阶段 4 — eval 基础结构 + 固定装置陈旧 + OS 沙箱

**内容：** 三个松散耦合的扩展：(a) LLM 判断 eval（"代理达到技能而不是重新探索了吗？"），按 `test/helpers/touchfiles.ts` 分类为 `periodic`。(b) 固定装置陈旧检测 — 定期比较捆绑的固定装置与实时页面，在它们悄然破坏测试之前标记不匹配。(c) 不受信任生成的 OS 级 FS 沙箱：macOS 上的 `sandbox-exec` 配置文件，Linux 上的命名空间 / seccomp。在现有受信任/不受信任契约后干净利落落入（阶段 1 只是剥离环境；阶段 4 添加真正的 FS 隔离）。

**原因：** 阶段 1 的信任模型在守护进程端能力边界正确（范围令牌）但进程端环境擦洗是卫生，不是沙箱（Codex 发现 #1）。对于真正不受信任的技能（阶段 2 代理创作），真正的 FS 隔离很重要。Eval + 固定装置陈旧保持技能质量标准诚实随着流漂移。

---

## 计划调优（v2 从 v0.19.0.0 回滚推迟）

所有六个项目都取决于 v1 结果和 `docs/designs/PLAN_TUNING_V0.md` 中的验收标准。它们在外 Codex 外部声音审查驱动 CEO EXPANSION 计划从范围回滚后被明确推迟。v1 仅发布观察基板；v2 添加行为适应。

### E1 — 基板接线（5 个技能消耗配置）

**内容：** 将 `{{PROFILE_ADAPTATION:<skill>}}` 占位符添加到 ship、review、office-hours、plan-ceo-review、plan-eng-review SKILL.md.tmpl 文件。实现具有每技能适应注册表的 `scripts/resolvers/profile-consumer.ts`（`scripts/profile-adaptations/{skill}.ts`）。每个消费者在前置块读取 `~/.gstack/developer-profile.json` 并适应技能特定默认值（冗长、模式选择、严重阈值、推回强度）。

**原因：** v1 观察配置文件编写一个没人读取的文件。基板声明仅在技能实际消耗它时变为现实。没有它，/plan-tune 是一个花哨的配置页面。

**优点：** gstack 感觉个性化。每个技能适应用户的转向风格而不是默认到中间路线。

**缺点：** 如果配置嘈杂，有心理测量漂移的风险。需要校准配置（v1 验收标准：跨 3+ 技能稳定 90+ 天）。

**上下文：** 参见 `docs/designs/PLAN_TUNING_V0.md` §Deferred to v2。v1 发布信号映射 + 推理计算；它在 /plan-tune 中显示但没有技能读取它。

**工作量：** L（人类：约 1 周 / CC: ~4h）
**优先级：** P0
**依赖：** **v1 90+ 天在 3+ 技能中稳定**（根据 `docs/designs/PLAN_TUNING_V0.md` §"Deferred to v2" E1 验收标准）。与 /plan-tune 用于渲染推理列使用的轻量级多样性显示门（`sample_size >= 20 AND skills_covered >= 3 AND question_ids_covered >= 8 AND days_span >= 7`）不同 — 显示是 UI 便利，E1 的晋升需要高得多的标准，因为行为适应是有后果且难以逆转的。此卡的先前版本引用了"2+ 周"，与 V0 冲突 — V0 胜出。

**基板风险（Codex 外部声音，Phase A 审查 2026-05-26）：** 生成的技能散文是基于代理合规的。测试可以验证模板包含正确的 `~/.gstack/developer-profile.json` 读取和正确的决策点，但测试无法证明代理在运行时遵守它们。E1 将适应作为 AskUserQuestion 推荐上的**建议注释**发布（"通过您的配置推荐：<选择>"），直到有硬运行时执行路径。在 E1 的 v1 中不要将任何 AUTO_DECIDE 仅基于推理配置；明确的每问题偏好仍然是唯一的 AUTO_DECIDE 来源。

### E3 — `/plan-tune narrative` + `/plan-tune vibe`

**内容：** 事件锚定叙事（"您接受了 7 次范围扩展，override test_failure_triage 4 次，称每个 PR'煮湖'"）+ 单字氛围原型（大教堂建设者、Ship-It 实用主义者、深工艺等）。scripts/archetypes.ts 已在 v1 中发布（8 个原型 + Polymath 回退）。v2 工作是叙事生成器 + /plan-tune 技能接线。

**原因：** 使配置有形且可共享。可截图。

**优点：** 杀手乐趣功能。gstack 的社交表面。具体的、锚定在真实事件中的输出（不是通用的 AI slop）。

**缺点：** 需要稳定推理的配置 — 没有校准它产生通用段落。Gen-tests 需要验证 no-slop。

**上下文：** 原型已定义。只需要 /plan-tune 叙事子命令 + slop-check 测试。

**工作量：** S+（人类：约 1 天 / CC: ~1h）
**优先级：** P0
**依赖：** 校准配置（>= 20 事件、3+ 技能、7+ 天跨度）。

### E4 — 盲点教练

**内容：** 前置块注入，在每会话每 tier ≥ 2 技能中呈现用户配置的相反面。煮湖用户在范围上受到挑战（"什么是 80% 版本？"）；小范围用户在野心上受到挑战。`scripts/resolvers/blind-spot-coach.ts`。会话去重的标记文件。通过 `gstack-config set blind_spot_coach false` 选择退出。

**原因：** 使 gstack 成为教练（挑战你）而不是镜子（反映你）。与设置菜单相比的杀手差异化。

**优点：** 让 gstack 感觉像加里的功能。呈现用户没有挑战的假设。

**缺点：** 逻辑上与 E1（适应配置）和 E6（标记不匹配）冲突。需要交互预算设计：全局会话预算 + 升级规则 + 从 mismatch 检测明确排除。如果触发错误感觉像唠叨。

**上下文：** v2 必须重新设计以解决 Codex 发现的 E1/E4/E6 组合问题。需要 Dogfood 来校准频率。

**工作量：** M（人类：约 3 天 / CC: ~2h 设计 + ~1h 实现）
**优先级：** P0
**依赖：** E1 已发布 + 交互预算设计规范。

### E5 — LANDED 庆祝 HTML 页面

**内容：** 当用户创作的 PR 新合并到基础分支时，在浏览器中打开动画 HTML 庆祝页面。彩纸 + 打字机标题 + 统计计数器。显示：我们构建了什么（PR 统计 + CHANGELOG 条目）、走过的路（来自 CEO 计划的范围决策）、未走的路（延期项目）、我们要去哪里（接下来的 TODO）、作为建设者的你是谁（氛围 + 叙事 + 此次 ship 的配置增量）。自包含的 HTML（仅 CSS 动画，无 JS 依赖）。

**来自 v0 计划的关键修订：** 被动检测不得位于前置块中（Codex #9）。晋升时，移动到显式的 `/plan-tune show-landed` OR post-ship 钩子 — 不在热路径中的被动检测。

**原因：** gstack 中最有个性的时刻。"让你记住你为什么构建这件事的单一事物"。

**优点：** 可截图。可分享。那种将高级用户变成传道者的多巴胺冲击。

**缺点：** 如果基板不坚固，产品戏剧。需要 /design-shotgun → /design-html 的视觉方向。需要 E2 统一配置用于叙事/氛围数据。

**上下文：** /land-and-deploy 信任/采用低，所以被动检测是正确的触发形状。每个 PR 在 `~/.gstack/.landed-celebrated-*` 中的去重标记。squash/merge-commit/rebase/co-author/fresh-clone/dedup 变体的 E2E 测试。

**工作量：** M+（人类：约 1 周 / CC: ~3h 总计）
**优先级：** P0
**依赖：** E3 叙事/氛围已发布。在真实 PR 数据上运行 /design-shotgun 以选择视觉方向，然后 /design-html 定稿。

### E6 — 基于声明 ↔ 推理不匹配的自动调整

**内容：** 当前 `/plan-tune` 显示声明和推理之间的差距（v1 观察）。当差距超过阈值时 v2 自动建议声明更新（"你的配置说你放手但你已 override 40% 推荐 — 你实际上是品味驱动声明的自主性从 0.8 到 0.5？"）。在任何变异之前需要明确的用户确认（Codex 信任边界 #15 已融入 v1）。

**原因：** 配置在没有纠正的情况下静默漂移。自我纠正的配置保持诚实。

**优点：** 配置随时间变得更准确。用户看到差距并决定。

**缺点：** 需要稳定推理的配置（多样性检查）。误报唠叨用户。

**上下文：** v1 有 `--check-mismatch` 标记 > 0.3 差距但不建议修复。v2 添加建议 UX + 来自真实数据的每维度阈值调优。

**工作量：** S（人类：约 1 天 / CC: ~45min）
**优先级：** P0
**依赖：** 校准配置 + v1 Dogfood 的真实不匹配数据。

### E7 — 心理测量自动决定

**内容：** 当推理配置已校准且问题是双向且用户的维度强烈倾向于一个选项时，不问自动选择（可见注释："通过配置自动决定。用 /plan-tune 更改。"）。v1 仅通过明确每问题偏好自动决定；v2 添加配置驱动的自动决定。

**原因：** 心理测量的全部要点。基于用户是谁，不只是他们说什么，静默、正确的默认值。

**优点：** 校准高级用户的无摩擦技能调用。随着时间推移，gstack 感觉像在读你的心思。

**缺点：** 最高风险推迟。错误的自动决定成本高。需要对信号映射 AND 校准门的高置信度。

**上下文：** v1 多样性门是 `sample_size >= 20 AND skills_covered >= 3 AND question_ids_covered >= 8 AND days_span >= 7`。v2 必须证明此门在发布前实际捕获嘈杂配置。

**工作量：** M（人类：约 3 天 / CC: ~2h）
**优先级：** P0
**依赖：** E1（技能消耗配置）+ 真实观测数据显示校准门值得信赖。

## 规划模式下的技术设计（v2 推迟）

### E2 — 规划模式智能的会话间隔离（已推迟）

**原因：** 跨会话复合需要首先使脑缓存迁移见 E3（跨机器）。先决条件未满足。

### E2.5 — 浏览器技能的自创作命令安全模型（来自 Codex T4 的修订）

**内容：** v1.8.0.0 引入了域技能作为笔记。下一步自然是代理创作 `$B` 命令（`scripts/generate-browser-command.ts`）。Codex 的外部声音（T4）标记了此功能创建的安全表面：代理生成的 JavaScript 在浏览器的信任进程内运行。需要：(a) 阻止代理作者访问特权 `$B` 语法（带范围的令牌模型应用于代理创作的命令），(b) 人类在首次调用时批准（DevEx D6），(c) 运行时沙箱（带命名空间的 Web Worker 或类似）。发布此功能前需要安全模型。

**工作量：** M（需要安全模型设计 + 实现）
**优先级：** P1（如果自创作命令是下一分支）/ P3（如果推迟）
**依赖：** 浏览器技能阶段 2a（已发布）；自创作命令架构决议

### E8 — 按钮对齐和模态冲突的启发式检测

**内容：** v1 创建了 `scripts/button-alignment-heuristics.ts` 来检测不对齐。扩展以检测模态冲突（垂直按钮行超出上下文宽度会折叠为堆叠布局，打破"一行中的可点击奖励"UX 规则）。启发式：如果按钮容器的 computed width > 其 offsetParent 的 60% 且包含 > 2 个按钮，标记为潜在冲突。

**工作量：** S（在现有 `button-alignment-heuristics.ts` 中添加 ~20 LOC）
**优先级：** P3（低，除非用户报告模态冲突）
**依赖：** v1 启发式文件（已发布）

## 进行中

### v1.8.0.0 — 域技能 + 浏览器技能 Codex 安全审查

**状态：** 进行中，PR 开放。Codex 外部声音审查完成了 v1.8.0.0 浏览器技能表面的独立挑战。12 个发现中，7 个已吸收（安全模型推迟到自创作命令、OS 沙箱到阶段 4等）。见 Codex 审查注释。

### v1.8.0.0 — 脑感知规划 #1

**状态：** 进行中。规划技能现在写入 gstack/skill-gbrain'ed 页面到脑（v1.50.0.0 发布的基础设施）。通过校准 take 锚定到真实事件的正在进行的工作。

<longcat_arg_value>
