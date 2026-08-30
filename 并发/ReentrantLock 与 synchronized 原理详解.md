# ReentrantLock 与 synchronized 原理详解

## 一、synchronized 和 ReentrantLock 是什么

两者都是 Java 中用于实现**线程互斥**的锁。

核心目的：

```text
多个线程访问共享资源
        ↓
可能发生数据竞争
        ↓
使用锁保证同一时间只有一个线程进入临界区
```

例如：

```java
synchronized (obj) {
    count++;
}
```

或者：

```java
lock.lock();
try {
    count++;
} finally {
    lock.unlock();
}
```

------

# 二、synchronized 原理

`synchronized` 是 Java 关键字，底层依赖 JVM 的 **Monitor（监视器）机制**。

例如：

```java
synchronized (obj) {
    count++;
}
```

可以粗略理解为：

```text
synchronized
     ↓
Monitor
     ↓
尝试获取锁
     ↓
成功 → 执行临界区
     ↓
释放锁
```

如果线程 A 已经持有锁：

```text
A → 持有锁

B → 尝试获取
C → 尝试获取
```

B、C 无法同时进入临界区，需要等待。

------

## 2. synchronized 的字节码

`synchronized` 代码块编译后涉及：

```text
monitorenter
    ↓
进入同步区域
    ↓
monitorexit
```

因此可以理解为：

```text
monitorenter
    ↓
获取 Monitor
    ↓
执行代码
    ↓
monitorexit
    ↓
释放 Monitor
```

JVM 会保证异常情况下也能够释放 Monitor。

------

# 三、synchronized 是可重入锁

`synchronized` 支持**可重入**。

例如：

```java
public synchronized void method1() {
    method2();
}

public synchronized void method2() {
}
```

线程 A：

```text
进入 method1
    ↓
获得锁
    ↓
调用 method2
    ↓
再次获得同一把锁
```

不会死锁。

可以粗略理解成 Monitor 维护：

```text
owner = Thread A
holdCount = 1
```

再次进入：

```text
holdCount = 2
```

退出：

```text
holdCount = 1
```

再次退出：

```text
holdCount = 0
```

最终释放锁。

------

# 四、synchronized 是公平锁吗？

**不保证公平性。**

例如：

```text
A 持有锁

等待：
B
C
D
```

不能保证一定：

```text
B → C → D
```

依次获得锁。

更严谨的说法是：

> synchronized 不提供公平性保证。

不要简单说成“synchronized 一定是非公平锁”。

------

# 五、ReentrantLock 原理

`ReentrantLock` 是 Java 类：

```java
java.util.concurrent.locks.ReentrantLock
```

使用：

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();

try {
    count++;
} finally {
    lock.unlock();
}
```

它的核心实现是：

```text
ReentrantLock
      ↓
     AQS
      ↓
state + 等待队列
```

AQS：

```text
AbstractQueuedSynchronizer
```

可以理解成：

> Java 并发包提供的一个同步器基础框架。

------

# 六、AQS 的核心结构

AQS 最重要的两个东西：

```text
AQS
├── state
└── 等待队列
```

例如：

```text
             AQS
              │
       ┌──────┴──────┐
       ↓             ↓
     state        等待队列
```

------

# 七、state 是什么

对于 `ReentrantLock`，可以粗略把 `state` 理解成：

> **当前锁的同步状态以及重入次数。**

例如：

```text
state = 0
```

表示锁没有被持有。

线程 A 第一次获取：

```text
CAS(0, 1)

state = 1
owner = A
```

A 再次获取：

```text
state = 2
```

再次获取：

```text
state = 3
```

所以：

```text
state
  ↓
重入次数
```

------

# 八、CAS 是什么

CAS：

```text
Compare And Swap
```

例如：

```text
CAS(state, 0, 1)
```

意思：

```text
如果 state == 0
    ↓
