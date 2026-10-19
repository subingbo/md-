# ConcurrentHashMap 原理与线程安全完整笔记（JDK 8）

## 一、ConcurrentHashMap 是什么？

`ConcurrentHashMap` 是 Java 提供的线程安全哈希表，位于 `java.util.concurrent` 包中，适用于多线程并发读写场景。

与 `HashMap` 相比：

| 集合              | 线程安全 | 主要实现方式                         |
| ----------------- | -------- | ------------------------------------ |
| HashMap           | 否       | 不提供并发同步机制                   |
| Hashtable         | 是       | synchronized 锁住整个 Hashtable 对象 |
| ConcurrentHashMap | 是       | CAS、synchronized、volatile 等机制   |

**核心目标：既要保证线程安全，又尽量减少线程之间的锁竞争。**

## 二、底层数据结构

### 1. JDK 7 与 JDK 8 的区别

**JDK 7：Segment + HashEntry**

```
ConcurrentHashMap
    |
    |-- Segment 0（ReentrantLock）
    |      |-- HashEntry 数组
    |      |-- 链表
    |
    |-- Segment 1（ReentrantLock）
    |      |-- HashEntry 数组
    |      |-- 链表
    |
    |-- Segment 2（ReentrantLock）
           |-- HashEntry 数组
           |-- 链表
```

Segment 的中文意思是“段”。

JDK 7 将整个哈希表划分成多个 Segment，每个 Segment 继承 `ReentrantLock`，负责保护自己管理的数据。

不同 Segment 可以并发修改，同一 Segment 中的写操作需要竞争锁。

这种方案叫作**分段锁**。

**JDK 8：Node 数组 + 链表 + 红黑树**

```
ConcurrentHashMap
    |
    |-- table[0] → Node A → Node B → null
    |
    |-- table[1] → null
    |
    |-- table[2] → Node C → Node D → null
    |
    |-- table[3] → TreeBin（红黑树）
```

JDK 8 取消了 Segment 分段锁，主要通过以下方式保证线程安全：

- 空桶写入：CAS。
- 非空桶写入：synchronized。
- 并发读取：volatile、安全发布及专门的并发访问机制。
- 扩容：CAS 协调和多线程协助迁移。

**区别：JDK 7 主要按 Segment 加锁；JDK 8 通常将锁粒度缩小到桶级别。**

### 2. 数组容量、元素数量、链表长度

三个概念必须区分。

```
              table 数组
           ┌─────────────┐
下标 0     │ Node A      │ → Node B → null
           ├─────────────┤
下标 1     │ null        │
           ├─────────────┤
下标 2     │ Node C      │ → Node D → Node E
           ├─────────────┤
下标 3     │ Node F      │ → null
           └─────────────┘
```

| 概念              | 数量 | 含义                   |
| ----------------- | ---- | ---------------------- |
| 数组容量 capacity | 4    | 底层数组有 4 个桶      |
| 元素数量 size     | 6    | 整个 Map 有 6 个键值对 |
| 桶 0 链表长度     | 2    | A、B 两个节点          |
| 桶 2 链表长度     | 3    | C、D、E 三个节点       |

数组容量不是元素数量。

数组有 4 个桶，不代表只能存 4 个键值对。因为每个桶可以通过链表或红黑树容纳多个节点。

## 三、Node 节点的字段

JDK 8 中 Node 的核心结构：

```
static class Node<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}
```

| 字段 | 作用               |
| ---- | ------------------ |
| hash | Key 的处理后哈希值 |
| key  | 存储 Key           |
| val  | 存储 Value         |
| next | 指向链表下一个节点 |

其中：

- `final hash`、`final key`：构造完成后不再修改。
- `volatile val`：支持 Value 更新的可见性。
- `volatile next`：支持链表节点之间连接关系的安全读取。

例如：

```
map.put("A", 100);
```

可以理解为：

