

# 理论篇

### 一、它解决什么问题


Hashmap本质是键值映射，数组需要下标才能O(1)访问，业务中的key往往是字符串或者对象，不是整数下标，Hashmap用hash函数将任意的key转为数组下标，实现了对象当下标的效果。

**一句话：用空间换时间，把"查找"从 O(n) 降到 O(1)**

没有 HashMap 之前，你要查一个元素存不存在，只能遍历数组/列表，线性扫。HashMap 的核心就一件事：**用 hash 函数把 key 映射到数组下标，直接定位到桶**。

实际解决的问题场景：

- 去重：`Set` 底层就是 HashMap
- 缓存：`LRUCache`、本地缓存
- 统计词频、分组聚合
- 查表：用户ID→用户信息、路径→配置文件等

**它本质解决的是"关联查询"问题**——两个东西之间存在映射关系，你怎么快速从 A 找到 B。

---

### 二、会踩坑的场景

**坑1：key 没重写 hashCode / equals**

```java
class User {
    String name;
    int age;
    // 没重写 hashCode 和 equals
}

Map<User, String> map = new HashMap<>();
map.put(new User("张三", 20), "北京");
map.get(new User("张三", 20)); // null！
```

两个内容相同的 User 对象，hashCode 不同（默认是内存地址），被放进了不同桶，查不到。

**坑2：并发 put 导致数据丢失或死循环**

- Java 7 扩容时，头插法可能导致**环形链表**，get 时死循环
- Java 8 改了尾插+Unsafe，不会死循环了，但**仍会丢数据**（两个线程同时 put 不同 key，算到同一个桶，后写的覆盖先写的）
- 解决：用 `ConcurrentHashMap`

**坑3：key 的 hashCode 全部相同 → 链表退化成 O(n)**

比如所有 key 都返回 0，全进一个桶。Java 8 有红黑树兜底（链表>8转树），但这是最坏情况，生产环境要杜绝。

**坑4：扩容引发性能抖动**

达到 loadFactor×capacity 就扩容，扩容要重新 hash 所有元素，O(n)。如果在循环里反复 put 触发扩容，性能会突变。解决：初始化时预估大小 `new HashMap<>(expectedSize / 0.75f + 1)`。

**坑5：key 是可变的**

```java
User u = new User("张三");
map.put(u, "北京");
u.name = "李四"; // 改 key 了！
map.get(u); // 还是 null
```

改了 key 之后，hashCode 变了，找到的桶不对。所以**key 最好是不可变对象**（String、Integer 等都是 immutable 的）。

---

### 三、如果让我设计一个替代方案

**先想清楚 HashMap 的弱点，针对性改：**

| HashMap 弱点      | 替代方案思路                                          |
| --------------- | ----------------------------------------------- |
| 哈希冲突退化 O(n)     | **跳表（SkipList）**：插入/查找都是 O(log n) 稳定，不用扩容重排     |
| 并发安全问题          | **分片锁 / 无锁**：如 ConcurrentHashMap 用 CAS+Node 级别锁 |
| 缓存失效场景不合适       | **LSM-Tree 思路**：写多读少场景用分段合并                     |
| 内存浪费（null slot） | **开放寻址法**：不用链表，冲突时直接找下一个空位                      |

**最值得讲的替代：开放寻址 HashMap**

这是最直接的改造思路，核心改动：

```
传统 HashMap（链地址法）：数组 + 每个桶挂链表
开放寻址 HashMap：数组 + 冲突时线性/二次探测找下一个空位
```

**具体怎么设计：**

1. **数据结构**：一个连续的 `Entry[] table`，没有 next 指针，省内存
2. **插入冲突**：`index = (hash + probe) % capacity`，依次找空位
3. **删除**：不能真删，标记为 `DELETED`（ tombstone ），否则中断后续探测链
4. **优点**：
    - 无链表节点内存开销，缓存局部性好（连续内存），查得快
    - 适合键数量可预估的场景
5. **缺点**：
    - 删除麻烦（tombstone）
    - 负载因子不能太高（>0.7 性能明显下降），而链地址法可以到 1.0 再扩容
    - 扩容时所有元素要重排

**更激进的方案：跳表 HashMap**

