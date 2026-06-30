# 修复 #1671：`/office-hours` 始终报告 SESSION_COUNT: 0

**状态：** SHIPPED
**分支：** fix-1671-profile-migration
**日期：** 2026-05-23
**Issue：** https://github.com/garrytan/gstack/issues/1671
**引入该 bug 的原始 PR：** garrytan/gstack#1039 / commit `0a803f9` / v1.0.0.0 / 2026-04-18

## 问题描述

`/office-hours` 在每次调用时都报告 `SESSION_COUNT: 0` 和 `TIER: introduction`，即便是已多次使用该技能的用户也是如此。用于跳过回访用户结束宣传的 `welcome_back` 层级（`bin/gstack-developer-profile:165-169`）无法到达。自 v1.0.0.0 起，每个全新 `$HOME` 用户上存活了约 5 周。

## 根因分析

v1.0.0.0 迁移将读取路径移至 `~/.gstack/developer-profile.json`，但将 office-hours/SKILL.md.tmpl 中的写入器留在了旧的 `~/.gstack/builder-profile.jsonl`。首次读取时创建的 `ensure_profile` 桩具有 `sessions: []`；后续写入操作进入一个读取器永远不再重新读取的文件。读取器和写入器在存储位置上产生了分歧。

完整根因分析（包括 RC2/RC3 后续跟进）：https://github.com/garrytan/gstack/issues/1671

## 修复方案

让写入器使用与读取器相同的文件。

### 变更

1. **`bin/gstack-developer-profile`** — 新增 `--log-session '<json>'` 子命令：
   - 验证必填字段（`date`、`mode`），无效输入时静默跳过（与 `bin/gstack-timeline-log:22-26` 一致）。
   - 通过 `bun -e` 读取现有 `developer-profile.json`。
   - 将条目追加到 `sessions[]`。更新 `signals_accumulated`（按信号字符串逐次递增，与 `do_migrate:67-69` 相同），合并 `resources_shown` 和 `topics`。
   - 使用原子 mktemp+mv 写入（与第 54 行现有模式一致）。
   - 写入后调用 `gstack-brain-enqueue "developer-profile.json"`，镜像 `bin/gstack-timeline-log:40`。

2. **`bin/gstack-developer-profile:do_read`** — 在选择 LAST_PROJECT / LAST_ASSIGNMENT / LAST_DESIGN_TITLE / CROSS_PROJECT / DESIGN_* 时过滤 `mode:"resources"` 条目。Phase 6 的资源自动附加发生在同一 /office-hours 调用中的真实会话之后；没有过滤器的话，该资源条目会覆盖用户下次会话的真实会话状态。这是一个被损坏写入器掩盖的潜在缺陷；由本修复激活。

3. **`office-hours/SKILL.md.tmpl`** — 在第 490 行和第 893 行交换写入器：
   - 原代码：`echo '{...}' >> "$GSTACK_STATE_ROOT/builder-profile.jsonl"`
   - 新代码：`~/.claude/skills/gstack/bin/gstack-developer-profile --log-session '{...}' 2>/dev/null || true`
   - 运行 `bun run gen:skill-docs` 重新生成 `office-hours/SKILL.md`。

### 修复中故意不包含的内容

- **无新增二进制文件。** `developer-profile.json` 的所有者二进制文件是 `gstack-developer-profile`；写入器作为子命令属于该二进制文件。`--log-session` 加入了该二进制文件已有的 `--migrate` / `--derive` 写入侧子命令边界，而非 `gstack-*-log` 事件写入器家族。动词名仍与 `gstack-*-log` 保持一致。
- **无 mkdir 锁。** 并发 /office-hours 调用在 `developer-profile.json` 上存在读-改-写竞争条件。代码库在 `gstack-config` 中接受相同竞争（r-m-w 操作 YAML，无锁）。非本修复引入；超出范围。
- **无 schema 版本递增。** 架构保持 `schema_version: 1`。修复不改变架构，只是让写入器使用它。
- **无受影响用户的自动对账。** 具有旧版 `builder-profile.jsonl` 条目记录的现有用户不会将其过去历史自动合并到 `developer-profile.json` 中。下次运行 /office-hours 时，首个新会话进入 `welcome_back`；过去数据保留在旧版文件中（在弃用期间仍可被其他工具读取）。大多数受影响用户仅有少量遗留会话，丢失主要是美观层面的。放弃仅用于一个版本的对账路径——Garry 的"适当规模差异"之声。
- **无 autoplan 时间线汇总（RC2）。** 单独关注点，单独 PR。
- **无项目范围 opt-in（RC3）。** 单独关注点，单独 PR。
- **无 gbrain glob 变更。** office-hours manifest 仍然 glob `~/.gstack/builder-profile.jsonl` 获取上下文；一旦新写入停止落点于此，快照将变冷。如成为 UX 问题则在跟进中更新。

### 测试（均为门控级、免费、确定性）

1. **`test/gstack-developer-profile.test.ts` 中的回归测试：**
   - 全新 `$HOME`。
   - 运行 /office-hours 前言：gstack-developer-profile 创建空桩。
   - 使用启动模式 JSON 调用 `--log-session`。
   - 再次运行 `--read`。断言 `SESSION_COUNT: 1`、`TIER: welcome_back`。
   - 在当前 main 分支上失败（子命令不存在）。修复后通过。

2. **`do_read` 模式过滤测试：** 在记录启动会话后跟一个资源条目，`--read` 返回 LAST_PROJECT / LAST_ASSIGNMENT / LAST_DESIGN_TITLE 来自真实会话而非资源条目。RESOURCES_SHOWN 仍正确聚合。

3. **验证 + 聚合测试：** `--log-session` 静默跳过无效 JSON / 缺少必填字段，注入缺失的 `ts`，保留用户设置的 `ts`，正确聚合多个会话的 signals/resources/topics。

4. **`test/static-no-legacy-writes.test.ts`（新增）中的静态 grep 不变量：** 遍历每个技能目录，断言除允许列表中的读取器（`gstack-developer-profile`、`gstack-memory-ingest.ts`、`gstack-artifacts-init`、文档文件外）没有任何生产代码路径写入 `builder-profile.jsonl`。防止未来写入器回退到旧版文件。

### 验收标准

- 在全新 `$HOME` 上第二次调用 `/office-hours` 返回 `TIER: welcome_back`。
- `bun test` 在隔离状态下通过被修改文件的测试。
- `bun run gen:skill-docs` 产生干净的 diff，与 `.tmpl` 编辑匹配。

### 发布

- 单次提交。按 CHANGELOG 风格指南递增 PATCH 版本。
- CHANGELOG 条目由 `/ship` 编写。面向用户的语气：引导用户体验到之前没有的内容（第二次访问时 welcome_back 层级生效）。

### 后续 TODO

- 在一个版本后完全弃用 `builder-profile.jsonl`（写入器 + shim + memory-ingest 类型）。
- 修复 RC2（autoplan 内联子技能，绕过其 timeline-log 前言）。
- 为具有多个 agent 身份的高级用户添加 `GSTACK_PROFILE_SCOPE` opt-in（RC3）。
- `/plan-tune` 目前不调用 `--derive`，因此 `inferred`/`gap` 可能漂移（已存在，与 #1671 无关）。
- `mode:"resources"` 条目在现有层级聚合器下膨胀 SESSION_COUNT（已存在，与 #1671 根因无关）。
