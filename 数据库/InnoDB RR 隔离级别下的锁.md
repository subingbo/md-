# InnoDB RR 隔离级别下的锁

> 重点：区分 **Record Lock、Gap Lock、Next-Key Lock、Insert Intention Lock**，以及它们和 S/X 锁的关系。

## 1. S 锁和 X 锁

### Record Lock

Record Lock 是对**具体索引记录**加的锁。

| 已有锁 | 再申请 S | 再申请 X |
| ------ | -------- | -------- |
| 无     | ✅        | ✅        |
| S      | ✅        | ❌        |
| X      | ❌        | ❌        |

因此：

```
S + S → 可以
S + X → 互斥
X + S → 互斥
X + X → 互斥
```

可以简单理解为：

```
S：共享读取
X：独占操作
```

# 2. Record Lock

Record Lock 锁的是**已经存在的索引记录**。

例如：

```
id
10
20
30
```

如果事务执行：

```
SELECT * FROM t
WHERE id = 20
FOR UPDATE;
```

对 `20` 加的是：

```
Record X Lock
```

即：

```
10    [20]    30
       🔒
       X
```

其他事务不能再对 `20` 加冲突的 Record Lock。

# 3. Gap Lock

Gap Lock 锁的是**索引记录之间的间隙**，而不是某条已有记录。

例如：

```
10          20          30
|-----------|-----------|
   Gap(10,20)
```

Gap Lock 锁：

```
(10,20)
```

它最主要的作用：

> **阻止其他事务向这个 Gap 中 INSERT 新记录。**

例如：

```
T1：

Gap Lock (10,20)
```

那么：

```
INSERT INTO t(id) VALUES(15);
```

会被阻塞。

## 4. Gap Lock 没有普通 S/X 锁那样的互斥关系

这是非常容易混淆的地方。

Record Lock：

```
S + X → 冲突
X + X → 冲突
```

但是 Gap Lock 的重点不是保护已有记录，而是：

> **阻止 INSERT。**

不同事务的 Gap Lock 可以共存。

例如：

```
T1：Gap Lock (10,20)
T2：Gap Lock (10,20)
```

两者可以共存。

因此不要把：

```
Gap S Lock
Gap X Lock
```

理解成普通 Record S/X Lock 那样严格互斥。

更准确地说：

> **Gap Lock 的 S/X 不具有普通 Record Lock 那样的互斥语义；Gap Lock 主要用于阻止插入。**

# 5. Next-Key Lock

Next-Key Lock 是 InnoDB RR 下非常重要的锁类型。

定义：

```
Next-Key Lock
=
Gap Lock
+
Record Lock
```

更准确地说：

> **对某条记录的 Next-Key Lock = 该记录前面的 Gap + 该记录本身。**

例如：

```
10        20        30
|---------|---------|
```

对 `20` 的 Next-Key Lock：

```
(10,20]
```

拆开：

```
(10,20) + [20]
   ↑        ↑
 Gap      Record
```

所以：

```
Next-Key Lock(20)
=
Gap Lock(10,20)
+
Record Lock(20)
```

**注意：**

它不包含：

```
(20,30)
```

所以不能把 Next-Key Lock 理解成：

```
前 Gap + 当前记录 + 后 Gap    ❌
```

# 6. 一个非常重要的区别

必须区分：

### 单个 Next-Key Lock

例如：

```
10        20        30
|---------|---------|
```

对 `20` 的 Next-Key Lock：

```
(10,20]
```

### 一条 SQL 最终获得的全部锁

一条 SQL 可能同时产生：

```
Record Lock
Gap Lock
Next-Key Lock
Insert Intention Lock
```

因此：

> **Next-Key Lock 的定义 ≠ 一条 SQL 最终锁住的全部范围。**

例如一条范围查询，为了保护扫描范围，最终可能出现：

```
(10,20] + (20,30)
```

其中：

```
(10,20] → Next-Key Lock
(20,30) → Gap Lock
```

所以看到一个 SQL 把某个记录后面的 Gap 也保护起来时，不能说：

> “这个后面的 Gap 是前一个记录的 Next-Key Lock。”

它可能是**另外的 Gap Lock**。

# 7. 为什么非唯一索引等值查询需要保护更大的范围？

