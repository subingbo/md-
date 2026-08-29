# Java 本地缓存与 Caffeine

## 1. 什么是本地缓存

本地缓存（Local Cache）：

> 将热点数据直接存储在当前 Java 应用的 JVM 内存中，减少 Redis、MySQL 等外部资源的访问。

基本结构：

```text
请求
 ↓
Java 应用
 ↓
本地缓存
 ↓ 命中
直接返回

 ↓ 未命中
Redis / MySQL
 ↓
写入本地缓存
 ↓
返回
```

特点：

- 数据存储在 JVM 内存
- 不需要网络通信
- 访问速度非常快
- 每个 JVM 都有自己的缓存
- 受 JVM 内存大小限制
- 多实例部署时存在数据不一致问题

------

# 2. 为什么需要本地缓存

假设：

```text
Java 应用 → Redis
```

即使 Redis 很快，也需要经过：

```text
网络通信
 ↓
序列化
 ↓
Redis 查询
 ↓
反序列化
```

而本地缓存：

```text
Java 应用 → JVM 内存
```

直接进行内存访问。

因此：

```text
本地缓存
    ↓
访问速度最快

Redis
    ↓
速度很快 + 多实例共享

MySQL
    ↓
最终数据源
```

------

# 3. HashMap 能不能做本地缓存？

可以。

最简单的实现：

```java
Map<Long, User> cache = new HashMap<>();

public User getUser(Long id) {
    User user = cache.get(id);

    if (user == null) {
        user = userMapper.selectById(id);
        cache.put(id, user);
    }

    return user;
}
```

本质就是：

```text
key → value
```

例如：

```text
1 → 张三
2 → 李四
3 → 王五
```

但是 `HashMap` 只是一个基础数据结构，并不是专业的缓存组件。

------

# 4. 为什么不直接使用 HashMap 做生产缓存？

## 4.1 没有过期机制

```java
cache.put(1L, user);
```

数据会一直存在。

除非主动：

```java
cache.remove(1L);
```

否则：

```text
10:00 → 存入
11:00 → 还在
第二天 → 还在
一个月后 → 还在
```

------

## 4.2 没有容量限制

如果不断写入：

```text
1
2
3
...
1000000
...
```

HashMap 会不断增长。

最终可能导致：

```text
缓存不断增长
 ↓
JVM Heap 增大
 ↓
GC 压力增加
 ↓
Full GC
 ↓
甚至 OOM
```

因此缓存通常需要：

> 最大容量 + 淘汰策略

------

## 4.3 HashMap 不是线程安全的

Spring Boot 是多线程环境。

可能同时存在：

```text
Thread-1 → cache.get()
Thread-2 → cache.put()
Thread-3 → cache.remove()
```

HashMap 本身不保证并发安全。

------

# 5. ConcurrentHashMap

如果只是解决线程安全问题，可以使用：

```java
ConcurrentHashMap<Long, User> cache =
        new ConcurrentHashMap<>();
```

它解决了：

```text
HashMap
   ↓
线程安全问题
```

但是仍然没有完整解决：

```text
过期
最大容量
淘汰策略
缓存统计
自动加载
```

所以：

```text
HashMap
    ↓
基础 Map

ConcurrentHashMap
    ↓
线程安全 Map

Caffeine
    ↓
专业本地缓存
```

------

# 6. Caffeine 是什么？

Caffeine 是 Java 生态中非常常用的**高性能本地缓存框架**。

可以理解为：

> 在 JVM 内存中提供完整缓存能力的组件。

它提供：

- 缓存存取
- 最大容量
- 过期时间
- 淘汰策略
- 自动加载
- 缓存刷新
- 缓存统计
- 并发控制
- 异步缓存等能力

------

# 7. Caffeine 基本使用

创建缓存：

```java
Cache<Long, User> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build();
```

写入：

```java
cache.put(1L, user);
```

查询：

```java
User user = cache.getIfPresent(1L);
```

删除：

```java
cache.invalidate(1L);
```

清空：

```java
cache.invalidateAll();
```

------

# 8. Caffeine 常见配置

## 8.1 maximumSize

```java
.maximumSize(10_000)
```

表示缓存最多维护大约 10000 个条目。

例如：

```text
缓存容量 = 3

A
B
C
```

继续加入：

```text
D
```

缓存容量超过限制，就必须淘汰部分数据：

```text
A B C
 ↓
淘汰部分数据
 ↓
A C D
```

具体淘汰谁由 Caffeine 的淘汰策略决定。

------

# 9. 缓存淘汰

缓存容量有限：

```text
内存有限
 ↓
缓存不能无限增长
 ↓
必须淘汰数据
```

常见淘汰策略：

### LRU

Least Recently Used

> 最近最少使用

例如：

