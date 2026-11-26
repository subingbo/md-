# Redis 面试笔记：ZSet、AOF、RDB 与分布式锁

## 一、ZSet（Sorted Set，有序集合）

### 1. 基本概念

ZSet 是 Redis 的有序集合，每个元素包含两个属性：

- **member**：成员，即实际存储的元素，通常是字符串。
- **score**：分数，类型为 double，用于排序。

可以理解为：

```
ZSet = member + score
```

例如游戏排行榜：

| member  | score |
| ------- | ----- |
| Alice   | 100   |
| Bob     | 80    |
| Charlie | 100   |

### 2. ZSet 的特点

**（1）member 唯一**

同一个 ZSet 中，member 不能重复。

如果再次添加相同的 member，会更新它原来的 score，而不是新增元素。

例如：

```
ZADD game:rank 100 Alice
ZADD game:rank 120 Alice
```

最终只有一个 Alice，score 为 120。

**（2）score 可以重复**

不同 member 可以具有相同的 score。

**（3）排序规则**

ZSet 首先按照 score 从小到大排序。

当 score 相同时，按照 member 的字典序排序。

例如：

| member  | score |
| ------- | ----- |
| Bob     | 80    |
| Alice   | 100   |
| Charlie | 100   |

Alice 和 Charlie 的 score 相同，因此再按照 member 的字典序排序。

### 3. 常用命令

```
# 添加元素
ZADD game:rank 100 Alice

# 查询所有成员和分数
ZRANGE game:rank 0 -1 WITHSCORES

# 查询排行榜（从高到低）
ZREVRANGE game:rank 0 -1 WITHSCORES

# 查询成员分数
ZSCORE game:rank Alice

# 查询成员排名
ZRANK game:rank Alice

# 删除成员
ZREM game:rank Alice
```

### 4. 底层数据结构

ZSet 根据数据规模和编码条件，使用不同的数据结构：

- 小数据量：Listpack。
- 较大数据量：跳表（Skip List）+ 哈希表（Hash Table）。

跳表主要负责按照 score 排序和范围查询，哈希表主要负责快速查找 member 对应的 score。

注意：不是所有 ZSet 都一定使用跳表和哈希表。

### 5. 常见应用场景

- 游戏积分排行榜。
- 商品热度排行。
- 用户活跃度排行。
- 按时间或权重排序的任务队列。

## 二、Redis 持久化

### 1. 为什么需要持久化？

Redis 是基于内存的数据库，数据主要存储在内存中。

如果 Redis 服务器宕机或重启，单纯存在内存中的数据就可能丢失。

因此 Redis 提供持久化机制，将数据保存到磁盘，以便重启后恢复。

Redis 常见的持久化方式：

1. AOF（Append Only File）。
2. RDB（Redis Database）。
3. 混合持久化（RDB + AOF）。

### 2. AOF：记录写操作命令

AOF（Append Only File，追加文件）记录能够改变 Redis 数据的写操作命令。

**核心思想：记录操作过程，恢复时重新执行命令。**

例如，执行以下命令：

```
SET counter 0
INCR counter
INCR counter
INCR counter
```

AOF 中记录的操作类似：

```
SET counter 0
INCR counter
INCR counter
INCR counter
```

Redis 重启时，读取 AOF 文件并重新执行这些命令，最终恢复：

```
counter = 3
```

注意：

- SET、DEL、INCR、HSET、LPUSH、ZADD 等写操作会被记录。
- GET、HGET、ZRANGE 等纯读操作不会被记录。

AOF 实际使用 Redis 协议格式记录命令，不是简单地保存成上面的文本示意形式。

### 3. AOF 的三种刷盘策略

Redis 通过 `appendfsync` 配置刷盘策略。

| 策略     | 含义                   | 特点                       |
| -------- | ---------------------- | -------------------------- |
| always   | 每次写操作后同步刷盘   | 数据安全性高，性能开销较大 |
| everysec | 每秒刷盘一次           | 性能和安全性较均衡         |
| no       | 由操作系统决定刷盘时机 | Redis 不主动控制刷盘时机   |

常见默认策略是 everysec。

