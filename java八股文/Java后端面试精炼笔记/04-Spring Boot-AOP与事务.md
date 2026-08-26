---
tags: [Spring-Boot, Spring-AOP, Transactional, 事务, 面试]
---

# Spring Boot、AOP 与事务

[[03-JVM内存-GC与排查|上一篇：JVM]] · [[00-Java后端面试知识地图|知识地图]] · [[05-MyBatis核心|下一篇：MyBatis]]

## 1. 自动配置为什么“自动”

```text
@SpringBootApplication
→ 启用组件扫描、配置类和自动配置
→ 根据classpath依赖、配置属性、已有Bean判断条件
→ 条件满足时注册默认Bean
→ 用户自定义Bean时，部分自动配置通过MissingBean条件退让
```

常见条件：`@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`。自动配置不是扫描所有业务类后无条件创建 Bean，而是导入候选自动配置并做条件匹配。

## 2. AOP 与事务的关系

```text
外部调用
→ Spring代理
→ TransactionInterceptor读取事务元数据
→ TransactionManager开启/加入事务
→ 目标Service方法
→ 正常返回提交；异常按规则回滚或提交
```

`@Transactional` 是事务元数据，真正执行事务的是代理、事务拦截器和事务管理器。

## 3. 自调用为什么可能失效

默认代理模式下：

```text
外部 → Proxy → target.methodA()
                   └→ this.methodB()
```

内部 `this` 调用没有再次经过代理，因此 `methodB` 上独立的事务增强和传播配置没有机会执行。推荐将方法拆到另一个 Spring Bean，通过注入对象调用；不要为了面试只背“加注解就行”。

## 4. 回滚规则

Spring 常见默认：`RuntimeException` 和 `Error` 回滚，checked exception 默认不回滚；可用 `rollbackFor` 调整。若异常在事务方法内部被 catch 且不再抛出，代理看到正常返回，事务可能提交。

> [!note] 版本边界
> 较新的 Spring Framework 提供全局调整默认回滚规则的能力，因此面试最好说“常见默认行为”，并以项目配置为准。

## 5. REQUIRED 与 REQUIRES_NEW

| 传播 | 行为 |
|---|---|
| REQUIRED | 有事务就加入，没有就创建；常见默认 |
| REQUIRES_NEW | 挂起外层事务，创建独立事务；内层独立提交/回滚 |

连接池资源有限时，大量 `REQUIRES_NEW` 可能增加连接占用和等待；不能只把它理解成“嵌套一个事务”。

## 6. 事务失效排查

```text
Bean是否由Spring管理
→ 调用是否经过代理
→ 是否同类自调用
→ 异常是否被吞掉
→ 异常是否满足回滚规则
→ 是否跨新线程/异步边界
→ 是否使用正确的数据源和事务管理器
```

## 7. 项目落地

SmartRenew 案例统一见：[[03-申请提交与Outbox-Inbox]]、[[05-商户验券与幂等核销]]。这里不重复粘贴项目链路。

## 8. 60 秒回答

> Spring Boot 通过 `@SpringBootApplication` 启用自动配置，根据 classpath、配置属性和已有 Bean 等条件注册默认 Bean，用户自定义时会按条件退让。Spring 声明式事务基于 AOP 代理，外部调用先经过事务拦截器，再由事务管理器开启或加入事务。默认代理模式下同类 `this` 调用绕过代理，异常被吞掉或回滚规则不匹配也可能导致不回滚。REQUIRED 加入当前事务，REQUIRES_NEW 挂起外层并创建独立事务。

## 官方参考

- https://docs.spring.io/spring-boot/reference/using/auto-configuration.html
- https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html

