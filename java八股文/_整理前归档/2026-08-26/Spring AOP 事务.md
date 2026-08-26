`@Transactional` 是基于 Spring AOP 和代理实现的。Spring 启动时会为带事务方法的 Bean 创建代理对象。外部调用事务方法时，会先进入代理，代理通过事务管理器开启或加入事务，然后调用真正的业务方法。方法正常结束就提交；抛出符合回滚规则的异常就回滚。因为事务依赖代理，所以同一个类内部通过 `this` 调用事务方法，可能绕过代理，导致事务不生效。

# Spring Boot 自动配置、AOP 与事务面试复习

> 范围：Spring Boot 自动配置、Spring AOP、`@Transactional`、传播、回滚、自调用、异常吞掉、SmartRenew 事务案例。

## 1. 总体知识图

```text
Spring Boot 启动
├── 组件扫描：发现自己写的 Bean
└── 自动配置：按条件提供默认 Bean
                    ↓
                 IOC 容器

业务调用
Controller
↓
Service Proxy
↓
AOP / 事务拦截器
↓
BEGIN
↓
目标 Service 方法 → Mapper / SQL
↓
正常返回 → COMMIT
异常外抛 → 判断回滚规则 → ROLLBACK / COMMIT
```

## 2. Spring Boot 自动配置

### IOC 与自动配置

- IOC：负责创建、保存、注入和管理 Bean。
- 自动配置：Spring Boot 根据项目环境，决定向 IOC 注册哪些默认 Bean。
- 手动 `new` 出来的对象默认不是 Spring Bean。

### `@SpringBootApplication`

面试主干上可理解为：

```text
@SpringBootApplication
├── @SpringBootConfiguration / @Configuration：参与配置
├── @ComponentScan：扫描自己写的组件
└── @EnableAutoConfiguration：启用自动配置
```

### 自动配置链路

```text
引入 Starter / 依赖
↓
ClassPath 出现相关类
↓
加载对应自动配置类
↓
判断 Conditional 条件
↓
条件满足
↓
创建默认 Bean 并注册到 IOC
```

常见条件：

| 注解 | 含义 |
|---|---|
| `@ConditionalOnClass` | ClassPath 存在相关类才配置 |
| `@ConditionalOnMissingBean` | 用户未提供相应 Bean 时才提供默认 Bean |
| `@ConditionalOnProperty` | 配置属性满足要求时才生效 |

核心思想：**Spring Boot 不是无脑创建 Bean，而是基于依赖、配置和已有 Bean 做条件判断。**

### 30 秒面试回答

> Spring Boot 会加载自动配置候选，并根据 ClassPath、配置属性和容器中已有 Bean 等条件进行匹配。条件满足后，自动配置类创建默认 Bean 并注册到 IOC 容器。`@ConditionalOnMissingBean` 让框架在用户已经自定义 Bean 时主动退让。

## 3. Spring AOP

### AOP 解决什么问题

事务、日志、权限和监控等公共逻辑会横跨多个业务方法。AOP 将这些横切逻辑集中管理，减少重复和业务耦合。

### 代理调用链

```text
调用者
↓
Spring Proxy
├── 方法前增强
├── 调用 Target 目标方法
└── 方法后增强
```

- JDK 动态代理：基于接口。
- 类代理/CGLIB 思想：通过目标类的子类进行增强。
- 面试核心：**调用必须经过代理，AOP 增强才有机会执行。**

### 30 秒面试回答

> AOP 用于把事务、日志等横切逻辑从业务代码中抽离。Spring 通常为目标 Bean 创建代理对象，外部调用先进入代理，由代理执行增强逻辑，再调用真正的目标方法。

## 4. `@Transactional` 实现原理

`@Transactional` 本身只是事务元数据，真正工作依赖：

```text
Spring IOC
+ AOP Proxy
+ TransactionInterceptor
+ TransactionManager
```

执行链：

```text
调用事务方法
↓
进入 Spring Proxy
↓
事务拦截器读取 @Transactional
↓
事务管理器开启事务
↓
执行目标方法和 SQL
↓
正常返回 → COMMIT
异常外抛 → 判断规则 → ROLLBACK / COMMIT
```

事务生效的关键条件：

1. 对象由 Spring 管理，而不是业务代码自己 `new`。
2. 本次调用经过 Spring 代理。
3. 数据库操作使用对应的数据源和事务管理器。
4. 异常能够传播到事务拦截器并满足回滚规则。

### 30 秒面试回答

