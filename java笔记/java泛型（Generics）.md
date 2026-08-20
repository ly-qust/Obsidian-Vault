

---
# 目录

1. 泛型是什么 & 为什么要用
    
2. 基本语法：泛型类、泛型方法、泛型接口
    
3. 有界类型参数（上界 / 下界）
    
4. 通配符（`?`）与 PECS 原则
    
5. 常见签名与推荐写法（实用小技巧）
    
6. 泛型与类型擦除（Type Erasure）要点
    
7. 常见限制与注意事项（坑）
    
8. 典型示例（代码）
    
9. 常见问题（FAQ）
    
10. 速查表（快速回顾）
    

---

# 1. 泛型是什么 & 为什么要用

- **泛型（Generics）**：在编译时提供类型参数化的机制（JDK 5 引入），本质是 **参数化类型**。
    
- **目的**：
    
    - 提供**编译时类型安全检查**（减少强制类型转换）。
        
    - 提高代码复用性与可读性（同一份代码处理多种类型）。
        
    - 在集合框架中极其重要：`List<T>`, `Map<K,V>` 等。
        

---

# 2. 基本语法

## 泛型类

```java
public class Box<T> {
    private T t;
    public void add(T t) { this.t = t; }
    public T get() { return t; }
}
```

使用：

```java
Box<Integer> bi = new Box<>();
bi.add(10);
Integer v = bi.get();
```

## 泛型方法

- 类型参数声明在返回类型之前：
    

```java
public static <E> void printArray(E[] arr) {
    for (E e : arr) System.out.print(e + " ");
}
```

- 泛型方法可在非泛型类或泛型类中定义。
    

## 泛型接口

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

---

# 3. 有界类型参数（Bounded Type Parameters）

- **上界（extends）**：限制类型必须是某个类型或其子类 / 实现类。
    

```java
public static <T extends Number> void foo(T t) { ... }
```

- 用于需要调用目标类型的特定方法或属性（例如 `compareTo`）。
    
- **更通用的比较签名（推荐）**：
    

```java
public static <T extends Comparable<? super T>> T maximum(T x, T y, T z) { ... }
```

解释：允许 `T` 与其超类型的 `Comparable` 兼容（协变场景更安全）。

---

# 4. 通配符（Wildcard）与 PECS

- `?` 表示未知类型（通配符）。
    
    - `List<?>`：可以接收任何具体类型的 List（只读安全，不能添加任意元素）。
        
- 上限通配符：`List<? extends Number>` —— 可读（取出为 Number 或其子类型），不能安全写入（不能 add 一个 Number）。
    
- 下限通配符：`List<? super Integer>` —— 可写（可以 add Integer），读取得到的是 `Object`（或需类型转换）。
    
- **PECS 原则（Producer Extends, Consumer Super）**：
    
    - 当结构产生（返回）T：使用 `? extends T`。
        
    - 当结构消费（接受）T：使用 `? super T`。
        

示例：

```java
void copy(List<? super T> dest, List<? extends T> src) { ... }
```

---

# 5. 常见签名与推荐写法（实践技巧）

- 推荐比较方法签名：
    
    ```java
    public static <T extends Comparable<? super T>> T maximum(T... items)
    ```
    
- 当需要比较但类型本身不实现 `Comparable`，传入 `Comparator`：
    
    ```java
    public static <T> T max(T a, T b, Comparator<? super T> cmp) { ... }
    ```
    
- 当允许 `null` 时明确处理（`nullsFirst` / `nullsLast` 或手写判空）。
    
- 使用钻石操作符 `<>`（Java 7+）减少冗长类型声明。
    

---

# 6. 泛型与类型擦除（Type Erasure）要点

- Java 泛型通过 **类型擦除** 实现：编译器在编译期检查类型，运行时类型参数被擦除（替换为上界或 `Object`）。
    
