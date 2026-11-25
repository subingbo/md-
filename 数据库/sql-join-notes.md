# SQL 多表查询与执行原理

> 核心结论：SQL 描述“想要什么数据”，MySQL 优化器决定“怎样找到这些数据”。  
> `FROM → JOIN → ON → WHERE → SELECT` 是便于理解的**逻辑处理模型**，不等于 MySQL 实际逐步执行、逐步生成完整中间表的物理过程。

## 1. 可直接运行的示例数据

以下 SQL 适用于 MySQL。为方便观察 `LEFT JOIN`，员工“赵六”的 `dept_id` 故意设置为不存在的 `40`。

```sql
DROP TABLE IF EXISTS employee;
DROP TABLE IF EXISTS department;

CREATE TABLE department (
    id        BIGINT PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL
) ENGINE = InnoDB;

CREATE TABLE employee (
    id      BIGINT PRIMARY KEY,
    name    VARCHAR(50) NOT NULL,
    dept_id BIGINT,
    age     INT,
    INDEX idx_employee_dept_id (dept_id)
) ENGINE = InnoDB;

INSERT INTO department (id, dept_name) VALUES
    (10, '技术部'),
    (20, '市场部'),
    (30, '人事部');

INSERT INTO employee (id, name, dept_id, age) VALUES
    (1, '张三', 10, 28),
    (2, '李四', 20, 31),
    (3, '王五', 10, 25),
    (4, '赵六', 40, 28),
    (5, '钱七', NULL, 35);
```

数据如下：

**employee：**

| id | name | dept_id | age |
|---:|---|---:|---:|
| 1 | 张三 | 10 | 28 |
| 2 | 李四 | 20 | 31 |
| 3 | 王五 | 10 | 25 |
| 4 | 赵六 | 40 | 28 |
| 5 | 钱七 | NULL | 35 |

**department：**

| id | dept_name |
|---:|---|
| 10 | 技术部 |
| 20 | 市场部 |
| 30 | 人事部 |

---

## 2. 行与列：先分清两个方向

- **行（row、记录）**：一个具体对象，例如“张三”这一名员工。
- **列（column、字段）**：对象的某个属性，例如 `name`、`age`。

通常可以先这样理解：

- `FROM`、`JOIN`、`ON`：确定数据来自哪里，以及不同表的行如何匹配。
- `WHERE`：决定最终保留哪些**行**。
- `SELECT`：决定最终返回哪些**列**，也可以计算表达式。

```sql
SELECT name, age       -- 只返回这两列
FROM employee
WHERE age = 28;        -- 只保留年龄为 28 的行
```

结果：

| name | age |
|---|---:|
| 张三 | 28 |
| 赵六 | 28 |

因此，“查哪些行”和“返回哪些列”是两个不同问题。

---

## 3. FROM、JOIN、ON、WHERE、SELECT 的关系

### 3.1 JOIN 属于 FROM 子句，ON 属于对应的 JOIN

```sql
SELECT e.name, d.dept_name
FROM employee AS e
INNER JOIN department AS d
    ON e.dept_id = d.id
WHERE e.age >= 28;
```

可以把结构看成：

```text
SELECT 要返回的列
FROM (
    左表
    JOIN 右表
    ON 两表的行匹配条件
)
WHERE 对行进行过滤
```

各关键字的职责：

| 关键字 | 主要职责 |
|---|---|
| `FROM` | 指定数据来源；表连接也写在此子句中 |
| `JOIN` | 把另一数据源加入查询，建立行的组合关系 |
| `ON` | 指定当前这次连接中，左右两边的行怎样匹配 |
| `WHERE` | 排除不满足条件的行 |
| `SELECT` | 决定返回哪些列或表达式 |

连续连接多张表时，每个 `JOIN` 通常都有自己的 `ON`：

```sql
SELECT ...
FROM table_a AS a
JOIN table_b AS b ON a.b_id = b.id
JOIN table_c AS c ON b.c_id = c.id;
```

### 3.2 简化的逻辑处理顺序

对当前讨论范围，可以记为：

```text
FROM / JOIN / ON → WHERE → SELECT
```

例如：