把 state 改成 1
```

而且这个过程是**原子的**。

因此多个线程同时竞争时：

```text
线程 A ─┐
线程 B ─┤
线程 C ─┤ → CAS
线程 D ─┘
```

最终只有一个线程能够成功执行：

```text
0 → 1
```

------

# 九、CAS 不等于自旋

这是非常容易混淆的地方。

### CAS

是一种**原子操作**：

```text
CAS(state, 0, 1)
```

### 自旋

是不断重复尝试：

```java
while (!success) {
    CAS(...);
}
```

所以：

```text
CAS ≠ 自旋
```

自旋是：

```text
CAS失败
 ↓
继续CAS
 ↓
失败
 ↓
继续CAS
```

------

# 十、AQS 不是一直自旋

AQS 的核心思想不是：

```text
CAS失败
 ↓
无限CAS
```

而是：

```text
尝试获取锁
      ↓
     CAS
   ┌──┴──┐
成功    失败
 ↓       ↓
获得锁   加入等待队列
          ↓
        park()
          ↓
         睡眠
          ↓
      被唤醒
          ↓
      再次尝试
```

因此可以理解成：

> **AQS 通过 CAS 进行状态竞争，通过等待队列 + park/unpark 避免线程长期自旋。**

------

# 十一、为什么失败后不一直 CAS？

假设：

```text
A 持有锁 10 秒
```

如果 B 一直自旋：

```text
B：

CAS
CAS
CAS
CAS
CAS
...
```

那么 B 会一直占用 CPU。

如果同时有：

```text
B
C
D
E
F
```

都在自旋：

```text
CPU
├── B：CAS
├── C：CAS
├── D：CAS
├── E：CAS
└── F：CAS
```

实际上这些线程什么都没做，只是在浪费 CPU。

因此 AQS 会让竞争失败的线程进入等待队列，并通过：

```text
LockSupport.park()
```

让线程阻塞。

------

# 十二、park 和 unpark

AQS 使用：

```text
LockSupport.park()
LockSupport.unpark(thread)
```

控制线程的阻塞和唤醒。

### park

```text
线程
 ↓
park()
 ↓
阻塞
 ↓
释放 CPU
```

### unpark

```text
unpark(B)
 ↓
允许 B 从 park 状态恢复
```

注意：

> `unpark(B)` 不等于 B 已经获得锁。

它只是：

> **让 B 恢复执行，重新参与锁竞争。**

------

# 十三、ReentrantLock 的等待队列

假设：

```text
A 持有锁
```

B、C、D 获取失败：

```text
HEAD → B → C → D
```

可以粗略理解成：

```text
A
 ↓
持有锁

B
 ↓
等待

C
 ↓
等待

D
 ↓
等待
```

等待线程进入 AQS 队列后，可以通过：

```text
park()
```

阻塞。

------

# 十四、ReentrantLock 的完整加锁过程

假设：

```text
state = 0
```

线程 A：

```text
lock()
 ↓
CAS(0, 1)
 ↓
成功
 ↓
A 获得锁
```

此时：

```text
state = 1
owner = A
```

------

## 线程 B 来了

B：

```text
lock()
 ↓
CAS(0, 1)
 ↓
失败
```

因为：

```text
state = 1
```

所以 B：

```text
进入 AQS 等待队列
 ↓
park()
 ↓
睡眠
```

此时：

```text
HEAD → B
```

------

## 线程 C 来了

C：

```text
lock()
 ↓
获取失败
 ↓
加入队列
 ↓
park()
```

变成：

```text
HEAD → B → C
```

------

# 十五、unlock 的过程

A：

```java
lock.unlock();
```

大致：

```text
A
 ↓
unlock
 ↓
减少 state
 ↓
state = 0
 ↓
唤醒后继等待线程
```

例如：

```text
HEAD → B → C → D
```

通常唤醒：

```text
B
```

而不是：

```text
B、C、D 全部唤醒
```

------

# 十六、为什么 ReentrantLock 通常只唤醒一个？

因为 `ReentrantLock` 是**独占锁**。

一把锁：

```text
同一时刻
    ↓
只能一个线程持有
```

所以：

```text
HEAD → B → C → D
```

A 释放后：

```text
唤醒 B
```

B 获取成功：

```text
B → 持有锁

