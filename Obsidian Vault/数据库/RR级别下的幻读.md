## 实验：RR 级别下的“幽灵记录”

准备一张简单的表 `users`，目前只有一条数据：`id=1`。

|**事务 A (Session A)**|**事务 B (Session B)**|
|---|---|
|`BEGIN;`|`BEGIN;`|
|`SELECT * FROM users WHERE id = 5;`<br><br>  <br><br>_(快照读：结果为空)_||
||`INSERT INTO users (id, name) VALUES (5, 'Gemini');`|
||`COMMIT;`|
|`SELECT * FROM users WHERE id = 5;`<br><br>  <br><br>_(快照读：依然为空，MVCC 起作用)_||
|**`UPDATE users SET name='Modified' WHERE id = 5;`**<br><br>  <br><br>_(当前读：触发神奇现象)_||
|`SELECT * FROM users WHERE id = 5;`<br><br>  <br><br>_(快照读：发现记录出现了！)_||
|`COMMIT;`||

---

### 为什么会这样？（原理解析）

1. **MVCC 的坚守**：在事务 A 的第二次 `SELECT` 时，由于 RR 级别的 ReadView 是第一次查询生成的，它认为 `id=5` 的记录不存在，这是正常的。
    
2. **当前读的越权**：当事务 A 执行 `UPDATE` 时，它必须操作数据的**最新版本**（当前读）。由于事务 B 已经提交，`UPDATE` 语句成功找到了 `id=5` 这行并修改了它。
    
3. **版本夺取**：重点来了！一旦 `UPDATE` 成功，这行记录的隐藏列 `DB_TRX_ID`（事务 ID）就变成了**事务 A 的 ID**。
    
4. **幻读发生**：事务 A 在最后一次 `SELECT` 时，发现这行数据的 `DB_TRX_ID` 等于自己的事务 ID。根据 MVCC 的可见性算法：**“自己改的数据，自己当然能看到”**。于是，这一行原本“不存在”的数据就这样蹦了出来。
    

---

## 最佳实践：如何写出健壮的事务代码？

为了避免上述尴尬和潜在的逻辑漏洞，建议遵循以下原则：

1. **一致性读取**：如果你对数据一致性要求极高，在开启事务后的第一次查询就使用 `SELECT ... FOR UPDATE`。这会直接加 **Next-Key Lock**，让事务 B 的 `INSERT` 根本无法执行，从而从物理层面杜绝幻读。
    
2. **事务短小**：事务开启后尽快提交，减少锁定的时间范围。
    
3. **索引优化**：间隙锁是基于索引的。如果你的查询条件没有索引，MySQL 可能会**锁住全表所有的间隙**，这会导致并发性能瞬间崩盘。