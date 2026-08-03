# Java并发基础学习文档：synchronized、wait/notify、CAS、自旋、volatile、伪共享

---

# 一、synchronized 同步锁

## 1. synchronized 的本质

`synchronized` 的本质：

> 通过对象的 Monitor（监视器）实现互斥访问。

任何 synchronized 最终都是：

```java
synchronized(某个对象){

    临界区代码

}
```

其中：

* 某个对象 = 锁对象
* 临界区 = 同一时间只能一个线程执行的代码

---

# 二、什么是临界区？

临界区：

> 访问共享资源，并且需要保证线程安全的一段代码。

例如：

```java
class Counter {

    int count = 0;

    void add(){

        count++;

    }
}
```

多个线程同时执行：

```java
count++;
```

实际上：

```
读取count
+
加1
+
写回count
```

不是一个原子操作。

可能：

```
初始：
count=0


线程A:
读取0

线程B:
读取0


A写1

B写1


最终：
count=1
```

丢失一次更新。

所以：

```java
synchronized(this){

    count++;

}
```

保证：

```
获取锁
 ↓
进入临界区
 ↓
修改共享数据
 ↓
释放锁
```

---

# 三、synchronized 三种形式

## 1. synchronized代码块

```java
Object lock = new Object();

synchronized(lock){

}
```

锁：

```
lock对象
```

什么时候获取锁：

```
执行到代码块时
```

---

## 2. synchronized实例方法

```java
public synchronized void add(){

}
```

等价：

```java
public void add(){

    synchronized(this){

    }

}
```

锁：

```
当前对象this
```

获取时机：

```
调用方法时
```

---

## 3. synchronized静态方法

```java
public static synchronized void add(){

}
```

等价：

```java
synchronized(ClassName.class){

}
```

锁：

```
Class对象
```

原因：

静态方法没有：

```
this
```

所以使用：

```
类对象作为锁
```

---

# 四、锁对象决定谁互斥

核心原则：

> 拿同一把锁的线程才会互相等待。

---

## 同一个对象

```java
Counter c = new Counter();
```

两个线程：

```
线程A:
c.add()


线程B:
c.add()
```

锁：

```
c对象
```

互斥。

---

## 不同对象

```java
Counter c1 = new Counter();

Counter c2 = new Counter();
```

线程：

```
c1.add()

c2.add()
```

锁：

```
c1对象

c2对象
```

两把锁。

不互斥。

---

# 五、为什么使用 private Object lock？

推荐：

```java
private final Object lock = new Object();
```

原因：

## 1. 控制锁粒度

例如：

```java
public void test(){

    查询数据库();

    synchronized(lock){

        修改数据;

    }

    打印日志();

}
```

只有修改数据需要锁。

---

## 2. 防止外部抢锁

如果：

```java
synchronized(this)
```

外部可以：

```java
synchronized(obj){

}
```

抢你的锁。

private lock：

外部无法访问。

---

# 六、wait / notify / notifyAll

## 1. 作用

它们用于：

> 线程之间等待和唤醒。

---

## 2. 调用对象

例如：

```java
lock.wait();

lock.notify();
```

这里：

代码操作：

```
lock对象
```

实际影响：

```
线程
```

---

## 3. 为什么必须持有锁？

调用：

```java
lock.wait();
```

前必须：

```
当前线程持有lock锁
```

否则：

```
IllegalMonitorStateException
```

---

正确：

```java
synchronized(lock){

    lock.wait();

}
```

流程：

```
获得lock锁

↓

wait()

↓

释放lock锁

↓

进入等待队列
```

---

# 七、notify到底做什么？

notify：

不是：

```
把锁交给别人
```

而是：

```
唤醒等待这个锁的线程
```

流程：

线程A：

```
持有lock

修改共享状态

notify()

继续执行

释放lock
```

线程B：

```
被唤醒

重新竞争lock

成功后继续执行
```

---

# 八、sleep 和 wait区别

|        | wait     | sleep    |
| ------ | -------- | -------- |
| 属于   | Object   | Thread   |
| 作用   | 线程等待 | 线程休眠 |
| 释放锁 | 释放     | 不释放   |
| 需要锁 | 需要     | 不需要   |

---

例：

```java
synchronized(lock){

    Thread.sleep(10000);

}
```

10秒期间：

```
线程仍然持有lock
```

---

# 九、自旋锁（Spin）

## 1. 什么是自旋？

线程拿不到锁时：

