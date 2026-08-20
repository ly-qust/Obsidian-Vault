

---

# ☕ Java 注释与注解总结

## 一、Java 中的三种注释类型

Java 提供三种注释方式，用于**代码说明、文档生成和调试辅助**：

|类型|语法|用途|
|---|---|---|
|单行注释|`//`|对一行代码或说明进行简短注释|
|多行注释|`/* ... */`|注释多行内容，常用于说明或临时屏蔽代码|
|文档注释（Javadoc 注释）|`/** ... */`|用于生成 HTML API 文档，描述类、方法、属性等|

---

## 二、文档注释（Javadoc）

### 🧩 1. 基本语法

以 `/**` 开始，以 `*/` 结束。  
位于类、方法、变量等声明之前。

```java
/**
 * 类的说明
 * @author 作者
 * @version 1.0
 */
public class Example {
    /**
     * 方法的说明
     * @param x 参数说明
     * @return 返回值说明
     */
    public int add(int x) {
        return x + 1;
    }
}
```

---

### 🛠 2. 生成文档命令

使用 **javadoc** 工具生成 HTML API 文档：

```bash
javadoc Example.java
```

生成的内容包括：

- 类与包的详细说明
    
- 方法与属性说明
    
- 继承结构与索引
    
- 超链接跳转（通过 @see、@link 等）
    

---

## 三、常用 Javadoc 标签说明

|标签|说明|示例|
|---|---|---|
|`@author`|类作者信息|`@author 李宇`|
|`@version`|版本号|`@version 1.0`|
|`@param`|方法参数说明|`@param num 要平方的值`|
|`@return`|返回值说明|`@return 平方后的结果`|
|`@throws` / `@exception`|抛出的异常类型|`@throws IOException 输入错误`|
|`@see`|链接到相关类/方法|`@see java.io.BufferedReader`|
|`@since`|指定引入版本|`@since JDK 1.5`|
|`@deprecated`|表示方法/类已过时|`@deprecated 请使用 newMethod()`|
|`{@link}`|内嵌链接|`{@link java.util.List}`|
|`{@value}`|显示常量值（static 字段）|`{@value #MAX_VALUE}`|
|`{@inheritDoc}`|继承父类或接口的文档注释|`{@inheritDoc}`|
|`@serial` / `@serialData` / `@serialField`|用于序列化说明|`@serialField name type description`|

> ✅ **注意**：`@return` 不能用于 `void` 方法，否则 javadoc 会发出警告。

---

### 📄 4. 示例：完整的 Javadoc 注释类

```java
import java.io.*;

/**
 * 这个类演示了文档注释的使用。
 * @author A
 * @version 1.2
 * @since JDK 1.5
 */
public class SquareNum {
    /**
     * 返回一个数的平方。
     * @param num 要平方的数。
     * @return 平方值。
     */
    public double square(double num) {
        return num * num;
    }

    /**
     * 从用户输入中读取一个数字。
     * @return 输入的数值。
     * @throws IOException 输入错误时抛出。
     * @see IOException
     */
    public double getNumber() throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        return Double.parseDouble(br.readLine());
    }

    /**
     * 主方法：演示平方功能。
     * @param args 命令行参数。
     * @throws IOException 输入错误。
     */
    public static void main(String[] args) throws IOException {
        SquareNum ob = new SquareNum();
        System.out.println("Enter value to be squared:");
        double val = ob.getNumber();
        System.out.println("Squared value is " + ob.square(val));
    }
}
```

---

## 四、Javadoc 输出内容

执行：

```bash
javadoc SquareNum.java
```

输出内容（在当前目录生成 HTML 文件）：

- `SquareNum.html`：类说明页
    
- `index.html`：文档首页
    
- `package-summary.html`：包摘要
    
- `package-tree.html`：继承结构
    
- `constant-values.html`：常量值列表
    

---

## 五、IDE 自动化注释模板（如 Eclipse / IntelliJ）

很多 IDE 支持 **自动生成 Javadoc 模板**（如 Eclipse 的 JAutodoc 插件）。  
可通过快捷键自动插入标准化注释模板。

### 🧾 文件头模板：

```java
/*
 * <p>项目名称: ${project_name}</p>
 * <p>文件名称: ${file_name}</p>
 * <p>描述: [类型描述]</p>
 * <p>创建时间: ${date}</p>
 * <p>公司信息: *****公司 ****部</p>
 * @author ***
 * @version v1.0
 * @update [序号][日期YYYY-MM-DD] [更改人][变更描述]
 */
```

