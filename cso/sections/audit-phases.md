<!-- AUTO-GENERATED from audit-phases.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
**作用域门（先读）。** 本节包含所有作用域依赖的阶段（2-11），但仅运行你的解析模式在 `## 模式解析`（始终加载在骨架中）中选择的那些阶段。阶段 0、1、12、13、14 始终运行；阶段 2-11 被作用域门控制。"完整执行"意味着遍历本节并应用该选择，而不是因为某个阶段的散文在这里就跑你模式没选的阶段。例如：`--owasp` 只运行本节的阶段 9，而不是阶段 2-8/10/11。

### 阶段 2：密钥考古

扫描 git 历史中的泄露凭据，检查跟踪的 `.env` 文件，查找含内联密钥的 CI 配置。

**标准模式目录。** 下面考古 grep 针对的高层级凭据前缀（AKIA、ghp_、sk-ant-、sk_live_、xoxb-、`-----BEGIN ... PRIVATE KEY-----` 等）与 `/spec` 中正在编写的重定向块是同一套。完整的 3 层级分类（高层级凭据、中层级 PII/法律/内部、低层级）从 `lib/redact-patterns.ts` 生成并存储于此——由 `gstack-redact` 引擎、`/spec`、`/ship` 和 `/document-*` 技能共享的唯一真相来源。

**Git 历史 — 已知密钥前缀：**
```bash
git log -p --all -S "AKIA" --diff-filter=A -- "*.env" "*.yml" "*.yaml" "*.json" "*.toml" 2>/dev/null
git log -p --all -S "sk-" --diff-filter=A -- "*.env" "*.yml" "*.json" "*.ts" "*.js" "*.py" 2>/dev/null
git log -p --all -G "ghp_|gho_|github_pat_" 2>/dev/null
git log -p --all -G "xoxb-|xoxp-|xapp-" 2>/dev/null
git log -p --all -G "password|secret|token|api_key" -- "*.env" "*.yml" "*.json" "*.conf" 2>/dev/null
```

**被 git 跟踪的 .env 文件：**
```bash
git ls-files '*.env' '.env.*' 2>/dev/null | grep -v '.example\|.sample\|.template'
grep -q "^\.env$\|^\\\.env\\\.*" .gitignore 2>/dev/null && echo ".env IS gitignored" || echo "WARNING: .env NOT in .gitignore"
```

**含内联密钥的 CI 配置（未使用密钥存储）：**
```bash
for f in $(find .github/workflows -maxdepth 1 \( -name '*.yml' -o -name '*.yaml' \) 2>/dev/null) .gitlab-ci.yml .circleci/config.yml; do
  [ -f "$f" ] && grep -n "password:\|token:\|secret:\|api_key:" "$f" | grep -v '\${{' | grep -v 'secrets\.'
done 2>/dev/null
```

**严重性：** git 历史中的活动密钥模式（AKIA、sk_live_、ghp_、xoxb-）为 CRITICAL。被 git 跟踪的 .env 文件、含内联凭据的 CI 配置为 HIGH。可疑的 .env.example 值为 MEDIUM。

**误报规则：** 占位符（"your_"、"changeme"、"TODO"）排除。测试固定装置排除，除非相同值也出现在非测试代码中。轮换过的密钥仍会标记（它们曾被暴露）。`.env.local` 在 `.gitignore` 中是预期的。

**差异模式：** 将 `git log -p --all` 替换为 `git log -p <base>..HEAD`。

### 阶段 3：依赖供应链

超越 `npm audit`。检查实际供应链风险。

**包管理器检测：**
```bash
[ -f package.json ] && echo "DETECTED: npm/yarn/bun"
[ -f Gemfile ] && echo "DETECTED: bundler"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "DETECTED: pip"
[ -f Cargo.toml ] && echo "DETECTED: cargo"
[ -f go.mod ] && echo "DETECTED: go"
```