```
Node
├── hash = hash("A")
├── key  = "A"
├── val  = Integer(100)
└── next = null
```

如果执行：

```
map.put("A", 200);
```

对于已有节点，主要是将：

```
node.val = Integer.valueOf(200);
```

即把 `val` 引用从指向 `Integer(100)` 替换为指向 `Integer(200)`。

并不是逐位修改原来 Integer 对象中的数值。

## 四、CAS 如何保证线程安全？

CAS 全称 Compare And Swap，即比较并交换。

**核心原理：只有当前位置的实际值等于预期值时，才允许写入新值。**

### 1. 不使用 CAS 的问题

假设 `table[3]` 为空。

线程 A：

```
map.put("A", 100);
```

线程 B：

```
map.put("B", 200);
```

两个 Key 恰好映射到同一个桶。

可能发生：

```
线程 A                     线程 B

检查 table[3] 为空
                           检查 table[3] 为空

写入 Node A
                           写入 Node B
```

最后 Node B 覆盖 Node A，导致数据丢失。

### 2. 使用 CAS

JDK 8 中的空桶插入可以简化为：

```
casTabAt(tab, i, null, newNode);
```

参数的含义：

- `tab`：数组。
- `i`：目标桶下标。
- `null`：期望当前桶为空。
- `newNode`：准备插入的新节点。

执行过程：

```
线程 A：CAS(null, Node A) → 成功

线程 B：CAS(null, Node B) → 失败
                              |
                              ↓
                      重新读取桶状态
                              |
                              ↓
                      发现桶已经非空
                              |
                              ↓
                      进入非空桶处理逻辑
```

CAS 保证比较和交换是一个原子操作。

因此两个线程不能同时成功向同一个空桶安装不同的头节点。

### 3. CAS 属于乐观锁思想

CAS 不需要预先获取互斥锁。

它假设冲突较少，先尝试修改；如果发现状态变化，就重新尝试或进入其他处理路径。

需要注意：

**CAS 并不意味着所有操作都无需加锁。**

对于链表遍历、节点追加、红黑树修改等复合操作，单次 CAS 往往不够。

## 五、synchronized 如何保证线程安全？

### 1. 为什么非空桶要加锁？

假设原链表为：

```
Node A → Node B → null
```

两个线程同时追加节点：

```
// 线程 1
map.put("C", 300);

// 线程 2
map.put("D", 400);
```

如果它们同时发现 B 是尾节点，就可能同时修改 `B.next`，造成其中一个新节点丢失。

因此需要互斥保护。

### 2. JDK 8 的桶级锁

简化源码：

```
Node<K,V> f = tabAt(tab, i);

if (f != null) {
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            // 查找已有 Key
            // 修改已有节点的 val
            // 或追加新节点
        }
    }
}
```

这里：

```
synchronized (f)
```

锁住的是当前桶头对象 `f`，不是整个 Map。

加锁后再次检查：

```
tabAt(tab, i) == f
```

是为了确认桶头没有被其他操作替换。

这样，不同普通桶的写操作通常可以并发执行。

| 操作                          | 是否需要竞争相同的桶锁 |
| ----------------------------- | ---------------------- |
| 两个线程同时修改桶 3          | 是                     |
| 一个修改桶 3，另一个修改桶 5  | 通常不需要             |
| 一个修改桶 3，另一个 get 桶 3 | get 通常不获取桶锁     |

对于红黑树桶，还有 `TreeBin` 专门负责树结构的并发管理。

## 六、链表什么时候变成红黑树？

JDK 8 中有三个重要常量：

```
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```

| 常量                 | 值   | 作用                     |
| -------------------- | ---- | ------------------------ |
| TREEIFY_THRESHOLD    | 8    | 链表树化判断阈值         |
| UNTREEIFY_THRESHOLD  | 6    | 树退化为链表时使用的阈值 |
| MIN_TREEIFY_CAPACITY | 64   | 允许树化的最小数组容量   |

