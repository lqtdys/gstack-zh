# /sync-gbrain 批处理摄取迁移

**状态：** 在 garrytan/dublin-v1 上实现（D1-D8 决策落于此 PR）
**分支：** garrytan/dublin-v1
**负责人：** Garry Tan
**触发：** /investigate 运行，2026-05-09
**预计工作量：** 人工约 3 天 / CC+gstack 约 2 小时
**涉及文件：** 4 个源文件 + 1 个测试 = 共 5 个（低于预估）

## 决策（审查后）

本文档记录原始架构。最终架构按以下 8 个审查决策落地：
`/Users/garrytan/.claude/plans/purrfect-tumbling-quiche.md`：

- **D1** 层级化暂存目录（按 slug 段创建 mkdir -p）— 保留
- **D2** 同一 PR 中切换 + 删除遗留代码（无 `--legacy-ingest` 标志）— 保留
- **D3** 先扫描源文件，仅暂存干净的 — 保留
- **D4** ~~三态 OK/DEGRADED/ERR 判定~~ 按 Codex 发现 7 折叠为 OK/ERR
  （gbrain 的 content_hash 幂等性使得第三态冗余）
- **D5** ~~state schema 中的 skip_reason 字段~~ 按 Codex 发现 7 放弃
  （重跑成本低；无需永久跳过追踪）
- **D6** 信任 gbrain 的 content_hash 幂等性；去掉记账脚手架
  （skip_reason、三态、SIGTERM checkpoint）
- **D7** 通过 `~/.gbrain/sync-failures.jsonl` 进行单文件故障检测
  （字节偏移快照 + 仅追加读取）
- **D8** 捆绑 3 个范围内已有的修复：F6 atomic saveState
  （tmp+rename）、F8 isolated-stage 基准测试、F9 完整文件 sha256 哈希
  （不再有 1MB 上限）

## 从 gbrain 源验证

通过阅读 `~/git/gbrain/src/` 验证的三个特性：

- **幂等性** 在 `core/import-file.ts:242-243, :478` — content_hash 检查，未更改则跳过，已更改则覆盖。
- **Frontmatter 一致性** 在 `core/import-file.ts:228, 297, 410-422` —
  title/type/tags 受尊重；仅 frontmatter 缺失时才自动推断。
- **路径权威 slug** 在 `core/sync.ts:260`（`slugifyPath`），
  在 `core/import-file.ts:429` 强制执行。
- **单文件故障暴露** 在 `commands/import.ts:308-310`，
  `:28` 的注释："callers can gate state advances" — D7 使用的有意 API。

## 性能：计划 vs 实测（2026-05-10 性能审查后）

| 指标 | 计划目标 | 实测 | 判定 |
|---|---|---|---|
| 5135 个文件的准备阶段 | — | <10s | 快 |
| 5135 个文件的 `gbrain import` | — | >10 min | gbrain 端性能问题，已提单 |
| 循环/挂起（原始 Bug） | 从不 | 从不 | 已修复 |
| SIGTERM 时内存摄取退出 null | 否 | 否 — state 写入成功；子 gbrain 随父进程死亡 | 已修复 |
| FILE_TOO_LARGE 阻塞 last_commit | 否 | 否 — 失败路径通过 D7 排除 | 已修复 |

**初始性能失误 + 修正。** 第一次冷启动测量
（~12 min）由 1841 个顺序 gitleaks 子进程产生主导，
每个约 256ms — 一个冗余的安全门。跨机器
泄漏边界是 `gstack-brain-sync`（bin/gstack-brain-sync:78-110，
在 `git commit` 前对暂存 diff 进行基于正则的秘密扫描）。
在摄取到本地 PGLite 之前扫描每个源文件并不会改变
暴露 — 秘密已经以明文驻留在磁盘上。我们通过 `--scan-secrets` 将单文件 gitleaks 设为可选。默认关闭。
这将准备阶段从 ~12 min 缩短到 10 秒以内。

剩余的冷启动开销是 `gbrain import` 本身，其扩展性
比大型暂存目录差（501 个文件 10 秒；5031 个超过 10 min）。
这是 gbrain 端的性能问题，不是 gstack 架构。
已提为 TODO；修复可能在 gbrain 的 content_hash 检查
循环或自动链接对账阶段。

## F9 哈希迁移（一次性悬崖）

