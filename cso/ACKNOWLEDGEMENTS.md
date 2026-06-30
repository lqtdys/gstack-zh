# 致谢

/cso v2 的研究借鉴了安全审计领域的多项成果。特别感谢：

- **[Sentry Security Review](https://github.com/getsentry/skills)** — 基于置信度的报告系统（只有高置信度的发现才被报告）和"报告前先研究"的方法论（追踪数据流、检查上游验证）验证了我们 8/10 的每日置信度门槛。TimOnWeb 在 5 个被测安全技能中将其评为唯一值得安装的一个。
- **[Trail of Bits Skills](https://github.com/trailofbits/skills)** — 审计上下文构建方法论（在找 bug 前先建立心智模型）直接启发了 Phase 0。他们的变体分析概念（发现一个漏洞？搜索整个代码库的相同模式）启发了 Phase 12 的变体分析步骤。
- **[Shannon by Keygraph](https://github.com/KeygraphHQ/shannon)** — 自主 AI 渗透测试工具，在 XBOW 基准测试中达到 96.15%（100/104 利用）。验证了 AI 可以做真正的安全测试，不仅仅是清单扫描。我们的 Phase 12 主动验证是 Shannon 实时操作的静态分析版本。
- **[afiqiqmal/claude-security-audit](https://github.com/afiqiqmal/claude-security-audit)** — AI/LLM 特定的安全检查（提示注入、RAG 中毒、工具调用权限）启发了 Phase 7。他们的框架级自动检测（检测"Next.js"而非仅"Node/TypeScript"）启发了 Phase 0 的框架检测步骤。
- **[Snyk ToxicSkills Research](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)** — 发现 36% 的 AI 代理技能有安全缺陷、13.4% 是恶意的研究结果启发了 Phase 8（技能供应链扫描）。
- **[Daniel Miessler 的个人 AI 基础设施](https://github.com/danielmiessler/Personal_AI_Infrastructure)** — 事件响应手册和保护文件概念为补救和 LLM 安全阶段提供了信息。
- **[McGo/claude-code-security-audit](https://github.com/McGo/claude-code-security-audit)** — 生成可分享报告和可执行 Epic 的想法影响了我们的报告格式演进。
- **[Claude Code Security Pack](https://dev.to/myougatheaxo/automate-owasp-security-audits-with-claude-code-security-pack-4mah)** — 模块化方法（分离的 /security-audit、/secret-scanner、/deps-check 技能）验证了这些是不同的关注点。我们的统一方法牺牲了模块化以换取跨阶段推理。
- **[Anthropic Claude Code Security](https://www.anthropic.com/news/claude-code-security)** — 多阶段验证和置信度评分验证了我们的并行发现验证方法。在开源中发现了 500+ 个零日漏洞。
- **[@gus_argon](https://x.com/gus_aragon/status/2035841289602904360)** — 识别了关键的 v1 盲点：无栈检测（运行所有语言模式）、使用 bash grep 而非 Claude Code 的 Grep 工具、`| head -20` 悄然截断结果，以及序言膨胀。这些直接塑造了 v2 的栈优先方法和 Grep 工具强制要求。
