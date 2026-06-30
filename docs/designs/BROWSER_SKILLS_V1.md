# 浏览器技能 v1 — 将重复的浏览器流程固化为代码

**状态：** 第 1 阶段已在 `garrytan/browserharness` 上发布。第 2-4 阶段枚举如下。
**最后更新：** 2026-04-26
**作者：** garrytan（+ /plan-eng-review 和 /codex 外部声音审查）

## 这是什么

浏览器技能是将重复的浏览器流程固化为确定性 Playwright 脚本的每任务目录。每个技能有：

```
browser-skills/<name>/
├── SKILL.md                        # frontmatter + 散文契约
├── script.ts                       # 确定性逻辑
├── _lib/browse-client.ts           # 供应商副本的 SDK
├── fixtures/<host>-<date>.html     # 捕获的测试页面
└── script.test.ts                  # 针对固定装置的解析器测试
```

用户（或在第 2 阶段，刚刚跑通流程的代理）创建一个技能。后续调用运行脚本，200ms 内返回 JSON 而不是
代理通过 `$B 原语重新探索所需的 30 秒。

已发布的参考是 `hackernews-frontpage`：抓取 HN 首页，
返回 30 个故事作为 JSON。试试 `$B skill list` 和 `$B skill run hackernews-frontpage`。

## 为什么这与领域技能（v1.8.0.0）不同

- **领域技能** = "代理记住网站的事实。" 按主机名键的 JSONL 笔记，在会话开始时注入提示。状态机处理隔离 → 活动 → 全局推广。
- **浏览器技能** = "代理将程序固化为确定性脚本。" 每任务目录，通过 `$B skill run` 执行，在守护进程上有按生成作用域隔离的 token。

两者使用相同的心理模型（每主机、三层作用域）。程序层是更大生产力收益所在，因为它将抓取和表单自动化推出潜在空间，进入可重现的代码。

## 为什么这不是现有的 P1（"自写 `$B` 命令"）

原始 P1 因 Codex 的 T1 反对而被阻止：代理编写的 TypeScript 不能安全地在守护进程内运行（ambient globals、构造器小工具、审批和执行之间的 top-level-await TOCTOU）。正确的设计是"带能力传递 IPC 的进程外工作者隔离。"这是一个可能永远无法发布的硬项目。

浏览器技能通过将脚本作为**独立的 Bun 进程**在守护进程**外**运行完全绕过了这个问题。守护进程从不导入或评估技能代码。技能通过环回 HTTP 与守护进程通信 —— 使用与任何外部客户端相同的线路格式。

批准的计划取代了现有的 P1。

---

## 分阶段

| 阶段 | 分支 | 范围 |
|-------|--------|-------|
| **1** | `garrytan/browserharness` | SDK、存储、`$B skill list/run/show/test/rm` 子命令、受限 token 模型、捆绑的 `hackernews-frontpage` 参考。**已发布（v1.19.0.0，与 Phase 2a 合并）。** |
| **2a** | `garrytan/browserharness`（继续） | `/scrape <意图>`（只读，带 match/prototype 路径的单一入口）+ `/skillify`（将原型固化为永久技能）。添加 `browse/src/browser-skill-write.ts` D3 原子写入帮助程序。**正在发布 v1.19.0.0。** |
| **2b** | 新（`browser-skills-automate`） | `/automate` 技能模板（`/scrape` 的变更流同级）。重用 `/skillify` 和 D3 帮助器。运行非固化的每变更步骤确认门。P0 在 TODOS 中。 |
| **3** | 新（`browser-skills-resolver`） | 会话开始时的解析器注入（每主机浏览器技能发现）。镜像领域技能注入。`gstack-config browser_skillify_prompts` 旋钮。 |
| **4** | 新 | Eval 测试基础设施（LLM-judge）、固定装置过时检测、定期针对活动页面重新验证、用于不受信任生成的 OS 级 FS 沙箱。 |

---

## 第 1 阶段架构

### 锁定决策（13 个）

1. **第 1 阶段 = 完整存储 + SDK + 子命令 + 捆绑参考。** 还没有代理创作。第 2 阶段交付 `/scrape` 和 `/automate`。
2. **第 2 阶段的两个动词：`/scrape`（只读）和 `/automate`（变更）。** 它们共享 skillify 批准门机制，但作为独立的技能模板存在。
3. **取代了 TODOS.md 中现有的自写-`$B` P1。** 用户可见目标相同，无守护进程内隔离问题。
4. **SDK 分发：每个技能内部的同级文件（选项 E）。** 规范 SDK 位于 `browse/src/browse-client.ts`（约 250 LOC）。每个技能在 `<skill>/_lib/browse-client.ts` 中附带一个副本。第 2 阶段的生成器在生成每个脚本时复制当前 SDK。每个技能完全自包含：将目录复制到任何地方都能运行。版本漂移不可能（SDK 冻结在技能编写时的版本）。磁盘成本：每技能约 3KB。
5. **三层查找：捆绑 → 全局 → 项目。** 捆绑技能作为 gstack 安装的一部分只读发布（`<gstack-install>/browser-skills/<name>/`）。全局位于 `~/.gstack/browser-skills/<name>/`。每项目位于 `<project>/.gstack/browser-skills/<name>/`。查找按优先级顺序遍历层级 项目 → 全局 → bundle；首次命中胜出。**`$B skill list` 在每个技能名称旁打印已解析的层级**，所以"为什么跑那个？"永远不是调试之谜。
6. **信任模型：生成时的受限 token，不是 env-scrub-as-sandbox。** 参见下面的"信任模型"。（经 Codex 标记为安全剧场后从原始 env-scrub 计划修订。）
7. **单一真实来源：仅 SKILL.md frontmatter。** 无 `meta.json`。Frontmatter 包含 host、triggers、args、version、source、trusted。
SHA256/过时性推迟到第 4 阶段作为单独的 `.checksum` 附属文件。
8. **无 INDEX.json。遍历目录。** `$B skill list` 枚举三层并解析每个 SKILL.md frontmatter。50 个技能约 5-10ms。消除整个"索引偏离磁盘"的 bug 类别。
9. **`$B skill run` 输出协议。** stdout = JSON。stderr = 流式日志。退出 0/非零。默认 60s 超时可经由 `--timeout=Ns` 覆盖。最大 stdout 1MB（超出则截断 + 非零退出）。匹配 `gh` / `kubectl` / `docker` 约定。
10. **固定装置重放：两种模式对应两种测试类型。** SDK 单元测试建立测试内 mock HTTP 服务器。端到端技能测试通过脚本导出的解析器函数解析捆绑的 HTML 固定装置（无需守护进程）。第 1 阶段仅固定装置对 `hackernews-frontpage` 足够；第 2 阶段 `/automate` 需要更丰富的固定装置。
11. **`hackernews-frontpage` 参考技能。** 抓取 HN 首页（标题、分数、评论数）。无身份验证，HTML 稳定，理想的固定装置测试目标。
12. **Token/端口发现：生成技能仅限受限-token 环境变量；
    独立调试运行回退到状态文件。** 经由 `$B skill run` 生成时，SDK 从环境读取 `GSTACK_PORT` + `GSTACK_SKILL_TOKEN`。对于独立 `bun run script.ts`，SDK 回退到 `<project>/.gstack/browse.json`（按 `config.ts:50` 的实际状态文件路径）。
13. **CHANGELOG 诚实。** 第 1 阶段牵头人：人类可以编写确定性浏览器脚本 gstack 运行。第 1 阶段明确说明代理创作在下一次发布中落地。无捏造的 perf 数字 —— 第 1 阶段无前后对比。

### 信任模型（决策 #6 详情）

两个正交轴：

| 轴 | 机制 | 默认 |
|------|-----------|---------|
| **守护进程端能力** | 绑定到 `read+write` 作用域的按生成受限 token（17 个命令的浏览器接口，减去管理命令如 `eval`/`js`/`cookies`/`storage`）。单次使用的 clientId 编码技能名称 + 生成 id。在生成退出时撤销。 | 始终作用域化（从不是守护进程 root token）。 |
| **进程端环境访问** | SKILL.md frontmatter `trusted: true` 传递 `process.env` 减去 `GSTACK_TOKEN`。`trusted: false`（默认）删除除最小白名单（LANG、LC_ALL、TERM、TZ、锁定 PATH）外的所有内容，并显式剥离密钥模式键（TOKEN/KEY/SECRET/PASSWORD、AWS_*、AZURE_*、GCP_*、ANTHROPIC_*、OPENAI_*、GITHUB_* 等）。 | 不受信任（必须选择加入）。 |

`GSTACK_PORT` 和 `GSTACK_SKILL_TOKEN` 始终最后注入，所以父进程无法通过在环境中设置它们来覆盖。

**这个做对了什么：** 守护进程端的受限 token 可由守护进程执行。尝试调用 `eval`（管理作用域）的技能即使 SDK 暴露了它也会获得 403。能力边界在正确的位置。

**这个没有关闭的：** Bun 没有内置的 FS 沙箱。不受信任的技能仍然可以 `import 'fs'` 并读取操作系统用户能读取的任何内容（例如 `~/.ssh/id_rsa`）。环境清理是卫生措施，不是沙箱。OS 级隔离（`sandbox-exec`、命名空间）是第 4 阶段的工作，干净地放入现有的 trusted/untrusted 合约后面。

原计划称环境清理为沙箱。Codex 正确地标记了那是
剧场。修订后的计划称其为它的本质：尽力而为的卫生加
纵深防御，真正的边界在守护进程端的受限 token。

### 文件布局

```
browse/src/
├── browse-client.ts                # 规范 SDK（约 250 LOC）
├── browser-skills.ts               # 三级遍历 + frontmatter 解析器 + tombstones
├── browser-skill-commands.ts       # $B skill list/show/run/test/rm + spawnSkill
└── skill-token.ts                  # mintSkillToken / revokeSkillToken 包装器

