# Redis 分布式锁

## 1. 为什么需要分布式锁？

在分布式系统中，同一个资源可能同时被多个服务实例操作。

例如库存：

```text
库存 = 1

服务A ──┐
        ├──→ 修改库存
服务B ──┘
```

如果没有锁：

```text
A：读取库存 = 1
B：读取库存 = 1

A：库存 - 1
B：库存 - 1
```

可能导致超卖。

分布式锁的目的：

```text
多个服务实例
      ↓
竞争同一把锁
      ↓
只有获得锁的实例
      ↓
才能操作共享资源
```

------

# 2. Redis 实现分布式锁的基本方式

最常见的方式：

```bash
SET lock_key unique_value NX PX 10000
```

例如：

```bash
SET order:123:lock abc123 NX PX 10000
```

含义：

| 参数           | 含义                  |
| -------------- | --------------------- |
| `SET`          | 设置 Redis Key        |
| `lock_key`     | 锁的 Key              |
| `unique_value` | 当前客户端唯一标识    |
| `NX`           | Key 不存在时才能设置  |
| `PX 10000`     | 锁 10000ms 后自动过期 |

------

# 3. NX：保证互斥

假设：

```text
Redis 中不存在：

order:123:lock
```

服务 A：

```bash
SET order:123:lock A NX PX 10000
```

成功：

```text
order:123:lock = A
```

A 获得锁。

此时服务 B：

```bash
SET order:123:lock B NX PX 10000
```

因为 Key 已经存在：

```text
SET NX → 失败
```

所以：

```text
A → 获得锁
B → 获取失败
```

## NX 的作用

> `NX` 保证同一时间只有一个客户端能够成功创建这把锁。

------

# 4. PX：防止死锁

假设没有过期时间：

```text
A
 ↓
获得锁
 ↓
突然宕机
```

那么：

```text
Redis：

lock = A

一直存在
```

其他客户端永远无法获得锁。

这就可能形成死锁。

所以：

```bash
SET lock A NX PX 10000
```

表示：

```text
A 获得锁
    ↓
最多存在 10 秒
    ↓
A 正常释放 → 提前删除
    ↓
A 宕机 → 10 秒后自动过期
```

因此：

> `PX` 的作用是给锁设置租约时间，防止客户端宕机后锁永久存在。

------

# 5. 为什么 value 必须是唯一值？

不能简单写：

```text
SET lock 1 NX PX 10000
```

应该使用：

```text
SET lock UUID NX PX 10000
```

例如：

```text
A：

lock = abc123
B：

lock = xyz789
```

因为释放锁的时候必须知道：

> 当前这把锁是不是我自己的？

------

# 6. 为什么不能直接 DEL？

错误：

```bash
DEL lock
```

存在一个非常严重的问题。

假设：

```text
A 获得锁
TTL = 10 秒
```

但是 A 执行业务很慢：

```text
0秒：A 获得锁
...
10秒：锁自动过期
```

此时 B：

```text
B 获得锁
lock = B
```

然后 A 的业务终于执行完：

```text
A → DEL lock
```

结果：

```text
B 的锁被 A 删除了！
```

所以不能直接：

```text
DEL lock
```

------

# 7. 正确释放锁：判断 value + 删除

正确逻辑：

```text
如果 Redis 中：

lock.value == 我的 unique_value

那么：

删除 lock
```

例如：

```text
Redis：

lock = B
```

A 尝试释放：

```text
我的 value = A
Redis value = B
```

不相等：

```text
不能删除
```

这样可以避免 A 删除 B 的锁。

------

# 8. 为什么需要 Lua？

问题在于：

```text
GET
 ↓
判断
 ↓
DEL
```

这是多个操作。

如果分开执行，可能发生：

```text
A：

GET lock
 ↓
发现 value = A

        ↓
        锁过期

        ↓
        B 获得锁
        lock = B

A：

DEL lock
 ↓
把 B 的锁删除
```

所以必须保证：

```text
判断 + 删除
```

是一个原子操作。

Lua 可以让 Redis 一次性执行：

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end

return 0
```

逻辑：

```text
GET lock
   ↓
判断是不是自己的 value
   ↓
   ┌───────┴───────┐
   ↓               ↓
  是               否
   ↓               ↓
 DEL              什么都不做
```

------

# 9. SET NX PX + Lua 总结

加锁：

```bash
SET lock_key unique_value NX PX 10000
```

释放锁：

```text
Lua：

if value == 我的value:
    DEL lock
```

完整流程：

```text
客户端
  ↓
SET NX PX
  ↓
获得锁
  ↓
执行业务
  ↓
Lua 判断 value
  ↓
是自己的锁？
  ├── 是 → 删除
  └── 否 → 不删除
