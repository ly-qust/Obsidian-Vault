> [!info] 当前面试主入口
> [[redis/Redis面试精炼笔记/00-Redis面试知识地图]]
> 本文为完整 Reference，需要补细节时再查。

在秒杀系统中，如果说 Redis 是“全公司的共享书库”，那么 **Caffeine** 就是你“桌子上的笔记本”。

Caffeine 是目前 **Java 领域性能最强、最快** 的本地缓存库，被称为“**缓存之王**”。在 Spring Boot 2.x 之后，它取代了 Guava Cache 成为默认的缓存实现。

以下是关于 Caffeine 的深度解析：

---

### 1. 为什么有了 Redis 还需要 Caffeine？

这是很多人的第一疑问。虽然 Redis 很快（毫秒级），但它毕竟是**远程服务**：
*   **网络开销：** 你的应用服务器访问 Redis 需要经过网卡、网线、交换机。
*   **序列化开销：** 从 Redis 取出的数据是字节流，需要转换成 Java 对象（JSON 或 Protobuf 转换）。
*   **带宽限制：** 当瞬间流量达到 100w/s 时，内网带宽可能成为瓶颈。

**Caffeine 的优势：**
*   **零网络开销：** 数据就在你的 JVM 堆内存里。
*   **零序列化：** 直接存取 Java 对象引用。
*   **性能：** 访问耗时在 **纳秒 (ns)** 级别，比 Redis 快上千倍。

---

### 2. Caffeine 的核心：W-TinyLFU 算法

为什么 Caffeine 性能这么好？因为它解决了一个核心命题：**如何在有限的内存里，留下最有价值的数据？**

传统的算法有：
*   **LRU (Least Recently Used):** 淘汰掉最久没被访问的数据。缺点：如果突然来了一次全量扫描（比如某个爬虫），会把热点数据全刷掉。
*   **LFU (Least Frequently Used):** 淘汰掉访问频率最低的数据。缺点：需要维护频率计数，占空间，且旧的热数据很难被淘汰。

**Caffeine 的黑科技：W-TinyLFU**
它结合了 LRU 和 LFU 的优点，像一个聪明的管理员：
1.  **Window 区：** 新进来的数据先在窗口区待着（LRU）。
2.  **Filter (过滤器)：** 记录数据的访问频率（使用类似布隆过滤器的算法，极其省内存）。
3.  **Main 区：** 当 Window 区满了，新数据想进入主区，管理员会对比：这个新人的“潜力”（最近访问频率）是否比主区里那个“最不活跃”的老人高？如果高，就替换，否则拒绝。

**结论：** 这套算法保证了 Caffeine 在极小内存占用下，依然能拥有极高的**缓存命中率**。

---

### 3. 如何在代码中使用 Caffeine？

它的 API 非常接近 Java 的 `Map`，采用 Builder 模式构建。

#### A. 依赖引入 (Maven)
```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
```

#### B. 基础用法
```java
Cache<String, Object> cache = Caffeine.newBuilder()
    .initialCapacity(100)          // 初始容量
    .maximumSize(1000)             // 最大条数（超过后按算法淘汰）
    .expireAfterWrite(Duration.ofSeconds(60)) // 写入 60s 后过期
    .recordStats()                 // 开启统计（命中率等）
    .build();

// 存
cache.put("product_101", productObj);

// 取（如果不存在，返回 null）
Object val = cache.getIfPresent("product_101");

// 自动加载取（如果不存在，执行 lambda 去数据库/Redis 查并存入）
Object val2 = cache.get("product_101", k -> findProductInDB(k));
```

---

### 4. 在秒杀系统中的实战配置

在秒杀场景下，Caffeine 通常用来存放以下三类数据：

1.  **商品详情（热点）：** 商品名、描述、图片链接。这些数据在秒杀期间不会变。
2.  **活动规则：** 活动什么时候开始，限购几件。
3.  **售罄标记（Sold Out Flag）：** **这是最关键的一点！**

#### 场景模拟：防止“缓存击穿”
当商品库存卖光后，Redis 里的库存变 0。但此时依然每秒有 10 万个请求过来查 Redis。
**优化手段：**
1.  第一个请求发现 Redis 库存为 0。
2.  在应用服务器的 **Caffeine** 里存入一个标记：`is_sold_out_101 = true`，过期时间 30 秒。
3.  后续的 999,999 个请求进来，先查本地 Caffeine，发现 `true`，直接返回“已售罄”。
4.  **结果：** 只有 1 个请求发往了 Redis，剩下的全在本地被挡回去了。这就是“**分层过滤**”的威力。

---

### 5. 注意事项（避坑指南）

1.  **内存管理：** 本地缓存占用的是 JVM 堆内存。如果设置 `maximumSize` 太大，会导致频繁的 GC（垃圾回收），反而让系统变慢。
2.  **数据一致性：** 本地缓存很难做到“多台服务器实时同步”。
    *   *对策：* 秒杀期间，商品价格、详情设置短的过期时间（如 5-10 秒），或者使用 Redis 的消息队列通知每台服务器刷新本地缓存。
3.  **不要什么都存：** 只存高频访问的热点数据。如果什么都往 Caffeine 放，它就变成了另一个“慢速数据库”。

### 总结

*   **Redis：** 保证**数据一致**（所有服务器看同一份数据）、**持久化**。
*   **Caffeine：** 保证**极致性能**、**保护下游**（挡住对 Redis 的疯狂请求）。

在 Java 21 虚拟线程的环境下，虚拟线程负责处理业务逻辑，而 Caffeine 负责在内存中极速提供数据。两者结合，可以让单机抗住极高的并发流量。
