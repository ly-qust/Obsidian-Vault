
---

# 🧩 Java 集合框架总结

## 一、集合框架的设计目标

- **高性能**：基础集合（如数组、链表、树、哈希表）需高效实现。
    
- **统一性**：不同类型集合以相似方式操作。
    
- **可扩展性**：方便扩展和适应新集合类型。
    

---

## 二、集合框架的组成

|组成部分|描述|
|---|---|
|**接口（Interfaces）**|定义集合的抽象数据类型，如 `Collection`, `List`, `Set`, `Map`。|
|**实现类（Implementations）**|接口的具体实现，如 `ArrayList`, `HashSet`, `HashMap`。|
|**算法（Algorithms）**|操作集合的算法，如排序、搜索（`Collections` 类提供）。|

---

## 三、集合框架体系结构

```
                 ┌─────────────────────┐
                 │     Iterable<E>     │
                 └─────────┬───────────┘
                           │
                     Collection<E>
             ┌─────────────┼──────────────┐
             │              │              │
           List<E>         Set<E>         Queue<E>
             │              │               │
 ┌───────────┼───────────┐  │         ┌────┴─────┐
 │           │           │  │         │          │
ArrayList  LinkedList   Vector  HashSet   LinkedHashSet  TreeSet
 │                         │
 Stack                └─────────────┐
                                   │
                                SortedSet<E>


Map<K,V>
│
├── HashMap
│    ├── LinkedHashMap
│    └── WeakHashMap
│
├── TreeMap
├── Hashtable
│    └── Properties
└── IdentityHashMap
```

---

## 四、主要接口简介

| 接口             | 特点                      |
| -------------- | ----------------------- |
| **Collection** | 最基本集合接口，存储一组对象（无序、不唯一）。 |
| **List**       | 有序、可重复、支持索引访问。          |
| **Set**        | 无序、唯一。                  |
| **SortedSet**  | 有序的 Set。                |
| **Queue**      | 按先进先出（FIFO）规则存储元素。      |
| **Map**        | 键值对存储，不允许重复键。           |
| **SortedMap**  | 按键升序排列的 Map。            |

---

## 五、Set 与 List 的区别

|特点|Set|List|
|---|---|---|
|元素顺序|无序|有序（插入顺序）|
|是否允许重复|否|是|
|查找效率|较低|较高（按索引）|
|插入/删除效率|高|低（会移动元素）|
|常用实现类|`HashSet`, `TreeSet`|`ArrayList`, `LinkedList`, `Vector`|

---

## 六、常见集合实现类

|类名|特点|
|---|---|
|**ArrayList**|动态数组实现，随机访问快，插入删除慢。|
|**LinkedList**|链表实现，插入删除快，查找慢。|
|**HashSet**|基于哈希表，不重复，无序。|
|**LinkedHashSet**|保留插入顺序。|
|**TreeSet**|自动排序（自然顺序或比较器）。|
|**HashMap**|键值对存储，访问速度快，允许一个 `null` 键。|
|**LinkedHashMap**|按插入或访问顺序排序。|
|**TreeMap**|键自动排序。|
|**Hashtable**|线程安全版 `HashMap`（已过时）。|
|**Vector**|线程安全版 `ArrayList`。|
|**Stack**|继承自 `Vector`，实现后进先出（LIFO）。|
|**Properties**|键值均为字符串，常用于配置文件。|
|**BitSet**|位向量实现，用于布尔值存储。|

---

## 七、集合算法（`Collections` 工具类）

|类型|示例|
|---|---|
|排序|`Collections.sort(list)`|
|搜索|`Collections.binarySearch(list, key)`|
|复制|`Collections.copy(dest, src)`|
|反转|`Collections.reverse(list)`|
|不可变集合|`Collections.unmodifiableList(list)`|

> 常量集合：`Collections.EMPTY_SET`, `EMPTY_LIST`, `EMPTY_MAP`

---

## 八、集合遍历方式

### 遍历 List

```java
List<String> list = new ArrayList<>();
list.add("Hello");
list.add("World");

// 1️⃣ 增强 for 循环
for (String str : list)
    System.out.println(str);

// 2️⃣ 转数组
String[] arr = list.toArray(new String[0]);
for (String s : arr)
    System.out.println(s);

// 3️⃣ 使用迭代器
Iterator<String> it = list.iterator();
while (it.hasNext())
    System.out.println(it.next());
```

---

### 遍历 Map

```java
Map<String, String> map = new HashMap<>();
map.put("1", "value1");
map.put("2", "value2");

// 1️⃣ 通过 keySet
for (String key : map.keySet())
    System.out.println("key=" + key + ", value=" + map.get(key));

// 2️⃣ 使用 Iterator + entrySet
Iterator<Map.Entry<String, String>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<String, String> entry = it.next();
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// 3️⃣ 推荐：增强 for
for (Map.Entry<String, String> entry : map.entrySet())
    System.out.println(entry.getKey() + " = " + entry.getValue());

// 4️⃣ 遍历所有 value
for (String v : map.values())
    System.out.println("value=" + v);
```

---

## 九、比较器 Comparator

```java
TreeSet<Integer> set = new TreeSet<>((a, b) -> b - a); // 降序排序
```

- **功能**：定义自定义排序规则。
    
- **方法**：`int compare(T o1, T o2)`
    
- **应用**：用于 `TreeSet`, `TreeMap`, 或 `Collections.sort()`。
    

---

## 🔟 总结

- **Collection** 存储单个元素；**Map** 存储键值对。
    
- 统一接口 + 多种实现 + 实用算法 = **强大且灵活的框架**。
    
