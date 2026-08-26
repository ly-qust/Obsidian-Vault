## 一、整体调用链

```
前端
  ↓ HTTP 请求
Controller
  ↓ DTO / 参数
Service
  ↓ Entity / 查询参数
Mapper
  ↓ SQL
MySQL
```

一句话记忆：

> **DTO 装数据，Controller 管协议，Service 管业务，Mapper 管 SQL，MySQL 管数据、事务和约束。**

---

## 二、面试回答

> 项目后端采用 Controller、Service、Mapper 分层结构。Controller 负责 HTTP 协议层工作，例如接收参数、基础校验、获取当前登录用户以及返回响应；DTO 用于在接口和不同层之间传递数据；Service 负责核心业务规则、状态流转和事务编排；Mapper 负责执行 SQL；MySQL 负责数据持久化、事务和唯一约束。
> 
> 这样可以把接口协议、业务逻辑和数据库访问分开，降低耦合，方便测试和后续修改。例如，申请提交的业务判断放在 Service，申请状态和审核记录的 SQL 放在 Mapper，相关数据库修改由 MySQL 事务保证原子性。

---

## 三、DTO：负责传递数据

DTO 是 Data Transfer Object，即数据传输对象。

主要作用：

```
接收接口请求数据
返回接口响应数据
在不同层之间传递必要数据
避免直接暴露数据库实体
```

例如：

```
public class ApplicationSubmitRequest {
    private String remark;
}
```

Controller 可以用它接收请求体：

```
@PostMapping("/applications/{id}/submit")
public Result<Void> submit(
        @PathVariable Long id,
        @RequestBody ApplicationSubmitRequest request) {
    // ...
}
```

DTO 一般不负责：

```
执行SQL
开启事务
编排复杂业务
```

---

## 四、Controller：负责 HTTP 协议层

Controller 是 HTTP 请求进入后端的入口。

主要负责：

```
匹配请求地址和HTTP方法
接收路径参数、查询参数和请求体
执行基础格式校验
获取当前登录用户
调用Service
返回统一响应
```

例如：

```
@RestController
@RequestMapping("/applications")
public class ApplicationController {

    private final ApplicationService applicationService;

    @PostMapping("/{id}/submit")
    public Result<Void> submit(
            @PathVariable Long id,
            @RequestBody ApplicationSubmitRequest request,
            CurrentUser currentUser) {

        applicationService.submit(id, request, currentUser.getId());
        return Result.success();
    }
}
```

Controller 应该保持较薄，不应直接完成：

```
判断申请是否满足全部业务规则
编排多个数据库修改
处理复杂状态流转
直接扣减额度
直接操作多个Mapper
```

这些应该交给 Service。

一句话记忆：

> **Controller 负责接住请求、取出参数、调用 Service、返回结果。**

---

## 五、Service：负责业务规则和事务编排

Service 是业务逻辑的核心层。

主要负责：

```
业务规则判断
权限和资源归属校验
状态流转
事务边界
调用多个Mapper
调用其他业务模块提供的Service
```

例如，提交申请可能需要：

```
① 查询申请
② 判断申请是否属于当前用户
③ 判断状态是否为DRAFT
④ 判断必要材料是否齐全
⑤ 判断活动是否有效
⑥ 修改申请状态
⑦ 创建审核任务
```

示例：

```
@Service
public class ApplicationService {

    private final ApplicationMapper applicationMapper;
    private final ReviewTaskMapper reviewTaskMapper;

    @Transactional
    public void submit(
            Long applicationId,
            ApplicationSubmitRequest request,
            Long userId) {

        Application application =
                applicationMapper.selectById(applicationId);

        if (application == null) {
            throw new BusinessException("申请不存在");
        }

        if (!application.getUserId().equals(userId)) {
            throw new BusinessException("无权提交该申请");
        }

        if (!ApplicationStatus.DRAFT.equals(application.getStatus())) {
            throw new BusinessException("当前状态不能提交");
        }

        int updated = applicationMapper.submit(
                applicationId,
                ApplicationStatus.DRAFT,
                ApplicationStatus.SUBMITTED);

        if (updated != 1) {
            throw new BusinessException("申请状态已经发生变化");
        }

        reviewTaskMapper.insert(applicationId);
    }
}
```

一句话记忆：

> **Service 决定业务能不能做、按照什么顺序做，以及哪些数据库操作必须共同成功。**

---

## 六、Mapper：负责数据库访问

Mapper 负责执行 SQL，以及 Java 对象与数据库记录之间的映射。

主要负责：

```
查询数据
插入数据
更新数据
删除数据
执行条件更新
返回SQL影响行数
```

例如：

```
@Mapper
public interface ApplicationMapper {

    Application selectById(Long id);

    int submit(
            @Param("id") Long id,
            @Param("oldStatus") String oldStatus,
            @Param("newStatus") String newStatus);
}
```

对应的条件更新：

```
UPDATE application
SET status = #{newStatus}
WHERE id = #{id}
  AND status = #{oldStatus}
```

然后根据影响行数判断：

```
影响行数 = 1：更新成功
影响行数 = 0：数据不存在或状态已经变化
```

Mapper 一般不负责：

