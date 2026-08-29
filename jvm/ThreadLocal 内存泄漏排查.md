# ThreadLocal 内存泄漏排查

## 1. 核心结论

排查 ThreadLocal 内存泄漏，本质上要回答两个问题：

1. **谁占用了大量内存？**
2. **为什么这些对象一直没有被 GC？**

对应工具：

```text
Heap Dump
   ↓
Dominator Tree
   ↓
Retained Heap
   ↓
找到占内存的大户
   ↓
Path to GC Roots
   ↓
找到对象为什么一直存活
```

------

# 2. Heap Dump 是什么？

Heap Dump 可以理解成：

> **某一时刻 JVM 堆内存中对象和对象引用关系的快照。**

例如 JVM 中存在：

```text
Thread
 └── ThreadLocalMap
      └── Entry
           └── LargeObject
```

执行：

```bash
jcmd <pid> GC.heap_dump heap.hprof
```

就可以生成：

```text
heap.hprof
```

然后使用 Eclipse MAT、JProfiler、YourKit 等工具分析。

------

# 3. 如何生成 Heap Dump？

## 方法一：jcmd

推荐：

```bash
jcmd <pid> GC.heap_dump heap.hprof
```

例如：

```bash
jcmd 12345 GC.heap_dump heap.hprof
```

------

## 方法二：jmap

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>
```

例如：

```bash
jmap -dump:live,format=b,file=heap.hprof 12345
```

其中：

```text
-dump       生成 Heap Dump
-live       只保留存活对象
-format=b   二进制格式
```

------

# 4. 排查内存异常的完整流程

不要拿 Heap Dump 后一个对象一个对象地找。

正确思路：

```text
发现 JVM 内存异常
       ↓
jstat 观察 GC
       ↓
判断 Full GC 后内存是否下降
       ↓
生成 Heap Dump
       ↓
MAT 分析
       ↓
Dominator Tree
       ↓
Retained Heap 排序
       ↓
找到内存大户
       ↓
Path to GC Roots
       ↓
找到为什么对象一直存活
       ↓
回到代码定位问题
```

------

# 5. 第一步：观察 GC

可以使用：

```bash
jstat -gcutil <pid> 1000
```

例如：

```bash
jstat -gcutil 12345 1000
```

重点观察：

```text
O       老年代使用率
YGC     Young GC 次数
FGC     Full GC 次数
```

如果发现：

```text
Old Gen：

70%
75%
80%
85%
90%
```

并且 Full GC 后：

```text
Full GC
   ↓
