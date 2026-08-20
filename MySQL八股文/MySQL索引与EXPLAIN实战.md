索引设计原则
等值过滤列通常放在前面，范围列或排序列放在后面；但最终要以真实 SQL、数据分布和 `EXPLAIN` 验证。

## 一、核心结论

索引的本质是用额外的有序数据结构和写入成本，换取更少的数据扫描。

分析一条 SQL 时，不要只问“是否用了索引”，而要继续判断：

```text
定位是否精准 → 预计扫描多少行 → 是否需要回表 → 是否额外排序/临时表 → 实际耗时
```

## 二、索引脑内模型

### 1. 聚簇索引

InnoDB 表数据按照主键索引组织，聚簇索引叶子节点保存完整行数据。一张表只能有一个聚簇索引。

### 2. 二级索引

二级索引叶子节点主要保存：

```text
二级索引列 + 主键值
```

如果二级索引没有查询需要的全部字段，就根据主键再次访问聚簇索引取得完整数据。

### 3. 回表

```text
二级索引找到主键
→ 使用主键访问聚簇索引
→ 取得索引中不存在的列
```

回表不是“回到数据库”，而是从二级索引树再次访问聚簇索引树。

### 4. 覆盖索引

查询所需字段都能从某个索引中取得，不需要回表。

```sql
CREATE INDEX idx_application_user_campaign
    ON application (user_id, campaign_id);

SELECT id, user_id, campaign_id
FROM application
WHERE user_id = 100 AND campaign_id = 20;
```

InnoDB 二级索引还保存主键 `id`，因此该查询可以被覆盖，`Extra` 常见 `Using index`。

如果再查询 `status`，而索引中没有 `status`，通常需要回表。

## 三、联合索引与最左匹配

联合索引 `(a, b, c)` 可以理解为：先按 `a` 排列，`a` 相同时按 `b` 排列，前两列相同时再按 `c` 排列。

因此通常可以支持：

```sql
WHERE a = ?
WHERE a = ? AND b = ?
WHERE a = ? AND b = ? AND c = ?
```

不能直接依赖它精准定位：

```sql
WHERE b = ?
```

关键认识：

- `WHERE` 条件的书写顺序通常不影响索引匹配，优化器会生成执行计划。
- 中间列缺失时，前面的列仍可能生效，但后续列不一定能继续用于定位或排序。
- 遇到范围条件后，后续列通常难以继续用于缩小索引扫描范围。
- “索引失效”不一定是整个索引完全不用，应具体说明哪些列还能发挥作用。

## 四、EXPLAIN 核心判断表

| 字段 | 重点含义 | 面试判断 |
|---|---|---|
| `type` | 表访问方式 | 常见关注顺序：`const`、`eq_ref`、`ref`、`range`、`index`、`ALL`；越靠后通常扫描越多，但必须结合数据量判断 |
| `key` | 优化器实际选择的索引 | 不为空只表示用了索引，不代表一定快，也不代表用了预期索引 |
| `rows` | 优化器估算需要检查的行数 | 不是最终返回行数，估算也可能和真实值有偏差 |
| `Extra` | 额外执行信息 | 重点看覆盖索引、额外过滤、排序和临时表 |

### 常见 `type`

| type | 含义 |
|---|---|
| `const` | 通过主键或唯一索引的常量等值条件，最多匹配一行 |
| `eq_ref` | 多表连接中，前表每行通过主键或唯一索引最多匹配一行 |
| `ref` | 普通非唯一索引的等值匹配，可能得到多行 |
| `range` | 使用索引进行范围扫描，如 `<`、`>`、`BETWEEN`、部分 `IN` |
| `index` | 扫描整棵索引，通常比全表行更窄，但仍可能扫描很多记录 |
| `ALL` | 全表扫描，大表上需要重点关注 |

### 常见 `Extra`

| Extra | 含义 |
|---|---|
| `Using index` | 使用覆盖索引，不需要回表 |
| `Using where` | 读取后还要根据 `WHERE` 条件过滤 |
| `Using index condition` | 使用索引条件下推，在存储引擎层尽早过滤 |
| `Using filesort` | 索引不能直接提供目标顺序，需要额外排序；不代表一定写磁盘 |
| `Using temporary` | 需要临时表，常见于部分分组、去重或排序场景 |

## 五、三条 SQL 实战

### SQL 1：覆盖索引

```sql
SELECT id, user_id, campaign_id
FROM application
WHERE user_id = 100
  AND campaign_id = 20;
```

可用索引：

```sql
(user_id, campaign_id)
```

预期判断：等值匹配，扫描行数少，查询列被二级索引覆盖，`Extra` 可能为 `Using index`。

### SQL 2：过滤并排序

