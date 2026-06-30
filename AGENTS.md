# gstack — AI 工程工作流

gstack 是一组 SKILL.md 文件，为 AI 代理提供软件
开发的结构化角色。每个技能都是专家级：CEO 评审员、工程经理、
设计师、QA 负责人、发布工程师、调试器等。

## 可用技能

技能文件存放在 `.agents/skills/` 目录下（Claude Code 上则为 `~/.claude/skills/gstack/`）。
通过名称调用（如 `/office-hours`）。

### 计划审查模式技能

| 技能 | 作用 |
|-------|-------------|
| `/office-hours`  | 从这里开始。重写你的产品创意后再写代码。 |
| `/plan-ceo-review` | CEO 级评审：从请求中找出10星级产品。 |
| `/plan-eng-review`  | 锁定架构、数据流、边界情况和测试。 |
| `/plan-design-review` | 为每个设计维度评分0-10，并说明10分标准。 |
| `/plan-devex-review` | DX 模式评审：TTHW、魔法时刻、易触及的风险和人员追踪。 |
| `/plan-tune`  | 为 AskUserQuestion 的敏感性做自我调优。 |
| `/autoplan`  | 一键运行 CEO → 设计 → 工程 → DX 评审。 |
| `/design-consultation` | 从零开始构建完整的设计系统。 |
| `/spec`  | 将模糊意图转化为精确、可执行的5阶段规格。自动发起 GitHub PR；可选在 `%` 中启动 Claude Code 代理。 |

### 实施 + 审查

| 技能 | 作用 |
|-------|-------------|
| `/review`  | 落地前 PR 审查。发现能通过 CI 但在 prod 中崩溃的bug。 |
| `/codex`  | 通过 OpenAI Codex 获取第二意见。审查、质疑或咨询模式。 |
| `/investigate`  | 系统地追查根因。没有调查就不修复。 |
| `/design-review`  | 现场站点视觉审计 + 循环修复与原子提交。 |
| `/design-shotgun` | 生成多个 AI 设计变体、对比展示板并进行迭代。 |
| `/design-html` | 生产级质量的 Pretext 原生 HTML/CSS。 |
| `/devex-review`  | 现场 DX 审计（对比流程实际测量 TTHW）。 |
| `/qa`  | 打开真实浏览器、发现问题、修复、重新验证。 |
| `/qa-only`  | 与 /qa 相同方法，但仅报告 — 无代码变更。 |
| `/scrape`  | 从网页中提取数据。首次调用：原型设计；已编码：约200ms。 |
| `/skillify`  | 将最近成功的 `/scrape` 流编码为永久性浏览器 skill。 |

### 发布 + 部署

| 技能 | 作用 |
|-------|-------------|
| `/ship`  | 运行测试、审查、推送、打开 PR。工作区感知的版本队列。 |
| `/land-and-deploy` | 合并 PR，等待 CI 和部署，验证生产健康。 |
| `/canary`  | 部署后使用浏览守护进程进行循环监控。 |
| `/landing-report` | 只读的工作区感知的船运队列仪表盘。 |
| `/document-release` | 更新所有文档以匹配刚发布的内容。 |
| `/document-generate` | 从代码生成 Diataxis 文档（教程/操作/参考/说明）。 |
| `/setup-deploy` | 一次性部署配置检测（Fly.io、Render、Vercel 等）。 |
| `/gstack-upgrade` | 将 gstack 更新到最新版本。 |

### 运维 + 记忆

| 技能 | 作用 |
|-------|-------------|
| `/context-save` | 保存工作上下文（git 状态、决策、剩余工作）。 |
| `/context-restore` | 从保存的上下文恢复，甚至可跨 Conductor 工作区。 |
| `/learn`  | 管理 gstack 在会话中学习到的内容。 |
| `/retro`  | 每周回顾，包含个人分解和发货记录。 |
| `/health`  | 代码质量仪表盘（类型检查、lint、测试、死代码）。 |
| `/benchmark`  | 性能回归检测（页面加载、Core Web Vitals）。 |
| `/benchmark-models` | 跨模型技能基准（Claude、GPT、Gemini 并排对比）。 |
| `/cso`  | OWASP Top 10 + STRIDE 安全审计。 |
| `/setup-gbrain` | 设置 gbrain 进行跨机器会话记忆同步。 |
| `/sync-gbrain` | 让 gbrain 与此仓库的代码保持同步；在 CLAUDE.md 中刷新代理搜索指南。 |

