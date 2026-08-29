
> [!info] 当前面试主入口
> [[MySQL八股文/MySQL面试精炼笔记/00-MySQL面试知识地图]]
> 本文为完整 Reference，需要补细节时再查。

下面是一份可以直接整理进笔记、反复背诵的版本。先记住一个前提：

> 以下 SQL 假设 `id` 是 `application` 表的主键，索引为：
> 
> ```
> CREATE INDEX idx_status_campaign_time
> ON application(status, campaign_id, create_time);
> ```

## 一、先背 8 句话

1. **B+Tree 是 MySQL 最常用的索引结构，数据按有序树组织，适合等值、范围、排序和分组。**
    
2. **聚簇索引决定 InnoDB 表中数据行的实际组织方式，主键索引的叶子节点保存完整行数据。**
    
3. **二级索引的叶子节点保存索引列和主键值，需要完整行数据时，通常要根据主键回到聚簇索引查找，这就是回表。**
    
4. **联合索引不是多个单列索引的简单相加，而是一棵按照多个列组合排序的 B+Tree。**
    
5. **联合索引遵循最左匹配原则，只有从最左列开始，索引才能有效定位连续范围。**
    
6. **等值条件通常可以继续使用后续索引列；范围条件之后的列，一般不能继续用于缩小扫描区间，但可能用于 ICP 或过滤。**
    
7. **覆盖索引是查询所需列全部包含在索引中，查询可以直接从索引叶子节点返回结果，从而减少回表。**
    
8. **EXPLAIN 实际分析时，重点看 `type`、`key`、`key_len`、`rows` 和 `Extra`，最终还要结合实际数据量和执行耗时判断。**
    

## 二、五个基础概念

### 1. B+Tree

B+Tree 是一种多路平衡树：

- 非叶子节点主要保存目录信息。
- 叶子节点保存索引记录。
- 叶子节点之间通常有链表，便于范围扫描。
- 树高较低，磁盘 I/O 次数较少。

因此它适合：

```
WHERE id = 10
WHERE id > 10
ORDER BY id
GROUP BY id
```

### 2. 聚簇索引

InnoDB 的聚簇索引通常就是主键索引：

```
主键索引叶子节点 = 主键值 + 完整数据行
```

所以：

```
SELECT * FROM application WHERE id = 10;
```

找到主键索引叶子节点后，就已经拿到了完整行，不需要再查其他地方。

### 3. 二级索引

例如：

```
CREATE INDEX idx_status ON application(status);
```

二级索引叶子节点大致保存：

```
status + 主键 id
```

如果查询还需要其他字段：

```
SELECT create_time
FROM application
WHERE status = 'SUBMITTED';
```

流程是：

1. 先查 `idx_status`。
2. 得到主键 `id`。
3. 再根据 `id` 查聚簇索引。
4. 取出 `create_time`。

第 3 步就是回表。

### 4. 联合索引

联合索引：

```
(status, campaign_id, create_time)
```

不是三棵索引，而是一棵按照组合键排序的索引：

```
(status, campaign_id, create_time)
```

排序效果类似：

```
(status=APPROVED, campaign_id=1, create_time=...)
(status=APPROVED, campaign_id=2, create_time=...)
(status=SUBMITTED, campaign_id=1, create_time=...)
(status=SUBMITTED, campaign_id=10, create_time=...)
```

因此列的顺序会直接影响索引能否被有效使用。

### 5. 覆盖索引

如果查询需要的所有列都在索引中，就称为覆盖索引。

例如：

```
SELECT status, campaign_id, create_time
FROM application
WHERE status = 'SUBMITTED';
```

如果索引是：

```
(status, campaign_id, create_time)
```

那么查询需要的列都在索引叶子节点中，可以直接从索引返回结果。

如果查询还选择主键 `id`，在 InnoDB 中二级索引叶子节点通常也保存主键值，因此：

```
SELECT id, status, campaign_id
FROM application
WHERE status = 'SUBMITTED';
```

通常也可以覆盖。

## 三、联合索引为什么有顺序

联合索引本质上是按照元组排序：

```
(status, campaign_id, create_time)
```

先比较 `status`，`status` 相同时再比较 `campaign_id`，前两列相同时再比较 `create_time`。

因此：

```
WHERE status = 'SUBMITTED'
```

可以使用第一列。

```
WHERE status = 'SUBMITTED'
  AND campaign_id = 10
```

可以使用前两列。

```
WHERE campaign_id = 10
```

没有指定第一列，无法直接定位到某个连续区间，通常不能有效使用这个联合索引。