```sql
SELECT id, application_no, status, create_time
FROM application
WHERE user_id = 100
  AND status = 'SUBMITTED'
ORDER BY create_time DESC
LIMIT 20;
```

优先验证的索引：

```sql
CREATE INDEX idx_application_user_status_create
    ON application (user_id, status, create_time DESC);
```

原因：`user_id`、`status` 是等值过滤列，`create_time` 在过滤后提供顺序，可以读取前 20 条后提前停止。

### SQL 3：Outbox 到期任务扫描

```sql
SELECT *
FROM mq_outbox
WHERE (status = 'PENDING' AND next_retry_time <= NOW())
   OR (status = 'SENDING' AND next_retry_time <= NOW())
ORDER BY id
LIMIT 100;
```

项目索引：

```sql
(status, next_retry_time)
```

判断：`status` 等值、`next_retry_time` 范围查询，因此 `type` 常见 `range`。索引可以缩小到期任务范围，但不能保证结果整体按 `id` 有序，仍可能出现 `Using filesort`。

不要盲目改成 `(status, id, next_retry_time)`：它可能有利于按 `id` 扫描，却削弱 `next_retry_time` 的范围过滤。需要用真实数据比较扫描量、排序成本和执行耗时。

## 六、为什么索引可能失效或只能部分生效

常见原因：

- 没有满足联合索引最左前缀。
- 对索引列进行计算或函数处理，如 `user_id + 1 = 101`。
- 隐式类型转换。
- 前导模糊匹配，如 `LIKE '%abc'`。
- 范围条件使后续列难以继续缩小扫描范围。
- 数据量很小或条件选择性很差，优化器判断全表扫描更便宜。
- `OR` 分支无法得到合适索引支持。
- 排序方向、字段顺序与索引顺序不匹配。

“索引失效”面试回答不要绝对化：先看 `EXPLAIN.key`，再说明联合索引中哪些列用于访问、哪些只参与过滤或排序。

## 七、为什么不是所有字段都建索引

索引有成本：

- 占用磁盘与 Buffer Pool。
- `INSERT`、`UPDATE`、`DELETE` 要维护索引树。
- 索引越多，写放大越明显。
- 低选择性字段的单列索引未必有效。
- 重复或高度重叠索引会增加维护成本。

设计原则：从高频、重要的真实 SQL 出发，围绕过滤、关联、排序和覆盖设计，再用 `EXPLAIN` 与实际耗时验证。

## 八、慢 SQL 下一步查什么

```text
1. 获取真实 SQL、参数和执行频率
2. 看 EXPLAIN / EXPLAIN ANALYZE
3. 检查 type、key、rows、Extra
4. 检查表数据量、选择性和统计信息
5. 判断是否扫描过多、频繁回表、额外排序或临时表
6. 检查锁等待、磁盘 IO、Buffer Pool 和数据库负载
7. 优化 SQL 或索引后重新测量
```

不要看到慢 SQL 就直接加索引。慢可能来自锁等待、返回数据过多、深分页、网络、磁盘或下游压力。

## 九、普通索引与唯一索引

普通索引主要提升查询效率；唯一索引还负责在数据库写入阶段守住业务不变量。

```sql
UNIQUE (campaign_id, user_id)
```

能够保证同一用户在同一活动中最多一条申请。应用层“先查再插”存在并发窗口，唯一约束才是最终防线。

## 十、90 秒面试表达

> 我分析索引时不会只看有没有使用索引，而是先看 SQL 的过滤、排序和返回列，再结合 EXPLAIN 的 type、key、rows、Extra 判断访问方式、扫描量、是否回表以及是否存在额外排序。InnoDB 的聚簇索引叶子节点保存完整行，二级索引叶子节点保存索引列和主键；查询列全部在二级索引中就是覆盖索引，否则需要根据主键回表。联合索引按照字段顺序组织，所以通常把高频等值过滤列放在前面，再考虑范围或排序列，但这不是固定公式，还要结合选择性和真实数据验证。索引不是越多越好，因为会增加空间和写入维护成本。遇到慢 SQL，我会先拿到真实参数和执行计划，再检查扫描行数、回表、filesort、临时表和锁等待，优化后用实际耗时复测。

## 十一、面试自测

1. `key` 不为空为什么仍可能很慢？
2. `rows` 是扫描行数还是返回行数？
3. `Using filesort` 是否一定发生磁盘排序？
4. `Using index` 和 `Using index condition` 有什么区别？
5. `(a,b,c)` 在缺少 `b` 时，`a` 和 `c` 分别还能做什么？
6. 为什么联合索引字段顺序不能只按 WHERE 的书写顺序决定？
7. 为什么“先查再插”不能代替唯一约束？

