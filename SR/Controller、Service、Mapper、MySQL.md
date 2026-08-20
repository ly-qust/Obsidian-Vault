
## 面试回答

> 项目后端采用Controller、Service、Mapper的分层结构。Controller负责HTTP协议层工作，例如接收参数、获取当前用户和返回响应；Service负责核心业务规则、状态流转和事务编排；Mapper负责SQL查询和条件更新；MySQL负责数据持久化、事务和唯一约束。
> 
> 这样做可以把接口协议、业务逻辑和数据库访问分开，降低耦合，也方便单元测试和后续修改。例如申请提交的业务判断放在Service，申请状态和审核记录的SQL放在Mapper，最终由MySQL事务保证数据一致性。

## 每一层怎么解释

### Controller

负责：

```
接收HTTP请求
校验基础参数
获取当前登录用户
调用Service
转换返回结果
```

例如：

```
POST /applications/{id}/submit
```

Controller可以接收申请ID和提交参数，然后调用：

```
applicationService.submit(id, currentUser)
```

不应该在Controller里完成：

```
判断申请是否满足所有规则
修改多个业务表
扣减额度
发券
```

这些属于业务编排。

### Service

Service是最重要的一层。

负责：

```
业务规则判断
状态流转
权限和资源校验
事务边界
调用多个Mapper或其他模块服务
```

例如申请提交可能需要判断：

```
申请是否属于当前用户
申请当前是否为DRAFT
必需材料是否齐全
活动是否仍然有效
然后修改申请状态
生成审核任务
```

这些应该由Service统一编排。

### Mapper

负责和数据库交互：

```
查询申请
更新申请状态
条件扣减额度
判断更新影响行数
```

例如并发扣减额度时，不推荐：

```
先查询 available_amount
Java里判断够不够
再执行UPDATE
```

更可靠的是让数据库一次完成条件更新：

```
UPDATE quota_pool
SET available_amount = available_amount - #{amount}
WHERE id = #{id}
  AND available_amount >= #{amount}
```

然后根据影响行数判断是否扣减成功。

### MySQL

MySQL不只是“存数据”，还负责：

```
事务
唯一约束
外键或数据约束
条件更新
最终事实来源
```

比如重复发券不能只靠Java代码判断，还应该有：

```
application_id UNIQUE
```

这样即使代码重试、消息重复消费，数据库也能阻止重复数据。

## 面试官可能追问：为什么不把逻辑都写在Controller？

回答：

> Controller如果同时处理参数、权限、状态、事务和数据库操作，会变得难以测试和维护。以后同一个业务被定时任务、消息消费者或其他接口调用时，还会出现逻辑重复。因此Controller应该尽量薄，把业务规则集中到Service。

## 面试官可能追问：Service只是调用Mapper可以吗？

回答：

> 如果Service只有一行Mapper调用，说明它可能只是一个转发层。但简单查询可以这样做，复杂业务必须由Service负责状态、权限、事务和多个数据操作的编排。不能为了形式强行把所有逻辑都塞进Service，也不能让Controller承担业务。

## 面试官可能追问：事务应该放在哪里？

回答：

> 事务通常放在Service的业务方法上，因为Service最清楚一次完整业务操作包含哪些数据库修改。例如审核通过、额度扣减和发券需要根据业务边界决定是否放在同一个事务中。Controller层事务范围通常过大，不适合作为主要事务边界。

补充一句更严谨：

> 异步消息发送、Redis预占和MySQL事务之间不能简单认为天然属于同一个事务，需要通过Outbox、补偿或幂等设计保证最终一致性。

## 面试官可能追问：查询后再更新为什么有并发问题？

假设额度剩余10：

```
请求A查询到10
请求B也查询到10
A扣减8
B也扣减8
```

两个请求都认为额度够，最终可能扣成负数或超发。

所以应该让数据库执行带条件的更新：

```
只有 available_amount >= amount 时才允许更新
```

然后检查：

```
影响行数 = 1：成功
影响行数 = 0：额度不足或版本冲突
```

## 替代方案怎么说？

如果面试官问“有没有更高级的设计”，可以说：

> 对当前项目，Controller、Service、Mapper分层已经足够直观。如果领域规则变得复杂，可以进一步采用六边形架构，把Web请求和数据库都作为适配器，核心业务放到Domain层，通过Repository接口隔离基础设施。但这会增加抽象数量，当前项目不一定需要为了形式复杂化。

六边形架构可以理解成：

```
HTTP请求
   ↓
Web Adapter
   ↓
Application Service
   ↓
Domain业务模型
   ↓
Repository接口
   ↓
MySQL Adapter
```

## 这部分的常见坑

你可以主动说出几个：

```
Controller过胖
Service没有业务，只是转发
跨模块直接调用对方Mapper
事务范围过大或过小
只靠Java判断唯一性
查询后再更新造成并发问题
把MySQL当成普通存储，不使用约束
```

## 最后背诵版

> Controller管协议，Service管业务，Mapper管SQL，MySQL管事实和约束。Controller保持薄，Service负责状态和事务，Mapper负责查询和条件更新，数据库负责最终一致性。分层的目的不是让目录好看，而是隔离变化、方便测试，并把并发和重复数据交给数据库约束兜底。