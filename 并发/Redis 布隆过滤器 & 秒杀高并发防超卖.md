# Redis 布隆过滤器 & 秒杀高并发防超卖

## 一、布隆过滤器是什么？

布隆过滤器（Bloom Filter）是一种：

> **空间效率很高的概率型数据结构，用来判断一个数据“可能存在”还是“一定不存在”。**

它由两部分组成：

1. **位图数组（Bitmap）**
   - 初始全部为 `0`
   - 例如长度为 8：

```text
位置：0 1 2 3 4 5 6 7
值：  0 0 0 0 0 0 0 0
```

2. **多个哈希函数**
   - 一个数据经过多个 Hash 函数
   - 得到多个哈希值
   - 再对位图长度取模，得到多个 bit 下标

---

# 二、布隆过滤器如何添加数据？

假设：

```text
位图长度 = 8
哈希函数数量 = 3
```

数据：

```text
x
```

经过 3 个哈希函数：

```text
hash1(x) % 8 = 1
hash2(x) % 8 = 4
hash3(x) % 8 = 6
```

于是把：

```text
bit[1]
bit[4]
bit[6]
```

设置为 `1`：

```text
位置：0 1 2 3 4 5 6 7
值：  0 1 0 0 1 0 1 0
```

注意：

> Bloom Filter 并不是把 `x` 本身存进去，而是把 `x` 对应的几个 bit 设置成 `1`。

---

# 三、查询数据时怎么判断？

假设查询：

```text
x
```

再执行相同的三个 Hash：

```text
1、4、6
```

然后检查：

```text
bit[1]
bit[4]
bit[6]
```

## 情况 1：有一个 bit 是 0

例如：

```text
bit[1] = 1
bit[4] = 0
bit[6] = 1
```

那么：

```text
x 一定不存在
```

为什么？

因为如果 `x` 真的加入过 Bloom Filter：

```text
bit[1]
bit[4]
bit[6]
```

一定都会被设置成 `1`。

所以只要发现一个是 `0`：

> `x` 以前一定没有被加入过。

---

## 情况 2：所有 bit 都是 1

例如：

```text
bit[1] = 1
bit[4] = 1
bit[6] = 1
```

只能说明：

```text
x 可能存在
```

不能证明：

```text
x 一定存在
```

因为可能存在哈希冲突。

例如：

```text
x → 1、4、6

y → 1、4、6
```

虽然数据库里没有 `y`，但是：

```text
bit[1] = 1
bit[4] = 1
bit[6] = 1
```

所以 Bloom Filter 会认为：

```text
y 可能存在
```

但实际上：

```text
数据库没有 y
```

这就是**误判（假阳性，False Positive）**。

---

# 四、布隆过滤器最重要的两个性质

| Bloom Filter 结果 | 实际情况                   |
| ----------------- | -------------------------- |
| `false`           | **一定不存在**             |
| `true`            | **可能存在，也可能不存在** |

因此：

```text
不存在判断 → 可以相信
存在判断   → 还要最终确认
```

不会出现：

```text
实际存在
↓
Bloom Filter 判断不存在
```

即：

> **Bloom Filter 没有假阴性（False Negative），但存在假阳性（False Positive）。**

---

# 五、为什么会产生误判？

因为不同数据经过 Hash 后可能落到相同的位置。

例如：

```text
x → 1、4、6
y → 1、4、6
```

位图只有：

```text
0 1 0 0 1 0 1 0
```

Bloom Filter 只知道：

```text
1、4、6 曾经被某些数据设置过
```

但不知道：

```text
到底是谁设置的
```

所以多个数据不断加入后，越来越多的 bit 会变成 `1`：

```text
0 0 0 0 0 0 0 0
        ↓
0 1 0 0 1 0 1 0
        ↓
1 1 1 0 1 0 1 0
        ↓
1 1 1 1 1 1 1 0
```

如果最终接近：

```text
1 1 1 1 1 1 1 1
```

那么很多不存在的数据都会被判断成：

```text
可能存在
```

误判率就会很高。

---

# 六、布隆过滤器为什么适合防缓存穿透？

缓存穿透：

```text
请求不存在的数据
        ↓
Redis 没有
        ↓
查询 MySQL
        ↓
大量不存在请求打到 MySQL
```

例如：

```text
100万个恶意请求
        ↓
Bloom Filter
        ↓
绝大多数请求发现一定不存在
        ↓
直接返回
```

只有：

```text
Bloom Filter = true
```

的请求才继续：

```text
Bloom Filter
    ↓
Redis
    ↓
MySQL
```

