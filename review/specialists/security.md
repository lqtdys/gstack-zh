# 安全专家审查清单

适用范围：当 SCOPE_AUTH=true 或（SCOPE_BACKEND=true 且 diff > 100 行）时

输出：JSON 对象，每行一个发现。Schema：
```json
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"security","summary":"...","fix":"...","fingerprint":"path:line:security","specialist":"security"}
```
可选字段：line, fix, fingerprint, evidence, test_stub。

如果没有发现：输出 `NO FINDINGS`，不输出其他内容。

---

本清单比主 CRITICAL 通道检查更深入。主代理已检查 SQL 注入、竞态条件、LLM 信任和枚举完整性。本专家重点关注 auth/authz 模式、加密误用和攻击面扩展。

## 分类

### 信任边界处的输入验证
- 在控制器/处理程序级别接受用户输入但未验证

- 查询参数直接用于数据库查询或文件路径

- 请求体字段未进行类型检查或 schema 验证即被接受

- 文件上传未进行类型/大小/内容验证

- Webhook 载荷未经签名验证即被处理

### 认证与授权绕过
- 端点缺少身份验证中间件（检查路由定义）

- 授权检查默认"允许"而非"拒绝"

- 角色升级路径（用户可修改自己的角色/权限）

- 直接对象引用漏洞（用户 A 通过更改 ID 访问用户 B 的数据）

- 会话固定或会话劫持机会

- Token/API 密钥验证未检查过期时间

### 注入向量（SQL 之外）
- 通过带用户控制参数的子进程调用进行命令注入

- 模板注入（Jinja2、ERB、Handlebars）使用用户输入

- 目录查询中的 LDAP 注入

- 通过用户控制的 URL（fetch、重定向、webhook 目标）进行 SSRF

- 通过用户控制的文件路径进行路径遍历（../../etc/passwd）

- 通过用户控制的 HTTP 头部值进行头部注入

### 加密误用
- 安全敏感操作使用弱哈希算法（MD5、SHA1）

- 用于令牌或密钥的可预测随机性（Math.random、rand()）

- 对密钥、令牌或摘要进行非恒定时间比较（==）

- 硬编码的加密密钥或 IV

- 密码哈希中缺少盐值

### 密钥曝光
- API 密钥、令牌或密码存在于源代码中（甚至在注释中）

- 密钥被记录在应用日志或错误消息中

- URL 中的凭证（查询参数或基本认证）

- 敏感数据在错误响应中返回给用户

- PII 以纯文本存储而预期应加密

### XSS 通过逃逸通道
- Rails：对用户控制数据使用 .html_safe、raw()

- React：使用带用户内容的 dangerouslySetInnerHTML

- Vue：使用带用户内容的 v-html

- Django：对用户输入使用 |safe、mark_safe()

- 通用：innerHTML 赋值使用未清理的数据

### 反序列化
- 反序列化不可信数据（pickle、Marshal、YAML.load、JSON.parse 的可执行类型）

- 从用户输入或外部 API 接受序列化对象但未进行 schema 验证