### 1. 什么叫树化阈值？

树化阈值就是判断链表是否足够长、需要考虑转换为红黑树的标准。

但不是链表一达到 8 就必然立即树化。

在 JDK 8 的 `putVal()` 中，树化检查由插入过程中的 `binCount` 触发。普通链表桶中追加新节点时，实际触发时机还受到计数方式影响。

### 2. 为什么还要检查数组容量？

因为链表长，可能只是数组太小。

例如：

```
容量 16
桶 3：A → B → C → D → E → F → G → H
```

此时有两种优化方法。

**方法 A：直接树化**

将链表转换为红黑树，改善该桶的查找性能。

**方法 B：先扩容**

```
容量 16 → 容量 32
```

扩容后，原来处于同一个桶的节点可能被拆分到不同桶中。

```
扩容前：

桶 3：A → B → C → D → E → F → G → H

扩容后（示意）：

桶 3： A → C → E → G
桶 19：B → D → F → H
```

链表自然缩短，不需要转换成红黑树。

因此，JDK 8 的判断逻辑可以概括为：

```
if (链表达到树化判断条件) {
    if (数组容量 < 64) {
        // 优先扩容
    } else {
        // 树化
    }
}
```

### 3. 为什么选 64？

64 是工程设计阈值，并不是数学上严格推导出来的唯一最优值。

这个阈值体现了 JDK 的优化策略：

- 容量较小时，优先通过扩容减少冲突。
- 容量足够大时，如果链表仍然很长，再考虑树化。
- 避免为了少量冲突过早引入红黑树的额外维护成本。

**注意：哈希冲突不等于线程竞争。**

即使只有一个线程，也可能出现哈希冲突。

如果多个 Key 的哈希值非常接近甚至相同，扩容也未必能有效解决链表过长的问题。

### 4. 扩容一定导致树化吗？

不一定。

扩容和树化是两种不同的优化方式：

| 操作       | 核心目的                     |
| ---------- | ---------------------------- |
| 扩容       | 增加桶数量，尽量分散节点     |
| 树化       | 优化单个桶内节点的组织结构   |
| 退化为链表 | 当树节点减少时，降低维护成本 |

另外，扩容还可以由整个 Map 的元素数量达到扩容条件触发，并不一定要等某条链表变长。

## 七、volatile 如何保证 get 的安全性？

这是理解 `ConcurrentHashMap` 并发读取的关键。

### 1. volatile 的作用

volatile 主要提供：

- 可见性。
- 有序性保证。
- 对 volatile long/double 的单次读写原子性。

但它不能保证任意复合操作的原子性。

例如：

```
volatile int count = 0;

count++;
```

`count++` 包含读取、加一、写回。

两个线程同时执行，仍然可能丢失更新。

### 2. ConcurrentHashMap 的 volatile 字段

```
final K key;
volatile V val;
volatile Node<K,V> next;
```

其中：

`val`：使读取线程能够按照 volatile 语义观察到 Value 更新。

`next`：使读取线程能够安全地遍历已经发布的链表连接。

另外：

```
transient volatile Node<K,V>[] table;
```

table 本身是 volatile 数组引用。

但要注意：

**volatile 数组引用不意味着数组中的每个槽位自动具有 volatile 语义。**

所以 JDK 8 还使用：

```
tabAt(tab, i)
```

通过具有 volatile 读取语义的方式访问数组槽位。

### 3. 安全发布是什么意思？

假设写线程需要插入 Node C：

```
Node A → Node B → null
```

流程：

```
1. 创建 Node C
2. 初始化 C 的 hash、key、val、next
3. 通过 B.next 发布 C
4. 其他线程读取 B.next
5. 访问已经初始化好的 Node C
```

其中 `next` 使用 volatile 语义，配合节点的 final 字段及正确的发布顺序，使读线程能够安全地访问新节点。

这就是安全发布的关键思想。

## 八、为什么 get() 不需要加锁？