所以 Bloom Filter 的定位是：

> **数据库前面的一个快速过滤器。**

---

# 七、Bloom Filter 能处理什么数据？

不限于数字。

只要能够计算 Hash，都可以。

例如：

```text
用户ID
商品ID
订单ID
字符串
URL
IP
文件Hash
```

例如：

```text
"apple"
    ↓
Hash
    ↓
多个 bit 下标
```

所以：

> Bloom Filter 本质上处理的是“数据的哈希结果”，不是“数字”。

---

# 八、Bloom Filter 的容量怎么设置？

主要关注两个参数：

```text
n = 预计元素数量
p = 目标误判率
```

例如：

```text
n = 1000万
p = 1%
```

这里的 `1000万` 指的是：

> **这个 Bloom Filter 预计要存放 1000 万个元素。**

不是：

```text
整个 Redis 的数据量
```

也不是：

```text
Redis Key 的数量
```

而是：

```text
某一个 Bloom Filter
        ↓
预计会加入多少元素
```

---

# 九、m 和 k 是什么？

底层还有两个参数：

```text
m = 位图总 bit 数
k = Hash 函数数量
```

一般不需要手动计算。

成熟组件通常根据：

```text
n
+
p
```

自动计算：

```text
m
+
k
```

理论公式：

\[
m \approx -\frac{n\ln p}{(\ln2)^2}
\]

\[
k \approx \frac{m}{n}\ln2
\]

所以实际开发通常只需要考虑：

```text
预计数据量
+
允许误判率
```

---

# 十、Bloom Filter 放在哪里？

Bloom Filter 是一种数据结构，不等于 Redis。

它可以放在：

```text
JVM 内存
Redis
其他存储系统
```

---

## 1. Java 本地 Bloom Filter

例如 Guava：

```java
BloomFilter<String> bloomFilter =
        BloomFilter.create(
                Funnels.stringFunnel(StandardCharsets.UTF_8),
                10_000_000,
                0.01
        );
```

查询：

```java
bloomFilter.mightContain("1001");
```

此时：

```text
Bloom Filter
    ↓
JVM 本地内存
```

优点：

```text
查询非常快
不需要网络访问
```

缺点：

```text
每个 JVM 实例一份
实例之间不共享
服务重启需要重新构建
实例扩容需要重新初始化
```

---

## 2. Redis Bloom Filter

如果使用 RedisBloom：

```text
Java 实例 A ─┐
Java 实例 B ─┼──→ Redis
Java 实例 C ─┘      ↓
                Bloom Filter
```

多个 Java 实例共享一份。

RedisBloom 常见命令：

```redis
BF.RESERVE goods_bloom 0.01 10000000
```

表示：

```text
Bloom Filter:
goods_bloom

预计元素：
1000万

误判率：
1%
```

添加：

```redis
BF.ADD goods_bloom 1001
BF.ADD goods_bloom 1002
```

查询：

```redis
BF.EXISTS goods_bloom 1001
```

返回：

```text
1
```

代表：

```text
可能存在
```

查询：

```redis
BF.EXISTS goods_bloom 9999
```

返回：

```text
0
```

代表：

```text
一定不存在
```

---

# 十一、Java 如何使用 Redis 中的 Bloom Filter？

普通 `RedisTemplate` 主要封装：

```text
String
Hash
List
Set
ZSet
```

RedisBloom 的：

```text
BF.ADD
BF.EXISTS
BF.RESERVE
```

不是普通 Redis 核心数据结构 API。

因此可以使用：

```java
redisTemplate.execute(...)
```

底层执行 Redis 命令。

调用链：

```text
Java
 ↓
RedisTemplate
 ↓
Redis 网络连接
 ↓
Redis Server
 ↓
执行 BF.EXISTS
 ↓
返回结果
 ↓
Java
```

例如概念上：

```java
redisTemplate.execute(
        connection ->
                connection.execute(
                        "BF.EXISTS",
                        keyBytes,
                        valueBytes
                )
);
```

---

# 十二、RedisTemplate.execute() 到底是什么？

可以理解为：

> **通过 RedisTemplate 获取底层 Redis 连接，直接让 Redis 服务器执行 Redis 命令。**

例如：

```java
redisTemplate.opsForValue().get("goods:1");
```

大致对应：

```redis
GET goods:1
```

而：

```java
connection.execute("BF.EXISTS", ...)
```

对应：

```redis
BF.EXISTS goods_bloom 1001
```

所以：