### 🧱 方法注释模板：

```java
/**
 * @Title: ${enclosing_method}
 * @Description: [功能描述]
 * @param ${tags}
 * @return ${return_type}
 * @author ***
 * @CreateDate: ${date} ${time}
 * @update: [序号][日期YYYY-MM-DD] [更改人][变更描述]
 */
```

### 🔑 Getter / Setter 注释模板：

```java
/**
 * 获取 ${bare_field_name}
 */

/**
 * 设置 ${bare_field_name}
 * @param ${param} ${field}
 */
```

---

## 六、Java 中的注解（Annotation）与注释的区别

|对比项|注释（Comment）|注解（Annotation）|
|---|---|---|
|语法|`//`, `/* */`, `/** */`|`@Override`, `@Deprecated`, `@SuppressWarnings` 等|
|作用时间|编译时忽略|编译时 / 运行时均可生效|
|是否参与编译|❌ 不参与编译|✅ 参与编译与运行逻辑|
|功能|仅用于代码说明|可影响程序行为（如反射、框架注入）|
|处理工具|`javadoc`|由编译器或框架（如 Spring）解析|
|举例|`/** 方法描述 */`|`@Override`, `@Autowired`, `@Test`|

---

## 七、常见内置注解（Annotation）

|注解|作用|
|---|---|
|`@Override`|表示重写父类方法|
|`@Deprecated`|标记方法或类已过时|
|`@SuppressWarnings`|抑制编译器警告|
|`@FunctionalInterface`|限定接口中只能有一个抽象方法|
|`@SafeVarargs`|抑制泛型可变参数的警告|
|`@Retention`、`@Target`|元注解，用于自定义注解时定义作用域与生命周期|

---

## 🧠 八、总结对比速查表

|分类|注释类型|工具支持|主要用途|
|---|---|---|---|
|`//`|单行注释|无|简短说明或临时屏蔽|
|`/* ... */`|多行注释|无|屏蔽大段代码或说明|
|`/** ... */`|文档注释|javadoc|生成 API 文档|
|`@Annotation`|注解（Annotation）|编译器/框架|参与代码逻辑、元数据定义|

---

## ✅ 九、重点记忆

1. **三种注释：** `//`、`/*...*/`、`/**...*/`
    
2. **文档注释只能出现在类、接口、方法、字段声明前。**
    
3. **`javadoc` 工具能自动生成 HTML 文档。**
    
4. **常用标签：** `@param`、`@return`、`@throws`、`@see`、`@version`。
    
5. **注释 ≠ 注解（Annotation）**。  
    注释只说明，注解可影响编译或运行行为。
    
6. **推荐使用 JAutodoc 或 IDE 模板**自动生成标准注释，保证规范性。
    

---



# ☕ Java 注解（Annotation）进阶篇

## 一、什么是注解（Annotation）

注解是 Java 5 引入的一种**元数据机制**，用于向类、方法、字段等添加**额外信息**，可在编译期、类加载期或运行期被解析和使用。

> 注解本身不改变程序逻辑，但可以被工具或框架用来生成代码、做检查或执行特定操作。

### 1. 注解的基本语法

```java
@AnnotationName
@AnnotationName(value = "something")
```

---

## 二、Java 内置注解（Built-in Annotations）

|注解|说明|使用示例|
|---|---|---|
|`@Override`|表示方法重写父类方法|`@Override public String toString() {...}`|
|`@Deprecated`|标记方法或类已过时|`@Deprecated public void oldMethod(){}`|
|`@SuppressWarnings`|抑制编译器警告|`@SuppressWarnings("unchecked")`|
|`@SafeVarargs`|抑制泛型可变参数警告|`@SafeVarargs public final <T> void method(T... args)`|
|`@FunctionalInterface`|表示接口为函数式接口|`@FunctionalInterface interface Func { void apply(); }`|

---

## 三、元注解（Meta-Annotations）

元注解是用于注解其他注解的注解。Java 提供以下元注解：

|元注解|说明|
|---|---|
|`@Retention`|指定注解的生命周期：编译期、类加载期或运行期|
|`@Target`|指定注解可以应用的程序元素：类、方法、字段等|
|`@Documented`|注解是否包含在 Javadoc 中|
|`@Inherited`|子类是否继承父类的注解|
|`@Repeatable`|注解可以重复使用（Java 8 引入）|

