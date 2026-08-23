# MySQL InnoDB：Buffer Pool、Undo Log、Redo Log、Binlog、WAL 与两阶段提交

> 本笔记整理本次对话中关于 MySQL 更新流程、Buffer Pool、Undo Log、Redo
> Log、Binlog、WAL、LSN、刷盘以及 2PC
> 的完整讨论，并结合本次提供的技术文章《Redo Logging in
> InnoDB》的内容进行统一梳理。
>
> 核心原则：**先区分"生成/修改""写入文件""flush/fsync
> 持久化""数据页刷盘"这几个不同概念。**

------------------------------------------------------------------------

## 1. MySQL UPDATE 的整体分层

MySQL 可以粗略分成：

``` text
                    MySQL Server
┌────────────────────────────────────────────┐
│ SQL → 优化器 → 执行器                     │
│                                            │
│ 负责执行 SQL、计算更新前后的记录           │
└──────────────────┬─────────────────────────┘
                   │
                   │ 调用存储引擎接口
                   ↓
┌────────────────────────────────────────────┐
│                  InnoDB                    │
│                                            │
│ Buffer Pool                                │
│ Undo Log                                   │
│ Redo Log                                   │
│ 数据页、索引页、事务等                     │
└────────────────────────────────────────────┘
```

### Buffer Pool 是谁的？

**Buffer Pool 属于 InnoDB。**

它不是 MySQL Server 执行器自己的缓存。

数据页通常先从磁盘加载到 Buffer Pool，之后 InnoDB
在内存中的页副本上执行修改。

------------------------------------------------------------------------

# 2. UPDATE 时，执行器到底做了什么？

例如：

``` sql
UPDATE user SET age = 21 WHERE id = 1;
```

假设原记录：

``` text
id = 1
name = 张三
age = 20
```

执行器可以得到：

``` text
更新前记录：
id = 1
name = 张三
age = 20

更新后记录：
id = 1
name = 张三
age = 21
```

这里的"更新后记录"非常容易误解。

## 它不是已经修改后的数据库记录

它实际上是：

> **执行器根据 UPDATE 语句计算出来的目标记录。**

此时真正的 InnoDB 数据页可能仍然是：

``` text
age = 20
```

执行器比较：

``` text
old record == new record ?
```

### 如果一样

例如：

``` sql
UPDATE user SET age = 20 WHERE id = 1;
```

那么：

``` text
old:
age = 20

new:
age = 20
```

两者相同，可以不继续执行真正的数据更新。

### 如果不一样

``` text
old:
age = 20

new:
age = 21
```

执行器把更新所需的信息交给 InnoDB。

然后由 InnoDB 真正修改 Buffer Pool 中的数据页。

因此：

> **执行器负责计算"要改成什么"，InnoDB 负责真正修改数据页。**

------------------------------------------------------------------------

# 3. Buffer Pool 与数据页

InnoDB 的数据文件由页组成。

读取或修改一个页时，需要把它加载到内存。

因此可以理解成：

``` text
磁盘
  │
  │ 数据页
  ↓
Buffer Pool
  │
  │ 修改
  ↓
Dirty Page（脏页）
```

所谓脏页：

> **Buffer Pool
> 中已经被修改，但对应磁盘数据页还没有同步到最新状态的页。**

例如：

``` text
磁盘：
age = 20

Buffer Pool：
age = 21
```

此时 Buffer Pool 中的页就是脏页。

------------------------------------------------------------------------

# 4. 为什么不直接把修改后的 B+ 树数据页写到磁盘？

核心原因之一：

> **数据页的物理位置通常是分散的，直接频繁持久化数据页可能产生大量随机
> I/O。**

假设事务修改：

``` text
Page C
Page A
Page D
Page B
```

这些页在磁盘上的位置可能完全不同：

``` text
C → A → D → B
```

如果每次修改都立即把数据页持久化：

``` text
修改 Page C → 写磁盘
修改 Page A → 写磁盘
修改 Page D → 写磁盘
修改 Page B → 写磁盘
```