老年代仍然很高
```

说明存在大量对象无法回收的可能。

------

# 6. 第二步：快速查看哪些对象占内存

可以使用：

```bash
jcmd <pid> GC.class_histogram
```

例如：

```bash
jcmd 12345 GC.class_histogram
```

可能看到：

```text
num     #instances       #bytes       Class
------------------------------------------------
1       10000            1200000000   [B
2       500000           100000000    java.lang.String
3       100000            80000000    java.util.HashMap
```

其中：

```text
[B
```

代表：

```text
byte[]
```

这个命令可以快速回答：

> **现在 JVM 里什么类型的对象最多、占内存最多？**

------

# 7. 第三步：生成 Heap Dump

```bash
jcmd 12345 GC.heap_dump heap.hprof
```

然后使用：

```text
Eclipse MAT
```

打开：

```text
heap.hprof
```

------

# 8. MAT 中首先看什么？

## 8.1 Leak Suspects

MAT 提供：

```text
Leak Suspects Report
```

它会尝试自动寻找明显的内存泄漏嫌疑。

例如可能发现：

```text
Thread
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
LargeObject
```

并提示某个线程保留了大量内存。

------

# 9. Histogram

Histogram 主要回答：

> **哪些类型的对象占内存最多？**

例如：

```text
Class              Objects       Shallow Heap
------------------------------------------------
byte[]             12000         1.2 GB
String             500000        100 MB
HashMap            100000         80 MB
```

注意：

> Histogram 适合找“对象类型”，但不能直接告诉你“为什么对象没有被 GC”。

------

# 10. Shallow Heap 和 Retained Heap

这是 MAT 中非常重要的两个概念。

## 10.1 Shallow Heap

表示：

> **对象自身占用的内存。**

例如：

```text
ThreadLocalMap
Shallow Heap = 1 KB
```

它自己可能很小。

------

## 10.2 Retained Heap

表示：

> **如果这个对象被回收，能够连带释放的大致内存。**

例如：

```text
ThreadLocalMap
 ├── Entry
 │    └── byte[500MB]
 │
 ├── Entry
 │    └── Object[100MB]
 │
 └── Entry
      └── String[50MB]
```

那么：

```text
Shallow Heap：
ThreadLocalMap ≈ 很小

Retained Heap：
ThreadLocalMap ≈ 650MB
```

因此排查内存泄漏时：

> **Retained Heap 通常比 Shallow Heap 更有价值。**

------

# 11. Dominator Tree

## 11.1 Dominator 是什么？

简单理解：

> **如果 A 不存在了，B 也会因为失去引用而可以被回收，那么 A 可以认为“支配”B。**

例如：

```text
A
└── B
    └── C
```

如果 A 被回收：

```text
A 消失
 ↓
B 没有引用
 ↓
B 可以被回收
 ↓
C 也可以被回收
```

所以 A 控制了 B、C。

------

# 12. 为什么 Dominator Tree 很适合排查内存泄漏？

因为 JVM 可能有：

```text
几百万个对象
```

不可能一个个检查。

Dominator Tree 可以帮助我们快速发现：

> **谁控制了大量内存？**

例如：

```text
Dominator Tree

Thread-1                    1.8 GB
Thread-2                    200 MB
Spring Cache                100 MB
HashMap                      80 MB
```

发现：

```text
Thread-1
Retained Heap = 1.8 GB
```

就应该重点检查。

继续展开：

```text
Thread-1
└── ThreadLocalMap          1.8 GB
     └── Entry              1.8 GB
          └── LargeObject   1.8 GB
```

于是就找到了：

```text
Thread
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
LargeObject
```

------

# 13. Dominator Tree 和引用链不是一回事

这是最容易混淆的地方。

## Dominator Tree

回答：

> **谁控制了大量内存？**

```text
ThreadLocalMap
Retained Heap = 1.8 GB
```

------

## Path to GC Roots

回答：

> **为什么这个对象还活着？**

例如：

```text
GC Root
 ↓
Thread
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
LargeObject
```

所以：

```text
Dominator Tree
    ↓
找内存大户
```

而：

```text
Path to GC Roots
    ↓
找为什么无法 GC
```

------

# 14. GC Root 是什么？

GC Root 可以理解成：

> **JVM 认为必须从这些对象开始向下查找引用关系，只要对象还能从 GC Root 到达，就不能被 GC。**

常见 GC Root 包括：

```text
线程栈中的局部变量
活跃线程
静态变量
JNI 引用
某些 JVM 内部引用
```

------

# 15. ThreadLocal 为什么会产生泄漏？

ThreadLocalMap 中的 Entry 本质上类似：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {

    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

因此：

```text
Entry
├── key   → 弱引用
└── value → 强引用
```

------

# 16. ThreadLocal 泄漏的典型引用链

假设：

```java
ThreadLocal<byte[]> local = new ThreadLocal<>();

local.set(new byte[200 * 1024 * 1024]);
```

如果 ThreadLocal 对象被 GC：

```text
ThreadLocal
   ↓
被 GC
```

可能出现：

```text
Entry
├── key   = null
└── value = byte[200MB]
```

但是线程还活着：

```text
Thread
 ↓
threadLocals
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
value
 ↓
byte[200MB]
```

因此这个 200MB 对象仍然无法被 GC。

------

# 17. 为什么线程池特别容易出现这个问题？

普通线程：

```text
Thread
 ↓
执行任务
 ↓
任务结束
 ↓
Thread 结束
 ↓
ThreadLocalMap 最终也可以回收
```

线程池：

```text
ThreadPool
 ↓
Worker Thread
 ↓
长期存活
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
value
```

线程池中的 Worker 可能长期存在。

所以：

> **ThreadLocal + 线程池 + 没有 remove() 是重点排查场景。**

------

# 18. MAT 中怎么确认 ThreadLocal 泄漏？

重点寻找：

```text
Thread
 ↓
threadLocals
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
value
```

尤其关注：

```text
Entry
├── referent = null
└── value = 大对象
```

这里：

```text
referent
```

就是弱引用指向的 ThreadLocal。

如果：

```text
referent = null
```

说明 ThreadLocal Key 已经被 GC。

但是：

```text
value
```

还活着。

如果这个 value 又很大：

```text
byte[]
Map
List
业务对象
缓存对象
```

就需要重点检查。

------

# 19. 实际排查不要一个一个对象找

错误思路：

```text
找到 byte[]
 ↓
检查
 ↓
再找另一个 byte[]
 ↓
再检查
 ↓
...
```

正确思路：

```text
Heap Dump
 ↓
Dominator Tree
 ↓
按 Retained Heap 找最大的几个对象
 ↓
发现 Thread / ThreadLocalMap
 ↓
展开
 ↓
找到大量 value
 ↓
Path to GC Roots
 ↓
确认引用链
```

也就是说：

> **先从“内存规模”缩小范围，再从“引用关系”确认原因。**

------

# 20. 一套完整排查案例

假设 JVM：

```text
Heap = 4GB
```

运行一段时间：

```text
Old Gen = 3.5GB
```

Full GC：

```text
GC 前：3.5GB
GC 后：3.3GB
```

说明大量对象仍然存活。

------

## 第一步：生成 Heap Dump

```bash
jcmd 12345 GC.heap_dump heap.hprof
```

------

## 第二步：MAT → Dominator Tree

发现：

```text
Thread-42
Retained Heap = 1.8GB
```

------

## 第三步：展开

```text
Thread-42
└── ThreadLocalMap
     └── Entry
          └── LargeObject
```

发现：

```text
ThreadLocalMap
Retained Heap = 1.8GB
```

------

## 第四步：Path to GC Roots

看到：

```text
GC Root
 ↓
Thread-42
 ↓
threadLocals
 ↓
ThreadLocalMap
 ↓
Entry
 ↓
LargeObject
```

------

## 第五步：检查 Entry

发现：

```text
Entry
├── referent = null
└── value = LargeObject
```

此时高度怀疑 ThreadLocal 使用不当。

------

## 第六步：回到代码

搜索：

```text
ThreadLocal
```

重点检查：

```java
threadLocal.set(...)
```

有没有对应：

```java
finally {
    threadLocal.remove();
}
```

推荐：

```java
try {
    threadLocal.set(context);

    // 业务代码

} finally {
    threadLocal.remove();
}
```

------

# 21. 常用命令总结

## 查看 Java 进程

```bash
jps -l
```

------

## 查看 GC

```bash
jstat -gcutil <pid> 1000
```

------

## 查看对象直方图

```bash
jcmd <pid> GC.class_histogram
```

------

## 生成 Heap Dump

```bash
jcmd <pid> GC.heap_dump heap.hprof
```

或者：

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>
```

------

# 22. 最终记忆框架

```text
                JVM 内存异常
                     │
                     ↓
             jstat -gcutil
                     │
                     ↓
          Full GC 后仍然很高？
                     │
                     ↓
                Heap Dump
                     │
                     ↓
                 MAT
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
     Histogram            Dominator Tree
          ↓                     ↓
   哪类对象最多？          谁控制内存最多？
                                │
                                ↓
                         Retained Heap
                                │
                                ↓
                         找到内存大户
                                │
                                ↓
                       Path to GC Roots
                                │
                                ↓
                       为什么无法 GC？
                                │
                                ↓
                    ThreadLocal / Cache /
                    Static / Collection...
                                │
                                ↓
                         回到代码定位
```

------

# 23. 面试回答模板

如果面试官问：

> 怎么排查 ThreadLocal 内存泄漏？

可以回答：

> 我会先通过 `jstat -gcutil` 观察 GC 情况，如果发现 Full GC 后老年代仍然维持高位，就生成 Heap Dump。
>
> 然后使用 MAT 分析，先通过 Dominator Tree 按 Retained Heap 找出占用内存比较大的对象或对象树，而不是一个对象一个对象地排查。
>
> 如果发现 Thread 或 ThreadLocalMap 占用了大量 Retained Heap，就进一步查看其内部的 Entry 和 value。
>
> 然后通过 Path to GC Roots 查看为什么这些对象仍然存活。如果发现 `Thread → threadLocals → ThreadLocalMap → Entry → value` 这条引用链，并且 Entry 的 key，也就是 referent，已经是 null，而 value 仍然存在，同时线程又是线程池中的长期存活线程，那么就高度怀疑是 ThreadLocal 使用不当导致的内存泄漏。
>
> 最后回到代码检查 `ThreadLocal.set()`，确认是否在 `finally` 中调用 `remove()`。

------

# 24. 一句话总结

```text
Heap Dump
    = 给 JVM 堆拍快照

Dominator Tree
    = 找谁控制了最多内存

Retained Heap
    = 这个对象死掉后大概能释放多少内存

Path to GC Roots
    = 找为什么这个对象还活着

ThreadLocal 泄漏
    = Thread 长期存活
      ↓
      ThreadLocalMap
      ↓
      Entry
      ↓
      value
      ↓
      大对象长期存活
```

**排查的核心不是“找大对象”，而是“先找 Retained Heap 大的对象，再沿 GC Root 引用链找到它为什么一直活着”。**