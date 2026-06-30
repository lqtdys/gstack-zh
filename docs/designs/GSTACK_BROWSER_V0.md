# 修复 #1671：`/office-hours` 始终报告 SESSION_COUNT: 0

**状态：** 已发版
**分支：** fix-1671-profile-migration
**日期：** 2026-05-23
**问题：** https://github.com/garrytan/gstack/issues/1671
**引入 bug 的原始 PR：** garrytan/gstack#1039 / 提交 `0a803f9` / v1.0.0.0 / 2026-04-18

## 问题

`/office-hours` 在每次调用时都报告 `SESSION_COUNT: 0` 和 `TIER: introduction`，即使用户已多次运行该技能。为跳过回访用户的收尾推销而存在的 `welcome_back` tier（`bin/gstack-developer-profile:165-169`）无法达到。自 v1.0.0.0 起，每个新 `$HOME` 用户身上已存活约 5 周。

## 根本原因

v1.0.0.0 迁移将读取路径移至 `~/.gstack/developer-profile.json`，但将写入器留在 `office-hours/SKILL.md.tmpl` 写入旧路径 `~/.gstack/builder-profile.jsonl`。首次读取时创建的 `ensure_profile` 存根具有 `sessions: []`；后续写入进入读取器永远不会重新读取的文件。读取器和写入器对存储位置不一致。

完整的根本原因分析（包括 RC2/RC3 跟进）：https://github.com/garrytan/gstack/issues/1671

## 修复

让写入器使用与读取器相同的文件。

### 变更

1. **`bin/gstack-developer-profile`** — 添加 `--log-session '<json>'` 子命令：
   - 验证必填字段（`date`、`mode`），无效输入时静默跳过（与 `bin/gstack-timeline-log:22-26` 匹配）
   - 通过 `bun -e` 读取现有 `developer-profile.json`
   - 追加条目到 `sessions[]`。更新 `signals_accumulated`（按信号字符串递增，与 `do_migrate:67-69` 相同），并集 `resources_shared` 和 `topics`
   - 原子性 mktemp + mv 写入（与第 54 行的现有模式匹配）
   - 写入后调用 `gstack-brain-enqueue "developer-profile.json"`，镜像 `bin/gstack-timeline-log:40`

2. **`bin/gstack-developer-profile:do_read`** — 在选取 LAST_PROJECT / LAST_ASSIGNMENT / LAST_DESIGN_TITLE / CROSS_PROJECT / DESIGN_* 时过滤 `mode: "resources"` 条目。Phase 6 资源自动追加在同一个 /office-hours 调用中真实会话之后发生；没有该过滤，资源条目会覆盖用户下次会话的真实会话状态。被破坏写入器掩盖的潜在 bug；由该修复激活。

3. **`office-hours/SKILL.md.tmpl`** — 在第 490 和第 893 行交换写入器：
   - 从：`echo '{...}' >> "$GSTACK_STATE_ROOT/builder-profile.jsonl"`
   - 到：`~/.claude/skills/gstack/bin/gstack-developer-profile --log-session '{...}' 2>/dev/null || true`
   - 运行 `bun run gen:skill-docs` 以重新生成 `office-hours/SKILL.md`。

### 修复中**未包含**的内容（有意为之）

- **无新二进制文件。** `developer-profile.json` 的所有者二进制文件是 `gstack-developer-profile`；写入器作为子命令归属在那里。`--log-session` 加入该二进制文件现有的 `--migrate` / `--derive` 写入侧子命令边界，而不是 `gstack-*-log` 事件写入器家族。动词名仍与 `gstack-*-log` 匹配。
- **无 mkdir 锁。** 并发 /office-hours 调用对 `developer-profile.json` 有读-改-写竞争。代码库在 `gstack-config`（对 YAML 的 r-m-w，无锁）中接受相同竞争。非此修复引入；超出范围。
- **无 schema 升级。** Schema 保持 `schema_version: 1`。修复不更改 schema，只是让写入器使用它。
- **无受影响用户的自动对账。** 拥有孤立 `builder-profile.jsonl` 条目的现有用户不会自动将其过去历史合并到 `developer-profile.json` 中。他们下次运行 /office-hours 时，第一个新会话进入 `welcome_back`；过去数据留在旧文件中（在弃用期间仍可被其他工具读取）。大多数受影响用户只有少量孤立会话，所以损失主要是审美层面的。放弃了一次性发布对账路径——Garry 的"合适大小的 diff"之声。
- **无 autoplan 时间线汇总（RC2）。** 独立关注，独立 PR。
- **无项目范围 opt-in（RC3）。** 独立关注，独立 PR。
- **无 gbrain glob 变更。** office-hours manifest 仍 glob `~/.gstack/builder-profile.jsonl` 以获取上下文；一旦新写入停止落入，快照变冷。如果成为 UX 问题则在跟进中更新。

### 测试（全部 gate-tier，免费，确定性）

1. `test/gstack-developer-profile.test.ts` 中的**回归测试**：
   - 新 `$HOME`
   - 运行 /office-hours 前导码：gstack-developer-profile 创建空存根
   - 使用 startup 模式 JSON 调用 `--log-session`
   - 再次运行 `--read`。断言 `SESSION_COUNT: 1`，`TIER: welcome_back`
   - 在当前 main 上失败（子命令不存在）。带修复通过。

2. **`do_read` 模式过滤测试：** 在记录了一个 startup 会话后跟一个资源条目，`--read` 返回真实会话的 LAST_PROJECT / LAST_ASSIGNMENT / LAST_DESIGN_TITLE，而非资源条目的。RESOURCES_SHOWN 仍正确聚合。

3. **验证 + 聚合测试：** `--log-session` 静默跳过无效 JSON / 缺少必填字段，如缺少则注入 `ts`，保留用户设置的 `ts`，正确跨多个会话聚合信号/资源/主题。

4. `test/static-no-legacy-writes.test.ts` 中的**静态 grep 不变量**（新）：遍历每个技能目录，断言除列入允许列表的读取器（`gstack-developer-profile`、`gstack-memory-ingest.ts`、`gstack-artifacts-init`，doc 文件）外，没有生产代码路径写入 `builder-profile.jsonl`。防止未来写入器回归到旧文件。

### 验收标准

- 在新 `$HOME` 上第二次 `/office-hours` 调用返回 `TIER: welcome_back`
- `bun test` 在隔离状态下对受影响文件通过
- `bun run gen:skill-docs` 产生与 `.tmpl` 编辑匹配的干净 diff

### 推出

- 一个提交。按 CHANGELOG 样式指南 PATCH 版本升级
- CHANGELOG 条目由 `/ship` 编写。用户面对的语言：以用户现在体验到之前没有的内容开头（welcome_back tier 在第二次访问时启动）

## 后续 TODO

- 在一个版本后完全弃用 `builder-profile.jsonl`（写入器 + shim + memory-ingest 类型）
- 修复 RC2（autoplan 内联子技能，绕过它们的时间线日志前导码）
- 为多智能体身份的高级用户添加 `GSTACK_PROFILE_SCOPE` opt-in（RC3）
- /plan-tune 当前不调用 `--derive`，所以 `inferred`/`gap` 可能漂移（预存，与 #1671 无关）
- `mode: "resources"` 条目在现有 tier 聚合器下膨胀 SESSION_COUNT（预存，与 #1671 根本原因无关）