```text
RedisTemplate
├── opsForValue()
├── opsForHash()
├── opsForSet()
└── execute()
      ↓
   更底层
      ↓
直接执行 Redis 命令
```

注意：

> `execute()` 是 Java 方法，但是具体 Redis 命令是在 Redis Server 中执行的。

---

# 十三、秒杀中的超卖问题

秒杀首先需要解决：

```text
高并发
+
超卖
```

假设：

```text
商品库存 = 100
```

突然：

```text
100万个请求
```

如果全部直接访问 MySQL：

```text
100万个请求
        ↓
MySQL
        ↓
连接池
        ↓
锁竞争
        ↓
大量等待
        ↓
超时
```

所以一般不会让所有请求直接打 MySQL。

---

# 十四、数据库直接扣库存为什么可能超卖？

## 错误方式：先 SELECT，再判断

假设：

```text
stock = 1
```

两个事务并发：

```sql
SELECT stock
FROM goods
WHERE goods_id = 1;
```

可能出现：

```text
事务A：读到 1
事务B：读到 1
```

因为普通 `SELECT` 并不会让两个事务互斥。

然后：

```text
A：认为有库存
B：认为有库存
```

最后都创建订单：

```text
A → 订单成功
B → 订单成功
```

虽然数据库库存可能最终是：

```text
0
```

但是实际上：

```text
卖出了2件
实际库存只有1件
```

这就是超卖。

---

# 十五、数据库加行锁：SELECT FOR UPDATE

可以：

```sql
SELECT *
FROM goods
WHERE goods_id = ?
FOR UPDATE;
```

假设：

```text
stock = 1
```

A：

```text
SELECT FOR UPDATE
        ↓
锁住这一行
        ↓
读取 stock = 1
        ↓
UPDATE stock = 0
        ↓
COMMIT
```

B：

```text
SELECT FOR UPDATE
        ↓
等待 A
        ↓
A 提交
        ↓
B 获取锁
        ↓
读取 stock = 0
        ↓
库存不足
```

因此可以防止超卖。

但问题：

```text
大量请求
    ↓
同时竞争同一行
    ↓
大量锁等待
    ↓
数据库吞吐下降
```

---

# 十六、MySQL 更常用的条件更新

可以直接：

```sql
UPDATE goods
SET stock = stock - 1
WHERE goods_id = ?
  AND stock > 0;
```

然后检查：

```java
int affectedRows = mapper.deductStock(goodsId);
```

如果：

```text
affectedRows == 1
```

表示：

```text
扣库存成功
```

如果：

```text
affectedRows == 0
```

表示：

```text
库存不足
```

---

# 十七、为什么 UPDATE ... WHERE stock > 0 不会简单地超卖？

假设：

```text
stock = 1
```

A、B 可以并发到达 MySQL。

但是修改的是：

```text
同一行
```

因此 MySQL 对修改会进行并发控制。

大致过程：

```text
A：
UPDATE ... stock > 0
        ↓
拿到该行修改锁
        ↓
stock = 1
        ↓
条件满足
        ↓
stock = 0
        ↓
提交

B：
等待 A
        ↓
A 提交
        ↓
继续
        ↓
当前 stock = 0
        ↓
stock > 0 不成立
        ↓
affectedRows = 0
```

所以：

```text
A → 成功
B → 失败
```

---

# 十八、原子性 ≠ 互斥

这是并发里非常重要的概念。

## 原子性

表示：

> 一个操作不可被拆开。

例如：

```text
DECR stock
```

是一个原子操作。

---

## 互斥

表示：

> 多个执行者不能同时进入某个临界区。

例如：

```text
A 拿锁
    ↓
B 等待
    ↓
A 执行
    ↓
A 释放
    ↓
B 执行
```

这是互斥。

---

## CAS

CAS 不要求所有线程互斥。

它允许：

```text
A、B 并发执行
```

但是修改时要求：

```text
当前值必须还是我期望的值
```

例如：

```text
stock = 1
```

A、B 都读到：

```text
1
```

A：

```text
CAS(1 → 0)
```

成功。

此时：

```text
stock = 0
```

B：

```text
CAS(1 → 0)
```

失败。

因为：

```text
expected = 1
actual   = 0
```

所以：

```text
A → 成功
B → 失败
```

CAS 的核心是：

> **允许并发，但在修改时检测冲突。**

---

# 十九、Redis DECR 为什么不能单独使用？

假设：

```text
stock = 1
```

直接：

```redis
DECR stock
```

A：

```text
1 → 0
```

B：

```text
0 → -1
```

