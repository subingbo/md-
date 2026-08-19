# MySQL EXPLAIN：const、eq_ref、ref 与联表索引

## 1. 联表查询的 EXPLAIN

对于普通多表 JOIN：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id;
```

`EXPLAIN` 通常会为**每张参与查询的表显示一行执行计划**。

例如两张表：

```text
id | table | type
---+-------+--------
1  | u     | const
1  | o     | eq_ref
```

可以理解为：

```text
user
 ↓
orders
```

第一行描述 `user` 怎么查，第二行描述 `orders` 怎么查。

> 注意：复杂 SQL（子查询、UNION、派生表等）不一定严格“一张表一行”。

------

# 2. 联表查询是怎么使用索引的？

最典型的 SQL：

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

执行过程：

```text
① 查询 user

u.id = 100
    ↓
PRIMARY KEY
    ↓
找到 user.id = 100
    ↓
得到一行
```

然后把这一行中的 `u.id` 拿出来：

```text
② 查询 orders

u.id = 100
    ↓
o.user_id = 100
    ↓
UNIQUE INDEX
    ↓
最多找到一条
```

整体可以理解成：

```text
                 user
                  │
            PRIMARY KEY
                  │
              id = 100
                  │
                  ↓
              找到一行
                  │
                  │ 把 u.id = 100
                  ↓
                orders
                  │
           UNIQUE(user_id)
                  │
          user_id = 100
                  ↓
              最多一行
```

这就是联表查询中非常重要的思想：

> **前表查询出来的数据，可以作为后表索引查询的条件。**

------

# 3. Nested-Loop Join

MySQL 常见的联表执行方式可以理解成嵌套循环：

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

如果 `user` 查出：

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
使用 orders.user_id 索引

u.id = 5
    ↓
orders.user_id = 5
    ↓
使用 orders.user_id 索引

u.id = 9
    ↓
orders.user_id = 9
    ↓
使用 orders.user_id 索引
```

因此：

> **JOIN 条件中的被驱动表字段，建立合适的索引非常重要。**

------

# 4. const

## 4.1 定义

`const`：

> **使用主键或唯一索引与常量进行等值比较，并且最多只能找到一行。**

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
type = const
```

核心：

```text
常量
 ↓
主键/唯一索引
 ↓
一行
```

------

# 5. eq_ref

## 5.1 定义

`eq_ref`：

> **联表时，使用前面表的一行数据，去当前表的主键或唯一索引进行等值查找，并且最多匹配一行。**

例如：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id;
```

假设：

```text
user.id
    → PRIMARY KEY

orders.user_id
    → UNIQUE INDEX
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
```

此时 `orders` 这一行的：

```text
type = eq_ref
```

------

# 6. const 和 eq_ref 的核心区别

这是最重要的区别：

```text
const：

常量
 ↓
主键/唯一索引
 ↓
最多一行
```

而：

```text
eq_ref：

前表字段
 ↓
主键/唯一索引
 ↓
最多一行
```

也就是说：

| 类型     | 查询值来自哪里 | 索引          | 最大匹配行数 |
| -------- | -------------- | ------------- | ------------ |
| `const`  | 常量           | 主键/唯一索引 | 1            |
| `eq_ref` | 前表的一行     | 主键/唯一索引 | 1            |
| `ref`    | 前表的一行     | 普通索引      | 多行         |

------

# 7. `eq_ref` 不要求“主键对主键”

这是容易产生误解的地方。

例如：

```sql
JOIN orders o ON o.user_id = u.id
```

完全可以：

```text
user.id
    ↓
PRIMARY KEY

orders.user_id
    ↓
UNIQUE INDEX
```

仍然是：

```text
eq_ref
```

并不要求：

```text
user.id     → PRIMARY KEY
orders.id   → PRIMARY KEY
```

真正的要求是：

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

> **决定 `eq_ref` 的关键，是“后表用于匹配的字段是不是主键/唯一索引”，而不是两边是不是主键。**

------

# 8. ref

如果后表使用的是**普通非唯一索引**：

```sql
SELECT *
FROM user u
JOIN orders o ON o.user_id = u.id;
```

假设：

```text
orders.user_id
    ↓
普通 INDEX
```

那么：

```text
u.id = 100
    ↓
o.user_id = 100
    ↓
普通索引
    ↓
可能找到很多订单
```

例如：

```text
order 1
order 2
order 3
order 4
...
```

此时：

```text
type = ref
```

所以：

```text
eq_ref：

前表字段
 ↓
唯一索引
 ↓
最多一行
ref：

前表字段
 ↓
普通索引
 ↓
可能多行
```

------

# 9. 一个完整例子

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
user.PRIMARY KEY
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
orders.UNIQUE INDEX
        ↓
最多一行
        ↓
eq_ref
```

所以 EXPLAIN 可能是：

```text
id | table | type    | key
---+-------+---------+--------------
1  | u     | const   | PRIMARY
1  | o     | eq_ref  | user_id
```

整个执行过程：

```text
              user
                │
          WHERE id = 100
                │
          PRIMARY KEY
                │
                ↓
             const
                │
                │ u.id = 100
                ↓
              orders
                │
       user_id = u.id
                │
          UNIQUE INDEX
                │
                ↓
             eq_ref
```

------

# 10. 如果 `orders.user_id` 不是唯一索引

如果：

```text
orders.user_id
    ↓
普通 INDEX
```

执行过程：

```text
u.id = 100
    ↓
orders.user_id = 100
    ↓
普通 INDEX
    ↓
可能找到多行
```

那么：

```text
EXPLAIN

id | table | type
---+-------+------
1  | u     | const
1  | o     | ref
```

因此：

```text
user       → const
orders     → ref
```

------

# 11. 三种类型直接对比

```text
                    查询条件的值
                         │
           ┌─────────────┴─────────────┐
           ↓                           ↓
         常量                       前表字段
           │                           │
           ↓                           ↓
      主键/唯一索引              主键/唯一索引
           │                           │
           ↓                           ↓
         const                     eq_ref
           │                           │
        最多1行                    最多1行
```

如果前表字段去匹配的是普通索引：

```text
前表字段
   ↓
普通索引
   ↓
ref
   ↓
可能多行
```

------

# 12. 面试记忆版

直接记下面这三个：

```text
const
= 常量 + 主键/唯一索引
= 最多一行
eq_ref
= 前表字段 + 主键/唯一索引
= 最多一行
= 常见于 JOIN
ref
= 前表字段 + 普通索引
= 可能多行
= 常见于 JOIN
```

最核心的一句话：

> **`const` 是“拿固定值查唯一记录”，`eq_ref` 是“拿前表的一行数据查后表的唯一记录”，`ref` 是“拿前表的一行数据查后表的多条可能记录”。**