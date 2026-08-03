# 📚 SQL 子查询 & LATERAL 完整笔记

------

# 🧠 一、子查询分类

## 1️⃣ 非相关子查询（Non-correlated Subquery）

```
SELECT *
FROM orders
WHERE amount > (
    SELECT AVG(amount) FROM orders
);
```

### 特点

- ❌ 不依赖外层查询
- ✅ 只执行一次
- ✅ 可以独立运行

------

## 2️⃣ 相关子查询（Correlated Subquery）

```
SELECT *
FROM orders o
WHERE o.amount > (
    SELECT AVG(amount)
    FROM orders
    WHERE user_id = o.user_id
);
```

### 特点

- ✅ 使用外层字段（`o.user_id`）
- ✅ 每一行都会执行子查询
- ❌ 不能独立运行

------

## 🧩 判断口诀

> **子查询能不能单独运行？**

- 能 → 非相关子查询
- 不能 → 相关子查询

------

# 🧠 二、子查询字段来源规则（作用域）

## ✅ 子查询中字段来源只能是：

1. 自己的表
2. 外层查询的表（相关子查询）
3. 常量

------

## ❌ 非法示例

```
SELECT *
FROM orders o
WHERE EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.user_id = users.id  -- ❌ users 未定义
);
```

👉 错误：字段不属于任何作用域

------

## 🔥 作用域规则

```
外层查询
   ↓
子查询
```

- 子查询 ✅ 可以访问外层
- 外层 ❌ 不能访问子查询

------

# 🧠 三、FROM / WHERE 子查询的本质区别

------

## ✅ WHERE 子查询（可以用外层字段）

```
SELECT *
FROM department d
WHERE EXISTS (
    SELECT 1
    FROM employee e
    WHERE e.dept_id = d.dept_id
);
```

### 执行逻辑

```
for 每一行 d:
    执行子查询
```

------

## ❌ FROM / JOIN 子查询（默认不能用外层字段）

```
SELECT *
FROM department d
JOIN (
    SELECT *
    FROM employee e
    WHERE e.dept_id = d.dept_id  -- ❌ 报错
) t;
```

### 原因

```
FROM 阶段：
d 和 子查询 t 是“同级”
```

👉 同级不能互相引用

------

# 🧠 四、JOIN 子查询 ≠ 外层子查询

## 关键点

```
FROM department d
JOIN (...) t
```

等价于：

```
FROM
 ├── d
 └── t
```

👉 两者是**并列关系（siblings）**

------

# 🚀 五、LATERAL（横向子查询）

------

## ✅ 作用

> **让 FROM 子查询可以引用外层字段**

------

## 示例

```
SELECT d.dept_name, t.max_salary
FROM department d
JOIN LATERAL (
    SELECT MAX(e.salary) AS max_salary
    FROM employee e
    WHERE e.dept_id = d.dept_id
) t ON TRUE;
```

------

## 🧠 执行模型

```
for 每一行 d:
    执行子查询（可以用 d）
    JOIN 结果
```

------

## 🔥 本质

> LATERAL = “按行执行的子查询”

------

# 🧠 六、LATERAL vs 普通 JOIN

| 对比项       | 普通 JOIN | LATERAL |
| ------------ | --------- | ------- |
| 是否依赖外层 | ❌         | ✅       |
| 执行方式     | 一次性    | 按行    |
| 能否 LIMIT   | ❌         | ✅       |
| 能否 Top N   | ❌         | ✅       |

------

# 🚀 七、典型应用场景

## 🎯 每组 Top 1

```
SELECT d.*, t.*
FROM department d
JOIN LATERAL (
    SELECT *
    FROM employee e
    WHERE e.dept_id = d.dept_id
    ORDER BY salary DESC
    LIMIT 1
) t ON TRUE;
```

------

# ⚠️ 八、性能分析

------

## ❌ 慢的情况

```
外层 N 行 × 内层全表扫描
```

👉 O(N × M)

------

## ✅ 快的情况（关键）

```
有索引 + LIMIT
```

👉 O(N × log M)

------

## 🔥 关键结论

> **不是执行次数决定性能，而是每次执行成本**

------

# 🧠 九、SELECT vs FROM（你刚刚的核心问题）

------

## ❗ 误区

```
SELECT 了什么 → 就只能用什么 ❌
```

------

## ✅ 正确

```
FROM 决定“能用什么”
SELECT 决定“输出什么”
```

------

## 示例

```
SELECT d.dept_name
FROM department d
WHERE d.dept_id = 10;
```

👉 虽然没 SELECT `dept_id`，但仍然可以用

------

# 🧠 十、SQL 执行顺序（必须记）

```
1. FROM
2. JOIN
3. WHERE
4. GROUP BY
5. HAVING
6. SELECT
7. ORDER BY
```

------

# 🧩 十一、终极总结

------

## 📌 子查询

- 是否相关 → 看是否依赖外层字段
- 是否能用外层字段 → 看作用域

------

## 📌 FROM 子查询

- 默认 ❌ 不能用外层字段
- 因为是同级关系

------

## 📌 LATERAL

- ✅ 让子查询“按行依赖外层”
- 本质 = 表函数调用

------

## 📌 一句话总结

> **WHERE 子查询是“算值”，LATERAL 是“造表”**