> Spring 声明式事务基于 AOP 代理。外部调用事务方法时先进入代理，事务拦截器根据 `@Transactional` 获取事务属性，通过事务管理器开启事务，再执行目标方法；正常返回时提交，异常时根据回滚规则决定回滚还是提交。

## 5. 事务传播

传播行为解决：**事务方法调用另一个事务方法时，使用哪个事务？**

| 传播行为 | 核心含义 |
|---|---|
| `REQUIRED` | 有事务就加入，没有就创建；默认行为 |
| `REQUIRES_NEW` | 挂起外层事务，创建独立新事务 |

```text
REQUIRED：
A事务
└── B加入A事务

REQUIRES_NEW：
A事务挂起
↓
B新事务独立提交/回滚
↓
恢复A事务
```

易错点：

- `REQUIRES_NEW` 不是在原事务中简单嵌套，而是独立事务。
- 内层新事务已经提交后，外层随后回滚通常不会撤销内层结果。
- 如果发生同类自调用，内层方法的传播配置可能没有机会生效。

## 6. 回滚规则与异常吞掉

默认规则：

| 异常类型 | 默认行为 |
|---|---|
| `RuntimeException` | 回滚 |
| `Error` | 回滚 |
| checked exception | 默认通常不回滚 |

需要让 checked exception 回滚时可配置：

```java
@Transactional(rollbackFor = Exception.class)
```

异常吞掉：

```java
@Transactional
public void test() {
    try {
        update();
        doSomething();
    } catch (Exception e) {
        log.error("失败", e);
    }
}
```

```text
异常发生
↓
方法内部 catch 且不重新抛出
↓
代理看见方法正常返回
↓
事务可能 COMMIT
```

想让事务按异常回滚，通常应让异常继续向外传播，或明确把事务标记为回滚。

## 7. 自调用导致事务失效

```java
public void methodA() {
    methodB(); // 等价于 this.methodB()
}

@Transactional
public void methodB() {
}
```

```text
外部 → Proxy → 目标对象.methodA()
                    ↓
               this.methodB()
                    ↓
              没有重新经过 Proxy
```

真正原因：

```text
@Transactional 依赖代理
↓
内部 this 调用绕过代理
↓
事务拦截器没有机会介入
```

推荐处理：把 `methodB()` 拆到另一个 Spring Bean，通过该 Bean 的代理调用。

## 8. 事务失效排查链

```text
Bean 是否由 Spring 管理？
↓
调用是否经过 Proxy？
↓
是否发生 self-invocation？
↓
异常是否被 catch 吞掉？
↓
异常类型是否满足 rollback 规则？
↓
事务边界是否正确？
↓
SQL 是否使用同一事务管理器/数据源？
↓
是否跨越异步任务或新线程？
```

## 9. SmartRenew 项目事务案例

`submitForReview()` 的事务边界：

```text
@Transactional
↓
SELECT ... FOR UPDATE 锁定申请
↓
权限、状态、材料校验
↓
更新 application 为 SUBMITTED
↓
写入 PENDING Outbox
↓
正常返回 → 一起 COMMIT
异常外抛 → 一起 ROLLBACK
```

项目中的 `BizException` 继承 `RuntimeException`，默认满足回滚规则。

MQ 发送安排在 `afterCommit()`：

```text
事务内：更新业务表 + 写 Outbox
事务提交后：发送 MQ
```

这样可以避免数据库事务回滚后消息却已经发送。`FOR UPDATE` 的行锁也依赖事务边界，通常在提交或回滚后释放。

## 10. 高频面试题关键词

1. 自动配置原理：**自动配置类 + 条件判断 + Bean 注册**。
2. AOP 为什么需要代理：**不修改业务代码，在调用前后插入公共逻辑**。
3. `@Transactional` 原理：**Proxy → 拦截器 → BEGIN → Target → COMMIT/ROLLBACK**。
4. 自调用为什么失效：**`this` 调用没有重新经过 Proxy**。
5. catch 后为什么可能不回滚：**异常未到达事务拦截器，代理看到正常返回**。
6. `REQUIRED` 与 `REQUIRES_NEW`：**加入当前事务 vs 挂起外层并新建独立事务**。

## 11. 最终记忆链

```text
Spring Boot 为什么少写配置？
→ 自动配置

为什么不会乱创建 Bean？
→ Conditional

公共事务逻辑为什么不用手写？
→ AOP

AOP 怎么插入公共逻辑？
→ Proxy

@Transactional 为什么生效？
→ 调用经过 Proxy 和事务拦截器

为什么有时不生效？
→ 没经过 Proxy / 异常被吞 / 回滚规则不匹配
```