所以可能出现：

```text
stock = -1
```

因此不能简单：

```text
DECR
```

然后：

```text
返回成功
```

因为第二个请求也可能得到一个结果，并把库存减成负数。

---

# 二十、正确做法：判断 + 扣减必须是一个原子操作

例如 Redis Lua：

```lua
local stock = tonumber(redis.call('GET', KEYS[1]))

if stock == nil or stock <= 0 then
    return 0
end

redis.call('DECR', KEYS[1])

return 1
```

关键不是：

```text
if stock > 0
```

本身。

关键是：

```text
判断库存
+
扣减库存
```

必须放在：

> **同一个原子执行单元**

中。

---

# 二十一、为什么两个 Lua 请求不会出现“都读到最后一个库存”？

假设：

```text
stock = 1
```

A、B 同时请求。

Redis 不会让两个 Lua 脚本交错执行。

执行类似：

```text
A 的 Lua：
    GET → 1
    判断 > 0
    DECR → 0
    返回成功

B 的 Lua：
    GET → 0
    判断失败
    返回失败
```

而不会：

```text
A：GET → 1
B：GET → 1
A：DECR
B：DECR
```

所以最终：

```text
stock = 0

A → 成功
B → 失败
```

---

# 二十二、这里要注意：并发请求 ≠ Redis 内部并行执行 Lua

请求层面：

```text
A ─┐
B ─┼──→ Redis
C ─┘
```

可以并发。

但 Lua 脚本执行时：

```text
A Lua 完整执行
        ↓
B Lua 完整执行
        ↓
C Lua 完整执行
```

不会在 A 的脚本执行到一半时插入 B。

因此：

```text
HTTP 请求：可以并发
Java 线程：可以并发
访问 Redis：可以并发
Lua 脚本执行：不会互相交错
```

---

# 二十三、为什么 Redis 预减库存？

假设：

```text
MySQL库存 = 100
Redis库存 = 100
```

来了：

```text
100万请求
```

先通过 Redis：

```text
100万请求
        ↓
Redis 原子预减
        ↓
前100个抢到库存
        ↓
其他请求直接失败
```

然后：

```text
前100个请求
        ↓
MQ
        ↓
消费者
        ↓
MySQL
```

于是：

```text
MySQL 不需要处理100万请求
只需要处理大约100个成功请求
```

这就是：

> **预减库存 + 异步削峰**

---

# 二十四、预减库存是什么意思？

预减：

> **在真正创建订单、最终修改数据库库存之前，先在 Redis 中抢一个库存名额。**

例如：

```text
Redis stock = 100
```

用户请求：

```text
Redis 原子判断 + 扣减
        ↓
100 → 99
        ↓
抢到一个库存名额
```

然后进入：

```text
MQ
```

最后：

```text
MySQL
```

所以：

```text
Redis扣库存
    ↓
预扣

MySQL扣库存
    ↓
最终扣减
```

Redis 里的扣减不能简单等同于：

```text
数据库已经真正减库存
```

---

# 二十五、为什么 Redis 预减还需要 MQ？

因为秒杀流量通常非常大。

例如：

```text
100万请求
```

真正库存：

```text
100
```

如果让全部请求直接进入 MySQL：

```text
100万 → MySQL
```

压力非常大。

所以：

```text
100万
 ↓
Redis
 ↓
只有100左右进入MQ
 ↓
消费者慢慢处理
 ↓
MySQL
```

MQ 的核心作用：

> **削峰、异步处理、解耦。**

---

# 二十六、为什么不用 CAS 解决整个秒杀？

CAS 可以解决：

```text
库存变量的并发修改
```

例如：

```text
stock = 1

A → CAS(1 → 0) ✅
B → CAS(1 → 0) ❌
```

但是 CAS 无法解决：

```text
100万请求如何削峰
订单如何创建
重复下单
MQ消息丢失
MQ重复消费
Redis和MySQL一致性
支付
库存回补
```

所以不同技术负责不同问题：

```text
CAS / Redis Lua
        ↓
库存并发竞争

MQ
        ↓
流量削峰

MySQL条件更新
        ↓
最终防超卖

唯一索引
        ↓
防重复订单

事务
        ↓
保证数据库操作一致性

重试 / 补偿
        ↓
最终一致性
```

---

# 二十七、完整秒杀架构

