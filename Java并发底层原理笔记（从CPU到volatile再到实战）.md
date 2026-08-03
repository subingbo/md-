# Java并发底层原理笔记（从CPU到volatile再到实战）

------

# 一、问题起点：为什么需要理解底层？

在多线程程序中，经常会遇到这些问题：

- 明明写了变量，另一个线程却读不到
- 程序偶尔出错，且无法复现
- CPU占用很高，但程序没做什么事
- 多线程反而比单线程慢

👉 这些问题的根本原因：

```text
CPU缓存 + 内存乱序 + 可见性问题
```

------

# 二、CPU层基础（必须理解）

## 2.1 Cache Line（缓存行）

- CPU缓存按 **Cache Line（通常64字节）** 存储
- 一个变量不是单独缓存，而是和周围数据一起缓存

👉 关键影响：

```text
多个变量可能在同一个 Cache Line 中
```

------

## 2.2 Store Buffer（写缓冲）

### 定义

```text
CPU写数据时不会立即写入内存，而是先写入Store Buffer
```

### 作用

- 避免CPU等待内存（提高性能）

### 问题

```text
写入延迟 → 其他线程看不到最新值
```

------

## 2.3 Invalidate Queue（失效队列）

### 定义

```text
CPU不会立刻处理缓存失效通知，而是放入队列延迟处理
```

### 问题

```text
读到旧值（因为缓存还没失效）
```

------

# 三、内存屏障（Memory Barrier）

## 3.1 分类

| 类型  | 作用         |
| ----- | ------------ |
| wmb() | 保证写顺序   |
| rmb() | 保证读顺序   |
| mb()  | 保证读写顺序 |

------

## 3.2 本质

```text
wmb → 刷新 Store Buffer
rmb → 清空 Invalidate Queue
```

------

# 四、Java内存模型（JMM）

Java通过JMM屏蔽CPU差异，提供统一语义。

------

# 五、volatile关键字

## 5.1 示例代码

```java
volatile boolean flag = false;

// 线程1
flag = true;

// 线程2
while (!flag) {}
```

------

## 5.2 为什么不会死循环？

### 没有 volatile 时

```text
线程1：
flag=true → 可能在Store Buffer

线程2：
一直读取旧缓存 → 死循环
```

------

### 有 volatile 时

#### 写操作

```text
写屏障（类似 wmb）：
- 强制刷新 Store Buffer
- 通知其他CPU缓存失效
```

#### 读操作

```text
读屏障（类似 rmb）：
- 清空 Invalidate Queue
- 强制读取最新值
```

------

## 5.3 一句话总结

```text
volatile = 可见性 + 有序性（但不保证原子性）
```

------

# 六、伪共享（False Sharing）

## 6.1 定义

```text
多个线程访问不同变量，但这些变量在同一个Cache Line
→ 导致性能下降
```

------

## 6.2 示例

```java
class Counter {
    volatile long a;
    volatile long b;
}
```

两个线程分别修改 a 和 b：

👉 仍然会冲突（因为同一 Cache Line）

------

## 6.3 解决方案

```java
@sun.misc.Contended
class Counter {
    volatile long a;
}
```

或：

```java
class Counter {
    long a;
    long padding1, padding2, padding3, padding4, padding5, padding6, padding7;
}
```

------

# 七、CAS（Compare-And-Swap）

## 7.1 示例

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

------

## 7.2 本质

```text
CAS依赖：
- 可见性（volatile）
- 内存顺序
```

------

# 八、真实业务场景（非常重要）

## 场景：高并发开关控制（配置热更新）

👉 常见于：

- 限流开关
- 灰度发布
- 功能开关

------

## ❌ 错误写法（可能失效）

```java
class Config {
    boolean enabled = false;
}

Config config = new Config();

// 更新线程
config.enabled = true;

// 工作线程
while (!config.enabled) {
    // 等待开启
}
```

👉 问题：

```text
可能永远看不到 enabled=true
```

------

## ✅ 正确写法（使用volatile）

```java
class Config {
    volatile boolean enabled = false;
}

Config config = new Config();

// 更新线程
config.enabled = true;

// 工作线程
while (!config.enabled) {
    // 等待开启
}
```

------

## ✅ 更优写法（避免CPU空转）

```java
class Config {
    volatile boolean enabled = false;
}

Config config = new Config();

// 工作线程
while (!config.enabled) {
    Thread.sleep(10);
}
```

------

## 🚀 实际业务优化版本（推荐）

```java
class FeatureSwitch {

    private volatile boolean enabled = false;

    public void enable() {
        this.enabled = true;
    }

    public void disable() {
        this.enabled = false;
    }

    public boolean isEnabled() {
        return enabled;
    }
}
```

使用：

```java
if (featureSwitch.isEnabled()) {
    // 执行业务逻辑
}
```

------

# 九、为什么有时候锁反而更快？

## 场景

```java
while (!atomic.compareAndSet(x, y)) {
    // 自旋
}
```

👉 问题：

- 高竞争 → CAS频繁失败
- CPU空转
- cache line抖动

------

## 结论

```text
低竞争 → CAS快
高竞争 → 锁更好
```

------

# 十、核心知识链总结

```text
CPU缓存
 → Cache Line
 → Store Buffer（写延迟）
 → Invalidate Queue（读延迟）
 → 内存屏障
 → JMM
 → volatile
 → CAS
 → 伪共享
 → 性能优化
```

------

# 十一、面试总结（必背）

## volatile作用

```text
1. 保证可见性（通过内存屏障）
2. 禁止指令重排
3. 不保证原子性
```

------

## 伪共享

```text
多个线程访问不同变量，但在同一Cache Line，导致性能下降
```

------

## 内存屏障

```text
wmb：保证写顺序
rmb：保证读顺序
```

------

# 十二、最终一句话理解

```text
并发问题的本质 = CPU缓存 + 可见性 + 顺序性
```

------

# 完