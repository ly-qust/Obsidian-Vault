---
tags: [RabbitMQ, SmartRenew, 故障注入, Outbox, Inbox, 实验记录, 面试]
date: 2026-08-27
---

# SmartRenew RabbitMQ 故障实验

[[02-重试、死信与幂等|上一篇：重试与幂等]] · [[00-RabbitMQ面试知识地图|知识地图]]

> [!warning] 实验状态
> 本页是“可执行实验方案与记录模板”。未实际运行前，只能写“设计上存在/待验证”，不能写成“已经验证”。

## 1. 四个故障窗口

| 故障窗口 | 现象 | 当前防线 | 仍有边界 |
|---|---|---|---|
| Broker 未接收 | Producer 连接失败、Confirm 异常 | Outbox 先落库；发送异常进入重试；定时扫描 | 超过次数变 `FAILED` 后当前扫描不再自动捞回 |
| Exchange 无法路由 | Broker 收到，但没有 Queue 匹配 | `mandatory=true` + Return；只有 Confirm ACK 且无 Return 才 `SENT` | Routing Key 一直错误时重试也无效；需修配置 |
| 消费事务失败 | Queue 有消息，业务抛异常 | Retry Exchange/Queue，TTL 约 15 秒；超限 DLQ | 永久 Bug、坏数据仍需人工/定时补偿；业务必须幂等 |
| 业务提交后 ACK 前宕机 | DB 成功，RabbitMQ 未收到 ACK | 可能重投；Inbox + 状态机 + 唯一约束降低重复副作用 | Inbox、业务事务和 ACK 不跨系统原子 |

## 2. 推荐实验：停止 RabbitMQ

选择理由：不改业务代码、恢复简单、最容易同时观察 Outbox、日志和最终消费状态。容器名以 `docker ps` 实际输出为准；材料中使用过 `smartrenew-rabbitmq-dev`，不要盲目假设环境一定相同。

### 2.1 实验目标

验证以下链路：

```text
RabbitMQ 不可用
→ application + Outbox 仍成功落库
→ Producer 发送失败并记录重试
→ RabbitMQ 恢复
→ Outbox 定时重发
→ SENT
→ Consumer 消费
→ Inbox / 业务最终完成
```

本次实验不声称验证：Return 错误路由、消费重试队列、DLQ、重复消息幂等、ACK 前宕机重投；这些需要单独注入故障。

### 2.2 第 0 步：确认正常状态

```bash
docker ps
```

确认后端、MySQL、Redis、RabbitMQ 正常运行，并记下待测试环境的 RabbitMQ 容器名、申请接口和数据库连接信息。

### 2.3 第 1 步：停止 RabbitMQ

```bash
docker stop <实际RabbitMQ容器名>
```

不要停止后端和 MySQL。目标状态：

```text
Spring Boot ✅
MySQL ✅
Redis ✅
RabbitMQ ❌
```

### 2.4 第 2 步：提交一条新申请

通过真实前端或接口提交一条申请，记录 `applicationId`。预期：

```text
Application → SUBMITTED ✅
Outbox → PENDING ✅
afterCommit dispatch
RabbitMQ 发送失败 ❌
```

### 2.5 第 3 步：保存证据

#### 后端日志

关注：

```text
Failed to publish application review event
outboxId=...
```

记录时间、`outboxId`、`applicationId`、异常根因和是否出现重试日志。

#### Outbox 状态

```sql
SELECT
    id,
    event_key,
    biz_key,
    status,
    retry_count,
    next_retry_time,
    last_error,
    sent_time
FROM mq_outbox
ORDER BY id DESC
LIMIT 5;
```

重点不是 `retry_count` 必须恰好为 1，而是观察：

```text
没有 SENT
有错误记录
有下一次重试时间
```

#### 业务表与 Inbox

```sql
SELECT id, status
FROM application
WHERE id = <applicationId>;

SELECT message_key, consumer_group, status, consumed_time
FROM mq_inbox
ORDER BY id DESC
LIMIT 5;
```

申请应仍然存在；在消息尚未成功消费时，不应把这条新事件误写成 Inbox SUCCESS。

### 2.6 第 4 步：恢复 RabbitMQ

拍完日志和表状态后立即恢复：