```text
                 100万请求
                     │
                     ▼
               网关 / 限流
                     │
                     ▼
              Redis预减库存
             （Lua原子操作）
                     │
            ┌────────┴────────┐
            │                 │
       库存不足             抢到库存
            │                 │
            ▼                 ▼
          失败               MQ
                              │
                              ▼
                        MQ Consumer
                              │
                              ▼
                           MySQL
                        ┌─────┴─────┐
                        │           │
                     扣库存       创建订单
                        │           │
                        └─────┬─────┘
                              │
                              ▼
                            成功
```

---

# 二十八、MySQL 最后一层为什么还要 `stock > 0`？

因为：

```text
Redis
```

和：

```text
MySQL
```

是两个独立系统。

即使：

```text
Redis预减成功
```

也不能认为：

```text
MySQL一定成功
```

可能发生：

```text
Redis扣减成功
        ↓
MQ重复消费
        ↓
服务异常
        ↓
数据不一致
```

所以最终数据库仍然使用：

```sql
UPDATE goods
SET stock = stock - 1
WHERE goods_id = ?
  AND stock > 0;
```

相当于：

> **数据库最后一道防线。**

---

# 二十九、为什么还需要唯一索引？

假设用户 A 发起两次请求：

```text
请求1
请求2
```

如果没有幂等控制：

```text
两个请求都可能进入MQ
        ↓
消费者处理两次
        ↓
创建两个订单
```

因此数据库通常增加：

```sql
UNIQUE KEY uk_user_goods(user_id, goods_id)
```

这样：

```text
同一个用户
+
同一个商品
```

只能创建一条订单。

---

# 三十、秒杀问题最终可以拆成几个独立问题

```text
问题1：大量请求打数据库
    ↓
Redis预减
+
限流

问题2：库存并发扣减
    ↓
Redis Lua / CAS
+
MySQL条件更新

问题3：大量请求同时处理
    ↓
MQ削峰

问题4：重复下单
    ↓
唯一索引
+
幂等

问题5：Redis和MySQL不一致
    ↓
MQ可靠投递
+
重试
+
补偿
+
最终一致性
```

---

# 三十一、几个容易混淆的概念

## 1. 原子性

```text
一个操作不可被拆开
```

不是：

```text
所有事务互斥
```

---

## 2. 互斥

```text
同一时刻只有一个执行者进入临界区
```

典型：

```text
分布式锁
```

---

## 3. CAS

```text
允许并发
+
修改时检查旧值
+
冲突就失败
```

---

## 4. Redis Lua

把：

```text
判断
+
修改
+
其他条件
```

放进一个 Redis 原子脚本执行。

---

## 5. Redis 预减

```text
Redis先抢库存名额
```

目的是：

```text
快速过滤
+
削峰
+
保护MySQL
```

---

## 6. MySQL 条件更新

```sql
UPDATE ...
SET stock = stock - 1
WHERE stock > 0;
```

目的是：

```text
最终防止数据库超卖
```

---

## 7. Bloom Filter

```text
false → 一定不存在
true  → 可能存在
```

目的是：

```text
快速过滤不存在的数据
```

---

# 三十二、最终记忆图

```text
                 高并发请求
                     │
                     ▼
                  限流
                     │
                     ▼
             ┌──────────────┐
             │ Redis        │
             │ Bloom Filter │ ← 防缓存穿透
             └──────┬───────┘
                    │
                    ▼
             Redis预减库存
             Lua原子操作
                    │
           ┌────────┴────────┐
           │                 │
       库存不足             成功
           │                 │
           ▼                 ▼
         失败               MQ
                              │
                              ▼
                           消费者
                              │
                              ▼
                           MySQL
                              │
                    UPDATE stock > 0
                              │
                              ▼
                         创建订单
                              │
                              ▼
                        唯一索引幂等
```

---

# 三十三、一句话总结

### Bloom Filter

> **快速排除“一定不存在”的数据，允许假阳性，不允许假阴性。**

### CAS

> **允许并发，通过“期望值 == 当前值”保证只有一个修改成功。**

### Redis Lua

> **把“判断 + 扣减”等多个 Redis 操作放进一个原子执行单元。**

### Redis 预减库存

> **先在 Redis 抢库存名额，把大量失败请求挡住，保护 MySQL。**

### MQ

> **削峰填谷，把抢到库存的少量请求异步交给后端处理。**

### MySQL 条件更新

> **作为最终数据库层面的防超卖兜底。**

### 唯一索引

> **防止同一个用户重复创建同一个商品的秒杀订单。**

### 整个秒杀系统

> **不是靠某一种技术解决全部问题，而是让 Redis、Lua/CAS、MQ、MySQL、唯一索引分别解决不同层面的并发问题。**