---
tags: [Java, 泛型, 异常, 反射, 注解, Stream]
priority: P0
status: learning
---

# 泛型、异常、反射、注解与现代 Java

## 一句话结论

这些机制共同解决“类型约束、失败传播、运行期扩展和声明式编程”，也是 Spring、MyBatis 与现代 Java API 的基础。

## 一、泛型解决什么问题

泛型把类型约束前移到编译期，减少强制转换，并让容器和算法在类型安全下复用。

```java
List<String> names = new ArrayList<>();
names.add("Alice");
String first = names.get(0);
```

### 类型擦除

Java 泛型主要通过擦除实现：运行期通常没有 `List<String>` 与 `List<Integer>` 两个不同 Class。

影响：

- 不能 `new T()`、`new T[]`。
- 不能直接 `instanceof List<String>`。
- 泛型类型不能按实参重载。
- 编译器可能生成桥接方法以维持多态。
- 框架若要恢复泛型信息，需要读取字段、方法签名等元数据，而不是只看对象 Class。

## 二、通配符与 PECS

- `? extends T`：生产 T，适合读取，不能安全写入具体子类型。
- `? super T`：消费 T，适合写入 T，读取只能当 Object。
- 口诀：Producer Extends, Consumer Super。

```java
static <T> void copy(List<? extends T> src, List<? super T> dst) {
    for (T item : src) {
        dst.add(item);
    }
}
```

`List<Object>` 不是 `List<String>` 的父类型；泛型默认不变，避免把 Integer 写入实际的 String 列表。

## 三、异常体系

```text
Throwable
├─ Error：通常是严重运行环境问题
└─ Exception
   ├─ RuntimeException：非受检异常
   └─ 其他 Exception：受检异常
```

### 异常处理原则

1. 在能恢复、补充上下文或转换边界的层处理。
2. 不空 catch，不只打印后继续假装成功。
3. 包装异常时保留 cause。
4. 业务可预期失败使用稳定错误码；系统异常保留完整日志和 trace_id。
5. 日志记录一次即可，避免每层重复打印同一堆栈。

```java
try {
    repository.save(order);
} catch (DataAccessException ex) {
    throw new OrderPersistenceException("save order failed: " + order.id(), ex);
}
```

### finally 与 try-with-resources

try-with-resources 自动关闭实现 `AutoCloseable` 的资源，并正确处理关闭阶段的 suppressed exception。

```java
try (InputStream in = Files.newInputStream(path)) {
    return in.readAllBytes();
}
```

不要在 finally 中 return，它可能吞掉原异常或覆盖返回值。

## 四、反射

反射允许运行期读取类、字段、方法、构造器和注解，并动态创建或调用对象。

框架用途：

- Spring 扫描 Bean、构造对象、注入依赖、创建代理。
- MyBatis 根据 Mapper 方法和结果映射构造调用链。
- JSON 框架读取属性和类型信息。
- 测试框架发现测试方法。

代价与边界：类型错误推迟到运行期、可读性下降、封装可能被绕过、AOT/native image 需要显式反射元数据。普通业务代码优先使用明确接口。

## 五、注解

注解是元数据，不会自动执行逻辑。必须有编译器、框架、代理或反射代码读取它。

关键元注解：

- `@Target`：可标注位置。
- `@Retention`：SOURCE、CLASS、RUNTIME。
- `@Inherited`：只对类继承有特定作用，不会自动传播到方法。
- `@Repeatable`：允许重复标注。

为什么 `@Transactional` 能工作：Spring 找到注解后通过事务基础设施创建代理，代理在方法前后开启、提交或回滚事务；不是 JVM 看到注解就自动开事务。

## 六、Lambda 与函数式接口

Lambda 为只有一个抽象方法的函数式接口提供简洁实现。

常见接口：

| 接口 | 输入/输出 | 场景 |
|---|---|---|
| Predicate<T> | T → boolean | 过滤 |
| Function<T,R> | T → R | 映射 |
| Consumer<T> | T → void | 消费 |
| Supplier<T> | () → T | 延迟创建 |

捕获的局部变量必须是 final 或 effectively final，原因是 Lambda 可能在方法栈帧结束后仍执行，不能依赖可变局部槽位。

## 七、Stream

Stream 是声明式数据处理流水线，不是数据容器。

```java
Map<Status, Long> countByStatus = orders.stream()
    .filter(Order::isValid)
    .collect(Collectors.groupingBy(Order::status, Collectors.counting()));
```

关键点：

- 中间操作通常惰性，终止操作触发计算。
- Stream 通常只能消费一次。
- 避免在流水线中修改外部共享状态。
- `parallelStream()` 使用公共 ForkJoinPool，不等于自动变快；小数据、阻塞 IO、共享状态和线程上下文都可能使其更差。
- 数据库查询不要先全量加载再用 Stream 过滤，应把可下推条件交给 SQL。

## 八、Optional

Optional 适合作为可能缺失的返回值，不建议用于实体字段、DTO 字段或方法参数。

```java
return repository.findById(id)
    .orElseThrow(() -> new NotFoundException("order not found"));
```

避免 `isPresent()` 后立刻 `get()`；优先使用 `map`、`flatMap`、`orElseGet`、`orElseThrow`。

注意 `orElse` 会先计算默认值，昂贵或有副作用的默认逻辑使用 `orElseGet`。

## 九、日期时间 API

- `Instant`：时间线上的 UTC 瞬间，适合存储事件时间。
- `LocalDateTime`：没有时区，不代表全球唯一瞬间。
- `ZonedDateTime`：日期时间加时区规则。
- `Duration`：秒/纳秒时间量；`Period`：年月日历量。
- 服务端存储通常统一 UTC，展示时再转换用户时区。

## 十、容易答错

- “泛型在运行期完整保留”——通常被擦除，但签名元数据可能保留部分信息。
- “注解本身会执行”——注解只是元数据。
- “受检异常更高级”——两类异常只是编译期处理要求不同。
- “parallelStream 一定更快”——必须考虑任务粒度、池竞争和阻塞。
- “Optional 能解决所有 null”——它只是显式表达返回值可能缺失。

## 十一、高频追问

1. 为什么泛型数组难以创建？
2. `<? extends T>` 为什么不能 add？
3. try-with-resources 多个资源按什么顺序关闭？
4. Spring 如何读取运行期注解？
5. Lambda 与匿名内部类的 `this` 有什么不同？
6. Stream 的惰性求值有什么价值？

## Reference

- [[java笔记/11异常处理]]、[[java笔记/java异常处理]]
- [[java笔记/14泛型与枚举深化]]、[[java笔记/java泛型（Generics）]]
- [[java笔记/18反射]]、[[java笔记/java反射]]
- [[java笔记/19注解]]、[[java笔记/java注解]]
- [[java笔记/java日期时间]]

