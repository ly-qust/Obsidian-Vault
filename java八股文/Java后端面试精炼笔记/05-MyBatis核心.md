---
tags: [Java, MyBatis, MyBatis-Plus, SQL, 面试]
---

# MyBatis 核心

[[04-Spring Boot-AOP与事务|上一篇：Spring]] · [[00-Java后端面试知识地图|知识地图]] · [[06-高频面试自测|下一篇：自测]]

## 1. 一次 Mapper 调用

```text
Mapper代理方法
→ namespace + statement id定位MappedStatement
→ 参数绑定
→ 生成SQL（含动态SQL）
→ JDBC执行
→ resultType/resultMap映射结果
```

Mapper 接口本身通常没有手写实现类，由 MyBatis 创建代理。

## 2. `#{}` 与 `${}`

| 写法 | 行为 | 使用原则 |
|---|---|---|
| `#{value}` | 预编译参数占位 | 普通值一律优先，降低注入风险 |
| `${value}` | 文本直接拼接 | 只用于无法参数化的结构片段，并做严格白名单 |

表名、列名、排序方向不能盲目接收前端字符串后使用 `${}`。

## 3. 动态 SQL

常用：`if`、`choose`、`where`、`set`、`foreach`。`where` 能按条件生成 WHERE 并处理多余 AND；`set` 能处理动态更新末尾逗号。

## 4. resultType 与 resultMap

- 字段名与属性名可以自动匹配时使用 `resultType`；
- 列名不同、嵌套对象、一对一/一对多或精确映射时使用 `resultMap`；
- 列表查询注意 N+1，优先评估 JOIN、批量查询或合理的分步加载。

## 5. MyBatis 与 MyBatis-Plus

MyBatis-Plus 提供通用 CRUD、Wrapper、分页等增强，提高简单场景效率；复杂 SQL、执行计划、索引、事务和数据库约束仍需开发者负责，必要时回到自定义 XML。

## 6. 项目例子

SmartRenew 核销条件更新位于 `VoucherMapper.xml`：把 `status` 和 `version` 写进 WHERE，并检查影响行数。MyBatis 负责执行映射，真正的并发正确性来自 SQL 条件、InnoDB 锁、事务和唯一约束。

## 7. 30 秒回答

> MyBatis 的 Mapper 调用由代理根据 namespace 和 statement id 找到映射语句，完成参数绑定、SQL 执行和结果映射。普通值使用 `#{}` 走预编译参数，`${}` 是文本拼接，只能对结构性内容做白名单后谨慎使用。MyBatis-Plus 主要减少通用 CRUD，但复杂 SQL、索引、事务和数据库约束仍然需要自己设计。

## 官方参考

- https://mybatis.org/mybatis-3/sqlmap-xml.html
- https://mybatis.org/mybatis-3/dynamic-sql.html