```text
A B C
```

访问：

```text
A
```

那么 A 变成最近使用的数据。

如果需要淘汰：

```text
B
```

可能优先被淘汰。

------

### LFU

Least Frequently Used

> 最不经常使用

例如：

```text
A → 100 次
B → 50 次
C → 1 次
```

C 的访问频率最低，更容易被淘汰。

------

# 10. Caffeine 的 Window TinyLFU

Caffeine 的核心淘汰策略之一：

> **Window TinyLFU**

它不是简单的 LRU，也不是简单的 LFU。

可以粗略理解为：

```text
最近访问情况
       +
历史访问频率
       ↓
综合判断数据是否应该继续留在缓存
```

这样能够更好地适应实际业务中的缓存访问模式。

面试可以回答：

> Caffeine 使用 Window TinyLFU 等机制进行缓存淘汰，在近期访问情况和历史访问频率之间进行平衡，相比简单的 LRU 等策略能够获得更好的缓存命中效果。

------

# 11. Caffeine 的过期机制

最常见的三个 API：

```java
expireAfterWrite()
expireAfterAccess()
refreshAfterWrite()
```

------

## 11.1 expireAfterWrite

```java
.expireAfterWrite(10, TimeUnit.MINUTES)
```

含义：

> 数据写入缓存后 10 分钟过期。

例如：

```text
10:00 → put(A)
```

那么：

```text
10:10 → A 过期
```

即使：

```text
10:05 → get(A)
```

也不会重新计算过期时间。

------

## 11.2 expireAfterAccess

```java
.expireAfterAccess(10, TimeUnit.MINUTES)
```

含义：

> 如果连续 10 分钟没有访问该数据，就过期。

例如：

```text
10:00 → put(A)
10:05 → get(A)
10:12 → get(A)
10:20 → get(A)
```

每次访问都会重新计算过期时间。

所以：

```text
持续访问
 ↓
一直活跃

长时间不访问
 ↓
过期
```

适合：

> 热点数据缓存。

------

# 12. expireAfterWrite vs expireAfterAccess

|                  | expireAfterWrite | expireAfterAccess   |
| ---------------- | ---------------- | ------------------- |
| 计时起点         | 写入             | 最后一次访问        |
| 访问是否延长时间 | ❌                | ✅                   |
| 适合             | 固定时间刷新     | 热点数据            |
| 示例             | 10 分钟后过期    | 10 分钟没访问才过期 |

记忆：

```text
Write
 ↓
从写入开始计时

Access
 ↓
从最后一次访问开始计时
```

------

# 13. refreshAfterWrite

```java
.refreshAfterWrite(10, TimeUnit.MINUTES)
```

它和 `expireAfterWrite` 不一样。

### expire

```text
达到时间
 ↓
数据失效
 ↓
缓存没有数据
```

### refresh

```text
达到刷新时间
 ↓
重新加载数据
 ↓
更新缓存
```

可以粗略理解为：

```text
expire = 过期

refresh = 刷新
```

refresh 更适合：

> 希望缓存数据定期更新，同时尽量减少缓存突然失效带来的影响。

------

# 14. Caffeine 自动加载

普通写法：

```java
User user = cache.getIfPresent(id);

if (user == null) {
    user = userMapper.selectById(id);
    cache.put(id, user);
}
```

Caffeine 可以使用 `LoadingCache`：

```java
LoadingCache<Long, User> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build(id -> userMapper.selectById(id));
```

之后：

```java
User user = cache.get(id);
```

内部逻辑：

```text
cache.get(id)
      ↓
检查缓存
   ↙       ↘
命中       未命中
 ↓           ↓
返回      执行 loader
             ↓
       查询数据库
             ↓
       写入缓存
             ↓
           返回
```

所以 Caffeine 可以帮助实现：

> Cache Aside 的读流程。

------

# 15. Cache Aside

经典缓存旁路模式：

```text
读取：

请求
 ↓
查询缓存
 ↓
命中 → 返回

未命中
 ↓
查询数据库
 ↓
写入缓存
 ↓
返回
```

代码：

```java
User user = cache.getIfPresent(id);

if (user == null) {
    user = userMapper.selectById(id);
    cache.put(id, user);
}

return user;
```

或者通过 `LoadingCache` 简化。

------

# 16. Caffeine 与 Redis 的区别

|            | Caffeine   | Redis          |
| ---------- | ---------- | -------------- |
| 类型       | 本地缓存   | 分布式缓存     |
| 存储位置   | JVM 内存   | Redis 服务器   |
| 网络       | 不需要     | 需要           |
| 速度       | 更快       | 很快           |
| 多实例共享 | ❌          | ✅              |
| 内存来源   | JVM Heap   | Redis          |
| 容量       | 相对有限   | 更大           |
| 数据一致性 | 较难处理   | 更容易统一     |
| 典型用途   | 极热点数据 | 分布式共享缓存 |

