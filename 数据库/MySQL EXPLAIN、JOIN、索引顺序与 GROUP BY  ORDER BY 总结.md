# MySQL EXPLAIN、JOIN、索引顺序与 GROUP BY / ORDER BY 总结

## 1. EXPLAIN 的基本理解

`EXPLAIN` 用来查看 MySQL 的执行计划。

对于普通多表 JOIN：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id;
```

通常会看到：

```text
id | table | type
---+-------+--------
1  | u     | ...
1  | o     | ...
```

可以先简单理解为：

> **一张参与查询的表，通常对应执行计划中的一行。**

例如三表 JOIN：

```text
user
  ↓
orders
  ↓
product
```

通常会看到三张表对应的访问信息。

但复杂 SQL 中存在子查询、派生表、UNION 等情况，因此不能绝对认为“一张表一定只有一行”。

------

# 2. JOIN 中索引是怎么使用的？

考虑：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id
WHERE u.id = 100;
```

假设：

```text
user.id
    ↓
PRIMARY KEY

orders.user_id
    ↓
UNIQUE INDEX
```

可以把执行过程理解成：

```text
① 查询 user

u.id = 100
    ↓
PRIMARY KEY
    ↓
找到 user.id = 100
    ↓
得到一行


② 查询 orders

拿到前面的 u.id = 100
    ↓
o.user_id = 100
    ↓
UNIQUE INDEX
    ↓
最多找到一行
```

核心思想：

> **JOIN 时，前面表查询出来的一行数据，可以作为后面表索引查询的条件。**

------

# 3. Nested-Loop Join

MySQL 常见的 JOIN 执行方式可以抽象成：

```text
驱动表 A
   ↓
查出一行 A
   ↓
拿 A 中的字段
   ↓
去 B 表索引查
   ↓
得到匹配的 B
   ↓
再处理 A 的下一行
```

例如：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id;
```

如果 `user` 查出来：

```text
u.id
----
1
5
9
```

那么可以理解成：

```text
u.id = 1
   ↓
orders.user_id = 1
   ↓
使用索引

u.id = 5
   ↓
orders.user_id = 5
   ↓
使用索引

u.id = 9
   ↓
orders.user_id = 9
   ↓
使用索引
```

所以 JOIN 中非常重要的一点：

> **被驱动表的 JOIN 字段通常应该建立合适的索引。**

------

# 4. const

## 4.1 定义

`const` 表示：

> **使用主键或唯一索引与常量进行等值匹配，并且最多只能找到一行。**

例如：

```sql
SELECT *
FROM user
WHERE id = 100;
```

如果：

```text
user.id → PRIMARY KEY
```

那么：

```text
id = 100
   ↓
PRIMARY KEY
   ↓
最多一行
   ↓
const
```

记忆：

```text
const
= 常量
+ 主键/唯一索引
+ 最多一行
```

------

# 5. eq_ref

## 5.1 定义

`eq_ref` 表示：

> **JOIN 时，拿前面表的一行数据，去当前表的主键或唯一索引进行等值查询，并且最多匹配一行。**

例如：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id;
```

假设：

```text
user.id
    ↓
PRIMARY KEY

orders.user_id
    ↓
UNIQUE INDEX
```

执行：

```text
u.id = 100
    ↓
o.user_id = 100
    ↓
UNIQUE INDEX
    ↓
最多一行
    ↓
eq_ref
```

记忆：

```text
eq_ref
= 前表字段
+ 主键/唯一索引
+ 最多一行
+ 常见于 JOIN
```

------

# 6. eq_ref 不要求“主键对主键”

这是非常容易搞错的一点。

例如：

```sql
JOIN orders o ON o.user_id = u.id
```

完全可以是：

```text
user.id
    → PRIMARY KEY

orders.user_id
    → UNIQUE INDEX
```

仍然可以得到：

```text
eq_ref
```

并不要求：

```text
user.id    → PRIMARY KEY
orders.id  → PRIMARY KEY
```

真正重要的是：

```text
前表字段
   ↓
后表的主键/唯一索引
   ↓
等值匹配
   ↓
最多一行
```

所以：

> **决定 `eq_ref` 的关键是后表用于匹配的字段是否为主键/唯一索引，而不是两边是否都是主键。**

------

# 7. ref

如果后表使用的是普通非唯一索引：

```text
orders.user_id
    ↓
普通 INDEX
```

那么：

```text
u.id = 100
    ↓
orders.user_id = 100
    ↓
普通索引
    ↓
可能找到多行
```

此时通常：

```text
type = ref
```

记忆：

```text
ref
= 前表字段
+ 普通索引
+ 可能多行
```

------

# 8. const、eq_ref、ref 对比

