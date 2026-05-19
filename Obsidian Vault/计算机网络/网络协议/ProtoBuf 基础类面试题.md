
## 1. 什么是 ProtoBuf？

ProtoBuf 是 Google 推出的：

- 一种数据序列化协议
- 类似：
    - JSON
    - XML
    - Thrift
- 但：
    - 更小
    - 更快
    - 强类型
    - 跨语言

核心作用：

- RPC 通讯
- 微服务接口
- MQ 消息传输
- 配置同步
- 存储结构化数据

---

## 2. ProtoBuf 为什么比 JSON 快？

这是经典问题。

核心原因：

|对比项|ProtoBuf|JSON|
|---|---|---|
|数据格式|二进制|文本|
|字段名|用数字 tag|用字符串|
|类型信息|schema 已定义|运行时解析|
|序列化成本|低|高|
|数据大小|小|大|

例如：

JSON：

```
{  "name":"Barry",  "age":18}
```

ProtoBuf：

```
message User {  string name = 1;  int32 age = 2;}
```

传输时：

- 不会传 `"name"`
- 不会传 `"age"`
- 只传：
    - field number
    - value

所以：

- 网络包更小
- CPU 解析更快

---

# ProtoBuf 编码原理（重点）

## 3. ProtoBuf 为什么体积小？

### （1）字段名不参与传输

JSON：

```
"name":"Barry"
```

ProtoBuf：

```
1:Barry
```

---

### （2）Varint 编码

整数使用变长编码：

```
小数字 -> 少字节大数字 -> 多字节
```

例如：

|数值|占用|
|---|---|
|1|1 byte|
|127|1 byte|
|128|2 bytes|

因此：

- 大量小整数场景特别省空间

---

## 4. 什么是 Varint？

ProtoBuf 最核心问题之一。

ProtoBuf 对整数采用：

# 变长编码（Variable Integer）

思想：

```
高位表示是否继续低7位存数据
```

例如：

```
1 byte:1xxxxxxx -> 后面还有0xxxxxxx -> 结束
```

优点：

- 小数字极省空间
- 非常适合：
    - ID
    - 状态码
    - 数量
    - 枚举

---

## 5. 为什么 int32 负数会占很多字节？

高频陷阱题。

因为：

```
Varint 对负数不友好
```

负数：

```
补码高位全是1
```

会导致：

```
int32(-1)
```

可能编码成：

```
10 bytes
```

---

## 6. sint32 和 int32 的区别？

经典题。

`sint32`：

- 使用 ZigZag 编码

解决：

```
负数占用空间过大
```

---

ZigZag：

```
0 -> 0-1 -> 11 -> 2-2 -> 3
```

把：

```
负数映射成小正数
```

再做 Varint。

因此：

|类型|负数效率|
|---|---|
|int32|差|
|sint32|好|

---

# schema 设计类问题

## 7. 为什么 field number 不能随便改？

ProtoBuf 的兼容性核心。

例如：

```
string name = 1;
```

编码时：

```
tag=1
```

如果改成：

```
string name = 2;
```

旧数据：

```
tag=1
```

新程序：

```
读不到
```

**因此：field number 一旦发布不能修改**

---

## 8. 删除字段为什么不能复用 tag？

例如：

```
string name = 1;
```

后面删掉。

如果复用：

```
int32 age = 1;
```

旧数据：

```
1 -> Barry
```

新程序：

```
1 -> age
```

直接解析错乱。

因此：

```
reserved 1;
```

---

## 9. ProtoBuf 如何做向前/向后兼容？

这是架构面试高频。

ProtoBuf 的兼容性：来源于，**Unknown Field（未知字段）**

新版本：

```
string email = 3;
```

旧版本：不认识 field=3。

ProtoBuf：

- 自动跳过
- 不报错

因此：

- 老客户端能读新消息
- 新客户端能读老消息

---

## 10. 哪些修改是兼容的？