先看简化版代码，仅表示普通链表桶的读取流程：

```
public V get(Object key) {
    int hash = spread(key.hashCode());

    Node<K,V>[] tab = table;

    if (tab == null) {
        return null;
    }

    Node<K,V> e =
        tabAt(tab, (tab.length - 1) & hash);

    while (e != null) {
        if (e.hash == hash &&
            (e.key == key || key.equals(e.key))) {
            return e.val;
        }

        e = e.next;
    }

    return null;
}
```

这里没有 `synchronized`。

因为：

1. 通过 `tabAt()` 安全读取桶头。
2. 通过 volatile 的 `next` 遍历链表。
3. 通过 volatile 的 `val` 读取 Value。
4. 链表结构修改由写线程的 CAS、桶锁等机制协调。

因此普通桶的 get 不需要参与写线程的互斥锁竞争。

真实源码还处理红黑树桶、扩容转发节点等情况，不能把这个简化代码当成完整实现。

## 九、put() 和 get() 同时执行，线程安全吗？

线程安全，但需要区分操作是否重叠。

### 场景一：修改已经存在的 Key

初始数据：

```
map.put("A", 100);
```

两个线程：

```
// 线程 1
map.put("A", 200);

// 线程 2
map.get("A");
```

底层线程 1 相当于执行：

```
node.val = Integer.valueOf(200);
```

由于 `val` 存储的是对象引用，因此这是一次引用替换，而不是逐位修改 Integer 对象。

可能发生：

**情况 A：get 先读取**

```
线程 2 读取 val = 100
线程 1 更新 val = 200
```

返回 100。

**情况 B：put 先完成更新**

```
线程 1 更新 val = 200
线程 2 读取 val = 200
```

返回 200。

当两个操作重叠时，这两种结果都可能合法。

但如果 put 已完成，之后才开始 get，而且没有其他修改，那么 get 应当返回新值 200。

### 场景二：插入全新的 Key

```
// 线程 1
map.put("B", 300);

// 线程 2
map.get("B");
```

可能发生：

- B 尚未发布：get 返回 null。
- B 已经发布且被找到：get 返回 300。

这不是线程不安全，而是两个操作的先后关系不同。

### 场景三：扩容时读取

扩容过程中，部分桶需要迁移到新数组。

JDK 8 使用 `ForwardingNode` 表示正在迁移或已经迁移的桶。

读线程遇到转发节点时，可以继续到新数组中查找。

因此扩容时的 get 也有对应的并发协议，不是简单地对整个 Map 加锁。

### 核心区别

**线程安全不等于每一次 get 都必须读取到某个并发 put 最终写入的值。**

线程安全意味着操作符合相应的并发语义，并且不会破坏数据结构。

## 十、Java 可能读到写了一半的数值吗？

有可能。这称为 **读写撕裂（Tearing）**。

Java 内存模型允许将非 volatile 的 `long` 和 `double` 的 64 位读写拆分为两个 32 位操作。

因此从 Java 规范层面讲，可能读到新旧数值拼接后的结果。

例如：

```
class Data {
    long value;
}
```

两个线程同时读写 value 时，Java 规范不保证非 volatile long 的单次读写一定不可分割。

但现代常见的 64 位 JVM 往往会提供更强的实际实现保证，不能把规范允许的现象理解为一定发生。

### 1. 不同类型的单次读写原子性

| 类型                  | 单次读写原子性 |
| --------------------- | -------------- |
| byte、short、char     | 保证           |
| int、float            | 保证           |
| 普通 long、double     | 规范不保证     |
| volatile long、double | 保证           |
| 对象引用              | 保证           |

例如：

```
int value = 100;
value = 200;
```

另一个线程不会因为读写撕裂而读到 150。

但是没有同步时，可能仍然读到旧值 100。

### 2. 为什么 ConcurrentHashMap 不会读到半个引用？

```
volatile V val;
```