这就是最左匹配原则。

## 四、范围查询如何影响后续列

假设索引是：

```
(a, b, c, d)
```

查询：

```
WHERE a = 1
  AND b = 2
  AND c > 10
  AND d = 20
```

通常可以这样理解：

- `a`：用于定位。
- `b`：用于定位。
- `c`：用于范围扫描。
- `d`：通常不能再参与 B+Tree 扫描区间的确定。

原因是 `c > 10` 后，`c` 已经是一段范围，后面的 `d` 值分布在这个范围内部，无法继续形成一个连续的索引区间。

但 `d` 仍然可能：

- 通过 Index Condition Pushdown 提前过滤。(ICP下推)
- 作为存储引擎层面的判断条件。
- 在回表后由 Server 层继续过滤。

所以要区分：

> 范围查询后续列通常不能继续用于缩小索引扫描范围，但不代表后续列完全没有作用。

你的第三条 SQL 中，`create_time` 已经是联合索引的最后一列，因此没有更后面的列受到影响。



可以用 ICP索引下推解决回表问题
### 索引下推（Index Condition Pushdown, ICP）—— 抢救，提前过滤

假设你的查询变了，你需要查一个不在联合索引里的字段，比如 address（家庭住址）：  
SELECT address FROM employees WHERE dept_id > 10 AND age = 25;

注意这里有两个关键点：

1. 你要 address，但索引里只有 dept_id 和 age。所以**一定逃不掉“回表”**去翻档案柜。
    
2. dept_id > 10 是范围查询，根据最左匹配原则，后面的 age = 25 **无法用于快速定位**（失效了）。
    

**如果没有 ICP（MySQL 5.6 之前的笨办法）：**

1. 存储引擎在 Excel 上找到所有 dept_id > 10 的人（假设有100个）。
    
2. 引擎直接拿着这 100 个人的编号，去档案柜把 **100 份完整的真实档案**全部翻出来（**回表 100 次**），交给上层的 MySQL Server。
    
3. MySQL Server 打开这 100 份档案，挨个看年龄是不是 25 岁，把不是 25 岁的扔掉，最后可能只剩下 5 个人。
    

- **痛点：** 为了这 5 个人，你笨拙地回表了 100 次，浪费了大量的磁盘 I/O！
    

**有了 ICP（MySQL 5.6 之后的聪明办法）：**

1. 存储引擎在 Excel 上找到所有 dept_id > 10 的人（100个）。
    
2. 引擎突然变聪明了：“等等，虽然 age=25 不能用来跳跃翻页，但是我的 Excel 上本来就写着这些人的 age 啊！我为什么不**在这个环节直接看一眼**呢？”
    
3. 引擎直接在 Excel 上把 age != 25 的人划掉（这就叫**条件过滤下推**到了存储引擎层）。
    
4. 过滤后，发现只有 5 个人符合 age = 25。引擎只拿着这 5 个人的编号，去档案柜取档案（**回表 5 次**）。
    

- **结果：** 回表次数从 100 次暴降到 5 次，性能大幅提升！
    
- **Explain 显示：** Using index condition（表示用了索引下推技术）。



## 五、三条 SQL 的纸面分析

### SQL 1

```
SELECT id, status, campaign_id
FROM application
WHERE status = 'SUBMITTED'
  AND campaign_id = 10;
```

#### 可能使用的索引列

可以使用：

```
status、campaign_id
```

`create_time` 没有查询条件，通常不能用于进一步定位。

#### 是否覆盖索引

假设 `id` 是主键，则通常是覆盖索引：

- `status`：在索引中。
- `campaign_id`：在索引中。
- `id`：作为 InnoDB 二级索引记录中的主键值存在。

#### 是否回表

通常不需要回表。

#### EXPLAIN 重点验证

可能看到：

```
type: ref
key: idx_status_campaign_time
key_len: 包含 status 和 campaign_id 的长度
rows: 预计扫描行数较少
Extra: Using index
```

其中：

- `ref` 表示通过非唯一索引等值匹配。
- `Using index` 通常表示覆盖索引。
- 如果 `status` 和 `campaign_id` 组合具有唯一性，也可能出现其他访问类型，但不要只凭预期判断，必须看实际 EXPLAIN。

### SQL 2

```
SELECT *
FROM application
WHERE campaign_id = 10;
```

#### 可能使用的索引列

常规情况下不能有效使用：

```
idx_status_campaign_time
```

因为查询跳过了最左列 `status`。

#### 是否覆盖索引

