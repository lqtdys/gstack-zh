# 测试专家评审清单

范围：始终开启（每次评审）
输出：JSON 对象，每行一条发现。Schema：
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"testing","summary":"...","fix":"...","fingerprint":"path:line:testing","specialist":"testing"}
可选：line, fix, fingerprint, evidence, test_stub。
如果没有发现：输出 `NO FINDINGS` 且不输出其他内容。

---

## 类别

### 缺少负面路径测试
- 处理错误、拒绝或无效输入的新代码路径没有对应测试
- 未测试的守卫子句和提前返回
- try/catch、rescue 或错误边界中的错误分支没有失败路径测试
- 代码中已断言的权限/认证检查从未测试"被拒绝"的情况

### 缺少边界情况覆盖
- 边界值：零、负数、最大整数、空字符串、空数组、nil/null/undefined
- 单元素集合（循环中的差一错误）
- 用户输入中的 Unicode 和特殊字符
- 没有并发访问模式测试

### 测试隔离违规
- 测试共享可变状态（类变量、全局单例、未清理的 DB 记录）
- 顺序相关测试（按顺序通过，随机化后失败）
- 依赖系统时钟、时区或区域设置的测试
- 调用真实网络请求而非使用 stub/mock 的测试

### 不稳定测试模式
- 定时相关断言（sleep, setTimeout, 严格超时的 waitFor）
- 对无序结果（哈希键、Set 迭代、异步解决顺序）的断言
- 依赖外部服务（API、数据库）而没有回退的测试
- 随机化测试数据但没有种子控制

### 安全执行测试缺失
- 控制器中 auth/authz 检查没有测试"未授权"情况
- 限流逻辑没有测试证明它真正拦截
- 输入净化没有测试恶意输入
- CSRF/CORS 配置没有集成测试

### 覆盖缺口
- 新的公共方法/函数零测试覆盖
- 已更改的方法中现有测试只覆盖旧行为而非新分支
- 从多处调用但只间接测试的工具函数