可能产生大量随机 I/O。

------------------------------------------------------------------------

# 5. Redo Log 的核心价值

Redo Log **并没有把"修改 Page C、A、D、B"本身变成顺序操作**。

这一点非常重要。

真正发生的是：

``` text
随机修改 Buffer Pool 中的数据页
        ↓
产生对应的 Redo Log
        ↓
把 Redo Log 顺序追加写入 Redo Log 文件
        ↓
事务可以完成持久性要求
        ↓
数据页以后再刷盘
```

所以准确说法是：

> **Redo Log
> 把"事务提交时必须完成的持久化工作"，从随机的数据页写入转移成了更适合顺序写的日志写入。**

而不是：

> "Redo Log 把随机的数据页修改变成了顺序修改。"

------------------------------------------------------------------------

# 6. Redo Log 的完整生命周期

本次提供的技术文章明确描述了 Redo Log 的生命周期。

## 6.1 修改内存中的数据页

文章指出：

> page 的修改首先发生在内存中的副本，修改后的页如果还没有刷盘，就成为
> dirty page。

可以表示：

``` text
磁盘 Page
   ↓
Buffer Pool
   ↓
修改
   ↓
Dirty Page
```

## 6.2 产生 Redo Log

修改页时，同时生成对应的 Redo Log。

技术文章描述：

``` text
数据页修改
    ↓
MTR Buffer
    ↓
Global Redo Log Buffer
```

MTR（Mini-Transaction）拥有自己的局部 Redo 缓冲区。

`mtr_commit()` 时，Redo Log 从 MTR Buffer 转移到全局 Redo Log Buffer。

## 6.3 Redo Log Buffer 写入 Redo Log 文件

之后：

``` text
Redo Log Buffer
      ↓
Redo Log File
```

这里必须区分：

``` text
write
```

和：

``` text
flush
```

技术文章明确说明，这两个步骤是分开的，因为中间可能存在操作系统的文件缓存。

可以抽象成：

``` text
Redo Log Buffer
      ↓
写入文件
      ↓
OS 文件缓存
      ↓
flush / fsync
      ↓
持久化存储
```

## 6.4 Dirty Page 之后再刷盘

数据页不需要和 Redo Log 同时立即刷入数据文件。

可以：

``` text
Redo Log
   ↓
先持久化

Dirty Page
   ↓
之后某个时间
   ↓
刷入数据文件
```

这就是 WAL。

------------------------------------------------------------------------

# 7. WAL（Write-Ahead Logging）

WAL 的核心规则：

> **对应数据页刷盘之前，相关 Redo Log 必须先持久化。**

也就是：

``` text
修改 Buffer Pool
      ↓
Dirty Page
      ↓
产生 Redo
      ↓
Redo 持久化
      ↓
以后 Dirty Page 再刷盘
```

不能反过来：

``` text
❌ Dirty Page 先落盘
❌ Redo 还没有持久化
```

否则如果数据页已经落盘，而机器随后崩溃，Redo
又没有持久化，就失去了可靠的恢复依据。

------------------------------------------------------------------------

# 8. Redo Log Buffer 是内存吗？

是。

InnoDB 中有：

``` text
Buffer Pool
Redo Log Buffer
```

二者都是 InnoDB 的内存结构，但作用不同。

``` text
Buffer Pool
    ↓
缓存数据页、索引页等

Redo Log Buffer
    ↓
缓存 Redo Log
```

可以理解成：

``` text
                InnoDB
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
   Buffer Pool         Redo Log Buffer
        │                   │
     数据页               Redo Log
        │                   │
        ↓                   ↓
   数据文件             Redo Log 文件
```

------------------------------------------------------------------------

# 9. Redo Log 是不是只有 COMMIT 时才写磁盘？

不是。

这是本次讨论中特别纠正的一点。

Redo Log 可能在事务 COMMIT 之前就写入并持久化。

例如：

``` text
事务执行
   ↓
产生大量 Redo
   ↓
Redo Log Buffer 快满
   ↓
提前写/flush Redo Log
   ↓
事务继续执行
   ↓
COMMIT
```