------

# 17. 为什么不能只使用 Caffeine？

因为微服务通常有多个实例：

```text
             Nginx
               ↓
       ┌───────┼───────┐
       ↓       ↓       ↓
     JVM1     JVM2    JVM3
       ↓       ↓       ↓
  Caffeine  Caffeine Caffeine
```

三个 JVM 的缓存完全独立：

```text
JVM1 → 张三
JVM2 → 张三
JVM3 → 张三
```

如果 JVM1 修改：

```text
张三 → 李四
```

可能变成：

```text
JVM1 → 李四
JVM2 → 张三
JVM3 → 张三
```

这就是：

> **本地缓存不共享导致的数据一致性问题。**

------

# 18. Caffeine + Redis 二级缓存

生产环境常见：

```text
                 请求
                   ↓
          ┌────────────────┐
          │    Caffeine    │
          │   L1 本地缓存   │
          └────────────────┘
                   ↓ miss
          ┌────────────────┐
          │     Redis      │
          │   L2 分布式缓存 │
          └────────────────┘
                   ↓ miss
          ┌────────────────┐
          │     MySQL      │
          └────────────────┘
```

读取流程：

```text
请求
 ↓
Caffeine
 ↓
命中 → 返回

未命中
 ↓
Redis
 ↓
命中 → 写入 Caffeine → 返回

未命中
 ↓
MySQL
 ↓
写 Redis
 ↓
写 Caffeine
 ↓
返回
```

------

# 19. 为什么使用两级缓存？

### Caffeine

优势：

```text
不经过网络
 ↓
访问速度极快
```

缺点：

```text
每个 JVM 独立
 ↓
无法共享
```

### Redis

优势：

```text
多个 JVM
 ↓
共享 Redis
```

缺点：

```text
需要网络通信
```

所以：

```text
Caffeine
 ↓
解决访问速度

Redis
 ↓
解决分布式共享
```

------

# 20. 二级缓存的一致性问题

例如初始：

```text
MySQL    = 张三
Redis    = 张三
JVM1     = 张三
JVM2     = 张三
JVM3     = 张三
```

修改数据库：

```text
MySQL = 李四
```

如果没有同步缓存：

```text
MySQL    = 李四
Redis    = 张三
JVM1     = 张三
JVM2     = 张三
JVM3     = 张三
```

就会出现：

> **脏缓存。**

------

# 21. 如何解决本地缓存一致性？

一种典型方案：

```text
修改数据
   ↓
MySQL
   ↓
删除 Redis
   ↓
发送缓存失效消息
   ↓
MQ / Redis PubSub
   ↓
通知所有 JVM
   ↓
各 JVM 删除 Caffeine
```

最终：

```text
JVM1 Caffeine → 删除
JVM2 Caffeine → 删除
JVM3 Caffeine → 删除
Redis         → 删除
```

下一次查询：

```text
Caffeine miss
 ↓
Redis miss
 ↓
MySQL
 ↓
重新加载
```

------

# 22. Caffeine 与 Spring Cache

Spring 本身提供了缓存抽象：

```java
@Cacheable
@CacheEvict
@CachePut
```

例如：

```java
@Cacheable(value = "user", key = "#id")
public User getUser(Long id) {
    return userMapper.selectById(id);
}
```

调用：

```java
getUser(1L);
```

第一次：

```text
缓存没有
 ↓
执行方法
 ↓
查询数据库
 ↓
缓存结果
```

第二次：

```text
查询缓存
 ↓
命中
 ↓
不执行数据库查询
```

Spring Cache 可以接入：

```text
Caffeine
Redis
其他缓存实现
```

因此：

```text
Spring Cache
     ↓
缓存抽象层

Caffeine
     ↓
具体缓存实现
```

------

# 23. HashMap、ConcurrentHashMap、Caffeine 的关系

不要理解成：

```text
Caffeine 出现了
 ↓
HashMap 没用了
```

正确理解：

```text
Java 集合
   │
   ├── HashMap
   │     ↓
   │   普通 Map
   │
   └── ConcurrentHashMap
         ↓
       并发 Map


缓存框架
   │
   └── Caffeine
         ↓
       专业本地缓存
```

------

# 24. 三者对比

| 能力           | HashMap | ConcurrentHashMap | Caffeine |
| -------------- | ------- | ----------------- | -------- |
| Key-Value      | ✅       | ✅                 | ✅        |
| 线程安全       | ❌       | ✅                 | ✅        |
| 过期           | ❌       | ❌                 | ✅        |
| 最大容量       | ❌       | ❌                 | ✅        |
| 淘汰策略       | ❌       | ❌                 | ✅        |
| 自动加载       | ❌       | ❌                 | ✅        |
| 刷新           | ❌       | ❌                 | ✅        |
| 缓存统计       | ❌       | ❌                 | ✅        |
| 专门为缓存设计 | ❌       | ❌                 | ✅        |