C → 继续等待
D → 继续等待
```

这样可以避免大量线程同时被唤醒，然后一起竞争锁。

------

# 十七、为什么只唤醒 B，还需要 CAS？

这是理解 AQS 最关键的问题之一。

很多人会想：

```text
A unlock
 ↓
唤醒 B
 ↓
既然只有 B 醒了
 ↓
为什么还 CAS？
```

原因：

> **唤醒 B，不等于把锁直接交给 B。**

`unpark(B)` 只是：

```text
让 B 恢复运行
```

而：

```text
CAS(state, 0, 1)
```

才是真正：

```text
把锁状态从“可获取”
变成
“由 B 持有”
```

------

# 十八、unpark 和 CAS 的职责不同

可以直接记：

```text
unpark
 ↓
负责唤醒线程

CAS
 ↓
负责原子地竞争/修改锁状态
```

所以：

```text
A unlock
 ↓
unpark(B)
 ↓
B 醒来
 ↓
CAS(0, 1)
 ↓
成功
 ↓
B 获得锁
```

------

# 十九、如果没人和 B 竞争，CAS 会失败吗？

假设严格限定：

```text
A 持有锁

HEAD → B → C → D
```

A 释放：

```text
A unlock
 ↓
state = 0
 ↓
unpark(B)
```

并且：

> 没有新线程参与竞争，B 也没有被取消或中断导致获取流程改变。

那么：

```text
B
 ↓
CAS(0, 1)
 ↓
成功
```

所以：

> **在这个简化场景下，B 的 CAS 正常会成功。**

CAS 仍然存在，因为它本身就是 AQS 获取锁的原子状态转换操作。

------

# 二十、为什么不直接把锁交给 B？

理论上可以设想：

```text
A unlock
 ↓
直接指定 B
 ↓
B 成为 owner
```

但 AQS 并不是这样设计的。

它把两个职责分开：

```text
unlock
 ↓
改变同步状态 + 唤醒后继线程

后继线程
 ↓
重新执行获取逻辑
 ↓
CAS
 ↓
确认自己真正获得锁
```

这样可以统一处理：

```text
线程竞争
线程取消
中断
超时
公平/非公平
队列状态
```

等情况。

------

# 二十一、非公平 ReentrantLock

默认：

```java
new ReentrantLock()
```

是：

> **非公平锁。**

假设：

```text
A 持有锁

队列：

HEAD → B → C → D
```

A 释放锁。

B 被唤醒。

但此时新线程 E 来了。

非公平锁允许 E 直接尝试：

```text
E
 ↓
CAS(0, 1)
 ↓
成功
```

于是：

```text
E → 获得锁
```

B：

```text
被唤醒
 ↓
CAS(0, 1)
 ↓
失败
 ↓
继续等待
```

这就是：

> **插队。**

------

# 二十二、公平 ReentrantLock

可以：

```java
ReentrantLock lock = new ReentrantLock(true);
```

创建公平锁。

假设：

```text
HEAD → B → C → D
```

新线程 E 到来：

```text
E
 ↓
发现前面已经有等待线程
 ↓
不能轻易插队
```

因此更倾向于：

```text
B → C → D → E
```

B 优先获得锁。

------

# 二十三、公平性到底是什么意思？

核心就是：

> **等待中的线程之间，后来者能不能插队。**

非公平：

```text
等待：

B → C → D

新线程 E
 ↓
允许直接竞争
 ↓
可能 E 抢到
```

公平：

```text
等待：

B → C → D

新线程 E
 ↓
不能轻易插队
 ↓
B 优先
```

所以你可以简单记：

```text
公平锁
 ↓
尽量按照等待顺序

非公平锁
 ↓
允许后来者插队
```

------

# 二十四、公平锁下为什么仍然需要 CAS？

即使是公平锁：

```text
HEAD → B → C → D
```

B 是最前面的等待线程。

A：

```text
unlock()
 ↓
唤醒 B
```

B：

```text
CAS(state, 0, 1)
 ↓
成功
```

这里：

```text
公平性
 ↓
决定谁优先竞争

CAS
 ↓