- 用多层跳表代替"数组+桶"，每层都是有序链表
- 查找 O(log n)，最坏情况也不会退化
- 天然支持有序遍历（HashMap 本身无序，但有些场景需要）
- Java 的 `TreeMap` 就是红黑树，跳表是它的并发友好替代

**面试回答的层次：**

> "如果让我设计替代方案，我会从三个方向考虑：
> 
> 1. **解决冲突方式**：从链地址法改成开放寻址，省内存、提高缓存命中率，适合写多读少且内存敏感的场景
> 2. **解决退化问题**：用跳表替代红黑树兜底，O(log n) 稳定，不用判断树化阈值
> 3. **解决并发问题**：借鉴 ConcurrentHashMap 的分段锁 + CAS，或者用无锁的 hopscotch hashing
> 
> 但要说为什么不用这些替代——因为 HashMap 在绝大多数场景已经足够好，开放寻址的 tombstone 管理和负载因子限制反而增加了复杂度，跳表的内存开销更大。工程上选 HashMap 是平衡了实现成本和性能的最佳选择。"

---

**总结一句话：**

> HashMap 解决了"关联查询"的效率问题，踩坑主要在于 key 设计、并发、哈希质量、可变 key。替代方案的核心思路是**换冲突解决策略（开放寻址）或换数据结构（跳表）**，但实际生产选 HashMap 是因为它综合成本最低。



# 面试篇
## 一、基础题（必背，热身用）

1. HashMap 的底层数据结构是什么？JDK 1.7 和 1.8 有什么区别？
2. HashMap 的默认初始容量、负载因子、扩容阈值分别是多少？
3. HashMap 允许 key 和 value 为 null 吗？null key 存在哪里？
允许为空，null key 的hash固定为0，存在下标为0的地方，只能有一个null key，null value有多个
4. HashMap 的时间复杂度是多少？最坏情况呢？
- **平均情况**：O (1)（无哈希冲突，直接定位数组下标）
- **链表情况**：O (n)（冲突元素挂链表，需遍历）
- **红黑树情况**：O (log n)（链表过长转树）
- **最坏情况**：O (n) 或 O (log n)（取决于是否树化）
1. HashMap 和 Hashtable 有什么区别？
![[Pasted image 20260817112932.png]]
2. HashMap 和 LinkedHashMap 有什么区别？
- LinkedHashMap **继承自 HashMap**，底层结构相同
- 额外维护了一个**双向链表**，记录元素的**插入顺序**
- 支持**访问顺序**（`accessOrder=true`）：每次 get/put 后把元素移到链表尾部
- 可用于实现 **LRU 缓存**
- 遍历 LinkedHashMap 是按插入顺序，HashMap 遍历顺序不确定

1. HashMap 是线程安全的吗？不安全的话用什么替代？
ConcurrentHashMap
2. 重写 equals 为什么必须重写 hashCode？
HashMap 判断 key 相等的逻辑是**先比 hashCode，再比 equals**：

- hashCode 不同 → 直接判定 key 不同，**不会调用 equals**
- hashCode 相同 → 才调用 equals 确认

1. 为什么 String、Integer 这些包装类适合作为 HashMap 的 key？

 **不可变**：hashCode 不会变，不会因为修改 key 导致找不到数据（内存泄漏）
 **缓存了 hashCode**：String 内部有 `hash` 字段，第一次计算后缓存，后续直接返回
**equals/hashCode 重写得当**：基于内容计算，符合规范
**hash 分布均匀**：String 的 hashCode 算法（31 倍累乘）冲突率低

6. HashMap 的遍历方式有哪些？哪种效率最高？为什么？
```

// 1. entrySet（推荐，效率最高）
for (Map.Entry<K,V> e : map.entrySet()) { e.getKey(); e.getValue(); }

// 2. keySet（效率低，需要二次get）
for (K key : map.keySet()) { map.get(key); }

// 3. values（只能拿value，拿不到key）
for (V v : map.values()) { }

// 4. Iterator（可在遍历中安全删除）
Iterator<Map.Entry<K,V>> it = map.entrySet().iterator();


```

---

## 二、进阶题（高频考点，重点掌握）

