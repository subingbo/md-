# 🚀 高并发计数与 CAS 优化全链路笔记

------

# 一、问题起点：CAS 在高并发下为什么变慢？

## 1️⃣ 表面现象

```text
多个线程同时 CAS 同一个变量
→ 不断失败 + 自旋重试
→ 性能下降
```

------

## 2️⃣ 本质原因（核心‼️）

> ❗不是自旋本身，而是 **cache line 竞争**

```text
多个 CPU 核同时修改同一个变量
→ 该变量所在 cache line 来回迁移
→ cache coherence（MESI）开销巨大
```

------

## 3️⃣ 关键结论

> CAS 慢的本质是：
> ❗**多个线程竞争同一个 cache line**

------

# 二、两种竞争模型（必须区分）

------

## 🔴 真共享（True Sharing）

```text
多个线程 → 修改同一个变量
AtomicLong count = new AtomicLong();
count.incrementAndGet();
```

👉 必然竞争同一个 cache line ❌

------

## 🟡 伪共享（False Sharing）

```text
多个线程 → 修改不同变量，但在同一 cache line
class A {
    volatile long x;
    volatile long y;
}
```

👉 也会冲突 ❌

------

# 三、解决思路（核心思想）

> ❗避免多个线程同时写同一个 cache line

------

## 解决方向：

```text
1️⃣ 分散变量（最重要）
2️⃣ 减少竞争线程
3️⃣ 避免伪共享
```

------

# 四、LongAdder 的核心思想

------

## 1️⃣ 本质结构

```text
LongAdder =
    base（低竞争）
    +
    Cell[]（高竞争分散）
```

------

## 2️⃣ 工作方式

### 🔹 低竞争

```text
直接 CAS base
```

------

### 🔹 高竞争

```text
线程A → cell[0]
线程B → cell[1]
线程C → cell[2]
```

👉 每个线程写不同变量 ✔

------

## 3️⃣ 最终结果

```java
sum = base + 所有 cell
```

------

## 4️⃣ 核心优势

> ❗把“单点竞争” → “分布式写入”

------

# 五、为什么 LongAdder 更快？

------

## ❌ AtomicLong

```text
所有线程 → 一个变量
→ cache line 抖动
→ CAS 冲突严重
```

------

## ✅ LongAdder

```text
线程 → 不同 cell
→ 不同 cache line
→ 无竞争
```

------

# 六、LongAdder 使用方式

------

## 基本用法

```java
LongAdder adder = new LongAdder();

adder.increment();   // +1
adder.add(5);        // +5

long v = adder.sum();
```

------

## 注意点

> ❗sum() 不是强一致（统计用 OK）

------

# 七、ConcurrentHashMap + LongAdder（经典组合）

------

## 标准写法

```java
map.computeIfAbsent(key, k -> new LongAdder())
   .increment();
```

------

## 含义

```text
没有计数器 → 创建一个
有计数器 → 直接使用
然后 +1
```

------

## 等价写法

```java
LongAdder adder = map.computeIfAbsent(key, k -> new LongAdder());
adder.increment();
```

------

# 八、computeIfAbsent 原理

------

## 1️⃣ 线程安全保证

来自：

- CAS（空桶初始化）
- synchronized（桶级锁）

------

## 2️⃣ 执行流程

```text
1. hash 定位桶
2. 桶为空 → CAS 插入
3. 桶不为空 → 锁桶
4. 查找 key
    存在 → 返回
    不存在 → 创建 value
```

------

## 3️⃣ 关键特性

> ❗只锁一个桶，不锁整个 map

------

## 4️⃣ lambda 注意点

```java
k -> new LongAdder()
```

> ❗可能执行多次，但只会有一个生效
> ❗不能写副作用代码

------

# 九、整体设计精髓（重点‼️）

------

## 两层并发优化

```text
ConcurrentHashMap → 解决 key 竞争
LongAdder         → 解决 value 竞争
```

------

## 最终效果

```text
多 key → 不冲突
同 key → 低冲突
```

------

# 十、@Contended 的作用（补充）

------

## 用于解决：

```text
伪共享（false sharing）
```

------

## 示例

```java
@Contended
volatile long x;
```

------

## 但注意：

> ❗不能解决 CAS 高竞争（真共享）

------

# 十一、JVM 是否会自动优化？

------

## ❌ 不会：

```text
不会自动拆变量
不会自动分散写
不会自动避免 CAS 冲突
```

------

## ✅ 原因：

```text
1. 不知道是否有竞争
2. 会改变语义
3. 成本太高
```

------

# 十二、最终总结（面试版）

------

## 1️⃣ CAS 性能问题本质

> 多线程竞争同一个 cache line，导致缓存一致性开销巨大

------

## 2️⃣ LongAdder 优化本质

> 将单点 CAS 拆分为多个独立变量上的 CAS，避免竞争

------

## 3️⃣ computeIfAbsent 作用

> 保证并发下 value 只初始化一次

------

## 4️⃣ 最佳实践

```java
map.computeIfAbsent(key, k -> new LongAdder())
   .increment();
```

------

# 十三、终极一句话总结

> ❗并发优化的本质不是让线程更快，而是让线程不要竞争同一个资源