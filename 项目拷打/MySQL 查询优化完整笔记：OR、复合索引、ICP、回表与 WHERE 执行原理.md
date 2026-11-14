# MySQL 查询优化完整笔记：OR、复合索引、ICP、回表与 WHERE 执行原理

> 学习目标：通过校园活动查询优化案例，理解 MySQL 为什么放弃索引、复合索引如何减少扫描量、索引条件下推如何减少回表，以及 `WHERE` 和 `JOIN` 的逻辑执行顺序与实际执行过程。适用于 Java 后端开发、项目性能优化和 MySQL 面试。

## 一、案例背景：校园活动查询优化

### 1.1 业务场景

校园活动系统有两张主要数据表。

`activity` 是活动表，保存活动创建者、时间、类别和状态等信息。

| id   | creator_id | start_time | category | status |
| ---- | ---------- | ---------- | -------- | ------ |
| 101  | 1          | 2026-09-05 | 学习     | OPEN   |
| 102  | 2          | 2026-09-08 | 运动     | OPEN   |
| 103  | 1          | 2026-10-12 | 社交     | OPEN   |
| 104  | 3          | 2026-11-01 | 学习     | CLOSED |

`activity_member` 是活动报名表，用于维护用户与活动的多对多关系。

| activity_id | user_id |
| ----------- | ------- |
| 102         | 1       |
| 103         | 1       |
| 104         | 1       |

业务需求是查询用户 1 在指定时间段内参与的所有活动，包括用户自己创建的活动，以及报名参加的活动。

最初采用一条包含 `OR EXISTS` 的 SQL：

```
SELECT
    a.id,
    a.creator_id,
    a.category,
    a.start_time,
    a.end_time,
    a.status
FROM activity a
WHERE (
    a.creator_id = 1
    OR EXISTS (
        SELECT 1
        FROM activity_member m
        WHERE m.activity_id = a.id
          AND m.user_id = 1
    )
)
AND a.start_time >= '2026-09-01'
AND a.start_time < '2026-10-31';
```

这里的 `OR` 表示两个条件满足任意一个即可，`EXISTS` 判断报名表中是否存在对应记录。

注意：`start_time < '2026-10-31'` 不包含 10 月 31 日。如果业务需要包含整个月，应改为 `start_time < '2026-11-01'`。

### 1.2 优化前的问题

原始查询的执行计划大致为：

| EXPLAIN 字段 | 值     | 含义                 |
| ------------ | ------ | -------------------- |
| type         | ALL    | 全表扫描             |
| key          | NULL   | 没有使用索引         |
| rows         | 20,000 | 预计检查约 20,000 行 |

当时活动表只有 `idx_place_id` 和 `idx_start_time` 等索引，缺少针对创建者的索引。

同时，时间范围覆盖了大约 60% 的数据，因此单独通过 `idx_start_time` 读取数据可能并不划算。

最终，优化器选择全表扫描。

这里必须区分两个概念：索引失效和优化器未选择索引。 实际上，索引可能完全可用，但由于成本估算较高，优化器仍然选择全表扫描。

## 二、为什么 OR 可能导致索引失效？

### 2.1 OR 的基本原理

考虑以下 SQL：

```
SELECT *
FROM activity
WHERE creator_id = 1
   OR place_id = 100;
```

这表示查询所有由用户 1 创建的活动，或者地点为 100 的活动。

假设只有 `creator_id` 存在索引，那么 MySQL 虽然可以通过索引找到用户 1 创建的活动，却不能直接排除其他活动，因为其他用户创建的活动也可能满足 `place_id = 100`。

因此，只使用 `creator_id` 索引无法得到完整结果集。

MySQL 需要考虑其他执行方案，例如全表扫描。

但这并不意味着只要出现 `OR`，索引就一定失效。

### 2.2 两边都有索引时，可能使用 Index Merge

假设建立两个索引：

```
CREATE INDEX idx_creator
ON activity (creator_id);
CREATE INDEX idx_place
ON activity (place_id);
```

MySQL 有机会使用 Index Merge（索引合并）分别查询两个索引，再合并结果。

```
                 OR 查询
                    |
          +---------+---------+
          |                   |
     idx_creator         idx_place
          |                   |
      101,103,105          102,103
          |                   |
          +---------+---------+
                    |
                合并去重
                    |
            101,102,103,105
                    |
                 按需回表
```

