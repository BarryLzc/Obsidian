参考原阿里TMF框架

## 1. 核心接口定义

| **接口**              | **职责**                                                                                                                                                                     |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Template`          | 业务模板，定义 **Pre/Handle/Post** 三阶段生命周期。它让框架可以用统一的 any 接口调度所有 Plugin，而每个具体业务场景的 Template 负责把 any<br>  和强类型之间来回转换，同时控制插件输入的构造方式和多个插件输出的聚合策略。Plugin（租户实现）只需关心自己的强类型业务逻辑，不需要感知框架。 |
| `Plugin`            | 租户级插件，标识自身类型和所属模板（写业务逻辑的地方）                                                                                                                                                |
| `IndependentPlugin` | 独立插件，自己处理逻辑，不经过 Template 委托                                                                                                                                                |
| `Matcher`           | 判断某个 Plugin 是否应该对当前请求执行                                                                                                                                                    |
| `TenantMatcher`     | 从上下文提取租户标识                                                                                                                                                                 |
| `Filter`            | 执行前的过滤/拦截器                                                                                                                                                                 |
| `UniversalTemplate` | 绕过标准生命周期，自定义编排子 Executor                                                                                                                                                   |

---

## 2. 执行流程

### 2.1 请求入口

1. **请求进入** 2. $\rightarrow$ **Adapter** (Gin/gRPC) 封装为 `model.Context`
    
2. $\rightarrow$ `Executor.Handle(context, request)`
    
3. $\rightarrow$ `Executor.HandleIn(context, globalIns)`

### 2.2 HandleIn 内部流程核心步骤

> [!NOTE] 流程说明
> 
> 1. **Filter 链过滤**：任一失败则 `fastFail`。
>     
> 2. **TenantMatcher 提取租户**：写入 context。
>     
> 3. **UniversalTemplate 判定**：若存在则直接调用，跳过后续所有步骤。
>     
> 4. **递归编排**：遍历子 Executor（依赖模板），递归 `HandleIn` 并累积 `globalIns`。
>     
> 5. **插件筛选**：通过 `Matcher` 筛选出匹配的 Plugin 列表。
>     
> 6. **参数构造**：调用 `Template.PreHandlePlugins` 构造插件请求。
>     
> 7. **调度执行**：通过 **Barrier** (ALL/ANY/ONCE) 逐个调用 `Template.HandlePlugin`。
>     
> 8. **结果聚合**：调用 `Template.PostHandlePlugins` 返回最终输出。
>     

---

## 3. Template 生命周期

框架实现了标准的模板方法模式，将执行过程拆解为三个阶段：

1. **PreHandlePlugins(context, globalIns)**
    
    - **作用**：从全局状态提取/构造插件的输入参数。
        
    - **输出**：`in` (具体的请求对象)。
    
2. **HandlePlugin(context, plugin, in, gis)**
    
    - **作用**：Template 将 Plugin 类型断言为业务接口并委托调用。
        
    - **输出**：`Out`, `PluginGlobalOut`, `FastFail`。
    
3. **PostHandlePlugins(context, outs, errs)**
    
    - **作用**：聚合所有插件结果，决定最终返回值。
        
    - **输出**：`TemplateOut`, `FastFail`。
    

---

## 4. Barrier 匹配策略

|**策略**|**行为**|
|---|---|
|**ALL**|执行所有匹配插件，任一 `FastFail` 则中止|
|**ANY**|顺序执行，第一个成功（非 `FastFail`）的结果即返回|
|**ONCE**|严格要求只有 1 个匹配 Plugin，否则报错|

---

## 5. 注册与编排机制

### 5.1 init 自动注册模式

Go

```
// 模板注册 (definition 层)
func init() {
    openmf.RegisterTemplate(ctx, &QueryAppsTemplate{}, model.NewConfig(model.ALL), &TenantMatcher{})
}

// 插件注册 (custom 层)
func init() {
    openmf.RegisterPlugin(&QueryAppsPlugin{}, &Matcher{})
}

// UniversalTemplate 注册
func init() {
    openmf.RegisterUniversalTemplate(ctx, recommend_apps.RecommendApps, &UniversalTemplate{})
}
```

### 5.2 Executor 树 (TOML 配置)

通过依赖关系构建执行树：

Ini, TOML

```
[[openMF.template]]
type = "RecommendApps"
    [[openMF.template.dependencies]]
    type = "QueryApps"
    [[openMF.template.dependencies]]
    type = "QueryStore"
```

**运行时结构：**

- **RecommendApps (Executor)**
    
    - ├── **QueryApps** (子 Executor)
        
    - └── **QueryStore** (子 Executor)
        

---

## 6. 资源管理 (Resource)

资源按 **租户** 维度进行隔离存储：

- `resources[tenant]` $\rightarrow$ 映射关系：`tenant` $\rightarrow$ `resourceType` $\rightarrow$ `supportBusiness` $\rightarrow$ `resourceKey`。
    
- **Plugin 获取方式**：通过 `GetResources(ctx)` 获取对应的 `gorm.DB` 或 `redis.Client`。

---

## 7. 待修复 Bug 清单

- [ ] **Context 丢失**：`executor.go` 中 `context.WithValue` 返回值未赋回。
    
- [ ] **接口不一致**：`UniversalTemplate` 实现中多出了 `resource` 参数，导致编译失败。
    
- [ ] **适配器冗余**：gRPC 适配器直接拷贝自 Gin，参数类型错误。
    
- [ ] **数据一致性**：`allProcess` 存储了 `output` 整体，而 `any/once` 仅存储 `out.Out`。
    
- [ ] **调试代码残留**：`barrier.go` 中存在 `fmt.Printf` 打印指针地址。
    
- [ ] **特性缺失**：`waitTimeout` 已定义但未在异步场景下逻辑实现。

---

## 8. 设计模式总结

- **模板方法**：生命周期三阶段封装。
    
- **策略模式**：Matcher 与 Filter 的可插拔设计。
    
- **组合模式**：Executor 树的递归递归调用。
    
- **适配器模式**：多协议 (Gin/gRPC) 的统一抽象。
    
- **控制反转 (IoC)**：全局 Registry 配合 `init()` 实现插件发现。