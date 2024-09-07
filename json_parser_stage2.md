
# 阶段 2：实现 JSON 解析器的具体功能与业务需求

**目标**：构建一个 JSON 解析器，它能够接收客户端发送的 JSON 请求，并将其解析为对应的数据库操作。这个解析器是项目的核心，负责根据请求的内容，动态生成 SQL 查询，并执行 CRUD（创建、读取、更新、删除）操作。

---

## 功能需求

### 1. 接收 JSON 请求
- 解析客户端发送的 JSON 格式请求，识别不同操作类型（如查询、插入、更新、删除）。
- 确定请求所针对的表名和字段。
- 支持嵌套的 JSON 结构，处理复杂查询和多表关联。

### 2. 操作类型识别与映射
- 根据 JSON 请求中的 `method` 或类似标识符，自动识别操作类型（如 `GET`、`POST`、`PUT`、`DELETE`），并将其映射为 SQL 操作。
- 例如：
  - `GET` → 查询（`SELECT`）
  - `POST` → 插入（`INSERT`）
  - `PUT` → 更新（`UPDATE`）
  - `DELETE` → 删除（`DELETE`）

### 3. 字段映射与解析
- 解析 JSON 中指定的字段，确保这些字段映射到数据库表中的列。
- 例如，如果请求中指定了 `{"name": "John", "age": 30}`，解析器将生成 SQL 查询，选择或操作 `name` 和 `age` 字段。
- 支持字段的筛选和别名映射，允许指定哪些字段需要返回。

### 4. 条件解析
- 支持在 JSON 中通过条件表达式过滤数据（如 `where` 子句）。
- 例如：`{"age": { "gt": 18 }}` 应该生成 `WHERE age > 18` 的 SQL 条件。
- 支持以下常见的条件运算符：
  - `=`（等于）
  - `!=`（不等于）
  - `>`（大于）
  - `<`（小于）
  - `>=`（大于等于）
  - `<=`（小于等于）
  - `LIKE`（模糊匹配）
  - `IN`（集合包含）
  - `BETWEEN`（范围查询）
- 支持 `AND` 和 `OR` 逻辑运算，允许组合多个查询条件。

### 5. 分页与排序
- 支持 `limit` 和 `offset` 来处理分页请求。
- 解析 `orderBy` 和 `sort`，支持对查询结果按指定字段进行升序或降序排列。
- 例如：`{"page": 1, "count": 10, "orderBy": "age", "sort": "desc"}` 应该生成 `LIMIT 10 OFFSET 0 ORDER BY age DESC`。

### 6. 多表关联（Join）
- 支持通过 JSON 请求进行多表关联查询（如 SQL 中的 `JOIN`）。
- 解析 JSON 中的表关联关系，并生成相应的 `JOIN` 语句。
- 例如：`{"join": [{"table": "orders", "on": "users.id = orders.user_id"}]}` 应该生成 `INNER JOIN orders ON users.id = orders.user_id`。

### 7. 动态表名与数据库切换
- 根据 JSON 请求中的表名字段（如 `table`）确定操作的表。
- 支持在请求中指定数据库，允许在不同的数据库中操作数据。

### 8. 结果过滤与映射
- 支持根据请求中的字段筛选条件，返回指定的字段。
- 例如：`{"fields": ["name", "age"]}` 只返回 `name` 和 `age` 字段的值。

### 9. 错误处理
- 当请求中的 JSON 格式不符合预期，或者指定的字段或操作无效时，解析器应该返回有意义的错误信息。
- 错误类型包括但不限于：
  - JSON 格式错误
  - 表名或字段不存在
  - 操作类型无效
  - SQL 生成错误

---

## 业务需求

### 1. 动态查询
- 后端需要根据用户的需求，接受和处理动态查询请求。例如，用户可以发送带有任意条件的查询 JSON，系统应自动生成对应的 SQL 语句并返回结果，而不需要开发人员编写特定的查询代码。

### 2. 多样化数据操作
- 后端支持通过一个统一的接口，完成创建、读取、更新和删除（CRUD）操作，用户通过不同的 JSON 请求即可完成相应的数据库操作。

### 3. 数据过滤与权限控制
- 支持对数据的过滤和权限控制。根据用户身份和角色限制可以访问的表、字段或数据范围。JSON 解析器需要与权限模块集成，确保用户请求只返回他们有权限访问的数据。

### 4. 性能与扩展性
- JSON 解析器需要具备足够的扩展性和高效性。要支持高并发的请求处理，并且能够扩展到不同的数据库和复杂查询场景。
- 解析器还需要易于扩展，后续可以添加支持更多高级查询功能（如 `GROUP BY`、聚合函数等）。

### 5. 日志和监控
- 每次 JSON 解析器生成的 SQL 操作应该有详细的日志记录，便于后期调试和监控。
- 日志记录应包括请求的解析耗时、SQL 查询执行时间、响应时间等关键信息。

---

## 具体功能实现示例

### 1. 查询请求示例
```json
{
  "method": "GET",
  "table": "users",
  "fields": ["id", "name", "age"],
  "where": {
    "age": { "gt": 18 },
    "name": { "like": "%John%" }
  },
  "orderBy": "age",
  "sort": "desc",
  "page": 1,
  "count": 10
}
```
- 生成 SQL：
```sql
SELECT id, name, age FROM users 
WHERE age > 18 AND name LIKE '%John%' 
ORDER BY age DESC LIMIT 10 OFFSET 0;
```

### 2. 插入请求示例
```json
{
  "method": "POST",
  "table": "users",
  "data": {
    "name": "John",
    "age": 25,
    "email": "john@example.com"
  }
}
```
- 生成 SQL：
```sql
INSERT INTO users (name, age, email) VALUES ('John', 25, 'john@example.com');
```

### 3. 更新请求示例
```json
{
  "method": "PUT",
  "table": "users",
  "where": { "id": 1 },
  "data": {
    "age": 26
  }
}
```
- 生成 SQL：
```sql
UPDATE users SET age = 26 WHERE id = 1;
```

### 4. 删除请求示例
```json
{
  "method": "DELETE",
  "table": "users",
  "where": { "id": 1 }
}
```
- 生成 SQL：
```sql
DELETE FROM users WHERE id = 1;
```

---

## 解析器的最终目标
- **动态性与通用性**：解析器可以处理复杂的、灵活的 JSON 请求，实现任意条件的数据库操作。
- **可扩展性**：易于扩展，未来可以支持更多复杂查询、关联操作、聚合等高级功能。
- **高效性与安全性**：确保解析器能够高效处理请求，生成优化的 SQL 语句，并且保证数据操作的安全性。