```

------

# 10. 单 Redis 分布式锁的问题

如果只有一个 Redis：

```text
客户端
   ↓
Redis
```

假设：

```text
A → Redis
  → 获得锁
```

然后 Redis 突然故障：

```text
Redis ❌
```

如果发生故障转移，而且新的 Redis 节点没有及时拥有原来的锁状态：

```text
新的 Redis
   ↓
发现没有 lock
```

此时 B：

```text
B → 新 Redis
  → 获得锁
```

于是可能出现：

```text
A：认为自己持有锁
B：也认为自己持有锁
```

这会破坏分布式锁最重要的：

> 互斥性。

因此产生了 RedLock。

------

# 11. RedLock 是什么？

RedLock 是 Redis 作者提出的一种基于多个独立 Redis 实例实现分布式锁的方法。

例如部署 5 个 Redis：

```text
R1
R2
R3
R4
R5
```

客户端同时尝试在多个 Redis 上加锁：

```text
客户端 A
  │
  ├──→ R1
  ├──→ R2
  ├──→ R3
  ├──→ R4
  └──→ R5
```

只有获得多数 Redis 实例的锁，才认为自己获得了锁。

------

# 12. RedLock 的多数派

5 个 Redis：

```text
R1 R2 R3 R4 R5
```

多数派：

```text
5 / 2 + 1 = 3
```

因此至少需要：

```text
3 个 Redis 加锁成功
```

例如：

```text
R1 ✓
R2 ✓
R3 ✓
R4 ✗
R5 ✗
```

成功：

```text
3 / 5
```

满足多数派：

```text
获得 RedLock
```

------

# 13. 只有两个 Redis 成功怎么办？

例如：

```text
R1 ✓
R2 ✓
R3 ?
R4 ✗
R5 ✗
```

如果 R3 一直没有返回：

```text
最终确认成功的只有：

2 / 5
```

而：

```text
2 < 3
```

所以：

> 不能认为获得了 RedLock。

即使 R3 实际上已经加锁成功，但是响应因为网络问题没有返回：

```text
R3：

实际上：
lock = A

客户端：

没有收到响应
→ 当作失败处理
```

客户端仍然只能确认：

```text
2 个成功
```

所以：

> 不能认为自己获得了锁。

------

# 14. Redis 请求超时 timeout

这里非常重要。

访问一个 Redis 实例时，不能无限等待。

例如：

```text
R1 → 正常
R2 → 正常
R3 → 挂了
R4 → 正常
R5 → 正常
```

如果访问 R3：

```text
A → R3
   ↓
一直等待
   ↓
10秒
   ↓
30秒
   ↓
一直等
```

整个 RedLock 获取过程就会被拖慢。

因此需要给单个 Redis 请求设置较短的 timeout。

例如：

```text
单 Redis timeout = 20ms
```

那么：

```text
A → R3
   ↓
等待 20ms
   ↓
没有响应
   ↓
timeout
   ↓
认为 R3 这次加锁失败
```

然后继续处理其他 Redis。

------

# 15. timeout 和 TTL 的关系

这是学习 RedLock 最容易混淆的地方。

需要区分：

## ① Lock TTL

例如：

```text
PX 10000
```

表示：

```text
锁 TTL = 10秒
```

即：

> 锁最多存在 10 秒。

------

## ② 单 Redis 请求 timeout

例如：

```text
timeout = 20ms
```

表示：

> 访问某一个 Redis 实例，最多等待 20ms。

------

## 正确关系

通常希望：

```text
单 Redis timeout << Lock TTL
```

例如：

```text
单节点 timeout = 20ms

Lock TTL = 10秒
```

即：

```text
20ms << 10s
```

------

# 16. 为什么 timeout 必须远小于 TTL？

因为 RedLock 要：

```text
快速尝试多个 Redis
```

而不是：

```text
在某一个挂掉的 Redis 上长时间等待。
```

例如：

```text
TTL = 10秒
```

如果：

```text
单 Redis timeout = 5秒
```

那么：

```text
R1 → 等5秒
R2 → 等5秒
...
```

很容易出现：

```text
锁的 TTL 已经快用完
```

甚至：

```text
加锁过程耗时 > TTL
```

这时即使拿到了多数 Redis，也不能认为自己拥有一个有效的锁。

所以：

```text
单 Redis timeout
        ↓
应该很短

Lock TTL
        ↓
相对更长
```

------

# 17. RedLock 不要求所有 Redis 都成功

这是一个非常重要的理解。

5 个 Redis：

```text
R1 R2 R3 R4 R5
```

不要求：

```text
5 / 5
```

只需要：

```text
≥ 3 / 5
```

例如：

```text
R1 ✓
R2 ✓
R3 ✓
R4 ✗
R5 ✗
```

可以成功。

再比如：

```text
R1 ✓
R2 ✓
R3 timeout
R4 ✓
R5 timeout
```

确认：

```text
3 / 5
```

也可以成功。

------

# 18. “没返回”和“加锁失败”不是完全一样

假设：

```text
A → R3：SET lock A
```

R3 实际已经执行成功：

```text
R3：