```sql
SELECT e.name, d.dept_name
FROM employee AS e
JOIN department AS d
    ON e.dept_id = d.id
WHERE e.age >= 28;
```

逻辑上可以理解为：

1. 从 `employee` 和 `department` 取得数据来源。
2. 按 `e.dept_id = d.id` 建立匹配的行组合。
3. 用 `e.age >= 28` 过滤行。
4. 最终只输出 `e.name`、`d.dept_name` 两列。

这里的“先、后”用于解释 SQL 的含义，**不表示 MySQL 必须真的先生成一张完整连接结果表，再做下一步**。

---

## 4. INNER JOIN 和 LEFT JOIN 返回什么

### 4.1 INNER JOIN：只保留匹配成功的组合

`JOIN` 不写类型时，通常就是 `INNER JOIN`。

```sql
SELECT e.id, e.name, e.dept_id,
       d.id AS department_id, d.dept_name
FROM employee AS e
INNER JOIN department AS d
    ON e.dept_id = d.id
ORDER BY e.id;
```

结果：

| id | name | dept_id | department_id | dept_name |
|---:|---|---:|---:|---|
| 1 | 张三 | 10 | 10 | 技术部 |
| 2 | 李四 | 20 | 20 | 市场部 |
| 3 | 王五 | 10 | 10 | 技术部 |

- 赵六的 `dept_id = 40`，右表没有对应部门，所以被排除。
- 钱七的 `dept_id` 是 `NULL`，也无法用等号匹配。
- 人事部没有员工；由于左边从员工表开始，也不会凭空产生员工行。

### 4.2 LEFT JOIN：保留左表全部行

```sql
SELECT e.id, e.name, e.dept_id,
       d.id AS department_id, d.dept_name
FROM employee AS e
LEFT JOIN department AS d
    ON e.dept_id = d.id
ORDER BY e.id;
```

结果：

| id | name | dept_id | department_id | dept_name |
|---:|---|---:|---:|---|
| 1 | 张三 | 10 | 10 | 技术部 |
| 2 | 李四 | 20 | 20 | 市场部 |
| 3 | 王五 | 10 | 10 | 技术部 |
| 4 | 赵六 | 40 | NULL | NULL |
| 5 | 钱七 | NULL | NULL | NULL |

`LEFT JOIN` 的规则是：

1. 左表的每一行都保留。
2. 右表匹配成功时，补上右表的列。
3. 匹配失败时，右表的列用 `NULL` 补齐。

记忆：

```text
INNER JOIN = 只要匹配成功的行
LEFT JOIN  = 左表全要，右表能匹配就补上
```

---

## 5. ON 与 WHERE：在 LEFT JOIN 中尤其重要

`ON` 决定右表的哪些行可以参加匹配；`WHERE` 过滤连接完成后的逻辑结果。

### 条件写在 ON 中

```sql
SELECT e.name, d.dept_name
FROM employee AS e
LEFT JOIN department AS d
    ON e.dept_id = d.id
   AND d.id = 10
ORDER BY e.id;
```

含义：保留**全部员工**，但右边只有 10 号部门能够匹配。李四仍会出现，只是部门列为 `NULL`。

### 条件写在 WHERE 中

```sql
SELECT e.name, d.dept_name
FROM employee AS e
LEFT JOIN department AS d
    ON e.dept_id = d.id
WHERE d.id = 10
ORDER BY e.id;
```

含义：先按 `LEFT JOIN` 的语义连接，再只保留 `d.id = 10` 的结果行。右表未匹配的行中 `d.id` 为 `NULL`，不满足条件，因此会被过滤掉。

所以，`LEFT JOIN` 后在 `WHERE` 中要求右表某列等于具体值，常会使结果表现得像 `INNER JOIN`。但不能简单说 `ON` 和 `WHERE` 永远不可互换；是否等价取决于连接类型、条件内容以及 `NULL` 语义。

---

## 6. 逻辑处理不等于物理执行

为了理解 SQL 的含义，可以想象：

```text
两表按 ON 组合 → WHERE 过滤 → SELECT 取列
```

这是**逻辑模型**。它定义查询结果应该是什么。