执行计划可能显示：

```
type: index_merge
key: idx_creator,idx_place
Extra: Using union(...)
```

但 Index Merge 只针对同一张表上的索引扫描，并不是跨表的索引合并。

此外，即使两个字段都有索引，MySQL 也不保证一定使用 Index Merge。如果估算认为全表扫描成本更低，仍然可能使用 `ALL`。

### 2.3 为什么 OR EXISTS 更难优化？

原 SQL 的两个条件分别是：

```
a.creator_id = 1
```

以及：

```
EXISTS (
    SELECT 1
    FROM activity_member m
    WHERE m.activity_id = a.id
      AND m.user_id = 1
)
```

第二个条件是相关子查询，因为它引用了外层查询的 `a.id`。

它的逻辑含义是：对当前活动，判断用户 1 是否存在报名关系。

MySQL 有多种子查询优化机制，例如半连接（Semijoin）转换，但 `EXISTS` 位于 `OR` 表达式内部时，不满足常规半连接转换要求的顶层条件位置，因此可能限制优化器采用高效执行方案。

原查询也不能简单通过 `activity.creator_id` 索引完成，因为还有一部分结果需要通过报名关系获取。

当查询最终采用全表扫描时，相关子查询可能会被反复执行。但不能只凭 SQL 文本就断言一定执行了 20,000 次子查询，具体循环次数要通过 `EXPLAIN ANALYZE` 验证。

### 2.4 OR 与 AND 的区别

对于：

```
WHERE creator_id = 1
  AND start_time >= '2026-09-01'
```

两个条件必须同时成立，MySQL 可以利用复合索引逐步缩小候选范围。

而对于：

```
WHERE creator_id = 1
   OR start_time >= '2026-09-01'
```

两个条件满足任意一个即可。如果只扫描其中一个条件的索引，会遗漏满足另一个条件的记录。

因此，`AND` 和 `OR` 对索引访问路径的要求不同。

不过，AND 并不保证使用索引，OR 也不必然导致索引失效，最终都取决于索引结构、数据分布和优化器的成本估算。

## 三、索引选择性：为什么有索引却不用？

### 3.1 什么是选择性？

索引选择性可以衡量一个条件过滤数据的能力。

对于具体查询条件，常用匹配比例来理解：

\[ \text{匹配比例}=\frac{\text{满足条件的记录数}}{\text{总记录数}} \]

匹配比例越低，说明能够排除的数据越多，该条件通常越适合作为索引访问入口。

案例中活动表有 20,000 行，用户 1 创建的活动约 100 行：

\[ \frac{100}{20000}=0.5\% \]

说明 `creator_id = 1` 可以过滤掉约 99.5% 的数据。

而时间范围覆盖约 60%：

\[ \frac{12000}{20000}=60\% \]

说明单独依靠时间索引，仍然可能需要处理 12,000 条记录。

因此，创建者条件的过滤能力远强于时间条件。

这里讨论的是具体谓词的匹配比例。数据库统计学中还常用 `COUNT(DISTINCT column) / COUNT(*)` 等指标描述列的区分度，不应与某个查询值的实际匹配比例完全混为一谈。

### 3.2 为什么扫描 60% 的数据可能不如全表扫描？

InnoDB 普通二级索引的叶子节点通常保存索引字段和主键值，并非完整数据行。

如果根据时间索引找到 12,000 条记录，而查询又需要不在索引中的业务字段，就可能涉及大量聚簇索引查找。

对于高比例匹配数据的查询，二级索引扫描加回表的总成本有时会高于直接扫描聚簇索引。

因此优化器可能主动放弃时间索引。

不过，这不是固定阈值。能否使用索引还取决于是否覆盖查询、数据页缓存情况、记录宽度、统计信息及查询成本。

## 四、复合索引与最左前缀原则

### 4.1 为什么建立 `(creator_id, start_time)`？

优化后的创建活动查询：

```
SELECT
    id,
    creator_id,
    category,
    start_time,
    end_time,
    status
FROM activity
WHERE creator_id = 1
  AND start_time >= '2026-09-01'
  AND start_time < '2026-10-31';
```

建立复合索引：

```
ALTER TABLE activity
ADD KEY idx_creator_start (
    creator_id,
    start_time
);
```

