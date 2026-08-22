### 1. 业务全链路

~~~text
用户 → Controller → submitForReview
→ @Transactional
→ selectByIdForUpdate
→ 归属/状态/材料校验
→ application 改 SUBMITTED
→ 写 MqOutbox(PENDING)
→ COMMIT
→ afterCommit
→ RabbitMQ
→ Consumer
→ Inbox去重
→ processSubmittedApplication
~~~

### 2. ApplicationStatus

真实状态：DRAFT、SUBMITTED、PENDING_REVIEW、NEED_MORE_INFO、APPROVED、REJECTED、ISSUED、QUOTA_FAILED。

~~~text
DRAFT/NEED_MORE_INFO → SUBMITTED
SUBMITTED → PENDING_REVIEW（有材料规则）
SUBMITTED → APPROVED（无材料规则）
SUBMITTED → REJECTED（规则快照缺失）
PENDING_REVIEW → APPROVED / REJECTED / NEED_MORE_INFO
APPROVED → ISSUED / QUOTA_FAILED
~~~

REJECTED、ISSUED、QUOTA_FAILED 是代码标记的最终状态。

### 3. submitForReview代码阅读地图

真实文件：[ApplicationServiceImpl.java](D:/project/SmartRenew-Platform/backend/src/main/java/com/smartrenew/platform/application/service/impl/ApplicationServiceImpl.java)。

1. @Transactional：业务更新与 Outbox 原子提交。
2. selectByIdForUpdate：先锁当前申请，再判断状态。
3. 校验存在和用户归属。
4. 只允许 DRAFT 或 NEED_MORE_INFO。
5. 按规则快照校验必需材料，异常触发回滚。
6. 更新申请为 SUBMITTED，设置提交时间，清理完成/拒绝信息。
7. recordSubmittedEvent：写 Outbox，不直接发 MQ。
8. 方法返回后 COMMIT。
9. Outbox 的 afterCommit 调用 dispatch 发送 MQ。

顺序原因：锁必须在校验前；业务和 Outbox 必须在提交前；MQ 必须在提交后。

### 4. selectByIdForUpdate

真实文件：[ApplicationMapper.java](D:/project/SmartRenew-Platform/backend/src/main/java/com/smartrenew/platform/application/mapper/ApplicationMapper.java)。

~~~sql
SELECT ... FROM application
WHERE id = #{applicationId}
FOR UPDATE
~~~

A 先拿锁、校验、更新、写 Outbox、提交；B 等待，随后读到当前 SUBMITTED，校验失败，不重复提交。锁的是数据库记录/索引资源，不是 Java 方法。

### 5. 事务边界

~~~text
BEGIN → 锁定 → 校验 → 更新申请 → 写 Outbox → COMMIT
~~~

中途失败：申请和 Outbox 一起回滚。Outbox 写成功但事务回滚：Outbox 也回滚。两者提交成功：MQ 暂时不可用时仍有待发送凭证。

### 6. Outbox、afterCommit与MQ

真实发布器：[ApplicationSubmittedEventPublisher.java](D:/project/SmartRenew-Platform/backend/src/main/java/com/smartrenew/platform/review/mq/ApplicationSubmittedEventPublisher.java)。

Outbox 记录 eventKey、eventType、bizKey、payload、状态和重试时间。业务数据和 Outbox 一起 COMMIT；afterCommit 后发送 RabbitMQ。ACK 且未 returned 才标记 SENT；失败回到 PENDING，记录错误和下一次重试时间，定时任务扫描。

Outbox 不等于 Exactly Once：发送成功但标记 SENT 前宕机，重试可能重复发送。

真实消费者：[ApplicationSubmittedEventConsumer.java](D:/project/SmartRenew-Platform/backend/src/main/java/com/smartrenew/platform/review/mq/ApplicationSubmittedEventConsumer.java)。

~~~text
eventId → alreadyConsumed
已消费：ACK跳过
未消费：执行业务 → recordSuccess → ACK
失败：retry queue；超限：failed + dead-letter
~~~

最终一致性来自：Outbox 可靠落库 + 发布重试 + 消费幂等。

### 7. 项目面试答辩

**为什么用 FOR UPDATE？**  
提交是读状态、校验、修改的复合操作。事务中当前读并锁申请行，后来的请求等待后读当前状态，避免重复提交和重复 Outbox。

**为什么设计 Outbox？**  
业务状态和消息凭证同事务提交，保证业务成功时有可重试事件；它不保证只发送一次。

**为什么不能事务内发 MQ？**  
远程调用可能超时，放进事务会延长事务、行锁和连接占用。先提交本地数据与 Outbox，再 afterCommit 发送。

**MQ 失败怎么办？**  
Outbox 保留待发送状态，记录重试次数和时间，定时扫描重试，超限进入失败/死信。

**两个请求同时提交？**  
A 获锁并提交；B 等待后读到 SUBMITTED，状态校验失败，不重复处理。

### 8. SmartRenew薄弱点与纠错

| 易错说法 | 正确说法 |
|---|---|
| FOR UPDATE 锁住后面的事务 | 锁匹配申请记录，竞争事务等待 |
| RR 足够保护提交流程 | 读改写仍需当前读和行锁 |
| Outbox 保证只发一次 | 可能重复，Inbox/eventId 做幂等 |
| afterCommit 让 DB 和 MQ 同事务 | 只保证 DB 提交后再发，二者仍不同 |
| MQ 失败回滚已提交业务 | 靠 Outbox 重试补偿 |
| 事务越大越安全 | 只把必须原子提交的本地 DB 操作放进去 |

### 9. SmartRenew闭卷速记

~~~text
DRAFT/NEED_MORE_INFO → SUBMITTED。
submitForReview：事务 → FOR UPDATE → 归属/状态/材料校验
→ 更新申请 → 写 Outbox(PENDING) → COMMIT → afterCommit → MQ。
FOR UPDATE 防同一申请并发提交；Outbox 保消息凭证，不保证只发一次。
发布失败重试；重复投递 Inbox/eventId 幂等；失败超限进死信。
~~~

## 闭卷口述训练

1. 什么是不可重复读？为什么 RR 和 RC 不同？
2. FOR UPDATE 解决什么问题？
3. SmartRenew 为什么需要 Outbox？
4. 为什么不能在数据库事务内直接调用 MQ？
5. 两个请求同时提交同一申请时，真实代码如何避免重复？

参考答案必须覆盖：同一行值变化、RC/RR 读视图、当前读与行锁、业务和 Outbox 同事务、afterCommit、重试、Inbox 幂等。