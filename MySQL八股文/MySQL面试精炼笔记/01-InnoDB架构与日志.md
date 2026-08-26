---
tags: [MySQL, InnoDB, Buffer-Pool, redo, undo, binlog, 面试]
---

# InnoDB 架构与日志

[[00-MySQL面试知识地图|返回知识地图]] · [[02-索引与SQL优化|下一篇：索引与优化]]

## 1. MySQL 分层

```text
客户端
→ Server层：连接、权限、解析、优化、执行、binlog
→ 存储引擎层：InnoDB负责数据、索引、事务、锁、MVCC、redo/undo
```

## 2. InnoDB 核心结构

![[Pasted image 20260817143059.png|900]]

> [!note] 图的边界
> 这是概念示意图。undo 记录位于 undo 页中，页面可被 Buffer Pool 缓存并持久化到 undo 表空间，不能简单理解为一个纯内存模块。

| 结构 | 作用 |
|---|---|
| Buffer Pool | 缓存数据页、索引页和部分系统页，减少磁盘读取 |
| 脏页 | Buffer Pool 中已修改但尚未写回数据文件的页 |
| redo log buffer / redo | 先记录页修改，支持 WAL 和崩溃恢复 |
| undo | 保存修改前信息，用于回滚和 MVCC |
| Change Buffer | 缓存部分未在内存中的非唯一二级索引页修改，之后合并 |
| AHI | InnoDB 根据热点自动维护的内存哈希结构，不是用户创建的索引 |
| `.ibd` | 保存表的数据页和索引页 |
| binlog | Server 层归档日志，用于复制和时间点恢复 |

## 3. 一次 UPDATE 的主流程

```sql
UPDATE account SET balance = balance - 100 WHERE id = 1;
```

```text
通过索引定位记录并获得必要的锁
→ 生成 undo，保存旧值
→ 修改 Buffer Pool 中的数据页，形成脏页
→ 生成 redo，先进入 redo log buffer
→ Server 层生成 binlog event
→ 提交时协调 redo 与 binlog
→ 后台线程按策略异步刷脏页到数据文件
```

事务提交不等于数据页已经写入 `.ibd`。在相应持久化配置下，已持久化的 redo 能在崩溃后恢复数据页。

## 4. 三种日志

| 日志 | 所属层 | 核心作用 | 记忆 |
|---|---|---|---|
| undo | InnoDB | 回滚、MVCC 历史版本 | 如何回到过去 |
| redo | InnoDB | WAL、本机崩溃恢复 | 如何重做已完成修改 |
| binlog | Server | 复制、归档、时间点恢复 | 对外记录发生了什么 |

redo 不能替代 binlog：redo 面向 InnoDB 页恢复且空间循环使用；binlog 是 Server 层归档事件，可用于复制和恢复。

## 5. WAL 与两阶段提交

WAL（Write-Ahead Logging）：先保证日志可恢复，再异步写数据页。顺序追加 redo 通常比立即随机刷多个数据页成本低。

内部两阶段提交：

```text
redo prepare
→ 写入并按配置刷 binlog
→ redo commit
```

目的：保证 redo 与 binlog 对同一事务的状态一致，避免本机恢复结果与复制结果不一致。

崩溃时若 redo 处于 prepare：

- binlog 中存在对应事务：按已提交处理；
- binlog 中不存在：回滚该事务。

## 6. 崩溃恢复

```text
redo：恢复已提交但尚未刷回数据文件的页修改
undo：回滚崩溃时未提交的事务
binlog：参与 prepared 事务判断，主要服务复制与时间点恢复
```

常见强持久性配置：

```text
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

安全性更高，但同步刷盘开销也更大；还需区分“写入操作系统缓存”和“fsync 到稳定存储”。

## 7. 面试标准回答

> InnoDB 通过 Buffer Pool 和 WAL 兼顾性能与持久性。更新时先生成 undo，修改 Buffer Pool 中的数据页并形成脏页，同时生成 redo；Server 层还会生成 binlog。提交阶段通过 redo prepare、binlog、redo commit 的两阶段提交保证两份日志一致。事务提交不要求数据页立即写入 `.ibd`，后续可依据 redo 完成崩溃恢复，脏页由后台异步刷盘。

## 8. 易错点

- binlog 不是 InnoDB 的本机页级恢复日志。
- undo 不只用于回滚，也为 MVCC 提供历史版本。
- 提交成功不等于数据页已落盘。
- Change Buffer 主要面向非唯一二级索引页，不是任意写操作缓存。
- AHI 与覆盖索引没有直接关系。