### 浏览器 + 代理集成

| 技能 | 作用 |
|-------|-------------|
| `/browse`  | 无头浏览器 — 真实 Chromium，真实点击，约100ms/命令。 |
| `/open-gstack-browser` | 启动可见的 GStack 浏览器（含侧栏 + 隐身模式）。 |
| `/setup-browser-cookies` | 从真实浏览器导入 cookie 用于认证测试。 |
| `/pair-agent` | 将远程 AI 代理（OpenClaw、Codex 等）与你的浏览器配对。 |

### iOS QA — 通过 USB 或 Tailscale 驱动真实 iPhone（v1.43.0.0+）

| 技能 | 作用 |
|-------|-------------|
| `/ios-qa` | 通过 USB CoreDevice 隧道 + 嵌入式 StateServer 进行实时设备 iOS QA。可选择性地通过 Tailscale 暴露设备，让远程代理可以驱动它。 |
| `/ios-fix` | 带回归快照捕获的自主 iOS bug 修复工具。 |
| `/ios-design-review` | 真机上的设计师视角 QA — 10维度 Apple HIG 评分板。 |
| `/ios-clean` | 便利工具：在 Release 构建前剥离 DebugBridge + #if DEBUG 接线。 |
| `/ios-sync` | 根据最新的上游模板重新生成 iOS 调试桥。 |

配套 CLI（在连接设备的 Mac 上运行）：

| 命令 | 作用 |
|---------|-------------|
| `gstack-ios-qa-daemon` | Mac 端代理。默认环回；`--tailnet` 添加 Tailscale 面向监听器，含能力等级和审计日志。 |
| `gstack-ios-qa-mint` | 用于 tailnet 白名单的机主授权 CLI（`grant`/`revoke`/`list`）。 |

端到端指南：[docs/howto-ios-testing-with-gstack.md](docs/howto-ios-testing-with-gstack.md)。

### 安全 + 范围控制

| 技能 | 作用 |
|-------|-------------|
| `/careful`  | 破坏性命令前警告（rm -rf、DROP TABLE、强制推送）。 |
| `/freeze`  | 将编辑锁定到一个目录。硬性阻止，不仅是警告。 |
| `/guard`  | 同时激活 careful + freeze。 |
| `/unfreeze` | 移除目录编辑限制。 |
| `/make-pdf` | 将任何 markdown 文件转换为出版质量 PDF。 |
| `/diagram` | 英文进，图表出：mermaid 源码 + 可编辑 .excalidraw + SVG/PNG，离线使用。 |

## 构建命令

```bash
bun install              # 安装依赖
bun test                 # 运行免费测试（无 API 花费）
bun run test:windows     # Windows 安全子集（在 windows-latest 上运行）
bun run build            # 生成文档 + 编译二进制
bun run gen:skill-docs   # 从模板重新生成 SKILL.md 文件
bun run skill:check      # 所有技能的健康仪表盘
```

## 平台支持

- **macOS** + **Linux**：支持完整测试套件。
- **Windows**：在 `windows-latest` 上通过 `windows-free-tests` CI 任务
  运行精选的 Windows 安全子集。设置脚本 (`./setup`) 目前需要 Git Bash 或
  MSYS；未来将支持原生 PowerShell。`bin/gstack-paths` 辅助工具通过 `CLAUDE_PLUGIN_DATA` / `GSTACK_HOME` 解析状态
  根路径，使插件安装在各平台都能工作。

## 关键约定

- SKILL.md 文件是从 `.tmpl` 模板**生成**的。编辑模板，而非输出文件。
- 运行 `bun run gen:skill-docs --host codex` 重新生成 Codex 专用输出。
- 浏览二进制提供无头浏览器访问。在技能中使用 `$B <command>`。
- 安全技能（careful、freeze、guard）使用内联建议性说明 — 破坏性操作前必须确认。
- 状态路径通过 `bin/gstack-paths` 解析（通过 `eval "$(...)"` 引入）。
  遵循 `GSTACK_HOME`、`CLAUDE_PLUGIN_DATA`、`CLAUDE_PLANS_DIR`。
- `claude` CLI 二进制通过 `browse/src/claude-bin.ts` 解析（`Bun.which()` + `GSTACK_CLAUDE_BIN` 重写）。
  设置 `GSTACK_CLAUDE_BIN=wsl` 加 `GSTACK_CLAUDE_BIN_ARGS='["claude"]'` 以通过 WSL 在 Windows 上运行 Claude。
