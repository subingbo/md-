# MySQL MVCC、快照读、当前读与事务可见性

## 1. MVCC解决的问题

MVCC（Multi-Version Concurrency Control，多版本并发控制）的核心目标：

- 提高读并发
- 避免读操作阻塞写操作
- 保证事务读取的一致性

它主要解决：

- 脏读
- 不可重复读（RR隔离级别下）
- 部分幻读问题

但是：

> MVCC保证的是数据版本可见性，不保证事务内所有操作看到完全相同的数据。

------

# 2. InnoDB行版本

InnoDB不会直接覆盖修改数据，而是通过版本链保存历史记录。

例如：

初始数据：

```
id | name | age
1  | Tom  | 18
```

B事务修改：

```
UPDATE user
SET name='Bob'
WHERE id=1;
```

内部类似：

```
最新版本:
id=1
name=Bob
age=18
trx=B

        ↓

undo log:
id=1
name=Tom
age=18
```

每次修改都会产生新的版本。

------

# 3. Read View（快照）

事务第一次执行普通SELECT时，会生成一个Read View。

例如：

A事务：

```
BEGIN;

SELECT *
FROM user
WHERE id=1;
```

此时创建快照。

之后其他事务提交的新版本：

```
B修改
C修改
D修改
```

对于A普通SELECT来说：

不可见。

------

# 4. 普通SELECT：快照读

普通查询：

```
SELECT *
FROM user;
```

属于：

```
快照读
```

读取规则：

```
事务开始时可见的数据版本
+
当前事务自己的修改
```

不是：

```
永远读取最新数据
```

------

## 示例

初始：

```
id | name
1  | Tom
```

A:

```
BEGIN;

SELECT *
FROM user;
```

看到：

```
Tom
```

B:

```
UPDATE user
SET name='Bob';

COMMIT;
```

A:

```
SELECT *
FROM user;
```

RR隔离级别：

仍然看到：

```
Tom
```

因为B版本不属于A的Read View。

------

# 5. 当前读

以下操作属于当前读：

```
UPDATE

DELETE

SELECT ... FOR UPDATE

SELECT ... LOCK IN SHARE MODE
```

特点：

> 读取最新提交版本。

原因：

写操作不能基于旧快照修改，否则可能覆盖别人已经提交的数据。

------

# 6. UPDATE执行过程

UPDATE不是：

```
拿快照
↓
修改
```

而是：

```
当前读
↓
读取最新版本
↓
修改
↓
生成自己的新版本
```

------

## 示例

初始：

```
id | name | age
1  | Tom  | 18
```

------

B事务：

```
UPDATE user
SET name='Bob'
WHERE id=1;

COMMIT;
```

当前：

```
id | name | age
1  | Bob  | 18
```

------

A事务：

```
UPDATE user
SET age=20
WHERE id=1;
```

A不会基于：

```
Tom 18
```

修改。

而是当前读：

```
Bob 18
```

然后生成：

```
Bob 20
```

版本链：

```
A版本:
name=Bob
age=20

        ↓

B版本:
name=Bob
age=18

        ↓

旧版本:
name=Tom
age=18
```

------

# 7. 为什么A能看到B修改的数据？

例如：

过程：

```
A SELECT
    ↓
看到 Tom

B UPDATE name=Bob
    ↓
提交

A UPDATE age=20
    ↓
当前读读取 Bob
    ↓
生成 Bob,20

A SELECT
```

结果：

```
name=Bob
age=20
```

看起来：

> A读到了B修改的name。

实际上：

A读取的是：

```
A自己生成的新版本
```

而不是：

```
B版本
```

版本关系：

```
A版本
(B的数据 + A的数据)

↓

B版本

↓

旧版本
```

------

# 8. 为什么这不算不可重复读？

不可重复读定义：

> 同一个事务中，两次相同的读取操作，由于其他事务修改，结果不同。

典型：

```
A SELECT
    ↓
Tom

B UPDATE
    ↓

A SELECT
    ↓
Bob
```

这里：

第一次读：

A快照

第二次读：

B修改后的版本

这才叫不可重复读。

------

但是：

```
A SELECT
    ↓
Tom

B UPDATE
    ↓

A UPDATE
    ↓

A SELECT
    ↓
Bob
```

原因：

不是A直接读取B版本。

而是：

```
A自己的修改版本包含了B的数据
```

属于：

> 事务自己的写入可见（Read Your Own Writes）

------

# 9. 事务自己的修改一定可见

MVCC规则：

```
可见数据 =
Read View中的版本
+
当前事务自己的修改
```

------

例如：

A:

```
BEGIN;

INSERT INTO user(id,name)
VALUES(10,'Tom');


SELECT *
FROM user
WHERE id=10;
```

虽然插入的数据不存在于事务快照：

但是：

结果：

```
10 Tom
```

因为这是A自己创建的数据。

------

# 10. MVCC无法解决的问题

MVCC解决：

| 问题         | 是否解决 |
| ------------ | -------- |
| 脏读         | ✅        |
| 不可重复读   | ✅（RR）  |
| 普通幻读     | 部分解决 |
| 业务逻辑冲突 | ❌        |

------

例如库存：

A：

```
SELECT stock
FROM product
WHERE id=1;
```

看到：

```
stock=1
```

B：

```
UPDATE product
SET stock=0;
```

A：

```
UPDATE product
SET sold=sold+1;
```

可能产生：

```
stock=0
sold+1
```

MVCC不会阻止。

因为：

MVCC保证：

```
你看到的数据版本一致
```

不是：

```
业务状态永远有效
```

------

# 11. 解决业务冲突的方法

## 方式1：悲观锁

当前读加锁：

```
SELECT *
FROM product
WHERE id=1
FOR UPDATE;
```

效果：

```
A读取
 ↓
加锁

B修改
 ↓
等待
```

------

## 方式2：乐观锁

增加版本号：

表：

```
id | stock | version

1  | 10    | 5
```

更新：

```
UPDATE product
SET stock=9,
version=6
WHERE id=1
AND version=5;
```

如果别人修改过：

```
version=6
```

更新失败：

```
影响行数=0
```

说明数据已经变化。

------

# 12. 核心总结

## 读取

普通SELECT：

```
快照读
↓
Read View
↓
历史版本
```

------

## 写入

UPDATE/DELETE：

```
当前读
↓
最新提交版本
↓
生成新版本
```

------

## 可见性

```
事务看到的数据

=
事务开始时快照

+

自己产生的修改
```

------

## 最重要的一句话

> MVCC保证的是“读到哪个版本”，不是“整个事务期间所有操作看到同一个世界”。

> 普通SELECT追求一致快照，UPDATE追求基于最新数据修改，因此同一个事务里快照读和当前读可能看到不同的数据版本。