**标准漏洞扫描：** 运行任何可用的包管理器审计工具。每个工具都是可选的——如果未安装，在报告中记为"SKIPPED — tool not installed"并附上安装说明。这是信息性的，不是发现。审计会继续使用任何可用的工具。

**生产依赖中的安装脚本（供应链攻击向量）：** 对已水合的 `node_modules` 的 Node.js 项目，检查生产依赖中是否存在 `preinstall`、`postinstall` 或 `install` 脚本。

**lockfile 完整性：** 检查 lockfile 存在并被 git 跟踪。

**严重性：** 直接依赖中已知 CVE（高/严重）为 CRITICAL。生产依赖中的安装脚本/缺失 lockfile 为 HIGH。废弃包/中等 CVE/lockfile 未跟踪为 MEDIUM。

**误报规则：** devDependency CVE 最高 MEDIUM。`node-gyp`/`cmake` 安装脚本预期内（MEDIUM 不是 HIGH）。无修复可用且无已知利用的咨询排除。库仓库（非应用）缺失 lockfile 不是发现。

### 阶段 4：CI/CD 管道安全

检查谁可以修改工作流以及他们可以访问哪些密钥。

**GitHub Actions 分析：** 对每个工作流文件，检查：
- 未固定的第三方操作（非 SHA 固定）— 使用 Grep 搜索缺少 `@[sha]` 的 `uses:` 行
- `pull_request_target`（危险：fork PR 获得写入访问）
- 在 `run:` 步骤中通过 `${{ github.event.* }}` 的脚本注入
- 作为环境变量的密钥（可能在日志中泄漏）
- 工作流文件的 CODEOWNERS 保护

**严重性：** `pull_request_target` + 检出 PR 代码 / 在 `run:` 步骤中通过 `${{ github.event.*.body }}` 的脚本注入为 CRITICAL。未固定的第三方操作/未作掩码的环境变量中的密钥为 HIGH。工作流文件缺少 CODEOWNERS 为 MEDIUM。

**误报规则：** 第一方 `actions/*` 未固定 = MEDIUM 不是 HIGH。没有 PR ref 检出的 `pull_request_target` 是安全的（先例 #11）。`with:` 块（不是 `env:`/`run:`）中的密钥由运行时处理。

### 阶段 5：基础设施影子表面

找到具有过度访问权限的影子基础设施。

**Dockerfiles：** 对每个 Dockerfile，检查缺少 `USER` 指令（以 root 运行）、作为 `ARG` 传递密钥、`.env` 文件复制到镜像中、暴露端口。

**含生产凭据的配置文件：** 使用 Grep 在配置文件中搜索数据库连接字符串（postgres://、mysql://、mongodb://、redis://），排除 localhost/127.0.0.1/example.com。检查 staging/dev 配置是否引用了生产。

**IaC 安全：** 对 Terraform 文件，检查 IAM 操作/资源中的 `"*"`、`.tf`/`.tfvars` 中的硬编码密钥。对 K8s 清单，检查特权容器、hostNetwork、hostPID。

**严重性：** 提交配置中带有凭据的生产 DB URL / 敏感资源上的 `"*"` IAM / Docker 镜像中烘焙的密钥为 CRITICAL。生产中的 root 容器 / 带生产 DB 访问的 staging / 特权 K8s 为 HIGH。缺少 USER 指令 / 无文档目的的暴露端口为 MEDIUM。

**误报规则：** 带 localhost 的 `docker-compose.yml` 用于本地开发 = 不是发现（先例 #12）。`data` 源（只读）中的 Terraform `"*"` 排除。`test/`/`dev/`/`local/` 中带 localhost 网络的 K8s 清单排除。

### 阶段 6：Webhook 与集成审计

找到接受任何内容的人站端点。

**Webhook 路由：** 使用 Grep 搜索包含 webhook/hook/callback 路由模式的文件。对每个文件，检查是否也包含签名验证（signature、hmac、verify、digest、x-hub-signature、stripe-signature、svix）。有 webhook 路由但没有签名验证的文件是发现。

