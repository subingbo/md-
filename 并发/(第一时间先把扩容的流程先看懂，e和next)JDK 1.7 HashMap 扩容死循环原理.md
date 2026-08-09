# JDK 1.7 HashMap 扩容死循环原理

## 1. HashMap 为什么线程不安全？

HashMap 在多线程环境下可能出现：

### 1.1 put 数据覆盖

两个线程同时 put：

```java
map.put("a", 1);
map.put("a", 2);
```

执行过程：

线程 A：

```
计算 hash
找到桶位置
准备插入
```

线程 B：

```
计算 hash
找到桶位置
准备插入
```

因为没有锁保护，可能：

```
线程A的数据覆盖线程B的数据
```

导致数据丢失。

------

### 1.2 扩容时链表结构异常

HashMap 扩容时需要迁移节点。

多线程同时 resize：

- 多个线程同时修改 Entry 节点
- 修改 next 指针
- 可能导致链表结构错误

JDK 1.7 最大的问题：

> 扩容采用头插法，多线程 resize 可能形成环形链表。

------

# 2. JDK 1.7 HashMap 数据结构

JDK 1.7：

```
数组 + 链表
```

结构：

```
table[0]

    Entry
      |
      v
      A -> B -> C -> null
```

数组保存的是链表头节点：

```java
table[0] = A;
```

表示：

```
table[0]
   |
   v
   A
   |
   v
   B
   |
   v
   C
```

------

# 3. JDK 1.7 扩容为什么需要迁移？

HashMap 扩容：

```
旧数组容量
        ↓
新数组容量
```

例如：

```
oldTable

[0]
 |
 A -> B -> C


扩容后：

newTable

[0]
[1]
[2]
[3]
...
```

因为数组长度变化：

```
hash % oldCapacity
```

变成：

```
hash % newCapacity
```

所以节点需要重新计算位置。

------

# 4. JDK 1.7 扩容核心代码

简化：

```java
while(e != null){

    Entry next = e.next;

    e.next = newTable[index];

    newTable[index] = e;

    e = next;
}
```

关键：

```java
e.next = newTable[index];
```

这就是：

# 头插法

------

# 5. 什么是头插法？

假设新数组某个桶：

```
newTable[0] = null
```

原链表：

```
A -> B -> C
```

迁移过程：

------

## 第一次迁移 A

当前：

```
e = A
```

保存下一个节点：

```java
next = e.next;
```

得到：

```
next = B
```

然后：

```java
A.next = newTable[0];
```

因为：

```
newTable[0] = null
```

所以：

```
A.next = null
```

此时：

```
A -> null
```

然后：

```java
newTable[0] = A;
```

结果：

```
newTable[0]

A -> null
```

------

## 第二次迁移 B

当前：

```
e = B
```

保存：

```java
next = B.next;
```

得到：

```
next = C
```

执行：

```java
B.next = newTable[0];
```

注意：

现在：

```
newTable[0] = A
```

所以：

```
B.next = A
```

结果：

```
B -> A
```

然后：

```java
newTable[0] = B;
```

现在：

```
newTable[0]

B -> A -> null
```

------

## 第三次迁移 C

执行：

```java
C.next = newTable[0];
```

此时：

```
newTable[0] = B
```

所以：

```
C -> B
```

然后：

```java
newTable[0] = C;
```

最终：

```
C -> B -> A -> null
```

------

# 6. 头插法的核心特点

原链表：

```
A -> B -> C
```

迁移后：

```
C -> B -> A
```

特点：

- 新节点永远插入链表头部
- 链表顺序反转
- 会修改每个节点的 next 指针

关键代码：

```java
e.next = newTable[index];

newTable[index] = e;
```

第一句：

让当前节点指向旧链表头。

第二句：

让数组指向当前节点。

------

# 7. JDK 1.7 扩容死循环过程

## 原链表

```
A -> B -> null
```

------

## 线程 T1

开始迁移 A：

```java
next = A.next;
```

保存：

```
next = B
```

修改：

```java
A.next = null;
```

现在：

```
A -> null
```

线程暂停。

------

## 线程 T2

继续扩容。

迁移 A：

```
newTable:

A -> null
```

迁移 B：

执行：

```java
B.next = newTable[0];
```

此时：

```
newTable[0] = A
```

所以：

```
B.next = A
```

得到：

```
B -> A
```

------

## T1 恢复

T1之前保存：

```
next = B
```

继续执行。

由于 T2 已经修改：

```
B.next = A
```

T1继续修改：

```
A.next = B
```

最终：

```
A -> B
↑    |
|____|
```

形成环形链表。

------

# 8. 为什么 get 会死循环？

HashMap 查询：

```java
while(e != null){

    if(e.key.equals(key)){
        return e.value;
    }

    e = e.next;
}
```

正常：

```
A -> B -> null
```

最后：

```
e = null
```

结束。

但是环：

```
A -> B
↑    |
|____|
```

遍历：

```
A
B
A
B
A
B
...
```

永远不会：

```
e = null
```

导致：

```
死循环
CPU 飙升
```

------

# 9. JDK 1.8 为什么解决？

JDK 1.8：

不再使用头插法。

采用：

```
尾插法思想
```

迁移时保持原链表顺序。

例如：

JDK 1.7：

```
A -> B -> C

变成：

C -> B -> A
```

JDK 1.8：

```
A -> B -> C

保持：

A -> B -> C
```

因此：

- 不会因为链表反转导致环
- resize 死循环问题解决

但是：

注意：

HashMap 1.8 仍然不是线程安全。

仍可能：

- put 覆盖
- 数据丢失
- 并发修改异常

------

# 面试回答模板

> HashMap 在多线程环境下不是线程安全的，put 操作可能导致数据覆盖和数据丢失。
>
> JDK1.7 中 HashMap 扩容采用头插法迁移链表，迁移过程中会修改节点的 next 指针。多个线程同时 resize 时，由于共享 Entry 节点，一个线程修改后的链表可能被另一个线程继续修改，导致 next 指针形成环形结构。
>
> 当调用 get 查询时，链表遍历无法到达 null，从而出现死循环。
>
> JDK1.8 使用尾插法思想优化扩容，避免了链表环的问题，但 HashMap 依然不是线程安全的。

```

```