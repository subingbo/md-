# CountDownLatch 和 CyclicBarrier 区别详解

## 1. 概述

`CountDownLatch` 和 `CyclicBarrier` 都是 Java 并发包 `java.util.concurrent` 中用于线程同步的工具。

它们看起来都涉及一个“计数”，但是解决的问题不同：

- **CountDownLatch：等待多个任务完成**
- **CyclicBarrier：多个线程等待彼此到达同一个阶段**

一句话：

> CountDownLatch 是“别人完成通知我”；CyclicBarrier 是“大家到齐一起走”。

---

# 一、CountDownLatch

## 1.1 基本概念

`CountDownLatch`（倒计时锁存器）：

内部维护一个计数器：

count = N

```
当任务完成时调用：

```java
countDown()
```

计数减 1。

当：

```
count == 0
```

等待线程被唤醒。

------

## 1.2 工作模型

```
             主线程

              |
           await()
              |
              |
      -------------------
      |        |        |
     任务1     任务2     任务3
      |        |        |
 countDown countDown countDown


count = 0

        ↓

主线程继续执行
```

特点：

- 一个线程等待多个线程
- 等待的是任务完成事件
- 计数只能减少
- 一次性使用

------

## 1.3 使用示例

例如：

系统启动需要加载：

- 配置文件
- 数据库连接
- 缓存数据

```java
CountDownLatch latch = new CountDownLatch(3);


new Thread(() -> {

    loadConfig();

    latch.countDown();

}).start();


new Thread(() -> {

    loadDatabase();

    latch.countDown();

}).start();


new Thread(() -> {

    loadCache();

    latch.countDown();

}).start();



latch.await();


System.out.println("系统启动完成");
```

执行过程：

```
初始：

count = 3


配置加载完成：

count = 2


数据库加载完成：

count = 1


缓存加载完成：

count = 0


await() 被唤醒
```

------

## 1.4 CountDownLatch 的特点

### ① 不能重置

例如：

```
count = 3

countDown()
countDown()
countDown()

count = 0
```

之后：

```
永远保持 0
```

无法恢复：

```
count = 3
```

所以：

> CountDownLatch 是一次性的。

------

### ② count 表示任务数量

例如：

```java
new CountDownLatch(5)
```

表示：

```
还有5个任务没有完成
```

不是：

```
5个线程必须集合
```

任务完成：

```
countDown()
```

------

# 二、CyclicBarrier

## 2.1 基本概念

`CyclicBarrier`（循环屏障）：

用于多个线程互相等待。

创建：

```java
CyclicBarrier barrier =
        new CyclicBarrier(3);
```

表示：

> 需要3个线程同时到达屏障点。

------

## 2.2 工作模型

```
线程1
 |
await()
 |
等待


线程2
 |
await()
 |
等待


线程3
 |
await()
 |
达到数量


================

三个线程一起继续
```

------

## 2.3 使用示例

假设：

三个线程完成第一阶段后，再执行第二阶段。

```java
CyclicBarrier barrier =
        new CyclicBarrier(3);



线程1:

step1();

barrier.await();

step2();



线程2:

step1();

barrier.await();

step2();



线程3:

step1();

barrier.await();

step2();
```

执行：

```
线程1:
完成step1
等待


线程2:
完成step1
等待


线程3:
完成step1


所有线程到达


继续执行step2
```

------

# 三、CountDownLatch 能否实现 CyclicBarrier？

答案：

> 可以实现一次性的类似效果，但是不能完全替代。

------

## 3.1 模拟一次屏障

例如：

三个线程需要等待：

```
线程1完成
线程2完成
线程3完成
```

可以：

```java
CountDownLatch latch =
        new CountDownLatch(3);


线程:

step1();

latch.countDown();

latch.await();

step2();
```

第一次运行：

```
线程1 countDown

count=2


线程2 countDown

count=1


线程3 countDown

count=0


所有线程继续
```

效果类似 Barrier。

------

# 四、两者核心区别

| 对比       | CountDownLatch       | CyclicBarrier    |
| ---------- | -------------------- | ---------------- |
| 中文       | 倒计时锁存器         | 循环屏障         |
| 核心目的   | 等任务完成           | 等线程集合       |
| 谁等待     | 一个线程等待多个任务 | 多个线程互相等待 |
| count含义  | 未完成任务数量       | 参与线程数量     |
| 减少方式   | countDown()          | await()到达      |
| 是否可重复 | ❌ 不可               | ✅ 可循环         |
| 典型模型   | 1 等 N               | N 等 N           |

------

# 五、生活类比

## CountDownLatch

老师等待学生交作业：

```
老师:

等待3份作业


学生A √
学生B √
学生C √


老师开始批改
```

老师不参与计数。

------

## CyclicBarrier

朋友约饭：

```
朋友A 到达
等待

朋友B 到达
等待

朋友C 到达


人齐

一起吃饭
```

每个人都是参与者。

------

# 六、应用场景

## CountDownLatch 常用场景

### 1. 主线程等待子任务

例如：

```
订单处理:

查询库存
查询价格
查询优惠


全部完成

返回结果
```

### 2. 并发测试

例如：

100个线程同时请求：

```java
latch.await();
```

等待统一开始。

------

### 3. 服务启动

例如：

Spring Boot 启动：

```
加载配置
初始化缓存
连接数据库

完成后启动服务
```

------

## CyclicBarrier 常用场景

### 1. 多阶段计算

例如 MapReduce：

```
Map阶段

↓

等待所有Map完成

↓

Reduce阶段
```

------

### 2. 并行算法

例如：

多个线程计算矩阵：

```
第一轮计算

↓

同步

↓

第二轮计算

↓

同步
```

------

# 七、底层理解

## CountDownLatch

关注：

```
任务有没有完成
```

模型：

```
任务
 |
countDown
 |
等待者
 |
await
```

------

## CyclicBarrier

关注：

```
线程有没有到齐
```

模型：

```
线程1 \
线程2  ---> barrier ---> 下一阶段
线程3 /
```

------

# 八、选择原则

如果你的问题是：

> 我要等待几个异步任务结束

选择：

```
CountDownLatch
```

例如：

```
三个接口并发调用
等待全部返回
合并结果
```

------

如果你的问题是：

> 多个线程必须同时进入下一阶段

选择：

```
CyclicBarrier
```

例如：

```
100个线程完成第一轮计算
一起进入第二轮计算
```

------

# 九、最终记忆

```
CountDownLatch:

完成一个任务，减少一次计数
count到0，通知等待者


CyclicBarrier:

线程到达一个集合点
人数满足，一起继续
可以重复使用
```

核心区别：

> CountDownLatch 等的是“事情完成”，CyclicBarrier 等的是“人到齐”。