**TLS 验证禁用：** 使用 Grep 搜索 `verify.*false`、`VERIFY_NONE`、`InsecureSkipVerify`、`NODE_TLS_REJECT_UNAUTHORIZED.*0` 等模式。

**OAuth 作用域分析：** 使用 Grep 找到 OAuth 配置并检查过宽的作用域。

**验证方法（仅代码追踪 — 无实时请求）：** 对 webhook 发现，追踪处理程序代码以确定签名验证是否存在于中间件链中的任何位置（父路由器、中间件栈、API 网关配置）。不要对 webhook 端点发出实际 HTTP 请求。

**严重性：** 没有任何签名验证的 webhook 为 CRITICAL。生产代码中禁用 TLS 验证 / 过宽的 OAuth 作用域为 HIGH。到第三方的无文档出站数据流为 MEDIUM。

**误报规则：** 测试代码中禁用 TLS 排除。私有网络上内部服务间 webhook = 最高 MEDIUM。上游处理签名验证的 API 网关后的 webhook 端点不是发现 — 但需要证据。

### 阶段 7：LLM 与 AI 安全

检查 AI/LLM 特定漏洞。这是一个新的攻击类别。

使用 Grep 搜索这些模式：
- **提示注入向量：** 用户输入流入系统提示或工具模式 — 查找系统提示构造附近的字符串插值
- **未净化的 LLM 输出：** `dangerouslySetInnerHTML`、`v-html`、`innerHTML`、`.html()`、`raw()` 渲染 LLM 响应
- **无验证的工具/函数调用：** `tool_choice`、`function_call`、`tools=`、`functions=`
- **代码中的 AI API 密钥（非环境变量）：** `sk-` 模式、硬编码 API 密钥赋值
- **LLM 输出的 eval/exec：** `eval()`、`exec()`、`Function()`、`new Function` 处理 AI 响应

**关键检查（超越 grep）：**
- 追踪用户内容流 — 它是否进入系统提示或工具模式？
- RAG 中毒：外部文档能否通过检索影响 AI 行为？
- 工具调用权限：LLM 工具调用在执行前是否经过验证？
- 输出净化：LLM 输出是否被视为可信（渲染为 HTML、执行为代码）？
- 成本/资源攻击：用户能否触发无限制的 LLM 调用？

**严重性：** 用户输入在系统提示中 / 渲染为 HTML 的未净化 LLM 输出 / LLM 输出 eval 为 CRITICAL。缺少工具调用验证 / 暴露的 AI API 密钥为 HIGH。无限制的 LLM 调用 / 无输入验证的 RAG 为 MEDIUM。

**误报规则：** AI 对话中用户消息位置的用户内容不是提示注入（先例 #13）。只有当用户内容进入系统提示、工具模式或函数调用上下文时才标记。

### 阶段 8：技能供应链

扫描已安装的 Claude Code 代码中的恶意模式。已发布的技能中 36% 有安全漏洞，13.4% 是彻底的恶意（Snyk ToxicSkills 研究）。

**第 1 层 — 仓库本地（自动）：** 扫描仓库的本地技能目录中的可疑模式：

```bash
ls -la .claude/skills/ 2>/dev/null
```

