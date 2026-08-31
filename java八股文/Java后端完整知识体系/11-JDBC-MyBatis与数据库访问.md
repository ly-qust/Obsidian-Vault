---
tags: [Java, JDBC, MyBatis, 数据访问]
priority: P0
status: learning
---

# JDBC、MyBatis 与数据库访问

## 一句话结论

MyBatis 没有绕过 JDBC：它把 Mapper 方法转换为 MappedStatement，完成参数绑定、SQL 执行和结果映射；连接、事务和数据库成本仍然存在。

## 一、JDBC 执行链

```text
DataSource获取Connection
→ 准备PreparedStatement
→ 绑定参数
→ 发送SQL给数据库
→ 数据库解析/优化/执行
→ 返回ResultSet或更新行数
→ Java对象映射
→ 提交/回滚
→ 关闭资源/归还连接池
```

## 二、PreparedStatement

参数绑定把 SQL 结构与数据分离，是防止 SQL 注入的核心手段之一，也可能帮助数据库复用执行计划。

```java
try (PreparedStatement ps = connection.prepareStatement(
        "select id, name from user where id = ?")) {
    ps.setLong(1, id);
}
```

表名、列名、排序方向等 SQL 结构不能用普通占位符绑定，必须白名单映射，不能直接拼接用户输入。

## 三、连接池

连接创建包含网络和认证成本，池化复用并限制数据库并发。

关键参数：最大连接、最小空闲、获取超时、连接超时、最大生命周期、检测策略。

排障：

- 获取连接慢：池耗尽、连接泄漏、慢 SQL、事务过长。
- 连接频繁失效：数据库/网络 idle timeout 与池生命周期不匹配。
- 最大连接过大：数据库线程、内存和锁竞争反而上升。

## 四、MyBatis 核心对象

| 对象 | 作用 | 生命周期 |
|---|---|---|
| Configuration | 全局配置、Mapper、类型处理等 | 应用级 |
| SqlSessionFactory | 创建 SqlSession | 应用级复用 |
| SqlSession | 执行、事务、本地缓存 | 请求/事务级，非线程安全 |
| MapperProxy | 把接口方法转为映射语句调用 | 通常由 Spring 管理代理 |
| MappedStatement | 一条映射语句的元数据 | 配置级 |
| Executor | 执行、缓存、批处理 | SqlSession 内部 |

不要把 SqlSession 保存为共享单例字段；在 Spring 集成中使用线程安全的 Mapper 代理，由框架管理实际会话。

## 五、Mapper 代理执行链

```text
调用Mapper接口方法
→ MapperProxy拦截
→ 根据接口名+方法名定位MappedStatement
→ 构造参数对象/BoundSql
→ Executor执行
→ ParameterHandler绑定参数
→ JDBC调用
→ ResultSetHandler映射结果
→ 返回对象
```

Mapper 接口方式比字符串 statement ID 更具类型和 IDE 支持，但 SQL 正确性仍需测试和审查。

## 六、`#{}` 与 `${}`

- `#{}` 生成 JDBC 参数占位并安全绑定值。
- `${}` 是文本替换，可能造成 SQL 注入。

动态列名/排序只能用后端枚举白名单映射：

```java
String orderBy = switch (sortField) {
    case CREATED_AT -> "created_at";
    case AMOUNT -> "amount";
};
```

## 七、参数绑定

- 单个简单参数可直接引用。
- 多参数建议使用 `@Param` 或明确 DTO，避免依赖编译参数名和隐式编号。
- 集合批量参数需要理解 `collection/list/array` 名称或显式 `@Param`。
- null 值可能需要明确 JDBC 类型。

## 八、ResultMap

ResultMap 处理列与属性映射、嵌套对象、集合、鉴别器和构造器。

- 数据库下划线与 Java 驼峰可配置自动转换，但关键映射显式写更稳。
- 一对多 join 会产生重复父行，需要结果聚合。
- 嵌套 select 容易产生 N+1。
- 大结果集要分页/流式处理，避免一次装入全部对象。

## 九、动态 SQL

常见标签：`if`、`choose`、`where`、`set`、`foreach`、`trim`。

风险：组合分支过多难以测试、空集合生成非法 SQL、批量参数超限、条件缺失导致全表更新。更新/删除必须防止无条件执行。

## 十、一级缓存

一级缓存通常属于 SqlSession/Executor 范围。查询可能命中本地缓存；更新、提交、回滚或配置的 flush 条件会清理。

边界：

- 它不是跨请求全局缓存。
- 同一事务内重复查询可能返回缓存对象，修改对象引用会影响后续观察。
- 与数据库外部更新或多实例并发不构成一致性保证。

## 十一、二级缓存

二级缓存通常按 Mapper namespace 共享，提交后才把待写条目进入真实缓存；更新会触发清理语义。

生产中慎用：跨表关联、频繁更新、对象序列化和多节点一致性会增加复杂度。明确缓存责任时，通常使用 Redis/Caffeine 并设计失效与监控，而不是“打开二级缓存就完成优化”。

## 十二、N+1 问题

先查 N 个父对象，再对每个父对象查一次关联数据，形成 1+N 次数据库往返。

解决：join、批量 `IN`、分两次批量查询后组装、按业务设计专用查询。不能只靠扩大连接池掩盖。

## 十三、分页与批量

- 深分页 `offset` 大时扫描和丢弃成本高，可用基于稳定索引的游标/seek 分页。
- 批量 insert/update 减少往返，但要控制批大小、事务大小、SQL 长度和锁持有时间。
- MyBatis Batch Executor 的语义与普通执行不同，必须显式 flush 并正确处理部分失败。
- 大事务会增加 undo、锁时间、复制延迟和故障恢复成本。

## 十四、插件机制

MyBatis 插件可拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler 等扩展点。

分页、审计等插件会改变 SQL 或执行链。排障时检查最终 BoundSql、插件顺序和数据库实际执行计划，不能只看 Mapper XML 原文。

## 十五、事务集成

Spring 管理事务时，连接与 SqlSession 绑定到当前事务上下文；Mapper 调用共享同一事务资源。

异步线程不会自动继承数据库事务。方法返回后再启动任务时，外层事务可能尚未提交或已经回滚，必须通过事件、Outbox 或明确事务后回调设计。

## 十六、高频追问

1. MyBatis Mapper 接口没有实现类为什么能执行？
2. SqlSession 为什么非线程安全？
3. `#{}` 和 `${}` 区别？
4. 一级缓存何时清理？
5. 二级缓存为什么生产慎用？
6. N+1 如何发现和解决？
7. MyBatis 与 Spring 事务如何使用同一连接？

## Reference

- [[java八股文/Java后端面试精炼笔记/05-MyBatis核心]]
- [[java笔记/21ORM与MyBatis]]
- [[java笔记/22MyBatis与Spring]]
- [[java笔记/23Spring Boot整合MyBatis]]
- [[SQL/SQL注入]]