lock = A
```

但是响应在网络中丢失：

```text
R3 → A

响应丢失
```

A 看到：

```text
timeout
```

所以 A 只能认为：

```text
R3 不计入成功
```

但是 R3 上可能已经存在：

```text
lock = A
```

所以 A 判断：

```text
成功：
R1 ✓
R2 ✓
R3 timeout
R4 ✗
R5 ✗

2 / 5
```

最终：

```text
加锁失败
```

------

# 19. 加锁失败后怎么办？

如果客户端之前已经在某些 Redis 上成功加锁：

```text
R1 ✓
R2 ✓
R3 timeout
R4 ✗
R5 ✗
```

即使最终没有达到多数派，也应该尝试清理：

```text
R1 → 删除自己的锁
R2 → 删除自己的锁
R3 → 尝试删除自己的锁
```

如果 R3 仍然不可访问：

```text
清理失败
```

也没关系。

因为：

```text
R3 上的锁
    ↓
最终会因为 TTL
    ↓
自动过期
```

因此：

> TTL 是故障情况下的兜底机制。

------

# 20. RedLock 判断成功的两个核心条件

可以简单记成：

```text
① 获得多数 Redis 实例的锁

        +

② 获取锁的整个过程耗时小于 TTL
```

例如：

```text
5 个 Redis

R1 ✓
R2 ✓
R3 ✓
R4 ✗
R5 ✗

3 / 5
```

并且：

```text
加锁耗时 = 30ms
TTL = 10s
```

那么：

```text
30ms << 10s
```

锁还有大量有效时间。

所以可以认为：

```text
RedLock 获取成功
```

------

# 21. 为什么加锁过程耗时也很重要？

假设：

```text
TTL = 10秒
```

但是：

```text
加锁过程耗时 = 11秒
```

即使：

```text
3 / 5 Redis 成功
```

也没有意义。

因为：

```text
锁 TTL：

0s ───────────────→ 10s
                     ↑
                 已经过期
```

所以：

```text
多数派成功
```

并不是唯一条件。

还需要：

```text
锁还有足够的剩余有效时间
```

------

# 22. RedLock 中的时间模型

可以理解成：

```text
开始加锁
   │
   ↓
尝试 R1
   ↓
尝试 R2
   ↓
尝试 R3
   ↓
尝试 R4
   ↓
尝试 R5
   │
   ↓
判断多数派
   │
   ↓
计算已经消耗的时间
   │
   ↓
TTL - 已耗时间 - 安全余量
   │
   ↓
还有足够时间？
   ├── 是 → 获得锁
   └── 否 → 加锁失败
```

------

# 23. 为什么 TTL 不能无限长？

例如：

```text
TTL = 1小时
```

A：

```text
获得锁
 ↓
网络断开
 ↓
A已经无法继续工作
```

但是 Redis：

```text
lock = A
TTL = 59分钟
```

B：

```text
B → 无法获得锁
```

可能需要等待很久。

所以 TTL 不能随便设置得非常长。

------

# 24. TTL 也不能太短

例如：

```text
TTL = 1秒
```

业务：

```text
A 获得锁
 ↓
业务处理需要5秒
```

那么：

```text
0秒 → A获得锁
1秒 → 锁自动过期
2秒 → B获得锁
3秒 → A还在处理
```

这时可能出现：

```text
A、B同时认为自己拥有锁
```

因此：

> TTL 应该覆盖正常业务执行时间，并留有合理余量。

------

# 25. RedLock 的整体结构

```text
                    RedLock
                       │
              多个独立 Redis
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
       R1             R2             R3...
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                 尝试快速加锁
                       ↓
                达到多数派？
                 ┌─────┴─────┐
                 ↓           ↓
                是           否
                 ↓           ↓
          判断剩余TTL       加锁失败
                 ↓
          时间足够？
           ┌─────┴─────┐
           ↓           ↓
          是           否
           ↓           ↓
        获得锁        加锁失败
```

------

# 26. 单 Redis 锁 vs RedLock

| 对比             | 单 Redis    | RedLock                |
| ---------------- | ----------- | ---------------------- |
| Redis 实例       | 1个         | 多个                   |
| 加锁方式         | SET NX PX   | 多个实例分别 SET NX PX |
| 判断成功         | 单节点成功  | 多数节点成功           |
| 单点故障         | 存在        | 降低单点故障影响       |
| 实现复杂度       | 低          | 高                     |
| 网络开销         | 较低        | 更高                   |
| 需要考虑 timeout | 是          | 尤其重要               |
| 需要考虑 TTL     | 是          | 尤其重要               |
| 释放锁           | value + Lua | 清理多个实例上的锁     |

------

# 27. Redis List 消息队列和分布式锁不要混淆

Redis List：

```text
生产者
   ↓
