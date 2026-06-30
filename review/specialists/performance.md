# 性能专家审查清单

适用范围：当 SCOPE_BACKEND=true 或 SCOPE_FRONTEND=true 时

输出：JSON 对象，每行一个发现。Schema：
```json
{"severity":"CRITICAL|INFORMATIONAL","confidence":N,"path":"file","line":N,"category":"performance","summary":"...","fix":"...","fingerprint":"path:line:performance","specialist":"performance"}
```
可选字段：line, fix, fingerprint, evidence, test_stub。

如果没有发现：输出 `NO FINDINGS`，不输出其他内容。

---

## 分类

### N+1 查询
- ActiveRecord/ORM 关联在循环中被遍历但未使用预加载（.includes、joinedload、include）

- 在迭代块（each、map、forEach）内部执行数据库查询，本可批量处理

- 嵌套序列化器触发了懒加载关联

- GraphQL 解析器按字段查询而非批量处理（检查是否使用 DataLoader）

### 缺少数据库索引
- 新增的 WHERE 子句针对无索引的列（检查迁移文件或 schema）

- 新增的 ORDER BY 针对非索引列

- 组合查询（WHERE a AND b）没有组合索引

- 新增的外键列未创建索引

### 算法复杂度
- O(n²) 或更差的模式：对集合的嵌套循环、Array.find 嵌套在 Array.map 中

- 重复的线性搜索，本可使用哈希/映射/集合查找

- 循环内的字符串拼接（应使用 join 或 StringBuilder）

- 对大型集合多次排序或过滤，本可只处理一次

### 包体积影响（前端）
- 新增的重量级生产依赖（moment.js、lodash 完整版、jquery）

- 桶式导入（从 'library' 导入）而非深层导入（从 'library/specific' 导入）

- 大型静态资源（图片、字体）未优化即提交

- 缺少路由级别的代码分割

### 渲染性能（前端）
- 请求瀑布：顺序 API 调用本可并行（Promise.all）

- 因不稳定引用导致的不必要重渲染（render 中创建新对象/数组）

- 缺少 React.memo、useMemo 或 useCallback 来优化昂贵计算

- 循环中读写 DOM 属性导致的布局抖动

- 缺少首屏以下图片的 loading="lazy"

### 缺少分页
- 列表端点返回无边界结果（无 LIMIT，无分页参数）

- 数据库查询未设置 LIMIT，随数据量增长

- API 响应嵌入完整嵌套对象而非使用 ID 加展开

### 异步上下文中的阻塞
- 异步函数内的同步 I/O（文件读取、子进程、HTTP 请求）

- time.sleep() / Thread.sleep() 在基于事件循环的处理器中

- CPU 密集型计算阻塞主线程且未使用 worker 卸载