- 所有类与接口均在 `java.util` 包中。
    
- 使用 **泛型** 可避免类型转换问题。
    


### 示例
```java
// 导入 Java 集合框架相关类  
import java.util.*;  
  
/**  
 * Java 集合框架综合演示  
 * 演示 List、Set、Map 的创建、遍历、排序、去重等操作  
 */  
public class Main {  
  
    public static void main(String[] args) {  
  
        System.out.println("========== 1️⃣ List 示例 ==========");  
        // 1️⃣ 创建一个 List（ArrayList 实现类）  
        // ArrayList 是基于动态数组的数据结构，支持随机访问，并允许重复元素  
        List<String> names = new ArrayList<>();  
        names.add("Alice");     // 添加字符串元素  
        names.add("Bob");  
        names.add("Charlie");  
        names.add("Alice");     // 允许重复元素  
        System.out.println("原始 List：" + names);  
  
        // 按索引访问列表中的元素（从0开始）  
        System.out.println("第二个元素：" + names.get(1));  
  
        // 使用增强 for 循环遍历 List 中的所有元素  
        System.out.println("使用增强 for 遍历：");  
        for (String name : names) {  
            System.out.println(name);  
        }  
  
        // 使用 Iterator 迭代器遍历并移除重复项  
        System.out.println("使用 Iterator 遍历并删除重复项：");  
        Iterator<String> it = names.iterator();   // 获取迭代器  
        Set<String> seen = new HashSet<>();       // 用于记录已出现过的元素  
        while (it.hasNext()) {// 判断是否还有下一个元素  
            String n = it.next(); // 获取下一个元素  
            if (!seen.add(n)) {                   // 若 add 返回 false 表示该元素已经存在  
                it.remove();                      // 移除当前元素  
            }  
        }  
        System.out.println("去重后的 List：" + names);  
  
        // 使用 Collections 工具类对 List 进行自然顺序排序（字典序）  
        Collections.sort(names);  
        System.out.println("排序后的 List：" + names);  
  
  
        System.out.println("\n========== 2️⃣ Set 示例 ==========");  
        // 2️⃣ 创建一个 Set（HashSet 实现类）  
        // HashSet 不允许重复元素，不保证插入顺序  
        Set<Integer> numbers = new HashSet<>();  
        numbers.add(30);  
        numbers.add(10);  
        numbers.add(20);  
        numbers.add(10);                          // 重复值不会被加入  
  
        System.out.println("HashSet 无序且唯一：" + numbers);  
  
        // 改用 TreeSet（自动排序）  
        // TreeSet 基于红黑树实现，默认会对元素进行升序排列  
        Set<Integer> sortedNumbers = new TreeSet<>(numbers);  
        System.out.println("TreeSet 自动排序：" + sortedNumbers);  
  
  
        System.out.println("\n========== 3️⃣ Map 示例 ==========");  
        // 3️⃣ 创建一个 Map（HashMap 实现类）  
        // HashMap 存储键值对，key 不可重复，value 可重复  
        Map<String, Integer> scoreMap = new HashMap<>();  
        scoreMap.put("Alice", 95);               // 插入键值对  
        scoreMap.put("Bob", 82);  
        scoreMap.put("Charlie", 90);  
  
        System.out.println("原始 Map：" + scoreMap);  
  
        // 遍历方式 1：通过 keySet 获取所有 key 并输出对应的 value        System.out.println("通过 keySet 遍历：");  
        for (String key : scoreMap.keySet()) {  
            System.out.println(key + " => " + scoreMap.get(key));  
        }  
  
        // 遍历方式 2：通过 entrySet 直接获取键值对（推荐方式）  
        System.out.println("通过 entrySet 遍历：");  
        for (Map.Entry<String, Integer> entry : scoreMap.entrySet()) {  
            System.out.println(entry.getKey() + " => " + entry.getValue());  
        }  
  
        // 按 value 排序 Map（使用 List + Comparator）  
        System.out.println("按分数排序后的 Map：");  
        List<Map.Entry<String, Integer>> list = new ArrayList<>(scoreMap.entrySet());  
        list.sort((e1, e2) -> e2.getValue() - e1.getValue()); // 使用 Lambda 表达式按 value 降序排序  
        for (Map.Entry<String, Integer> entry : list) {  
            System.out.println(entry.getKey() + " => " + entry.getValue());  
        }  
  
  
        System.out.println("\n========== 4️⃣ Comparator 自定义排序 ==========");  
        // 4️⃣ 使用 Comparator 对象集合自定义排序规则  
        class Student {  
            String name;  
            int age;  
            int score;  
  
            Student(String name, int age, int score) {  
                this.name = name;  
                this.age = age;  
                this.score = score;  
            }  
  
            @Override  
            public String toString() {  
                return name + "(" + age + "岁, 分数=" + score + ")"; // 定义打印格式  
            }  
        }  
  
        List<Student> students = new ArrayList<>();  
        students.add(new Student("Alice", 20, 95));  
        students.add(new Student("Bob", 22, 82));  
        students.add(new Student("Charlie", 21, 90));  
  
        System.out.println("原始学生列表：");  
        for (Student s : students) System.out.println(s);  
  
        // 使用 Lambda 表达式按分数降序排序  
        students.sort((s1, s2) -> s2.score - s1.score);  
        System.out.println("\n按分数降序排序：");  
        for (Student s : students) System.out.println(s);  
  
        // 使用 Comparator.comparingInt 方法按年龄升序排序  
        students.sort(Comparator.comparingInt(s -> s.age));  
        System.out.println("\n按年龄升序排序：");  
        for (Student s : students) System.out.println(s);  
    }  
}
```

---