InnoDB 使用 B+ 树组织索引记录。对于复合索引 `(creator_id, start_time)`，索引键首先按照 `creator_id` 排序，同一创建者内部再按照 `start_time` 排序。

逻辑排列如下：

```
creator_id | start_time  | id
-----------+-------------+----
1          | 2026-08-10  | 11
1          | 2026-09-05  | 101
1          | 2026-09-19  | 109
1          | 2026-10-12  | 103
1          | 2026-11-09  | 135
2          | 2026-09-08  | 102
2          | 2026-10-04  | 120
3          | 2026-09-11  | 130
```

MySQL 可以先在索引中定位 `creator_id = 1` 的区间，然后在该区间内进行时间范围扫描。

这就是等值列在前、范围列在后的典型索引设计方式。

### 4.2 什么是最左前缀原则？

假设建立了：

```
CREATE INDEX idx_user_time_status
ON activity (
    creator_id,
    start_time,
    status
);
```

典型情况如下：

| WHERE 条件                                                | 索引使用情况                                                 |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `creator_id = 1`                                          | 可利用第一列定位                                             |
| `creator_id = 1 AND start_time > ...`                     | 可利用前两列缩小范围                                         |
| `creator_id = 1 AND start_time > ... AND status = 'OPEN'` | 前两列可定位范围，第三列还可能用于索引层过滤                 |
| `start_time > ...`                                        | 无法按普通最左前缀直接定位第一列，部分版本及条件下可能使用 Skip Scan |
| `status = 'OPEN'`                                         | 通常不能直接利用该复合索引高效定位                           |

关键点是：复合索引按照列的定义顺序形成字典序排序，因此连续匹配左侧字段通常更有利于高效定位。

### 4.3 为什么范围列后的字段可能不能继续缩小扫描范围？

考虑：

```
WHERE creator_id = 1
  AND start_time >= '2026-09-01'
  AND start_time < '2026-10-31'
  AND status = 'OPEN'
```

对于 `(creator_id, start_time, status)`，MySQL 能利用前两个条件确定索引扫描区间。

但因为 `start_time` 已经是一个连续范围，在这个范围内 `status` 并不一定整体连续，所以通常无法继续用 `status` 把扫描范围缩小成单独的连续索引区间。

不过，第三列仍然可能在索引层参与过滤。这就涉及索引条件下推。

## 五、索引条件下推 ICP：为什么能够减少回表？

### 5.1 什么是回表？

InnoDB 中常见两类索引：

- 聚簇索引（Clustered Index）：通常由主键构建，叶子节点存放完整数据行。
- 二级索引（Secondary Index）：叶子记录保存索引键和主键值，通常不包含所有业务字段。

例如有一个二级索引：

```
CREATE INDEX idx_creator_start
ON activity (creator_id, start_time);
```

当 SQL 需要查询：

```
SELECT id, creator_id, start_time, category
FROM activity
WHERE creator_id = 1;
```

二级索引中可以读取 `creator_id`、`start_time` 和主键 `id`，但 `category` 不在索引中。

MySQL 就需要根据主键继续访问聚簇索引，读取 `category`。

```
       二级索引 idx_creator_start
                  |
       creator_id = 1
       start_time = 2026-09-05
       id = 101
                  |
                  v
         根据主键 id = 101
                  |
                  v
           聚簇索引查找
                  |
       获取 category 等字段
```

这就是回表。

### 5.2 什么是 ICP？

ICP 的全称是 Index Condition Pushdown，即索引条件下推。

它的核心思路是：

在通过二级索引扫描记录时，先利用索引中已有的字段过滤不符合条件的记录，再决定是否访问完整数据行。

假设索引为：

```
CREATE INDEX idx_creator_start_status
ON activity (
    creator_id,
    start_time,
    status
);
```

查询如下：

```
SELECT id, creator_id, start_time, status, category
FROM activity
WHERE creator_id = 1
  AND start_time >= '2026-09-01'
  AND start_time < '2026-10-31'
  AND status = 'OPEN';
```

假设通过前两项条件找到 100 条索引记录，其中只有 10 条满足 `status = 'OPEN'`。

未使用 ICP 时，简化流程是：

```
扫描 100 条二级索引记录
              |
              v
        回表读取 100 行
              |
              v
        过滤 status
              |
              v
         保留 10 行
```

