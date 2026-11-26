# MySQL EXPLAIN 执行计划、JOIN 索引与 CTE 物化

## 一、EXPLAIN 是什么？

`EXPLAIN` 是 MySQL 提供的 SQL 执行计划分析工具，用于查看 MySQL 准备如何执行一条 SQL。

主要用途：

1. 判断 SQL 是否使用索引。
2. 判断表的访问方式。
3. 查看 JOIN 的执行顺序和关联方式。
4. 估算需要扫描的数据行数。
5. 判断是否使用临时表、额外排序、覆盖索引等。
6. 分析 SQL 性能问题。

基本语法：

```
EXPLAIN SELECT * FROM users WHERE id = 100;
```

EXPLAIN 本身主要展示优化器生成的执行计划，不会真正执行普通 SELECT 查询。

MySQL 还提供了：

```
EXPLAIN FORMAT=TREE
SELECT * FROM users WHERE id = 100;
```

以树形结构展示执行计划。

以及：

```
EXPLAIN ANALYZE
SELECT * FROM users WHERE id = 100;
```

真正执行查询并展示实际执行情况。

注意：`EXPLAIN ANALYZE` 会执行 SQL，测试大数据量查询时需要注意资源消耗。

## 二、EXPLAIN 各字段的含义

传统格式的 EXPLAIN 主要包含以下字段：

| 字段          | 含义                           |
| ------------- | ------------------------------ |
| id            | 查询执行的标识                 |
| select_type   | 查询类型                       |
| table         | 当前访问的表                   |
| partitions    | 匹配的分区                     |
| type          | 当前表的访问方式               |
| possible_keys | 可能使用的索引                 |
| key           | 实际使用的索引                 |
| key_len       | 实际使用的索引长度             |
| ref           | 索引查找使用的列或常量         |
| rows          | 预计需要检查的行数             |
| filtered      | 预计经过条件过滤后剩余的百分比 |
| Extra         | 额外执行信息                   |

### 1. id

表示查询的标识。

一般情况下：

- id 相同：属于同一个查询块。
- id 不同：通常代表不同的查询块，例如子查询。
- id 为 NULL：可能表示 UNION 等操作中的结果合并步骤。

### 2. select_type

表示查询的类型，常见值：

| 类型         | 含义                           |
| ------------ | ------------------------------ |
| SIMPLE       | 简单查询，不包含子查询或 UNION |
| PRIMARY      | 最外层查询                     |
| SUBQUERY     | 子查询                         |
| DERIVED      | 派生表                         |
| UNION        | UNION 中的第二个及后续 SELECT  |
| UNION RESULT | UNION 结果合并                 |

### 3. table

表示当前执行步骤访问的表。

例如：

```
SELECT *
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

EXPLAIN 可能返回 users 和 orders 两行。

每一行对应一个表的访问步骤，而不是把数据库表物理拆分。

需要注意：EXPLAIN 中的访问顺序由优化器决定，不一定与 SQL 的书写顺序一致。

### 4. type：访问类型

这是 EXPLAIN 中最重要的字段之一，表示 MySQL 如何访问当前表。

常见类型：

| type        | 含义                         | 说明             |
| ----------- | ---------------------------- | ---------------- |
| system      | 系统表或极少量数据的特殊情况 | 特殊情况         |
| const       | 主键或唯一索引常量查找       | 通常最多一条     |
| eq_ref      | 主键或唯一索引关联查找       | 每次关联最多一条 |
| ref         | 普通索引等值查找             | 可能返回多条     |
| fulltext    | 全文索引查找                 | 全文搜索         |
| ref_or_null | 索引查找并额外查找 NULL      | 特殊访问方式     |
| index_merge | 使用多个索引合并             | 多索引访问       |
| range       | 索引范围扫描                 | 范围查询         |
| index       | 全索引扫描                   | 扫描整个索引     |
| ALL         | 全表扫描                     | 扫描整张表       |

常见访问类型的粗略性能参考：

```
ALL < index < range < ref < eq_ref < const
```

注意，这只是一般情况下的参考，并不是绝对的性能排序。实际性能还取决于数据量、索引选择性、回表成本、缓存命中等因素。

#### ALL：全表扫描

```
SELECT *
FROM users
WHERE name = '张三';
```

如果 name 没有索引，可能扫描整张表。

#### range：索引范围扫描

```
SELECT *
FROM users
WHERE id BETWEEN 100 AND 200;
```

id 是主键，可以通过 B+ 树定位范围。

#### ref：普通索引查找

```
SELECT *
FROM orders
WHERE user_id = 100;
```

如果 user_id 建立普通索引，可能使用 ref。

因为一个用户可能对应多个订单，所以可能返回多条记录。

#### eq_ref：唯一索引关联查找

```
SELECT *
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

