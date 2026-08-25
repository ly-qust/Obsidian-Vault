Lombok 的核心作用是：

> **在编译期间自动生成重复的 Java 代码，让源码更简洁。**

它不是运行时框架，也不是 Lambda。编译后的 `.class` 中仍然存在构造器、Getter、Setter 等真实方法。

## 最常用的注解

|注解|作用|
|---|---|
|`@Getter`|生成 Getter|
|`@Setter`|生成 Setter|
|`@RequiredArgsConstructor`|为未初始化的 `final`、`@NonNull` 字段生成构造器|
|`@NoArgsConstructor`|生成无参构造器|
|`@AllArgsConstructor`|生成包含全部实例字段的构造器|
|`@Data`|组合生成 Getter、Setter、`toString`、`equals`、`hashCode` 和必要参数构造器|
|`@Value`|生成偏不可变的数据类|
|`@Builder`|生成建造者模式代码|
|`@Slf4j`|生成日志对象 `log`|
|`@ToString`|生成 `toString()`|
|`@EqualsAndHashCode`|生成 `equals()` 和 `hashCode()`|
|`@NonNull`|对生成的方法或构造器参数增加非空检查|

## 1. Getter 和 Setter

原始代码：

```
public class User {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

使用 Lombok：

```
@Getter
@Setter
public class User {
    private String name;
}
```

## 2. 构造器

```
@NoArgsConstructor
@AllArgsConstructor
public class User {

    private Long id;
    private String name;
}
```

大致生成：

```
public User() {
}

public User(Long id, String name) {
    this.id = id;
    this.name = name;
}
```

Spring 中常见的是：

```
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderMapper orderMapper;
    private final PaymentService paymentService;
}
```

大致生成：

```
public OrderService(
        OrderMapper orderMapper,
        PaymentService paymentService) {
    this.orderMapper = orderMapper;
    this.paymentService = paymentService;
}
```

Spring 再通过这个构造器注入依赖。

## 3. `@Data`

```
@Data
public class UserDTO {
    private Long id;
    private String name;
}
```

它近似组合了：

```
@Getter
@Setter
@ToString
@EqualsAndHashCode
@RequiredArgsConstructor
```

适合简单 DTO，但不要看到所有类都直接加 `@Data`。

## 4. `@Builder`

```
@Builder
@Getter
public class User {
    private Long id;
    private String name;
    private Integer age;
}
```

使用：

```
User user = User.builder()
        .id(1L)
        .name("张三")
        .age(20)
        .build();
```

字段较多时，比超长构造器更容易阅读。

如果字段有默认值，需要注意：

```
@Builder.Default
private String status = "ACTIVE";
```

否则 Builder 创建对象时，普通字段初始化值可能不会成为 Builder 的默认值。

## 5. `@Slf4j`

不使用 Lombok：

```
private static final Logger log =
        LoggerFactory.getLogger(OrderService.class);
```

使用 Lombok：

```
@Slf4j
@Service
public class OrderService {

    public void createOrder() {
        log.info("开始创建订单");
    }
}
```

## 常见组合

Spring Service：

```
@Slf4j
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserMapper userMapper;
}
```

DTO：

```
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private Long id;
    private String name;
}
```

需要链式创建的请求对象：

```
@Getter
@Builder
public class CreateUserCommand {
    private String username;
    private String password;
}
```

## 使用时的注意点

- `@Data` 会生成很多东西，不一定都适合当前类。
- JPA/MyBatis 实体类使用 `@Data` 要谨慎，关联对象可能造成 `toString()` 递归，`equals/hashCode` 也可能不符合实体语义。
- `@AllArgsConstructor` 会把普通字段也加入构造器。
- `@RequiredArgsConstructor` 主要加入未初始化的 `final` 和 `@NonNull` 字段。
- `@SneakyThrows` 会隐藏受检异常，业务代码中应谨慎使用。
- 项目必须正确开启 Lombok 和注解处理，否则 IDE 可能提示找不到生成的方法。

面试可以这样回答：

> Lombok 是一个编译期代码生成工具，通过注解自动生成 Getter、Setter、构造器、Builder 和日志字段等样板代码。它可以提高开发效率，但不能为了省代码而滥用，例如实体类直接使用 `@Data`，可能引入不合适的 `equals`、`hashCode` 或 `toString` 行为。