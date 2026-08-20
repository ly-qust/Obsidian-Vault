# 第22章 MyBatis与Spring整合开发——基于韩顺平笔记全重点解析

本章是企业级开发的核心衔接内容，韩顺平笔记中系统讲解了**Spring与MyBatis整合的核心原理、整合步骤、配置方式、事务统一管理、实战开发规范**等内容。整合后可借助Spring的IOC容器管理MyBatis核心组件（如SqlSessionFactory、Mapper接口），通过Spring事务管理器统一控制事务，彻底简化开发流程。以下严格遵循笔记框架，从“整合原理→环境搭建→配置核心→实战案例→避坑对比”全维度解析，确保覆盖所有重点。

---

## 一、整合核心原理与优势（笔记22.1节）

### 1. 整合核心逻辑

MyBatis与Spring整合的核心是“将MyBatis的核心组件交给Spring IOC容器管理”，替代MyBatis原生的工具类操作，具体分工：

- **Spring负责**：管理数据库连接池（DataSource）、创建SqlSessionFactory、管理SqlSession、扫描Mapper接口并生成代理对象、统一管理事务。
    
- **MyBatis负责**：编写Mapper接口、配置SQL映射（XML/注解）、执行SQL操作、结果集映射。
    

### 2. 整合核心优势

1. **简化配置**：无需手动创建SqlSessionFactory和SqlSession，Spring自动注入Mapper接口。
    
2. **统一事务管理**：Spring事务管理器替代MyBatis手动提交/回滚，支持声明式事务（注解方式）。
    
3. **组件解耦**：通过依赖注入（DI）获取Mapper对象，无需硬编码获取资源。
    
4. **性能优化**：Spring整合第三方连接池（如HikariCP），提升连接复用效率。
    

---

## 二、整合环境搭建（笔记22.2节，核心基础）

以Maven项目为例，基于Spring 5.x + MyBatis 3.x + MySQL 8.x，按以下步骤完成整合。

### 1. 步骤1：导入整合所需Maven依赖

```xml
<!-- Spring核心依赖 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.3.20</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.3.20</version>
</dependency>

<!-- MyBatis核心依赖 -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.15</version>
</dependency>

<!-- Spring与MyBatis整合包 -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.7</version>
</dependency>

<!-- MySQL驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
</dependency>

<!-- HikariCP连接池 -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>4.0.3</version>
</dependency>

<!-- Spring事务依赖 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
    <version>5.3.20</version>
</dependency>

<!-- 日志依赖（可选） -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

### 2. 步骤2：编写核心配置文件

#### （1）Spring配置文件（applicationContext.xml）

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/tx
                           http://www.springframework.org/schema/tx/spring-tx.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 数据源配置 -->
    <bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/test_db?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
        <property name="maximumPoolSize" value="10"/>
        <property name="minimumIdle" value="2"/>
    </bean>

    <!-- SqlSessionFactory配置 -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    </bean>

    <!-- Mapper接口扫描 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.hspedu.mapper"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>

    <!-- 事务管理器 -->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- 开启声明式事务 -->
    <tx:annotation-driven transaction-manager="transactionManager"/>

    <!-- 开启组件扫描 -->
    <context:component-scan base-package="com.hspedu.service"/>
</beans>
```

#### （2）MyBatis核心配置文件（mybatis-config.xml）

```xml
<configuration>
    <!-- 实体类别名 -->
    <typeAliases>
        <package name="com.hspedu.entity"/>
    </typeAliases>

    <!-- 可选配置 -->
    <settings>
        <setting name="logImpl" value="LOG4J"/>
    </settings>
</configuration>
```

### 3. 步骤3：项目目录结构

```
src/main/java
└── com
    └── hspedu
        ├── entity       # 实体类
        │   └── User.java
        ├── mapper       # Mapper接口
        │   └── UserMapper.java
        ├── service      # 业务逻辑层
        │   ├── UserService.java
        │   └── impl
        │       └── UserServiceImpl.java
        └── test
            └── MyBatisSpringTest.java
src/main/resources
├── applicationContext.xml
├── mybatis-config.xml
├── mapper
│   └── UserMapper.xml
└── log4j.properties
```

---

## 三、整合实战开发（笔记22.3节，核心重点）

### 1. 实体类与Mapper接口/映射文件

**User.java**

```java
package com.hspedu.entity;
public class User {
    private Integer id;
    private String username;
    private Integer age;
    private String gender;
    // getter/setter、构造器省略
}
```

**UserMapper.java**

```java
package com.hspedu.mapper;
import com.hspedu.entity.User;
import org.apache.ibatis.annotations.Param;
import java.util.List;

public interface UserMapper {
    User getUserById(Integer id);
    List<User> getAllUsers();
    int addUser(User user);
    int updateUser(User user);
    int deleteUserById(@Param("uid") Integer id);
}
```

**UserMapper.xml**