如果 orders.user_id 建立唯一索引，且关联条件使用完整的唯一索引键，那么被驱动表可以采用 eq_ref。

每次关联最多匹配一条记录。

#### const：常量查找

```
SELECT *
FROM users
WHERE id = 100;
```

id 是主键，最多匹配一条记录。

### 5. possible_keys

表示优化器认为可能使用的索引。

它只是候选索引，并不代表一定使用。

### 6. key

表示实际使用的索引。

例如：

```
possible_keys: idx_user_id
key: idx_user_id
```

表示实际使用了 idx_user_id。

如果：

```
key: NULL
```

说明当前表访问步骤没有使用索引。

注意：没有使用索引不一定意味着 SQL 有问题。对于数据量很小的表，全表扫描可能成本更低。

### 7. key_len

表示实际使用的索引部分长度，单位是字节。

可以辅助判断联合索引使用了哪些索引列。

例如：

```
CREATE INDEX idx_name_age
ON users(name, age);
```

通过 key_len 可以分析实际使用的索引长度，但需要考虑字段类型、字符集、是否允许 NULL 等因素，不能简单地将长度直接等同于使用的字段数量。

### 8. ref

表示使用什么值与索引进行比较。

例如：

```
table: orders
type: ref
key: idx_user_id
ref: db.users.id
```

表示：

- 使用 orders 表自己的 idx_user_id 索引。
- 使用 users 表的 id 作为查找值。
- 在 orders.user_id 索引中查找对应记录。

ref 可能是常量，也可能是前面访问的表中的字段。

### 9. rows

表示优化器预计需要检查的行数。

例如：

```
table: orders
type: ref
key: idx_user_id
rows: 100
```

表示预计访问约 100 行。

通常在相同查询目标下，扫描行数越少越好。

注意：rows 是估算值，不是实际执行统计值。

### 10. filtered

表示预计经过当前表的条件过滤后，剩余记录的百分比。

例如：

```
rows: 100
filtered: 10.00
```

表示预计 100 行中约有 10% 能通过相关过滤条件。

可以粗略理解为：

100 × 10% = 10 行。

## 三、Extra：额外执行信息

Extra 是 EXPLAIN 中非常值得关注的字段。

### 1. Using index

表示使用覆盖索引。

例如：

```
SELECT user_id
FROM orders
WHERE user_id = 100;
```

如果 user_id 已经在索引中，MySQL 可以直接从索引中获取需要的数据，不需要再回表查询。

### 2. Using where

表示 MySQL 使用 WHERE 条件过滤记录。

它不代表一定没有使用索引。

### 3. Using index condition

表示使用了索引条件下推（ICP）。

MySQL 可以将部分过滤条件下推到存储引擎的索引扫描过程中，减少不必要的回表。

### 4. Using temporary

表示查询执行过程中使用了内部临时表。

常见于：

- GROUP BY。
- DISTINCT。
- UNION。
- 某些 ORDER BY 和 GROUP BY 组合。

并不是所有 GROUP BY 或 ORDER BY 都一定会出现 Using temporary，是否使用取决于执行计划。

### 5. Using filesort

表示 MySQL 需要额外排序，不能直接利用索引顺序完成排序。

注意：

Using filesort 不代表一定使用磁盘文件，它表示采用了额外排序过程。

