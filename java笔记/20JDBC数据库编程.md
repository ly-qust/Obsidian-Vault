
---

# 第20章 JDBC数据库编程（Java Database Connectivity）—— 基于韩顺平笔记全重点解析

本章是Java操作数据库的核心，韩顺平笔记中系统讲解了**JDBC的核心原理、数据库连接步骤、CRUD操作（增删改查）、PreparedStatement防SQL注入、事务管理、连接池**等实战内容，是开发后端系统（如管理系统、接口服务）的必备技能。以下严格遵循笔记框架，从“基础原理→核心API→实战案例→性能优化→避坑对比”全维度解析，确保覆盖所有重点。

---

## 一、JDBC核心概念（笔记20.1节）

### 1. 什么是JDBC（笔记20.1.1节）

- **定义**：JDBC是Java提供的一套用于操作关系型数据库（如MySQL、Oracle、SQL Server）的标准API，通过统一的接口屏蔽不同数据库的底层差异，实现“一次编写，多数据库兼容”。
    
- **核心作用**：Java程序通过JDBC连接数据库，执行SQL语句（查询、插入、更新、删除），并处理返回结果（如查询结果集）。
    
- **底层原理**：
    
    1. JDBC提供核心接口（如`Connection`、`Statement`、`ResultSet`）。
        
    2. 数据库厂商提供实现这些接口的驱动（如MySQL驱动`mysql-connector-java`）。
        
    3. Java程序通过JDBC接口调用驱动，间接操作数据库。
        

### 2. JDBC核心组件（笔记20.1.2节，高频考点）

JDBC API主要包含以下核心接口/类（均位于`java.sql`或`javax.sql`包）：

|组件|作用|
|---|---|
|`DriverManager`|驱动管理类，用于注册驱动、获取数据库连接（`getConnection()`）。|
|`Connection`|数据库连接对象，代表Java程序与数据库的连接，可创建执行SQL的对象。|
|`Statement`|SQL执行对象，用于执行静态SQL语句（无参数），存在SQL注入风险。|
|`PreparedStatement`|预处理SQL对象，用于执行带参数的SQL语句（防SQL注入，推荐使用）。|
|`ResultSet`|结果集对象，存储查询语句（`SELECT`）返回的数据，可遍历读取。|
|`SQLException`|JDBC操作抛出的异常（编译时异常，必须捕获或声明`throws`）。|

### 3. JDBC操作数据库的核心流程（笔记20.1.3节，重中之重）

所有JDBC操作都遵循以下6步流程，是实战开发的基础：

1. 导入JDBC驱动（如MySQL驱动jar包）。
    
2. 注册数据库驱动（JDK6+可省略，自动加载驱动）。
    
3. 通过`DriverManager`获取数据库连接（`Connection`）。
    
4. 创建`Statement`/`PreparedStatement`对象，编写SQL语句。
    
5. 执行SQL语句，处理结果（查询返回`ResultSet`，增删改返回影响行数）。
    
6. 关闭资源（按`ResultSet`→`Statement`→`Connection`顺序关闭，避免资源泄露）。
    

---

## 二、JDBC环境准备（笔记20.2节）

### 1. 核心准备工作

#### （1）导入数据库驱动

- 以MySQL为例，需导入MySQL驱动jar包（如`mysql-connector-java-8.0.30.jar`）：
    
    - 手动导入：将jar包复制到项目`lib`目录，右键“Add as Library”。
        
    - Maven项目：在`pom.xml`中添加依赖：
        
        ```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.30</version>
        </dependency>
        ```
        

#### （2）数据库连接参数（笔记20.2.2节）

连接数据库需指定4个核心参数，以MySQL为例：

- 驱动类名：MySQL8.0+为`com.mysql.cj.jdbc.Driver`（5.7及以下为`com.mysql.jdbc.Driver`）。
    
- 连接URL：
    
    ```
    jdbc:mysql://localhost:3306/数据库名?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8
    ```
    
    - `localhost`：数据库服务器地址（本地为`localhost`，远程为IP）。
        
    - `3306`：MySQL默认端口。
        
    - `数据库名`：需连接的具体数据库名称（需提前创建）。
        
    - 后缀参数：`useSSL=false`（关闭SSL）、`serverTimezone=UTC`（设置时区）、`characterEncoding=UTF-8`。
        
- 用户名：数据库登录账号（如`root`）。
    
- 密码：数据库登录密码（如`123456`）。
    

### 2. 实战：获取数据库连接（笔记20.2.3节）

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

// JDBC获取数据库连接
public class JDBCConnectionDemo {
    private static final String DRIVER = "com.mysql.cj.jdbc.Driver";
    private static final String URL = "jdbc:mysql://localhost:3306/test_db?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8";
    private static final String USER = "root";
    private static final String PASSWORD = "123456";

    public static Connection getConnection() {
        Connection conn = null;
        try {
            Class.forName(DRIVER); // 注册驱动
            conn = DriverManager.getConnection(URL, USER, PASSWORD);
            System.out.println("数据库连接成功！");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            System.out.println("驱动类加载失败！");
        } catch (SQLException e) {
            e.printStackTrace();
            System.out.println("数据库连接失败！");
        }
        return conn;
    }

    public static void main(String[] args) {
        getConnection();
    }
}
```

---

## 三、JDBC CRUD操作实战（笔记20.3节，核心重点）

以MySQL数据库`test_db`中的`user`表为例（表结构：`id`(INT,主键自增)、`username`(VARCHAR)、`age`(INT)、`gender`(VARCHAR)`），实现增删改查操作。

