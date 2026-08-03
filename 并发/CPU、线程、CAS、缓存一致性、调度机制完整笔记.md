# CPU、线程、CAS、缓存一致性、调度机制完整笔记

# 一、整体架构图

```text
用户请求
   ↓
提交任务（submit）
   ↓
线程池
   ↓
线程
   ↓
CPU调度
   ↓
CPU执行机器指令
   ↓
触发：
- cache
- CAS
- 内存屏障
- cache一致性协议
```

核心思想：

```text
线程只是“执行上下文”
真正干活的是CPU核
```

------

# 二、线程、CPU、任务的关系

------

# 1. 任务（Task）

例如：

```java
executor.submit(() -> {
    queryUser();
});
```

任务：

```text
只是“待执行工作”
```

还没真正运行。

------

# 2. 线程（Thread）

线程本质：

```text
CPU执行状态 + 上下文
```

包括：

- 寄存器
- 栈
- 程序计数器
- TLS
- task_struct

线程：

```text
不是CPU
而是CPU运行代码时的“身份”
```

------

# 3. CPU核（Core）

真正执行：

```text
机器指令
```

的是CPU核。

例如：

```text
8核CPU
≈
同一时刻最多真正并行8线程
```

------

# 三、线程池工作机制

Java ThreadPoolExecutor：

```text
核心线程
→ 队列
→ 最大线程
→ 拒绝策略
```

即：

```text
corePoolSize
→ queue
→ maximumPoolSize
→ reject
```

------

# 线程池执行流程

```text
请求进来
   ↓
核心线程还有空闲？
   ├─ 有 → 直接执行
   └─ 没有
         ↓
任务队列没满？
   ├─ 没满 → 进入队列等待
   └─ 满了
         ↓
最大线程数还没到？
   ├─ 没到 → 创建新线程执行
   └─ 到了
         ↓
触发拒绝策略
```

------

# 四、线程多为什么会卡

线程多：

```text
≠
性能高
```

因为：

CPU核有限。

例如：

```text
8核CPU
1000线程
```

意味着：

```text
只有8线程真正运行
其余线程等待调度
```

------

# 五、上下文切换（Context Switch）

本质：

```text
CPU换线程执行
```

CPU需要：

- 保存当前线程寄存器
- 保存栈状态
- 恢复另一个线程状态

------

# 真正昂贵的是：

```text
Cache失效
```

不是：

```text
保存寄存器
```

------

# 六、为什么线程多会导致大量切换

因为：

```text
CPU核数 << 线程数
```

OS只能：

```text
轮流给线程时间片
```

于是：

```text
不停切线程
```

------

# 七、调度器（Scheduler）

调度器负责：

```text
决定哪个线程在哪个CPU核运行
```

------

# Linux CFS

Linux使用：

```text
CFS（Completely Fair Scheduler）
```

核心：

```text
尽量公平分配CPU时间
```

维护：

- 红黑树
- vruntime

------

# 八、调度器是不是线程？

不是。

调度器：

```text
是OS内核代码
```

CPU会：

```text
暂停当前线程
进入内核态
执行schedule()
```

------

# 九、用户态与内核态

------

# 用户态

执行：

```text
业务代码
```

权限低。

------

# 内核态

执行：

```text
OS代码
```

例如：

- 调度
- IO
- 中断
- 内存管理

------

# 十、进入内核态后线程还存在吗

存在。

因为：

```text
线程 ≠ 用户态
```

线程是：

```text
执行上下文
```

无论：

- 用户态
- 内核态

CPU都需要知道：

```text
当前线程是谁
```

------

# 十一、CAS（Compare And Swap）

CAS本质：

```java
CAS(V, A, B)
```

意思：

```text
如果V == A
则改成B
否则失败
```

------

# 十二、CAS为什么失败

唯一原因：

```text
CAS目标值被别的线程修改
```

例如：

线程A读取：

```text
100
```

线程B先改成：

```text
101
```

线程A再CAS：

```text
CAS(value,100,101)
```

失败。

------

# 十三、CAS为什么耗CPU

因为CAS失败后：

```text
不会睡眠
```

而是：

```text
疯狂重试
```

例如：

```java
while(true){
    if(cas()){
        break;
    }
}
```

CPU一直执行：

- load
- cmp
- lock cmpxchg
- jump

------

# 十四、CAS为什么会导致CPU空转

例如：

```text
100线程CAS同一个变量
```

可能：

```text
只有1线程成功
99线程失败
```

于是：

```text
CPU大量时间都在失败重试
```

表现：

```text
CPU100%
业务推进很少
```

------

# 十五、什么是cache line

CPU cache不是按变量缓存。

而是：

```text
按cache line缓存
```

一般：

```text
64字节
```

------

# 十六、MESI缓存一致性协议

多核CPU：

每核都有自己的cache。

MESI保证：

