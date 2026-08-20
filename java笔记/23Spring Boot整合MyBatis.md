# 第23章 Spring Boot整合MyBatis（主流开发模式）—— 基于韩顺平笔记全重点解析

本章是企业级开发的终极简化方案，韩顺平笔记中系统讲解了**Spring Boot自动配置原理、MyBatis Starter依赖、零XML配置实战、分页插件集成、多数据源配置**等核心内容。Spring Boot通过“约定大于配置”理念，彻底简化Spring与MyBatis的整合流程，无需手动配置`SqlSessionFactory`、事务管理器等组件，是当前后端开发的主流模式。以下严格遵循笔记框架，从“自动配置→环境搭建→实战开发→高级特性→避坑对比”全维度解析，确保覆盖所有重点。

## 一、整合核心原理（笔记23.1节）

### 1. 核心优势（对比传统Spring整合）

- **零XML配置**：Spring Boot通过自动配置（AutoConfiguration）机制，替代传统Spring的`applicationContext.xml`和MyBatis的`mybatis-config.xml`。
    
- **Starter依赖简化**：引入`mybatis-spring-boot-starter`一站式依赖，自动导入MyBatis、Spring JDBC、连接池等核心jar包，无需手动管理版本。
    
- **自动组件装配**：Spring Boot自动创建`DataSource`、`SqlSessionFactory`、`MapperScannerConfigurer`等核心组件，无需手动声明Bean。
    
- **配置集中化**：所有配置（数据库连接、MyBatis参数）统一放在`application.yml`/`application.properties`文件中，维护便捷。
    

### 2. 自动配置核心类（笔记23.1.2节）

Spring Boot通过以下核心类实现MyBatis自动配置（无需手动编写）：

- `MyBatisAutoConfiguration`：自动配置`SqlSessionFactory`、`SqlSessionTemplate`等MyBatis核心组件。
    
- `MyBatisProperties`：绑定`application.yml`中`mybatis`前缀的配置参数（如映射文件路径、别名包）。
    
- `DataSourceAutoConfiguration`：自动配置数据源（默认使用HikariCP连接池）。
    

## 二、整合环境搭建（笔记23.2节，核心基础）

以Spring Boot 2.7.x + MyBatis 3.5.x + MySQL 8.x为例，采用Maven项目搭建，步骤极简。

### 1. 步骤1：创建Spring Boot项目（两种方式）