| type     | 查询值来源 | 后表索引      | 匹配结果  |
| -------- | ---------- | ------------- | --------- |
| `const`  | 常量       | 主键/唯一索引 | 最多 1 行 |
| `eq_ref` | 前表字段   | 主键/唯一索引 | 最多 1 行 |
| `ref`    | 前表字段   | 普通索引      | 可能多行  |

可以直接记：

```text
const：

常量
 ↓
唯一索引
 ↓
一行
eq_ref：

前表字段
 ↓
唯一索引
 ↓
一行
ref：

前表字段
 ↓
普通索引
 ↓
多行
```

------

# 9. 一个完整 JOIN 例子

SQL：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id
WHERE u.id = 100;
```

索引：

```text
user.id
    → PRIMARY KEY

orders.user_id
    → UNIQUE INDEX
```

执行过程：

```text
第一步：

WHERE u.id = 100
        ↓
PRIMARY KEY
        ↓
最多一行
        ↓
const
```

第二步：

```text
拿到 u.id = 100
        ↓
orders.user_id = 100
        ↓
UNIQUE INDEX
        ↓
最多一行
        ↓
eq_ref
```

所以 EXPLAIN 可能是：

```text
id | table | type
---+-------+--------
1  | u     | const
1  | o     | eq_ref
```

如果 `orders.user_id` 改成普通索引：

```text
id | table | type
---+-------+--------
1  | u     | const
1  | o     | ref
```

------

# 10. GROUP BY 和 type 是不同维度

例如：

```sql
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id;
```

`type` 描述：

> **数据是怎么被访问的。**

例如：

```text
const
eq_ref
ref
range
index
ALL
```

而 `GROUP BY` 解决的是：

> **拿到数据以后，如何进行分组。**

所以不能认为：

```text
GROUP BY → 某一种固定 type
```

两者是不同维度：

```text
type
 ↓
怎么找到数据

GROUP BY
 ↓
怎么进行分组
```

------

# 11. GROUP BY 可以利用索引顺序

假设：

```sql
INDEX idx_user_id(user_id)
```

索引本身按照 `user_id` 有序：

```text
1
1
1
2
2
3
3
3
4
...
```

执行：

```sql
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id;
```

MySQL 顺着索引读取：

```text
1 1 1
 ↓
一组

2 2
 ↓
一组

3 3 3
 ↓
一组
```

因此：

> **索引的有序性不仅可以用于查找，也可以帮助 GROUP BY。**

------

# 12. ORDER BY 可以利用索引顺序

例如：

```sql
INDEX idx_age(age)
```

索引逻辑顺序：

```text
18
19
20
21
22
...
```

执行：

```sql
SELECT *
FROM user
ORDER BY age;
```

MySQL 可以直接按照索引顺序读取：

```text
18 → 19 → 20 → 21 → 22
```

因此不一定需要额外排序。

核心理解：

```text
索引扫描
   ↓
天然得到有序数据
   ↓
这个顺序刚好满足 ORDER BY
   ↓
直接使用
   ↓
避免额外排序
```

------

# 13. 不是“只要查过索引一次，ORDER BY 就能使用”

这个说法需要修正。

不是：

```text
先查索引
 ↓
得到数据
 ↓
以后 ORDER BY 都可以使用这个顺序
```

而是：

> **索引本身具有固定的有序性，只有当这个有序性与 ORDER BY 要求的顺序匹配时，才能利用。**

例如：

```text
INDEX(age)
```

提供：

```text
age:
18
19
20
21
```

所以：

```sql
ORDER BY age
```

可以利用。

但是：

```sql
ORDER BY name
```

不能利用 `INDEX(age)` 的顺序。

因为：

```text
索引顺序：age
ORDER BY：name
```

两者不一致。

------

# 14. 联合索引的顺序

假设：

```sql
INDEX idx_a_b_c(a, b, c)
```

联合索引的排序规则：

```text
先按照 a
 ↓
a 相同时按照 b
 ↓
a、b 都相同时按照 c
```

例如：

```text
a  b  c
---------
1  1  3
1  1  8
1  2  1
1  3  5
2  1  2
2  2  4
3  1  7
```

可以理解为：

```text
a → b → c
```

------

# 15. 联合索引对 ORDER BY 的支持

对于：

```sql
INDEX(a, b, c)
```

以下情况可以利用其顺序：

```sql
ORDER BY a;
ORDER BY a, b;
ORDER BY a, b, c;
```

因为它们与索引顺序一致：

```text
a → b → c
```

------

# 16. 联合索引不能直接支持任意字段排序

例如：

```sql
INDEX(a, b)
```

不能简单地认为：

```sql
ORDER BY b
```

也能利用索引。

因为索引实际上是：

```text
先按 a
再按 b
```

例如：

```text
a  b
-----
1  1
1  2
2  1
2  3
```

如果只看 `b`：

```text
1
2
1
3
```

并不是全局有序。

所以：

```text
INDEX(a,b)

