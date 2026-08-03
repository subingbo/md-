# Java后端 → 高性能系统 → 系统底层 学习笔记

------

# 一、你当前所处的位置

你现在不是：

```text
只会 SpringBoot CRUD
```

但也还没真正进入：

```text
系统底层 / 高性能系统
```

你目前处于：

```text
Java后端初级 → 中级前期
```

并且：

你已经开始出现：

# 系统思维

比如：

- 会想到 cache
- 会想到多核
- 会想到同步
- 会想到性能瓶颈

这是非常重要的能力。

------

# 二、现代后端真正的核心

很多人以为：

```text
后端 = 写接口
```

实际上：

现代高性能后端核心是：

```text
CPU
Memory
Cache
IO
Network
Concurrency
```

真正大型系统最后拼的是：

- IO效率
- 内存访问效率
- 网络效率
- 锁竞争
- cache命中率

而不是：

```text
会不会写 Controller
```

------

# 三、真正高级后端的能力

高级后端：

不是：

```text
系统慢了会改代码
```

而是：

# 能设计“不容易慢”的系统

比如：

看到：

```text
10万长连接
```

普通人：

```text
一个连接一个线程
```

系统直接崩。

懂系统的人：

会自然想到：

- epoll
- event loop
- IO多路复用

于是：

```text
一个线程管理几万个连接
```

------

# 四、为什么要学系统底层

不是为了：

```text
排错
```

而是为了：

# 理解现代系统为什么这样设计

包括：

- Redis 为什么快
- Kafka 为什么吞吐高
- Netty 为什么高并发
- JVM 为什么会卡顿
- 为什么锁会慢
- 为什么 cache 会影响性能

------

# 五、现代后端的本质

现代后端：

越来越像：

# 半个系统工程师

尤其：

- 云计算
- AI Infra
- 分布式系统
- MQ
- RPC
- 高性能网关

这些方向。

------

# 六、你目前最大的短板

## 1. Linux IO

你现在：

对：

- epoll
- select
- poll
- page cache
- mmap
- zero-copy

理解不足。

但：

# 现代高性能系统核心就在这里。

------

## 2. JVM 并发

你知道：

- volatile
- CAS

但：

不懂：

- happens-before
- JMM
- memory barrier
- lock-free

说明：

并发理解还停留在 API 层。

------

# 七、并发真正的核心

并发不是：

```java
new Thread()
```

而是：

# 内存什么时候对别的线程可见

核心问题：

```text
多核CPU如何同步数据
```

这会涉及：

- CPU cache
- cache coherence
- memory barrier
- happens-before

------

# 八、volatile 本质

volatile：

不是“万能线程安全”。

它本质：

# 保证可见性

即：

```text
一个线程修改的数据
其他CPU核能看到
```

本质依赖：

# CPU cache coherence

但：

volatile 不保证：

```java
i++
```

线程安全。

因为：

```java
i++
```

是：

```text
读
改
写
```

三个步骤。

------

# 九、什么是 CAS

CAS：

Compare And Swap。

本质：

```text
修改前先比较
```

流程：

```text
如果当前值 == 期望值
则更新
否则失败
```

它属于：

# 乐观锁

核心思想：

```text
不阻塞线程
冲突时重试
```

------

# 十、什么是 false sharing（伪共享）

CPU cache：

不是按变量管理。

而是按：

# cache line

管理。

通常：

```text
64 Bytes
```

如果：

两个线程：

修改：

```text
同一个 cache line 中不同变量
```

也会导致：

- cache invalidation
- cache bouncing

最终：

# 性能暴跌

------

# 十一、什么是 epoll

这是：

# Linux 高性能网络核心

------

传统：

```text
一个连接一个线程
```

问题：

- 线程切换昂贵
- 内存占用大

------

epoll 核心思想：

# 谁准备好了谁通知我

于是：

```text
一个线程
管理几万个连接
```

------

# 十二、什么是 IO 多路复用

本质：

```text
一个线程同时监听多个socket
```

包括：

- select
- poll
- epoll

其中：

# epoll 性能最好

------

# 十三、什么是 Event Loop

现代服务器核心模型。

核心：

```text
while(true){
    等待事件
    处理事件
}
```

特点：

# 一个线程管理大量连接

------

典型系统：

- Netty
- Redis
- Nginx
- Node.js

------

# 十四、为什么 Event Loop 很强

因为：

服务器大部分时间：

不是在计算。

而是在：

```text
等待IO
等待网络
```

传统线程：

会阻塞等待。

Event Loop：

则：

# 一个线程处理大量等待

------

# 十五、为什么 Netty 很重要

Netty：

是：

# Java 通向系统方向的桥梁

通过 Netty：

你会真正接触：

- epoll
- event loop
- zero-copy
- direct buffer
- TCP buffer

