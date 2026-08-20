
## 一、解决什么实际问题

> 底层：HashMap 线程不安全；HashTable 全表锁，并发性能极差。

1. **HashMap 线程不安全问题** 多线程并发 `put`，会出现：

- 链表环（死循环，CPU100%，JDK7）
- 数据丢失、覆盖
- size计数错误，元素个数不准

2. **HashTable 的痛点** 所有方法加 `synchronized`，**锁整个对象**。多个线程哪怕操作不同key，也要互相排队，并发上不去。

✅ **ConcurrentHashMap 核心目标：线程安全 + 高并发读写性能**
![[Pasted image 20260818112018.png]]
- JDK1.7：分段锁 Segment，把大map拆多个小hash表，只锁一个分段，不同分段可以并发。数组+链表


- JDK1.8：废弃Segment，改用 **CAS + synchronized锁链表头/红黑树根节点**，只锁当前要修改的桶，其他桶完全不受影响。数组+链表+红黑树

![[Pasted image 20260818112002.png]]
插入时优先使用CAS无锁，失败采用synchronized上锁，只锁单node，避免全局堵塞

从分段加锁转向节点级别的并发控制

实际业务场景：

- 多线程环境下做本地内存缓存
- 接口统计计数、接口访问次数统计
- 线程池内存储业务元数据
    
    > 注意：它只是**局部锁**，不是全锁，复合操作依然不原子。
    

### 扩容
![[Pasted image 20260818112416.png]]
1.7局部扩容不牵连大局
![[Pasted image 20260818112400.png]]
1.8是一体化设计，数组扩容使用多线程协同搬家渐进式扩容
![[Pasted image 20260818112548.png]]

size方法的操作
1.7尝试3次不加锁，3次值一样直接返回，不一样加锁
1.8上了longadder的思想，每个线程有自己的一格位置累加，最后统一汇总



---

## 二、什么场景踩坑（面试高频）

> 大坑：**单个方法是线程安全，但复合操作不是原子操作**

### 坑1：`get() + put()` 组合，先查后写

```
// 错误！多线程下会数据覆盖
ConcurrentHashMap<String,Integer> map = new ConcurrentHashMap<>();
Integer val = map.get("a");
if(val == null){
    map.put("a",1);
}
```

两个线程同时get发现null，都会执行put，发生覆盖。 👉 不能用 get+put，要用 `computeIfAbsent`。

### 坑2：计数累加 `map.get(key) +1`

```
// 错误
int num = map.get("cnt");
map.put("cnt", num+1);
```

get和put是两次操作，中间其他线程修改，计数错乱。 ✅正确：`map.merge(key,1,Integer::sum)`

### 坑3：size() / isEmpty() 不是精确实时值

并发不断写入的时候，`size()`返回的是**估算值**，遍历过程中数据发生新增/删除，统计不准。

> 不是bug，是为性能牺牲强一致性。

### 坑4：迭代器是**弱一致性**，不会抛 ConcurrentModificationException

遍历的时候其他线程增删元素，迭代器不会报错，但可能看不到最新新增的数据。 不要依赖迭代器做强一致性快照。

### 坑5：value允许存null

`key不能null，value可以null`，`get(key)==null`分不清是key不存在，还是value存了null。

### 坑6：批量操作 putAll、clear

批量操作，不会锁住整个map，批量过程中其他线程依然可以读写，不要当做原子事务。

> 总结踩坑口诀： **单个API安全，多步组合不安全；涉及判断、计数、累加，必须用原子方法(merge/compute/computeIfAbsent)**

---

## 三、如果让你设计替代方案，怎么改（面试口述版）

> 面试官：如果让你自己实现一个线程安全hashmap，你怎么做？讲2套方案，权衡优缺点。

### 方案A：参考CHM思路，CAS + 桶级锁（偏向高并发读多写多）

1. 底层数组+链表/红黑树，和HashMap结构一致
2. **读操作不加锁**，使用CAS保证读取可见性，数组、节点用`volatile`修饰，保证其他线程修改立刻可见。
3. **写操作：**
    - 桶为空：使用CAS插入节点，成功直接返回；CAS失败自旋重试。
    - 桶已经有数据：synchronized锁住这个桶的头节点，只锁单个桶，其他桶不受影响。
