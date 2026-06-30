# Conductor 会话流式 API 提案

## 问题

当 Claude 通过 CDP 控制你的真实浏览器时（gstack `$B connect`），你需要看两个
窗口：**Conductor**（查看 Claude 的思考过程）和 **Chrome**（查看 Claude 的动作）。

gstack 的 Chrome 扩展侧边栏面板显示浏览活动 —— 每个命令、结果和错误。
但要实现*完整*的会话镜像（Claude 的思考过程、工具调用、代码编辑），
侧边栏需要 Conductor 暴露对话流。

## 这启用了什么

gstack Chrome 扩展侧边栏面板中的一个 "会话" 标签页，显示：
- Claude 的思考/内容（为性能截断）
- 工具调用名称 + 图标（Edit、Bash、Read 等）
- 带成本估算的轮次边界
- 对话进行时的实时更新

用户在一个地方看到所有内容 —— Claude 在浏览器中的动作 + Claude
的思考过程 —— 无需切换窗口。

## 提出的 API

### `GET http://127.0.0.1:{PORT}/workspace/{ID}/session/stream`

将 Claude Code 的对话重新作为 NDJSON 事件发出的 SSE 端点。

**事件类型**（重用 Claude Code 的 `--output-format stream-json` 格式）：

```
event: assistant
data: {"type":"assistant","content":"让我检查那个页面...","truncated":true}

event: tool_use
data: {"type":"tool_use","name":"Bash","input":"$B snapshot","truncated_input":true}

event: tool_result
data: {"type":"tool_result","name":"Bash","output":"[snapshot output...]","truncated_output":true}

event: turn_complete
data: {"type":"turn_complete","input_tokens":1234,"output_tokens":567,"cost_usd":0.02}
```

**内容截断：** 流中的工具输入/输出上限 500 字符。完整
数据保留在 Conductor 的 UI 中。侧边栏面板是一个摘要视图，不是替代品。

### `GET http://127.0.0.1:{PORT}/api/workspaces`

列出活跃工作区的发现端点。

```json
{
  "workspaces": [
    {
      "id": "abc123",
      "name": "gstack",
      "branch": "garrytan/chrome-extension-ctrl",
      "directory": "/Users/garry/gstack",
      "pid": 12345,
      "active": true
    }
  ]
}
```

Chrome 扩展通过将浏览器服务器的 git 仓库（从 `/health` 响应中获取）与工作区目录或名称匹配来自动选择工作区。

## 安全性

- **仅 localhost。** 与 Claude Code 自己调试输出相同的信任模型。
- **无需身份验证。** 如果 Conductor 需要身份验证，请在工作区列表中包含一个 Bearer token，扩展在 SSE 请求中传递。
- **内容截断** 是一个隐私功能 —— 长代码输出、文件内容和敏感工具结果永远离开不了 Conductor 的完整 UI。

## gstack 构建的（扩展侧）

已搭建在侧边栏面板 "会话" 标签页中（当前显示占位符）。

当 Conductor 的 API 可用时：
1. 侧边栏面板通过端口探测或手动输入发现 Conductor
2. 获取 `/api/workspaces` 并匹配到浏览器服务器的存储库
3. 打开 `EventSource` 到 `/workspace/{id}/session/stream`
4. 渲染：助手消息、工具名称 + 图标、轮次边界、成本
5. 优雅降级："连接 Conductor 以获取完整会话视图"

估计工作量：`sidepanel.js` 中约 200 LOC。

## Conductor 构建的（服务器侧）

1. 每个工作区重新发出 Claude Code 流式 JSON 的 SSE 端点
2. `/api/workspaces` 发现端点，带活跃工作区列表
3. 内容截断（工具输入/输出 500 字符上限）

估计工作量：如果 Conductor 内部已经在捕获 Claude Code 流式 JSON（它为自己的 UI 渲染做这件事），约 100-200 LOC。

## 设计决策

| 决策 | 选择 | 原理 |
|----------|--------|-----------|
| 传输 | SSE（不是 WebSocket） | 单向、自动重连、更简单 |
| 格式 | Claude 的 stream-json | Conductor 已经在解析它；无新架构 |
| 发现 | HTTP 端点（不是文件） | Chrome 扩展无法读取文件系统 |
| 身份验证 | 无（localhost） | 与浏览器服务器、CDP 端口、Claude Code 相同 |
| 截断 | 500 字符 | 侧边栏约 300px 宽；长内容无用 |
