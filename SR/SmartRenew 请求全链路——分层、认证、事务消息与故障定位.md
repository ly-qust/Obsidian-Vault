## 一张总图

```
客户端
→ Nginx 反向代理
→ Spring Security 过滤链
→ JWT 认证
→ 角色授权
→ Spring MVC 参数绑定
→ DTO Validation
→ Controller
→ Service
→ Mapper
→ MySQL
→ 统一 HTTP 响应
```

记忆核心：

```
Security 管身份和权限
Controller 管 HTTP
Service 管业务和事务
Mapper 管数据库访问
MySQL 管事实和约束
```

---

## 一、DI：对象为什么不自己 new？

```
Spring 扫描组件
→ 创建 Bean
→ 管理依赖关系
→ 构造器注入
→ Controller → Service → Mapper
```

Controller 不自己 `new ServiceImpl`，因为：

- 会依赖具体实现，耦合高。
- 不方便替换实现和单元测试。
- 可能绕过 `@Transactional` 等 Spring 代理能力。
- 对象创建和业务使用混在一起，职责不清。

面试表达：

> DI 是由 Spring 容器创建和管理 Bean，并将依赖注入使用方。Controller 只依赖 Service 接口，不负责创建具体实现，从而降低耦合、方便测试，也能保证事务代理等容器能力正常生效。

关键词：

```
Spring 容器 / Bean / 构造器注入 / 解耦 / 事务代理
```

---

## 二、REST：把一次操作表达清楚

一次接口由以下部分组成：

```
HTTP 方法
+ URL
+ 路径/查询参数
+ 请求体
+ 状态码
+ 响应 JSON
```

SmartRenew 示例：

```
POST /api/v1/applications
请求体：ApplicationCreateRequest
```

```
POST /api/v1/applications/{id}/submit
路径参数：id
请求体：无
```

易错点：

> 创建申请有 DTO 请求体；提交审核只有路径参数，不要混淆。

---

## 三、Validation：什么规则应该在哪一层？

请求校验链：

```
JSON
→ DTO
→ @Valid
→ 字段约束
→ 校验异常
→ GlobalExceptionHandler
→ HTTP 400
```

真实规则：

```
campaignId  → @NotNull
regionCode  → @NotBlank
itemPrice   → @NotNull + 必须大于 0
```

`itemPrice = 0` 时：

```
Spring MVC 参数校验失败
→ Controller 方法体不执行
→ Service 不执行
→ 返回 HTTP 400
```

规则分层：

|规则类型|应放位置|
|---|---|
|非空、格式、长度、数值范围|DTO Validation|
|资格、库存、状态、数据归属|Service|
|唯一性、外键、最终不变量|MySQL 约束|

面试表达：

> DTO 校验解决输入是否合法，Service 校验解决业务是否允许，数据库约束负责并发情况下的最终正确性。

---

## 四、申请提交：Controller、Service、Mapper 分别做什么？

真实入口：

```
POST /api/v1/applications/{id}/submit
→ ApplicationController.submitForReview()
→ ApplicationService.submitForReview()
→ ApplicationMapper
→ MySQL
```

Controller：

```
接收 applicationId
→ 从 SecurityContext 获取当前 userId
→ 调用 Service
→ 包装 Result 响应
```

Service：

```
锁定申请
→ 检查是否存在
→ 检查数据归属
→ 检查申请状态
→ 校验材料
→ 更新为 SUBMITTED
→ 写入 Outbox
→ 构造响应 DTO
```

Mapper：

```
SELECT ...
FROM application
WHERE id = ?
FOR UPDATE;
```

为什么使用 `FOR UPDATE`？

```
请求 A 锁定 DRAFT
→ A 更新并提交
→ 释放锁
→ 请求 B 获得锁
→ B 读到 SUBMITTED
→ 状态校验失败
```

面试表达：

> `FOR UPDATE` 配合事务，避免同一份申请被并发重复提交。第二个事务拿到锁后会看到最新状态，因此不会再次写 Outbox。

---

## 五、Outbox：业务成功但消息没发出去怎么办？

```
同一本地事务：
更新 application
+ 插入 mq_outbox
→ 一起提交或一起回滚
```

事务提交后：

```
应用程序 afterCommit 发送 RabbitMQ
→ 失败则保留 Outbox
→ 定时任务扫描并重发
```

必须准确表达：

> 不是数据库发送 RabbitMQ，而是应用程序读取 Outbox 后发送。

Outbox 保证的是：

```
消息有补偿依据
不是只发送一次
```

为什么可能重复发送？

```
RabbitMQ 已收到
→ Outbox 还没标记 SENT
→ 应用宕机
→ 重启后再次发送
```

因此：

```
Outbox → 至少一次发送
Consumer → 必须幂等
```

---

## 六、Inbox：重复消息为什么仍是难点？

典型重复投递窗口：

```
消费者业务处理成功
→ ACK 前宕机
→ RabbitMQ 重投
```

Inbox 的作用：

```
检查 messageKey
→ 已成功处理：跳过业务并 ACK
→ 未处理：执行业务、记录 Inbox、再 ACK
```

一句话：

> Outbox 防止该发的消息永久丢失，Inbox 防止重复消息造成重复业务。

### 当前实现边界

```
审核业务事务提交成功
→ Inbox 记录前宕机
→ RabbitMQ 重投
→ Inbox 查不到成功记录
→ 再次进入业务方法
```

核心原因：

> 审核业务事务和 Inbox 成功记录不在同一个本地事务中，不具备原子性。

当前其他防线：

```
申请状态检查
+ 条件更新/乐观锁
+ 数据库唯一约束
```

