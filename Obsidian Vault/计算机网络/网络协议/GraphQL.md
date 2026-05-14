#### 1. Schema（模式定义）

这是 GQL 的核心，定义了 API 的"契约"：

```graphql
type User {
  id: ID!
  name: String!
  age: Int
  posts: [Post]
}
```

- `!` 表示非空（Not Null）
- `[]` 表示数组
- Schema 是前后端共同遵守的类型系统
- [[GraphQL内部术语与AST解析]]

---

#### 2. 三种操作类型

|操作|作用|类比 REST|
|---|---|---|
|`Query`|查询数据（读）|GET|
|`Mutation`|修改数据（写）|POST/PUT/DELETE|
|`Subscription`|实时推送|WebSocket|

graphql

```graphql
# Query 示例
query {
  user(id: "1") {
    name
    posts { title }
  }
}
```

---

#### 3. Resolver（解析器）

Schema 定义"是什么"，Resolver 定义"怎么取"，是实际执行逻辑的地方：

javascript

```javascript
const resolvers = {
  Query: {
    user: (parent, args, context) => {
      return db.findUser(args.id)
    }
  }
}
```

每个字段都可以有独立的 resolver，GraphQL 会自动组合结果。

---

#### 4. Variables 和 Arguments（变量与参数）

graphql

```graphql
# 使用变量，避免硬编码
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
  }
}
```

变量通过 JSON 单独传入，更安全、可复用。

---

#### 5. Fragments（片段）

用于复用字段选择，避免重复：

graphql

```graphql
fragment UserInfo on User {
  id
  name
  email
}

query {
  user(id: "1") { ...UserInfo }
  currentUser { ...UserInfo }
}
```

---

#### 6. N+1 问题与 DataLoader

GraphQL 最常见的性能陷阱：

- 查询 10 个 Post，每个 Post 再查 Author → 触发 11 次 SQL
- 解决方案是 **DataLoader**，将多次请求合并成一次批量查询（batching + caching）

---

#### 7. Introspection（自省）

GraphQL 允许查询 Schema 本身，这是 GraphiQL、文档自动生成等工具的基础：

graphql

```graphql
{
  __schema {
    types { name }
  }
}
```

---

#### 知识优先级建议

```
Schema 定义 → Query/Mutation → Resolver → Variables
       ↓
   Fragment → N+1/DataLoader → Subscription
```