使用 ICP 后：

```
扫描 100 条二级索引记录
              |
              v
      在索引中检查 status
              |
              v
        剩余 10 条记录
              |
              v
         回表读取 10 行
```

ICP 并不一定减少最初的索引扫描范围，而是通过提前过滤减少不必要的完整行读取。

以上是简化模型，实际回表行为还可能受到 MVCC 可见性判断等因素影响。

### 5.3 怎么确认 ICP 是否生效？

执行：

```
EXPLAIN
SELECT id, creator_id, start_time, status, category
FROM activity
WHERE creator_id = 1
  AND start_time >= '2026-09-01'
  AND start_time < '2026-10-31'
  AND status = 'OPEN';
```

如果 `Extra` 中显示：

```
Using index condition
```

说明执行计划使用了索引条件下推。

注意，出现范围查询不等于自动使用 ICP。必须有适合在索引层检查的剩余条件，而且查询确实需要读取完整记录。

## 六、ICP 与覆盖索引的区别

这两个概念容易混淆，但作用完全不同。

### 6.1 覆盖索引是什么？

如果查询需要的所有字段都能直接从索引中获取，就称该索引覆盖了查询。

例如：

```
CREATE INDEX idx_creator_start_status
ON activity (
    creator_id,
    start_time,
    status
);
```

查询：

```
SELECT id, creator_id, start_time, status
FROM activity
WHERE creator_id = 1
  AND start_time >= '2026-09-01'
  AND start_time < '2026-10-31'
  AND status = 'OPEN';
```

需要的字段都包含在索引中，`id` 也作为 InnoDB 二级索引中的主键值存在。

因此，这个查询可能直接从二级索引获取全部需要的列，无需正常的回表取列过程。

`EXPLAIN` 的 `Extra` 中可能出现：

```
Using where; Using index
```

其中 `Using index` 通常表示采用了覆盖索引访问。

### 6.2 对比

| 对比项       | ICP                      | 覆盖索引                     |
| ------------ | ------------------------ | ---------------------------- |
| 完整名称     | Index Condition Pushdown | Covering Index               |
| 核心作用     | 在索引层提前过滤         | 直接从索引获取所需列         |
| 是否回表     | 仍可能回表，但减少次数   | 通常不需要回表               |
| 典型执行计划 | `Using index condition`  | `Using index`                |
| 优化目标     | 减少无效的数据行访问     | 避免为获取字段而访问聚簇索引 |

可以记成一句话：

ICP 是减少回表，覆盖索引是尽可能避免回表。

## 七、WHERE 的本质是什么？

### 7.1 SQL 逻辑执行顺序

从 SQL 逻辑语义上看，`WHERE` 是对 `FROM`、`JOIN` 等操作得到的关系结果进行行过滤。

典型逻辑执行顺序为：

```
FROM
  |
  v
JOIN ... ON
  |
  v
WHERE
  |
  v
GROUP BY
  |
  v
HAVING
  |
  v
SELECT
  |
  v
DISTINCT
  |
  v
ORDER BY
  |
  v
LIMIT
```

注意：这只是一个理解查询逻辑的常见模型，不代表数据库内部每个查询都严格按照这个顺序物理执行。

从关系代数的角度，`WHERE` 可以理解为选择操作（Selection），用于保留满足谓词条件的记录。

例如：

```
SELECT a.id, m.user_id
FROM activity a
JOIN activity_member m
  ON a.id = m.activity_id
WHERE m.user_id = 1;
```

逻辑上可以理解为：

1. 根据 `FROM` 找到活动表。
2. 根据 `JOIN ... ON` 关联报名表。
3. 使用 `WHERE` 保留 `user_id = 1` 的记录。
4. 使用 `SELECT` 返回指定字段。

这里要注意 SQL 的三值逻辑：`WHERE` 只保留条件结果为 `TRUE` 的行，`FALSE` 和 `UNKNOWN` 都会被过滤。

### 7.2 逻辑执行顺序不等于物理执行顺序

如果数据库真的把所有活动和报名记录完整 JOIN 后才过滤，可能产生大量不必要的计算。

因此，MySQL 优化器会尝试提前执行部分过滤条件。

例如：

```
SELECT a.id, m.user_id
FROM activity a
JOIN activity_member m
  ON a.id = m.activity_id
WHERE m.user_id = 1
  AND a.status = 'OPEN';
```

