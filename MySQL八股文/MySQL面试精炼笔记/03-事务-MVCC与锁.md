---
tags: [MySQL, 事务, MVCC, 锁, 死锁, 面试]
---

# 事务、MVCC 与锁

[[02-索引与SQL优化|上一篇：索引]] · [[00-MySQL面试知识地图|知识地图]] · [[04-主从复制与高可用|下一篇：复制]]

## 1. ACID

![[Pasted image 20260121094225.png|900]]

| 特性 | 标准理解 | 主要机制 |
|---|---|---|
| Atomicity 原子性 | 全部成功或全部失败 | undo、事务状态 |
| Consistency 一致性 | 事务前后满足约束和业务不变量 | A/I/D、约束与业务校验共同保证 |
| Isolation 隔离性 | 并发事务按规则互相隔离 | MVCC、锁 |
| Durability 持久性 | COMMIT 后故障恢复仍保留 | redo、刷盘策略 |

一致性不是数据库自动修复业务错误；数据库约束和业务逻辑共同维护不变量。

## 2. 并发异常与隔离级别

```text
脏读：读到其他事务未提交的数据。
不可重复读：同一事务重复读取同一行，值发生变化。
幻读：相同范围条件前后得到的行集合变化。
丢失更新：一个事务的更新覆盖另一个事务的修改。
```

| 级别 | 核心特点 |
|---|---|
| RU | 可能脏读，并发高但隔离弱 |
| RC | 每次一致性读通常创建新 Read View，可见新的已提交数据 |
| RR | 第一次一致性读通常建立 Read View，后续复用；InnoDB 默认 |
| Serializable | 普通读也使用更强锁定，隔离强、并发成本高 |

## 3. MVCC

MVCC 不是复制完整快照，而是通过：

```text
隐藏事务ID DB_TRX_ID
+ 回滚指针 DB_ROLL_PTR
+ undo版本链
+ Read View可见性规则
```

从版本链中找到当前事务可见的数据版本。

```text
RC：每条普通一致性读通常建立新的 Read View。
RR：第一次普通一致性读通常建立 Read View，后续复用。
本事务自己的修改对自己可见。
```

## 4. 快照读与当前读

快照读：

```sql
SELECT * FROM account WHERE id = 1;
```

RC、RR 下普通 SELECT 通常是一致性非锁定读，通过 MVCC 读取，不加行锁。

当前读/锁定读：

```sql
SELECT * FROM account WHERE id = 1 FOR UPDATE;
SELECT * FROM account WHERE id = 1 FOR SHARE;
UPDATE account SET balance = balance - 100 WHERE id = 1;
DELETE FROM account WHERE id = 1;
```

`FOR UPDATE` 本身不修改数据，而是读取当前数据并加锁，保护“读取、校验、修改”的竞态。普通快照读通常仍能通过 MVCC 读取旧版本。

## 5. 锁类型

| 锁 | 作用 |
|---|---|
| S 共享锁 | 允许兼容读取，阻止冲突写入 |
| X 排他锁 | 阻止其他事务的冲突修改或锁定读 |
| IS / IX 意向锁 | 表明准备在表中某些记录上加 S/X 行锁，协调表锁与行锁 |
| Record Lock | 锁住具体索引记录 |
| Gap Lock | 锁住索引记录之间的间隙，主要阻止插入 |
| Next-Key Lock | 记录锁 + 该记录前方间隙锁 |
| MDL | 保护表结构，事务可能阻塞 ALTER/DROP 等 DDL |

InnoDB 的行锁主要加在索引上。锁范围取决于隔离级别、索引、条件和实际执行计划，不一定只锁最终返回的行。

RR 下常见规则：

| 查询 | 通常的锁 |
|---|---|
| 唯一索引等值命中 | 记录锁 |
| 唯一索引等值未命中 | 目标值所在的间隙锁 |
| 非唯一索引等值或范围扫描 | 临键锁、间隙锁 |

RC 下普通搜索和扫描通常关闭间隙锁，主要保留记录锁；外键检查和重复键检查等场景例外。

## 6. 锁等待与死锁

锁等待：B 等 A 释放冲突锁，没有等待环。超时常见：

```text
1205 Lock wait timeout exceeded
```

死锁：A 等 B、B 又等 A，形成环。InnoDB 通常检测后回滚一个事务：

```text
1213 Deadlock found
SQLSTATE 40001
```

排查命令：

```sql
SELECT * FROM performance_schema.data_lock_waits;
SELECT * FROM performance_schema.data_locks;
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS;
```

前三者主要看当前等待；`SHOW ENGINE INNODB STATUS` 重点看最近一次已检测死锁的 `LATEST DETECTED DEADLOCK`。

解决思路：

- 多个事务按相同顺序访问数据；
- 建立合适索引，缩小扫描和锁范围；
- 缩短事务，避免在事务里进行 HTTP/MQ 等远程调用；
- 捕获死锁后重试整个事务；
- 不要把增大等待超时当成根治方案。

## 7. 面试标准回答

> InnoDB 对普通 SELECT 通常使用 MVCC，通过 Read View 和 undo 版本链读取可见版本；对 FOR UPDATE、UPDATE、DELETE 等当前读使用锁。行锁主要加在索引上，RR 下唯一索引等值命中通常是记录锁，未命中会锁目标间隙，范围或非唯一索引扫描通常使用临键锁防止插入。冲突锁产生等待，等待关系形成环就是死锁，InnoDB 会回滚一个事务，应用应重试整个事务。

## 8. 易错点

- RR 不是所有场景都“不需要锁”；先查后改仍需并发控制。
- `FOR UPDATE` 不等于执行了 UPDATE。
- 间隙锁主要防止插入，不是锁住一个已有对象。
- 接口正在卡住先查当前锁等待；已检测到的死锁通常已回滚一方。
- 长事务不仅占连接和锁，还可能拖慢 undo 清理。

