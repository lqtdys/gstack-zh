# QA 报告：{APP_NAME}

| 字段 | 值 |
|-------|-------|
| **日期** | {DATE} |
| **URL** | {URL} |
| **分支** | {BRANCH} |
| **提交** | {COMMIT_SHA} ({COMMIT_DATE}) |
| **PR** | {PR_NUMBER} ({PR_URL}) 或 "—" |
| **级别** | Quick / Standard / Exhaustive |
| **范围** | {SCOPE 或 "Full app"} |
| **持续时间** | {DURATION} |
| **访问的页面数** | {COUNT} |
| **截图数** | {COUNT} |
| **框架** | {DETECTED 或 "Unknown"} |
| **索引索引** | [所有 QA 运行](./index.md) |

## 健康评分：{SCORE}/100

| 类别 | 评分 |
|----------|-------|
| 控制台 | {0-100} |
| 链接 | {0-100} |
| 视觉 | {0-100} |
| 功能 | {0-100} |
| 用户体验 | {0-100} |
| 性能 | {0-100} |
| 无障碍 | {0-100} |

## 前 3 个需要修复的问题

1. **{ISSUE-NNN}：{title}** — {一行描述}
2. **{ISSUE-NNN}：{title}** — {一行描述}
3. **{ISSUE-NNN}：{title}** — {一行描述}

## 控制台健康状况

| 错误 | 计数 | 首次出现在 |
|-------|-------|------------|
| {错误消息} | {N} | {URL} |

## 总结

| 严重级别 | 计数 |
|----------|-------|
| 严重 | 0 |
| 高 | 0 |
| 中 | 0 |
| 低 | 0 |
| **总计** | **0** |

## 问题列表

### ISSUE-001：{简短标题}

| 字段 | 值 |
|-------|-------|
| **严重级别** | critical / high / medium / low |
| **类别** | visual / functional / ux / content / performance / console / accessibility |
| **URL** | {页面 URL} |

**描述：** {错误是什么，预期是什么实际是什么。}

**复现步骤：**

1. 导航到 {URL}
   ![步骤 1](screenshots/issue-001-step-1.png)
2. {操作}
   ![步骤 2](screenshots/issue-001-step-2.png)
3. **观察：** {错误是什么}
   ![结果](screenshots/issue-001-result.png)

---

## 已应用的修复（如果适用）

| 问题 | 修复状态 | 提交 | 更改的文件 |
|-------|-----------|--------|---------------|
| ISSUE-NNN | verified / best-effort / reverted / deferred | {SHA} | {files} |

### 前后对比证据

#### ISSUE-NNN：{title}
**之前：** ![之前状态](screenshots/issue-NNN-before.png)
**之后：** ![之后状态](screenshots/issue-NNN-after.png)

---

## 回归测试

| 问题 | 测试文件 | 状态 | 描述 |
|-------|-----------|--------|-------------|
| ISSUE-NNN | path/to/test | committed / deferred / skipped | 描述 |

### 延迟的测试

#### ISSUE-NNN：{title}
**前置条件：** {触发 bug 的设置状态}
**操作：** {用户的操作}
**预期：** {正确的行为}
**延迟原因：** {原因}

---

## 发版就绪指标

| 指标 | 值 |
|--------|-------|
| 健康评分 | {before} → {after}（{delta}） |
| 发现问题数 | N |
| 已修复问题数 | N（verified：X，best-effort：Y，reverted：Z） |
| 延迟问题数 | N |

**PR 摘要：** "QA 发现 N 个问题，修复了 M 个，健康评分 X → Y。"

---

## 回归（如果适用）

| 指标 | 基线 | 当前 | 变化 |
|--------|----------|---------|-------|
| 健康评分 | {N} | {N} | {+/-N} |
| 问题数 | {N} | {N} | {+/-N} |

**自基线以来修复了哪些：** {列表}
**自基线以来新增了哪些：** {列表}
