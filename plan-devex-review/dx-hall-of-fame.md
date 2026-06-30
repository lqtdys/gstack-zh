# DX 名人堂参考

仅阅读当前审查通道的部分。不要加载整个文件。

## 通道 1：入门

**黄金标准：**
- **Stripe**：7 行代码即可刷卡。文档在登录时预填充你的测试 API 密钥。Stripe Shell 在文档页面内运行 CLI。无需本地安装。
- **Vercel**：`git push` = 全球 CDN 上的实时站点，带 HTTPS。每个 PR 获得预览 URL。一条 CLI 命令：`vercel`。
- **Clerk**：`<SignIn />`、`<SignUp />`、`<UserButton />`。3 个 JSX 组件，开箱即用的邮箱、社交、MFA 认证。
- **Supabase**：创建 Postgres 表，即时自动生成 REST API + 实时 + 自文档化文档。
- **Firebase**：`onSnapshot()`。3 行代码实现所有客户端的实时同步，内置离线持久化。
- **Twilio**：控制台虚拟电话。无需购买号码、无需信用卡即可收发短信。结果：激活率提升 62%。

**反模式：**
- 在任何价值之前进行邮箱验证（破坏流程）
- 沙盒之前要求信用卡
- "自选冒险"多条路径（决策疲劳；一条黄金路径胜出）
- API 密钥隐藏在设置中（Stripe 将它们预填充到代码示例中）
- 无语言切换的静态代码示例
- 文档站点与仪表板分离（上下文切换）

## 通道 2：API/CLI/SDK 设计

**黄金标准：**
- **Stripe 前缀 ID**：`ch_` 表示收费，`cus_` 表示客户。自文档化。不可能传递错误的 ID 类型。
- **Stripe 可扩展对象**：默认返回 ID 字符串。`expand[]` 获取内联完整对象。嵌套扩展最多 4 层。
- **Stripe 幂等键**：在变更操作上传递 `Idempotency-Key` 头。安全重试。没有"我是否重复收费？"的焦虑。
- **Stripe API 版本控制**：首次调用将账户固定到当天的版本。通过 `Stripe-Version` 头按请求测试新版本。
- **GitHub CLI**：自动检测终端与管道。终端中人类可读，管道时制表符分隔。`gh pr <tab>` 显示所有 PR 操作。
- **SwiftUI 渐进式展开**：`Button("Save") { save() }` 到完全定制，每个级别相同的 API。
- **htmx**：HTML 属性替代 JS。总共 14KB。`hx-get="/search" hx-trigger="keyup changed delay:300ms"`。零构建步骤。
- **shadcn/ui**：将源代码复制到你的项目中。你拥有每一行。无依赖，无版本冲突。

**反模式：**
- 啰嗦 API：一次用户可见操作需要 5 次调用
- 命名不一致：`/users`（复数）vs `/user/123`（单数）vs `/create-order`（URL 中的动词）
- 隐式失败：200 OK 但错误嵌套在响应体中
- 上帝端点：47 种参数组合，每个子集行为不同
- 需要文档的 API：首次调用前 3 页文档 = 太多仪式

## 通道 3：错误消息与调试

**三个错误质量层级：**

**第 1 层，Elm（对话式编译器）：**
```
-- TYPE MISMATCH ---- src/Main.elm
I cannot do addition with String values like this one:
42|   "hello" + 1
     ^^^^^^^
Hint: To put strings together, use the (++) operator instead.
```
第一人称，完整句子，精确位置，建议修复，延伸阅读。

**第 2 层，Rust（带注释的源代码）：**
```
error[E0308]: mismatched types
 --> src/main.rs:4:20
help: consider borrowing here
  |
4 |     let name: &str = &get_name();
  |                       +
```
错误代码链接到教程。主要 + 次要标签。帮助部分显示精确编辑。