11. 详细说一下 HashMap 的 put 方法执行流程。
![[Pasted image 20260817114048.png]]
12. HashMap 的容量为什么必须是 2 的幂？
核心原因：计算下标用 `(n - 1) & hash` 替代 `hash % n`

- 当 n 是 2 的幂时，`n-1` 的二进制**全是 1**（如 15 = 1111）
- `hash & 1111` 等价于 `hash % 16`，但**位运算比取模快得多**
- 如果 n 不是 2 的幂（如 15），`n-1=14=1110`，与运算结果永远是偶数，奇数下标浪费，冲突翻倍
11. HashMap 的 hash 方法（扰动函数）是怎么实现的？为什么要这么做？
12. 为什么用 `(n - 1) & hash` 计算下标，而不是 `hash % n`？
与运算更快  当 n 是 2 的幂时，`(n-1) & hash` 与 `hash % n` **结果完全等价**
13. HashMap 什么时候触发扩容？扩容成多大？
14. JDK 1.8 扩容时元素怎么迁移？为什么不需要重新计算 hash？

容量从 n 翻倍到 2n，`(2n-1)` 比 `(n-1)` 多了一个高位的 1：
```
n=16:  n-1  = 01111
2n=32: 2n-1 = 11111
              ↑ 新增的高位
```
元素的新下标只取决于 hash 的**新增高位**是 0 还是 1：

- `hash & oldCap == 0` → 高位是 0 → **新下标 = 旧下标**（不变）
- `hash & oldCap != 0` → 高位是 1 → **新下标 = 旧下标 + 旧容量**

所以不需要重新计算 `hash % newCap`，只需要一次与运算判断，**省去了重新 hash 的开销**。

15. JDK 1.7 扩容时为什么可能出现死循环？1.8 怎么解决的？

**1.7 死循环原因**：

- 1.7 用**头插法**，扩容时链表顺序会**反转**
- 线程 A 扩容到一半被挂起，线程 B 完成扩容后链表已反转
- A 恢复后继续操作指针，可能形成**环形链表**
- 下次 get 遍历链表时 → 死循环 → CPU 100%

1.8 使用尾插法，扩容之后保持链表顺序不变，但是不保证线程安全



16. 链表什么时候转红黑树？为什么阈值是 8？

链表长度>=8同时数组容量>=64   泊松分布,负载因子 0.75 时链表长度达到 8 的概率约 **0.00000006**

17. 红黑树什么时候退化成链表？为什么退化阈值是 6 而不是 8？
6 过渡

18. 为什么用红黑树而不是 AVL 树？
插入删除查找频繁，红黑树不严格，性能好

19. HashMap 解决哈希冲突用的是什么方法？为什么不用开放寻址法？
- HashMap 用**链地址法**（拉链法）：冲突元素挂链表 / 树
- 不用开放寻址法的原因：
    1. 冲突时找下一个空位，容易产生**堆积**（元素挤在一起），冲突越来越多
    2. 删除麻烦：不能直接删，要标记 "已删除"，否则后续查找断链
    3. 链表 / 树能容纳任意多冲突，可扩展性强（还能树化）

18. HashMap 的负载因子为什么是 0.75？可以改成 1.0 吗？有什么影响？
- 0.75 是**时间和空间的折中**：
    - 太小（0.5）：空间浪费大，频繁扩容
    - 太大（1.0）：冲突多，链表长，查询慢
- **统计学依据**：泊松分布下，0.75 时链表长度≥8 的概率仅千万分之六

18. 说一下 HashMap 的 get 方法执行流程。
计算 key 的 hash 值

计算下标 i = (n - 1) & hash

检查数组该位置的首节点：

- hash 相同且 key 相同（== 或 equals）→ 直接返回

如果首节点不是目标：

- 是红黑树节点 → 调用 getTreeNode 在树中查找

- 是链表 → 遍历链表，逐个比较 hash 和 key

找不到 → 返回 null

19. HashMap 判断两个 key 相等的逻辑是什么？为什么先比 hash 再比 equals？

先判断 hash 是否相同，相同再判断 equals，hash 判断快

20. 什么是 fast-fail？HashMap 怎么实现的？
- **是什么**：遍历过程中如果 map 结构被修改（put/remove/clear），立即抛出 `ConcurrentModificationException`
- **怎么实现**：
    - `modCount` 字段：每次结构修改 +1
    - 迭代器创建时记录 `expectedModCount = modCount`
    - 每次 `next()` 时检查 `modCount == expectedModCount`，不等就抛异常