browser-skills/
└── hackernews-frontpage/           # 捆绑参考技能
    ├── SKILL.md
    ├── script.ts
    ├── _lib/browse-client.ts        # 与规范逐字节相同
    ├── fixtures/hn-2026-04-26.html
    └── script.test.ts

browse/test/
├── skill-token.test.ts              # mint/revoke 生命周期、作用域断言
├── browse-client.test.ts            # mock HTTP 服务器、线路格式、auth
├── browser-skills-storage.test.ts   # 三级遍历、frontmatter、tombstones
└── browser-skill-commands.test.ts   # parseRunArgs、dispatch、env scrub、spawn

test/skill-validation.test.ts       # 扩展：捆绑技能合约检查
```

### 不更改的内容

- 领域技能存储、状态机或注入。不动。
- 隧道表面白名单（`server.ts:118-123`）。相同的 17 个命令。
- L1-L6 安全栈。浏览器技能在第 1 阶段不向提示注入文本；第 3 阶段的解析器注入将使用现有的 UNTRUSTED 信封。
- `cli.ts` HTTP 客户端在 `sendCommand()`。SDK 是关注点分离的独立模块（库 vs CLI 进程）。

---

## Codex 外部声音发现（审查后回复）

/codex 审查标记了 8 个发现。计划按如下方式处理它们：

| # | 发现 | 第 1 阶段回复 |
|---|---------|------------------|
| 1 | 信任模型在没有 FS 沙箱的情况下是假的 | 上述决策 #6（受限 token）**已关闭**。 |
| 2 | 第 1 阶段对一个捆绑技能过度构建（查找层级、tombstones 等） | **确认但保留。** 用户选择完整第 1 阶段来锁定架构，然后第 2 阶段落地代理创作。每个子系统足够小，如果数据后来显示未使用可以干净地移除。 |
| 3 | `cli.ts:398` 的现有客户端模式可能使同级 SDK 多余 | **验证为假。** 第 398 行是 `extractTabId()` 的末尾（一个标志解析器）。实际 HTTP 客户端是 `cli.ts:401-467` 的 `sendCommand()`，但它与 CLI 耦合（`process.stdout.write`、`process.exit`、重启恢复）。不可重用为新库。新的 `browse-client.ts` 镜像其线路格式但形状为库。 |
| 4 | "首次命中胜出" 查找不透明 | **缓解**，通过在 `$B skill list` 和 `$B skill show` 中内联列出已解析的层级。未来：可选的 `--source bundled|global|project` 标志，如果层级覆盖被证明令人困惑。 |
| 5 | 原子技能打包比索引问题更重要；符号链接防御 | **第 1 阶段已关闭**：捆绑技能作为 gstack 安装的一部分发布（无实时写入；通过作为安装目录中的只读文件实现原子性）。第 2 阶段的 `writeBrowserSkill` 将写入临时目录然后重命名，并使用 `realpath`/`lstat` 规范（现有 `browse/src/path-security.ts`）。 |
| 6 | 从活动源的第 2 阶段合成薄弱（有损环形缓冲区） | **第 2 阶段设计未决问题。** 活动源是遥测，不是重放 IR。第 2 阶段需要结构化记录器或使用自己的上下文从头重新提示代理编写脚本。在第 2 阶段的设计审查中决定。 |
| 7 | Bun 运行时回归：作为独立 Bun 技能脚本重新引入 Bun 运行时要求 | **第 2 阶段分发未决问题。** 第 1 阶段绕过了这一点，因为捆绑参考技能发布在 gstack 安装内（已经用 Bun 构建）。第 2 阶段需要决定（a）为每个生成的技能附带 Bun 二进制文件，（b）编译技能为独立可执行文件，或使用 Node.js 加 `cli.ts` 的 HTTP 模式。 |
| 8 | `file://` 固定装置不能证明时序/auth/导航/惰性水合 | **记录的极限。** 对 `hackernews-frontpage` 足够。第 2 阶段 `/automate` 需要更丰富的固定装置（带时序的 mock daemon、记录的重播等）。 |