F9 `fileSha256` 从 1MB 上限哈希切换到完整文件。此更改之前的现有 state
条目携带旧的 1MB 上限哈希。对于 mtime 未更改的任何文件，
`fileChangedSinceState` 在 mtime 检查处返回 false，新哈希永远不会计算 —
所以未更改的文件行为完全一致。对于升级后 mtime **确实**更改的任何文件，
完整文件哈希被重新计算并（正确地）视为已更改，然后
重新导入。`gbrain doctor` 探测报告的 `updated_count` 在升级后首次运行时可能显示
膨胀的数字，因为每个被触及的文件都会跨越算法边界。无数据丢失，但值得了解。

## 后续跟进（已提为 TODO）

1. **gbrain import 在大目录上的性能** — 为什么 5031 个文件需要
   >10 min 而 501 个只需 10s。可能原因：`getPage(slug)` content_hash
   检查的 N+1 SQL、每页自动链接对账、
   FTS 索引更新无批量处理。存在于 gbrain，非 gstack。
2. **可选：源文件变更检测缓存** — 即使准备阶段快，
   遍历 5031 个文件也要一些时间。在批次级别（非每页）缓存
   "自上次成功导入以来无更改"状态，
   将跳过完全无操作的增量运行。

## 问题

`/sync-gbrain` 记忆阶段在全新 PGLite 上需要 35 分钟并退出 null，
丢失所有进度。后续运行重复相同的 35 分钟。在
连续两次运行中观察到（gbrain 0.30.0 broken-postgres 运行：712s 退出 null；
gbrain 0.31.2 PGLite 运行：2100s 退出 null，但实际持久化了 501 页）。

## 根因（来自 /investigate）

`bin/gstack-memory-ingest.ts` 中的两个复合 Bug：

1. **每文件子进程架构。** 第 911 行的摄取循环遍历
   `~/.gstack/projects/` 中的 1,841 个文件，并为每个文件产生两个子进程：
   - `gitleaks detect --no-git --source <path>` — 46ms 冷启动（`lib/gstack-memory-helpers.ts:157`）
   - `gbrain put <slug>` — 329ms 冷启动（`bin/gstack-memory-ingest.ts:823`）
   - 每文件底线：375ms × 1841 = 690s（11.5 min）纯子进程启动
     在任何实际工作之前。

2. **Kill-no-save 超时。** `bin/gstack-gbrain-sync.ts:442` 中的编排器
   强制执行 35 分钟超时。触发时，`spawnSync` 返回
   `result.status === null`，子进程收到 SIGTERM，内存中的
   摄取状态永远不刷新到 `~/.gstack/.transcript-ingest-state.json`。
   下次运行从相同的无进展状态开始 — 解释了
   "全部重做"的模式。

## 现场数据

| 指标 | 值 | 来源 |
|---|---|---|
| walkAllSources 中的文件数 | 1,841 | `find ~/.gstack/projects -type f \( -name "*.md" -o -name "*.jsonl" \)` |
| `gbrain put` 冷启动 | 329ms | `time (echo "test" \| gbrain put _bench)` |
| `gitleaks detect` 冷启动 | 46ms | `time gitleaks detect --no-git --source <small-file>` |
| 理论底线（仅子进程） | 690s / 11.5 min | 375ms × 1841 |
| 观测运行时间 | 2100s / 35 min | 与编排器超时完全匹配 |
| 实际持久化的页数 | 501 | gbrain sources list page_count |
| PGLite 运行期间增长 | 290 → 386 MB | `du -sh ~/.gbrain/brain.pglite` |

## 建议架构

用 **准备-然后-批处理** 管线替换逐文件子进程循环：

```
walkAllSources(ctx)
  → prepareStage（进程内，快速）：
       解析 transcripts/artifacts
       构建含自定义 YAML frontmatter 的 PageRecord
       gitleaks 扫描（对暂存目录执行一次子进程）
       将准备好的 .md 写入暂存目录
  → gbrain import <staging-dir> --no-embed（一次子进程）
  → 用所有成功的结果刷新状态文件
  → 清理暂存目录
```

### 为什么 `gbrain import <dir>` 是正确的批处理路径

- 已在 gbrain CLI 中发布（已验证：`gbrain --help` 显示 `import <dir> [--no-embed]`）。
- 在 gbrain 自己的运行时内进程内遍历目录 — 无子进程扇出。
- 尊重 gbrain 的批大小和嵌入批次调优。
- gbrain v0.31.2 导入在观测运行中 10 秒完成 501 页 + 2906 块；慢的是我们上面的逐文件 `gbrain put` 循环。

