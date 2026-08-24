# MySQL InnoDB 当前读与快照读

## 1. 两种读取方式

InnoDB 中读取数据主要分为：

- 快照读（Snapshot Read）
- 当前读（Current Read）

---

# 一、快照读（Snapshot Read）

## 1. 定义

普通的 `SELECT` 就是快照读：

```sql
SELECT * FROM user WHERE id = 1;
```

特点：

- 不加锁
- 读取历史版本
- 通过 MVCC 判断可见性
- 使用 Read View

------

## 2. Read View

Read View 是 InnoDB 用来判断：

> 当前事务能够看到哪些事务提交的数据版本。

版本链：

```
最新版本
   |
   v
trx_id = 200
name = B

   |
   v

trx_id = 100
name = A
```

普通 SELECT 会根据 Read View 判断：

- 当前版本是否可见
- 不可见则沿 undo log 找历史版本

------

## 3. Read View 创建时机

RR（Repeatable Read）隔离级别下：

> 第一次执行快照读时创建 Read View。

注意：

不是：

- 事务 BEGIN 时创建
- 第一个 SQL 创建

而是：

第一次需要快照读时创建。

例如：

```
BEGIN;

SELECT * FROM user;
```

这里创建 Read View。

------

但是：

```
BEGIN;

SELECT * FROM user FOR UPDATE;
```

不会创建 Read View。

因为：

`FOR UPDATE` 是当前读。

------

# 二、当前读（Current Read）

## 1. 定义

当前读读取的是：

> 数据的最新提交版本。

常见当前读：

```
SELECT ... FOR UPDATE;

SELECT ... LOCK IN SHARE MODE;

UPDATE;

DELETE;

INSERT;
```

------

## 2. SELECT FOR UPDATE

例如：

```
BEGIN;

SELECT *
FROM user
WHERE id = 1
FOR UPDATE;
```

执行：

1. 读取最新数据版本
2. 给记录加排他锁（X Lock）

效果：

```
user:

id=1

       X锁
        |
        ↓
     当前事务
```

其他事务：

```
UPDATE user
SET name='B'
WHERE id=1;
```

会阻塞。

------

# 三、当前读和快照读的区别

|                   | 当前读            | 快照读            |
| ----------------- | ----------------- | ----------------- |
| 代表SQL           | SELECT FOR UPDATE | 普通SELECT        |
| 读取版本          | 最新版本          | Read View可见版本 |
| 是否创建Read View | 否                | 是                |
| 是否加锁          | 是                | 否                |
| 是否读取undo版本  | 否                | 可能              |
| 是否阻塞写操作    | 可能              | 不会              |

------

# 四、FOR UPDATE不会影响快照读

重点：

```
BEGIN;

SELECT *
FROM user
WHERE id=1
FOR UPDATE;

SELECT *
FROM user
WHERE id=1;
```

第一个：

```
当前读
↓
读取最新版本
↓
加锁
```

第二个：

```
快照读
↓
根据Read View判断版本
```

两者读取机制独立。

------

# 五、为什么FOR UPDATE读到的数据，普通SELECT不一定读到？

因为：

```
FOR UPDATE
    |
    ↓
当前读
    |
    ↓
最新版本


普通SELECT
    |
    ↓
快照读
    |
    ↓
Read View
    |
    ↓
选择可见版本
```

当前读不会：

- 修改 Read View
- 更新 Read View
- 把读取结果保存给后续普通SELECT

------

# 六、Read View创建时机导致的差异

## 情况1：先普通读

```
BEGIN;

SELECT * FROM user;
```

创建：

```
Read View
```

之后：

```
SELECT * FROM user FOR UPDATE;
```

当前读：

```
读取最新版本
```

再：

```
SELECT * FROM user;
```

仍使用之前的 Read View。

可能：

```
普通读:
A

FOR UPDATE:
B

普通读:
A
```

------

## 情况2：先FOR UPDATE

```
BEGIN;

SELECT *
FROM user
FOR UPDATE;
```

不会创建 Read View。

之后：

```
SELECT *
FROM user;
```

第一次快照读：

此时创建 Read View。

因此可能看到当前最新状态。

------

# 七、写操作和Read View

UPDATE / DELETE / INSERT 属于当前读。

但是：

它们不会让已有 Read View 更新。

例如：

```
BEGIN;

SELECT * FROM user;
-- 创建Read View

UPDATE user
SET name='B'
WHERE id=1;

SELECT * FROM user;
```

第二次 SELECT：

虽然能看到自己的修改：

```
name=B
```

但原因不是 Read View 更新。

原因：

> MVCC规定：事务自己的修改自己可见。

------

# 八、锁和MVCC的关系

## 快照读

```
SELECT

      ↓

Read View

      ↓

历史版本

      ↓

无需加锁
```

------

## 当前读

```
SELECT FOR UPDATE

      ↓

最新版本

      ↓

加锁

      ↓

阻止其他事务修改
```

------

# 九、总结

1. 普通 SELECT 是快照读，依赖 Read View。
2. SELECT FOR UPDATE 是当前读：
   - 读取最新版本
   - 给记录加锁
   - 不创建Read View
3. 当前读不会改变快照读的视图。
4. 当前读读到的数据，不会自动成为后续普通SELECT看到的数据。
5. Read View是在第一次快照读时创建，而不是事务开始时。
6. UPDATE/DELETE/INSERT不会刷新Read View，但事务自己的修改对自己可见。

核心一句话：

> 当前读解决并发修改问题，快照读解决一致性读取问题；二者通过MVCC和锁机制配合，而不是共享读取结果。