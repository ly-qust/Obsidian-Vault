![[Pasted image 20251111203405.png]]
下面给你 **一份非常清晰、分类明确、可直接做课堂笔记的《Java 反射常用 API 全总结》** —— 全是你在写反射代码时最常用、最应该掌握的 API，包含说明 + 示例（不啰嗦，便于快速理解）。

---

# 📘 Java 反射常用 API 总结（最实用整理版）

---

## 1️⃣ **Class 类获取方式（3 种）**

|方式|代码示例|说明|
|---|---|---|
|**1. 类名.class**|`Class<?> c = Person.class;`|最常见、最安全，编译期已知类|
|**2. 对象.getClass()**|`Class<?> c = obj.getClass();`|运行期根据对象获取|
|**3. Class.forName()**|`Class<?> c = Class.forName("com.demo.Person");`|适用于读取配置、框架动态加载类|

✔ 对象反射的第一步永远是：**拿到 Class 对象**

---

## 2️⃣ **获取构造方法 Constructor**

### ✔ 常用 API

|API|解释|
|---|---|
|`getConstructors()`|获取所有 **public** 构造方法|
|`getDeclaredConstructors()`|获取所有构造（包含 private）|
|`getConstructor(Class...)`|获取某一个 public 构造|
|`getDeclaredConstructor(Class...)`|获取任意构造（可 private）|
|`newInstance()`（过时）|已废弃，推荐用 Constructor 实例化|
|`constructor.newInstance(args)`|真正推荐的实例化方式|

### ✔ 示例

```java
Class<?> c = Person.class;

// 获取无参构造
Constructor<?> cons = c.getDeclaredConstructor();
Object obj = cons.newInstance();

// 获取有参构造
Constructor<?> cons2 = c.getDeclaredConstructor(String.class, int.class);
cons2.setAccessible(true);  // private 时必须打开
Object obj2 = cons2.newInstance("Tom", 18);
```

---

## 3️⃣ **获取字段 Field（属性）**

### ✔ 常用 API

|API|作用|
|---|---|
|`getFields()`|获取所有 **public** 字段（包含父类）|
|`getDeclaredFields()`|获取所有字段（不含父类，可 private）|
|`getField(name)`|获取单个 public 字段|
|`getDeclaredField(name)`|获取单个任意字段（可 private）|
|`field.setAccessible(true)`|访问 private 必须打开|
|`field.get(obj)`|获取字段值|
|`field.set(obj, value)`|设置字段值|

### ✔ 示例

```java
Class<?> c = Person.class;
Field name = c.getDeclaredField("name");
name.setAccessible(true);

Person p = new Person();
name.set(p, "张三");
System.out.println(name.get(p));
```

---

## 4️⃣ **获取方法 Method**

### ✔ 常用 API

|API|用途|
|---|---|
|`getMethods()`|所有 public 方法（包含继承）|
|`getDeclaredMethods()`|所有方法（可 private，不含父类）|
|`getMethod(name, Class...)`|获取某个 public 方法|
|`getDeclaredMethod(name, Class...)`|获取任意方法，包括 private|
|`method.invoke(obj, args)`|调用方法|
|`method.setAccessible(true)`|访问 private 方法必备|

### ✔ 示例

```java
Class<?> c = Person.class;

Method m = c.getDeclaredMethod("say", String.class);
m.setAccessible(true);

Person p = new Person();
m.invoke(p, "你好");
```

---

## 5️⃣ **类信息相关 API**

这些用来“分析类结构”，框架很常用：

|API|说明|
|---|---|
|`c.getName()`|类全名：包名 + 类名|
|`c.getSimpleName()`|类名（不含包名）|
|`c.getPackageName()`|包名|
|`c.getSuperclass()`|父类|
|`c.getInterfaces()`|实现的接口|
|`c.isInterface()`|是否是接口|
|`c.isEnum()`|是否是枚举|
|`c.isAnnotation()`|是否是注解类|
|`c.isPrimitive()`|是否是基本数据类型|
|`c.getModifiers()`|修饰符（public/private）|

---

## 6️⃣ **注解 Annotation 相关 API**

|API|解释|
|---|---|
|`isAnnotationPresent(Class)`|是否存在某注解|
|`getAnnotation(Class)`|获取某注解实例|
|`getAnnotations()`|获取所有注解|
|`getDeclaredAnnotations()`|获取本类定义的注解（不继承）|

### ✔ 示例

```java
if(c.isAnnotationPresent(MyAnno.class)){
    MyAnno a = c.getAnnotation(MyAnno.class);
    System.out.println(a.value());
}
```

---

## 7️⃣ **Modifiers（修饰符）解析工具**

```java
Modifier.isPublic()
Modifier.isPrivate()
Modifier.isStatic()
Modifier.isFinal()
```

### 示例

```java
int mod = field.getModifiers();
if (Modifier.isPrivate(mod)) {
    System.out.println("字段是 private");
}
```

---

# 📌 重点记忆图（最实用精华版）

### ▶ 获取 Class

✔ `Person.class`  
✔ `obj.getClass()`  
✔ `Class.forName()`

### ▶ Constructor

✔ `getDeclaredConstructor()`  
✔ `newInstance()`

### ▶ Field

✔ `getDeclaredField()`  
✔ `field.setAccessible(true)`  
✔ `field.set(obj, value)`

### ▶ Method

✔ `getDeclaredMethod()`  
✔ `invoke(obj, args)`

### ▶ 注解

✔ `isAnnotationPresent()`  
✔ `getAnnotation()`

---

# 🧠 教学建议（你要给小学生讲可以用）

- 反射的核心是：**程序运行时还可以查看、创建、修改对象**
    
- 框架（Spring、MyBatis）大量使用反射
    
- 需要重点掌握：  
    **Class → Constructor → Field → Method → Annotation**
    

---

如果你需要，我还可以帮你：

✅ 整理成 **PPT 结构**  
✅ 做成 **复习思维导图**  
✅ 根据这些 API 出 **10 个循序渐进练习题**（你之前要求过）  
随便说一声即可！