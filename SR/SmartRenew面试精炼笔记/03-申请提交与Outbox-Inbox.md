---
tags: [SmartRenew, 事务, Outbox, Inbox, RabbitMQ, 幂等]
---

# 申请提交与 Outbox / Inbox

[[02-Security与JWT请求链|上一篇：Security]] · [[00-SmartRenew项目知识地图|知识地图]] · [[04-额度预占与可靠发券|下一篇：额度与发券]]

## 1. 申请提交事务

```text
@Transactional
→ SELECT ... FOR UPDATE锁定申请
→ 校验申请人、状态、材料
→ application: DRAFT → SUBMITTED
→ 插入PENDING Outbox
→ 一起提交或一起回滚
```

`FOR UPDATE` 解决的是两个请求同时基于旧状态做决策；事务保证申请状态和 Outbox 原子提交。两者作用不同，不能只说“加了事务就不会并发重复提交”。

## 2. 为什么不直接在事务里发 MQ

```text
先更新DB，再发MQ：DB成功但MQ失败 → 下游永远不知道
先发MQ，再更新DB：MQ成功但DB回滚 → 下游处理不存在的业务事实
```

Outbox 把“需要发送的事件”当作业务数据，在同一个数据库事务中持久化。提交后立即尝试发送，失败时由重试任务继续扫描 PENDING/FAILED 记录。

## 3. 消费端 Inbox

```text
收到消息
→ 尝试写入Inbox唯一键
→ 已存在：说明重复消息，幂等返回
→ 不存在：创建审核任务并更新Inbox状态
```

MQ 常见语义是至少一次投递，因此消费者必须按消息键或业务键幂等。Inbox 解决“消息重复消费”，业务唯一约束继续保证“一份申请只创建一份审核任务”。

## 4. Confirm、Return 与业务成功

- Confirm：Broker 是否接收发布；
- Return：消息是否无法路由；
- 消费 ACK：消费者是否确认处理；
- 业务成功：消费者事务内的业务表和 Inbox 是否成功提交。

四者不能混成一个“MQ 发送成功”。

## 5. 60 秒回答

> 用户提交申请时，我在本地事务中通过 `FOR UPDATE` 锁定申请，完成状态和材料校验，把申请改为 SUBMITTED，同时写入 PENDING Outbox；两者一起提交或回滚。事务提交后再发送 RabbitMQ，发送失败由 Outbox 重试，避免数据库与消息双写失配。消费端用 Inbox 唯一键和审核任务的业务唯一约束处理重复投递，因此 MQ 即使至少一次投递，也不会重复创建审核任务。