在正常运行条件下，如果发生宕机，通常最多丢失约 1 秒尚未刷盘的数据。

### 4. AOF 重写机制

问题：AOF 不断记录写操作，文件会越来越大。

例如：

```
SET counter 0
INCR counter
INCR counter
INCR counter
INCR counter
INCR counter
```

经过 AOF 重写后，可以将这些历史操作压缩成：

```
SET counter 5
```

这样就不需要保留所有历史操作，只需要保留能够恢复当前数据状态的命令。

注意：

AOF 重写不是简单删除旧日志，而是根据当前内存中的数据状态，生成一份新的、更精简的 AOF 文件。

## 三、RDB：记录某一时刻的数据快照

### 1. 基本概念

RDB（Redis Database）通过快照的形式，将某一时刻 Redis 内存中的数据保存到磁盘。

**核心思想：记录数据当前的状态，而不是记录数据如何变化。**

例如，Redis 执行：

```
SET counter 0
INCR counter
INCR counter
INCR counter
```

最终内存中的数据：

```
counter = 3
```

RDB 保存的是这个时间点的数据状态，而不是上面的四条操作命令。

RDB 使用二进制格式保存数据。

### 2. RDB 的生成方式

Redis 提供两种常见方式：

**SAVE**

```
SAVE
```

在主线程中同步生成 RDB 文件。

生成过程中会阻塞 Redis 主线程，影响客户端请求处理，因此通常不建议在生产环境随意使用。

**BGSAVE**

```
BGSAVE
```

通过 fork 创建子进程，由子进程负责生成 RDB 文件。

这样可以避免主线程直接执行整个快照写入过程。

实际生成过程中通常利用操作系统的写时复制（Copy-On-Write，COW）机制，降低父子进程之间的内存复制开销。

### 3. RDB 的优缺点

优点：

- 二进制快照文件通常比较紧凑。
- 恢复速度通常较快，可以直接加载快照。
- 适合定期备份和数据迁移。

缺点：

- 两次快照之间发生故障，可能丢失尚未保存的数据。
- 生成快照时 fork 子进程也有一定开销。
- 数据恢复到最近一次快照后，还需要考虑快照之后的数据变化。

## 四、AOF 和 RDB 的区别

### 1. 核心区别

| 对比维度   | AOF                      | RDB                        |
| ---------- | ------------------------ | -------------------------- |
| 保存内容   | 写操作命令               | 某一时刻的实际数据状态     |
| 文件形式   | Redis 协议格式的命令日志 | 二进制快照                 |
| 数据恢复   | 重新执行命令             | 加载快照                   |
| 恢复速度   | 通常较慢                 | 通常较快                   |
| 文件大小   | 通常较大                 | 通常较小                   |
| 数据安全性 | 可通过刷盘策略提高       | 两次快照之间的数据可能丢失 |
| 适用场景   | 对数据丢失比较敏感       | 定期备份、快速恢复         |

### 2. 为什么 RDB 文件通常比 AOF 文件更紧凑？

主要有三个原因：

**原因一：保存内容不同。**

RDB 只需要保存某个时间点的数据状态，而 AOF 通常保存一段时间内的写操作命令。

例如：

```
SET counter 0
INCR counter
INCR counter
INCR counter
INCR counter
INCR counter
```

AOF 可能记录：

```
SET counter 0
INCR counter
INCR counter
INCR counter
INCR counter
INCR counter
```

RDB 只需要保存：

```
counter = 5
```

**原因二：RDB 使用紧凑的二进制编码。**

RDB 不需要像 AOF 命令日志那样保存完整的命令名称和参数，可以使用更紧凑的数据编码方式。

**原因三：AOF 可能存在历史冗余。**

同一个 key 被反复修改时，AOF 可能保存多次历史写操作，而这些历史操作对于最终的数据状态可能已经没有意义。

因此，RDB 通常比 AOF 更紧凑。

注意：这是通常情况，不是绝对结论。AOF 重写之后，文件大小也可能明显缩小。

### 3. AOF 和 RDB 能否同时使用？

可以。

两者各有优势：

- RDB：恢复速度快，适合备份。
- AOF：可以通过刷盘策略控制数据丢失窗口。