优化器可能选择下面的访问路径：

```
通过 idx_user_id 找到用户 1 的报名记录
                    |
                    v
             获取 activity_id
                    |
                    v
          根据 activity 主键关联
                    |
                    v
              检查 status
                    |
                    v
               返回结果
```

它并不需要先生成一个包含全部报名用户的大型 JOIN 结果。

这属于查询优化中的谓词下推和 JOIN 顺序优化。

### 7.3 谓词下推与 ICP 是不是一回事？

不是。

谓词下推（Predicate Pushdown）是一个更广泛的查询优化概念，指尽可能把过滤条件提前到更接近数据源的位置执行，减少后续操作的数据量。

ICP 则是针对索引扫描的一种具体优化，它将可由索引记录判断的条件交给存储引擎提前执行，以减少完整行读取。

可以这样区分：

| 概念             | 主要作用位置           | 优化目的               |
| ---------------- | ---------------------- | ---------------------- |
| WHERE 谓词下推   | JOIN 前或数据访问阶段  | 减少后续参与计算的数据 |
| ICP 索引条件下推 | 存储引擎的索引扫描阶段 | 减少回表               |
| 复合索引范围定位 | B+ 树访问阶段          | 减少索引扫描范围       |

## 八、WHERE 和 JOIN ON 有什么区别？

### 8.1 INNER JOIN 中的区别

考虑：

```
SELECT *
FROM activity a
INNER JOIN activity_member m
  ON a.id = m.activity_id
WHERE m.user_id = 1;
```

对于内连接，许多单纯的条件可以放在 `ON` 或 `WHERE` 中，并且得到相同结果。

例如：

```
SELECT *
FROM activity a
INNER JOIN activity_member m
  ON a.id = m.activity_id
 AND m.user_id = 1;
```

这两个查询通常在结果上等价，优化器也经常可以把它们转换为相同或类似的执行计划。

但对于外连接，情况不同。

### 8.2 LEFT JOIN 中的区别

假设两张表的数据如下。

活动表：

| id   | name   |
| ---- | ------ |
| 101  | 篮球   |
| 102  | 跑步   |
| 103  | 羽毛球 |

报名表：

| activity_id | user_id |
| ----------- | ------- |
| 101         | 1       |
| 102         | 2       |

第一种查询：

```
SELECT a.id, m.user_id
FROM activity a
LEFT JOIN activity_member m
  ON a.id = m.activity_id
 AND m.user_id = 1;
```

结果：

| id   | user_id |
| ---- | ------- |
| 101  | 1       |
| 102  | NULL    |
| 103  | NULL    |

原因是 `LEFT JOIN` 保留左表的全部记录，`ON` 条件用于判断右表中哪些记录能够匹配。

即使没有匹配的报名记录，左表活动仍然保留，对应的右表字段填充为 `NULL`。

第二种查询：

```
SELECT a.id, m.user_id
FROM activity a
LEFT JOIN activity_member m
  ON a.id = m.activity_id
WHERE m.user_id = 1;
```

结果：

| id   | user_id |
| ---- | ------- |
| 101  | 1       |

因为 `WHERE m.user_id = 1` 会排除 `NULL`，因此没有匹配报名记录的活动也被过滤掉了。

在这个例子中，第二种查询最终就具有内连接的效果。

结论：对于 LEFT JOIN，ON 决定右表如何匹配，WHERE 决定连接结果中的哪些行最终保留。

还需要注意，优化器不能随意把外连接的 `WHERE` 条件提前到 `ON`，否则可能改变结果。只有满足语义等价条件时才能进行相应转换。

## 九、项目中为什么要拆分 OR 查询？

### 9.1 优化前的问题

原 SQL：

```
SELECT a.*
FROM activity a
WHERE (
    a.creator_id = 1
    OR EXISTS (
        SELECT 1
        FROM activity_member m
        WHERE m.activity_id = a.id
          AND m.user_id = 1
    )
)
AND a.start_time >= '2026-09-01'
AND a.start_time < '2026-10-31';
```

它同时包含两种业务关系：

- 用户创建了活动。
- 用户报名参加了活动。

这两种关系的最佳数据访问路径不同。

### 9.2 优化后的第一条查询：我创建的活动

