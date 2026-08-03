# 🧠 Java 并发锁体系完整笔记

------

# 一、并发问题本质

多线程访问共享资源会导致：

- 数据不一致
- 竞态条件（Race Condition）

👉 解决目标：

- 原子性（Atomicity）
- 可见性（Visibility）
- 有序性（Ordering）

------

# 二、并发控制的两种核心思想

------

## 1️⃣ 悲观锁（Blocking）

认为一定会发生冲突，直接加锁：

- synchronized
- ReentrantLock

特点：

- 线程互斥
- 未获取锁的线程阻塞

------

## 2️⃣ 乐观锁（Non-blocking）

认为冲突概率低，不加锁：

- CAS（Compare-And-Swap）

特点：

- 失败重试（自旋）
- 无阻塞

------

# 三、CAS（无锁机制）

------

## 1️⃣ 基本原理

CAS(V, old, new)：

- 如果 V == old → 更新为 new
- 否则 → 失败

👉 由 CPU 原子指令保证

------

## 2️⃣ 示例

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

底层逻辑：

```java
while (true) {
    int old = count;
    int newVal = old + 1;
    if (CAS(old, newVal)) break;
}
```

------

## 3️⃣ 优缺点

### ✅ 优点

- 无阻塞
- 高性能
- 无线程切换

### ❌ 缺点

- 自旋耗 CPU
- ABA 问题
- 只能保证单变量原子性

------

## ❗结论

👉 CAS 无法替代锁（无法保证多步骤原子性）

------

# 四、synchronized（JVM 内置锁）

------

## 1️⃣ 本质

- JVM 实现的 Monitor 锁
- 作用于对象头（Mark Word）

------

## 2️⃣ 使用方式

```java
synchronized (obj) {}

synchronized void method() {}
```

------

## 3️⃣ 特点

- 可重入
- 自动释放锁
- 简单安全

------

# 五、锁升级机制（核心🔥）

------

## 1️⃣ 升级路径

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

------

## 2️⃣ 各阶段详解

------

### 🟢 偏向锁（无竞争）

- 偏向第一个线程
- 不需要 CAS
- 几乎无开销

------

### 🟡 轻量级锁（少量竞争）

- CAS + 自旋
- 不阻塞线程

------

### 🔴 重量级锁（高竞争）

- 线程阻塞
- OS 互斥锁（mutex）
- 涉及用户态/内核态切换

------

## 3️⃣ 升级触发

| 场景     | 锁状态   |
| -------- | -------- |
| 单线程   | 偏向锁   |
| 少量竞争 | 轻量级锁 |
| 激烈竞争 | 重量级锁 |

------

## ❗重要结论

- 锁只会升级，不会降级
- synchronized 不是一开始就是互斥锁

------

# 六、synchronized 本质总结

```
能不锁 → 不锁（偏向锁）
能CAS → CAS（轻量级锁）
实在不行 → 阻塞（重量级锁）
```

👉 本质：JVM 动态优化锁

------

# 七、ReentrantLock（JUC 锁）

------

## 1️⃣ 本质

- 基于 AQS（AbstractQueuedSynchronizer）
- 使用 CAS + 队列

------

## 2️⃣ 使用方式

```java
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}
```

------

## 3️⃣ 特点

### ✅ 优点

- 可中断（lockInterruptibly）
- tryLock
- 公平锁 / 非公平锁
- Condition（多个等待队列）

------

### ❌ 缺点

- 需要手动释放锁
- 易出错（可能死锁）
- 无锁升级机制

------

# 八、synchronized vs ReentrantLock

------

| 维度     | synchronized | ReentrantLock |
| -------- | ------------ | ------------- |
| 实现层   | JVM          | Java（AQS）   |
| 锁升级   | ✅ 有         | ❌ 没有        |
| 自动释放 | ✅            | ❌             |
| 可中断   | ❌            | ✅             |
| tryLock  | ❌            | ✅             |
| 公平锁   | ❌            | ✅             |

------

## 🎯 选择原则

- 简单同步 → synchronized
- 复杂并发 → ReentrantLock

------

# 九、锁升级 vs AQS（核心区别）

------

## synchronized

```
偏向锁 → CAS → 阻塞
```

------

## ReentrantLock（AQS）

```
CAS → 入队 → 阻塞
```

------

👉 区别：

- synchronized：状态逐步升级
- ReentrantLock：获取失败直接进入队列

------

# 十、整体体系总结（最重要🔥）

------

## 三层并发控制模型

------

### 1️⃣ 无锁

- CAS
- AtomicXXX

👉 高性能，但功能有限

------

### 2️⃣ 轻量级锁

- CAS + 自旋

👉 不阻塞线程

------

### 3️⃣ 重量级锁

- synchronized（重量级）
- ReentrantLock

👉 阻塞线程，保证强一致性

------

# 🧾 面试总结（必背）

------

Java 并发控制分为：

1. 悲观锁（synchronized、ReentrantLock）
2. 乐观锁（CAS）

------

CAS：

- 无锁机制
- 只能保证单变量原子性

------

synchronized：

- JVM 内置锁
- 支持锁升级（偏向锁 → 轻量级锁 → 重量级锁）
- 自动优化性能

------

ReentrantLock：

- 基于 AQS
- 功能更强（tryLock、公平锁、Condition）
- 无锁升级机制

------

✅ 最终结论：

- CAS 是锁的补充
- synchronized 是自动优化锁
- ReentrantLock 是可控高级锁

------