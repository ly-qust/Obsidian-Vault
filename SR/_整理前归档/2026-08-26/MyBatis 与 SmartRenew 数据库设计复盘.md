
> 面试前复习顺序：先看“高频纠错”，再看六项产出，最后闭卷回答 90 秒项目题。  
> 证据范围：`D:\project\SmartRenew-Platform` 当前真实代码与 Flyway V1–V11。
---

# 一、今天最需要纠正的表达

| 容易说错 | 正确表达 |
|---|---|
| Mapper 负责业务规则 | Mapper 只负责数据访问；业务规则属于 Service |
| `@Param` 负责查询结果映射 | `@Param + #{}` 负责 Java 参数进入 SQL；`resultMap` 负责结果进入 Java 对象 |
| `where` 只是“筛选” | 有条件时添加 `WHERE`，去掉开头 `AND/OR`，无条件时不生成 `WHERE` |
| 所有筛选条件为空就返回空列表 | 通常表示不加筛选，仍按分页和排序查询；是否允许应由业务规则决定 |
| `#{}` 和 `${}` 都能安全接收输入 | `#{}` 是参数绑定；`${}` 是字符串替换，用户输入可能造成 SQL 注入 |
| `role` 限制“我的申请”数据 | 角色控制接口权限；`user_id` 控制当前用户的数据范围 |
| 无效状态自然会返回 400 | 原实现抛 `IllegalArgumentException` 并落入 500；今天已局部修复为 `APPLICATION_400` |
| 事务可以保证业务唯一 | 事务保证事务内原子性；业务唯一仍需唯一约束等并发防线 |
| 发券时 `FOR UPDATE` 锁券码 | 真实代码锁定的是 `application` 申请行 |
| 发券幂等键是 voucher_code | “同申请只发一券”由 `application_id` 唯一约束保证；券码唯一只防券码碰撞 |
| 核销只要幂等键唯一即可 | 不同幂等键仍可能竞争同一券，还需 `status + version` 条件更新和 `UNIQUE(voucher_id)` |
| Confirm ACK 直接更新 Outbox | RabbitMQ 返回 Confirm；应用发布器根据 Confirm/Return 更新 Outbox |
| Consumer ACK 等人工审核完成后发送 | 消费者完成本次消息处理、记录结果后 ACK；不等待人工审核完成 |
| 核心数据库“按 V1–V7 排版” | V1–V11 是演进记录；核心设计应按业务链解释 |

术语必须说准：

```text
review_task
quota_pool
voucher
write_off
Idempotency-Key
merchant order_no
```

---

# 产出 1：MyBatis 一页复习卡

## 1. 调用链

```text
Controller
→ Service
→ Mapper 接口
→ MyBatis / MyBatis-Plus
→ SQL
→ MySQL
→ Java 对象
```

## 2. Mapper

```text
Service → 业务逻辑、权限、状态流转、事务编排
Mapper  → 查询、插入、更新、删除
MySQL   → 数据事实、索引、约束、事务
```

XML 映射主干：

```text
namespace  → Mapper 接口完整类名
id         → Mapper 方法名
参数        → @Param 名称 / 参数对象属性
resultType → 简单结果类型
resultMap  → 显式列与属性映射
```

Mapper 方法名和 XML `id` 不一致时，MyBatis 通常在运行时抛出 `BindingException: Invalid bound statement`。

## 3. 参数与结果映射

```text
@Param + #{}
→ Java 参数进入 SQL
→ PreparedStatement 参数绑定
→ 普通查询值优先使用

resultMap
→ 数据库列映射到 Java 属性
→ 适合名称不同或复杂映射
```

```xml
<result column="user_id" property="userId"/>
```

`${}` 是原样字符串替换。表名、排序字段等确实无法使用普通参数占位时，必须来自白名单，不能直接信任用户输入。

## 4. 动态 SQL

```text
if      → 条件成立才加入 SQL 片段
where   → 自动处理 WHERE 和开头 AND/OR
foreach → 遍历集合，常用于 IN (?, ?, ?)
```

```xml
<where>
    <if test="status != null and status != ''">
        AND status = #{status}
    </if>
</where>
```

```xml
<if test="statuses != null and !statuses.isEmpty()">
    AND status IN
    <foreach collection="statuses" item="status"
             open="(" separator="," close=")">
        #{status}
    </foreach>
</if>
```

空集合必须明确处理，避免非法 `IN ()` 或错误的“无筛选”语义。

## 5. 查询排错链

```text
HTTP 参数
→ DTO 是否接收正确
→ Service 是否校验或修改
→ Mapper/@Param/Wrapper 条件开关
→ 最终 SQL 与绑定值
→ 数据库中是否有匹配数据
→ resultType/resultMap
→ VO 转换与响应
```

