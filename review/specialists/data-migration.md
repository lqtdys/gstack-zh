# 数据迁移专项审查清单

范围：当 SCOPE_MIGRATIONS=true 时
输出：JSON 对象，每行一条发现问题。格式：
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"data-migration","summary":"...","fix":"...","fingerprint":"path:line:data-migration","specialist":"data-migration"}
可选字段：line、fix、fingerprint、evidence、test_stub。
若无发现问题：仅输出 `NO FINDINGS`，不输出其他内容。

---

## 分类

### 可逆性
- 此迁移是否可以无数据损失地回滚？
- 是否存在对应的 down/rollback 迁移？
- 回滚操作是否真正撤销了更改，还是仅作 noop（空操作）？
- 回滚是否会破坏当前的应用程序代码？

### 数据丢失风险
- 丢弃仍包含数据的列（应先添加弃用期）
- 更改会截断数据的列类型（varchar(255) → varchar(50)）
- 删除表时未验证是否仍有代码引用
- 重命名列时未更新所有引用（ORM、原生 SQL、视图）
- 对包含现有 NULL 值的列添加 NOT NULL 约束（需先回填数据）

### 锁持续时间
- 在大表上使用 ALTER TABLE 时未使用 CONCURRENTLY（PostgreSQL）
- 在超过 10 万行的表上创建索引时未使用 CONCURRENTLY
- 可合并为一次锁获取的多个 ALTER TABLE 语句
- 在流量高峰时段执行会获取排他锁的架构变更

### 回填策略
- 新增 NOT NULL 列时缺少 DEFAULT 值（需先回填数据再添加约束）
- 新增带有计算默认值的列，需批量填充
- 缺少针对现有记录的回填脚本或 rake 任务
- 回填操作一次性更新所有行而非分批处理（锁定表）

### 索引创建
- 在生产表上 CREATE INDEX 时未使用 CONCURRENTLY
- 重复索引（新索引覆盖了与现有索引相同的列）
- 新外键列缺少索引
- 本应使用全量索引时使用了部分索引（或反之）

### 多阶段安全性
- 必须按特定顺序部署的迁移，需与应用程序代码配合
- 会破坏当前运行代码的架构变更（先部署代码，再迁移）
- 假定有部署边界的迁移（旧代码 + 新架构 = 崩溃）
- 在滚动部署期间缺少用于处理新旧代码混合的功能开关