技术文章明确提到：

> Redo Log Buffer 可以在事务提交时，或者 Log Buffer 满时被写入/flush 到
> Redo Log 文件。

因此：

> **"Redo Log 只有 COMMIT 时才写盘"是不准确的。**

------------------------------------------------------------------------

# 10. COMMIT 时 Redo Log 到底要求什么？

在常见的：

``` text
innodb_flush_log_at_trx_commit = 1
```

配置下，可以理解为：

``` text
COMMIT
  ↓
确保该事务需要持久化的 Redo Log 已经完成必要的写入和 flush
  ↓
COMMIT 返回成功
```

重点：

> **事务提交成功返回之前，相关 Redo Log 必须达到配置要求的持久化状态。**

但这不意味着：

> Redo Log 必须一直等到 COMMIT 才第一次写磁盘。

正确关系是：

``` text
可能提前持久化
      ↓
COMMIT 时再次确保提交所需的 Redo 已经持久化
```

------------------------------------------------------------------------

# 11. Redo Log 为什么能保证崩溃恢复？

假设：

``` text
Buffer Pool：
age = 21

磁盘数据页：
age = 20

Redo Log：
记录了 age 20 → 21
并且已经持久化
```

突然断电。

重启后：

``` text
磁盘数据页：
age = 20

Redo Log：
仍然存在
```

InnoDB 可以通过 Redo Recovery：

``` text
Redo Log
   ↓
重新执行之前的修改
   ↓
恢复数据页
   ↓
age = 21
```

所以：

> **Redo Log 记录的是"如何把之前的修改重新做一遍"的信息。**

------------------------------------------------------------------------

# 12. LSN 是什么？

LSN：

> **Log Sequence Number**

它把：

``` text
Dirty Page
Redo Log Record
Redo Log File
```

联系起来。

技术文章指出，每个 Redo Log Record 会对应一个 LSN；数据页也会关联一个
Page LSN。

可以简单理解：

``` text
Redo：
LSN 100
LSN 200
LSN 300
LSN 400
```

某个数据页：

``` text
Page A
page_lsn = 300
```

意味着：

> 在 Page A 能够安全刷盘之前，需要确保与它相关的 Redo
> 至少已经持久化到相应的 LSN。

------------------------------------------------------------------------

# 13. Redo Log 与 Dirty Page 的核心关系

这是理解 WAL 最重要的一组关系：

``` text
             Buffer Pool
                 │
                 ↓
            修改 Page A
                 │
                 ├────→ Page A 变 Dirty
                 │
                 ↓
             产生 Redo
                 │
                 ↓
            获得 LSN
                 │
                 ↓
        Redo Log 持久化
                 │
                 ↓
       Page A 才可以安全刷盘
```

技术文章明确指出：

> Redo Log 必须在对应 Dirty Page 刷盘之前 flush 到磁盘。

------------------------------------------------------------------------

# 14. Undo Log 是干什么的？

Undo Log 主要用于：

1.  **事务回滚**
2.  **MVCC / 旧版本读取**

例如：

``` text
原来：
age = 20

UPDATE：
age = 21
```

需要保留能够描述旧状态的信息：

``` text
Undo：
原来 age = 20
```

这样事务回滚时可以：

``` text
21 → 20
```

------------------------------------------------------------------------

# 15. Undo Log 是否必须先"单独落盘"？

这里需要非常严格地区分：

### 逻辑顺序

Undo 信息必须在进行不可逆的数据修改过程中得到准备。

可以理解为：

``` text
准备 Undo 信息
      ↓
修改数据页
```

因为如果没有旧版本信息，就无法正常进行事务回滚和 MVCC 所需的版本管理。

### 但持久化顺序不是：

``` text
❌ Undo fsync
   ↓
❌ 数据页修改
```

Undo Log 本身也是 InnoDB 管理的数据结构，其相关页的修改也可以由 Redo Log
保护。

可以抽象为：

