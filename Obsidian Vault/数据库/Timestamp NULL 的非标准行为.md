
在 MySQL 的早期版本（5.6.6 之前默认开启）中，`TIMESTAMP` 列有一种被称为 **"Automatic Initialization and Updating"** 的非标准行为。即使你现在使用的是 8.0，如果配置不当或迁移自旧版本，依然会遇到这些陷阱。

### 带来的核心问题：

- **隐式自动填充：** 默认情况下，表中的第一个 `TIMESTAMP` 列如果没有显式声明 `NULL`，MySQL 会自动为其加上 `NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`。
    
    - _后果：_ 当你只想记录“创建时间”时，结果每次更新记录，该时间都会莫名其妙地跳到当前时间。
        
- **NULL 值被篡改：** 在非标准模式下，如果你向一个定义为 `TIMESTAMP` 的列插入 `NULL`，MySQL 会“自作聪明”地将其转换为**当前系统时间**。
    
    - _后果：_ 破坏了业务逻辑。例如，你可能想用 `NULL` 表示“任务尚未完成”，结果数据库存进去了“现在的时间”。
        
- **跨数据库迁移风险：** 这种行为完全不符合 SQL 标准。如果你后期想将业务迁移到 PostgreSQL 或 Oracle，这些隐性逻辑会导致大量的代码重构和数据不一致。
    

### 解决方案：

目前推荐在配置文件（`my.cnf`）中开启：

SQL

```
explicit_defaults_for_timestamp = ON
```

开启后，`TIMESTAMP` 将表现得像 `DATETIME` 一样：支持显式的 `NULL`，且不会自动添加 `DEFAULT` 规则。