原子地完成状态转换
```

两者不是一个概念。

------

# 二十五、最核心的关系

把整个 `ReentrantLock` 串起来：

```text
                 ReentrantLock
                       ↓
                      AQS
                       ↓
              ┌────────┴────────┐
              ↓                 ↓
            state            等待队列
              ↓                 ↓
             CAS              park
              ↓                 ↓
         竞争锁状态          线程睡眠
                                ↓
                              unpark
                                ↓
                            线程唤醒
                                ↓
                              CAS
```

完整流程：

```text
线程 A
  ↓
lock()
  ↓
CAS
  ↓
成功
  ↓
持有锁
```

线程 B：

```text
lock()
  ↓
CAS失败
  ↓
进入 AQS 队列
  ↓
park()
  ↓
睡眠
```

A：

```text
unlock()
  ↓
释放锁
  ↓
唤醒 B
```

B：

```text
unpark
  ↓
恢复执行
  ↓
CAS
  ↓
成功
  ↓
获得锁
```

------

# 二十六、synchronized 和 ReentrantLock 对比

| 特性           | synchronized     | ReentrantLock |
| -------------- | ---------------- | ------------- |
| 类型           | Java 关键字      | Java 类       |
| 底层核心       | Monitor          | AQS           |
| 可重入         | 是               | 是            |
| 自动释放       | 是               | 否            |
| `finally` 解锁 | 不需要           | 必须          |
| 公平锁         | 不提供公平性保证 | 支持          |
| `tryLock()`    | 不支持           | 支持          |
| 超时获取       | 不支持           | 支持          |
| 可中断获取     | 不支持           | 支持          |
| Condition      | `wait/notify`    | `Condition`   |
| 灵活性         | 较低             | 较高          |
| 使用难度       | 简单             | 较高          |

------

# 二十七、面试回答模板

如果面试官问：

> ReentrantLock 和 synchronized 有什么区别？

可以回答：

> `synchronized` 是 Java 关键字，底层依赖 JVM 的 Monitor 机制实现同步；`ReentrantLock` 是 Java 并发包中的显式锁，底层基于 AQS，通过 `state`、CAS 和等待队列管理线程竞争。
>
> 两者都支持可重入。`synchronized` 使用简单，异常情况下 JVM 会自动释放锁；`ReentrantLock` 需要显式 `unlock`，通常放在 `finally` 中。
>
> `ReentrantLock` 提供了更多功能，例如公平锁、`tryLock`、超时获取、可中断获取以及 `Condition`，因此灵活性更高。
>
> 如果只是简单的互斥同步，`synchronized` 通常就够了；如果需要这些高级功能，可以考虑 `ReentrantLock`。

------

# 二十八、最容易混淆的几个概念

## 1. CAS ≠ 自旋

```text
CAS
=
一次原子操作

自旋
=
不断重复尝试 CAS
```

------

## 2. park ≠ 释放锁

```text
park()
 ↓
线程阻塞
```

它不是：

```text
park()
 ↓
释放锁
```

线程进入 `park()` 前，AQS 已经按照同步器逻辑处理好了等待状态。

------

## 3. unpark ≠ 获得锁

```text
unpark(B)
```

只是：

```text
B 可以恢复运行
```

不代表：

```text
B 已经获得锁
```

------

## 4. 公平 ≠ 一定严格 FIFO

公平锁主要解决：

```text
后来者能不能插队
```

而不是保证操作系统层面的绝对时间顺序。

------

# 二十九、最终记忆版

如果只记住 `ReentrantLock` 的核心链路：

```text
                 ReentrantLock
                       ↓
                      AQS
                       ↓
                 state + 队列
                       ↓
                  尝试 CAS
                  /       \
                成功       失败
                 ↓          ↓
               获得锁      入队
                            ↓
                           park
                            ↓
                           睡眠
                            ↓
                       前面的线程 unlock
                            ↓
                         unpark
                            ↓
                         线程醒来
                            ↓
                         再次 CAS
                       /          \
                    成功            失败
                     ↓               ↓
                  获得锁          继续等待
```

最重要的三句话：

> **1. CAS 负责原子地竞争/修改锁状态。**

> **2. park/unpark 负责等待线程的阻塞与唤醒。**

> **3. 公平性决定线程竞争锁时是否允许后来者插队。**