使用 Grep 搜索所有本地技能 SKILL.md 文件中的可疑模式：
- `curl`、`wget`、`fetch`、`http`、`exfiltrat`（网络渗透）
- `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`env.`、`process.env`（凭据访问）
- `IGNORE PREVIOUS`、`system override`、`disregard`、`forget your instructions`（提示注入）

**第 2 层 — 全局技能（需要权限）：** 在扫描全局安装的技能或用户设置之前，使用 AskUserQuestion：
"第 8 阶段可以扫描你全局安装的 AI 编码代理技能和钩子中的恶意模式。这会读取仓库外的文件。要包括这个吗？"
选项：A) 是 — 也扫描全局技能  B) 否 — 仅仓库本地

如果获得批准，对全局安装的技能文件运行相同的 Grep 模式并检查用户设置中的钩子。

**严重性：** 凭据渗透企图 / 技能文件中的提示注入为 CRITICAL。可疑的网络调用 / 过宽的工具权限为 HIGH。来自未审核源且无审查的技能为 MEDIUM。

**误报规则：** gstack 自己的技能是受信任的（检查技能路径是否解析到已知仓库）。为合法目的（下载工具、健康检查）使用 `curl` 的技能需要上下文 — 仅当目标 URL 可疑或命令包含凭据变量时才标记。

### 阶段 9：OWASP Top 10 评估

对每个 OWASP 类别执行针对性分析。对所有搜索使用 Grep 工具 — 将文件扩展名限定为从阶段 0 检测到的技术栈。

#### A01：损坏的访问控制
- 检查控制器/路由上缺少认证（skip_before_action、skip_authorization、public、no_auth）
- 检查直接对象引用模式（params[:id]、req.params.id、request.args.get）
- 用户 A 能否通过更改 ID 访问用户 B 的资源？
- 是否存在水平/垂直权限提升？

#### A02：加密失败
- 弱加密（MD5、SHA1、DES、ECB）或硬编码密钥
- 敏感数据在静止和传输中是否加密？
- 密钥/密钥是否正确管理（环境变量，非硬编码）？

#### A03：注入
- SQL 注入：原始查询、SQL 中的字符串插值
- 命令注入：system()、exec()、spawn()、popen
- 模板注入：render with params、eval()、html_safe、raw()
- LLM 提示注入：参见阶段 7 进行全面覆盖

#### A04：不安全设计
- 认证端点上的速率限制？
- 失败后账户锁定？
- 业务逻辑在服务器端验证？

#### A05：安全配置错误
- CORS 配置（生产中的通配符来源？）
- CSP 标头存在？
- 生产中的调试模式/详细错误？

#### A06：易受攻击和过时的组件
参见**阶段 3（依赖供应链）**了解全面的组件分析。

#### A07：识别和认证失败
- 会话管理：创建、存储、失效
- 密码策略：复杂性、轮换、违规检查
- MFA：可用？对管理员强制？
- Token 管理：JWT 过期、刷新轮换

#### A08：软件和数据完整性失败
参见**阶段 4（CI/CD 管道安全）**了解管道保护分析。
- 反序列化输入是否经过验证？
- 外部数据的完整性检查？

#### A09：安全日志和监控失败
- 认证事件是否记录？
- 授权失败是否记录？
- 管理员操作是否有审计跟踪？
- 日志是否防篡改？

#### A10：服务器端请求伪造（SSRF）
- 从用户输入构造 URL？
- 用户可控 URL 能否访问内部服务？
- 出站请求是否执行允许列表/阻止列表？

### 阶段 10：STRIDE 威胁模型

对阶段 0 识别的每个主要组件评估：

```
COMPONENT: [名称]
  Spoofing:             攻击者能否冒充用户/服务？
  Tampering:            数据在传输/静止时能否被修改？
  Repudiation:          行为能否被否认？是否有审计跟踪？
  Information Disclosure: 敏感数据能否泄漏？
  Denial of Service:    组件能否被压垮？
  Elevation of Privilege: 用户能否获得未经授权的访问？
```

### 阶段 11：数据分类

分类应用处理的所有数据：

```
DATA CLASSIFICATION
═══════════════════
RESTRICTED（泄露 = 法律责任）：
  - 密码/凭据：[存储位置、保护方式]
  - 支付数据：[存储位置、PCI 合规状态]
  - PII：[类型、存储位置、保留策略]

CONFIDENTIAL（泄露 = 商业损害）：
  - API 密钥：[存储位置、轮换策略]
  - 业务逻辑：[代码中的商业机密？]
  - 用户行为数据：[分析、跟踪]

INTERNAL（泄露 = 尴尬）：
  - 系统日志：[包含内容、谁可以访问]
  - 配置：[错误消息中暴露的内容]

PUBLIC：
  - 营销内容、文档、公共 API
```