---

## 第 2a 阶段 — `/scrape` + `/skillify`（正在发布 v1.19.0.0）

两个技能模板加一个帮助器模块。`/scrape <意图>` 是拉取页面数据的单一入口；新意图的第一次调用通过 `$B` 原语制作原型并返回 JSON，后续对匹配意图的调用在约 200ms 内路由到固化的浏览器技能。`/skillify` 将最近一次成功的原型固化为磁盘上的永久浏览器技能。变更流同级 `/automate` 推迟到第 2 阶段（P0 在 TODOS 中）。

### v1.19.0.0 计划审查期间锁定的决策（`/plan-eng-review`）

| ID | 决策 | 锁定行为 |
|----|----------|-----------------|
| **D1** | `/skillify` 来源保护 | 往回走 ≤10 个代理回合寻找一个明确界定的 `/scrape` 调用（原型的意图行 + 其尾随 JSON 输出）。如果未找到，拒绝并显示：*"在此对话中未找到最近的 /scrape 结果。请先运行 /scrape <意图>，然后说 /skillify。"* 无静默回退。 |
| **D2** | 合成输入切片 | 模板指示代理仅提取产生用户接受 JSON 的最终尝试 `$B` 调用加用户陈述的意图字符串。丢弃失败的丢弃选择器尝试、不丢弃相关内容、丢弃早期会话内容。通过选择选项（b）（从自己的上下文中重新提示，不是结构化记录器）关闭 Codex 发现 #6。 |
| **D3** | 原子写入规范 | `/skillify` 写入 `~/.gstack/.tmp/skillify-<spawnId>/`，针对临时目录运行 `$B skill test`，仅在成功 + 用户批准后重命名到最终层级路径。测试失败或批准拒绝：完全 `rm -rf` 临时目录（从未批准的技能无 tombstone）。新模块 `browse/src/browser-skill-write.ts`（`stageSkill` / `commitSkill` / `discardStaged`）按 Codex 发现 #5 使用 `realpath`/`lstat` 规范。 |
| **D4** | 测试范围 | 5 门控 E2E（scrape match、scrape prototype、skillify happy、skillify 来源拒绝、批准门拒绝）+ 1 个单元测试（原子写入帮助器故障清理）+ 1 个手动验证烟雾（变更意图拒绝）。在 `test/helpers/touchfiles.ts` 中注册。 |