### 6. Using join buffer

表示 JOIN 使用了连接缓冲区，具体可能涉及块嵌套循环或 Hash Join 等执行机制，需要结合 MySQL 版本和 TREE 执行计划分析。

## 四、JOIN 与索引

JOIN 的 ON 条件可以使用索引。

例如有两张表：

```
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    amount DECIMAL(10, 2)
);
```

创建普通索引：

```
CREATE INDEX idx_user_id
ON orders(user_id);
```

执行：

```
SELECT *
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

### 1. 驱动表和被驱动表

驱动表：先访问的表。

被驱动表：根据驱动表的结果，再去查找匹配记录的表。

假设执行顺序是：

1. 扫描 users。
2. 取出 users.id。
3. 使用 orders.user_id 索引查找订单。
4. 组合匹配结果。

那么：

- users 是驱动表。
- orders 是被驱动表。

注意：驱动表和被驱动表由优化器实际选择的访问顺序决定，不一定等于 SQL 中的左表和右表。

### 2. JOIN 如何使用索引？

典型的索引嵌套循环连接（Index Nested-Loop Join）可以理解为：

```
扫描驱动表 users
        |
        v
取出 users.id
        |
        v
使用 orders.user_id 索引查找
        |
        v
返回匹配的订单
        |
        v
组合查询结果
```

如果被驱动表的关联字段有合适的索引，就可以避免反复全表扫描。

### 3. JOIN 中的 ref 和 eq_ref

核心区别：

| type   | 索引类型               | 每次查找最多匹配 |
| ------ | ---------------------- | ---------------- |
| ref    | 非唯一索引等值查找     | 多条             |
| eq_ref | 主键或唯一索引关联查找 | 一条             |

例如：

```
CREATE INDEX idx_user_id
ON orders(user_id);
```

普通索引允许多个订单拥有相同的 user_id，因此通常是 ref。

如果改成：

```
CREATE UNIQUE INDEX uk_user_id
ON orders(user_id);
```

每个 user_id 最多对应一条订单，在符合 eq_ref 条件时，可以使用 eq_ref。

### 4. EXPLAIN 中究竟看哪张表的索引？

这是 JOIN 分析最重要的知识点。

执行：

```
EXPLAIN
SELECT *
FROM users u
JOIN orders o
ON u.id = o.user_id;
```

假设结果：

| table | type | key         | ref     |
| ----- | ---- | ----------- | ------- |
| u     | ALL  | NULL        | NULL    |
| o     | ref  | idx_user_id | db.u.id |

重点看第二行：

```
table = o
type = ref
key = idx_user_id
ref = db.u.id
```

含义：

- table：当前访问的是 orders 表。
- type：orders 使用 ref 方式访问。
- key：使用 orders 自己的 idx_user_id 索引。
- ref：拿 users.id 的值去 orders.user_id 索引中查找。

因此：

**key 看的是当前目标表使用了自己的哪个索引；ref 看的是用什么值去索引中查找。**

可以记成：

```
users.id = 100
      |
      v
orders.user_id 索引
      |
      v
查找 user_id = 100 的订单
```

### 5. JOIN 两边都必须有索引吗？

不一定。

在索引嵌套循环连接中，如果 users 是驱动表、orders 是被驱动表，那么 orders.user_id 的索引尤其重要。

如果优化器选择了不同的访问顺序，则需要重新分析当前被驱动表的索引情况。

因此，不能只看 SQL 的左表和右表，而要看 EXPLAIN 实际展示的访问顺序。

### 6. JOIN 不一定使用索引

MySQL 还可能使用其他连接算法：

| 算法                   | 特点                                                   |
| ---------------------- | ------------------------------------------------------ |
| Nested Loop Join       | 循环访问驱动表和被驱动表                               |
| Index Nested-Loop Join | 使用被驱动表的索引进行匹配                             |
| Hash Join              | 通过构建哈希表进行关联，适合部分没有合适索引的连接场景 |

MySQL 8.0.18 开始支持 Hash Join。

因此，没有索引不代表 JOIN 无法执行，只是优化器可能选择其他执行方式。

## 五、EXPLAIN ANALYZE：查看真实执行情况

普通 EXPLAIN 主要展示预计执行计划。

如果要查看实际执行情况，可以使用 MySQL 8.0.18 及以上版本的 EXPLAIN ANALYZE。

```
EXPLAIN ANALYZE
SELECT u.id, u.name, o.id, o.amount
FROM users u
JOIN orders o
ON u.id = o.user_id
WHERE u.id = 100;
```

可能得到类似结果：

```
-> Nested loop inner join
   (actual time=0.050..0.120 rows=5 loops=1)

    -> Single-row index lookup on u
       (actual time=0.020..0.025 rows=1 loops=1)

    -> Index lookup on o using idx_user_id
       (actual time=0.030..0.060 rows=5 loops=1)