### 兼容：

✅ 新增字段

✅ 删除字段（不复用 tag）

✅ optional 字段

### 不兼容：

❌ 修改 tag

❌ 修改字段类型

❌ repeated 改 singular


---

## 11. repeated 底层是什么？

```
repeated string tags = 1;
```

本质：

```
同一个 tag 出现多次
```

---

## 12. map 底层是什么？

ProtoBuf 没有真正 map。

实际上：

```
map<string, int32> scores = 1;
```

等价于：

```
message ScoresEntry {  string key = 1;  int32 value = 2;}repeated ScoresEntry scores = 1;
```

这是经典面试题。

---

## 13. oneof 是什么？

类似：

```
联合类型
```

例如：

```
oneof msg {  string text = 1;  bytes image = 2;}
```

表示：

```
同一时间只能有一个字段存在
```

适合：

- IM 消息
- 事件模型
- 多类型 payload

---

# gRPC 相关

## 14. gRPC 为什么使用 ProtoBuf？

经典问题。

因为：

ProtoBuf：

- 更快
- 更小
- 强 schema
- 自动生成代码

特别适合：

- 微服务高频调用
- 内网 RPC

---

## 15. ProtoBuf 和 JSON 怎么选？

### ProtoBuf 适合：

- 微服务
- RPC
- MQ
- 高性能场景
- 内网通讯

### JSON 适合：

- 前后端接口
- 可读性要求高
- 调试方便
- 开放 API

---

# Go 相关问题（你很容易被问）

## 16. Go 中 ProtoBuf 代码如何生成？

一般：

```
protoc --go_out=. user.proto
```

gRPC：

```
protoc \  --go_out=. \  --go-grpc_out=. \  user.proto
```

---

## 17. 为什么 ProtoBuf 生成代码不用反射也能快？

因为：

ProtoBuf：

- schema 编译期已知
- 生成静态代码

不像 JSON：

```
json.Unmarshal
```

很多时候：

- 依赖反射
- 动态解析

因此：

ProtoBuf CPU 开销更低。

---

# 深水区面试题（高级）

## 18. ProtoBuf 为什么不适合直接存数据库？

因为：

- 不可读
- 难 SQL 查询
- 难索引
- schema 演进复杂

通常：

- RPC 传输
- MQ 消息

更适合。

---

## 19. ProtoBuf 为什么跨语言？

因为：

.proto：

本质是：

# IDL（接口描述语言）

通过 protoc：

生成：

- Go
- Java
- Python
- C++
- Rust

代码。

---

## 20. ProtoBuf 和 Thrift 的区别？

经典对比。

|对比|ProtoBuf|Thrift|
|---|---|---|
|公司|Google|Facebook|
|性能|高|高|
|生态|更强|较弱|
|gRPC 支持|原生|弱|
|学习成本|低|中|

现在：

- 云原生
- K8s
- 微服务

基本更偏 ProtoBuf + gRPC。

---

## ProtoBuf 为什么适合 MQ？

因为：

- 消息体小
- 吞吐高
- schema 固定
- 跨语言

适合：

- Kafka
- Pulsar
- RocketMQ

---

## ProtoBuf 和 GraphQL 冲突吗？

**不冲突**

外部：GraphQL

内部： ProtoBuf RPC

---

## ProtoBuf 为什么适合微服务？

核心：

- 强 schema
- 自动生成 SDK
- 高性能
- 多语言统一接口

---

# 总结

> ProtoBuf 本质是二进制序列化协议，相比 JSON 它不传输字段名，而是使用 field number，同时整数采用 Varint 变长编码，因此消息体更小。  
> 另外 ProtoBuf 基于 schema 生成静态代码，不需要像 JSON 那样大量依赖反射解析，所以 CPU 开销也更低。  
> 在微服务和 RPC 场景下，ProtoBuf 通常配合 gRPC 使用，能够明显降低网络带宽和序列化耗时。