```
决定完整业务流程
编排多个业务步骤
处理HTTP请求和响应
```

一句话记忆：

> **Mapper 只关心数据库怎么查、怎么改。**

---

## 七、MySQL：负责数据、事务和约束

MySQL 不只是保存数据，还负责：

```
数据持久化
本地事务
唯一约束
非空约束
外键或其他数据约束
条件更新
并发控制
```

例如，同一份申请不能重复发券，可以增加唯一约束：

```
ALTER TABLE coupon_record
ADD CONSTRAINT uk_application_id UNIQUE (application_id);
```

即使出现接口重试或消息重复消费，数据库也能阻止重复记录。

需要注意：

> Java 业务判断能够提供友好的错误提示，但数据库约束才是防止重复数据的最后防线。

---

## 八、为什么不能把逻辑全部写在 Controller

面试回答：

> 如果 Controller 同时负责参数、权限、状态、事务和数据库操作，它会变得难以测试和维护。当同一业务还需要被定时任务、消息消费者或其他接口调用时，也容易产生重复代码。因此 Controller 应该保持较薄，把可复用的业务规则集中到 Service。

错误结构：

```
Controller
├── 接收参数
├── 判断状态
├── 执行业务
├── 调用多个Mapper
├── 控制事务
└── 返回结果
```

推荐结构：

```
Controller：接收请求、调用Service、返回结果
Service：处理业务、状态和事务
Mapper：执行SQL
```

---

## 九、Service 只有一行 Mapper 调用可以吗

面试回答：

> 简单查询中，Service 只有一行 Mapper 调用是可以接受的。但复杂业务必须由 Service 负责权限、状态、事务和多个数据库操作的编排。不能为了分层形式强行制造复杂逻辑，也不能因此把复杂业务放到 Controller。

判断标准：

```
简单CRUD
→ Service可以很薄

复杂业务
→ Service必须承担业务规则和事务编排
```

---

## 十、事务为什么通常放在 Service

事务通常放在 Service 的业务方法上：

```
@Transactional
public void submitApplication(...) {
    applicationMapper.updateStatus(...);
    reviewTaskMapper.insert(...);
}
```

原因是：

> Service 最清楚一次完整业务操作包含哪些数据库修改，以及这些修改是否必须共同成功。

```
全部成功 → 提交事务
任意一步失败 → 回滚事务
```

不推荐把主要事务边界放在 Controller，因为：

```
事务范围容易过大
可能包含参数转换和网络调用
Controller不应该决定业务事务边界
```

需要特别注意：

> MySQL 事务不能自动覆盖 RabbitMQ、Redis、HTTP 调用和文件操作。跨系统一致性通常需要 Outbox、幂等、重试或补偿机制。

---

## 十一、查询后再更新为什么有并发问题

假设剩余额度为 10：

```
请求A查询到10
请求B也查询到10

请求A判断额度足够
请求B也判断额度足够

A扣减8
B也扣减8
```

两个请求都认为额度足够，可能造成超额使用。

不推荐只在 Java 中：

```
先查询
↓
判断
↓
再更新
```

更可靠的方式是让数据库执行一次带条件的原子更新：

```
UPDATE quota_pool
SET available_amount = available_amount - #{amount}
WHERE id = #{id}
  AND available_amount >= #{amount};
```

然后检查影响行数：

```
影响行数 = 1
→ 扣减成功

影响行数 = 0
→ 额度不足、记录不存在或发生并发竞争
```

一句话记忆：

> **不要只依赖“先查再改”，尽量把判断条件放进 UPDATE，并检查影响行数。**

---

## 十二、更复杂的架构：六边形架构

当领域规则非常复杂时，可以进一步采用六边形架构：

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

面试回答：

> 当前项目采用 Controller、Service、Mapper 分层已经足够清晰。如果领域规则继续变复杂，可以采用六边形架构，把 HTTP 和数据库作为外部适配器，将核心业务放入 Domain 层，并通过 Repository 接口隔离基础设施。但这种设计会增加抽象数量，不应该为了架构形式而过度设计。

---

## 十三、常见问题

```
Controller过胖
Service只是机械转发
业务逻辑写进Mapper
跨模块直接调用对方Mapper
事务范围过大或过小
在数据库事务中执行耗时网络调用
只靠Java代码判断唯一性
先查询再更新造成并发问题
忽略SQL更新影响行数
把MySQL当成普通存储，不使用约束
误认为MySQL事务能够回滚消息和HTTP调用
```

---

## 十四、最终背诵版

> 项目后端采用 Controller、Service、Mapper 分层。DTO 用于传递接口数据；Controller 负责接收 HTTP 请求、基础参数校验和返回响应；Service 负责业务规则、状态流转和事务编排；Mapper 负责执行查询和条件更新；MySQL 负责数据持久化、本地事务和数据约束。
> 
> 分层的目的不是让目录更好看，而是隔离接口协议、业务逻辑和数据库访问，降低耦合并方便测试。Controller 应该保持较薄，复杂业务集中在 Service，并通过数据库条件更新、事务和唯一约束处理并发与重复数据问题。

超短口诀：

> **DTO 装数据，Controller 接请求，Service 做业务，Mapper 跑 SQL，MySQL 守数据。**