假设：

```
score
10
20
20
30
```

执行：

```
SELECT *
FROM t
WHERE score = 20
FOR UPDATE;
```

因为 `score` 是**非唯一索引**，所以数据库不能只锁住已有的两个 `20`。

否则其他事务仍然可能插入：

```
score = 20
```

从而导致新的符合条件的记录出现。

因此需要保护：

```
10        20       20        30
|---------|--------|---------|
       20 对应的范围
```

最终需要通过相应的 Record Lock / Gap Lock / Next-Key Lock 等组合，防止新的 `20` 插入。

# 8. 为什么唯一索引等值查询通常只需要 Record Lock？

假设：

```
id
10
20
30
```

`id` 是唯一索引。

执行：

```
SELECT *
FROM t
WHERE id = 20
FOR UPDATE;
```

因为：

```
id = 20
```

最多只有一条记录。

不可能再插入第二个：

```
id = 20
```

因此可以直接锁：

```
Record X Lock(20)
```

不需要为了这个等值条件额外保护 `20` 两边的 Gap。

所以可以记：

```
唯一索引等值查询
→ Record Lock

非唯一索引等值查询
→ 需要考虑 Gap / Next-Key Lock
```

# 9. UPDATE 和 DELETE 加什么锁？

## UPDATE

```
UPDATE t
SET name = 'A'
WHERE id = 20;
```

修改已有记录：

```
Record X Lock
```

即：

```
10    [20]    30
       X
```

如果是范围 UPDATE，则可能涉及 Gap / Next-Key Lock。

## DELETE

```
DELETE FROM t
WHERE id = 20;
```

删除本质上也是对记录进行修改，因此对被删除记录使用：

```
Record X Lock
```

范围 DELETE 同样可能涉及 Gap / Next-Key Lock。

# 10. SELECT FOR UPDATE 加什么锁？

```
SELECT *
FROM t
WHERE id = 20
FOR UPDATE;
```

这是**当前读 / 锁定读**。

对于需要锁住记录的情况，核心是：

```
Record X Lock
```

如果是范围查询，还可能涉及：

```
Next-Key Lock
Gap Lock
```

例如：

```
SELECT *
FROM t
WHERE id > 10 AND id <= 20
FOR UPDATE;
```

假设索引：

```
10        20        30
|---------|---------|
```

范围扫描过程中可能形成：

```
(10,20]
```

这样的 Next-Key Lock，并根据扫描边界涉及相应的 Gap Lock。

所以不要简单理解为：

```
FOR UPDATE = 只有 X Record Lock
```

正确的是：

```
FOR UPDATE
    ↓
锁定读
    ↓
记录 → X Record Lock
范围 → 可能有 Next-Key / Gap Lock
```

# 11. INSERT 加什么锁？

INSERT 比 UPDATE / DELETE 特殊。

例如：

```
INSERT INTO t(id) VALUES(20);
```

插入成功后，新记录会持有：

```
Record X Lock
```

但在真正插入之前，还涉及：

```
Insert Intention Lock
```

# 12. Insert Intention Lock 是什么？

Insert Intention Lock：

> **表示“我要在这个 Gap 中插入一条记录”。**

例如：

```
10 ---------------- 30
          Gap
```

事务 T1：

```
INSERT INTO t(id) VALUES(20);
```

事务 T2：

```
INSERT INTO t(id) VALUES(25);
```

它们都需要在：

```
(10,30)
```

这个 Gap 中插入。

它们可以分别持有 Insert Intention Lock。

重要：

```
Insert Intention
+
Insert Intention
```

通常**不会互相阻塞**。

因此同一个 Gap 中，不同位置的 INSERT 可以并发。

# 13. Insert Intention Lock 不能阻止其他 INSERT

这是最重要的区别之一。

错误理解：

```
Insert Intention Lock
→ 不让别人 INSERT
```

正确：

```
Insert Intention Lock
→ “我要 INSERT”
```

真正负责阻止 INSERT 的是：

```
Gap Lock
```

例如：

```
T1：
Gap Lock (10,30)
```

此时 T2：

```
INSERT INTO t(id) VALUES(20);
```

T2 想获得：

```
Insert Intention Lock
```

但发现：

```
Gap Lock
        ↕
Insert Intention Lock
```

