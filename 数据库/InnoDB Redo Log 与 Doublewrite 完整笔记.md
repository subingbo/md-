# InnoDB Redo Log 与 Doublewrite 完整笔记

## 1. 先记住最核心的区别

```text
Redo Log
→ 记录“数据页发生了什么修改”

Doublewrite
→ 保存“完整的数据页副本”，防止正式数据页发生撕裂
```

最核心的关系：

```text
Doublewrite：先保证 Page 本体完整
        ↓
Redo Log：再补上这个 Page 后续缺失的修改
```

---

# 2. 为什么需要 Doublewrite？

InnoDB 的一个数据页通常是 **16KB**。

假设内存中的 Page：

```text
┌──────────────────────┐
│ A' B' C' D' E' F' G' │
│ H' ...                │
└──────────────────────┘
```

需要写入 `.ibd` 文件。

很多人会误以为：

```text
写 16KB
↓
要么全部成功
要么全部失败
```

实际上，Page 写入过程中可能发生：

```text
写了一部分
↓
机器宕机
↓
剩余部分没有写完
```

例如原来的 Page：

```text
┌──────────────────────┐
│ A  B  C  D  E  F  G  H│
└──────────────────────┘
```

新 Page：

```text
┌──────────────────────┐
│ A' B' C' D' E' F' G' H'│
└──────────────────────┘
```

如果写到一半宕机：

```text
┌──────────────────────┐
│ A' B' C' D  E  F  G  H│
└──────────────────────┘
```

这就是：

> **Torn Page / Page Split Write / 部分页写（页撕裂）**

此时这个 Page 本身已经不是一个完整、可靠的 Page。

---

# 3. 为什么 Redo Log 不能简单替代 Doublewrite？

Redo Log 记录的是：

```text
Page X：
A → A'
B → B'
C → C'
...
```

它本质上是：

```text
“对数据页做了什么修改”
```

而不是：

```text
“完整的 16KB Page 是什么”
```

所以发生 Page 撕裂后：

```text
.ibd：

┌──────────────────┐
│ 新的一半         │
│ 旧的一半         │
└──────────────────┘
```

这时候首先需要的是：

```text
一个完整、可靠的 Page
```

而不是直接在一个已经不完整的 Page 上继续假设它是正常的。

因此恢复逻辑是：

```text
撕裂的 .ibd Page
        ↓
Doublewrite 提供完整 Page
        ↓
先恢复完整 Page
        ↓
Redo Log 再补后续修改
```

所以：

```text
Doublewrite
→ 解决“Page 本体坏了”

Redo Log
→ 解决“Page 的修改还没有全部落到数据文件”
```

---

# 4. Doublewrite 到底是什么？

Doublewrite 不是简单的“再写一次”。

它的本质是：

> **在正式修改 `.ibd` 之前，先把完整 Page 持久化保存一份。**

流程：

```text
Buffer Pool
    │
    │ 脏 Page
    ↓
Doublewrite Buffer
    │
    │ 写入 + Flush/Fsync
    ↓
磁盘上的 Doublewrite 区域
    │
    │ 已经存在完整副本
    ↓
再写真正的 .ibd Page
```

所以顺序非常重要：

```text
① Doublewrite 副本先持久化
② 再写真正的 .ibd Page
```

---

# 5. Doublewrite Buffer 是内存还是磁盘？

两者都有。

## 5.1 内存中的 Doublewrite Buffer

刷脏页时，Page 会先进入内存中的 Doublewrite Buffer。

```text
Buffer Pool
    ↓
内存中的 Doublewrite Buffer
```

它只是准备阶段。

---

## 5.2 磁盘上的 Doublewrite 区域

之后会把这些 Page 写入磁盘上的 Doublewrite 区域。

```text
内存 Doublewrite Buffer
        ↓
磁盘 Doublewrite 区域
        ↓
Flush / Fsync
```

真正用于**崩溃恢复**的是：

> 已经成功持久化到磁盘的 Doublewrite 副本。

所以不能理解成：

```text
Doublewrite 永远只是内存
```

也不能理解成：

```text
Doublewrite 只有一个固定内存区域
```

而应该理解成：

```text
内存中的 Doublewrite Buffer
+
磁盘上的 Doublewrite 区域
```

---

# 6. 为什么 Doublewrite 能解决 Page 撕裂？

最关键的是下面这个时间顺序。

