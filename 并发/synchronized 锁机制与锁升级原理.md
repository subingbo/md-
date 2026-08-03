# synchronized 锁机制与锁升级原理

## 1. synchronized 简介

`synchronized` 是 Java 提供的关键字，用于实现线程同步。

作用：

- 保证同一时刻只有一个线程进入临界区
- 保证线程之间的可见性
- 保证操作的原子性

例如：

```java
public synchronized void test() {

}
```

等价于：

```java
synchronized(this) {

}
```

------

# 2. synchronized 是可重入锁

## 什么是可重入？

可重入表示：

> 同一个线程已经获得一把锁后，可以再次获得同一把锁。

例如：

```java
public synchronized void methodA() {

    methodB();

}


public synchronized void methodB() {

}
```

执行：

```
线程A

进入methodA()

↓

获取对象锁

↓

调用methodB()

↓

再次请求同一把锁

↓

允许进入
```

不会死锁。

------

## 为什么需要记录重入次数？

因为 JVM 必须知道：

> 当前线程到底进入了多少次 synchronized。

例如：

```java
synchronized(obj){

    synchronized(obj){

    }

}
```

进入过程：

第一次：

```
Owner = ThreadA
count = 1
```

第二次：

```
Owner = ThreadA
count = 2
```

退出：

第一次：

```
count--

2 → 1
```

锁不能释放。

第二次：

```
count--

1 → 0
```

真正释放锁。

------

如果没有计数：

```
进入第一层

↓

进入第二层

↓

退出第二层

↓

直接释放锁
```

那么其他线程就可能进入。

但是第一层代码还没有执行完。

所以必须维护重入次数。

------

# 3. Java对象结构

一个 Java 对象：

```
Object

┌─────────────────┐
│ 对象头           │
├─────────────────┤
│ 实例数据         │
├─────────────────┤
│ 对齐填充         │
└─────────────────┘
```

其中：

对象头包含：

```
Mark Word
+
类型指针
```

------

# 4. Mark Word

Mark Word 是对象头中的一部分。

它保存：

- HashCode
- GC年龄
- 锁状态
- 锁相关信息

不同状态保存不同内容。

例如：

## 无锁状态

```
Mark Word

┌──────────────┐
│ 无锁信息     │
└──────────────┘
```

------

## 轻量级锁状态

保存：

```
指向线程栈 Lock Record 的指针
```

------

## 重量级锁状态

保存：

```
指向 ObjectMonitor 的指针
```

------

# 5. 什么是无锁状态？

注意：

无锁 ≠ 没有锁机制

而是：

> 对象当前没有线程持有锁。

例如：

```java
Object obj = new Object();
```

刚创建：

```
obj

状态：

无锁
```

表示：

```
Owner = null
```

------

类似：

```
会议室

存在

但是没人使用
```

------

# 6. CAS 是什么？

CAS：

Compare And Swap

比较并交换。

形式：

```
CAS(内存值, 期望值, 新值)
```

例如：

当前：

```
value = 0
```

执行：

```
CAS(0,1)
```

意思：

如果 value 还是0：

```
修改为1
```

否则失败。

------

CAS特点：

- 原子操作
- 不需要线程阻塞
- 依靠CPU指令实现

------

# 7. CAS 和锁的关系

很多人误解：

> CAS 是无锁，所以没有锁。

实际上：

CAS 是一种操作。

它可以用于：

- 无锁算法
- 实现锁

例如：

`synchronized` 的轻量级锁：

使用 CAS 修改对象头。

目的：

> 抢占锁。

------

# 8. synchronized 锁升级过程

现代 JDK：

```
无锁

↓

轻量级锁

↓

重量级锁
```

（历史版本还有偏向锁）

------

# 9. 无锁 → 轻量级锁

对象：

```
Object obj

Mark Word:

无锁
```

线程A：

```java
synchronized(obj){

}
```

JVM：

尝试 CAS 修改 Mark Word。

过程：

```
无锁对象

↓

线程A执行CAS

↓

CAS成功

↓

轻量级锁
```

------

此时：

```
Mark Word

记录线程A相关锁信息
```

线程A获得锁。

------

# 10. 为什么轻量级锁使用CAS？

因为可能多个线程同时竞争。

例如：

线程A：

```
CAS:

无锁 → A
```

线程B：

```
CAS:

无锁 → B
```

CAS保证：

只有一个成功。

结果：

```
线程A

CAS成功

获得锁


线程B

CAS失败
```

------

# 11. 线程B失败后怎么办？

线程B：

```
CAS失败
```

不会立即阻塞。

而是：

```
自旋
```

------