发生冲突。

因此 T2 等待。

# 14. 为什么 INSERT 不能只依靠 Record X Lock？

这是理解 Insert Intention Lock 的关键。

假设：

```
10 ---------------- 30
       空 Gap
```

现在要：

```
INSERT INTO t(id) VALUES(20);
```

问题：

> `20` 这条记录还不存在。

所以此时没有办法先申请：

```
Record X Lock(20)
```

因为根本没有 `20` 这条已有记录。

必须先解决：

> **我能不能进入** `**(10,30)**` **这个 Gap？**

所以需要：

```
INSERT
 ↓
Insert Intention Lock
 ↓
检查 Gap 是否有冲突的 Gap Lock
 ↓
可以 → 插入
不可以 → 等待
 ↓
插入成功
 ↓
新记录持有 Record X Lock
```

所以：

```
Record X Lock
→ 管记录

Insert Intention Lock
→ 管“我要进入这个 Gap 插入”
```

两者不是重复的。

# 15. Gap Lock 和 Insert Intention Lock 的关系

可以用一句话记：

```
Gap Lock
→ “这里不准插”

Insert Intention Lock
→ “我要在这里插”
```

因此：

```
Gap Lock
        ↕
Insert Intention Lock
       冲突
```

而：

```
Insert Intention Lock
        ↕
Insert Intention Lock
       通常不冲突
```

这使得：

```
T1 INSERT 20
T2 INSERT 25
```

可以并发，而：

```
T1：Gap Lock(10,30)
T2：INSERT 20
```

则 T2 会等待。

# 16. 四种锁的核心区别

| 锁                    | 锁什么               | 主要作用                       |
| --------------------- | -------------------- | ------------------------------ |
| Record Lock           | 一条已有索引记录     | 防止其他事务修改/删除该记录    |
| Gap Lock              | 索引间隙             | 阻止其他事务 INSERT            |
| Next-Key Lock         | 前 Gap + 当前 Record | 同时保护记录及其前面的范围     |
| Insert Intention Lock | 准备插入的 Gap       | 表示插入意图，与 Gap Lock 协调 |

# 17. 最容易混淆的几个关系

### ① Next-Key Lock 不是前后两个 Gap

错误：

```
Next-Key Lock
= 前 Gap + Record + 后 Gap
```

正确：

```
Next-Key Lock
= 前 Gap + 当前 Record
```

### ② 一条 SQL 的最终锁范围不等于一个 Next-Key Lock

例如：

```
SQL
 ↓
多个索引位置
 ↓
多个锁
 ↓
最终保护一个更大的范围
```

所以：

```
单个 Next-Key Lock
```

和：

```
整条 SQL 最终锁住的范围
```

是两个概念。

### ③ Gap Lock 不是普通的 X Record Lock

Gap Lock：

```
锁间隙
```

Record X Lock：

```
锁记录
```

所以：

```
Gap Lock ≠ Record X Lock
```

### ④ Insert Intention Lock 不是用来阻止 INSERT 的

```
Gap Lock
→ 阻止 INSERT

Insert Intention
→ 表示自己要 INSERT
```

# 18. 最终记忆图

```
                         InnoDB 锁
                            │
             ┌──────────────┴──────────────┐
             ↓                             ↓
        Record Lock                    Gap Lock
             │                             │
       ┌─────┴─────┐                 阻止 INSERT
       ↓           ↓                       │
      S Lock      X Lock                   │
       │           │                       │
       │        独占记录                   │
       │                                   │
       └──────────────┐                    │
                      ↓                    ↓
                 Next-Key Lock       Insert Intention
                      │                    │
                      ↓                    ↓
              前 Gap + Record       “我要在这里插”
```

## 最后压缩成 5 句话

1. **Record X Lock：锁一条记录。**
2. **Gap Lock：锁一个间隙，主要为了阻止 INSERT。**
3. **Next-Key Lock = 前面的 Gap + 当前 Record。**
4. **Insert Intention Lock：表示“我要往这个 Gap 插入”，本身不是禁止别人 INSERT 的锁。**
5. **一条 SQL 最终可以同时产生多个不同类型的锁，不能把整个 SQL 的锁定范围等同于一个 Next-Key Lock。**