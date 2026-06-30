# API 合同专家评审清单

范围：当 SCOPE_API=true 时
输出：JSON 对象，每行一条发现。Schema：
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"api-contract","summary":"...","fix":"...","fingerprint":"path:line:api-contract","specialist":"api-contract"}
可选：line, fix, fingerprint, evidence, test_stub。
如果没有发现：输出 `NO FINDINGS` 且不输出其他内容。

---

## 类别

### 破坏性变更
- 从响应体中删除字段（客户端可能依赖它们）
- 更改字段类型（string → number, object → array）
- 向现有端点添加新的必需参数
- 更改 HTTP 方法（GET → POST）或状态码（200 → 201）
- 重命名端点时未将旧路径保留为重定向/别名
- 更改认证要求（公开 → 需认证）

### 版本策略
- 进行破坏性变更但没有版本号升级（v1 → v2）
- 同一 API 中混合使用多种版本策略（URL vs header vs query param）
- 已弃用端点没有下线时间表或迁移指南
- version-specific 逻辑分散在控制器中而非集中

### 错误响应一致性
- 新端点返回与现有端点不同的错误格式
- 错误响应缺少标准字段（错误码、消息、详情）
- HTTP 状态码与错误类型不匹配（200 表示错误，500 表示验证错误）
- 错误消息泄露内部实现细节（堆栈跟踪、SQL）

### 限流与分页
- 新端点缺少限流而类似端点有
- 分页变更（offset → cursor）时没有向后兼容
- 更改页大小或默认限制但没有文档说明
- 分页响应中缺少总数或下一页指示符

### 文档漂移
- OpenAPI/Swagger 规范未更新以匹配新端点或已改参数
- README 或 API 文档在变更后描述旧行为
- 示例请求/响应已不再有效
- 新端点或已改参数缺少文档

### 向后兼容性
- 旧版本客户端：它们会崩溃吗？
- 无法强制更新的移动应用：API 对它们仍然有效吗？
- Webhook 负载变更但没有通知订阅者
- 需要使用新功能的 SDK 或客户端库变更
