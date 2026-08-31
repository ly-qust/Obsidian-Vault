---
tags: [Java, SpringMVC, AOP, 事务, SpringSecurity]
priority: P0
status: learning
---

# Spring MVC、AOP、事务与 Security

## 一句话结论

一次请求经过过滤器、安全链、DispatcherServlet、参数绑定、Controller、代理 Service 和数据访问；事务、鉴权和日志等能力都依赖明确的拦截位置与上下文边界。

## 一、Spring MVC 请求链

```text
客户端
→ 容器连接/工作线程
→ Filter链
→ Spring Security FilterChain
→ DispatcherServlet
→ HandlerMapping找到Controller
→ HandlerAdapter参数绑定/校验
→ Controller
→ Service代理
→ Mapper/数据库
→ 返回值处理/消息转换
→ 异常处理
→ HTTP响应
```

## 二、Controller 设计

- Controller 负责协议适配：参数、鉴权后的身份、校验、状态码和 DTO。
- 业务规则放 Service/领域层，不把数据库操作堆在 Controller。
- Entity 不直接作为外部请求/响应契约，避免字段泄漏和模型耦合。
- URL 表达资源，HTTP 方法表达操作语义；关键写操作考虑幂等键。

## 三、参数绑定与校验

- 使用请求 DTO + Jakarta Validation。
- 字段格式校验不等于业务校验；“申请额度是否足够”需要查询事实源并在事务中再次校验。
- 统一异常处理将校验失败、业务冲突、未认证、无权限、系统异常映射到稳定错误结构。
- 错误响应包含 trace_id，但不返回堆栈、SQL、密钥等内部信息。

## 四、Filter、Interceptor、AOP

| 机制 | 所在层 | 适合 |
|---|---|---|
| Filter | Servlet 容器链 | 编码、CORS、安全链、请求包装 |
| HandlerInterceptor | Spring MVC Handler 前后 | 路由级审计、上下文、通用检查 |
| AOP | Spring Bean 方法调用 | 事务、指标、缓存、方法级横切逻辑 |

不要混用：AOP 不一定覆盖非 Spring Bean，Interceptor 不拦截普通 Service 调用，Filter 看不到完整 Controller 方法语义。

## 五、AOP 代理

Spring AOP 常通过代理拦截 Bean 方法调用：

```text
调用方 → 代理 → Advisor/Interceptor链 → 目标方法
```

- 有接口时可使用 JDK 动态代理；也可使用基于子类的代理。
- final 类/方法、private 方法等会限制子类代理拦截。
- 自调用 `this.method()` 没有经过外部代理，事务、缓存、异步等可能失效。
- 代理只拦截符合切点且从代理对象进入的调用。

## 六、声明式事务机制

事务拦截器大致执行：

```text
根据注解获取事务属性
→ 从事务管理器获取/创建事务
→ 绑定连接等资源到当前执行上下文
→ 调用目标方法
→ 正常则提交
→ 满足回滚规则的异常则回滚
→ 清理资源
```

事务不是注解本身完成的，而是代理 + 事务管理器 + 数据源连接共同完成。

## 七、传播行为

| 传播 | 语义 | 典型边界 |
|---|---|---|
| REQUIRED | 有事务加入，没有则新建 | 默认主流程 |
| REQUIRES_NEW | 挂起外层，创建独立事务 | 独立提交；连接池压力增加 |
| SUPPORTS | 有则加入，无则非事务 | 只读辅助但语义较弱 |
| MANDATORY | 必须已有事务 | 强制调用上下文 |
| NOT_SUPPORTED | 挂起事务，非事务执行 | 明确不参与事务 |
| NEVER | 有事务则报错 | 禁止事务上下文 |
| NESTED | 保存点式嵌套，依赖管理器支持 | 不等于独立物理事务 |

`REQUIRES_NEW` 可能让一个外层线程同时需要额外连接；并发高且连接池小会互相等待。

## 八、隔离级别与回滚

Spring 隔离级别最终由数据库和驱动实现。不要只背枚举，要回到 MySQL MVCC 与锁。

默认情况下，Spring 声明式事务通常对 RuntimeException 和 Error 回滚；受检异常需要按规则配置。更重要的是：异常不能在事务方法内被吞掉并返回“成功”。

## 九、事务失效清单

- 对象不是 Spring Bean，手动 new。
- 自调用未经过代理。
- 方法不满足代理可拦截条件。
- 异常被 catch 后未重新抛出/未标记回滚。
- 异常类型不匹配回滚规则。
- 新线程/异步任务不共享原事务。
- 多数据源使用错误事务管理器。
- 数据库表或操作本身不支持预期事务语义。
- 把数据库事务误认为能原子覆盖 Redis、MQ、HTTP。

## 十、Spring Security 主线

```text
请求
→ Security FilterChain
→ 提取凭证/JWT
→ 验证签名、有效期和声明
→ 构造Authentication
→ 写入SecurityContext
→ URL/方法授权
→ Controller中的资源归属校验
```

认证回答“你是谁”，授权回答“你能做什么”。有某个角色不代表能操作任意资源；还需要对象级归属校验。

## 十一、JWT 边界

- JWT 通常是签名而不是加密，Payload 不放敏感信息。
- 验证算法、密钥、签发者、受众、过期时间，防止算法混淆和弱密钥。
- 短期 Access Token + 可撤销/轮换的 Refresh Token 是常见思路。
- 无状态 Token 的即时吊销较难，需要版本、黑名单、短过期或集中会话机制。
- 日志不记录完整 Token。

## 十二、CORS、CSRF 与 XSS

- CORS 是浏览器跨源策略，不是服务端身份认证。
- CSRF 利用浏览器自动携带凭证；是否需要防护取决于认证载体和请求方式。
- XSS 是不可信内容在页面执行；后端需输出编码、内容安全策略和输入治理协同。
- 参数化 SQL 防 SQL 注入，不要拼接用户输入。

## 十三、高频追问

1. Filter、Interceptor、AOP 的执行位置？
2. 为什么同类调用导致事务失效？
3. REQUIRED 与 REQUIRES_NEW 的连接池风险？
4. 事务能覆盖 MQ 和 Redis 吗？
5. JWT 为什么不能只 decode 不 verify？
6. 已认证用户为什么仍需资源归属校验？

## Reference

- [[javaweb/令牌验证 过滤器和拦截器]]
- [[SR/SmartRenew面试精炼笔记/02-Security与JWT请求链]]
- [[java八股文/Java后端面试精炼笔记/04-Spring Boot-AOP与事务]]
- [[MySQL八股文/MySQL面试精炼笔记/03-事务-MVCC与锁]]

