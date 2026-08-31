---
tags: [Java, Spring, IOC, Bean生命周期, SpringBoot]
priority: P0
status: learning
---

# Spring IOC、Bean 生命周期与 Boot 自动配置

## 一句话结论

Spring 的核心是容器管理对象定义、创建、依赖关系和扩展点；Spring Boot 在此基础上根据 classpath、配置和已有 Bean 条件化装配生产级应用。

## 一、IOC 与 DI

- IOC：对象的创建、组装和生命周期由容器控制。
- DI：容器把依赖传给对象，是实现 IOC 的主要方式。
- 价值：业务依赖抽象、基础设施可替换、配置集中、横切能力可统一代理。

构造器注入优先：依赖明确、便于测试、支持 final、对象创建后状态完整。字段注入隐藏依赖并增加测试和代理复杂度。

## 二、BeanDefinition

容器不是直接“扫描到对象”，而是先获得 BeanDefinition：类、作用域、构造参数、属性、初始化方法、条件等元数据，再实例化 Bean。

来源包括：组件扫描、`@Bean` 方法、自动配置、XML、编程式注册。

## 三、容器启动主线

```text
创建ApplicationContext
→ 读取配置与BeanDefinition
→ 执行BeanFactory/BeanDefinition后处理器
→ 注册BeanPostProcessor
→ 创建非懒加载单例
→ 依赖注入与生命周期回调
→ 发布容器事件
→ 应用就绪
```

面试不需要逐行背 refresh 源码，但要知道“定义阶段、实例化阶段、后处理器阶段、事件阶段”不同。

## 四、Bean 生命周期

典型顺序：

```text
实例化
→ 属性填充/依赖注入
→ Aware回调
→ BeanPostProcessor before
→ @PostConstruct / InitializingBean / init-method
→ BeanPostProcessor after（可能返回代理）
→ 对外使用
→ @PreDestroy / DisposableBean / destroy-method
```

实际顺序受扩展点和代理影响。业务代码不要过度依赖多个初始化机制的细枝末节，选择一种清晰方式。

## 五、Bean 作用域

- singleton：每个容器一个实例，不等于 JVM 全局单例，也不自动线程安全。
- prototype：每次获取创建新实例；容器通常不完整管理销毁。
- request/session：Web 请求或会话范围，依赖 Web 上下文和作用域代理。

单例 Service 应尽量无状态；请求数据放参数、局部变量或明确的请求上下文。

## 六、循环依赖

构造器循环依赖无法在对象完整创建前解决，通常直接失败，暴露设计耦合。

部分单例字段/Setter 循环依赖可通过提前暴露引用等机制处理，但：

- 不是所有场景都支持。
- prototype、构造器、代理时机等会影响结果。
- 解决循环依赖不等于设计合理。

优先重构职责：抽取第三个服务、发布领域事件、调整依赖方向。

## 七、BeanPostProcessor 与代理

BeanPostProcessor 可在 Bean 初始化前后处理实例，Spring AOP 自动代理就是重要用途之一。

因此，从容器获得的 Bean 可能不是原始对象，而是代理对象。绕过容器手动 `new` 出来的对象不会获得依赖注入、事务、缓存等容器能力。

## 八、Spring Boot 自动配置

自动配置本质是普通配置类加条件判断。常见条件：

- classpath 是否存在某个类。
- 容器是否已有某个 Bean。
- 配置属性是否开启。
- 当前应用类型或资源是否满足。

官方文档中的 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 表明：Boot 不是“魔法扫描一切”，而是条件化注册 Bean。

排查自动配置：查看启动条件评估、调试日志或 Actuator 的 `/actuator/conditions`，确认哪些条件匹配/不匹配。

## 九、Starter 与自动配置

- Starter 主要提供依赖集合和默认组合。
- Auto-configuration 提供条件化 Bean 定义。
- 引入 Starter 不代表所有功能自动启用；仍取决于 classpath、属性和用户 Bean。
- 用户自定义 Bean 常通过 `@ConditionalOnMissingBean` 让默认配置“退后”。

## 十、外部化配置

配置可能来自 properties/yaml、环境变量、系统属性、命令行、配置中心等，并有优先级覆盖规则。

推荐把一组业务配置绑定为类型安全的 `@ConfigurationProperties`，配合 `@Validated` 和 Jakarta Validation 在启动时失败，而不是运行到请求时才发现配置缺失。

```java
@ConfigurationProperties(prefix = "app.review")
@Validated
public record ReviewProperties(
    @NotNull Duration timeout,
    @Min(1) int maxRetry
) {}
```

敏感信息不写入仓库；配置覆盖顺序要可解释，避免同一属性散落多处。

## 十一、Profile

Profile 适合环境差异，不应承载复杂业务分支。多个 Profile 和配置位置同时启用时要明确最后生效值；排障先打印来源或使用 Actuator/environment 工具，不凭文件名猜。

## 十二、Spring Boot Web 启动

```text
main
→ SpringApplication.run
→ 创建ApplicationContext
→ 准备Environment
→ 加载BeanDefinition和自动配置
→ 创建嵌入式WebServer
→ 注册Servlet/Filter等
→ 启动端口并发布Ready事件
```

端口、线程、连接、超时等可通过配置或 WebServerFactoryCustomizer 定制，但调整前必须理解容器和下游容量。

## 十三、Actuator

Actuator 提供健康、指标、条件评估、配置等生产信息。端点暴露必须最小化并保护权限，不能把环境变量、密钥和内部拓扑公开到公网。

## 十四、容易答错

- “IOC 就是反射”——反射只是可能使用的实现工具，核心是对象控制权和依赖管理。
- “singleton Bean 线程安全”——作用域不等于线程安全。
- “Spring 能解决所有循环依赖”——错误且设计上不应依赖这种能力。
- “Boot 自动配置不需要配置”——它基于默认值和条件，仍可被配置和用户 Bean 覆盖。

## 十五、高频追问

1. 构造器注入为什么优于字段注入？
2. BeanPostProcessor 与 BeanFactoryPostProcessor 区别？
3. Bean 什么时候变成代理？
4. Boot 自动配置为何能按条件退让？
5. 配置属性校验为什么应在启动期失败？
6. 如何排查某个自动配置没有生效？

## Reference

- [[java八股文/Java后端面试精炼笔记/04-Spring Boot-AOP与事务]]
- [[java八股文/_整理前归档/2026-08-26/Spring AOP 事务]]
- [[javaweb/IOC]]、[[javaweb/Spring Boot]]