## 什么是自旋？

不停尝试：

```
锁释放了吗？

锁释放了吗？

锁释放了吗？
```

例如：

```text
while(true){

    try CAS

}
```

------

# 12. 为什么不直接阻塞？

因为阻塞有成本。

阻塞：

```
用户态

↓

内核态

↓

线程挂起
```

恢复：

```
线程唤醒

↓

重新调度CPU
```

成本较高。

如果：

```
线程A马上释放锁
```

那么：

```
自旋等待
```

反而更快。

------

# 13. 什么时候升级重量级锁？

不是：

```
一个线程 = 轻量级

两个线程 = 重量级
```

错误。

真正判断：

> 竞争是否激烈，自旋是否值得。

------

例如：

## 情况1

```
线程A持锁

线程B竞争

↓

CAS失败

↓

自旋

↓

A马上释放

↓

B成功
```

继续：

```
轻量级锁
```

------

## 情况2

```
线程A持锁

线程B竞争

↓

CAS失败

↓

一直自旋

↓

还拿不到
```

JVM认为：

```
继续自旋浪费CPU
```

于是：

```
轻量级锁

↓

重量级锁
```

------

# 14. 什么是 Monitor？

重量级锁底层使用：

```
ObjectMonitor
```

它是 JVM 创建的数据结构。

包含：

```
ObjectMonitor

├── Owner
│
├── EntryList
│
├── WaitSet
│
└── 重入次数
```

------

# 15. 为什么对象头保存 Monitor 地址？

因为对象头空间有限。

Mark Word：

只有固定大小。

无法存：

```
Owner

等待队列

等待集合

计数
```

所以：

重量级锁时：

对象：

```
Mark Word

保存：

Monitor地址
```

例如：

```
Object

Mark Word

0x1000
```

然后：

```
0x1000

↓

ObjectMonitor
```

------

结构：

```
Object

↓

Mark Word

↓

Monitor地址

↓

ObjectMonitor

    Owner

    EntryList

    WaitSet
```

------

# 16. EntryList 和 WaitSet

这是最容易混淆的地方。

## EntryList

表示：

> 等待获取锁的线程。

例如：

线程A：

```
持有锁
```

线程B：

```
想获取锁
```

那么：

```
EntryList

B
```

------

## WaitSet

表示：

> 调用了 wait() 主动等待条件的线程。

例如：

```java
synchronized(obj){

    obj.wait();

}
```

流程：

```
线程A

获得锁

↓

wait()

↓

释放锁

↓

进入WaitSet
```

------

# 17. notify之后发生什么？

线程：

```
WaitSet
```

调用：

```java
notify()
```

不会直接执行。

而是：

```
WaitSet

↓

EntryList

↓

重新竞争锁

↓

获得锁

↓

继续执行
```

------

# 18. 完整锁升级流程

```
Object创建


↓

无锁

Mark Word:

无锁状态


↓

线程A进入synchronized


↓

CAS修改Mark Word


↓

轻量级锁


↓

线程B竞争


↓

CAS失败


↓

自旋


↓

是否值得继续？


       是
       |
       ↓

继续轻量级锁


       否
       |
       ↓


升级重量级锁


↓

创建ObjectMonitor


↓

Mark Word保存Monitor地址


↓

线程进入EntryList


↓

park阻塞


↓

unlock


↓

unpark线程


↓

重新竞争
```

------

# 19. synchronized 和 ReentrantLock 对比

| synchronized   | ReentrantLock         |
| -------------- | --------------------- |
| JVM实现        | Java代码实现          |
| Monitor        | AQS                   |
| 自动释放锁     | 手动unlock            |
| 可重入         | 可重入                |
| 不支持超时     | tryLock支持           |
| 不支持中断等待 | lockInterruptibly支持 |
| 一个wait队列   | 多个Condition         |

------

# 20. 核心总结

记住下面几个点：

## 1.

无锁：

```
对象没人持有锁
```

不是没有锁机制。

------

## 2.

CAS：

```
不是锁

是获取锁的一种原子操作
```

------

## 3.

轻量级锁：

```
CAS + 自旋
```

适合：

```
竞争短
```

------

## 4.

重量级锁：

```
Monitor + park/unpark
```

适合：

```
竞争激烈
```

------

## 5.

对象头：

```
保存锁状态信息
```

重量级时：

```
保存Monitor地址
```

------

## 6.

Monitor：

```
真正管理重量级锁的数据结构
```

包含：

```
Owner

EntryList

WaitSet

重入次数
```

------

这套理解掌握后，`synchronized`、`ReentrantLock`、AQS、Condition 基本就串起来了。