```

重点关注：

| 指标        | 含义           |
| ----------- | -------------- |
| actual time | 实际执行耗时   |
| rows        | 实际产生的行数 |
| loops       | 执行次数       |

### 1. actual time

例如：

```
actual time=0.030..0.060
```

通常表示：

- 第一个值：产生第一行结果的时间。
- 第二个值：完成该节点输出的时间。

单位是毫秒。

### 2. rows

实际产生的行数，可以与普通 EXPLAIN 中的 rows 估算值比较。

如果预估行数和实际行数差距很大，可能说明统计信息不准确或数据分布估算存在偏差。

### 3. loops

表示执行节点被调用的次数。

在嵌套循环 JOIN 中，被驱动表的索引查找可能被执行多次。

因此，分析 JOIN 时，loops 尤其重要。

## 六、CTE：公共表表达式

CTE（Common Table Expression）是 MySQL 提供的一种查询组织方式。

通过 WITH 定义一个有名字的查询结果，供后续 SQL 使用。

基本语法：

```
WITH cte_name AS (
    SELECT ...
)
SELECT *
FROM cte_name;
```

例如：

```
WITH user_total AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM user_total
WHERE total > 300;
```

这里的 user_total 就是一个 CTE。

需要注意：

CTE 是逻辑上的查询组织方式，不代表一定创建物理临时表。

MySQL 优化器可以选择：

1. Merge（合并）。
2. Materialize（物化）。

## 七、CTE 合并（Merge）

合并指的是将 CTE 的查询块合并到外层查询中，不单独保存 CTE 的查询结果。

例如：

```
WITH temp AS (
    SELECT id, name
    FROM users
    WHERE status = 1
)
SELECT *
FROM temp
WHERE id > 100;
```

如果选择合并，可以理解为：

```
SELECT id, name
FROM users
WHERE status = 1
  AND id > 100;
```

原本有两个查询层次：

```
外层查询
    |
    v
CTE 查询
    |
    v
users 表
```

合并后：

```
一个查询块
    |
    v
users 表
```

这里不是简单拼接 SQL 字符串，而是优化器将查询块合并，让整个查询能够统一优化。

### 合并的好处

1. 不需要单独创建和维护 CTE 的中间结果。
2. 减少中间数据的生成。
3. 优化器可以对整体查询进行优化。
4. 有利于条件下推和索引使用。

### 条件下推

例如：

```
WITH temp AS (
    SELECT *
    FROM users
)
SELECT *
FROM temp
WHERE id = 100;
```

可以将外层条件下推到内层：

```
SELECT *
FROM users
WHERE id = 100;
```

这样可以提前过滤数据，减少不必要的数据读取。

## 八、CTE 物化（Materialization）

物化指的是：

MySQL 先执行 CTE 中的查询，将查询结果保存到内部临时表，然后外层查询再读取这张临时表。

例如：

```
WITH user_total AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM user_total
WHERE total > 300;
```

执行过程：

```
orders 原始订单表
        |
        v
执行 CTE 查询
        |
        v
GROUP BY + SUM
        |
        v
生成中间结果
        |
        v
Materialize
        |
        v
