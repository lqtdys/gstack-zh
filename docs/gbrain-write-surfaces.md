# gbrain 写入表面 — 写到哪里、如何验证

本文面向两类读者：

1. **智能体**：当计划技能渲染紧凑的 `## Brain Context
   Load` 或 `## Save Results to Brain` 块时，这些块引用本文。在实际使用 gbrain 时按需阅读 §Context Load 或 §Save Template。如果 `gbrain` 不在 PATH 上，则完全跳过。
2. **人类**：在针对真实 brain 运行计划技能后，使用手动探测部分确认页面实际落地位置。

## 写到哪里

| 宿主 + 检测状态 | 在计划技能的 SKILL.md 中渲染什么 |
|---|---|
| 任意宿主 + `gstack-config gbrain-refresh` 报告 `gbrain_local_status: "ok"` | 压缩的 brain-aware 块渲染。智能体在实际保存时按需读本文。每个计划技能约 250 token 开销。 |
| 任意宿主 + 未检测到 gbrain | 块在生成时被抑制。零 token 开销。校准取值仍会渲染（独立解析器，宿主无关）。 |
| GBrain 或 Hermes 宿主 | 块始终渲染，不管检测 — 这些宿主将 gbrain 集成视为一等公民。 |

`.gbrain-source` 只固定**读取** — 写入由 `~/.gbrain/config.json` 中配置的默认引擎决定。文档位于
`bin/gstack-gbrain-sync.ts` 供代码查找解析器使用；gstack 对 artifact `put` 语义同样视该合约为承载。如果用户报告写入落在错误的源，先查这里。

信任策略（`personal` vs `shared`，按端点哈希）控制自动推送
和写回。通过 `gstack-config set
brain_trust_policy@<endpoint-hash> personal` 设置。本地 PGLite 安装
默认 `personal`；远程 MCP 安装在 `/setup-gbrain` 步骤 9.5 期间提示。

## §Context Load（智能体在运行计划技能时阅读此节）

开始之前，在 brain 中搜索相关上下文：

1. **从用户请求中提取 2–4 个关键词**。选名词、错误
   名称、文件路径、技术术语 — 不要动词或形容词。
   例如：对于"部署后登录页坏了"，搜索
   `login broken deploy`。
2. **搜索**：`gbrain search "<keyword1 keyword2>"`。返回类似
   `[slug] Title (score: 0.85) - first line of content...` 的行。
3. **如果结果很少**（少于 3 个）：拓宽到最具体的单个
   关键词重试。如果仍然很少，不带 brain 上下文继续。
4. **读取前 3 个结果**：对每个运行 `gbrain get_page "<slug>"`。3 个后停止
   — 收益递减。
5. **使用上下文**来指导你的分析。当某个 brain 页面改变了你的思考时，
   在你的输出中引用具体的 slug。

如果 `gbrain search` 返回任何非零退出码（gbrain 不在 PATH 上、网络
抖动、限流），视为瞬态：不带 brain 上下文继续。不要
内联重试 — 用户可以稍后重新运行技能。

## §Save Template（智能体在实际保存时阅读此节）

技能完成后，保存输出。紧凑的解析器块已经显示了针对你的特定技能的 slug 前缀 + 标题 + 标签（例如
`gbrain put "ceo-plans/<feature-slug>" ...`）。完整模板：

```bash
gbrain put "<slug-prefix>/<feature-slug>" --content "$(cat <<'EOF'
---
title: "<Title>: <feature name>"
tags: [<tag>, <feature-slug>]
---
<skill output in markdown — the actual deliverable, not a summary>
EOF
)"
```

**Slug 指导**：`<feature-slug>` 应为 kebab-case、小写，且在
前缀内唯一。优先使用具体的项目/功能名称而非
抽象标签。例如：`auth-rate-limit` 而非 `security-fix`。

**Title 指导**：常量前缀（例如"CEO Plan"、"Eng Review"）
是固定的；后缀是人类可读的功能/主题名称。

**Tag 指导**：第一个标签是来自技能元数据的常量 `<tag>`（例如 `ceo-plan`、`eng-review`）。第二个标签是
`<feature-slug>`，使跨页面遍历工作。如果存在明显关系则添加更多标签
（例如 `[ceo-plan, auth-rate-limit, security]`）。

### Entity-stub 增强

保存主页面后，提取输出中提到的人名和组织名。对每个实体：

```bash
# 先检查页面是否存在
gbrain search "<entity name>"

# 如果无匹配，创建 stub
gbrain put "entities/<entity-slug>" --content "$(cat <<'EOF'
---
title: "<Person or Company Name>"
tags: [entity, person]
---
Stub page. Mentioned in <skill name> output. Replace with real bio when relevant.
EOF
)"
```

**仅提取真实名称** — 实际人名（例如"Garry Tan"）和公司/组织名称（例如"Y Combinator"）。跳过产品名、功能名、段落标题、技术术语（CSS 类名、函数名）和文件路径。有疑问时跳过。