MyBatis-Plus 额外检查：

```java
.eq(condition, column, value)
```

`condition == false` 时整条条件不会进入 SQL。

---

# 产出 2：SmartRenew 真实动态查询案例

## 需求

管理员分页查询申请，筛选条件 `status`、`campaignId` 均可为空；用户端查询还必须追加当前 `userId`，防止越权读取其他用户申请。

真实入口：

```text
GET /api/v1/admin/applications
GET /api/v1/applications
```

真实 DTO：

```java
long pageNo = 1;
long pageSize = 10;
String status;
Long campaignId;
```

## Mapper 与动态条件

`ApplicationMapper` 继承 `BaseMapper<Application>`。该列表没有自定义 XML，而由 MyBatis-Plus `lambdaQuery()` 生成 SQL。

```java
return lambdaQuery()
        .eq(userId != null,
                Application::getUserId,
                userId)
        .eq(StringUtils.hasText(request.getStatus()),
                Application::getStatus,
                request.getStatus())
        .eq(request.getCampaignId() != null,
                Application::getCampaignId,
                request.getCampaignId())
        .orderByDesc(Application::getCreateTime)
        .page(page);
```

等价理解：

```text
eq(true,  column, value)  → 加入 column = ?
eq(false, column, value)  → 忽略整个条件
```

## 两个边界测试

### 测试 1：管理员所有筛选条件为空

预期：

```sql
SELECT ...
FROM application
ORDER BY create_time DESC
LIMIT ?, ?
```

- 不出现非法 `WHERE AND`。
- 不返回“固定空列表”，而是查询第一页。
- 单元测试验证三个可选 `.eq` 开关均为 `false`。

### 测试 2：不存在的状态 `NOT_EXISTS`

原行为：`ApplicationStatus.fromCode()` 抛 `IllegalArgumentException`，全局处理为 500。  
修复后：查询入口局部转换为 `BizException("APPLICATION_400", ...)`，HTTP 语义为 400。

测试命令：

```text
mvn -q -Dtest=ApplicationServiceImplTest test
```

结果：退出码 0，目标测试类通过。

## AI Review 清单

- DTO 名称与 Wrapper 参数是否对应？
- Java 属性与数据库列是否对应？
- 条件开关是否可能错误为 `false`？
- 空条件是“查全部分页”还是“拒绝查询”？
- 空集合是否生成非法 `IN ()`？
- 是否使用 `#{}` 或安全的参数绑定？
- 结果映射、分页和排序是否正确？
- 非法枚举值返回 400 还是错误的 500？

---

# 产出 3：数据库演进图（V1–V11）

```text
V1  → 建立用户角色、活动规则、申请审核、额度券、核销、Inbox 等完整初始骨架
↓
V2  → 增加活动可见性、当前规则版本，以及地区/价格/比例/上限/材料规则
↓
V3  → 增加申请规则计算快照、驳回原因和材料文件元数据
↓
V4  → 新建 mq_outbox，补齐可靠消息发送端
↓
V5  → 增加额度变更日志和发券日志，增强审计与业务幂等
↓
V6  → 将历史中文分类统一为 HOME_APPLIANCE 编码
↓
V7  → 增加商户用户关系、核销订单号、幂等键及相应唯一约束
↓
V8  → 幂等初始化 USER 角色
↓
V9  → 额度池支持软删除，并限制同活动同分类只能有一个 ACTIVE 额度池
↓
V10 → 申请材料支持 FILE / TEXT 两种提交方式
↓
V11 → AI 客服日志增加会话、意图、引用和降级标记
```

Flyway 面试表达：

> Flyway 将数据库变化作为有版本的脚本纳入代码管理，记录已执行版本，并让开发、测试和生产按相同顺序演进。版本迁移不是自动回滚机制；生产回退需要补偿迁移或备份恢复方案。

---

# 产出 4：SmartRenew 核心 ER 关系图

```text
sys_user
   │
   └──────────────┐
                  ↓
campaign ───→ application ───→ review_task
   │              │  │              │
   │              │  └──→ mq_outbox │
   │              │          │       │
   │              │       RabbitMQ   │
   │              │          │       │
   │              │      mq_inbox ───┘
   │              │
   │              └────────────→ voucher ───→ write_off
   │                                  ↑
   ├──→ rule_version                  │
   │                                  │
   └──→ quota_pool ───────────────────┘

merchant ───→ write_off
```

职责：

```text
campaign / rule_version → 活动和可版本化的补贴规则
application             → 用户申请业务单据与规则计算快照
review_task             → 审核分派、状态、处理人和版本等工作流事实
quota_pool              → 活动分类额度及并发版本
voucher                 → 券当前状态和金额
write_off               → 核销交易、商户订单和审计证据
mq_outbox               → 待发布事件、重试和发送状态
mq_inbox                → 消费结果与消息去重记录
```