```
SELECT a.*
FROM activity a
WHERE a.creator_id = 1
  AND a.start_time >= '2026-09-01'
  AND a.start_time < '2026-10-31';
```

索引：

```
CREATE INDEX idx_creator_start
ON activity (creator_id, start_time);
```

可以直接按照创建者和活动时间定位记录。

### 9.3 优化后的第二条查询：我报名的活动

```
SELECT a.*
FROM activity_member m
JOIN activity a
  ON a.id = m.activity_id
WHERE m.user_id = 1
  AND a.start_time >= '2026-09-01'
  AND a.start_time < '2026-10-31';
```

这条查询主要依靠报名表上的用户索引。

假设存在：

```
CREATE INDEX idx_user_id
ON activity_member (user_id);
```

MySQL 可以先查询用户 1 的报名记录，再根据 `activity_id` 访问活动表主键。

如果报名表经常执行这类查询，也可以考虑：

```
CREATE INDEX idx_user_activity
ON activity_member (user_id, activity_id);
```

该索引能够按照用户定位报名记录，并直接从索引获取活动 ID。

不过，新增索引需要考虑写入成本及与既有索引的重复问题。

### 9.4 拆分后如何保证结果正确？

因为用户可能既是活动创建者，又报名了自己的活动，所以两条查询的结果可能存在重复。

如果应用层分别执行两条查询，就需要使用活动 ID 去重。

如果希望通过一条 SQL 合并结果，可以使用 `UNION`：

```
SELECT a.*
FROM activity a
WHERE a.creator_id = 1
  AND a.start_time >= '2026-09-01'
  AND a.start_time < '2026-10-31'

UNION

SELECT a.*
FROM activity_member m
JOIN activity a
  ON a.id = m.activity_id
WHERE m.user_id = 1
  AND a.start_time >= '2026-09-01'
  AND a.start_time < '2026-10-31';
```

`UNION` 默认去重，`UNION ALL` 不去重。因此如果希望保留原来 `OR EXISTS` 的去重语义，需要使用 `UNION` 或另外进行去重。

但 `UNION` 本身也有去重成本，是否比两次查询更高效仍然需要实测。

## 十、怎么使用 EXPLAIN 判断索引是否生效？

### 10.1 EXPLAIN 关键字段

| 字段          | 含义             | 关注点               |
| ------------- | ---------------- | -------------------- |
| type          | 表访问方式       | 是否全表扫描         |
| possible_keys | 候选索引         | 有哪些索引可能被使用 |
| key           | 实际选用的索引   | 是否使用预期索引     |
| key_len       | 使用的索引键长度 | 辅助判断索引使用情况 |
| rows          | 估计检查的行数   | 预计扫描量           |
| filtered      | 估计条件过滤比例 | 过滤能力             |
| Extra         | 额外执行信息     | ICP、覆盖索引等      |

常见的 `type` 类型包括：

| type        | 含义                               |
| ----------- | ---------------------------------- |
| ALL         | 全表扫描                           |
| index       | 扫描整个索引                       |
| range       | 索引范围扫描                       |
| ref         | 通过非唯一索引等值查找             |
| eq_ref      | 连接时通过唯一键匹配一行           |
| const       | 通过主键或唯一键等值条件定位常量表 |
| index_merge | 合并多个索引扫描结果               |

不能单纯认为 `index` 就一定比 `ALL` 快，也不能只通过 `type` 判断查询的最终性能。

### 10.2 EXPLAIN 中的 rows 是实际行数吗？

不是。

普通 `EXPLAIN` 的 `rows` 表示优化器预估需要检查的行数。

例如：

```
type: range
key: idx_creator_start
rows: 1
```

这不表示实际只读取了 1 条记录。

如果实际返回约 100 条，而优化器估算为 1 条，就说明该估计可能存在偏差，或者两组数字描述的不是同一个执行节点、同一种统计指标。

要确认真实扫描情况，应结合 `EXPLAIN ANALYZE`。

### 10.3 EXPLAIN ANALYZE

MySQL 8.0.18 开始支持 `EXPLAIN ANALYZE`。

示例：

```
EXPLAIN ANALYZE
SELECT a.*
FROM activity a
WHERE a.creator_id = 1
  AND a.start_time >= '2026-09-01'
  AND a.start_time < '2026-10-31';
```