### 保留当前代码做对的部分

- **自定义 YAML frontmatter 注入**（title、type、tags）— 通过将带 frontmatter 的 .md 文件写入暂存目录来保留。
- **秘密扫描** — 保留，但移到准备后、导入前的一次 `gitleaks detect --source <staging-dir>` 调用。有发现的文件在导入前从暂存目录删除或就地清除；暂存目录保证 gitleaks 只看到准备好的内容，而非内部 gbrain 状态。
- **部分 transcript 检测** — 保留在准备阶段；部分文件仍在 frontmatter 中获得 `partial: true` 字段。
- **无归属 transcript 过滤** — 保留在准备阶段。
- **每文件 mtime + sha256 状态追踪** — 保留；准备阶段记录暂入了什么，导入成功结果记录落地了什么。
- **增量模式** — `fileChangedSinceState` 检查留在准备循环顶部。

## 迁移步骤

### 第 1 步：从当前摄取循环提取 `preparePages`

将 `ingestPass` 中在遍历和 `gbrainPutPage` 调用之间的所有内容
（`bin/gstack-memory-ingest.ts` 的第 899-988 行）。移入新函数
`preparePages(args, ctx, state) → { staged: PreparedPage[], skipped, failed }`。

输出：`{ slug, body, source_path, mtime_ns, sha256, partial }` 列表，
其中 `body` 是包含 frontmatter 的完整 markdown。

### 第 2 步：添加暂存目录写入器

纯函数：`writeStaged(prepared, stagingDir) → { written, errors }`。
文件名：`${slug}.md`。幂等覆盖。

暂存目录生命周期：
- 创建在 `~/.gstack/.staging-ingest-${pid}-${ts}/`
- 在 `finally` 块中清理，即使 SIGTERM
- 每次摄取 pass 一个暂存目录 — 不在运行间重用

### 第 3 步：单次 gitleaks 遍历

将逐文件的 `secretScanFile(path)` 调用替换为准备后的一次调用：
`gitleaks detect --no-git --source <staging-dir> --report-format json --report-path -`。

解析 JSON 输出，构建 `Map<slug, findings[]>`。有发现的文件在导入前从暂存目录移除（或按现有清除策略就地清除，见 `lib/gstack-memory-helpers.ts`）。

### 第 4 步：用单次导入调用替换 `gbrainPutPage` 循环

```typescript
const importResult = spawnSync("gbrain", ["import", stagingDir], {
  stdio: ["ignore", "inherit", "inherit"],
  timeout: 30 * 60 * 1000, // 宽裕；整个批次
});
```

解析 stdout 中的 `Import complete` 行和 `failed` 计数。

### 第 5 步：部分成功时持久化状态

如果 gbrain import 报告 `imported=N, failed=M`，为 N 个成功的
slug 保存状态（不是全部）。失败的保持无状态以便下次重试，但成功的不会重做。

### 第 6 步：`gstack-memory-ingest.ts` 中的 SIGTERM 处理器

将 `main()` 包装在：
```typescript
let interrupted = false;
const flush = () => {
  if (interrupted) return;
  interrupted = true;
  saveState(state); // 尽力刷新已累积的内容
  cleanupStagingDir();
  process.exit(143);
};
process.on("SIGTERM", flush);
process.on("SIGINT", flush);
```

这独立地解除了 kill-no-save Bug — 即使批处理导入
超过编排器超时，准备阶段的状态也能存活。

### 第 7 步：编排器更新

在 `bin/gstack-gbrain-sync.ts:444`：
- 将 `result.status === 0` 改为 `result.status === 0 || (parsedSummary.imported > 0 && parsedSummary.imported >= parsedSummary.skipped + parsedSummary.failed)`。
  将部分成功（大部分页已导入）视为 OK，非 ERR。
- 在阶段摘要中暴露 `failed_count` 和 `partial_blockers` 以便用户看到
  `Memory ... OK 487/501 imported (14 FILE_TOO_LARGE)` 而非
  `ERR exited null`。

### 第 8 步：专门处理 FILE_TOO_LARGE