### 结转

- **默认层级：全局。** 程序偏重全局，每项目覆盖在 `/skillify` 时（镜像领域技能作用域）。第 1 阶段存储帮助器支持两种查询路径。
- **Bun 运行时分发。** Codex 发现 #7 保持打开。第 2a 阶段假设 Bun 在 PATH 上（gstack 已经通过 `setup:6-15` 要求）。记录在 `/skillify` SKILL.md "限制"中。真正的修复在第 4 阶段落地。

## 第 2b 阶段 — `/automate` 草图

`/scrape` 的变更流同级。相同的 skillify 模式（原样重用 `/skillify`和 D3 帮助器）。区别：非固化运行时每变更步骤 UNTRUSTED 包装摘要 + `AskUserQuestion` 确认门。固化后，技能无人值守运行（固化脚本枚举哪些 `$B click`/`fill`/`type` 调用运行）。参见 `TODOS.md` 中的 P0 条目。

## 第 3 阶段草图

会话开始时的解析器注入。镜像 `server.ts:722-743` 的领域技能注入：

```ts
const browserSkillsBlock = await renderBrowserSkillsForHost(hostname, projectSlug);
if (browserSkillsBlock) {
  systemPrompt += `\n\n${browserSkillsBlock}`;
}
```

`renderBrowserSkillsForHost()` 读取 3 个层级，过滤到 `host` 字段匹配的技能，并发出 UNTRUSTED 包装块列出它们。

