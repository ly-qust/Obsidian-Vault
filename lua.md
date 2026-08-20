在秒杀系统和高并发场景中，**Lua** 脚本被誉为“分布式锁和原子操作的终极武器”。

以下是关于 Lua 的详细讲解，以及为什么它在 Redis 中如此重要。

---

### 1. 什么是 Lua？

**Lua** 是一种轻量级、嵌入式的脚本语言。
*   **轻量：** 它的解释器非常小（只有几百 KB）。
*   **快速：** Lua 的执行速度在脚本语言中是数一数二的。
*   **嵌入式：** 它通常不作为独立程序运行，而是“嵌入”到其他应用中（比如 Nginx、Redis、游戏引擎如魔兽世界）。

**在 Redis 中的角色：**
Redis 内置了 Lua 解释器。你可以把一段逻辑写成 Lua 脚本发送给 Redis，Redis 会帮你执行这串逻辑。

---

### 2. 为什么秒杀必须用 Lua？（核心价值：原子性）

这是最关键的知识点。

#### A. 传统的“两次请求”问题
如果你用 Java 代码实现扣减库存：
1.  Java 发送 `GET stock` 给 Redis（请求 1）。
2.  Java 拿到结果，判断 `if (stock > 0)`。
3.  Java 发送 `DECR stock` 给 Redis（请求 2）。

**风险：** 在请求 1 和请求 2 之间，可能有另一个人的请求插了进来，把最后一件衣服抢走了。这时你的 Java 代码还在执行第 3 步，导致库存变成 -1（超卖）。

#### B. Lua 的“原子包裹”
当你把这段逻辑写进 Lua 脚本发给 Redis 时：
*   Redis 保证**整个脚本在执行期间，不会被任何其他指令插入**。
*   它就像一个“事务”，要么全部成功，要么不执行。
*   对于 Redis 来说，执行一个 Lua 脚本就像执行一个普通的 `SET` 命令一样，是**不可分割**的。

**结论：Lua 解决了并发下的“查询+扣减”的一致性问题。**

---

### 3. Lua 基础语法快速入门

如果你会 Java，学 Lua 只需 5 分钟：

*   **定义变量：** `local a = 10`（一定要加 `local`，否则会变成全局变量，污染 Redis）。
*   **条件判断：**
    ```lua
    if a > 10 then
        -- 做某事
    elseif a == 10 then
        -- 做某事
    else
        -- 做某事
    end
    ```
*   **逻辑运算符：** `and`, `or`, `not`（对应 Java 的 `&&`, `||`, `!`）。
*   **连接字符串：** 使用 `..`（例如 `"Hello " .. "World"`）。

---

### 4. 在 Redis 中使用 Lua

Redis 提供了 `EVAL` 命令来执行脚本。

#### 语法：
`EVAL "脚本内容" 参数个数 KEY1 KEY2 ... ARG1 ARG2 ...`

*   **KEYS[n]：** 表示脚本操作的 Redis 键名。
*   **ARGV[n]：** 表示脚本需要的参数（非键名）。

#### 实战示例：秒杀预减库存脚本
假设我们要实现：判断库存是否充足，充足就减 1，并记录抢到的用户 ID。

```lua
-- KEYS[1]: 库存的 Key (如 "item_101_stock")
-- KEYS[2]: 已抢购成功的用户 Set (如 "item_101_users")
-- ARGV[1]: 当前用户 ID

-- 1. 获取当前库存
local stock = redis.call('get', KEYS[1])

-- 2. 判断用户是否已经抢过 (幂等性校验)
local isMember = redis.call('sismember', KEYS[2], ARGV[1])
if isMember == 1 then
    return -1 -- 返回 -1 表示重复下单
end

-- 3. 判断库存是否足够
if (not stock) or (tonumber(stock) <= 0) then
    return 0 -- 返回 0 表示没库存了
end

-- 4. 扣减库存，并记录用户
redis.call('decr', KEYS[1])
redis.call('sadd', KEYS[2], ARGV[1])

return 1 -- 返回 1 表示抢购成功
```

---

### 5. Java 如何调用 Lua 脚本？

在 Spring Boot 中，我们通常使用 `StringRedisTemplate`。

```java
// 1. 准备脚本
String script = "local stock = redis.call('get', KEYS[1])..."; // 刚才那段Lua
DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
redisScript.setScriptText(script);
redisScript.setResultType(Long.class);

// 2. 执行脚本
Long result = redisTemplate.execute(
    redisScript, 
    Arrays.asList("item_101_stock", "item_101_users"), // 对应 KEYS
    "user_888"                                        // 对应 ARGV
);

// 3. 根据返回值判断结果
if (result == 1) { 
    // 发送 MQ 下单
}
```

---

### 6. Lua 脚本的注意事项（避坑指南）

1.  **不要写长循环和复杂逻辑：** Redis 是单线程的。如果你的 Lua 脚本跑了 100ms，那么这 100ms 内 Redis 什么都干不了，所有其他请求都会卡死。
2.  **集群限制：** 在 Redis Cluster（集群）环境下，Lua 脚本操作的多个 Key 必须落在**同一个槽（Slot）**上，否则会报错。
    *   *解决方法：* 使用 `{}` Hash Tag（例如 `stock:{item101}` 和 `users:{item101}`）。
3.  **脚本缓存（EVALSHA）：** 每次发送一大段脚本给 Redis 浪费带宽。实际工业生产中，我们会先用 `SCRIPT LOAD` 把脚本存到 Redis 里生成一个 **SHA1 校验码**，以后只传校验码和参数就行了。

### 总结
Lua 脚本在秒杀系统中的地位就是：**利用 Redis 的单线程特性，通过脚本封装，实现高性能的原子业务逻辑。** 它是防止超卖、保证数据一致性的“最后一道防线”。