## 第一步：生成完整 Page

```text
Buffer Pool：

┌──────────────────┐
│ A' B' C' D' E' F' │
│ G' H' ...         │
└──────────────────┘
```

---

## 第二步：先写 Doublewrite

```text
Buffer Pool
    ↓
Doublewrite
    ↓
Flush / Fsync
```

此时磁盘上有：

```text
Doublewrite：

┌──────────────────┐
│ A' B' C' D' E' F' │
│ G' H' ...         │
└──────────────────┘
```

完整副本已经持久化。

---

## 第三步：再写 `.ibd`

```text
Doublewrite 已经安全保存
        ↓
开始写 .ibd
```

假设 `.ibd` 写一半突然宕机：

```text
Doublewrite：

┌──────────────────┐
│ A' B' C' D' E' F' │
│ G' H' ...         │
└──────────────────┘
完整 ✓


.ibd：

┌──────────────────┐
│ A' B' C' D'      │
│ E  F  G  H       │
└──────────────────┘
撕裂 ✗
```

---

# 7. 崩溃恢复时怎么办？

启动恢复时发现 `.ibd` Page 已经损坏。

此时：

```text
.ibd 损坏 Page
       ↓
Doublewrite 中寻找完整副本
       ↓
恢复完整 Page
       ↓
再应用 Redo Log
```

最终得到：

```text
完整 Page + Redo 修改
```

---

# 8. 为什么不能直接把 `.ibd` Page 丢掉，然后靠 Redo 重做？

因为 `.ibd` 是**正式数据**。

一个 Page 里面可能已经包含：

```text
以前已经提交的事务
+
这次新的事务修改
+
更多历史数据
```

例如：

```text
.ibd Page：

旧数据
旧数据
旧数据
本次修改
```

如果直接：

```text
整个 Page 丢掉
```

那么之前已经持久化的数据也会一起丢失。

而 Redo Log 主要是：

```text
记录修改
```

而不是：

```text
完整保存整个 Page
```

所以不能简单理解为：

```text
丢掉 .ibd Page
↓
只靠 Redo
```

正确方式：

```text
Doublewrite
↓
找回完整 Page

Redo Log
↓
补上这个 Page 后续尚未完全落盘的修改
```

---

# 9. Doublewrite 自己也可能写坏，怎么办？

这是一个很容易产生的疑问：

> Doublewrite 也是写磁盘，难道它自己不会写一半吗？

会存在这种可能。

但 Doublewrite 的关键不在于：

```text
“Doublewrite 永远不会坏”
```

而在于：

```text
只有在 Doublewrite 副本已经成功 Flush/Fsync 后，
才继续覆盖真正的 .ibd Page。
```

因此有明确的先后关系：

```text
Doublewrite 还没有安全落盘
        ↓
不能依赖它恢复正式 Page
        ↓
此时正式 Page 还没有进入对应的覆盖阶段
```

而：

```text
Doublewrite 已经安全落盘
        ↓
才开始写 .ibd
        ↓
此时即使 .ibd 撕裂
        ↓
Doublewrite 中已有完整副本
```

所以真正的核心不是：

> Doublewrite 比 `.ibd` 更不容易写坏。

而是：

> **先把完整副本持久化，再去修改正式 Page。**

---

# 10. Doublewrite 和 `.ibd` 都是磁盘，为什么有区别？

不是因为：

```text
Doublewrite = 磁盘
.ibd        = 磁盘
```

所以两者完全一样。

区别在于：

```text
Doublewrite：
先写、先持久化、保存完整副本

.ibd：
后写，是最终正式数据位置
```

可以看成：

```text
                先
内存 Page ───────────→ Doublewrite
                         ↓
                       fsync
                         ↓
                  完整副本已存在
                         ↓
                       后
                         ↓
                    .ibd 正式位置
                         ↓
                   可能发生撕裂
```

因此 Doublewrite 提供的是：

> **一个已经持久化的“完整 Page 备份”。**

---

# 11. 顺序 I/O 是不是 Doublewrite 能恢复的根本原因？

不是。

这是一个非常重要的区分。

## 为什么 Doublewrite 能恢复？

根本原因：

```text
完整 Page
    ↓
先持久化到 Doublewrite
    ↓
再写 .ibd
```

也就是：

> **先建立可靠副本，再写正式位置。**

---

## 那为什么 Doublewrite 性能没有简单翻倍？