``` text
修改 Undo 页
修改数据页
      ↓
产生对应 Redo
      ↓
Redo 持久化
      ↓
Undo 页和数据页以后再刷盘
```

因此：

> **Undo 最终当然会持久化，但不等于每次修改都必须先把 Undo 单独 fsync
> 到磁盘。**

------------------------------------------------------------------------

# 16. 如果 Undo 没有落盘，机器崩溃，怎么回滚？

这是本次讨论中最重要的问题之一。

假设：

``` text
Buffer Pool：

数据页：
age = 21

Undo 页：
old age = 20
```

但是：

``` text
数据页还没刷盘
Undo 页还没刷盘
```

只要相关 Redo 已经持久化，就仍然可以恢复。

因为：

``` text
Undo 页的修改
        ↓
也有对应的 Redo
        ↓
Redo 已经持久化
```

机器崩溃：

``` text
        崩溃
          ↓
      MySQL 重启
          ↓
      Crash Recovery
          ↓
     先进行 Redo Recovery
          ↓
重新恢复 InnoDB 在崩溃前的状态
          ↓
Undo 信息也被恢复出来
          ↓
识别未提交事务
          ↓
使用 Undo 回滚未提交事务
```

所以可以把两者理解成：

> **Redo：把崩溃前的现场重新构造出来。**

> **Undo：在恢复出来的现场中撤销未提交事务。**

------------------------------------------------------------------------

# 17. Undo 和 Redo 的关系

不要理解成：

``` text
Undo = Redo 的反面
```

它们作用完全不同。

-----------------------------------------------------------------------
  项目                    Undo Log                Redo Log
----------------------- ----------------------- -----------------------
  核心目的                回滚、MVCC              崩溃恢复

  保存方向                如何回到旧状态          如何重新执行修改

  主要使用时机            回滚、MVCC              Crash Recovery

  属于                    InnoDB                  InnoDB

  是否是数据页            Undo 本身也由 InnoDB    Redo 是日志系统
                          作为数据结构管理        

  与数据页关系            保存旧版本/回滚信息     保证页修改可以恢复
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 18. Binlog 是什么？

Binlog 属于：

``` text
MySQL Server 层
```

而不是 InnoDB 专属的日志。

可以粗略理解：

``` text
MySQL Server
    │
    └── Binlog

InnoDB
    ├── Buffer Pool
    ├── Undo
    └── Redo
```

Binlog 主要用于：

-   主从复制
-   数据恢复
-   记录逻辑上的数据库变更

例如：

``` sql
UPDATE user SET age = 21 WHERE id = 1;
```

Binlog 可以记录这次事务对应的逻辑变更。

------------------------------------------------------------------------

# 19. Binlog Cache

事务执行过程中，Binlog 并不是简单地每执行一条 SQL
就立即把最终结果持久化到 Binlog 文件。

可以粗略理解：

``` text
SQL 执行
   ↓
生成 Binlog
   ↓
Binlog Cache
   ↓
事务提交阶段
   ↓
写入 Binlog 文件
   ↓
根据 sync_binlog 决定持久化策略
```

这里同样要区分：

``` text
写文件
```

和：

``` text
真正持久化
```

------------------------------------------------------------------------

# 20. Binlog 和数据页没有直接的"谁先刷盘"关系

不能简单说：

> "Binlog 必须在数据页落盘之前落盘。"

因为数据页什么时候刷盘是 Buffer Pool 的脏页刷盘机制决定的。

正确的核心关系是：

``` text
数据页：
可以很晚才刷盘

Redo：
必须先于对应数据页持久化

Binlog：
需要和 InnoDB 的事务提交通过两阶段提交保持一致
```

------------------------------------------------------------------------

# 21. 为什么需要 Redo + Binlog 两阶段提交？

因为：

``` text
Redo
```

和：

``` text
Binlog
```

属于不同系统。

如果不协调，就可能出现：

### 情况一

``` text
Binlog 已经持久化
Redo 没有持久化
机器崩溃
```

可能导致：