不是。

`SELECT *` 要求表中的所有列，而索引只包含：

```
status、campaign_id、create_time、主键
```

其他列仍然需要读取聚簇索引。

#### 是否回表

如果选择了这个二级索引，通常需要大量回表。

#### EXPLAIN 重点验证

常见结果可能是：

```
type: ALL
key: NULL
rows: 接近整张表的行数
Extra: Using where
```

也可能因为成本估算、数据分布等原因出现：

```
type: index
```

这表示扫描整个索引，并不表示查询非常高效。

MySQL 8.0 某些场景支持 Index Skip Scan，但它有适用条件；而本查询使用了 `SELECT *`，通常不应把 Skip Scan 当作默认结果。最终必须以 EXPLAIN 为准。

### SQL 3

```
SELECT id, status, campaign_id, create_time
FROM application
WHERE status = 'SUBMITTED'
  AND campaign_id = 10
  AND create_time >= '2026-08-01';
```

#### 可能使用的索引列

可以使用全部三列：

```
status = 等值匹配
campaign_id = 等值匹配
create_time >= 范围匹配
```

索引扫描逻辑大致是：

```
先定位 status = 'SUBMITTED'
再定位 campaign_id = 10
最后扫描 create_time >= '2026-08-01' 的范围
```

#### 是否覆盖索引

假设 `id` 是主键，则通常是覆盖索引。

查询中的四个字段都可以从二级索引叶子节点获取：

```
id、status、campaign_id、create_time
```

#### 是否回表

通常不需要回表。

#### EXPLAIN 重点验证

可能看到：

```
type: range
key: idx_status_campaign_time
key_len: 包含三个索引列
rows: 预计扫描的范围行数
Extra: Using index
```

重点关注：

- 是否真的使用 `idx_status_campaign_time`。
- `key_len` 是否包含三个列。
- `rows` 是否合理。
- 是否出现 `Using index`。
- 是否出现 `Using filesort`。
- 是否出现不必要的 `Using temporary`。

## 六、三条 SQL 对比表

|SQL|可使用列|覆盖索引|回表|典型 type|典型 Extra|
|---|---|---|---|---|---|
|SQL 1|`status、campaign_id`|是，假设 `id` 是主键|通常不回表|`ref`|`Using index`|
|SQL 2|常规无法有效使用该联合索引|否|如果走二级索引则需要回表|常见 `ALL`|`Using where`|
|SQL 3|`status、campaign_id、create_time`|是，假设 `id` 是主键|通常不回表|`range`|`Using index`|

这些是纸面上的常见结果，不是优化器的绝对承诺。真实结果还取决于：

- 表数据量。
- 列的区分度。
- 统计信息。
- 数据分布。
- 是否存在其他索引。
- MySQL 版本。
- 成本估算结果。

## 七、EXPLAIN 应该按什么顺序看

建议固定成：

```
type → key → key_len → rows → Extra
```

### 1. type：访问类型

常见类型从通常较优到较差大致是：

```
const
eq_ref
ref
range
index
ALL
```

常见含义：

- `const`：根据主键或唯一索引查一行。
- `eq_ref`：连接时通过主键或唯一索引匹配一行。
- `ref`：普通索引等值查询。
- `range`：索引范围扫描。
- `index`：全索引扫描。
- `ALL`：全表扫描。

注意：

> `index` 不等于高效，它可能只是扫描整棵索引；`ALL` 也不一定错误，小表全表扫描可能比使用索引更便宜。

### 2. key：实际命中的索引

- `key` 显示最终真正使用的索引。
- `possible_keys` 只是可能使用的索引。
- `key = NULL` 表示这一表访问没有选择索引。

### 3. key_len：使用了联合索引的多少部分

这是分析联合索引非常重要的字段。

例如索引是：

```
(status, campaign_id, create_time)
```

如果 `key_len` 只对应 `status`，说明只使用了第一列；如果对应前三列，说明三列都参与了索引访问。

具体长度会受以下因素影响：

- 数据类型。
- 字符集。
- 是否允许 `NULL`。
- 长度前缀。

因此不要只凭肉眼猜数字，要结合表结构计算。

### 4. rows：预计扫描行数

`rows` 是优化器估算需要检查的行数，不是最终返回的行数。

例如：

```
rows = 100000
```

代表可能要检查约 10 万行，而不是返回 10 万行。

### 5. Extra：额外执行信息

重点记这些：

