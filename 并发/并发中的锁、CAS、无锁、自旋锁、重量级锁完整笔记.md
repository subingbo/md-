# 并发中的锁、CAS、无锁、自旋锁、重量级锁完整笔记

------

# 一、并发控制到底在解决什么问题

核心问题：

```text
多个线程同时修改共享数据时，
如何保证数据正确。
```

例如：

```java
count++;
```

实际上不是原子操作：

```text
读取 count
+1
写回 count
```

如果多个线程同时执行：

可能出现：

```text
线程A读取10
线程B读取10

A写11
B写11
```

最终：

```text
结果应该是12
却变成11
```

这叫：

# 并发冲突

------

# 二、并发控制的两大思想

------

# 1. 悲观锁（Pessimistic Lock）

思想：

```text
我认为一定会冲突，
所以先锁起来。
```

即：

```text
先获得执行权限，
再执行逻辑。
```

例如：

- synchronized
- ReentrantLock
- mutex

------

# 2. 乐观锁（Optimistic Lock）

思想：

```text
我认为大概率没人冲突，
先执行，
失败再重试。
```

例如：

- CAS
- AtomicInteger
- 无锁结构

------

# 三、真正的同步机制分类（最重要）

真正并列的：

```text
无锁（lock-free）
自旋锁（spin lock）
阻塞锁（blocking lock）
```

------

# 四、CAS到底是什么

CAS：

# Compare And Swap（比较并交换）

是：

# CPU提供的原子指令。

例如：

```java
compareAndSet(old, new)
```

意思：

```text
如果当前值还是old
就修改成new
否则失败
```

------

# CPU底层

例如x86：

```asm
lock cmpxchg
```

CPU保证：

整个过程原子执行。

------

# 五、无锁（Lock-Free）

------

# 无锁的核心思想

```text
不提前限制线程执行。
```

所有线程：

都可以：

```text
直接尝试修改数据。
```

最后：

通过CAS决定：

```text
谁提交成功。
```

------

# 无锁核心流程

```java
while(true){
    old = value;
    new = old + 1;

    if(CAS(old,new))
        break;
}
```

------

# 注意

无锁：

不是：

```text
没有同步
```

而是：

```text
不用mutex/synchronized这种阻塞锁。
```

------

# 无锁真正本质

```text
允许多个线程同时执行逻辑，
只在最终提交时竞争。
```

------

# 六、自旋锁（Spin Lock）

------

# 自旋锁核心思想

```text
先抢执行权限，
抢不到就一直等。
```

例如：

```java
while(!CAS(lock, false, true)){
}
```

------

# 自旋锁特点

## 有锁拥有者

即：

```text
同一时间只有一个线程能进入临界区。
```

------

# 自旋锁本质

```text
CAS抢锁 + while循环等待
```

------

# 自旋锁失败线程在干什么

```text
空转等待锁释放。
```

即：

```text
busy waiting
```

------

# 七、重量级锁（Heavyweight Lock）

------

# 重量级锁核心思想

```text
抢不到锁就睡觉。
```

例如：

- mutex
- futex
- monitor

------

# 流程

```text
线程A拿锁
线程B抢不到
B park/sleep
```

锁释放：

```text
OS唤醒B
```

------

# 本质

```text
阻塞等待（blocking wait）
```

------

# 八、真正的核心区别（非常重要）

------

# 无锁

```text
谁都能执行，
最后竞争提交结果。
```

------

# 自旋锁

```text
只有拿到锁的人能执行，
别人while空转等待。
```

------

# 重量级锁

```text
只有拿到锁的人能执行，
别人直接睡觉等待。
```

------

# 九、锁 vs 无锁真正区别

------

# 锁

```text
先限制谁能执行，
再修改数据。
```

即：

```text
执行前串行化。
```

------

# 无锁

```text
所有线程都能执行，
最后再竞争提交。
```

即：

```text
提交时串行化。
```

------

# 十、为什么CAS低竞争更快

因为：

## CAS成功成本极低

例如：

```asm
lock cmpxchg
```

可能：

```text
几十ns
```

------

# 而锁可能涉及

- park
- syscall
- context switch
- scheduler

代价：

```text
几千ns~几万ns
```

------

# 所以：

低竞争下：

```text
CAS几乎总一次成功
```

于是：

非常快。

------

# 十一、为什么CAS高竞争会崩

因为：

多个线程：

同时CAS：

```text
同一个cache line
```

导致：

- cache line ping-pong
- invalidate storm
- coherence traffic
- retry风暴

------

# 结果

CPU大量时间：

消耗在：

```text
失败重试
```

而不是有效工作。

------

# 十二、cache line抖动

多个CPU核心：

同时修改：

```text
同一个cache line
```

CPU需要：

```text
缓存一致性同步
```

导致：

cache line：

```text
在多个core之间来回迁移
```

这叫：

# cache ping-pong

------

# 十三、为什么LongAdder更快

------

# AtomicLong的问题

所有线程：

CAS：

```text
同一个值
```

竞争极其激烈。

------

# LongAdder的思想

```text
把一个逻辑值拆成多个Cell
```

例如：

```text
Cell0
Cell1
Cell2
```

不同线程：

修改不同Cell。

最后：

```text
sum()
```

汇总。

------

# 本质

```text
减少对同一个cache line的竞争。
```

------

# 十四、自旋锁 vs 重量级锁

真正区别：

# 抢不到锁以后怎么办

------

# 自旋锁

```text
继续占CPU等待
```

优点：

- 无线程切换
- 低延迟

缺点：

- 烧CPU
- 高竞争崩

------

# 重量级锁

```text
线程睡觉等待
```

优点：

- CPU稳定
- 高竞争稳定

缺点：

- context switch昂贵

------

# 十五、轻量级锁是什么

JVM对synchronized的优化。

核心：

```text
CAS + 自旋
```

即：

```text
先尝试轻量竞争，
不要立刻阻塞线程。
```

------

# 十六、synchronized锁升级

现代JVM：

```text
无锁
  ↓
偏向锁（旧版本）
  ↓
轻量级锁（CAS+spin）
  ↓
重量级锁（mutex）
```

动态升级。

------

# 十七、真正的终极理解（最重要）

------

# 所有共享写最终都必须串行

因为：

```text
同一内存不可能同时被多个核心真正写入。
```

------

# 真正区别不在：

```text
最终谁写成功
```

而在：

# “系统在让这个成功发生前，

允许多少并发。”

------

# 锁

```text
提前限制并发。
```

------

# CAS

```text
允许并发，
最后再收敛。
```

------

# 十八、终极总结图

```text
【CPU层】
CAS原子指令
memory barrier
cache coherence

    ↓

【同步机制层】
无锁
自旋锁
阻塞锁

    ↓

【JVM实现层】
AtomicInteger
轻量级锁
重量级锁

    ↓

【Java语法层】
synchronized
ReentrantLock
```

------

# 十九、一句话总结所有概念

------

## CAS

```text
CPU提供的原子比较交换工具。
```

------

## 无锁

```text
不提前限制线程，
最后竞争提交结果。
```

------

## 自旋锁

```text
先抢执行权，
抢不到就while等待。
```

------

## 重量级锁

```text
先抢执行权，
抢不到就睡觉。
```

------

## 锁

```text
执行前串行化。
```

------

## 无锁

```text
提交时串行化。
```