MySQL 真正执行时关注的是**物理执行计划**，可能会：

- 先利用 `WHERE` 中的高选择性条件找到很少的行；
- 使用索引直接定位记录；
- 调整 `INNER JOIN` 的驱动顺序；
- 把条件尽量下推到更早阶段；
- 只读取查询真正需要的列或索引页；
- 选择嵌套循环等连接方式；
- 在必要时使用临时表、排序或物化结果。

例如：

```sql
SELECT e.name, d.dept_name
FROM employee AS e
JOIN department AS d
    ON e.dept_id = d.id
WHERE e.id = 1;
```

一种可能的物理执行思路是：

1. 通过 `employee` 的主键索引直接找到 `id = 1` 的员工。
2. 取得该员工的 `dept_id = 10`。
3. 通过 `department` 的主键索引直接找到 `id = 10` 的部门。
4. 组装并返回需要的两列。

它不需要先读取两张表的所有行，更不需要先在内存中构造完整笛卡尔积。

> “JOIN 逻辑上可从行组合来理解”不等于“数据库物理上一定先生成所有行组合”。

---

## 7. 为什么 FROM、WHERE、JOIN 不一定读取整张表

### 7.1 有合适索引：可以直接定位

```sql
SELECT *
FROM employee
WHERE id = 1;
```

`id` 是主键。InnoDB 可以通过主键的 B+ 树索引定位记录，通常只需访问少量索引页和数据页，而不是逐行检查全表。

连接也类似：

```sql
SELECT e.name, d.dept_name
FROM employee AS e
JOIN department AS d
    ON e.dept_id = d.id
WHERE e.id = 1;
```

两边用于定位的 `e.id`、`d.id` 都是主键，MySQL 有机会进行高效查找。

### 7.2 没有合适索引：可能全表扫描

```sql
SELECT *
FROM employee
WHERE age = 28;
```

示例表的 `age` 没有索引。为确认哪些员工是 28 岁，MySQL 可能逐行检查 `employee`，这就是**全表扫描**。

全表扫描的意思是数据库服务器需要扫描表中的大量甚至全部记录，不是把整张表先传给客户端。过滤仍然发生在 MySQL 服务器内部，客户端只收到查询结果。

### 7.3 有索引也不保证一定使用

可以添加索引再观察：

```sql
CREATE INDEX idx_employee_age ON employee(age);

EXPLAIN
SELECT *
FROM employee
WHERE age = 28;
```

优化器会根据成本估算决定是否使用索引。如果表很小、条件会命中大部分行，或者读取索引后还要进行大量回表，直接扫描可能反而更便宜。因此：

```text
存在索引 ≠ 一定使用索引
没有合适索引 + 需要检查大量记录 ≈ 容易发生全表扫描
```

---

## 8. EXPLAIN 入门

在查询前加 `EXPLAIN`，可以查看 MySQL 计划怎样访问表：

```sql
EXPLAIN
SELECT e.name, d.dept_name
FROM employee AS e
JOIN department AS d
    ON e.dept_id = d.id
WHERE e.id = 1;
```

常用字段：

| 字段 | 关注点 |
|---|---|
| `table` | 当前这一行计划访问哪张表 |
| `type` | 访问方式，如全表扫描、索引范围、等值查找 |
| `possible_keys` | 优化器认为可能使用的索引 |
| `key` | 实际计划使用的索引；`NULL` 表示未选择索引 |
| `key_len` | 计划使用的索引长度，可辅助判断复合索引用了多少部分 |
| `ref` | 用什么值或列与索引比较 |
| `rows` | 预计需要检查的行数，是估算值，不是必然的实际值 |
| `filtered` | 预计经过条件过滤后保留的百分比 |
| `Extra` | 补充信息，如 `Using index`、`Using where`、`Using temporary`、`Using filesort` |

常见 `type` 可先按下面理解。通常从上到下需要检查的范围逐渐缩小，但不能只凭这一列判断查询好坏。