- **方式1：Spring Initializr（推荐）**：访问[https://start.spring.io/](https://start.spring.io/)，选择：
    
    - 依赖：`MyBatis Framework`、`MySQL Driver`、`Spring Web`（可选，用于接口开发）。
        
    - 打包方式：Jar。
        
    - JDK版本：8/11。
        
- **方式2：手动创建Maven项目**：在`pom.xml`中添加Starter依赖。
    

### 2. 步骤2：导入核心Maven依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 父工程依赖（Spring Boot核心父依赖，统一版本管理） -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.15</version>
        <relativePath/>
    </parent>

    <groupId>com.hspedu</groupId>
    <artifactId>springboot-mybatis-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- 1. Spring Boot Web依赖（可选，用于开发接口） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 2. MyBatis Spring Boot Starter（核心依赖，自动导入MyBatis+Spring整合包） -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.3.2</version>
        </dependency>

        <!-- 3. MySQL驱动依赖（Spring Boot自动管理版本） -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- 4. 测试依赖（可选） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- 打包插件（Spring Boot专属，可生成可执行Jar） -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 3. 步骤3：编写核心配置文件（application.yml）

Spring Boot推荐使用YAML格式配置文件，放在`src/main/resources`目录下，统一配置数据库连接和MyBatis参数：

```yaml
# 服务器配置（可选）
server:
  port: 8080 # 项目启动端口

# 数据库连接配置（Spring Boot自动装配DataSource）
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test_db?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    username: root
    password: 123456
    # HikariCP连接池配置（Spring Boot默认使用，可选自定义参数）
    hikari:
      maximum-pool-size: 10 # 最大连接数
      minimum-idle: 2 # 最小空闲连接数
      connection-timeout: 30000 # 连接超时时间（30秒）

# MyBatis配置（前缀为mybatis，与MyBatisProperties绑定）
mybatis:
  type-aliases-package: com.hspedu.entity # 实体类别名包（简化映射文件）
  mapper-locations: classpath:mapper/*.xml # Mapper映射文件路径
  configuration:
    map-underscore-to-camel-case: true # 开启下划线转驼峰（如user_name→userName）
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl # 打印SQL日志（控制台输出）
```

### 4. 步骤4：编写项目目录结构（企业规范）

```
src/main/java
└── com
    └── hspedu
        ├── SpringBootMybatisDemoApplication.java # 项目启动类（核心）
        ├── entity       # 实体类（与数据库表对应）
        │   └── User.java
        ├── mapper       # Mapper接口（MyBatis操作数据库）
        │   └── UserMapper.java
        ├── service      # 服务层（业务逻辑）
        │   ├── UserService.java       # 接口
        │   └── impl                  # 实现类
        │       └── UserServiceImpl.java
        └── controller   # 控制层（接口开发，可选）
            └── UserController.java
src/main/resources
├── application.yml    # 核心配置文件（唯一配置文件）
└── mapper             # Mapper映射文件
    └── UserMapper.xml
```

### 5. 步骤5：编写项目启动类

启动类是Spring Boot项目入口，需添加`@MapperScan`注解扫描Mapper接口（或在Mapper接口加`@Mapper`）：

```java
package com.hspedu;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// @SpringBootApplication：Spring Boot核心注解，包含组件扫描、自动配置、入口类标识
@SpringBootApplication
// @MapperScan：扫描Mapper接口所在包，自动生成代理对象（替代传统Spring的MapperScannerConfigurer）
@MapperScan("com.hspedu.mapper")
public class SpringBootMybatisDemoApplication {
    public static void main(String[] args) {
        // 启动Spring Boot应用
        SpringApplication.run(SpringBootMybatisDemoApplication.class, args);
    }
}
```

## 三、整合实战开发（笔记23.3节，核心重点）

基于上述配置，实现用户CRUD操作，全程无XML配置，体现Spring Boot的简化优势。

### 1. 步骤1：编写实体类（User.java）

与数据库`user`表对应，包含属性、getter/setter、无参构造器：

```java
package com.hspedu.entity;

import lombok.Data; // 可选，使用Lombok简化getter/setter（需导入lombok依赖）

// @Data：Lombok注解，自动生成getter/setter/toString/无参构造器（简化代码）
@Data
public class User {
    private Integer id;       // 对应表中id（主键自增）
    private String username;  // 对应username
    private Integer age;      // 对应age
    private String gender;    // 对应gender
}
```

**注意**：使用Lombok需添加依赖（可选）：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### 2. 步骤2：编写Mapper接口与映射文件

#### （1）Mapper接口（UserMapper.java）

```java
package com.hspedu.mapper;

import com.hspedu.entity.User;
import org.apache.ibatis.annotations.Param;

import java.util.List;

// 无需加@Mapper注解（启动类已通过@MapperScan扫描）
public interface UserMapper {
    // 根据id查询用户
    User getUserById(Integer id);

    // 查询所有用户
    List<User> getAllUsers();

    // 新增用户
    int addUser(User user);

    // 更新用户
    int updateUser(User user);

    // 删除用户
    int deleteUserById(@Param("uid") Integer id);
}
```

#### （2）Mapper映射文件（UserMapper.xml）

放在`resources/mapper`目录下，与接口同名：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.hspedu.mapper.UserMapper">
    <!-- 查询单个用户（resultType用别名User，对应entity包下的User类） -->
    <select id="getUserById" resultType="User">
        SELECT id, username, age, gender FROM user WHERE id = #{id}
    </select>

    <!-- 查询所有用户 -->
    <select id="getAllUsers" resultType="User">
        SELECT id, username, age, gender FROM user
    </select>

    <!-- 新增用户（开启主键自增，回填id） -->
    <insert id="addUser" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO user(username, age, gender) VALUES(#{username}, #{age}, #{gender})
    </insert>

    <!-- 更新用户 -->
    <update id="updateUser">
        UPDATE user SET username = #{username}, age = #{age}, gender = #{gender} WHERE id = #{id}
    </update>

    <!-- 删除用户 -->
    <delete id="deleteUserById">
        DELETE FROM user WHERE id = #{uid}
    </delete>
</mapper>
```

### 3. 步骤3：编写Service层（业务逻辑）

通过`@Service`标识组件，`@Autowired`注入Mapper接口，`@Transactional`声明事务（Spring Boot自动管理）：

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

    // 自动注入Spring Boot生成的Mapper代理对象
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

    // 声明事务（异常时自动回滚）
    @Override
    @Transactional(rollbackFor = Exception.class)
    public int addUser(User user) {
        return userMapper.addUser(user);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public int updateUser(User user) {
        return userMapper.updateUser(user);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public int deleteUserById(Integer id) {
        return userMapper.deleteUserById(id);
    }
}
```

### 4. 步骤4：编写Controller层（接口开发，可选）

通过`@RestController`开发HTTP接口，供前端调用：

```java
package com.hspedu.controller;

import com.hspedu.entity.User;
import com.hspedu.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

// @RestController：组合注解，等同于@Controller + @ResponseBody（返回JSON数据）
@RestController
@RequestMapping("/user") // 接口统一前缀
public class UserController {

    @Autowired
    private UserService userService;

    // 根据id查询用户：GET请求，路径参数id
    @GetMapping("/{id}")
    public User getUserById(@PathVariable Integer id) {
        return userService.getUserById(id);
    }

    // 查询所有用户：GET请求
    @GetMapping("/list")
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }

    // 新增用户：POST请求，请求体为JSON
    @PostMapping
    public String addUser(@RequestBody User user) {
        int rows = userService.addUser(user);
        return rows > 0 ? "新增成功，用户ID：" + user.getId() : "新增失败";
    }

    // 更新用户：PUT请求，路径参数id
    @PutMapping("/{id}")
    public String updateUser(@PathVariable Integer id, @RequestBody User user) {
        user.setId(id);
        int rows = userService.updateUser(user);
        return rows > 0 ? "更新成功" : "更新失败";
    }

    // 删除用户：DELETE请求，路径参数id
    @DeleteMapping("/{id}")
    public String deleteUserById(@PathVariable Integer id) {
        int rows = userService.deleteUserById(id);
        return rows > 0 ? "删除成功" : "删除失败";
    }
}
```

### 5. 步骤5：测试（两种方式）

#### （1）接口测试（Postman/浏览器）

启动项目后，访问接口测试：

- 查询用户：`http://localhost:8080/user/1`（GET）
    
- 新增用户：`http://localhost:8080/user`（POST，请求体JSON：`{"username":"赵六","age":28,"gender":"男"}`）
    

#### （2）单元测试（Test类）

在`src/test/java`目录下编写测试类：

```java
package com.hspedu;

import com.hspedu.entity.User;
import com.hspedu.service.UserService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

// @SpringBootTest：Spring Boot测试注解，加载完整上下文
@SpringBootTest
public class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    public void testGetUserById() {
        User user = userService.getUserById(1);
        System.out.println("查询结果：" + user);
    }

    @Test
    public void testGetAllUsers() {
        List<User> userList = userService.getAllUsers();
        userList.forEach(System.out::println);
    }
}
```

## 四、整合高级特性（笔记23.4节）

### 1. 分页插件集成（PageHelper）

Spring Boot整合PageHelper简化分页操作，步骤如下：

#### （1）导入PageHelper Starter依赖

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.4.6</version>
</dependency>
```

#### （2）Service层添加分页方法

```java
// UserService接口
List<User> getUserByPage(int pageNum, int pageSize);

// UserServiceImpl实现类
@Override
public List<User> getUserByPage(int pageNum, int pageSize) {
    // PageHelper.startPage：分页拦截，必须放在查询方法前
    PageHelper.startPage(pageNum, pageSize);
    return userMapper.getAllUsers(); // 自动分页
}
```

#### （3）测试分页

```java
@Test
public void testGetUserByPage() {
    int pageNum = 1; // 第1页
    int pageSize = 2; // 每页2条数据
    List<User> userList = userService.getUserByPage(pageNum, pageSize);
    // 转换为Page对象，获取分页信息
    Page<User> page = (Page<User>) userList;
    System.out.println("总条数："
```

- page.getTotal());  
    System.out.println("总页数：" + page.getPages());  
    System.out.println("当前页数据：" + userList);  
    }
    

````

### 2. 多数据源配置（实战高频）
实际开发中可能需要操作多个数据库，Spring Boot通过配置多数据源实现：
#### （1）修改application.yml配置
```yaml
spring:
  datasource:
    # 主数据源（默认）
    master:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/test_db?useSSL=false&serverTimezone=UTC
      username: root
      password: 123456
    # 从数据源（第二个数据库）
    slave:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/test_db2?useSSL=false&serverTimezone=UTC
      username: root
      password: 123456
````

#### （2）编写多数据源配置类

```java
// 主数据源配置
@Configuration
@MapperScan(basePackages = "com.hspedu.mapper.master", sqlSessionFactoryRef = "masterSqlSessionFactory")
public class MasterDataSourceConfig {

    @Bean(name = "masterDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "masterSqlSessionFactory")
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/master/*.xml"));
        return sessionFactory.getObject();
    }
}

// 从数据源配置（类似主数据源，扫描slave包下的Mapper）
@Configuration
@MapperScan(basePackages = "com.hspedu.mapper.slave", sqlSessionFactoryRef = "slaveSqlSessionFactory")
public class SlaveDataSourceConfig {
    // 代码省略，与主数据源类似，关联spring.datasource.slave配置
}
```

### 3. 注解式SQL（无XML）

简单SQL可直接用MyBatis注解替代XML，简化开发：

```java
package com.hspedu.mapper;

import com.hspedu.entity.User;
import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Select;

@Mapper // 单个Mapper接口加@Mapper，无需启动类@MapperScan
public interface UserAnnotationMapper {
    // 注解式查询
    @Select("SELECT id, username, age, gender FROM user WHERE id = #{id}")
    User getUserById(Integer id);

    // 注解式删除
    @Delete("DELETE FROM user WHERE id = #{id}")
    int deleteUserById(Integer id);
}
```

## 五、整合避坑点（笔记23.5节核心）

1. **Mapper接口扫描失败**：
    
    - 未在启动类加`@MapperScan("com.hspedu.mapper")`，或包路径错误。
        
    - 单个Mapper接口未加`@Mapper`注解（且未配置扫描），导致Spring无法生成代理对象。
        
2. **映射文件未加载**：
    
    - `application.yml`中`mybatis.mapper-locations`配置错误（如路径写成`classpath:mappers/*.xml`，实际目录是`mapper`）。
        
    - Maven项目中，Mapper.xml文件放在`src/main/java`目录下，未配置资源过滤（需在pom.xml中添加资源过滤规则）。
        
3. **分页插件不生效**：
    
    - 未导入`pagehelper-spring-boot-starter`依赖，或版本与Spring Boot不兼容。
        
    - `PageHelper.startPage(pageNum, pageSize)`未放在查询方法前（必须是查询前的第一行代码）。
        
4. **事务不生效**：
    
    - 未添加`spring-boot-starter-web`或`spring-boot-starter-jdbc`依赖（事务管理器依赖JDBC包）。
        
    - `@Transactional`注解加在非public方法上（Spring事务仅对public方法生效）。
        
5. **配置文件格式错误**：
    
    - YAML文件缩进错误（如`spring`与`datasource`未对齐），导致配置未生效。
        
    - 数据库URL参数缺失（如未加`serverTimezone=UTC`，抛时区异常）。
        
6. **依赖冲突**：
    
    - 手动导入MyBatis、Spring JDBC等依赖，与`mybatis-spring-boot-starter`版本冲突（建议仅保留Starter依赖）。
        
7. **Lombok注解不生效**：
    
    - 未安装IDE的Lombok插件（如IDEA需在Plugins中搜索Lombok安装），导致getter/setter未生成。
        

## 六、第23章核心考点总结（韩顺平笔记重点提炼）

1. **整合核心**：
    
    - 核心思想：Spring Boot自动配置替代手动XML，Starter依赖简化依赖管理。
        
    - 关键配置：`application.yml`（数据库连接+MyBatis参数）、`@SpringBootApplication`（启动类）、`@MapperScan`（Mapper扫描）。
        
2. **实战重点**：
    
    - 分层开发：entity→mapper→service→controller，符合企业规范。
        
    - 高级特性：分页插件（PageHelper）、多数据源、注解式SQL。
        
    - 接口开发：`@RestController`+HTTP方法注解（`@GetMapping`/`@PostMapping`）。
        
3. **企业开发规范**：
    
    - 配置集中化：所有参数放在`application.yml`，避免硬编码。
        
    - 事务管理：增删改操作必须加`@Transactional(rollbackFor = Exception.class)`。
        
    - 日志配置：开启MyBatis SQL日志，便于调试。
        
4. **避坑关键**：
    
    - Mapper扫描、映射文件路径、分页插件顺序、事务注解位置、YAML格式。
        