Redis 也支持混合持久化，将 RDB 快照数据与后续 AOF 增量命令结合起来，以兼顾恢复速度和数据完整性。

## 五、Redis 分布式锁

### 1. 如何实现分布式锁？

可以使用 Redis 的 SET 命令：

```
SET lock_key unique_token NX PX 30000
```

参数含义：

- `lock_key`：锁对应的 Redis Key。
- `unique_token`：当前客户端生成的唯一标识。
- `NX`：只有 Key 不存在时才设置成功。
- `PX 30000`：设置过期时间为 30000 毫秒，即 30 秒。

这条命令能够原子地完成加锁和设置过期时间。

如果设置成功，说明获取锁成功；如果设置失败，说明锁已经被其他客户端持有。

### 2. 为什么需要过期时间？

假设客户端成功获取锁后，突然宕机，无法主动释放锁。

如果没有过期时间，其他客户端可能永远无法获取这把锁，产生死锁。

设置过期时间可以让锁在一定时间后自动释放。

### 3. 为什么不能直接 DEL 释放锁？

假设发生以下情况：

1. 客户端 A 获取锁，token 为 A。
2. A 执行业务时发生阻塞。
3. 锁过期，Redis 自动删除锁。
4. 客户端 B 获取锁，token 为 B。
5. A 恢复执行，直接执行 `DEL lock_key`。

此时 A 会把 B 的锁删除。

这就是误删其他客户端的锁。

### 4. 正确的释放方式

释放锁时，必须先比较锁的持有者标识，再删除锁，而且这两个操作必须是原子操作。

通常使用 Lua 脚本：

```
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

其中：

- `KEYS[1]`：锁对应的 Key。
- `ARGV[1]`：当前客户端持有的唯一 token。

只有 token 匹配，才允许删除锁。

### 5. Redisson 看门狗机制

问题：如果业务执行时间超过锁的过期时间怎么办？

例如：

- 锁的默认过期时间是 30 秒。
- 业务执行需要 60 秒。
- 30 秒后锁过期，其他客户端可能获取锁。

Redisson 提供看门狗（Watchdog）机制。

当持有者仍然正常持有锁时，看门狗会定期续期，避免锁因为固定租期到期而意外释放。

注意：

如果显式指定了 `leaseTime`，通常不会使用默认看门狗自动续期机制。

## 六、面试重点速记

| 面试问题                         | 核心回答                                                  |
| -------------------------------- | --------------------------------------------------------- |
| ZSet 的 member 和 score 是什么？ | member 是成员，score 是用于排序的分数                     |
| ZSet 的 member 能重复吗？        | 不能，相同 member 会更新 score                            |
| ZSet 的 score 能重复吗？         | 可以                                                      |
| ZSet 如何排序？                  | 先按 score，再按 member 字典序                            |
| ZSet 底层结构是什么？            | 小数据量通常使用 Listpack，较大数据量通常使用跳表和哈希表 |
| AOF 存什么？                     | 记录写操作命令                                            |
| RDB 存什么？                     | 某一时刻 Redis 内存中的实际数据状态                       |
| AOF 如何恢复？                   | 重新执行记录的命令                                        |
| RDB 如何恢复？                   | 加载二进制快照                                            |
| 为什么 RDB 通常更小？            | 不保存历史操作，采用紧凑的二进制编码                      |
| AOF 如何缩小文件？               | AOF 重写                                                  |
| Redis 如何实现原子加锁？         | `SET key token NX PX milliseconds`                        |
| 为什么不能直接 DEL 解锁？        | 可能误删其他客户端后来获取的锁                            |
| Redisson 看门狗有什么作用？      | 在持有者正常持锁时自动续期                                |

**最终记忆：**

- ZSet：member 唯一，score 可重复，按照 score 排序。
- AOF：记录写命令，恢复时重放。
- RDB：记录数据快照，恢复时加载。
- AOF 重写：根据当前数据状态生成更精简的命令。
- 分布式锁：原子加锁、唯一 token、过期时间、比较后原子释放。
- Redisson 看门狗：自动续期，避免业务未完成锁就过期。