``` text
Binlog：
这个事务存在

InnoDB：
这个事务没有可靠恢复
```

### 情况二

``` text
Redo 已经提交
Binlog 没有持久化
机器崩溃
```

可能导致：

``` text
InnoDB：
事务已经恢复

Binlog：
没有这次事务
```

主从复制就可能出现不一致。

所以需要：

> **Redo 与 Binlog 通过两阶段提交协调。**

------------------------------------------------------------------------

# 22. 两阶段提交的核心过程

可以用简化模型理解：

``` text
事务执行
    ↓
InnoDB 修改 Buffer Pool
    ↓
产生 Undo + Redo
    ↓
SQL 执行完成
    ↓
生成 Binlog
    ↓
Redo Prepare
    ↓
Binlog 持久化
    ↓
Redo Commit
    ↓
事务完成
```

更直观：

``` text
              事务执行
                  ↓
            修改 Buffer Pool
                  ↓
             Redo Prepare
                  ↓
           ┌─────────────┐
           │ Binlog 写入  │
           │ / 持久化     │
           └──────┬──────┘
                  ↓
             Redo Commit
                  ↓
                COMMIT
```

这里的"两阶段"主要是：

``` text
PREPARE
COMMIT
```

------------------------------------------------------------------------

# 23. 之前那张笔记中的哪些说法需要修正？

## 问题 1

原表述：

> 先更新数据，再把记录写到 Redo Log。

不够准确。

应该理解成：

``` text
修改 Buffer Pool 数据页
        ↕
产生对应 Redo
        ↓
MTR Buffer
        ↓
Global Redo Log Buffer
```

Redo 的生成与页修改是紧密伴随的，而不是"数据修改全部结束之后才生成"。

------------------------------------------------------------------------

## 问题 2

原表述：

> 事务提交时才把 Redo Log 写入磁盘。

错误。

Redo Log 可能：

``` text
事务执行过程中
```

就已经写入并 flush。

例如：

``` text
Redo Log Buffer 快满
      ↓
提前 flush
```

提交时只是需要确保提交所需的 Redo 达到配置要求的持久化状态。

------------------------------------------------------------------------

## 问题 3

原表述：

> Binlog 只有事务提交时才刷新到硬盘。

需要加条件。

应该区分：

``` text
Binlog Cache
      ↓
写 Binlog 文件
      ↓
是否立即 fsync
```

具体持久化行为受：

``` text
sync_binlog
```

等配置影响。

------------------------------------------------------------------------

## 问题 4

原表述：

> Undo Log 必须先单独落盘，然后才能修改数据。

不准确。

更准确：

``` text
Undo 信息需要先准备
        ↓
修改数据
        ↓
相关页修改由 Redo 保护
        ↓
Redo 持久化
        ↓
Undo 页以后可以刷盘
```

Undo 的持久性不要求每次都先独立 fsync。

------------------------------------------------------------------------

# 24. 最终的 UPDATE 完整模型

把所有东西放在一起：

``` text
客户端
  │
  │ UPDATE
  ↓
MySQL Server
  │
  │ 执行器
  │
  ├── 找到旧记录
  │
  ├── 计算新记录
  │
  └── old == new ?
          │
      ┌───┴────┐
      │        │
     是        否
      │        │
    不更新     ↓
           InnoDB
              │
              ↓
        Buffer Pool
              │
       ┌──────┴──────┐
       ↓             ↓
   修改数据页       Undo 信息
       │             │
       ↓             ↓
     Dirty        Undo 页修改
       │             │
       └──────┬──────┘
              ↓
          产生 Redo
              ↓
          MTR Buffer
              ↓
      Global Redo Buffer
              ↓
       Redo Log 文件
              ↓
         flush / 持久化
              │
              │
      SQL 执行过程中产生 Binlog
              ↓
         Binlog Cache
              ↓
        事务提交阶段
              ↓
        Redo Prepare
              ↓
        Binlog 写入/同步
              ↓
         Redo Commit
              ↓
            COMMIT
              ↓
       Dirty Page 暂时继续留在
          Buffer Pool
              ↓
       后台 checkpoint / 刷脏
              ↓
          数据文件
```