它实际执行查询，并展示相关执行计划节点的耗时、实际返回行数以及循环次数等信息。

因此，可以通过它判断查询到底扫描了多少记录、哪些节点执行时间较长、相关子查询执行了多少次。

注意，`EXPLAIN ANALYZE` 的节点耗时不等于 Java 应用看到的完整请求耗时，网络传输、MyBatis 映射及应用逻辑等还需要单独测量。

## 十一、为什么扫描量减少 200 倍，耗时只提升约 26 倍？

原案例中：

| 指标                 | 优化前  | 优化后     |
| -------------------- | ------- | ---------- |
| 估计或描述的扫描规模 | 20,000  | 约 100     |
| 查询耗时             | 3.50 ms | 0.13 ms    |
| 性能提升             | —       | 约 26.9 倍 |

假设扫描规模从 20,000 条降到 100 条，则数量减少 200 倍。

但查询执行耗时并不只取决于记录扫描数量，还包括索引遍历、条件判断、子查询或连接执行、回表及结果构建等开销。

若测量的是 Java 应用端到端耗时，还可能包含数据库通信、结果传输与对象映射等固定成本。

因此，扫描量降低 200 倍，并不意味着耗时也一定降低 200 倍。

此外，当 Buffer Pool 已经缓存相关数据页时，全表扫描也可能相当快，优化后的绝对收益就不会与扫描行数比例完全一致。

### 11.1 原报告中关于回表的一个错误

原文认为：

> 回表从 20,000 次降低到了约 100 次。

这并不准确。

如果原始执行计划是：

```
type = ALL
```

它代表直接扫描表数据。对于 InnoDB，通常是扫描聚簇索引中的完整记录，而不是先扫描 20,000 条二级索引记录，再逐条回表。

因此，优化前并不能直接认为发生了 20,000 次回表。

更准确的表述应该是：

原方案需要扫描约 20,000 条表记录；优化后利用复合索引缩小扫描范围，仅对少量候选记录按需回表，从而显著降低总体数据访问成本。

### 11.2 为什么地点热度查询只提升 3 倍？

地点热度查询的测试数据：

| 指标         | 优化前   | 优化后    |
| ------------ | -------- | --------- |
| 报告扫描行数 | 50,201   | 368       |
| p50          | 3.207 ms | 1.057 ms  |
| 提升         | —        | 约 3.0 倍 |

旧索引：

```
(map_id, place_id)
```

新索引：

```
(map_id, created_at)
```

查询条件主要是：

```
WHERE map_id = ?
  AND created_at >= ?
  AND created_at < ?
```

新索引可以直接通过 `map_id` 等值匹配和 `created_at` 范围扫描筛选备注记录，而不必先读取某张地图下的大量备注再按时间过滤。

但是地点热度查询还涉及关联查询、分组及去重统计，例如：

```
GROUP BY place_id
COUNT(DISTINCT ...)
```

这些操作没有因为新增索引而全部消失。

因此，虽然索引扫描量明显降低，整个查询的耗时只提升约 3 倍。

## 十二、完整的 MySQL 慢查询优化流程

面对真实项目中的慢 SQL，可以按照以下流程处理。

第一步，确认 SQL 和业务语义。 明确查询最终需要哪些记录，是否存在重复，日期边界是什么，以及能否拆分查询而不改变结果。

第二步，查看执行计划。 使用：

```
EXPLAIN
SELECT ...
```

关注 `type`、`key`、`rows`、`Extra` 等字段，初步定位是否存在全表扫描、索引使用不足或关联效率低下的问题。

第三步，分析数据分布。 例如：

```
SELECT COUNT(*)
FROM activity;
```

查看总记录数。

```
SELECT COUNT(*)
FROM activity
WHERE creator_id = 1;
```

查看指定用户的记录数量。

```
SELECT COUNT(*)
FROM activity
WHERE start_time >= '2026-09-01'
  AND start_time < '2026-10-31';
```

查看时间条件的匹配比例。

根据实际数据，判断哪些字段的过滤能力更强。

第四步，设计索引与 SQL。 例如根据创建者等值条件和时间范围建立 `(creator_id, start_time)`，必要时将不同业务访问路径拆分。

第五步，检查回表和 JOIN。 判断是否需要覆盖索引，是否有适合 ICP 的条件，能否通过已有索引提前缩小 JOIN 输入量。