### 1. 新增操作（INSERT）

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class JDBCInsertDemo {
    public static void main(String[] args) {
        Connection conn = null;
        PreparedStatement pstmt = null;

        try {
            conn = JDBCConnectionDemo.getConnection();
            String sql = "INSERT INTO user(username, age, gender) VALUES(?, ?, ?)";
            pstmt = conn.prepareStatement(sql);

            pstmt.setString(1, "张三");
            pstmt.setInt(2, 20);
            pstmt.setString(3, "男");

            int rows = pstmt.executeUpdate();
            System.out.println("新增成功，影响行数：" + rows);

        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            try { if (pstmt != null) pstmt.close(); if (conn != null) conn.close(); } 
            catch (SQLException e) { e.printStackTrace(); }
        }
    }
}
```

### 2. 查询操作（SELECT）

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class JDBCSelectDemo {
    public static void main(String[] args) {
        Connection conn = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try {
            conn = JDBCConnectionDemo.getConnection();
            String sql = "SELECT id, username, age, gender FROM user WHERE id = ?";
            pstmt = conn.prepareStatement(sql);
            pstmt.setInt(1, 1);
            rs = pstmt.executeQuery();

            while (rs.next()) {
                int id = rs.getInt("id");
                String username = rs.getString("username");
                int age = rs.getInt("age");
                String gender = rs.getString("gender");

                System.out.println("用户信息：id=" + id + ", username=" + username + ", age=" + age + ", gender=" + gender);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            try { if (rs != null) rs.close(); if (pstmt != null) pstmt.close(); if (conn != null) conn.close(); }
            catch (SQLException e) { e.printStackTrace(); }
        }
    }
}
```

### 3. 更新操作（UPDATE）

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class JDBCUpdateDemo {
    public static void main(String[] args) {
        Connection conn = null;
        PreparedStatement pstmt = null;

        try {
            conn = JDBCConnectionDemo.getConnection();
            String sql = "UPDATE user SET age = ? WHERE id = ?";
            pstmt = conn.prepareStatement(sql);
            pstmt.setInt(1, 22);
            pstmt.setInt(2, 1);

            int rows = pstmt.executeUpdate();
            System.out.println("更新成功，影响行数：" + rows);

        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            try { if (pstmt != null) pstmt.close(); if (conn != null) conn.close(); }
            catch (SQLException e) { e.printStackTrace(); }
        }
    }
}
```

### 4. 删除操作（DELETE）

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;

public class JDBCDeleteDemo {
    public static void main(String[] args) {
        Connection conn = null;
        PreparedStatement pstmt = null;

        try {
            conn = JDBCConnectionDemo.getConnection();
            String sql = "DELETE FROM user WHERE id = ?";
            pstmt = conn.prepareStatement(sql);
            pstmt.setInt(1, 1);

            int rows = pstmt.executeUpdate();
            System.out.println("删除成功，影响行数：" + rows);

        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            try { if (pstmt != null) pstmt.close(); if (conn != null) conn.close(); }
            catch (SQLException e) { e.printStackTrace(); }
        }
    }
}
```

---

## 四、JDBC核心进阶（笔记20.4节）

### 1. PreparedStatement vs Statement（笔记20.4.1节，高频考点）

|对比维度|Statement|PreparedStatement|
|---|---|---|
|SQL执行方式|静态SQL，直接拼接参数|预处理SQL，参数用`?`占位，动态设置参数|
|SQL注入风险|有|无|
|性能|多次执行相同SQL时性能低|多次执行相同SQL时性能高|
|代码可读性|差|好|
|推荐场景|无参数的静态SQL|带参数的SQL（开发首选）|

#### SQL注入演示

```java
// 恶意输入：username = "' OR '1'='1";
String sql = "SELECT * FROM user WHERE username='" + username + "' AND password='" + password + "'";
```

### 2. JDBC事务管理（笔记20.4.2节）

- **事务操作核心方法**：
    
    - `conn.setAutoCommit(false)`：关闭自动提交
        
    - `conn.commit()`：提交事务
        
    - `conn.rollback()`：回滚事务
        

#### 转账事务案例

```java
// 省略重复代码，参考原笔记20.4.2节完整示例
```

### 3. 数据库连接池（笔记20.4.3节）

- **核心问题**：频繁创建/关闭Connection消耗资源
    
- **连接池原理**：提前创建连接，使用完归还，复用连接
    
- **常用连接池**：C3P0、DBCP、HikariCP（推荐HikariCP）
    

#### HikariCP示例

```java
// 省略重复代码，参考原笔记20.4.3节完整示例
```

---

## 五、JDBC避坑点（笔记20.5节核心）

1. 驱动类名错误
    
2. 连接URL参数缺失
    
3. 资源关闭顺序错误
    
4. 事务未关闭自动提交
    
5. SQL注入风险
    
6. 连接池参数配置不合理
    
7. 结果集遍历错误
    
8. 数据库账号密码错误
    
9. SQL语法错误
    
10. 连接未归还（连接池场景）
    

---

## 六、第20章核心考点总结

1. **JDBC基础**：加载驱动→获取连接→执行SQL→处理结果→关闭资源
    
2. **核心操作**：CRUD操作、事务管理、防SQL注入
    
3. **性能优化**：连接池、预处理SQL
    
4. **避坑关键**：驱动配置、URL参数、资源关闭、事务管理、SQL注入、连接池参数
    

---