### 示例：

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME) // 注解在运行时可用
@Target(ElementType.METHOD)         // 只能用于方法
@Documented
public @interface MyAnnotation {
    String value() default "default";
}
```

---

## 四、注解的生命周期（RetentionPolicy）

`@Retention` 决定注解何时有效：

|RetentionPolicy|生命周期|解析方式|
|---|---|---|
|`SOURCE`|仅在源代码中存在，编译后丢弃|编译器使用|
|`CLASS`|编译后存在 class 文件中，但运行时不可见|类加载器不可见|
|`RUNTIME`|运行时依然存在，可通过反射读取|反射可访问|

---

## 五、注解的作用目标（Target）

`@Target` 用于指定注解能应用到的 Java 元素：

|ElementType|可作用目标|
|---|---|
|`TYPE`|类、接口、枚举|
|`FIELD`|字段|
|`METHOD`|方法|
|`PARAMETER`|方法参数|
|`CONSTRUCTOR`|构造器|
|`LOCAL_VARIABLE`|局部变量|
|`ANNOTATION_TYPE`|注解类型|
|`PACKAGE`|包|

---

## 六、自定义注解（Custom Annotation）

### 1. 定义注解

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @interface Info {
    String author() default "Unknown";
    String date();
    int version() default 1;
}
```

### 2. 使用注解

```java
@Info(author="李宇", date="2025-11-11", version=2)
public class Demo {
    @Info(date="2025-11-11")
    public void testMethod() {}
}
```

---

## 七、注解的反射使用（Runtime Reflection）

```java
import java.lang.reflect.Method;

public class AnnotationDemo {
    public static void main(String[] args) throws Exception {
        Class<Demo> clazz = Demo.class;

        // 类注解
        if (clazz.isAnnotationPresent(Info.class)) {
            Info info = clazz.getAnnotation(Info.class);
            System.out.println("Author: " + info.author());
            System.out.println("Date: " + info.date());
        }

        // 方法注解
        Method method = clazz.getMethod("testMethod");
        if (method.isAnnotationPresent(Info.class)) {
            Info info = method.getAnnotation(Info.class);
            System.out.println("Method Date: " + info.date());
        }
    }
}
```

> 输出：

```
Author: 李宇
Date: 2025-11-11
Method Date: 2025-11-11
```

---

## 八、重复注解（Repeatable Annotation）

Java 8 引入可重复注解：

```java
import java.lang.annotation.*;

@Repeatable(Tags.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Tag {
    String value();
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Tags {
    Tag[] value();
}

class Test {
    @Tag("A")
    @Tag("B")
    public void method() {}
}
```

---

## 九、常见框架中的注解应用

|框架|注解示例|功能说明|
|---|---|---|
|Spring|`@Autowired`, `@Component`, `@Service`, `@Controller`|依赖注入和组件扫描|
|JUnit|`@Test`, `@Before`, `@After`, `@Ignore`|单元测试控制|
|Hibernate / JPA|`@Entity`, `@Table`, `@Id`, `@Column`|ORM 映射数据库表|
|Lombok|`@Getter`, `@Setter`, `@Data`, `@Builder`|自动生成 getter/setter/构造器|

---

## 🔑 十、注解使用注意点

1. **不要滥用注解**：只在必要的地方添加注解，否则增加代码复杂度。
    
2. **运行时反射成本高**：注解解析时会有一定性能开销。
    
3. **元注解很重要**：`@Retention` 决定注解可见性，`@Target` 限定使用范围。
    
4. **重复注解要用 @Repeatable**，否则同一元素只能有一个注解。
    
5. **与泛型配合**：注解可用在泛型类和方法上，但不会影响泛型逻辑本身。
    

---

## 🔥 十一、总结速查

| 类型    | 用途           | 生命周期                 | 工具/解析方式 |
| ----- | ------------ | -------------------- | ------- |
| 内置注解  | 编译器或框架识别     | 编译期/运行期              | 编译器/框架  |
| 自定义注解 | 提供元数据        | SOURCE/CLASS/RUNTIME | 反射或工具   |
| 元注解   | 修饰注解本身       | 编译期/运行期              | 编译器/反射  |
| 重复注解  | Java 8 可重复标注 | RUNTIME              | 反射解析    |

---