人用 `tags: [entity, person]`，公司/团队用 `tags: [entity, organization]`。

### 错误处理

- **限流**：退出码 1，stderr 包含 `throttle`、`rate
  limit`、`capacity` 或 `busy`。推迟保存然后继续 — brain
  正忙；内容没有丢失，只是未在此运行中持久化。
- **任何其他非零退出码**：视为瞬态失败。不要内联重试 — 用户可以重新运行技能或运行
  `gstack-config gbrain-refresh`（如果他们怀疑 gbrain 自身
  配置有误）。
- **`gbrain: command not found`**：gbrain 不在 PATH 上。紧致的
  解析器块告诉你要跳过 — 你不应该走到这段代码。如果你
  碰到了，静默跳过并继续。

### 反向链接

如果你的保存输出提到了另一个 brain 页面，按名称或主题，在 markdown 主体底部添加反向链接行：

```
Related: [[other-page-slug]], [[another-slug]]
```

gbrain 自动将 `[[slug]]` 语法解析为渲染页面中的可点击链接。
仅在关系是具体的时候添加反向链接（例如"此 CEO 计划依赖于位于
`eng-reviews/auth-rate-limit` 的 eng review"）。不要编造联系。

### 完成摘要

在你的最终技能输出中，用一行注明 brain 使用情况：
"Brain: read 3 pages, saved 1 page, enriched 2 entity stubs, 0 throttles."
这有助于用户看到 brain 覆盖随时间增长。

## 持久化验证（自动化）

匹配对问题"我们希望保存的数据实际是否被保存"由 `test/skill-e2e-gbrain-roundtrip-local.test.ts` 覆盖：
针对隔离的临时 HOME 进行真实 `gbrain init --pglite` + `gbrain put` + `gbrain get` 往返。周期性等级。当
`VOYAGE_API_KEY` 未设置或 gbrain CLI 不在 PATH 上时跳过。

在打开触及解析器的 PR 之前运行：

```bash
EVALS=1 EVALS_TIER=periodic VOYAGE_API_KEY=$VOYAGE_API_KEY \
  bun test test/skill-e2e-gbrain-roundtrip-local.test.ts
```

如果你确实想在自己的 brain 上手动抽查（调试智能体应该保存的特定页面）：

```bash
gbrain get "<prefix>/<slug>"           # 预期 markdown + frontmatter
gbrain search "<slug fragment>"        # 预期 slug 在顶部结果中
gbrain sources list                    # 确认 gstack-brain-<user> 源
gbrain get "entities/<person>"         # 预期每个人物的 stub
```

## 远端 / Supabase / thin-client-MCP 路由

解析器发出单一 CLI 形态 — `gbrain put "<slug>" --content
"..."` — 对 gbrain 支持的每种引擎都可用。CLI
根据用户的 `~/.gbrain/config.json` 内部路由到本地 PGLite、远端 Supabase 或远端 MCP 端点。**gstack 不测试该路由**：存储层是 gbrain 遵守的合约，我们测试的本地 PGLite 上使用的同一 CLI 调用也适用于其他任何引擎。

如果你在 Supabase 或 thin-client MCP 上写入未落地：

1. `gbrain doctor --fast --json` — 引擎健康检查。如果有任何报告 `error`，先修那个。
2. `gbrain write` 自动写入要求 `gstack-config get brain_trust_policy@<endpoint-hash>` 为
   `personal`。运行 `gstack-config endpoint-hash` 获取当前哈希。如果是 `shared`，智能体在写入前提示 — 如果你拒绝了，重新运行技能。
3. 如果信任策略是 `personal` 且 `gbrain doctor` 干净但页面仍不存在，向 gbrain 提 issue — gstack 的 CLI 调用形态与 T11（`gbrain-roundtrip-local`）
   测试的相同。

## 自动化不验证的内容

- **Calibration takes（`takes_add`）**：目前这些回退到
  `gbrain put` 内的围栏块写入，因为
  `BRAIN_CALIBRATION_WRITEBACK` 为 FALSE，等待 gbrain v0.42+ 交付
  `takes_add` MCP 操作。当标志翻转时，按此文档重新运行探测，针对 `/office-hours` 并确认 `gbrain takes_list` 出现一个权重为预期值（office-hours 为 0.9，根据
  `scripts/brain-cache-spec.ts:151-157`）的 `kind=bet` 条目。
- **其他 4 个计划技能的 per-skill E2E**：只有 `/office-hours`
  有 fake-CLI E2E 覆盖（`test/skill-e2e-office-hours-brain-writeback.test.ts`）。
  解析器单元测试（`test/resolvers-gbrain-save-results.test.ts`）
  覆盖全部 5 个的连接。Per-skill E2E 扩展在 TODOS.md 中跟踪。
- **`.gbrain-source` 写入语义**：gstack 将文档化的
  只读合约视为承载，但不独立验证
  gbrain CLI 从不根据 pin 重新路由写入。如果你发现
  有此情况，那是要上报的 gbrain bug。