因此：

> **HashMap 是数据结构，ConcurrentHashMap 是并发容器，Caffeine 是缓存组件。**

------

# 25. 一张图记住整个知识体系

```text
                    Java 本地缓存
                         │
          ┌──────────────┴──────────────┐
          ↓                             ↓
      HashMap                    ConcurrentHashMap
          │                             │
      基础 Map                       并发 Map
          │                             │
          └──────────────┬──────────────┘
                         ↓
                    Caffeine
                         │
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
    过期机制           淘汰机制           自动加载
       │                 │                 │
       ↓                 ↓                 ↓
 expireWrite       Window TinyLFU     LoadingCache
 expireAccess
 refresh
                         │
                         ↓
                   多级缓存
                         │
                ┌────────┴────────┐
                ↓                 ↓
             Caffeine           Redis
               L1                 L2
                │                 │
                └────────┬────────┘
                         ↓
                       MySQL
                         │
                         ↓
                    一致性问题
                         │
                         ↓
                   MQ / PubSub
                         │
                         ↓
                  缓存失效通知
```

------

# 26. 面试重点

## 问：什么是本地缓存？

答：

> 本地缓存是将数据存储在当前 JVM 内存中，减少对 Redis、数据库等外部资源的访问。它不需要网络通信，因此访问速度非常快，但在分布式部署场景下不同实例之间缓存不共享。

------

## 问：为什么不用 HashMap 做缓存？

答：

> HashMap 可以实现最简单的本地缓存，但它本身不具备线程安全、过期、容量限制和淘汰等缓存能力。生产环境通常使用 Caffeine 这样的专业缓存框架。

------

## 问：ConcurrentHashMap 能不能做缓存？

答：

> 可以实现简单缓存，但它主要解决的是并发访问问题，本身没有完整的过期、淘汰、自动加载等缓存能力，所以复杂缓存场景通常使用 Caffeine。

------

## 问：Caffeine 为什么快？

可以从几个角度回答：

```text
JVM 本地内存
 ↓
不需要网络通信

高效的数据结构
 ↓
快速读写

针对并发场景优化
 ↓
降低锁竞争

高效的淘汰策略
 ↓
Window TinyLFU
```

------

## 问：Caffeine 有哪些过期策略？

重点记：

```java
expireAfterWrite()
expireAfterAccess()
refreshAfterWrite()
```

分别理解成：

```text
Write   → 写入后过期
Access  → 长时间不访问后过期
Refresh → 定期刷新
```

------

## 问：为什么使用 Caffeine + Redis？

> Caffeine 作为 JVM 内的 L1 缓存，可以获得更低的访问延迟；Redis 作为 L2 分布式缓存，可以让多个服务实例共享数据。两者结合能够同时兼顾访问性能和分布式场景。

------

## 问：Caffeine + Redis 最大的问题是什么？

重点回答：

> **缓存一致性。**

因为：

```text
JVM1 Caffeine
JVM2 Caffeine
JVM3 Caffeine
Redis
MySQL
```

存在多个数据副本。

因此需要考虑：

```text
缓存更新
缓存删除
失效通知
MQ
Redis Pub/Sub
双删策略
延迟双删
```

------

# 27. 最终记忆版

只需要先牢牢记住这 8 句话：

```text
1. HashMap 是基础 Map，不是专业缓存。

2. ConcurrentHashMap 解决并发问题，但不是完整缓存方案。

3. Caffeine 是 Java 常用的高性能本地缓存框架。

4. Caffeine 的核心能力包括过期、淘汰、容量限制、自动加载。

5. expireAfterWrite = 写入后过期。

6. expireAfterAccess = 一段时间不访问才过期。

7. Caffeine 使用 Window TinyLFU 等机制进行缓存淘汰。

8. 分布式系统中经常使用 Caffeine + Redis，
   但需要额外解决多实例缓存一致性问题。
```

**学习顺序建议：**

```text
HashMap
  ↓
ConcurrentHashMap
  ↓
Caffeine 基础 API
  ↓
expireAfterWrite / Access
  ↓
LoadingCache
  ↓
LRU / LFU
  ↓
Window TinyLFU
  ↓
Spring Cache
  ↓
Caffeine + Redis 二级缓存
  ↓
缓存一致性
  ↓
缓存穿透 / 击穿 / 雪崩
```

这条线学完后，就能把你之前学的 **Redis + Spring Cache + 并发 + JVM 内存**串成一个完整的缓存知识体系。