|Extra|含义|
|---|---|
|`Using index`|覆盖索引，数据可直接从索引获取|
|`Using index condition`|使用索引条件下推，仍可能回表|
|`Using where`|还需要额外执行条件过滤|
|`Using filesort`|需要额外排序|
|`Using temporary`|使用临时表|
|`Using index for group-by`|利用索引完成分组或去重|

## 八、常见索引失效或效果变差场景

### 1. 联合索引跳过最左列

```
INDEX(a, b, c)

WHERE b = 10
```

通常不能有效使用该索引。

### 2. 对索引列做函数或计算

```
WHERE DATE(create_time) = '2026-08-01'
WHERE YEAR(create_time) = 2026
WHERE id + 1 = 100
```

可以考虑改写条件或建立函数索引。

### 3. 隐式类型转换

例如数字列和字符串比较：

```
WHERE user_id = '100'
```

可能导致索引利用效果变差，应保持参数类型一致。

### 4. 前缀模糊查询

```
WHERE name LIKE '%abc'
```

通常无法利用 B+Tree 的有序性。

而：

```
WHERE name LIKE 'abc%'
```

通常可以转换为范围查找。

### 5. 范围条件出现在联合索引中间

```
INDEX(a, b, c)

WHERE a = 1
  AND b > 10
  AND c = 20
```

`c` 通常不能继续缩小索引扫描范围。

### 6. 使用低区分度字段

例如性别、状态等字段只有几个值，即使有索引，优化器也可能认为扫描大量索引再回表不如直接全表扫描。

### 7. `OR` 两边条件不适合索引

```
WHERE indexed_col = 1 OR non_indexed_col = 2
```

可能导致索引效果变差。不过 `OR` 并不是绝对不能使用索引，MySQL 有时会使用 Index Merge。

### 8. 否定条件或不等值条件

```
WHERE status <> 'DELETED'
WHERE id NOT IN (1, 2, 3)
```

并非绝对失效，但通常选择性较差，优化器可能放弃索引。

### 9. 查询返回大量数据

即使索引能定位，匹配行太多、回表次数太多时，优化器也可能选择全表扫描。

## 九、90 秒面试回答模板

可以直接背：

> 联合索引不是多个单列索引的简单组合，而是一棵按照多个列组合排序的 B+Tree。比如索引 `(status, campaign_id, create_time)`，数据首先按 `status` 排序，在 `status` 相同的范围内再按 `campaign_id` 排序，最后按 `create_time` 排序，因此查询必须尽量从最左列开始匹配。等值条件通常可以继续使用后续列，但范围条件之后的列一般不能继续缩小索引扫描区间。
> 
> 覆盖索引指查询需要的列全部包含在索引叶子节点中，InnoDB 二级索引叶子节点还保存主键值，因此如果查询只取索引列和主键，就可以直接返回，减少回表。
> 
> 分析执行计划时，我会先看 `type` 判断访问方式，再看 `key` 和 `key_len` 判断是否命中预期索引及使用了哪些列，然后看 `rows` 判断扫描量，最后看 `Extra`，重点关注 `Using index`、`Using index condition`、`Using filesort` 和 `Using temporary`。最终还要结合实际数据量和 `EXPLAIN ANALYZE` 验证。

## 十、闭卷自测题

1. 为什么联合索引列必须考虑顺序？
2. `(a,b,c)` 是否能有效支持只查询 `b` 的 SQL？
3. 范围条件后面的列为什么通常不能继续缩小扫描区间？
4. 覆盖索引为什么可以减少回表？
5. `Using index` 和 `Using index condition` 有什么区别？
6. `type = index` 和 `type = ALL` 分别代表什么？
7. `rows` 是返回行数还是预计扫描行数？
8. 为什么低区分度字段上的索引可能不被使用？
9. `SELECT *` 为什么更难形成覆盖索引？
10. 为什么联合索引不是多个单列索引的简单相加？
11. 对于 `(status,campaign_id,create_time)`，第三条 SQL 为什么能使用三列？
12. 如果查询只有 `campaign_id = 10`，应该如何改进索引？

第 12 题常见答案是：

```
CREATE INDEX idx_campaign_id ON application(campaign_id);
```

或者根据真实查询模式重新设计联合索引，而不是盲目把所有字段都塞进一棵索引。

官方参考：

- [MySQL Multiple-Column Indexes](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)
- [MySQL EXPLAIN Output](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [Index Condition Pushdown](https://dev.mysql.com/doc/refman/8.0/en/index-condition-pushdown-optimization.html)
- [Skip Scan Range Access Method](https://dev.mysql.com/doc/refman/8.0/en/range-optimization.html)