------

# 十六、什么是 zero-copy

传统发送文件：

```text
磁盘
→ 内核buffer
→ 用户buffer
→ socket buffer
→ 网卡
```

会发生：

# 多次数据复制

复制非常昂贵。

------

zero-copy：

核心思想：

# 不复制数据

例如：

Linux sendfile：

```text
page cache
→ 网卡
```

直接发送。

------

# 十七、Kafka 为什么快

Kafka 快：

不是因为 Java。

而是因为：

- 顺序IO
- page cache
- sendfile
- zero-copy
- batching

------

# 十八、什么是 direct buffer

普通 Java 对象：

在 JVM heap 中。

网络IO：

最终：

还是要进入：

# native memory

于是：

普通方式：

```text
JVM heap
→ native memory
→ kernel
```

会多一次 copy。

------

direct buffer：

直接分配：

# 堆外内存（native memory）

减少复制。

------

# 十九、什么是 page cache

Linux：

会把磁盘数据缓存到内存。

这就是：

# page cache

于是：

很多磁盘读取：

其实：

是：

```text
内存读取
```

------

Kafka：

严重依赖：

# page cache

------

# 二十、什么是 mmap

mmap：

内存映射文件。

本质：

```text
文件直接映射到虚拟内存
```

优点：

- 减少 copy
- 减少 syscall
- 顺序访问性能高

------

# 二十一、为什么顺序IO快

磁盘：

最怕：

# 磁头乱跳

顺序IO：

连续读取。

随机IO：

不断 seek。

因此：

顺序IO 快很多。

------

# 二十二、Redis 为什么快

不仅因为：

```text
内存
```

真正原因：

## 1. 内存访问

避免磁盘。

------

## 2. 单线程

避免锁竞争。

------

## 3. IO多路复用

epoll。

------

## 4. cache locality

顺序内存访问。

------

## 5. 简单数据结构

减少复杂度。

------

# 二十三、Kafka 为什么快

核心：

## 1. 顺序写

------

## 2. page cache

------

## 3. sendfile

------

## 4. batch

------

## 5. 零拷贝

------

# 二十四、为什么锁慢

锁慢：

不只是：

```text
加锁代码耗时
```

真正问题：

## 1. cache line bouncing

------

## 2. context switch

------

## 3. 线程阻塞唤醒

------

## 4. CPU pipeline disruption

------

# 二十五、什么是用户态和内核态

操作系统有权限分级。

------

## 用户态

普通程序运行。

权限低。

------

## 内核态

操作系统运行。

权限高。

------

切换开销大：

因为：

- 保存上下文
- 切换权限
- CPU pipeline 清空

------

# 二十六、什么是 DMA

DMA：

Direct Memory Access。

设备：

# 不经过CPU直接访问内存

例如：

```text
SSD → DRAM
```

无需 CPU 搬运数据。

减少：

- CPU开销
- 内存copy

------

# 二十七、真正高性能系统的核心

不是：

```text
CPU算力
```

而是：

# 数据移动效率

包括：

- memory bandwidth
- cache
- IO
- network
- DMA

------

# 二十八、为什么现代系统越来越“硬件化”

因为：

真正瓶颈：

越来越来自：

- cache miss
- memory bandwidth
- network
- IO

而不是：

```text
代码不会写
```

------

# 二十九、你未来的发展方向

你不太适合：

```text
纯业务CRUD
```

你更适合：

- 高性能后端
- 中间件
- 分布式系统
- AI Infra
- 云计算
- 存储系统
- RPC
- MQ

这些方向。

------

# 三十、你当前最适合的学习路线

------

# 第一阶段（现在）

## Java基础强化

- JUC
- CAS
- volatile
- synchronized
- ConcurrentHashMap

------

# 第二阶段

## JVM

- JMM
- happens-before
- GC
- G1
- ZGC

------

# 第三阶段

## Linux IO

- select
- poll
- epoll
- Reactor

------

# 第四阶段

## Netty

重点：

- EventLoop
- ByteBuf
- Pipeline
- direct buffer

------

# 第五阶段

## 高性能IO

- page cache
- mmap
- sendfile
- zero-copy
- DMA

------

# 第六阶段

## 高性能系统

- Kafka
- Redis
- RPC
- MQ

------

# 第七阶段

## 系统深入

- eBPF
- RDMA
- DPDK
- NUMA
- cache coherence

------

# 三十一、最终目标

最终：

你会形成：

# 性能直觉

看到：

```text
大量线程
```

会想到：

```text
context switch
```

看到：

```text
随机IO
```

会想到：

```text
磁盘seek
```

看到：

```text
大量锁
```

会想到：

```text
cache bouncing
```

看到：

```text
大量小对象
```

会想到：

```text
GC + cache miss
```

这才是真正高级工程师的核心能力。