4. **提供原子复合接口**：实现 `merge`、`computeIfAbsent`，把“判断+修改”逻辑放到锁内部，对外屏蔽多步操作，避免用户踩复合操作非原子的坑。
5. 计数：维护baseCount + counterCell数组（类似CHM的LongAdder思想），分散计数压力，避免一个计数变量多线程竞争。

> 缺点：复合业务逻辑，还是要使用者注意；无法提供全局快照。

### 方案B：读写锁 ReentrantReadWriteLock 封装HashMap（简单实现，读多写极少场景）

```
class MySafeMap<K,V>{
    private HashMap<K,V> inner = new HashMap<>();
    private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private Lock read = rwLock.readLock();
    private Lock write = rwLock.writeLock();

    public V get(K k){
        read.lock();
        try{ return inner.get(k); }finally {read.unlock();}
    }
    public V put(K k,V v){
        write.lock();
        try{ return inner.put(k,v); }finally {write.unlock();}
    }
}
```

- 优点：实现简单，批量操作天然原子，迭代器强一致。
- **致命缺点：只要有写，所有读全部阻塞。写多场景性能暴跌。** 适用：读多，写非常稀少的场景。

### 方案C：完全无锁（纯CAS，类似Caffeine）

基于CAS + 版本号，不使用synchronized。

> 实现复杂度极高，冲突多的时候大量自旋，适合做高性能缓存组件，普通业务不推荐手写。

### 自己设计取舍总结（面试可以直接背）

1. 如果业务**读写都频繁**：选桶粒度锁 + CAS，模仿ConcurrentHashMap；重点把复合操作内置，减少使用者出错。
2. 如果**读极多、写很少**：读写锁包装HashMap，开发简单，逻辑不容易出错。
3. 如果要求**完整快照、批量操作原子**：ConcurrentHashMap做不到，读写锁封装更合适。
4. 不要直接全局synchronized锁整个哈希表，会退化成HashTable，并发直接废掉。

---

## 面试简短背诵版（快速复述）

> ConcurrentHashMap解决HashMap线程不安全，HashTable锁粒度太大并发低的问题。 JDK1.8使用CAS + 锁桶头节点，锁粒度细化，读几乎无锁。 **最大坑点：单个方法线程安全，但是多步复合操作不原子，get+put、get然后累加会出错，要用merge/computeIfAbsent。size是估算值，迭代器弱一致。**

自己设计替代：

- 高并发读写场景：数组+链表，volatile保证可见性，空桶CAS，非空桶锁单个桶，内置原子复合接口，分散计数。
- 写很少读很多：ReentrantReadWriteLock包装HashMap，实现简单，但写会阻塞全部读。
- 不能直接锁整张哈希表，并发性能很差。

需要我给你出2道代码判断题，模拟面试手写找bug吗？





5. Java 8 ConcurrentHashMap大致如何组织数据？
![[Pasted image 20260818112902.png]]


6. put过程中CAS和`synchronized`分别用于什么位置？
- **CAS：桶为空的时候，无锁插入节点**
- **synchronized：桶已经存在数据，锁住桶头，修改链表 / 红黑树内部元素**

5. 为什么ConcurrentHashMap不允许`null`键和值？
单线程 HashMap 可以用`containsKey(key)`再判断一次区分。 但**并发环境，get 和 containsKey 中间，别的线程可以增删数据**，两次调用之间状态会变，无法可靠区分。 为了避免并发下语义模糊，直接禁止存 null。
6. 它与`Collections.synchronizedMap`的主要区别是什么？
`Collections.synchronizedMap(new HashMap<>())`：给 HashMap 所有方法包一层`synchronized`，**锁整个 Map 对象（全局锁）**

| 对比   | ConcurrentHashMap                                | Collections.synchronizedMap                       |
| ---- | ------------------------------------------------ | ------------------------------------------------- |
| 锁粒度  | 桶级别锁，只锁单个 hash 桶，多线程操作不同 key 可以并发读写              | 全局锁，锁住整个 map，任何读写互斥排队                             |
| 并发性能 | 读写并发高，读基本无阻塞                                     | 并发差，写的时候所有读全部卡住                                   |
| 复合操作 | 单个方法安全；`get+put`复合操作不安全，需要 merge/computeIfAbsent | 所有方法全部全局锁，多步操作也具备原子性                              |
| 迭代器  | **弱一致性**，遍历期间其他线程修改不会抛异常，可能看不到新数据                | fail‑fast，并发修改抛出`ConcurrentModificationException` |
| null | key、value 都不能 null                               | 允许 key=null，value=null                            |