因为 Doublewrite 区域主要是：

```text
批量写入
+
连续写入
+
统一 Flush/Fsync
```

而真正的 `.ibd` Page 写入可能分散在很多位置。

所以：

```text
Doublewrite 的顺序写
+
正式数据页的写入
```

比简单理解成：

```text
同一个 Page 完完整整写两次
```

更准确。

因此：

```text
恢复能力的核心
→ 持久化顺序 + 完整副本

性能方面的重要特点
→ 批量、连续写 Doublewrite
```

两者不要混淆。

---

# 12. Doublewrite 会不会进行 GC？

严格来说：

> **Doublewrite 不是传统意义上的 GC。**

它是一个可以循环复用的区域。

流程类似：

```text
这一批 Page：

Buffer Pool
    ↓
Doublewrite 区域
    ↓
fsync
    ↓
.ibd
```

等这一批 Page 已经完成正式写入后：

```text
这一批 Doublewrite 副本
        ↓
不再需要
        ↓
下一批直接覆盖复用
```

所以不是：

```text
写坏
↓
GC
↓
删除
```

而是：

```text
旧副本失效
↓
空间循环复用
↓
下一批 Page 覆盖
```

---

# 13. 最重要的三种崩溃情况

## 情况一：还没开始写 Doublewrite

```text
Buffer Pool
    ↓
还没把副本持久化
    ↓
宕机
```

此时正式 `.ibd` Page 还没有被这次刷盘覆盖。

主要依赖：

```text
Redo Log
```

恢复修改。

---

## 情况二：Doublewrite 已持久化，但 `.ibd` 还没写

```text
Doublewrite ✓
    ↓
还没写 .ibd
    ↓
宕机
```

正式 `.ibd` 还是完整的。

重启时可以利用：

```text
Doublewrite + Redo
```

完成恢复。

---

## 情况三：Doublewrite 已持久化，`.ibd` 正在写时宕机

这是 Doublewrite 最重要的场景：

```text
Doublewrite：
完整 Page ✓

.ibd：
Page 撕裂 ✗
```

恢复：

```text
Doublewrite
    ↓
恢复完整 Page
    ↓
Redo Log
    ↓
补修改
```

---

# 14. Redo Log + Doublewrite 的完整关系

可以直接记这个图：

```text
                    脏 Page
                       │
                       ↓
                 Buffer Pool
                       │
                       ↓
               Doublewrite Buffer
                       │
                 Write + Fsync
                       │
                       ↓
            磁盘 Doublewrite 区域
                       │
                 完整 Page ✓
                       │
                       ↓
                  写真正 .ibd
                       │
                       ↓
                ┌──────┴──────┐
                │             │
              正常          崩溃
                │             │
                ↓             ↓
             完成写入      可能 Page 撕裂
                              │
                              ↓
                       Doublewrite 恢复
                              │
                              ↓
                        完整 Page
                              │
                              ↓
                         Redo Log
                              │
                              ↓
                         最终 Page
```

---

# 15. 一句话理解

```text
Doublewrite：
“我先把完整 Page 保存一份，再去写正式 Page。”

Redo Log：
“如果正式 Page 的修改还没完全落盘，我记录了应该怎么修改。”
```

因此：

```text
Doublewrite
→ 防止 Page 撕裂导致 Page 本体损坏

Redo Log
→ 保证已经发生的 Page 修改最终能够恢复
```

最终恢复模型：

```text
先用 Doublewrite：
把“坏掉的 Page”恢复成“完整 Page”

再用 Redo：
把“缺失的修改”补回来
```

# 16. 面试版答案

**问：为什么 InnoDB 需要 Doublewrite Buffer？**

可以回答：

> InnoDB 的数据页可能在写入磁盘时发生部分写，导致 Page 撕裂。Redo Log 主要记录对数据页的修改，并不是完整 Page 的可靠副本，所以不能简单把已经撕裂的 Page 当成正常 Page 直接恢复。InnoDB 会先把完整 Page 持久化到 Doublewrite 区域，再写真正的数据文件。如果后续 `.ibd` Page 发生撕裂，崩溃恢复时可以先从 Doublewrite 找回完整 Page，再结合 Redo Log 重放尚未完成的修改，从而保证数据页的一致性。

最核心的一句话：

> **Doublewrite 保证“Page 本体可以找回来”，Redo Log 保证“Page 上的修改可以重放”。**