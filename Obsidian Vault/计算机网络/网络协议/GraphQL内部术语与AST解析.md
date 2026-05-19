#### 1. GID（Global ID）

Relay 规范中的概念，将类型和数据库 ID 编码成一个全局唯一字符串：

```javascript
// 原始信息
type: "User", id: "123"

// 编码后的 GID（Base64）
"VXNlcjoxMjM="  // btoa("User:123")
```

**意义**：客户端不需要知道类型，只凭一个 ID 就能通过 `node(id: "...")` 查任意对象。

```graphql
# Relay 的 Node 接口
interface Node {
  id: ID!
}

query {
  node(id: "VXNlcjoxMjM=") {
    ... on User { name }
    ... on Post { title }
  }
}
```

---

#### 2. 其他重要内部术语

##### Document（文档）

一次请求中完整的 GQL 文本，包含一个或多个 Operation。

##### Operation（操作）

Document 中的一个 query / mutation / subscription 块，有名字和变量声明。

##### Selection Set（选择集）

`{ }` 括号里的内容，表示"我要哪些字段"：

graphql

```graphql
{ name age posts { title } }  # 这整个就是一个 Selection Set
```

##### Field（字段）

Selection Set 中的每一项，可以有别名：

```graphql
{
  userName: name    # userName 是别名，name 是真实字段
  userAge: age
}
```

##### Directive（指令）

用 `@` 开头，动态控制查询行为：

```graphql
query($showAge: Boolean!) {
  user {
    name
    age @include(if: $showAge)   # 条件包含
    phone @skip(if: true)        # 条件跳过
  }
}
```

##### Inline Fragment（内联片段）

处理联合类型（Union）/ 接口（Interface）时按类型分支：

```graphql
{
  search(text: "hello") {
    ... on User { name }
    ... on Post { title }
  }
}
```

##### Input Type（输入类型）

Mutation 专用的参数类型，与普通 Type 区分：

```graphql
input CreateUserInput {
  name: String!
  email: String!
}

mutation {
  createUser(input: { name: "Tom", email: "t@t.com" }) {
    id
  }
}
```

##### Union vs Interface

| 特性   | Union | Interface |
| :--- | :---- | :-------- |
| 共同字段 | 不需要   | 必须实现      |
| 用途   | 异构结果集 | 共享行为约束    |

```graphql
union SearchResult = User | Post | Comment

interface Animal {
  name: String!
  sound: String!
}
```

---

#### 3. AST 树解析（核心机制）

GraphQL 执行一个请求，内部经历 **5 个阶段**：

```
请求字符串
    ↓
① Lexing（词法分析）
    ↓
② Parsing（语法分析）→ 生成 AST
    ↓
③ Validation（校验）
    ↓
④ Execution（执行）→ 调用 Resolver
    ↓
⑤ Response（返回 JSON）
```

---

##### ① Lexing — 词法分析

把字符串切成 Token 流：

```
"query { user(id: "1") { name } }"

→ Token 流：
[QUERY] [LBRACE] [NAME:user] [LPAREN] [NAME:id]
[COLON] [STRING:"1"] [RPAREN] [LBRACE] [NAME:name] [RBRACE] [RBRACE]
```

##### ② Parsing — 生成 AST

Token 流 → 结构化的抽象语法树：

json

```json
{
  "kind": "Document",
  "definitions": [{
    "kind": "OperationDefinition",
    "operation": "query",
    "selectionSet": {
      "kind": "SelectionSet",
      "selections": [{
        "kind": "Field",
        "name": { "kind": "Name", "value": "user" },
        "arguments": [{
          "kind": "Argument",
          "name": { "kind": "Name", "value": "id" },
          "value": { "kind": "StringValue", "value": "1" }
        }],
        "selectionSet": {
          "kind": "SelectionSet",
          "selections": [{
            "kind": "Field",
            "name": { "kind": "Name", "value": "name" }
          }]
        }
      }]
    }
  }]
}
```

AST 节点的 `kind` 字段标识类型，常见的有：

|Kind|含义|
|---|---|
|Document|整个文档根节点|
|OperationDefinition|query/mutation/subscription|
|SelectionSet|`{}` 选择集|
|Field|一个字段|
|Argument|字段参数|
|FragmentSpread|`...FragmentName`|
|InlineFragment|`... on Type {}`|
|Variable|`$varName`|

##### ③ Validation — 校验

遍历 AST，对照 Schema 检查：

- 字段是否存在
- 参数类型是否匹配
- 变量是否正确使用
- 片段是否有循环引用

##### ④ Execution — 执行

从根节点深度优先遍历 AST，逐字段调用 Resolver：

```
Document
 └── OperationDefinition (query)
      └── Field: user          → 调用 Query.user resolver
           └── Field: name     → 调用 User.name resolver
```

---

#### 4. AST 的实际应用

AST 不只是内部实现，开发中也直接用到：

**代码层面操作 AST（graphql-js）：**

javascript

```javascript
import { parse, visit } from 'graphql'

const ast = parse(`query { user { name } }`)

// 遍历 AST，做自定义处理
visit(ast, {
  Field(node) {
    console.log('访问字段:', node.name.value)
  }
})
```

**常见应用场景：**

- **权限控制** — 在 Validation 阶段分析 AST，拦截敏感字段
- **查询复杂度限制** — 遍历 AST 计算嵌套深度，防止恶意查询
- **Apollo Client 缓存** — 解析 AST 提取 `__typename` 和 `id` 做归一化缓存
- **代码生成** — 根据 AST 自动生成 TypeScript 类型（graphql-codegen）
- **持久化查询** — 将 AST hash 化，客户端只传 hash 而非完整查询