更严格方案：

```
同一本地事务
→ 利用唯一键插入 Inbox 占位
→ 执行业务
→ 标记 SUCCESS
→ 一起提交
→ 最后 ACK
```

这是项目面试中最有含金量的“实现边界说明”：

> 可以说明当前方案、剩余故障窗口和改进方向，但不能把改进方案说成已经实现。

---

## 七、登录与 JWT 认证链

登录：

```
LoginRequest
→ AuthController.login()
→ AuthService.login()
→ 查询用户
→ PasswordEncoder 校验密码
→ 查询角色
→ JwtTokenProvider 生成 Token
→ LoginResponse
```

Token 包含或关联：

```
userId
username
roles
```

带 Token 请求：

```
Authorization: Bearer <JWT>
```

处理链：

```
Spring Security Filter Chain
→ JwtFilter 读取 Authorization
→ 去掉 Bearer 前缀
→ 验证并解析 JWT
→ 创建 Authentication
→ 放入 SecurityContextHolder
→ 执行授权判断
```

关键纠正：

> JwtFilter 本身属于 Spring Security 过滤链，不是“先经过 JWT，再经过 Security”。

Controller 为什么知道用户已登录？

> 因为 JWT Filter 已经把当前用户身份和角色放入 SecurityContext，后续代码可以通过它获取当前用户。

---

## 八、认证和授权不要混淆

```
认证：你是谁？
授权：你能做什么？
```

|情况|状态码|优先排查|
|---|---|---|
|没有 Token、Token 无效或过期|401|JwtFilter、认证入口|
|Token 有效但角色不允许|403|Security 配置、角色守卫|
|参数不合法|400|JSON、DTO、Validation|
|业务状态冲突|409|Service、数据库约束|
|未处理的内部异常|500|Service、Mapper、基础设施|

这些只是排查入口，不是绝对结论。

---

## 九、SmartRenew 权限矩阵

|角色|核心权限|控制位置|
|---|---|---|
|USER|创建、查询、提交自己的申请|认证 + Service 归属校验|
|REVIEWER|审核中心|`ReviewAccessGuard`|
|ADMIN|管理接口、审核中心|`SecurityConfig` + `ReviewAccessGuard`|
|MERCHANT|验券、核销、核销记录|`MerchantAccessGuard`|

特别注意两种 403：

```
USER 访问 /api/v1/admin/**
→ SecurityConfig 拒绝
→ Controller 不执行
```

```
USER 访问审核中心
→ 先通过“已认证”
→ Controller 方法开始执行
→ ReviewAccessGuard 拒绝
→ BizException → HTTP 403
```

状态码相同，失败位置可能不同。

---

## 十、异常处理边界

业务异常：

```
Service 抛 BizException
→ GlobalExceptionHandler
→ 映射 HTTP 状态
→ Result.fail
→ ResponseEntity
```

Security 异常：

```
未认证 401
→ authenticationEntryPoint

无权限 403
→ accessDeniedHandler
```

为什么不能每个 Controller 都写 `try/catch`？

- 重复代码多。
- 响应格式容易不一致。
- Controller 职责混乱。
- 日志和状态码映射容易遗漏。

---

## 十一、今天建立的调试方法

看到错误不要从第一行异常开始猜：

```
先看 HTTP 状态码
→ 判断可能失败层
→ 找响应中的 traceId
→ 用 traceId 查日志
→ 找最底层 Caused by
→ 判断应该改哪一层
→ 修改后重新运行验证
```

真实启动故障：

```
ClassNotFoundException:
org.apache.commons.lang3.StringUtils
```

定位过程：

```
最底层异常是缺类
→ 检查 pom.xml
→ commons-lang3 被限制为 test 作用域
→ 正式运行 classpath 中不存在
```

最小修复：

```
删除 <scope>test</scope>
```

验证闭环：

```
重新构建
→ 应用启动成功
→ 8080 监听
→ 原异常消失
→ 发送真实 HTTP 请求
```

这体现了正确的 AI 协作方式：

> AI 给出修改后，不能只看“代码能编译”，还要判断修改层是否正确，并通过启动、HTTP 响应和日志证明修改有效。

---

## 十二、面试总回答deng

> SmartRenew 的请求首先通过 Nginx 反向代理进入 Spring Boot，然后经过 Spring Security 过滤链。JwtFilter 从 Authorization 请求头提取并解析 Token，将用户身份和角色放入 SecurityContext，Security 配置和角色守卫继续完成授权。认证授权通过后，Spring MVC 完成路由匹配，将 JSON 转换为 DTO，并在 Controller 方法执行前通过 `@Valid` 完成参数校验。Controller 负责 HTTP 协议适配，Service 负责业务规则、状态流转和事务，Mapper 负责访问 MySQL，数据库通过事务、唯一约束和条件更新保证最终数据正确性。业务异常由 GlobalExceptionHandler 转为统一响应，而认证和授权异常由 Security Handler 处理。申请提交时，业务状态和 Outbox 在同一本地事务中落库，事务提交后应用发送 RabbitMQ，失败由定时任务补偿；消费端使用 Inbox、状态检查和数据库约束降低重复消费影响，同时我也能说明当前业务事务与 Inbox 记录不原子的实现边界。

## 最短唤醒词

```
DI：容器创建，构造器注入
分层：Controller 协议，Service 业务，Mapper 访问
Validation：输入合法，不等于业务允许
Security：认证是谁，授权能做什么
Outbox：防丢，可重发
Inbox：防重，但当前不原子
数据库：唯一约束最终兜底
调试：状态码 → traceId → 根因 → 分层修改 → 运行验证
```