```xml
<mapper namespace="com.hspedu.mapper.UserMapper">
    <select id="getUserById" resultType="User">
        SELECT id, username, age, gender FROM user WHERE id = #{id}
    </select>
    <select id="getAllUsers" resultType="User">
        SELECT id, username, age, gender FROM user
    </select>
    <insert id="addUser" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO user(username, age, gender) VALUES(#{username}, #{age}, #{gender})
    </insert>
    <update id="updateUser">
        UPDATE user SET username=#{username}, age=#{age}, gender=#{gender} WHERE id=#{id}
    </update>
    <delete id="deleteUserById">
        DELETE FROM user WHERE id=#{uid}
    </delete>
</mapper>
```

### 2. Service层

**UserService.java**

```java
package com.hspedu.service;
import com.hspedu.entity.User;
import java.util.List;

public interface UserService {
    User getUserById(Integer id);
    List<User> getAllUsers();
    int addUser(User user);
    int updateUser(User user);
    int deleteUserById(Integer id);
    void transferTest();
}
```

**UserServiceImpl.java**

```java
package com.hspedu.service.impl;
import com.hspedu.entity.User;
import com.hspedu.mapper.UserMapper;
import com.hspedu.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public User getUserById(Integer id) {
        return userMapper.getUserById(id);
    }

    @Override
    public List<User> getAllUsers() {
        return userMapper.getAllUsers();
    }

    @Override
    @Transactional
    public int addUser(User user) {
        return userMapper.addUser(user);
    }

    @Override
    @Transactional
    public int updateUser(User user) {
        return userMapper.updateUser(user);
    }

    @Override
    @Transactional
    public int deleteUserById(Integer id) {
        return userMapper.deleteUserById(id);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void transferTest() {
        User user1 = new User(1, "张三", 20, "男");
        userMapper.updateUser(user1);
        int i = 1/0; // 模拟异常
        User user2 = new User(2, "李四", 22, "女");
        userMapper.updateUser(user2);
    }
}
```

### 3. 测试类

```java
package com.hspedu.test;
import com.hspedu.entity.User;
import com.hspedu.service.UserService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import java.util.List;

public class MyBatisSpringTest {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        UserService userService = context.getBean("userServiceImpl", UserService.class);

        User user = userService.getUserById(1);
        System.out.println("查询单个用户：" + user);

        List<User> userList = userService.getAllUsers();
        userList.forEach(System.out::println);

        User newUser = new User("王五", 25, "男");
        int addRows = userService.addUser(newUser);
        System.out.println("新增用户ID：" + newUser.getId());

        try {
            userService.transferTest();
        } catch (Exception e) {
            System.out.println("事务测试已回滚");
        }
    }
}
```

---

## 四、整合高级特性（笔记22.4节）

### 1. 声明式事务详解

**@Transactional核心属性**

|属性名|作用|示例|
|---|---|---|
|rollbackFor|指定异常类型回滚|`@Transactional(rollbackFor=Exception.class)`|
|propagation|事务传播行为|`@Transactional(propagation=Propagation.REQUIRED)`|
|isolation|事务隔离级别|`@Transactional(isolation=Isolation.READ_COMMITTED)`|
|timeout|超时时间（秒）|`@Transactional(timeout=30)`|
|readOnly|是否只读事务|`@Transactional(readOnly=true)`|

### 2. 注解式配置替代XML

```java
@Configuration
@ComponentScan("com.hspedu")
@MapperScan("com.hspedu.mapper")
@EnableTransactionManagement
public class SpringMyBatisConfig {

    @Bean
    public DataSource dataSource() { ... }

    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) { ... }

    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

```java
ApplicationContext context = new AnnotationConfigApplicationContext(SpringMyBatisConfig.class);
```

---

## 五、整合避坑点（笔记22.5节核心）

1. **Mapper未被扫描** → MapperScan路径错误。
    
2. **事务不生效** → 非public方法、内部方法调用、注解属性错误。
    
3. **数据源配置错误** → jdbcUrl/用户名/连接池未配置。
    
4. **Mapper XML未加载** → mapperLocations路径与文件目录不匹配。
    
5. **组件扫描范围错误** → @ComponentScan未覆盖Service。
    
6. **版本不兼容** → MyBatis与mybatis-spring版本匹配错误。
    

---

## 六、第22章核心考点总结

1. **整合核心**：MyBatis组件交给Spring IOC管理，通过DI简化开发。
    
2. **实战重点**：声明式事务、组件注入、XML/注解配置方式。
    
3. **企业开发规范**：分层开发、事务管理、连接池优化。
    
4. **避坑关键**：Mapper扫描、事务生效条件、映射文件加载、版本兼容、资源过滤。
    

---

本章内容是企业级Java开发必备技能，Spring与MyBatis整合是后端开发标准架构。下一章将学习“Spring Boot整合MyBatis”，进一步简化配置（自动装配核心组件），需打好本章基础。