- 后果：
    
    - 运行时无法判断泛型实际类型（例如 `List<String>` 与 `List<Integer>` 在运行时都是 `List`）。
        
    - 不能用泛型参数创建数组：`new T[10]` 会报错。
        
    - 不能重载仅由泛型不同的方法（擦除后签名冲突）。
        
    - 反射操作处理泛型需要 `Type`、`ParameterizedType` 等。
        

---

# 7. 常见限制与注意事项（坑）

- 不能在静态上下文直接使用类的类型参数（静态属于类，不是实例）。
    
- 不能实例化类型参数（`new T()` 不可）。
    
- 不能创建泛型数组（`T[] arr = new T[n]` 不可），可用 `@SuppressWarnings("unchecked")` + `Object[]` 或 `List<T>` 代替。
    
- 不能在运行时判断泛型类型（例如 `if (obj instanceof List<String>)` 不可）。
    
- 泛型和原始类型混用会产生不安全警告（避免 raw types）。
    
- `equals` 与 `compareTo` 一致性：若 `compareTo` 为 0，通常 `equals` 也应为 `true`（推荐但非强制）。
    
- 注意 `ClassCastException` 在类型擦除与不当强转时仍可能出现。
    

---

# 8. 典型示例（代码与解释）

## 例 1：泛型方法：查找最大值

```java
public static <T extends Comparable<? super T>> T maximum(T x, T y, T z) {
    T max = x;
    if (y != null && y.compareTo(max) > 0) max = y;
    if (z != null && z.compareTo(max) > 0) max = z;
    return max;
}
```

- 要点：`Comparable<? super T>` 推荐写法；处理 `null` 根据需要。
    

## 例 2：泛型类（Box）

```java
public class Box<T> {
    private T t;
    public void add(T t) { this.t = t; }
    public T get() { return t; }
}
```

## 例 3：PECS 用法（复制）

```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) dest.add(item);
}
```

- `src` 是 producer（extends），`dest` 是 consumer（super）。
    

## 例 4：使用 Comparator 的泛型方法

```java
public static <T> T maximum(T x, T y, T z, Comparator<? super T> cmp) {
    T max = x;
    if (cmp.compare(y, max) > 0) max = y;
    if (cmp.compare(z, max) > 0) max = z;
    return max;
}
```

- 更灵活：不要求 `T` 实现 `Comparable`。
    

---

# 9. 常见问题（FAQ）

**Q1：为什么要写 `<T extends Comparable<? super T>>` 而不是 `<T extends Comparable<T>>`？**  
A：前者更通用，支持 `Comparable` 定义在超类型中的情况（避免编译限制），是协变友好的惯用写法。

**Q2：能把泛型用于基本类型（int、double）吗？**  
A：泛型参数只能是引用类型。使用包装类（`Integer`, `Double`）或 Java 8+ 的原始类型专用方法/流。

**Q3：为什么 `List<Object>` 不能接受 `List<String>`？**  
A：泛型不协变。`List<String>` 不是 `List<Object>` 的子类型。若想兼容读取，可用 `List<?>` 或 `List<? extends Object>`。

**Q4：如何避免类型擦除带来的问题？**  
A：使用显式的 `Class<T>` 参数或 `TypeToken` 模式（Guava），并谨慎使用反射。

---

# 10. 速查表（快速回顾）

|概念|语法|要点|
|---|---|---|
|泛型类|`class Box<T> {}`|T 为类型变量|
|泛型方法|`public static <E> void m(E e)`|类型参数在返回类型前声明|
|有界上界|`<T extends Number>`|限定 T 必须是 Number 或子类|
|比较常用签名|`<T extends Comparable<? super T>>`|推荐用于比较|
|通配符上界|`List<? extends Number>`|可读，不可写|
|通配符下界|`List<? super Integer>`|可写，读出为 Object|
|PECS|Producer Extends, Consumer Super|记住何时用 extends / super|
|类型擦除|—|运行时无泛型信息；注意泛型数组/反射限制|

---

# 小结（一句话）

**泛型的核心是把类型当作参数以增强类型安全性与复用性；掌握有界类型参数、通配符和 PECS，是写出健壮泛型代码的关键。**

---
