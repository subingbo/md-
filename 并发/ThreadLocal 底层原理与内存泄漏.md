# ThreadLocal 底层原理与内存泄漏

## 1. ThreadLocal 是什么？

ThreadLocal 的作用：

> 为每个线程提供一份独立的变量副本，实现线程隔离。

例如：

```java
ThreadLocal<User> userLocal = new ThreadLocal<>();
```

线程 A：

```java
userLocal.set(userA);
```

线程 B：

```java
userLocal.set(userB);
```

结果：

```
线程A获取 userA

线程B获取 userB
```

互不影响。

------

# 2. 最重要的概念：ThreadLocal 不保存数据

很多人误解：

```
ThreadLocal
    |
    ↓
value
```

实际上不是。

ThreadLocal 只是一个 **key（钥匙）**。

真正保存数据的是：

```
Thread 对象内部的 ThreadLocalMap
```

结构：

```
Thread
 |
 | threadLocals
 ↓
ThreadLocalMap
 |
 ↓
Entry
 |
 +----------------+
 |                |
key              value
 |                |
 ↓                ↓
ThreadLocal       数据对象
```

------

# 3. threadLocals 是什么？

Thread 类内部：

```java
class Thread {

    ThreadLocal.ThreadLocalMap threadLocals;

}
```

也就是说：

每个线程都有一个自己的 ThreadLocalMap。

关系：

```
Thread
 |
threadLocals
 |
ThreadLocalMap
```

所以：

- 一个线程有自己的 ThreadLocalMap
- 不同线程之间 ThreadLocalMap 不一样

这就是线程隔离的原因。

------

# 4. ThreadLocalMap 和 Entry

ThreadLocalMap 类似 HashMap。

普通 HashMap：

```
HashMap

Entry

key ---> value
```

ThreadLocalMap：

```
ThreadLocalMap

Entry

key ---> value

key = ThreadLocal对象
value = 保存的数据
```

源码：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {

    Object value;

}
```

说明：

Entry 有两个部分：

```
Entry

key
 |
 ↓
ThreadLocal对象


value
 |
 ↓
业务对象
```

其中：

- key 是弱引用
- value 是强引用

------

# 5. set() 流程

代码：

```java
threadLocal.set(user);
```

流程：

```
1. 获取当前线程

Thread.currentThread()


2. 获取线程中的 ThreadLocalMap

Thread
 |
threadLocals
 |
ThreadLocalMap


3. 使用当前 ThreadLocal 作为 key

map.set(threadLocal, user)


4. 创建 Entry

Entry

key(ThreadLocal)
        |
        ↓
value(User)
```

最终：

```
Thread
 |
threadLocals
 |
ThreadLocalMap
 |
Entry

key --------> ThreadLocal

value ------> User
```

------

# 6. get() 流程

代码：

```java
threadLocal.get();
```

流程：

```
1. 获取当前线程

Thread.currentThread()


2. 获取当前线程的 ThreadLocalMap


3. 使用当前 ThreadLocal 查找 Entry


4. 返回 Entry.value
```

本质：

```
(Thread, ThreadLocal)
        |
        ↓
      value
```

也就是说：

决定数据的是：

```
线程 + ThreadLocal对象
```

------

# 7. 一个 ThreadLocal 是否只能绑定一个线程？

错误理解：

```
一个 ThreadLocal = 一个线程
```

正确：

```
一个 ThreadLocal 可以被多个线程使用
```

例如：

```java
static ThreadLocal<String> local = new ThreadLocal<>();
```

线程 A：

```
ThreadA
 |
ThreadLocalMap

local ---> "A"
```

线程 B：

```
ThreadB
 |
ThreadLocalMap

local ---> "B"
```

注意：

两个线程使用的是同一个 ThreadLocal 对象。

但是：

保存数据的位置不同。

------

# 8. 为什么 ThreadLocal 通常定义为 static？

例如：

```java
private static final ThreadLocal<User> USER =
        new ThreadLocal<>();
```

原因：

让所有地方访问同一个 ThreadLocal 对象。

例如：

Controller：

```java
USER.get();
```

Service：

```java
USER.get();
```

拿到的是当前线程自己的数据。

static 不代表数据共享。

共享的是：

```
ThreadLocal对象(key)
```

不是：

```
value
```

------

# 9. ThreadLocal 为什么使用弱引用？

Entry：

```
Entry

key(弱引用)
      |
      ↓
ThreadLocal

value(强引用)
      |
      ↓
User
```

原因：

防止 ThreadLocal 对象本身无法回收。

------

## 如果 key 是强引用

引用链：

```
Thread
 |
threadLocals
 |
ThreadLocalMap
 |
Entry
 |
key
 |
ThreadLocal
```

即使业务代码：

```java
local = null;
```

ThreadLocal 仍然被 ThreadLocalMap 引用。

导致：

ThreadLocal 永远无法回收。

------

## 弱引用

GC 时：

发现：

```
ThreadLocal
↑
只有弱引用
```

于是：

```
ThreadLocal对象被回收
```

变成：

```
Entry

key = null

value = User
```

------

# 10. 为什么弱引用不能直接解决内存泄漏？

因为真正大的通常是：

```
value
```

不是：

```
ThreadLocal
```

GC 后：

```
Thread
 |
threadLocals
 |
ThreadLocalMap
 |
Entry

key = null

value
 |
 ↓
大对象
```

value 仍然存在。

所以：

弱引用只是：

> 让 JVM 知道这个 Entry 对应的 ThreadLocal 已经失效。

真正释放 value：

需要：

```java
threadLocal.remove();
```

------

# 11. remove() 做了什么？

代码：

```java
threadLocal.remove();
```

删除：

```
ThreadLocalMap中的Entry
```

变成：

```
Thread
 |
threadLocals
 |
ThreadLocalMap
```

没有：

```
Entry
 |
value
```

于是：

value 没有引用。

GC 回收。

------

# 12. 为什么线程池容易产生内存泄漏？

普通线程：

```
Thread执行结束

↓

Thread销毁

↓

ThreadLocalMap销毁

↓

value释放
```

线程池：

```
线程一直存在

↓

ThreadLocalMap一直存在

↓

Entry一直存在

↓

value一直存在
```

如果忘记：

```java
remove()
```

可能导致：

```
内存持续增长

最终 OOM
```

------

# 13. 推荐写法

必须使用：

```java
try {

    threadLocal.set(value);

    //业务代码

} finally {

    threadLocal.remove();

}
```

保证线程池场景不会泄漏。

------

# 14. 面试总结

ThreadLocal 通过在线程 Thread 内部维护 ThreadLocalMap，实现线程隔离。

调用 set 时，会获取当前线程，把 ThreadLocal 作为 key，把业务数据作为 value 存入 ThreadLocalMap。

调用 get 时，会根据当前线程找到 ThreadLocalMap，再使用 ThreadLocal 找到对应 Entry，返回 value。

ThreadLocalMap 的 Entry 中 key 使用弱引用，是为了避免 ThreadLocal 对象本身无法回收，让 JVM 可以发现失效 Entry。

但是 value 是强引用，因此弱引用不能完全解决内存泄漏。在线程池等长生命周期线程中，如果不调用 remove，value 可能长期无法释放，所以实际开发中应该使用 try-finally 主动 remove。