ORDER BY a       √
ORDER BY a,b     √
ORDER BY b       ×
```

------

# 17. 联合索引前面的列被固定后，后面的列也可能用于排序

这是联合索引非常重要的一个规则。

假设：

```sql
INDEX(a, b)
```

查询：

```sql
SELECT *
FROM table
WHERE a = 10
ORDER BY b;
```

虽然整个索引是：

```text
a → b
```

但是因为：

```text
a = 10
```

已经把范围限制到了：

```text
a = 10
```

这个范围内部：

```text
b：
1
2
3
4
5
...
```

本身就是有序的。

因此：

```text
WHERE a = 10
ORDER BY b
```

可以利用：

```text
INDEX(a,b)
```

的有序性。

------

# 18. 联合索引排序规律总结

对于：

```text
INDEX(a,b,c)
```

可以记：

```text
ORDER BY a
        √

ORDER BY a,b
        √

ORDER BY a,b,c
        √
```

而：

```text
ORDER BY b
        ×

ORDER BY c
        ×
```

但如果：

```text
WHERE a = 常量
```

那么：

```text
ORDER BY b
        √
```

如果：

```text
WHERE a = 常量
```

那么：

```text
ORDER BY b,c
        √
```

核心：

> **联合索引的有序性是从左到右逐级建立的；前面的索引列被固定后，后面的索引列可以在这个固定范围内继续提供有序性。**

------

# 19. GROUP BY + ORDER BY

例如：

```sql
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id
ORDER BY user_id;
```

如果：

```text
INDEX(user_id)
```

索引已经按照：

```text
user_id
```

有序。

那么：

```text
GROUP BY user_id
```

需要的顺序：

```text
user_id
```

以及：

```text
ORDER BY user_id
```

需要的顺序：

```text
user_id
```

刚好一致。

因此索引的有序性可以同时帮助：

```text
GROUP BY
+
ORDER BY
```

------

# 20. EXPLAIN 中 GROUP BY / ORDER BY 重点看什么？

当 SQL 中出现：

```text
GROUP BY
ORDER BY
```

重点关注 `Extra`。

### `Using temporary`

表示使用了临时表。

```text
扫描数据
 ↓
临时表
 ↓
分组/处理
```

### `Using filesort`

表示 MySQL 需要执行额外的排序算法。

注意：

> `Using filesort` 不代表一定使用磁盘文件。

它是 MySQL 对额外排序过程的名称，可能在内存中完成。

### `Using index`

通常说明查询可以直接利用索引完成数据访问，例如覆盖索引场景。

------

# 21. JOIN、GROUP BY、ORDER BY 应该分别看

可以建立下面这个整体思维：

```text
SQL 执行计划
│
├── ① 怎么找到数据？
│      ↓
│    type / key / rows
│
├── ② JOIN 怎么关联？
│      ↓
│    前表字段
│       ↓
│    后表索引
│       ↓
│    const / eq_ref / ref ...
│
├── ③ GROUP BY 怎么分组？
│      ↓
│    能否利用索引顺序
│    是否 Using temporary
│
└── ④ ORDER BY 怎么排序？
       ↓
     能否利用索引顺序
     是否 Using filesort
```

------

# 22. 最核心的统一理解

索引有两个非常重要的能力：

```text
索引
├── ① 快速定位数据
│      ↓
│    WHERE / JOIN
│
└── ② 提供有序数据
       ↓
     ORDER BY
     GROUP BY
```

所以索引不仅仅是：

> “让我更快找到某条数据。”

它还有：

> **“我按照某种顺序保存/遍历数据，所以可以直接提供这种顺序。”**

------

# 23. 最终面试记忆版

## JOIN

```text
const
= 常量
+ 主键/唯一索引
+ 最多一行
eq_ref
= 前表字段
+ 主键/唯一索引
+ 最多一行
ref
= 前表字段
+ 普通索引
+ 可能多行
```

------

## 索引顺序

```text
INDEX(a,b,c)
```

本质：

```text
先 a
↓
再 b
↓
最后 c
```

因此：

```text
ORDER BY a       √
ORDER BY a,b     √
ORDER BY a,b,c   √
```

而：

```text
ORDER BY b       ×
ORDER BY c       ×
```

但是：

```text
WHERE a = 10
ORDER BY b
```

可以利用 `(a,b)` 的索引顺序。

------

## GROUP BY / ORDER BY

```text
type
↓
怎么找到数据

GROUP BY
↓
怎么分组

ORDER BY
↓
怎么排序
```

索引：

```text
既能帮助快速查找
又能提供有序数据
```

最终记住一句：

> **索引扫描得到的顺序，可以被 `ORDER BY`、`GROUP BY` 利用，但前提是索引提供的顺序与它们需要的顺序匹配；联合索引则遵循最左侧优先、逐级有序的规则。**