| `type` | 简化含义 |
|---|---|
| `ALL` | 全表扫描 |
| `index` | 扫描整个索引 |
| `range` | 扫描索引的一段范围 |
| `ref` | 使用非唯一索引等值查找，可能返回多行 |
| `eq_ref` | 连接时通过主键/唯一索引等值匹配，前一行对应至多一行 |
| `const` | 主键/唯一索引与常量等值比较，结果至多一行 |

对比实验：

```sql
-- 主键等值查询，通常能看到 const
EXPLAIN SELECT * FROM employee WHERE id = 1;

-- age 尚未建立索引时，通常容易看到 ALL
-- 若已执行上一节的 CREATE INDEX，可先删除它再比较
DROP INDEX idx_employee_age ON employee;
EXPLAIN SELECT * FROM employee WHERE age = 28;

-- 建立索引后再次观察
CREATE INDEX idx_employee_age ON employee(age);
EXPLAIN SELECT * FROM employee WHERE age = 28;
```

示例数据只有 5 行，优化器可能认为全表扫描成本更低，因此实际计划不一定与大型表相同。这不是索引失效，而是成本选择。

如果 MySQL 版本支持，还可以用：

```sql
EXPLAIN ANALYZE
SELECT *
FROM employee
WHERE id = 1;
```

`EXPLAIN ANALYZE` 会真正执行查询，并给出实际行数和耗时等信息；普通 `EXPLAIN` 主要展示计划和估算。不要对会修改大量数据的语句随意使用会实际执行的分析方式。

---

## 9. 客户端—服务器执行流程

以 Java、Navicat 或命令行客户端为例：

```text
客户端发送 SQL
       ↓
MySQL 解析并检查 SQL
       ↓
优化器选择执行计划
       ↓
执行器按计划调用存储引擎读取需要的页和记录
       ↓
在服务器端完成连接、过滤、表达式计算等工作
       ↓
把结果集发送给客户端
       ↓
客户端展示结果，或由 Java 通过 ResultSet 读取
```

因此下面的查询：

```sql
SELECT e.name, d.dept_name
FROM employee AS e
JOIN department AS d
    ON e.dept_id = d.id
WHERE e.id = 1;
```

并不是：

```text
MySQL 把 employee 和 department 整表传给 Java
→ Java 自己 JOIN
→ Java 自己 WHERE
```

而是：

```text
Java 发送 SQL
→ MySQL 在服务器端访问、连接并过滤数据
→ MySQL 只返回 name 和 dept_name 组成的结果集
```

### MySQL 可能流式返回结果行

“数据库执行完后返回结果”是正确的整体理解，但不必理解成“必须先把整个结果集全部算完并存好，再一次性发送”。

在执行计划允许时，MySQL 可以一边执行，一边把已经产生的结果行发送给客户端；客户端驱动也可能逐行读取。另一方面，排序、分组、去重或某些连接计划可能需要先缓存、排序或物化一部分甚至全部数据。客户端驱动还可能默认缓冲结果集。

所以更准确的表述是：

> 查询计算由数据库服务器执行，客户端接收结果；结果行可能边产生边传输，也可能因具体算子和客户端配置而需要缓冲。

---

## 10. 最终记忆框架

### 语义层面

```text
FROM：数据从哪里来
JOIN：再关联哪个数据源
ON：两边的行怎样匹配
WHERE：保留哪些行
SELECT：返回哪些列或表达式
```

### 连接结果

```text
INNER JOIN：只保留左右匹配成功的行组合
LEFT JOIN ：保留左表全部行；右表匹配失败时补 NULL
```

### 执行层面

```text
逻辑处理顺序 ≠ 物理执行步骤
使用 FROM/JOIN/WHERE ≠ 必然读取整张表
是否全表扫描取决于索引、条件、数据分布和优化器的成本判断
EXPLAIN 用于观察执行计划，而不是背诵 SQL 的逻辑顺序
```

### 客户端与数据库

```text
客户端负责发送 SQL、读取结果
MySQL 服务器负责解析、优化、访问数据、连接和过滤
服务器可能流式发送已经产生的结果行
```

一句话总结：

> **SQL 的逻辑模型帮助我们判断结果应该是什么；MySQL 的物理执行计划决定实际读取什么、使用什么索引以及以什么顺序完成查询。**