第六步，用 EXPLAIN ANALYZE 验证。 不仅查看优化器估算，也检查实际行数、循环次数以及执行时间。

第七步，进行重复压测。 在相同数据量、缓存状态及并发条件下对比优化前后的 p50、p95 和平均耗时，避免通过单次 SQL 执行得出不可靠结论。

此外，新增复合索引会增加索引存储与写入维护成本，因此必须结合真实查询频率和写入负载决定是否保留。

## 十三、MySQL 面试常见追问

### 1. OR 一定会导致索引失效吗？

不一定。两个 `OR` 条件如果都能利用同一张表的索引，MySQL 可能通过 Index Merge 合并结果；如果只有部分条件能够高效使用索引，或者索引扫描及结果合并的成本较高，优化器可能选择全表扫描。对于 `OR EXISTS`，还需要考虑相关子查询和半连接优化限制。

### 2. 范围查询之后的索引列一定失效吗？

不一定。范围列后面的字段可能无法继续缩小连续索引扫描区间，但仍然可以通过 ICP 等机制在索引层参与过滤。

### 3. ICP 和覆盖索引有什么区别？

ICP 是在回表之前利用索引字段提前过滤，减少不必要的回表；覆盖索引则是查询需要的字段都能在索引中获取，通常无需为了获取这些字段而回表。

### 4. WHERE 是 JOIN 之后执行的吗？

从 SQL 逻辑语义上讲，`WHERE` 是过滤 JOIN 后的结果。但物理执行时，优化器可能将条件提前到表访问或 JOIN 之前，以减少实际处理的数据量。外连接场景还必须考虑语义等价性，不能随意移动过滤条件。

### 5. 为什么建立索引后 MySQL 还是全表扫描？

可能是因为条件选择性较低、返回的数据比例较大、索引不覆盖查询需要的字段、回表成本较高，或者优化器统计信息存在偏差。不能仅依据索引是否存在判断它会不会被使用。

### 6. 如何确认 SQL 优化确实有效？

先通过 `EXPLAIN` 判断访问路径是否改善，再通过 `EXPLAIN ANALYZE` 检查实际行数和各节点耗时，最后在一致的测试条件下重复测量查询耗时与响应延迟。

## 十四、知识点关系总结

```
                SQL 查询
                   |
                   v
            SQL 语义与查询改写
                   |
          OR / EXISTS / JOIN
                   |
                   v
            MySQL 查询优化器
                   |
           根据成本选择执行计划
                   |
           +-------+-------+
           |               |
        全表扫描        索引访问
                           |
                           v
                     B+ 树定位
                           |
                    复合索引最左匹配
                           |
                    缩小扫描范围
                           |
                           v
                    索引条件下推 ICP
                           |
                    过滤候选记录
                           |
                           v
                   是否需要完整字段
                           |
                  +--------+--------+
                  |                 |
                不需要             需要
                  |                 |
               覆盖索引           按需回表
                  |                 |
                  +--------+--------+
                           |
                           v
                     JOIN / 聚合等
                           |
                           v
                       返回结果
```

这是一张帮助理解概念的示意图，并非 MySQL 对每条 SQL 都严格执行的固定顺序。

### 最终需要记住的五句话

1. OR 不会必然导致索引失效。 是否使用索引，取决于条件可索引性、可选执行路径和优化器的成本判断。
2. 复合索引优先考虑等值匹配与范围匹配的组合。 高选择性等值列在前、范围列在后，是常见且有效的设计，但并非适用于所有查询的绝对规则。
3. ICP 减少回表，覆盖索引避免常规回表。 范围列后的索引字段仍可能用于索引层过滤。
4. WHERE 在逻辑上过滤 JOIN 结果，但实际执行可以提前过滤。 逻辑执行顺序与物理执行计划必须区分。
5. SQL 优化的核心是减少不必要的数据访问和计算。 不能仅凭索引数量、`EXPLAIN` 的估算行数或单次耗时判断优化效果。

对于校园活动查询案例，最重要的优化思想是：将原本混在 `OR EXISTS` 中的两条业务数据访问路径拆开，分别利用创建者复合索引和报名用户索引，让 MySQL 尽早缩小候选集，而不是扫描整张活动表后再判断用户关系。