**第 3 层，Stripe API（带 doc_url 的结构化）：**
```json
{"error":{"type":"invalid_request_error","code":"resource_missing","message":"No such customer: 'cus_nonexistent'","param":"customer","doc_url":"https://stripe.com/docs/error-codes/resource-missing"}}
```
五个字段，零歧义。

**公式：** 发生了什么 + 为什么 + 如何修复 + 在哪里了解更多 + 导致它的实际值。

**反模式：** TypeScript 将"你的意思是？"埋在长错误链的底部。最有用的信息应该最先出现。

## 通道 4：文档与学习

**黄金标准：**
- **Stripe 文档**：三栏布局（导航 / 内容 / 实时代码）。登录时注入 API 密钥。语言切换器在所有页面保持。悬停高亮。Stripe Shell 用于浏览器内 API 调用。构建并开源了 Markdoc。功能在文档完成前不发布。文档贡献影响绩效评估。
- 52% 的开发者因缺乏文档而受阻（Postman 2023）
- 拥有世界级文档的公司采用率提高 2.5 倍
- "文档即产品"：与功能一起发布，否则功能不发布

## 通道 5：升级与迁移路径

**黄金标准：**
- **Next.js**：`npx @next/codemod upgrade major`。一条命令升级 Next.js、React、React DOM，运行所有相关的 codemod。
- **AG Grid**：v31+ 的每个版本都包含 codemod。
- **Stripe API 版本控制**：内部一个代码库。每个账户版本固定。破坏性变更永远不会让你惊讶。
- **Martin Fowler 的管道模式**：组合小的、可测试的转换，而不是一个单体 codemod。
- Maven Central 中 21.9% 的破坏性变更未记录（Ochoa 等，2021）

## 通道 6：开发者环境与工具

**黄金标准：**
- **Bun**：比 npm install 快 100 倍，比 Node.js 运行时快 4 倍。速度就是 DX。
- 平均每天 87 次中断；每次恢复需要 25 分钟。开发者每天只编码 2-4 小时。
- DXI 每提高 1 分 = 每位开发者每周节省 13 分钟。
- **GitHub Copilot**：任务完成速度提高 55.8%。PR 时间从 9.6 天缩短到 2.4 天。

## 通道 7：社区与生态系统
- 开发工具在购买前需要约 14 次曝光（Matt Biilmann，Netlify）。与季度 OKR 周期不兼容。
- 拥有强大开发者体验的团队有 4-5 倍的性能乘数（DevEx 框架）。

## 通道 8：DX 测量

**三个学术框架：**
1. **SPACE**（Microsoft Research，2021）：满意度、绩效、活动、沟通、效率。至少测量 3 个维度。
2. **DevEx**（ACM Queue，2023）：反馈循环、认知负荷、心流状态。结合感知 + 工作流数据。
3. **Fagerholm & Munch**（IEEE，2021）：认知、情感、意动。心理学的"心灵三部曲"。

## Claude Code 技能 DX 清单

用于审查 Claude Code 技能、MCP 服务器或 AI 代理工具的计划时。

- [ ] **AskUserQuestion 设计**：每次调用一个问题。重新锚定上下文（项目、分支、任务）。视觉反馈的浏览器交接。
- [ ] **状态存储**：全局（~/.tool/）vs 每个项目（$SLUG/）vs 每个会话。追加写入 JSONL 用于审计跟踪。
- [ ] **渐进式同意**：带标记文件的一次性提示。永远不重复询问。可逆。
- [ ] **自动升级**：带缓存的版本检查 + 休眠退避。迁移脚本。内联提供。
- [ ] **技能组合**：利益链。审查链。带部分跳过的内联调用。
- [ ] **错误恢复**：从失败处恢复。保留部分结果。检查点安全。
- [ ] **会话连续性**：时间线事件。压缩恢复。跨会话学习。
- [ ] **有界自主**：清晰的操作限制。破坏性操作的强制升级。审计跟踪。

参考实现：gstack 的 design-shotgun 循环、自动升级流、渐进式同意、分层存储。