`val` 是对象引用。

Java 保证单次引用读取和赋值的原子性。

对于：

```
map.put("A", 200);
```

当已有节点的 Value 从 `Integer(100)` 变为 `Integer(200)` 时，写线程替换的是引用。

因此读线程可以读到旧引用或新引用，但不会读到一个被写坏的半截引用。

volatile 进一步提供可见性和有序性保证。

### 3. Map 安全不代表 Value 内部安全

例如：

```
class User {
    int age;
}

ConcurrentHashMap<String, User> map =
    new ConcurrentHashMap<>();
```

如果两个线程执行：

```
// 线程 1
map.get("A").age = 20;

// 线程 2
int age = map.get("A").age;
```

ConcurrentHashMap 并不能自动保证 User 内部 `age` 字段的并发可见性。

因为这里修改的是 User 对象内部的普通字段，不是 Node 的 volatile val 引用。

需要额外使用 volatile、锁、原子类或者不可变对象等方式保护。

## 十一、ConcurrentHashMap 线程安全，为什么 get + put 仍可能出错？

虽然单个 get 和 put 都是线程安全的，但组合操作不一定具有原子性。

例如：

```
Integer count = map.get("count");
map.put("count", count + 1);
```

假设 count 原来为 10。

```
线程 A                   线程 B

get → 10
                         get → 10

put → 11
                         put → 11
```

最终结果可能是 11，而不是 12。

因为：

```
读取 → 计算 → 写回
```

不是一个不可分割的操作。

正确做法之一：

```
map.merge("count", 1, Integer::sum);
```

`merge()` 可以完成针对该 Key 的原子更新。

对于高并发计数场景，也可以考虑 `LongAdder` 等工具。

## 十二、完整 put() 执行流程

JDK 8 的 `put()` 可以概括为：

```
put(key, value)
       |
       v
检查 key / value 是否为 null
       |
       v
计算 hash
       |
       v
数组是否已初始化？
       |
       |-- 否 → 通过 CAS 等机制协调初始化
       |
       v
计算目标桶下标
       |
       v
目标桶是否为空？
       |
       |-- 是 → CAS 插入节点
       |
       |-- 否
             |
             v
       是否遇到扩容转发节点？
             |
             |-- 是 → 协助扩容
             |
             |-- 否
                   |
                   v
             synchronized 锁住桶头
                   |
                   v
             校验桶头状态
                   |
                   v
             链表或红黑树中查找 Key
                   |
                   |-- 已存在 → 更新 val
                   |
                   |-- 不存在 → 插入节点
                   |
                   v
             检查是否需要树化
                   |
                   |-- 数组容量 < 64
                   |       → 优先扩容
                   |
                   |-- 数组容量 >= 64
                           → 满足条件时树化
```

最后根据操作情况更新计数，并可能触发或协助扩容。

上述流程是便于记忆的简化版本。实际源码中的计数、扩容与树化检查位置更细致，不能理解为所有步骤都会顺序执行。

## 十三、面试高频追问

### 1. ConcurrentHashMap 为什么线程安全？

因为 JDK 8 结合了 CAS、synchronized、volatile 和专门的并发结构协议。

CAS 保证空桶原子插入，synchronized 保护非空桶修改，volatile 和安全发布机制支持并发读取。

### 2. 已经使用 synchronized，为什么还要 CAS？

因为空桶插入只需要原子地安装节点，不需要为此获取桶锁。

CAS 减少不必要的锁竞争。

而非空桶中的查找、修改和插入涉及复合操作，使用 synchronized 更容易保证结构正确。

### 3. 为什么 get 不加锁？

因为它可以利用 volatile 语义和安全发布机制读取节点，同时底层结构针对并发修改作了专门设计。

但 get 不加锁不代表能够获取整个 Map 的一致性快照。

### 4. 树化阈值是多少？

JDK 8 树化判断阈值为 8，最小树化数组容量为 64。