当 gbrain 报告 FILE_TOO_LARGE 时，记录到新的
`~/.gstack/.ingest-skip-list.json`，以便下次准备阶段完全跳过该文件。
避免重新暂存总是失败的文件。用户可以通过新的 `gstack-memory-ingest --skip-list` 标志查看跳过列表。

## 测试计划

1. **单元测试（免费，在 `bun test` 中运行）：**
   - 对 50 个文件的 fixture 语料库运行 `preparePages`：断言 YAML 正确、
     部分检测工作、无归属已过滤。
   - `writeStaged` 覆盖幂等性。
   - 使用子进程测试工具验证 SIGTERM 处理器刷新行为。

2. **集成测试（免费，在 `bun test` 中运行）：**
   - 在临时 PGLite 上端到端：准备 → gitleaks → gbrain import，
     断言 page_count 与导入计数匹配。
   - 部分成功路径：注入故意的 FILE_TOO_LARGE；断言成功的仍然被记录，失败记录到跳过列表。
   - SIGTERM 跨运行保持：生成摄取、在中间点杀死、重启，
     断言恢复的 state。

3. **基准门控（定期，付费）：**
   - 1841 个文件 fixture 的冷启动：断言在 8 分钟内。
   - 增量运行（无更改）：断言在 60 秒内。
   - 测试 fixture：`~/.gstack/projects/` 快照的副本，用于可重复计时。

## 回滚策略

- `gstack-memory-ingest` 上的新 `--legacy-ingest` 标志保留旧的
  逐文件路径一个发布周期可调用。
- 如果批处理路径在真实语料上衰退，设置
  `gstack-config set memory_ingest_path legacy` 即可回滚，无需重新部署。
- 确认批处理后稳定一个小版本，即移除标志 + 旧路径。

## 风险与待 plan-eng-review 确认的开放问题

1. **gbrain import 在重叠 slug 上的幂等性。** 如果之前的运行
   用旧内容将 slug X 写入了 PGLite，`gbrain import` 导入
   updated-X 是覆盖还是重复？需在依赖前测试。

2. **`gbrain import` 解析器内的 frontmatter 注入。** 当前代码
   知道如何将 title/type/tags 注入现有 frontmatter 块
   （第 794-821 行）。`gbrain import` 是否像 `gbrain put` 一样
   尊重这些字段？在单元测试中验证。

3. **暂存目录磁盘压力。** 1841 个文件 × 平均 ~50KB = 约 92MB 的
   暂存 .md 内容。开发机上可以接受但值得了解。
   替代方案：将准备好的内容流式传输到通过管道传给导入的 tar（如果 gbrain
   支持）— 可能不支持，V1 忽略。

4. **跨 worktree 并发。** `~/.gstack/.staging-ingest-${pid}-${ts}/`
   是 pid 命名空间化的，所以两个并发的 /sync-gbrain 运行不会碰撞。
   但编排器已经在 `~/.gstack/.sync-gbrain.lock` 持有锁，
   所以这是双保险。保留。

5. **"memory ingest exited null" 消息。** 此更改后，
   编排器在真实的 OOM kill 或 SIGKILL 时仍可能看到 status=null。
   判定块是否应更诚实？例如，
   `ERR memory: killed by signal SIGTERM at 35:00 (timeout)`。

6. **我们是否应完全弃用 `gbrain put` 用于记忆？**
   旧路径存在于 V1.5 的 `put_file` 迁移计划中。有了批处理导入
   工作，我们是否仍需要单页 put 作为临时摄取的后备？
   大概需要（用于编排器外触发的 `~/.gstack/.transcript-ingest-state.json`
   更新），但值得确认。

## 这不是什么

- 非 gbrain CLI 更改。所有工作都在 gstack 中。
- 非 CLAUDE.md voice/UX 更改。
- 非面向用户的新功能。CHANGELOG 条目将读作："记忆摄取在冷启动上快约 10× 并能承受中断。"

## 验收标准

- 1841 个文件上的冷 `/sync-gbrain` 在 8 分钟内完成。
- 增量 `/sync-gbrain`（无文件更改）在 60 秒内完成。
- 运行中 SIGTERM 刷新状态；下次运行恢复而不重做
  已成功导入的文件。
- FILE_TOO_LARGE 故障不阻塞 sync.last_commit 推进。
- 所有现有测试 fixture（transcripts、learnings、design-docs、ceo-plans）
  以完整 frontmatter 正确摄取。
- 部分 transcript 或无归属 transcript 处理无衰退。