- **为什么需要**：并发修改时遍历可能出现重复、遗漏、死循环等诡异行为，fast-fail **尽早暴露问题**

20. 遍历 HashMap 时用 remove 会怎样？怎么安全删除？
- for-each 遍历中调用 `map.remove(key)` → 抛 `ConcurrentModificationException`（fast-fail）
- **安全删除方式**：使用 Iterator 的 `remove()` 方法

```
Iterator<Map.Entry<K,V>> it = map.entrySet().iterator();
while (it.hasNext()) {
    it.next();
    it.remove(); // 安全，会同步更新 expectedModCount
}
```

- Iterator.remove () 会同步更新 `expectedModCount`，所以不会触发 fast-fail

21. `new HashMap(1000)` 实际容量是多少？为什么？
22. 如果要往 HashMap 放 1000 个元素，初始容量设多少合适？为什么？
23. HashMap 的 key 可以是可变对象吗？会有什么问题？
- **可以用，但强烈不推荐**
- 问题：key 修改后 hashCode 变了，再也找不到原来的数据 → **内存泄漏**


21. modCount 字段是干什么用的？
- `modCount` 记录 HashMap **结构修改的次数**
- 每次 put（新增）、remove、clear 都会 +1
- **覆盖 value 不算**结构修改，不 +1
- 主要作用：实现 **fast-fail** 机制，迭代器遍历时检测并发修改
- 也可用于调试：观察 map 被修改了多少次

---

## 三、高阶题（拉开差距，冲击大厂）

31. JDK 1.7 头插法和 1.8 尾插法各有什么优缺点？为什么 1.8 改成尾插？
32. 详细描述 JDK 1.7 并发扩容导致死循环的过程。
33. 除了死循环，并发使用 HashMap 还可能出现哪些问题？
34. 为什么树化要求数组容量 ≥ 64？容量不够时为什么优先扩容而不是树化？
35. 扰动函数为什么用异或，不用与运算或或运算？
36. HashMap 的 `tableSizeFor` 方法是怎么实现的？为什么要先 `cap - 1`？
37. 为什么 HashMap 不直接在构造函数里初始化数组，而要等到第一次 put？
38. 红黑树插入元素后怎么维持平衡？（能说多少说多少）
39. HashMap 中 Node 节点和 TreeNode 节点有什么区别？
40. 如果让你自己实现一个 HashMap，你会怎么设计？说一下核心思路。
41. 为什么 HashMap 扩容是翻倍而不是 +10 或者其他增量？
42. HashMap 的 `resize()` 方法在多线程下可能导致什么问题？
43. 为什么 HashMap 的链表长度超过 8 才树化，而不是 4 或 16？背后的统计学依据是什么？
44. ConcurrentHashMap 和 HashMap 的区别是什么？ConcurrentHashMap 怎么保证线程安全？
45. 说一下你对 HashMap 整体设计思想的理解。

---

## 四、场景 / 手撕题

## 46. 写一个简单的 HashMap（数组 + 链表）

**核心思路：**

```
HashMap = 数组 + 链表（拉链法解决哈希冲突）
```

**数据结构设计：**

- 底层是一个 `Node[] table` 数组
- 每个数组位置存一条链表，链表节点是 `Node<K,V>`，含 `key`、`value`、`next`

**关键操作：**

- **put(key, value)**：算 `hash(key) % capacity` 找到数组下标 → 遍历该位置链表 → 有相同 key 就覆盖 value，没有就插入链表头部
- **get(key)**：算下标 → 遍历链表用 `equals()` 比较 key → 找到返回 value，否则返回 null
- **hash 计算**：简单点就是 `key.hashCode() & (capacity - 1)`（capacity 为 2 的幂时等价于取模）

**面试加分：**

- 提到负载因子 0.75，超过就扩容（2 倍扩容 + 重新 hash）
- 说清 Java 8 之后链表长度 >8 会转红黑树

---

## 47. 如何用 LinkedHashMap 实现 LRU 缓存？

**核心思路：**

```
LRU = 最近使用的放前面，淘汰最久未用的
```