------------------------------------------------------------------------

# 25. 崩溃恢复模型

假设事务还没完成就崩溃：

``` text
Buffer Pool
├── 数据页最新修改
└── Undo 页最新修改

Redo Log
└── 已经持久化的 Redo
```

重启：

``` text
             MySQL Crash
                  ↓
            Crash Recovery
                  ↓
             Redo Recovery
                  ↓
       根据 Redo 恢复 InnoDB 页面
                  ↓
        数据页 + Undo 页恢复
                  ↓
        判断哪些事务没有提交
                  ↓
             Undo Rollback
                  ↓
             恢复一致状态
```

所以：

``` text
Redo = 重做
Undo = 回滚
```

二者不是竞争关系，而是配合关系。

------------------------------------------------------------------------

# 26. 一句话理解三种日志

``` text
Undo Log
→ “如果这个事务没提交，怎么把它撤回来？”

Redo Log
→ “如果机器崩溃，怎么把已经发生的 InnoDB 修改重新做出来？”

Binlog
→ “这个事务在逻辑上对数据库做了什么变化？”
```

------------------------------------------------------------------------

# 27. 一句话理解 WAL

> **先让 Redo Log 达到可靠持久化状态，再允许对应的数据页真正刷盘。**

``` text
Buffer Pool 修改
      ↓
产生 Redo
      ↓
Redo 持久化
      ↓
Dirty Page
      ↓
以后刷数据文件
```

------------------------------------------------------------------------

# 28. 一句话理解两阶段提交

> **Redo 和 Binlog
> 属于不同层次，必须保证它们对同一个事务的状态一致，因此使用 Prepare →
> Binlog → Commit 的协调过程。**

``` text
Redo Prepare
     ↓
Binlog 持久化
     ↓
Redo Commit
```

------------------------------------------------------------------------

# 29. 最终记忆版

如果面试时只剩 30 秒，可以这样回答：

``` text
UPDATE 时，InnoDB 先在 Buffer Pool 中修改数据页，
数据页变成 Dirty Page，同时产生 Undo 和 Redo。

Undo 用于事务回滚和 MVCC；
Redo 用于崩溃恢复。

Redo Log 先进入内存中的 Redo Log Buffer，
之后写入并 flush 到 Redo Log 文件。
根据 WAL 原则，对应的 Redo 必须先于 Dirty Page 持久化，
数据页可以之后再刷入数据文件。

Binlog 属于 MySQL Server 层，主要用于主从复制和数据恢复。
事务提交时，Redo 和 Binlog 通过两阶段提交协调：
Redo Prepare → Binlog 持久化 → Redo Commit。

如果发生崩溃，先通过 Redo Recovery 恢复 InnoDB 的状态，
包括必要的 Undo 信息，然后再利用 Undo 回滚未提交事务。
```

------------------------------------------------------------------------

## 30. 本次技术文章中的关键依据

本次提供的《Redo Logging in InnoDB》重点支持以下结论：

-   数据页修改首先发生在内存副本，之后成为 Dirty
    Page。fileciteturn0file0L15-L19
-   修改页时生成 Redo，Redo 从 MTR Buffer 转移到全局 Redo Log
    Buffer。fileciteturn0file0L17-L19
-   Redo Log 写文件和 flush
    是两个不同步骤。fileciteturn0file0L17-L20
-   Redo 必须先于对应 Dirty Page 刷盘，这是
    WAL。fileciteturn0file0L20-L24
-   Redo Log Buffer 可以在事务提交或 Buffer 满等情况下被
    flush。fileciteturn0file0L122-L128
-   LSN 将 Dirty Page、Redo Log Record 和 Redo Log File
    联系起来。fileciteturn0file0L64-L69
-   Redo Log 的生命周期包括生成、进入 Log Buffer、获得 LSN、写入并 flush
    到磁盘，以及在恢复阶段重新执行。fileciteturn0file0L183-L190
