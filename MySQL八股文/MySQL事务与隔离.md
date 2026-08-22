### 1. 知识地图

~~~text
事务 → ACID → 并发异常 → 隔离级别 → MVCC/一致性读
                         → 当前读/FOR UPDATE → 长事务
~~~

### 2. ACID

| 特性 | 面试理解 | 解决问题 | SmartRenew体现 |
|---|---|---|---|
| Atomicity 原子性 | 全部成功或全部失败 | 避免业务更新与事件写入不一致 | 申请更新与 Outbox 同事务 |
| Consistency 一致性 | 事务前后满足约束和业务不变量 | 防非法状态、唯一性和规则破坏 | 状态、材料、归属校验 |
| Isolation 隔离性 | 并发事务按规则互相隔离 | 减少异常读和更新竞争 | FOR UPDATE 锁申请行 |
| Durability 持久性 | COMMIT 后故障恢复仍保留 | 防已提交数据丢失 | 已提交 Outbox 可重试 |

一致性不是数据库自动修复业务错误，而是数据库约束和业务校验共同维护不变量。

### 3. 异常读

#### 脏读

~~~text
A: BEGIN; UPDATE balance=50;  -- 未提交
B: SELECT → 50
A: ROLLBACK;                  -- 最终仍是100
~~~

读到别人未提交、可能回滚的数据就是脏读。判断关键是对方是否 COMMIT。

#### 不可重复读

~~~text
A: BEGIN; SELECT → 100
B: UPDATE → 50; COMMIT
A: 再 SELECT → 50
~~~

同一事务、同一行、前后读取值不同。数据已提交，所以不是脏读。

#### 幻读

~~~text
A: 范围查询 → 2行
B: 插入满足条件的新行并 COMMIT
A: 相同范围查询 → 3行
~~~

原有行未必改变，变化的是范围结果集。

一句话区分：

~~~text
脏读：未提交的数据
不可重复读：同一行的值变了
幻读：同一条件的行集合变了
~~~

### 4. 隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 | 并发能力 | 重点 |
|---|---|---|---|---|---|
| RU | 允许 | 允许 | 允许 | 高但一致性弱 | 几乎无有效读隔离 |
| RC | 阻止 | 允许 | 允许 | 较高 | 普通读通常看新的已提交视图 |
| RR | 阻止 | 通常阻止 | InnoDB 普通一致性读通常避免，当前读需结合锁 | 中等 | MySQL/InnoDB 默认 |
| Serializable | 阻止 | 阻止 | 阻止 | 较低 | 强隔离，等待更多 |

隔离级别不是越高越好：一致性增强通常伴随锁等待、冲突、延迟和吞吐成本。RR 不能机械理解为所有场景绝对无幻读；普通 SELECT 是快照读，FOR UPDATE、UPDATE、DELETE 是当前读。

### 5. RC/RR理论实验卡

今天没有真实执行，以下是理论结果。

两会话先设置：

~~~sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
~~~

~~~sql
-- A
START TRANSACTION;
SELECT status FROM application WHERE id=1; -- SUBMITTED
-- B
START TRANSACTION;
UPDATE application SET status='NEED_MORE_INFO' WHERE id=1;
COMMIT;
-- A
SELECT status FROM application WHERE id=1; -- 预期 NEED_MORE_INFO
~~~

RC 中普通读通常得到新的已提交视图，因此出现不可重复读，不是脏读。

把两会话改为 REPEATABLE READ，保持顺序：A 两次普通 SELECT 理论上都看到第一次快照中的 SUBMITTED。若使用 FOR UPDATE，则变成当前读。

锁竞争实验：

~~~sql
-- A
START TRANSACTION;
SELECT * FROM application WHERE id=1 FOR UPDATE;
-- B 执行同样语句，理论上等待 A
-- A COMMIT 后，B 继续并读取当前版本
~~~

### 6. MVCC、读类型与 FOR UPDATE

- MVCC 通过多版本和事务视图让普通读减少读写阻塞。
- 普通 SELECT 通常是一致性读/快照读，不锁住后续业务判断。
- SELECT ... FOR UPDATE 是当前读，读取当前版本并尝试锁匹配记录。

它解决“读取、校验、修改”之间的竞态。锁的是数据库记录/相关索引资源，不是 Java 方法；锁通常持有到 COMMIT 或 ROLLBACK。

### 7. 长事务

~~~text
BEGIN → 加锁 → 远程调用/复杂计算 → UPDATE → COMMIT
      ↓
锁久、连接久、其他事务等待、并发下降
~~~

MQ/HTTP 不应随意放进数据库事务，网络等待会放大锁等待、连接占用和数据库压力。

### 8. 高频回答

- 脏读：读到未提交、可能回滚的数据；坑是把所有新值都叫脏读。
- 不可重复读与幻读：前者是同一行值变化，后者是范围结果集变化。
- RC/RR：RC 普通读通常看新的已提交视图，RR 普通读通常复用快照；坑是说 RR 绝对消灭幻读。
- FOR UPDATE：当前读并锁匹配记录，保护读改写；坑是说它锁 Java 方法或让查询更快。
- 长事务：锁、连接、等待时间变长，吞吐下降；坑是只说“慢”。

### 9. 我的薄弱点

| 薄弱点 | 正确抓手 |
|---|---|
| 初次认为读到未提交值“没问题” | 先看写方是否提交；未提交被读就是脏读 |
| 认为最终值正确就没有不可重复读 | 看同一事务同一行前后是否不同 |
| 混淆幻读与不可重复读 | 单行字段 vs 范围行集合 |
| 把 FOR UPDATE 说成幻读或锁住后续事务 | 锁数据库记录，解决同一行检查更新竞态 |
| 把 Outbox 说成保证不重复 | Outbox 保凭证；重试补偿；Inbox 幂等 |
| 不熟长事务原因 | 远程等待延长事务、锁和连接 |

### 10. 错误表达

- “读到新值就是脏读” → 关键是是否未提交。
- “不可重复读就是多一行” → 多一行是幻读。
- “RR 完全解决幻读” → 必须区分快照读和当前读。
- “FOR UPDATE 锁住后面的事务” → 锁住匹配记录，竞争事务等待。
- “RR 后不用 FOR UPDATE” → RR 不自动保护检查后更新。
- “Outbox 保证只发一次” → 可能重复投递，消费端要幂等。
- “MQ 失败可回滚已提交数据库” → 提交后靠 Outbox 重试。
- “隔离级别越高越好” → 是一致性与并发成本的权衡。

### 11. MySQL闭卷速记

~~~text
ACID：原子、一致、隔离、持久。
脏读=未提交；不可重复读=同一行值变；幻读=范围集合变。
RC=新已提交视图；RR=普通快照读通常稳定。
普通 SELECT=一致性读；FOR UPDATE=当前读+锁。
长事务=锁久、连接久、等待多、吞吐低。