**为什么 LinkedHashMap 能天然支持？**

- 它内部维护了一个**双向链表**，记录了插入/访问顺序
- 构造函数传 `accessOrder = true`，每次 `get()` 会把这个节点移到链表尾部（表示最新使用）
- 重写 `removeEldestEntry()` 方法，当 size > capacity 时返回 true，`put()` 会自动删掉最旧的节点（链表头部）

**伪代码结构：**

```java
class LRUCache extends LinkedHashMap<K,V> {
    int capacity;
    public LRUCache(int cap) {
        super(cap, 0.75f, true); // true = 按访问顺序
        this.capacity = cap;
    }
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity; // 超过容量删最老的
    }
}
```

**面试加分：**

- 手写版本是 HashMap + 双向链表 + HashMap 存节点引用，O(1) 查找 + O(1) 移动
- 提到 `LinkedHashMap` 内部本身就是这套结构，只是封装好了

---

## 48. 100万条数据的大文件，统计出现次数最多的前10个单词？

**核心思路：分治 + 堆**

**为什么不能一次性全放内存？**

- 100万条数据可能很大，直接全读进内存有风险（虽然100万不算特别大，但思路要通用）

**步骤：**

1. **HashMap 统计词频**：逐行读文件，`map.put(word, map.getOrDefault(word, 0) + 1)`，得到每个单词的出现次数
2. **用大小为 10 的小顶堆找 TopK**：
    - 遍历 HashMap 的 entry，维护一个**最小堆**（堆顶是当前 Top10 中最小的）
    - 如果当前 entry 的 count > 堆顶 count，弹出堆顶、插入当前 entry
    - 遍历完，堆里就是出现次数最多的前10个

**时间复杂度：** O(n) 统计 + O(m log k) 堆排序（m 是不同单词数，k=10）

**面试加分（海量数据版）：**

- 如果数据大到内存放不下 → **哈希分桶**：把文件按 `hash(word) % N` 分成 N 个小文件，每个小文件单独统计 Top10，最后归并
- 这就是 MapReduce 的核心思想

---

## 49. 怎么判断两个 HashMap 是否相等？

**核心思路：**

```java
map1.equals(map2)
```

**equals 的判断逻辑（HashMap 源码）：**

1. 引用相同 → true
2. 类型不是 Map → false
3. size 不同 → false
4. **遍历 map1 的每个 entry**：在 map2 中用 `equals()` 比较 key 和 value
    - key 相同：`k.equals(e.getKey())`
    - value 相同：`v.equals(e.getValue())`

**关键点：**

- 比较的是**内容相等**，不是引用相等
- key 和 value 都要用 `equals()` 递归比较（value 如果是对象，也走 equals）
- 顺序无关，只要 key-value 对完全一致就行

**面试加分：**

- 提到 `hashCode()` 也要一致（equals 的契约要求）
- 如果是自定义对象作 value，要重写 equals/hashCode，否则返回 false

---

## 50. HashMap 中所有 key 的 hashCode 都相同，会发生什么？怎么优化？

**会发生什么：**

```
哈希冲突 → 所有元素都在同一个数组位置 → 退化成一条链表
```

- `put` 的时间复杂度从 O(1) 变成 **O(n)**（要遍历整个链表比较 key）
- `get` 同样变成 O(n)

**优化方案（Java 8 的改进）：**

- 当链表长度 > 8 时，**链表转红黑树**
- 红黑树查找复杂度 O(log n)，即使所有 key hash 相同，性能也不会退化到 O(n)

**根本解法：**

- 重写 key 的 `hashCode()`，让不同对象有不同的 hash 值（这是设计 key 类时的责任）
- 例如 `String` 的 hashCode 是字符的加权求和，`Integer` 直接返回值，都很好

**面试加分：**

- 提到 `hash()` 扰动函数：Java 8 里 `h ^ (h >>> 16)`，让高16位也参与运算，减轻低质量 hashCode 的影响
- 但根本问题还是要 key 类正确重写 hashCode

---

**总结一句话记忆：**

> 46 拉链法 → 47 accessOrder+removeEldest → 48 HashMap+小顶堆 → 49 逐 entry equals → 50 退化成链表→红黑树兜底