```text
多个CPU最终看到一致数据
```

状态：

| 状态 | 含义      |
| ---- | --------- |
| M    | Modified  |
| E    | Exclusive |
| S    | Shared    |
| I    | Invalid   |

------

# 十七、cache line bouncing（缓存行抖动）

多个CPU核：

```text
频繁修改同一个cache line
```

导致：

```text
cache line在CPU核之间来回转移
```

例如：

```text
CPU1拿到写权限
→ CPU2 invalidate CPU1
→ CPU2拿到
→ CPU1再抢回来
```

------

# 十八、cache line抖动会导致什么

不会导致：

```text
没人能修改变量
```

MESI保证：

```text
总有CPU核最终获得写权限
```

但会导致：

- CAS大量失败
- cache频繁失效
- CPU大量同步cache
- coherence traffic暴涨

------

# 十九、什么是伪共享（False Sharing）

两个变量：

```text
逻辑无关
```

但：

```text
在同一个cache line
```

例如：

```java
class Data{
    volatile long a;
    volatile long b;
}
```

线程A修改a。

线程B修改b。

虽然：

```text
不是同一个变量
```

但：

```text
还是会互相invalidate cache
```

------

# 二十、伪共享会导致CAS失败吗

不会直接导致。

因为：

CAS失败唯一原因：

```text
CAS目标值变化
```

但：

伪共享会：

- 增加cache同步
- 增加cache失效
- 拉长CAS耗时

于是：

```text
间接增加CAS冲突概率
```

------

# 二十一、CPU为什么会乱序执行

因为：

```text
内存太慢
```

CPU不想等待。

例如：

```java
a = arr[i];
b = x + y;
c = m + n;
```

CPU发现：

```text
b/c不依赖a
```

于是：

```text
先算b/c
后台等待a
```

------

# 二十二、现代CPU主要优化

------

# 1. Out-of-Order Execution

乱序执行。

------

# 2. Speculative Execution

推测执行。

------

# 3. Pipeline

流水线。

------

# 4. Store Buffer

延迟写内存。

------

# 5. Prefetch

提前读内存。

------

# 二十三、为什么CAS/锁会限制CPU优化

因为：

```text
多线程共享数据
必须保证一致性
```

CPU不能再：

```text
随便乱序
```

------

# 二十四、什么叫“可并行空间减少”

不是：

```text
完全不能并行
```

而是：

```text
CPU原本能自由优化的范围变小
```

例如：

------

# 普通代码

CPU：

```text
可以大胆乱序
```

------

# CAS/锁

CPU：

```text
必须严格同步
```

因此：

- 乱序减少
- speculation减少
- store buffer flush
- pipeline stall

------

# 二十五、memory barrier（内存屏障）

作用：

```text
强制CPU保证执行顺序
```

例如：

```text
前面的必须真正完成
后面的才能开始
```

会导致：

- pipeline stall
- reorder buffer等待
- store buffer flush

------

# 二十六、为什么cache同步也会“占CPU”

虽然：

```text
不是软件代码
```

但：

```text
会占用CPU内部硬件资源
```

例如：

- pipeline
- cache subsystem
- execution units

------

# 二十七、CPU为什么会stall

例如：

CAS等待cache line同步。

CPU：

```text
不能继续推进后续依赖指令
```

于是：

```text
pipeline stall
```

表现：

```text
CPU很忙
但业务推进少
```

------

# 二十八、真正的高并发瓶颈

通常不是：

```text
CPU计算不够
```

而是：

- cache miss
- context switch
- lock contention
- cache coherence
- memory stall

------

# 二十九、为什么高并发共享写特别贵

因为：

每次共享写：

都会：

```text
限制CPU自由优化空间
```

并导致：

- cache line bouncing
- memory barrier
- coherence traffic

------

# 三十、真正高性能系统的目标

核心：

```text
减少共享写
减少同步
减少cache竞争
```

例如：

- LongAdder
- Disruptor
- Netty
- Reactor

本质都是：

```text
减少CPU同步成本
```

------

# 三十一、最终核心总结

------

# 1. 真正执行代码的是CPU核

线程只是：

```text
执行上下文
```

------

# 2. 高并发性能瓶颈很多来自：

- cache一致性
- context switch
- lock contention
- memory stall

不是纯计算。

------

# 3. CAS不是免费

低竞争：

```text
非常快
```

高竞争：

```text
CPU空转严重
```

------

# 4. cache line bouncing本质

```text
多个CPU核频繁争夺cache line写权限
```

------

# 5. 现代CPU为了性能会：

- 乱序执行
- 推测执行
- pipeline并行

------

# 6. 锁/CAS/volatile本质

```text
限制CPU自由优化
以换取多线程一致性
```

------

# 7. 高性能系统核心思想

```text
减少共享数据竞争
减少同步
减少cache同步
```