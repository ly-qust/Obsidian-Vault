# 第21章 ORM框架入门（MyBatis）—— 韩顺平笔记整理

本章是JDBC的进阶升级，系统讲解了**ORM思想、MyBatis核心原理、环境搭建、CRUD操作（XML/注解方式）、动态SQL、事务管理**等内容，是企业级开发中替代原生JDBC的主流方案。

---

## 一、ORM与MyBatis核心概念

### 1. ORM思想

- **定义**：Object Relational Mapping，将Java对象与数据库表建立映射关系，通过操作对象间接操作数据库。
    
- **作用**：
    
    1. 屏蔽JDBC繁琐操作（连接、资源关闭、SQL拼接）。
        
    2. 实现对象操作到SQL操作自动转换（如`userMapper.insert(user)`）。
        
    3. 简化开发，提高效率，减少重复代码。
        

### 2. MyBatis核心原理

- **定位**：轻量级ORM，封装JDBC，支持自定义SQL，兼顾便捷性与可控性。
    
- **核心流程**：
    
    1. 加载`mybatis-config.xml`。
        
    2. 创建`SqlSessionFactory`。
        
    3. 获取`SqlSession`。
        
    4. 获取Mapper接口代理对象。
        
    5. 调用方法执行SQL。
        
    6. 提交事务并关闭`SqlSession`。
        

### 3. 核心组件

|组件|作用|
|---|---|
|`SqlSessionFactory`|会话工厂，全局唯一，用于生成`SqlSession`|
|`SqlSession`|数据库会话对象，封装JDBC连接，提供CRUD方法|
|`Mapper接口`|自定义接口，方法与SQL对应，MyBatis生成代理|
|映射文件(XML)|存储SQL语句和对象-表映射关系|
|核心配置文件|全局配置（数据库连接、映射文件、别名等）|

---

## 二、MyBatis环境搭建

### 1. 导入依赖（pom.xml）

```xml
<dependency>org.mybatis:mybatis:3.5.15</dependency>
<dependency>mysql:mysql-connector-java:8.0.30</dependency>
<dependency>log4j:log4j:1.2.17</dependency>
```

### 2. 核心配置文件（mybatis-config.xml）

- 配置环境、事务管理、数据源、映射文件扫描。
    
- 数据源可使用POOLED（内置连接池）。
    

### 3. 实体类（User.java）

- 与表`user`字段对应。
    
- 提供无参/有参构造器、getter/setter、toString。
    

### 4. Mapper接口与映射文件

- **UserMapper.java**：定义CRUD方法。
    
- **UserMapper.xml**：SQL语句映射接口方法。
    
- CRUD示例：
    
    - `select`：查询单条/多条。
        
    - `insert`：新增用户，开启主键自增。
        
    - `update`：更新用户。
        
    - `delete`：删除用户（接口参数可用`@Param`）。
        

### 5. MyBatis工具类

```java
public class MyBatisUtils {
    private static SqlSessionFactory sqlSessionFactory;
    static { /* 初始化 */ }
    public static SqlSession getSqlSession() { return sqlSessionFactory.openSession(false); }
}
```

---

## 三、CRUD操作实战

### 1. 查询（SELECT）

```java
User user = userMapper.getUserById(1);
List<User> users = userMapper.getAllUsers();
```

### 2. 新增（INSERT）

```java
int rows = userMapper.addUser(new User("李四",22,"女"));
sqlSession.commit(); // 手动提交事务
```

### 3. 更新（UPDATE）

```java
userMapper.updateUser(updateUser);
sqlSession.commit();
```

### 4. 删除（DELETE）

```java
userMapper.deleteUserById(2);
sqlSession.commit();
```

---

## 四、高级特性

### 1. 注解式开发

- 使用`@Select`, `@Insert`, `@Update`, `@Delete`直接写SQL，减少XML。
    

### 2. 动态SQL

- 标签：`if`, `where`, `foreach`, `set`。
    
- 示例：
    
    - 动态条件查询：`<where> <if test="username != null">AND username LIKE …</if> </where>`
        
    - 动态更新：`<set> <if test="age!=null">age=#{age},</if> </set>`
        
    - 批量删除：`<foreach collection="ids" item="id" open="(" separator="," close=")">#{id}</foreach>`
        

### 3. 类型别名

- 在`mybatis-config.xml`配置别名，简化映射文件：
    

```xml
<typeAliases>
    <package name="com.hspedu.entity"/>
</typeAliases>
```

---

## 五、避坑点

1. Mapper接口与XML不匹配：`namespace`、方法名、文件名、路径必须一致。
    
2. 事务未提交：增删改操作需调用`sqlSession.commit()`。
    
3. 参数传递错误：多参数需用`@Param("name")`。
    
4. 动态SQL语法错误：`where`/`set`未正确使用。
    
5. 配置文件路径错误：`mybatis-config.xml`必须在resources根目录。
    
6. 连接池参数不合理：默认值可能不适合高并发。
    
7. 日志未打印SQL：需配置log4j。
    
8. 主键自增未配置：`useGeneratedKeys="true" keyProperty="id"`。
    
9. 字段名与属性名不一致：需使用`resultMap`映射。
    

---

## 六、核心考点总结

1. **基础流程**：加载配置 → 创建`SqlSessionFactory` → 获取`SqlSession` → 获取Mapper → 执行SQL → 提交/关闭。
    
2. **核心组件**：`SqlSessionFactory`、`SqlSession`、Mapper接口、映射文件。
    
3. **操作技巧**：CRUD、动态SQL、注解开发。
    
4. **优化配置**：类型别名、包扫描、连接池优化。
    
5. **避坑**：接口与XML匹配、事务、参数、字段映射、动态SQL。
    

---

本章为企业级Java开发的MyBatis基础，Spring Boot项目ORM标配，打好基础后可学习“Spring整合MyBatis”。