Redis List
   ↓
消费者
```

可以：

```text
多生产者 + 多消费者
```

但多个消费者属于：

> 竞争消费。

例如：

```text
             Redis List
            /    |    \
           ↓     ↓     ↓
          A      B      C
```

一条消息：

```text
订单1
```

只会被其中一个消费者取走。

适合：

```text
任务队列
```

------

# 28. Pub/Sub 和 List 的区别

## Pub/Sub

```text
生产者
   ↓
Redis Channel
   ↓
A、B、C
```

一条消息可以被：

```text
A 收到
B 收到
C 收到
```

属于：

> 广播。

但是：

```text
消费者离线
    ↓
消息不会等待它
    ↓
可能丢失
```

------

## List

```text
生产者
   ↓
Redis List
   ↓
A、B、C
```

多个消费者：

```text
竞争消费
```

一条消息通常只被一个消费者取走。

可以使用：

```bash
BLPOP
BRPOP
```

进行阻塞消费。

但是原生 List 缺少：

```text
ACK
失败重试
死信队列
消费状态管理
```

因此功能弱于专业 MQ。

------

# 29. Redis 分布式锁常见面试题

## Q1：为什么使用 SET NX PX？

```text
NX → 保证互斥
PX → 防止锁永久存在
```

------

## Q2：为什么 value 要唯一？

防止：

```text
A 的锁过期
 ↓
B 获得锁
 ↓
A 执行完
 ↓
A 删除 B 的锁
```

所以需要通过 value 判断：

```text
这把锁是不是我的？
```

------

## Q3：为什么释放锁需要 Lua？

因为：

```text
判断 value
+
删除 key
```

必须保证原子性。

否则可能：

```text
GET
 ↓
锁过期
 ↓
别人获得锁
 ↓
DEL
```

导致误删别人的锁。

------

## Q4：RedLock 为什么需要多个 Redis？

为了减少：

```text
单个 Redis 实例故障
```

导致整个分布式锁失效的问题。

------

## Q5：5 个 Redis 为什么需要 3 个成功？

因为采用多数派：

```text
N / 2 + 1
```

5 个：

```text
5 / 2 + 1 = 3
```

------

## Q6：如果只有两个 Redis 成功呢？

```text
2 / 5
```

达不到多数派：

```text
加锁失败
```

即使另一个 Redis：

```text
实际上可能已经加锁成功
但是响应没有返回
```

客户端也不能把它算作确认成功。

------

## Q7：为什么单节点 timeout 要远小于 TTL？

因为需要：

```text
快速尝试多个 Redis
```

不能：

```text
在故障 Redis 上长时间等待
```

否则可能：

```text
加锁耗时过长
 ↓
TTL 快用完
 ↓
即使达到多数派
 ↓
锁也没有足够有效时间
```

所以：

```text
单 Redis timeout << Lock TTL
```

------

# 30. 最终记忆版

Redis 分布式锁最基础的实现：

```bash
SET lock_key unique_value NX PX 10000
```

记住：

```text
NX
 ↓
只有第一个客户端能创建锁
 ↓
保证互斥

PX
 ↓
锁自动过期
 ↓
防止客户端宕机造成死锁

unique_value
 ↓
标识锁的持有者
 ↓
防止误删别人的锁

Lua
 ↓
判断 value + 删除
 ↓
保证释放锁的原子性
```

RedLock：

```text
多个独立 Redis
      ↓
快速尝试加锁
      ↓
多数派成功
      ↓
并且整个加锁过程耗时 < TTL
      ↓
获得锁
```

如果：

```text
只有 2 / 5 成功
```

则：

```text
加锁失败
```

如果：

```text
某个 Redis 一直不返回
```

则：

```text
等待单节点 timeout
        ↓
超时
        ↓
不再等待该节点
        ↓
继续其他节点
```

即使：

```text
Redis 实际已经加锁
但响应丢失
```

客户端也只能：

```text
认为这次没有获得该节点的确认
```

最终如果没有达到多数派：

```text
加锁失败
 ↓
清理已经成功的锁
 ↓
无法清理的锁
 ↓
依靠 TTL 自动过期
```

最重要的时间关系：

```text
单 Redis 请求 timeout << Lock TTL
```

而不是：

```text
Lock TTL < Redis 故障检测时间
```

> **核心思想：RedLock 要快速地从多个 Redis 中获得多数派，同时保证获得的锁还有足够的剩余有效时间。**