```bash
docker start <实际RabbitMQ容器名>
```

当前 Outbox 扫描间隔约 15 秒，租约约 30 秒；等待时间取决于 `next_retry_time`、扫描时机和应用连接恢复，不要把“固定 15 秒必定恢复”写成保证。

### 2.7 第 5 步：验证自动恢复

日志应关注成功发布记录：

```text
Published application review event successfully.
outboxId=...
applicationId=...
```

再次查询：

```sql
SELECT
    id,
    event_key,
    biz_key,
    status,
    retry_count,
    last_error,
    sent_time
FROM mq_outbox
WHERE id = <outboxId>;
```

预期：

```text
status = SENT
sent_time 有值
```

再查 Inbox 和申请状态：

```sql
SELECT message_key, consumer_group, status, consumed_time
FROM mq_inbox
WHERE biz_key = '<applicationId>'
ORDER BY id DESC;

SELECT id, status, current_review_task_id, complete_time
FROM application
WHERE id = <applicationId>;
```

业务最终状态由规则决定，可能进入 `PENDING_REVIEW`、`APPROVED` 或 `REJECTED`，不要预先写死某一个结果。

## 3. 为什么 Queue 可能看不到积压

RabbitMQ 管理后台出现 `Ready = 0` 不代表实验失败：

```text
消息进入 Queue
→ Consumer 立即拿走
→ 业务成功
→ ACK
→ Ready 又回到 0
```

证据优先级：

```text
1. 后端失败/成功日志
2. Outbox PENDING/SENDING → SENT
3. Inbox SUCCESS
4. Application / ReviewTask 状态
5. Queue Ready/Unacked 作为辅助现象
```

## 4. 实验记录模板

```text
实验日期：
环境：后端版本 / RabbitMQ 版本 / 容器名
注入故障：
applicationId：
outboxId：

故障前：
- application 状态：
- Outbox 状态：
- Inbox 状态：

故障中：
- 后端日志：
- Outbox status/retry_count/next_retry_time/last_error：
- Queue 现象：

恢复后：
- RabbitMQ 恢复时间：
- Outbox 是否 SENT：
- Inbox 是否 SUCCESS：
- Application / ReviewTask 最终状态：

本次真实验证：
尚未验证：
下一条可执行命令：
```

## 5. 其他故障实验（不要混写结论）

### 错误 Routing Key

只有在不改变 Producer 和 Binding 共同读取的同一个配置项时才有意义。若两者一起改成相同错误值，仍可能精确匹配，不能注入“无法路由”故障。应在隔离环境让 Producer 使用错误 Key，而 Binding 保持原值，然后观察 Confirm 与 Return。

### 消费者主动抛异常

观察：

```text
主 Queue → Retry Exchange/Queue → TTL → 主 Exchange/Queue
```

验证 `x-retry-count`、15 秒延迟、达到上限后的 Dead Queue，并记录这次才是真正验证了消费重试/DLQ。

### ACK 前宕机

要把业务提交和 ACK 之间制造可控暂停，确认 RabbitMQ 重新投递同一消息，再查看 Inbox、业务状态和是否重复产生副作用。该实验风险较高，必须使用测试数据。

## 6. 90 秒故障实验汇报

> 我注入的是 Broker 不可用故障。提交申请后，application 和 mq_outbox 在同一事务中仍然落库，申请状态为 SUBMITTED，Outbox 保留 PENDING/SENDING，并记录重试时间和错误；RabbitMQ 恢复后，定时任务重新 dispatch，Outbox 最终变为 SENT，Consumer 再完成审核并写入 Inbox。这个实验真实验证了 Outbox 对生产端短暂故障的恢复能力，没有验证 Return 错误路由、消费 Retry/DLQ 和 ACK 前宕机重投，因此这些只能表述为代码已有设计或待验证边界。

## 7. 高频自测

1. 停 RabbitMQ 时为什么申请数据不能一起丢？
2. Outbox 处于 FAILED 后，恢复 Broker 为什么不一定自动恢复？
3. 为什么 Queue Ready=0 不能单独证明消息链路没有问题？
4. 本次实验真实验证了什么，明确没有验证什么？
5. 错误 Routing Key 实验为什么可能误判？
6. 如何区分 Broker 重投和应用层 Retry Queue 转发？