不马上阻塞。

而是：

```
循环尝试获取锁
```

例如：

```
while(lock没有释放){

}
```

---

## 2. 为什么自旋？

因为线程阻塞有成本：

```
运行状态

↓

阻塞

↓

唤醒

↓

重新调度
```

需要上下文切换。

---

自旋：

```
继续占CPU

等待锁释放

立即获取
```

避免切换。

---

## 3. 自旋适合

短等待：

```
几纳秒
几微秒
```

不适合：

```
长时间等待
```

否则：

```
CPU空转
```

---

# 十、偏向锁和自旋区别

## 偏向锁

解决：

> 一个线程反复获取同一把锁。

例如：

```
线程A

进入锁
退出

再次进入

再次进入
```

JVM记录：

```
锁偏向线程A
```

减少竞争成本。

---

## 自旋

解决：

> 锁被别人短暂占用。

例如：

```
线程A持锁

线程B等待

A马上释放

B直接获取
```

---

区别：

|      | 偏向锁           | 自旋         |
| ---- | ---------------- | ------------ |
| 层次 | JVM锁优化        | 线程等待策略 |
| 目的 | 减少重复加锁成本 | 减少阻塞切换 |
| 对象 | 锁状态           | 线程行为     |

---

# 十一、CAS

CAS：

Compare And Swap

比较并交换。

核心：

```
旧值 == 期待值

↓

修改新值
```

例如：

```
当前count=0


线程A:
CAS(0,1)

成功


线程B:
CAS(0,1)

失败
```

---

CAS特点：

优点：

```
无锁
高性能
```

缺点：

```
竞争激烈时不断重试
```

---

# 十二、volatile

## 1. volatile不是锁

错误：

```
volatile保证线程安全
```

正确：

```
volatile保证可见性和有序性
```

---

# 2. volatile保证可见性

普通变量：

```
CPU1缓存:

x=1


CPU2缓存:

x=0
```

volatile：

写入后：

```
通知其他CPU缓存失效
```

读取：

```
重新获取最新值
```

---

# 3. volatile禁止重排序

例如：

```java
int data;

volatile boolean ready;
```

线程A：

```java
data=100;

ready=true;
```

volatile保证：

不能变：

```
ready=true

↓

data=100
```

必须：

```
data=100

↓

ready=true
```

---

## volatile保护范围

不是：

```
保护整个代码块
```

而是：

```
volatile变量读写前后的内存操作顺序
```

---

# 十三、伪共享（False Sharing）

## 1. 什么是伪共享？

多个线程修改：

不同变量

但是：

变量位于同一个CPU缓存行。

例如：

```
Cache Line

| x | y |
```

线程：

```
CPU1修改x

CPU2修改y
```

虽然：

```
x和y没有关系
```

但是：

缓存行整体失效。

导致：

```
缓存同步频繁

性能下降
```

---

# 十四、为什么伪共享示例使用volatile + while？

例如：

```java
volatile long x;

while(true){

    x++;

}
```

目的：

## while

制造大量写操作：

```
不断修改变量
```

放大缓存竞争。

---

## volatile

保证：

```
真实发生内存访问

禁止JVM优化掉
```

让CPU缓存一致性开销出现。

---

# 十五、整体关系总结

```
                Java并发
                    |
        ----------------------------
        |                          |
      线程同步                  CPU优化
        |                          |
   synchronized                 CAS
        |                          |
   wait/notify                自旋
        |
   线程等待控制


CPU层:
        |
     volatile
        |
   内存可见性
   指令顺序


硬件层:
        |
    缓存行
        |
    伪共享
```

---

# 最终记忆版

| 技术         | 解决问题                 |
| ------------ | ------------------------ |
| synchronized | 同一时间只能一个线程访问 |
| wait/notify  | 线程等待和唤醒           |
| sleep        | 线程休眠，不释放锁       |
| CAS          | 无锁更新共享变量         |
| 自旋         | 避免短等待阻塞           |
| volatile     | 可见性 + 禁止重排序      |
| 偏向锁       | 减少单线程重复加锁       |
| 伪共享       | 减少CPU缓存行竞争        |

核心思想：

> synchronized解决“谁能进去”；
> wait/notify解决“什么时候继续”；
> CAS解决“怎么无锁修改”；
> volatile解决“别人什么时候看到”；
> 自旋解决“要不要阻塞”；
> 伪共享解决“CPU缓存为什么慢”。