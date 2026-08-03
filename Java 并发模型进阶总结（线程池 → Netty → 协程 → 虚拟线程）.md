# Java 并发模型进阶总结（线程池 → Netty → 协程 → 虚拟线程）

------

# 一、线程池与任务类型

## 1.1 两类任务本质

| 类型      | 特点                  | 问题                     |
| --------- | --------------------- | ------------------------ |
| IO密集型  | 会阻塞（等网络/磁盘） | 线程被占着不干活         |
| CPU密集型 | 持续占用CPU           | 线程数过多会导致切换开销 |

------

## 1.2 为什么不能混用线程池？

```text
IO任务会阻塞线程 → CPU任务拿不到线程 → 性能下降
```

------

## 1.3 正确做法

```java
// IO线程池（多线程）
ExecutorService ioPool = Executors.newCachedThreadPool();

// CPU线程池（核数）
ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);
```

------

## 1.4 核心思想

> 🔥 不是减少线程，而是**避免线程被错误占用**

------

# 二、线程的本质问题

## 2.1 线程的限制

```text
1. 阻塞会占住线程
2. 线程数量有限
3. 上下文切换成本高
4. CPU缓存失效
```

------

## 2.2 为什么线程多反而慢？

```text
线程数 > CPU核数 →
    上下文切换 ↑
    缓存命中 ↓
    调度开销 ↑
```

------

## 2.3 本质总结

> 🔥 线程的问题不是“不够”，而是“使用方式不对”

------

# 三、Netty（事件驱动模型）

## 3.1 核心思想

> 🔥 一个线程处理多个请求（靠事件）

------

## 3.2 执行模型

```text
while (true) {
    等待事件（select/epoll）
    有事件 → 处理
}
```

------

## 3.3 为什么能高并发？

```text
非阻塞IO：
没数据 → 不阻塞
有数据 → 才处理
```

------

## 3.4 本质变化

```text
线程 ≠ 请求
线程 = 事件调度器
```

------

## 3.5 缺点

```text
❌ 回调地狱
❌ 代码复杂
❌ 可读性差
```

------

# 四、协程（Coroutine）

## 4.1 定义

> 🔥 协程 = 可挂起、可恢复的执行单元

------

## 4.2 核心能力

```text
✔ 挂起（suspend）
✔ 恢复（resume）
✔ 不占用线程
```

------

## 4.3 本质

```text
阻塞 → 挂起
```

------

## 4.4 回调 vs 协程

### 回调写法

```java
callA(data -> {
    callB(data, result -> {
        process(result);
    });
});
```

------

### 协程写法

```java
String a = callA();
String b = callB(a);
process(b);
```

------

## 4.5 核心区别

> 🔥 协程 = 把回调隐藏成同步代码

------

# 五、虚拟线程（Virtual Thread）

## 5.1 定义

> 🔥 虚拟线程 = Java里的协程实现（Project Loom）

------

## 5.2 使用方式

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

executor.submit(() -> {
    String data = callServiceA();
    process(data);
});
```

------

## 5.3 特点

```text
✔ 写法像线程
✔ 行为像协程
✔ 可以创建大量实例
```

------

## 5.4 核心机制

```text
阻塞 → 挂起 → 释放线程 → 恢复执行
```

------

## 5.5 与普通线程区别

| 维度 | 普通线程 | 虚拟线程 |
| ---- | -------- | -------- |
| 阻塞 | 占线程   | 不占线程 |
| 数量 | 几千     | 几十万   |
| 调度 | OS       | JVM      |

------

# 六、CompletableFuture 对比

## 6.1 CompletableFuture

```java
CompletableFuture
    .supplyAsync(() -> callA())
    .thenApply(a -> callB(a));
```

👉 本质：

```text
手动写异步流程
```

------

## 6.2 虚拟线程

```java
String a = callA();
String b = callB(a);
```

👉 本质：

```text
自动异步执行
```

------

## 6.3 总结

> 🔥 CompletableFuture = 显式异步
> 🔥 虚拟线程 = 隐式异步

------

# 七、挂起 vs 阻塞（核心区别）

## 7.1 线程

```text
阻塞 = 占着线程
```

------

## 7.2 协程

```text
挂起 = 释放线程
```

------

## 7.3 类比

```text
线程：占着座位等
协程：让出座位等
```

------

# 八、底层原理（Continuation）

## 8.1 核心机制

> 🔥 协程把“调用栈”变成数据

------

## 8.2 普通线程

```text
调用栈在内核 → 无法控制
```

------

## 8.3 协程

```text
调用栈在JVM → 可保存 / 恢复
```

------

## 8.4 执行流程

```text
执行 → 挂起 → 保存栈 → 恢复 → 继续执行
```

------

# 九、Pinning（虚拟线程大坑）

## 9.1 什么是 pinning？

> 🔥 虚拟线程无法挂起，被“钉在”OS线程上

------

## 9.2 典型错误

```java
synchronized(lock) {
    httpCall(); // ❌
}
```

------

## 9.3 原因

```text
synchronized 绑定OS线程 → 无法切换
```

------

## 9.4 解决方案

```java
Lock lock = new ReentrantLock();
```

或：

```text
不要在锁里做IO
```

------

# 十、Spring Boot 实战

## 10.1 开启虚拟线程

```properties
spring.threads.virtual.enabled=true
```

------

## 10.2 适用场景

```text
✔ Web接口
✔ 数据库访问
✔ RPC调用
```

------

## 10.3 注意事项

```text
❗ 控制数据库连接池
❗ 做限流
❗ 避免 synchronized
```

------

# 十一、三种模型总结

## 11.1 对比

| 模型          | 特点        |
| ------------- | ----------- |
| 线程池        | 简单但浪费  |
| Netty         | 高效但复杂  |
| 协程/虚拟线程 | 高效 + 易写 |

------

## 11.2 一句话总结

> 🔥 **线程：阻塞占用资源**
> 🔥 **Netty：避免阻塞**
> 🔥 **协程：允许阻塞但不占资源**

------

# 十二、终极总结（必须记住🔥）

> 🔥 **并发问题的本质不是线程不够，而是调度方式不对**
>
> 🔥 **协程/虚拟线程的核心价值：让“等待”不再浪费资源**