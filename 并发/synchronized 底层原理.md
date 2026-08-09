synchronized 底层原理笔记

------

# synchronized 底层原理

## 1. synchronized 是什么？

`synchronized` 是 Java 提供的关键字，用于实现线程同步。

它可以修饰：

- 实例方法
- 静态方法
- 代码块

作用：

> 保证同一时间只有一个线程可以执行临界区代码。

------

# 2. synchronized 锁的是什么？

重点：

**synchronized 锁的是对象，而不是代码。**

例如：

```java
Object lock = new Object();

synchronized(lock){
    System.out.println("hello");
}
```

这里：

```text
锁对象 = lock
```

JVM 会使用这个对象作为锁。

------

# 3. Java 对象结构

每个 Java 对象在 JVM 中都有对象头：

```
Object

+----------------+
| 对象头          |
|                |
| Mark Word      |
| Class Pointer  |
+----------------+
| 实例数据        |
+----------------+
```

其中：

## Mark Word

保存对象运行时相关信息：

- hashCode
- GC 年龄
- 锁状态
- 锁指针等

`synchronized` 的锁信息主要和 Mark Word 有关。

------

# 4. synchronized 底层 Monitor

JVM 中每个对象都可以关联一个 Monitor。

Monitor 可以理解为：

> JVM 管理对象锁的一个监视器。

Monitor 主要保存：

```
Monitor

+----------------+
| Owner          | 当前持有锁的线程
| Count          | 重入次数
| WaitSet        | 等待线程
+----------------+
```

------

例如：

线程 A 获取锁：

```java
synchronized(lock){

}
```

Monitor：

```
Owner = Thread A
Count = 1
```

表示：

- 当前锁属于线程 A
- A 获取了一次

------

# 5. synchronized 加锁过程

## 5.1 无锁状态

对象刚创建：

```
lock对象

Mark Word

无锁状态
```

------

## 5.2 CAS 尝试获取锁

线程执行：

```java
synchronized(lock)
```

JVM 会尝试通过 CAS 修改 Mark Word。

过程：

```
无锁

↓

CAS竞争

↓

获取锁
```

成功：

线程获得锁。

失败：

说明存在竞争。

------

# 6. 锁升级过程

synchronized 锁一般经历：

```
无锁

↓

偏向锁

↓

轻量级锁

↓

重量级锁
```

（不同 JDK 版本实现有所变化）

------

## 6.1 轻量级锁

Mark Word：

保存：

```
指向线程栈中 Lock Record 的指针
```

结构：

```
对象 Mark Word

        ↓

线程栈

Lock Record
```

适用于：

线程交替竞争。

------

## 6.2 重量级锁

竞争严重时：

Mark Word：

保存：

```
Monitor 地址
```

结构：

```
对象

Mark Word

    ↓

Monitor

+-------------+
| Owner       |
| Count       |
| WaitSet     |
+-------------+
```

------

# 7. synchronized 为什么是可重入锁？

## 什么是可重入？

同一个线程可以多次获取同一把锁。

不会因为自己已经持有锁而阻塞。

------

例子：

```java
public synchronized void method1(){

    method2();

}


public synchronized void method2(){

}
```

执行：

线程 A：

进入 method1：

第一次获取锁：

```
Owner = A
Count = 1
```

------

调用 method2：

再次获取同一把锁：

因为：

```
Owner = A
```

发现是同一个线程。

不会阻塞。

只增加次数：

```
Owner = A
Count = 2
```

------

method2 执行结束：

释放一次：

```
Count:

2

↓

1
```

------

method1 执行结束：

再次释放：

```
Count:

1

↓

0
```

当：

```
Count = 0
```

才真正释放锁。

------

# 8. 为什么需要可重入？

如果 synchronized 不支持重入：

```
线程A

获取method1锁

Count=1


进入method2

再次请求同一把锁


发现锁已经被A持有

等待释放


但是A必须执行完method2才能释放
```

结果：

自己等待自己。

产生死锁。

------

# 9. 面试标准回答

> synchronized 是 JVM 基于 Monitor 实现的锁机制。每个 Java 对象都有对象头，其中 Mark Word 会记录锁状态信息。当线程进入 synchronized 代码块时，会尝试获取对象对应的 Monitor。Monitor 中维护当前持有锁的线程以及重入次数，因此 synchronized 是可重入锁。同一个线程多次获取同一把锁时，只增加重入次数，直到次数归零才释放锁。

------

# 10. 面试关键词记忆

```
synchronized

锁对象

↓

对象头

↓

Mark Word

↓

锁状态

↓

Monitor

↓

Owner 当前线程

Count 重入次数

WaitSet 等待队列
```

一句话：

> synchronized 锁的是对象，对象头 Mark Word 保存锁状态，重量级锁关联 Monitor，Monitor 保存持有线程和重入次数，因此 synchronized 支持可重入。