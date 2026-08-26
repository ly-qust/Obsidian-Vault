---
tags: [Java, 集合, HashMap, ConcurrentHashMap, 面试]
---

# 集合与 Map 选型

[[00-Java后端面试知识地图|返回知识地图]] · [[02-并发工具选型|下一篇：并发]]

## 1. 一张表选型

| 结构 | 核心实现 | 查询 | 插入/删除 | 典型场景 |
|---|---|---|---|---|
| ArrayList | 动态数组 | 下标 O(1)，查找 O(n) | 尾部均摊 O(1)，中间 O(n) | 查询多、需要随机访问 |
| LinkedList | 双向链表 | O(n) | 已定位节点 O(1)，定位仍可能 O(n) | 双端队列；一般业务集合少用 |
| HashMap | 数组 + 链表/红黑树 | 平均 O(1) | 平均 O(1) | 单线程键值映射 |
| HashSet | 基于 HashMap | 平均 O(1) | 平均 O(1) | 去重、存在性判断 |
| ConcurrentHashMap | 并发哈希表 | 平均 O(1) | 平均 O(1) | 多线程共享 Map |

## 2. HashMap

```text
put：计算hash → 定位桶 → 相同key覆盖 → 否则新增 → 必要时树化/扩容
get：计算hash → 定位桶 → hash与equals匹配 → 返回value
```

面试边界：平均 O(1) 不代表最坏 O(1)；HashMap 不保证线程安全；扩容会重新组织桶分布。

## 3. ConcurrentHashMap

JDK 8 常用概括：CAS 配合桶级同步，减少全表锁竞争。单个方法线程安全不代表多步业务天然原子：

```java
if (!map.containsKey(key)) {
    map.put(key, value); // check-then-act仍存在竞态
}
```

按业务使用 `putIfAbsent`、`computeIfAbsent`、`compute` 等原子复合 API。

## 4. 高频陷阱

- `equals` 相等的 key 必须具有相同 `hashCode`；
- 可变对象作为 key，参与 hash/equals 的字段改变后可能无法正确查找；
- LinkedList 的“插删 O(1)”以已经定位节点为前提；
- ConcurrentHashMap 不能替代跨多个结构或多个步骤的事务/锁。

## 5. 30 秒回答

> 集合选型先看访问模式和并发要求。需要随机访问优先 ArrayList，普通键值映射用 HashMap，多线程共享 Map 用 ConcurrentHashMap。但 ConcurrentHashMap 只保证其 API 规定的原子性，先查再写仍要使用 `putIfAbsent` 或 `compute` 等复合原子方法。复杂度回答要带前提，例如哈希查询是平均 O(1)，链表插删 O(1) 也要求已定位节点。