`application` 不能由 `review_task` 替代：前者是业务单据，后者是围绕单据产生的审核工作项。  
`write_off` 不能只用 `voucher.status` 替代：状态只说明当前结果，核销表保留商户、订单、幂等键、金额、时间和完整审计证据。

---

# 产出 5：数据库多层防线表

| 业务约束 | 应用层 | 事务/并发控制 | 数据库兜底 |
|---|---|---|---|
| 同活动同用户只申请一次 | 先查已有申请，提前友好返回 | 创建申请处于事务中；并发请求仍可能同时通过预查 | `UNIQUE(campaign_id, user_id)`；捕获唯一冲突 |
| 同申请只发一张券 | 查询 `voucher.application_id`，存在则返回已有券 | 新事务中 `SELECT application ... FOR UPDATE`；额度以 `version` 条件更新并有限重试 | `UNIQUE(application_id)`；`voucher_code` 另有唯一约束防随机碰撞 |
| 同券只核销一次 | 查商户幂等键、订单号、已有核销记录和券状态 | `@Transactional`；`UPDATE voucher ... WHERE status='ISSUED' AND version=?`，只有一个请求成功 | `UNIQUE(voucher_id)`、`UNIQUE(merchant_id, order_no)`、`UNIQUE(merchant_id, idempotency_key)` |

可靠消息真实边界：

```text
申请状态 + Outbox
→ 同一业务事务提交

定时发布器
→ 扫描 PENDING/可重试 Outbox
→ RabbitTemplate 发布
→ Confirm ACK 且无 Return 后标记 SENT

消费者
→ 先查 Inbox 是否已成功消费
→ 调用带事务的审核流水线
→ 根据申请状态和 review_task.application_id 唯一约束保持幂等
→ 记录 Inbox SUCCESS
→ Consumer ACK
```

注意：当前代码中审核流水线事务与之后的 Inbox SUCCESS 记录不是同一个数据库事务。两者之间故障可能导致重投，因此仍依赖申请状态检查、任务唯一约束和重复消息处理逻辑兜底；面试时不要编造为“一个事务同时提交 Inbox 和 ReviewTask”。

多层防线：

```text
请求层参数校验
→ Service 业务规则
→ 事务、行锁或版本条件更新
→ 数据库唯一约束最终兜底
```

---

# 产出 6：90 秒项目回答

> SmartRenew 的数据库使用 Flyway V1 到 V11 做版本化演进，但核心设计是按照业务链划分的。活动和规则由 `campaign`、`rule_version` 保存；用户申请进入 `application`，提交后进入 `review_task` 审核流程；审核通过后从 `quota_pool` 扣减额度并生成 `voucher`，商户核销时写入 `write_off`，既更新券状态，也保留订单和审计证据。
>
> 可靠消息方面，申请状态更新和 `mq_outbox` 在同一个业务事务中提交。应用中的定时发布器扫描 Outbox，并通过 `RabbitTemplate` 发送 RabbitMQ；消费者先检查 `mq_inbox`，再执行带事务的审核流水线，依靠申请状态和审核任务唯一约束处理重复消息，成功后记录 Inbox 并 ACK。
>
> 数据正确性不能只依赖业务代码的 `if`。同活动同用户申请由 `UNIQUE(campaign_id, user_id)` 兜底；发券时通过 `FOR UPDATE` 锁定申请行，并用 `UNIQUE(application_id)` 保证同申请只有一张券，额度扣减使用版本条件更新防止并发超扣；核销使用幂等键和商户订单号防重，通过 `status + version` 把券从 `ISSUED` 更新为 `USED`，最后由 `UNIQUE(voucher_id)` 阻止同券生成两条核销记录。整体形成业务校验、事务与并发控制、数据库约束三层防线。

---

# 面试前闭卷题

1. 为什么所有查询条件为空时不一定返回空列表？
2. `@Param + #{}` 与 `resultMap` 的方向有什么区别？
3. 为什么“先查再插”不能保证同活动同用户只申请一次？
4. 为什么事务不能代替唯一约束？
5. 发券时 `FOR UPDATE` 到底锁哪一行？
6. `UNIQUE(application_id)` 与 `UNIQUE(voucher_code)` 分别防什么？
7. 两个不同 Idempotency-Key 同时核销一张券，哪两层还能阻止重复？
8. Outbox、Publisher Confirm、Inbox、Consumer ACK 各负责哪一段？
9. 为什么 `write_off` 不能只用 `voucher.status` 替代？
10. V9 的 `active_unique_flag` 为什么可以只限制 ACTIVE 额度池？
