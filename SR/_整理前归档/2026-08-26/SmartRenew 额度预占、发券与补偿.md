## 1. 一句话概括

> Redis 负责高并发快速预占，MySQL 通过条件更新和乐观锁做最终扣减；额度、券、日志和申请状态在同一个本地事务中提交，失败时回滚数据库并补偿 Redis。

## 2. 主流程

```text
申请审核通过：APPROVED
↓
Redis Lua 原子预占额度
↓
开启独立 MySQL 事务
↓
条件扣减 quota_pool
↓
创建 Voucher
↓
记录额度变更日志和发券日志
↓
申请改为 ISSUED
↓
COMMIT
↓
删除 Redis 预占令牌并刷新余额
```

核心原则：

- Redis 是预占和快速拦截层。
- MySQL 是最终事实来源。
- Redis 异常时可以降级到 MySQL 确认。
- Redis 与 MySQL 之间不是分布式强事务，而是预占、补偿和刷新实现最终一致性。

## 3. Redis 预占

```text
余额Key：smartrenew:quota:remaining:{quotaPoolId}
令牌Key：smartrenew:quota:reserve:{reserveToken}
```

- 金额转成整数“分”，避免浮点误差。
- Lua 将“读取、判断、扣减”作为原子操作。
- 余额不足直接返回失败。
- Redis 不可用时跳过预占，继续由 MySQL 最终确认。

## 4. MySQL 条件扣减

```sql
UPDATE quota_pool
SET available_amount = available_amount - #{amount},
    used_amount = used_amount + #{amount},
    version = version + 1
WHERE id = #{id}
  AND version = #{version}
  AND status = 'ACTIVE'
  AND available_amount >= #{amount};
```

两层保证：

- `available_amount >= amount`：防止余额扣成负数。
- `version = version`：乐观锁防止并发覆盖，冲突后有限重试。

## 5. 防止重复发券

```text
申请记录 FOR UPDATE 行锁
+
发券前按 applicationId 查询
+
UNIQUE(voucher.application_id)
+
UNIQUE(voucher.voucher_code)
```

- 同一申请只能生成一张券。
- 随机券码冲突时重新生成，数据库唯一约束最终兜底。

## 6. 本地事务边界

同一个 MySQL 事务包含：

```text
额度池扣减
+ Voucher 插入
+ quota_change_log
+ voucher_issue_log
+ Application → ISSUED
```

任一步失败：

```text
全部 ROLLBACK
```

额度流程使用 `TransactionTemplate + REQUIRES_NEW`，与审核事务分离：

```text
审核事务：保存 APPROVED
↓ COMMIT
额度事务：扣减并发券
↓ 成功 ISSUED / 失败 QUOTA_FAILED
```

## 7. 三个故障窗口

### 窗口一：Redis 不可用

```text
Redis 超时或宕机
↓
降级到 MySQL 条件扣减
```

影响性能，但 MySQL 仍能防止超扣。

### 窗口二：Redis 预占成功，MySQL 失败

```text
MySQL事务 ROLLBACK
↓
Redis INCRBY 加回额度
↓
删除 reserveToken
↓
新事务记录 QUOTA_FAILED
```

如果 Redis 补偿失败，发券日志记录 `COMPENSATION_PENDING`，为告警和人工处理提供依据。

### 窗口三：MySQL 已提交，Redis 清理前宕机

```text
MySQL结果正确
但Redis余额可能偏低、Token可能残留
```

现有防线：Token TTL、正常路径清理和刷新、重复处理时查询已有券。

待完善：定时对账、补偿重试、以 MySQL 余额修复 Redis。

## 8. 当前方案的两个边界

1. Redis `DECRBY` 和写入预占 Token 不是同一个原子操作；进程在两步之间宕机可能留下无 Token 的预占。
2. Redis 补偿的 `INCRBY + DELETE` 不是原子、幂等操作，重复补偿可能多加额度。

改进方向：使用 Lua 同时完成预占与 Token 写入，并使用 Lua 判断 Token 存在后再原子补偿。

## 9. V5 数据库迁移

V1 已有 `quota_pool` 和 `voucher`，V5 主要增加：

| 内容 | 作用 |
|---|---|
| `quota_change_log` | 记录额度变化前后金额和业务原因 |
| `voucher_issue_log` | 记录发券成功、失败、预占令牌和补偿状态 |
| `UNIQUE(biz_key, change_type)` | 防止同一业务重复记录同类额度变化 |
| 组合索引 | 优化额度池和用户券查询 |

V5 的核心价值：**审计、排障、补偿依据和查询性能。**

## 10. 高频追问

### 为什么同时使用 Redis 和 MySQL？

> Redis 用于快速预占和削峰，MySQL 用于最终确认和持久化。Redis 可以降级，MySQL 是最终事实来源。

### 如何防止超发？

```text
Redis Lua 原子预占
+ MySQL余额条件
+ version乐观锁
+ 申请行锁
+ Voucher唯一约束
```

### 发券中途失败怎么办？

> 数据库内的额度扣减、券、日志和申请状态一起回滚；Redis 已预占时执行补偿，再用新事务记录失败状态。

### 这是强一致吗？

> MySQL 内部是本地事务原子性；Redis 与 MySQL 之间采用补偿机制实现最终一致性，不是分布式强事务。

## 11. 90 秒项目回答

> SmartRenew 在申请审核通过后进入独立额度事务。系统先通过 Redis Lua 脚本原子预占额度，金额使用整数分保存；如果 Redis 不可用，则降级到 MySQL 最终确认。MySQL 使用带余额条件和 version 的 UPDATE 完成额度扣减，既防止余额为负，也通过乐观锁处理并发冲突。
>
> 扣减成功后，系统在同一个本地事务中创建 Voucher、记录额度变更日志和发券日志，并把申请改为 ISSUED。任一步失败都会整体回滚。系统通过申请行锁、按申请查询已有券以及 application_id 唯一约束防止重复发券。
>
> 如果 Redis 已预占但数据库事务失败，系统会补偿 Redis，并用新事务记录 QUOTA_FAILED。因此 MySQL 是最终事实来源，Redis 和 MySQL 之间通过预占、补偿和缓存刷新实现最终一致性。当前还可以继续完善补偿重试、定时对账以及 Redis 预占和 Token 写入的原子性。

## 12. 最终记忆链

```text
Redis预占
→ MySQL条件扣减
→ 同事务发券
→ 唯一约束防重
→ 失败整体回滚
→ Redis补偿
→ 日志记录与最终一致性
```