达到链表长度条件后，不一定立即树化，还需要检查容量及对应代码路径。

### 5. 为什么先扩容，不直接树化？

因为链表过长可能是桶数量不足造成的。

扩容可能分散节点，比直接维护红黑树更合适。

### 6. 扩容和树化有什么区别？

扩容是增加数组的桶数量。

树化是改变某个桶内的数据结构。

### 7. ConcurrentHashMap 为什么不允许 null？

ConcurrentHashMap 不允许 null Key 和 null Value。

一个重要原因是：在并发读取中，`get()` 返回 null 需要明确表示没有找到映射，否则无法可靠区分“没有这个 Key”和“这个 Key 对应的 Value 为 null”。

### 8. volatile 能保证线程安全吗？

不能笼统地说可以。

volatile 提供可见性和有序性保证，但不能保证任意复合操作的原子性。

例如 volatile int 的 `count++` 仍然不是原子操作。

## 十四、面试标准回答（约 1～2 分钟）

**面试官：JDK 8 ConcurrentHashMap 是如何保证线程安全的？**

JDK 8 的 ConcurrentHashMap 底层采用 Node 数组、链表和红黑树，主要结合 CAS、synchronized 和 volatile 实现线程安全。

首先，在 put 操作中，如果目标桶为空，就通过 CAS 原子地插入节点，防止多个线程同时向空桶写入时发生覆盖。

如果桶中已经存在节点，就通常通过 synchronized 锁住桶头，保护桶内的更新和插入操作。相比 JDK 7 的 Segment 分段锁，JDK 8 的锁粒度更小。

对于 get 操作，通常不需要加锁。因为 Node 的 val 和 next 使用 volatile 修饰，数组槽位也通过具有 volatile 语义的方法访问，加上节点的安全发布以及并发扩容机制，可以保证读取过程的安全性。

因此，即使一个线程正在 put，另一个线程也可以执行 get。对于并发重叠的读写，get 可能返回修改前或修改后的合法值，但不会因为没有获取桶锁而破坏数据结构。

另外，JDK 8 在链表达到树化判断条件时，还会检查数组容量。如果容量小于 64，就优先扩容；否则在符合条件时将链表转换为红黑树，提高哈希冲突严重时的查询效率。

## 十五、最终速记

| 知识点               | 必记结论                       |
| -------------------- | ------------------------------ |
| JDK 7                | Segment + ReentrantLock 分段锁 |
| JDK 8                | Node 数组 + 链表 + 红黑树      |
| CAS                  | 空桶原子插入                   |
| synchronized         | 保护非空桶内的复合修改         |
| volatile             | 可见性、有序性，支持安全发布   |
| val                  | Node 的 Value 引用             |
| next                 | 指向链表下一个节点             |
| capacity             | 底层数组的桶数量               |
| size                 | Map 的键值对数量               |
| TREEIFY_THRESHOLD    | 8                              |
| UNTREEIFY_THRESHOLD  | 6                              |
| MIN_TREEIFY_CAPACITY | 64                             |
| 容量小但链表长       | 优先扩容                       |
| 容量足够且链表长     | 考虑树化                       |
| get                  | 通常不获取桶锁                 |
| put + get            | 允许并发执行                   |
| volatile count++     | 不是原子操作                   |
| 普通 long/double     | 规范允许读写撕裂               |
| 对象引用赋值         | 单次操作具有原子性             |
| get + put 组合       | 不一定具有原子性               |

**最后记住三个核心逻辑：**

1. **为什么写入安全？** CAS 解决空桶竞争，synchronized 解决非空桶的复合修改竞争。
2. **为什么读取安全？** volatile、安全发布以及专门的并发数据结构保证读取能够正确进行，而不是依赖读线程获取桶锁。
3. **为什么先扩容再树化？** 扩容优先尝试减少哈希冲突；只有容量足够大、桶中节点仍然很多时，才考虑红黑树。