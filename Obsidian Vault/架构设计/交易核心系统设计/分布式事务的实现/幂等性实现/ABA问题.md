## 一、什么是 ABA 问题

**定义：**

ABA 问题指的是：一个变量从 A 改成了 B，又被改回了 A，看起来值没变，但实际上状态已经变化过。

在并发情况下，这会导致**错误的逻辑判断**。  
比如一个线程以为值没变，于是继续执行，实际上它忽略了中间的变化。

---

### 💰 举个交易系统的例子：

假设有一个账户余额表：

|用户|余额|
|---|---|
|Alice|100|

两个线程同时操作这笔钱：

#### T1（转账线程）：

- 读取余额 = 100

- 判断余额足够（>50）

- 打算扣减 50


#### T2（系统对账线程）：

- 读取余额 = 100

- 暂时扣减 100（冻结或转出）

- 又因为事务失败，回滚余额到 100


这时，T1 再执行扣减 50，就会成功。  
但实际上这 100 元已经被系统冻结过一次再解冻。T1 没有感知到这段变化，这就出现了 **ABA 问题**。

---

## 🧠 二、为什么乐观锁可以解决 ABA 问题

### 乐观锁的核心思想：

不加数据库锁，而是在更新时检测“数据有没有被改过”。  
一般用 **版本号（version）** 或 **时间戳（update_time）** 来判断。

---

### ✅ 使用乐观锁改写上面的例子：

数据库表结构：

|用户|余额|version|
|---|---|---|
|Alice|100|1|

#### 操作流程：

1. **T1 读取**：余额=100，version=1
    
2. **T2 执行冻结再回滚：**
    
    - 冻结余额：update account set balance=0, version=version+1 where user='Alice' and version=1  
        → version 更新为 2
        
    - 回滚余额：update account set balance=100, version=version+1 where user='Alice' and version=2  
        → version 更新为 3
        
3. **T1 更新时：**
    
    - T1 尝试执行：update account set balance=50, version=version+1 where user='Alice' and version=1
        
    - 因为当前 version=3，不匹配 → **更新失败**
        

T1 发现更新失败后，会重新读取最新值再决定是否重试。  
这样就检测到了中间变化，避免了 ABA 问题。

---

## 💡 三、总结类比

|场景|问题|
|---|---|
|**ABA**|值“看似没变”，但中间经历过变化|
|**悲观锁**|直接加锁，别人不能动|
|**乐观锁**|不加锁，但用版本号或时间戳检测是否“动过”|

---

## 🧾 四、实际应用举例（SQL）

`-- 假设用户账户表 CREATE TABLE account (   id BIGINT PRIMARY KEY,   balance DECIMAL(10,2),   version INT );  -- 读取余额 SELECT balance, version FROM account WHERE id = 1;  -- 扣钱操作（使用乐观锁） UPDATE account SET balance = balance - 50, version = version + 1 WHERE id = 1 AND version = 1;`

如果上面的更新返回 `affected_rows = 0`，  
说明在你更新期间有人改过余额或版本号，  
你就要 **重新读取最新数据再重试**。

---

✅ **结论：**

在交易场景中，ABA 问题会导致逻辑判断错误（例如余额状态被误判），  
而使用 **乐观锁机制（版本号 / 时间戳校验）**，可以在提交更新时检测数据是否被改动，  
从而有效避免 ABA 问题。