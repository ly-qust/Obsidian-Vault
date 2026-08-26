
## 1.生产端


SmartRenew 生产者
      ↓
   Exchange
      ↓
    Queue
      ↓
   Consumer


完整路线：

```
Outbox
  ↓
生产者发送
  ↓
Exchange
  │
  ├─ 没到：Confirm NACK
  │
  └─ 到了：Confirm ACK
             ↓
       能否路由到 Queue？
       │
       ├─ 不能：Return
       └─ 能：进入 Queue
                  ↓
               Consumer
                  ↓
             处理业务 + Inbox
                  ↓
                 ACK
```
```
应用程序查询 PENDING Outbox
    ↓
调用 RabbitTemplate
    ↓
RabbitMQ处理发布请求
    ├─ Broker处理失败：Confirm NACK
    └─ Broker处理成功
          ↓
       Exchange路由
          ├─ 无法路由：Return（mandatory=true）
          │             + 之后仍可能 Confirm ACK
          └─ 成功路由：Queue接受消息
                        + Confirm ACK
                              ↓
                          Consumer
                              ↓
                    业务处理 + Inbox去重
                              ↓
                        Consumer ACK
```
```
Confirm：检查消息有没有到 Exchange。

Return：消息到了 Exchange，
但找不到可以路由的 Queue。

Consumer ACK：消费者已经处理完成，
RabbitMQ 可以删除 Queue 中的消息。
```

## 2.消费端
```
Consumer 收到消息
↓
业务执行成功
↓
Inbox 记录 SUCCESS
↓
还没 ACK 就宕机
```
Inbox 记录已处理的消息，使重复投递时可以跳过业务，实现消费幂等。


| 宕机位置              | 结果                         |
| ----------------- | -------------------------- |
| 收到消息，业务还没执行       | 没有 ACK，RabbitMQ 重新投递       |
| 业务执行失败            | 业务事务回滚，消息进入重试队列            |
| 业务成功，Inbox 尚未记录   | 可能重新投递；当前项目还依赖申请状态判断避免重复处理 |
| Inbox 已记录，ACK 前宕机 | 重新投递后 Inbox 判重，跳过业务并 ACK   |