保存到内部临时表
        |
        v
外层查询读取
        |
        v
WHERE total > 300
        |
        v
返回结果
```

### 1. 内部临时表是什么？

它是 MySQL 在执行查询过程中内部创建和管理的临时数据结构。

它不是用户主动执行：

```
CREATE TEMPORARY TABLE ...
```

创建的用户临时表。

内部临时表可能使用内存，也可能在特定条件下使用磁盘。

它的生命周期由 MySQL 查询执行过程管理。

### 2. 什么时候会物化？

MySQL 中，某些查询结构不能被合并，需要采用物化策略。

常见情况：

| SQL 特征 | 示例                |
| -------- | ------------------- |
| 聚合函数 | SUM()、COUNT()      |
| GROUP BY | GROUP BY user_id    |
| DISTINCT | SELECT DISTINCT     |
| HAVING   | HAVING COUNT(*) > 2 |
| LIMIT    | LIMIT 10            |
| UNION    | UNION、UNION ALL    |
| 窗口函数 | ROW_NUMBER()        |

另外：

**递归 CTE 一定会被物化。**

注意，这些情况描述的是阻止合并的条件，不代表每一种临时表都会在传统 EXPLAIN 中显示 Using temporary。

### 3. 物化有什么好处？

物化虽然需要生成中间结果，但不一定比合并差。

例如一个 CTE 被引用多次：

```
WITH temp AS (
    SELECT user_id, COUNT(*) AS total
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM temp a
JOIN temp b
ON a.user_id = b.user_id;
```

物化可以让 CTE 的结果生成一次，再供多个引用使用。

否则重复执行相同的计算逻辑可能带来额外开销。

MySQL 还可能为物化后的临时表自动创建索引，提高后续查询效率。

## 九、Using temporary 和 Materialize 的区别

这是两个容易混淆的概念。

| 概念                   | 含义                        |
| ---------------------- | --------------------------- |
| WITH                   | 定义 CTE                    |
| Merge                  | 将 CTE 查询块合并到外层     |
| Materialize            | 将 CTE 结果保存到内部临时表 |
| Using temporary        | 执行过程中使用了内部临时表  |
| CREATE TEMPORARY TABLE | 用户主动创建临时表          |

它们之间的关系：

- 使用 WITH 不代表一定物化。
- CTE 合并后通常不需要单独物化该 CTE。
- CTE 物化会使用内部临时表。
- 使用内部临时表不一定是因为 CTE，也可能是 GROUP BY、DISTINCT 等其他操作。
- 传统 EXPLAIN 中没有 Using temporary，不代表一定没有发生 CTE 物化。

### 怎么查看 CTE 是否物化？

使用：

```
EXPLAIN FORMAT=TREE
WITH temp AS (
    SELECT user_id, COUNT(*) AS total
    FROM orders
    GROUP BY user_id
)
SELECT *
FROM temp
WHERE total > 3;
```

重点观察执行计划中是否出现：

```
Materialize CTE temp
```

如果出现，就表示 CTE 被物化。

如果 CTE 被合并，则通常会直接看到对原始表的访问，而没有独立的 CTE 物化节点。

## 十、实际 SQL 性能排查流程

当发现 SQL 查询比较慢时，可以按照下面的流程分析。

### 第一步：查看表的索引

```
SHOW INDEX FROM users;
SHOW INDEX FROM orders;
```

确认关联字段和查询条件是否有合适的索引。

### 第二步：查看执行计划

```
EXPLAIN
SELECT u.id, u.name, o.id, o.amount
FROM users u
JOIN orders o
ON u.id = o.user_id
WHERE u.id = 100;
```

### 第三步：检查访问类型

重点关注 type：

- ALL：是否存在不必要的全表扫描。
- index：是否扫描了整个索引。
- range：是否合理使用索引范围扫描。
- ref：是否使用普通索引查找。
- eq_ref：是否使用唯一索引关联。

### 第四步：检查索引

重点关注：

- possible_keys：可能使用的索引。
- key：实际使用的索引。
- key_len：索引使用长度。
- ref：使用什么值查找索引。

### 第五步：检查扫描行数

重点关注 rows 和 filtered。

如果估算扫描行数很大，需要考虑索引选择性、过滤条件和查询方式。

### 第六步：检查 Extra

重点关注：

- Using index：是否使用覆盖索引。
- Using index condition：是否使用索引条件下推。
- Using temporary：是否使用内部临时表。
- Using filesort：是否需要额外排序。

### 第七步：查看实际执行情况

```
EXPLAIN ANALYZE
SELECT ...;
```

对比预估行数与实际行数，查看 actual time 和 loops。

## 十一、常见面试问题

### 1. JOIN 的 ON 条件可以使用索引吗？

可以。

MySQL 可以使用被驱动表的连接字段索引快速查找匹配记录，避免反复全表扫描。

### 2. ref 和 eq_ref 有什么区别？

ref 通常使用非唯一索引查找，可能返回多条记录。

eq_ref 通常使用主键或唯一索引进行关联查找，每次最多匹配一条记录。

### 3. EXPLAIN 中 JOIN 的 key 看哪张表？

看当前 EXPLAIN 行对应的表。

例如 orders 这一行的 key，就表示 orders 表实际使用的索引。

### 4. EXPLAIN 中的 ref 是什么意思？

表示索引查找使用的值。

例如：

```
key = idx_user_id
ref = db.users.id
```

表示使用 users.id 的值去 orders.user_id 索引中查找。

### 5. rows 是实际扫描行数吗？

不是。

普通 EXPLAIN 中的 rows 是优化器估算值。

需要使用 EXPLAIN ANALYZE 查看实际执行情况。

### 6. Using temporary 一定代表 WITH 吗？

不是。

Using temporary 表示执行过程中使用内部临时表。

WITH 是定义 CTE 的语法，两者不是同一个概念。

### 7. 使用 WITH 一定会物化吗？

不是。

MySQL 可以选择 Merge 或 Materialize。

### 8. 物化一定比合并慢吗？

不是。

物化有创建和保存中间结果的开销，但如果 CTE 被多次引用，可以复用计算结果，减少重复执行。

### 9. Using filesort 是磁盘排序吗？

不一定。

它表示 MySQL 使用额外排序过程，不代表一定发生磁盘 I/O。

### 10. key 为 NULL 一定代表 SQL 有问题吗？

不一定。

可能是表数据量很小，也可能是优化器认为全表扫描成本更低，还可能是索引选择性较差或者查询条件无法有效利用索引。

## 十二、最终知识框架

### EXPLAIN

核心用途：分析 SQL 执行计划。

重点字段：

- type：访问方式。
- key：使用的索引。
- ref：索引查找使用的值。
- rows：预计扫描行数。
- filtered：预计过滤比例。
- Extra：额外执行信息。

### JOIN

核心用途：多表关联。

重点知识：

- 驱动表：先访问的表。
- 被驱动表：根据驱动表结果进行关联的表。
- ref：普通索引查找。
- eq_ref：唯一索引关联查找。
- key：当前表自己的索引。
- ref：使用什么值查找当前表的索引。

### EXPLAIN ANALYZE

核心用途：查看真实执行情况。

重点指标：

- actual time：实际执行耗时。
- rows：实际产生的行数。
- loops：节点执行次数。

### CTE

核心用途：组织复杂查询。

两种执行策略：

- Merge：将 CTE 查询块合并到外层查询。
- Materialize：将 CTE 结果保存到内部临时表。

### 内部临时表

核心用途：保存查询执行过程中的中间结果。

常见场景：

- CTE 物化。
- GROUP BY。
- DISTINCT。
- UNION。
- 部分排序和分组操作。

最终记忆：

**EXPLAIN 看执行计划，type 看访问方式，key 看当前表使用的索引，ref 看索引查找值，rows 看预估扫描行数，Extra 看额外执行操作。JOIN 重点关注被驱动表的索引使用情况；CTE 则要区分合并和物化。**