`gstack-config browser_skillify_prompts`（默认关闭）：当打开，在 `/qa`、`/design-review` 等中的任务结束时的推动在活动源显示 ≥N 个对同一主机的命令且该主机+意图尚不存在技能时触发。

## 第 4 阶段草图

- LLM-judge 评估（"代理是伸手去拿技能还是重新探索？"）
- 固定装置过时检测 — 比较捆绑固定装置与活动页面
- 用于不受信任生成的 OS 级 FS 沙箱（macOS 上的 `sandbox-exec`，
  Linux 上的命名空间 / seccomp）
- `$B skill upgrade <名称>` — 当规范 SDK 更改时重新生成同级 SDK 副本

---

## 验证（第 1 阶段）

`bun test` 通过新的测试文件：
- `browse/test/skill-token.test.ts` — 15 个断言
- `browse/test/browse-client.test.ts` — 26 个断言
- `browse/test/browser-skills-storage.test.ts` — 31 个断言
- `browse/test/browser-skill-commands.test.ts` — 29 个断言
- `browser-skills/hackernews-frontpage/script.test.ts` — 13 个断言
- `test/skill-validation.test.ts` — 7 个新捆绑技能断言

端到端，守护进程运行：

```bash
$B skill list                            # 显示 hackernews-frontpage（捆绑）
$B skill show hackernews-frontpage       # 打印 SKILL.md
$B skill run hackernews-frontpage        # 返回 30 个故事的 JSON
$B skill test hackernews-frontpage       # 运行 script.test.ts
```
