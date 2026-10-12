# Redis 版本号 + Lua 原子操作：解决并发场景下的缓存旧值回填问题

## 一、问题背景：为什么 Redis 会被写入旧数据？

在使用 Cache Aside（旁路缓存）模式时，通常采用以下策略：

- 查询数据：先查 Redis，未命中再查 MySQL，并回填 Redis。
- 更新数据：先更新 MySQL，事务提交后删除 Redis 缓存。

这种方案在高并发场景下，可能出现旧查询结果覆盖新数据的问题。

### 1.1 并发问题示例

假设：

```
MySQL：name = 张三
Redis：缓存不存在
```

两个线程同时执行：

| 时间 | 线程 A：查询           | 线程 B：更新      |
| ---- | ---------------------- | ----------------- |
| T1   | 查询 Redis，未命中     |                   |
| T2   | 查询 MySQL，得到张三   |                   |
| T3   | 线程 A 暂停            |                   |
| T4   |                        | 更新 MySQL 为李四 |
| T5   |                        | MySQL 事务提交    |
| T6   |                        | 删除 Redis 缓存   |
| T7   | 将张三写入 Redis       |                   |
| T8   | Redis 最终保存旧值张三 |                   |

最终：

```
MySQL：李四（新数据）
Redis：张三（旧数据）
```

根本原因：线程 A 查询到旧数据后，无法判断回填缓存前数据是否已经发生更新。

## 二、解决方案：Redis Version + CAS + Lua

核心思想是：使用独立版本号记录数据变更，查询前获取版本号，回填时再次校验版本号。

### 2.1 Redis Key 设计

假设缓存用户 ID 为 1 的数据：

```
user:{1}:data       → 用户数据
user:{1}:version    → 数据版本号
```

例如：

```
user:{1}:data       = {"id":1,"name":"张三"}
user:{1}:version    = 5
```

两个 Key 的生命周期不同：

| Key     | 用途             | 是否允许删除   | 是否设置 TTL |
| ------- | ---------------- | -------------- | ------------ |
| data    | 保存业务缓存数据 | 允许           | 通常设置     |
| version | 记录数据更新次数 | 不随 data 删除 | 通常不设置   |

使用 `{1}` 是为了保证 Redis Cluster 下两个 Key 处于相同哈希槽，允许 Lua 同时操作。

### 2.2 Version 如何更新？

Version 不是删除后重新创建，而是使用 `INCR` 自增。

```
INCR user:{1}:version
```

例如：

```
更新前：
data    = 张三
version = 5

MySQL 更新成功并提交后：

INCR version
DEL data

更新后：
data    = null
version = 6
```

Redis 的 `INCR` 是原子命令。

如果 Key 不存在，Redis 会按照初始值 0 处理，第一次 `INCR` 返回 1。

但版本号不能因为过期、淘汰或重启而随意重置，否则可能出现版本回退或 ABA 问题。

### 2.3 完整查询流程

```
查询线程 A
    │
    ▼
读取 Redis data
    │
    ├── 命中 → 返回缓存
    │
    └── 未命中
          │
          ▼
    获取 Redis version = 5
          │
          ▼
    查询 MySQL
          │
          ▼
    获得数据库数据
          │
          ▼
    执行 Lua 脚本
          │
          ▼
    检查 Redis 当前 version
          │
          ├── 等于 5 → 允许回填
          │
          └── 不等于 5 → 拒绝回填
```

注意：查询开始时读取的版本号是快照，最终是否允许回填，取决于 Lua 执行时的版本号。

## 三、为什么需要 Lua？普通 Java 判断不行吗？

### 3.1 不使用 Lua 的问题

假设通过 Java 实现：

```
String currentVersion = redis.opsForValue().get(versionKey);

if (currentVersion.equals(oldVersion)) {
    redis.opsForValue().set(dataKey, data);
}
```

存在以下并发问题：

| 时间 | 线程 A：回填         | 线程 B：更新     |
| ---- | -------------------- | ---------------- |
| T1   | 检查 version=5，通过 |                  |
| T2   | 线程 A 暂停          |                  |
| T3   |                      | MySQL 更新并提交 |
| T4   |                      | version 自增为 6 |
| T5   |                      | 删除缓存         |
| T6   | 执行 SET，写入旧值   |                  |
| T7   | 最终 Redis 保存旧值  |                  |

这里的根本问题是：

版本检查与缓存写入属于两次独立的 Redis 操作，无法保证原子性。

即使 `GET` 和 `SET` 分别是原子命令，也不代表它们组合起来是原子的。

这种问题称为 TOCTOU（Time Of Check To Time Of Use），即检查与实际使用之间的竞态问题。

### 3.2 使用 Lua 实现原子校验

```
-- KEYS[1]：version Key
-- KEYS[2]：data Key
-- ARGV[1]：查询前获取的 version
-- ARGV[2]：数据库查询结果 JSON
-- ARGV[3]：缓存 TTL（秒）

local currentVersion = redis.call('GET', KEYS[1]) or '0'

if currentVersion ~= ARGV[1] then
    return 0
end

redis.call('SET', KEYS[2], ARGV[2], 'EX', tonumber(ARGV[3]))

return 1
```

整个 Lua 脚本是一个原子执行单元：

```
Redis 执行 Lua
    ├── GET version
    ├── 比较 version
    └── SET data（版本一致时）
          │
          ▼
      Lua 执行结束
          │
          ▼
      处理其他客户端命令
```

其他客户端无法在 Lua 的 `GET` 与 `SET` 之间插入 `INCR`、`DEL` 等 Redis 命令。

### 3.3 更新时也使用 Lua

```
-- KEYS[1]：version Key
-- KEYS[2]：data Key

redis.call('INCR', KEYS[1])
redis.call('DEL', KEYS[2])

return 1
```

这样版本递增和缓存删除也是原子执行的。

注意，Redis Lua 的原子性指其他命令无法穿插执行，并不意味着执行出错时会自动回滚此前已经执行的写操作。

## 四、Lua 到底锁住了什么？

这是 Redis Lua 原子性最容易理解错误的地方。

Lua 既不是给特定 Key 加锁，也不是给 Java 线程加锁，而是独占 Redis 实例的命令执行过程。

### 4.1 Redis 的执行模型

Redis 请求处理可以分为三个阶段：

客户端

GET / SET / INCR / EVAL

网络 I/O 阶段

Socket、epoll、请求接收与响应发送

命令执行阶段（主线程）

执行 GET、SET、INCR、Lua 等操作

返回执行结果

写入 Socket，返回客户端

其中：

- 网络 I/O：负责客户端网络请求的接收、读取与响应发送。
- 命令执行：负责真正读取、修改 Redis 数据。
- Lua：运行在 Redis 命令执行过程中，保证脚本内操作不被其他客户端命令插队。

Redis 6.0 开始引入可选的 I/O 多线程，用于辅助处理部分网络读写工作。但普通命令及 Lua 执行仍主要由主线程串行处理。

### 4.2 Lua 执行期间可以进行什么操作？

| 操作                               | 是否可以进行               |
| ---------------------------------- | -------------------------- |
| 执行其他客户端的 GET               | 不可以，需要等待           |
| 执行其他客户端的 SET               | 不可以，需要等待           |
| 执行其他客户端的 INCR              | 不可以，需要等待           |
| 执行其他 Lua 脚本                  | 不可以，需要等待           |
| 操作另一个无关的 Key               | 同实例内也需要等待         |
| 内核接收网络数据                   | 可以                       |
| Redis I/O 工作线程处理部分网络任务 | 可能继续，取决于配置和阶段 |
| 后台持久化子进程执行工作           | 可以继续                   |

Lua 阻塞的是同一个 Redis 实例的其他命令执行，不是冻结整个操作系统或 Redis 的所有后台活动。

Redis Cluster 的其他节点也不会因为某个节点执行 Lua 而全部停止。

### 4.3 Redis 的 epoll 会停止吗？

需要区分两个概念：

- `epoll`：Linux 的 I/O 多路复用机制，用来检测 Socket 的就绪事件。
- Redis 事件循环：负责处理这些事件并执行对应请求。

Lua 执行期间：

1. 内核仍然可以接收网络数据并维护 Socket 缓冲区。
2. Redis 主线程不能返回事件循环去处理其他普通命令。
3. 即使启用了 I/O 多线程，也不能认为所有网络请求都能持续被处理。
4. 脚本过长会造成命令排队、请求延迟增加，严重时网络缓冲区也会积压。

所以，I/O 多路复用仍是底层机制，但 Redis 主线程执行 Lua 时无法像空闲时那样继续调度其他请求。

## 五、Lua 会不会降低 Redis 性能？

会产生阻塞，但不一定降低整体性能。

关键在于脚本执行时间和节省的网络往返成本。

### 5.1 Java 多次操作

```
Java
  │
  ├── GET version ──→ Redis
  │
  ←── 返回版本号 ────
  │
  ├── Java 比较版本
  │
  └── SET data ────→ Redis
```

存在两个问题：

- 需要多次网络往返。
- 检查和写入之间可能被插队。

### 5.2 Lua 一次执行

```
Java
  │
  └── EVALSHA ─────→ Redis
                        │
                        ├── GET version
                        ├── 比较版本
                        └── SET data
  ←── 返回结果 ──────────
```

优势是：

- 只需要一次网络往返。
- 在 Redis 内部直接完成判断。
- 不存在检查与写入之间的竞态问题。

对于 `GET + 比较 + SET` 这样的短 Lua 脚本，执行开销通常较小，减少网络往返后，整体请求耗时还可能更低。

### 5.3 哪些 Lua 操作有性能风险？

| 操作             | 性能影响         |
| ---------------- | ---------------- |
| GET + 比较 + SET | 通常很小         |
| INCR + DEL       | 通常很小         |
| 遍历几万个 Key   | 可能明显阻塞     |
| 大量循环计算     | 可能严重阻塞     |
| 大集合复杂操作   | 可能造成较大延迟 |

例如：

```
for i = 1, 10000000 do
    -- 长时间计算
end
```

这种脚本可能长期占据 Redis 命令执行线程，导致其他客户端请求排队。

因此：

Lua 应只负责短时间、必须原子执行的操作，不要把复杂业务逻辑放进去。

尤其不要试图将 MySQL 查询和 Redis 回填的整个过程都包在 Lua 中。

## 六、Spring Boot + MyBatis 实现示例

### 6.1 Redis Lua 脚本

```
resources/lua/cache-fill.lua
local current = redis.call('GET', KEYS[1]) or '0'

if current ~= ARGV[1] then
    return 0
end

redis.call('SET', KEYS[2], ARGV[2], 'EX', tonumber(ARGV[3]))

return 1
resources/lua/cache-invalidate.lua
redis.call('INCR', KEYS[1])
redis.call('DEL', KEYS[2])

return 1
```

### 6.2 Java Service 示例

假设已有 `User` 实体和 `UserMapper`，下面省略 import 与异常类型定义。

```
@Service
public class UserService {

    private final StringRedisTemplate redis;
    private final UserMapper userMapper;
    private final ObjectMapper objectMapper;

    private final DefaultRedisScript<Long> fillScript;
    private final DefaultRedisScript<Long> invalidateScript;

    public UserService(
            StringRedisTemplate redis,
            UserMapper userMapper,
            ObjectMapper objectMapper) {

        this.redis = redis;
        this.userMapper = userMapper;
        this.objectMapper = objectMapper;

        fillScript = new DefaultRedisScript<>();
        fillScript.setLocation(
            new ClassPathResource("lua/cache-fill.lua")
        );
        fillScript.setResultType(Long.class);

        invalidateScript = new DefaultRedisScript<>();
        invalidateScript.setLocation(
            new ClassPathResource("lua/cache-invalidate.lua")
        );
        invalidateScript.setResultType(Long.class);
    }

    // 查询用户
    public User getUser(Long id) throws JsonProcessingException {

        String dataKey = "user:{" + id + "}:data";
        String versionKey = "user:{" + id + "}:version";

        for (int i = 0; i < 3; i++) {

            // 1. 读取缓存
            String cache = redis.opsForValue().get(dataKey);

            if (cache != null) {
                return objectMapper.readValue(cache, User.class);
            }

            // 2. 记录查询前的版本
            String version = redis.opsForValue().get(versionKey);

            if (version == null) {
                version = "0";
            }

            // 3. 查询 MySQL 主库
            User user = userMapper.selectById(id);

            if (user == null) {
                return null;
            }

            // 4. Lua 原子检查版本并回填
            Long result = redis.execute(
                fillScript,
                List.of(versionKey, dataKey),
                version,
                objectMapper.writeValueAsString(user),
                "600"
            );

            if (Long.valueOf(1).equals(result)) {
                return user;
            }

            // 版本发生变化，重新尝试
        }

        // 重试失败，直接查询主库，不回填
        return userMapper.selectById(id);
    }

    // 更新用户
    @Transactional
    public void updateUser(Long id, String name) {

        int rows = userMapper.updateName(id, name);

        if (rows != 1) {
            throw new IllegalStateException("用户更新失败");
        }

        String dataKey = "user:{" + id + "}:data";
        String versionKey = "user:{" + id + "}:version";

        // 数据库提交成功后，执行缓存失效
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {

                @Override
                public void afterCommit() {
                    redis.execute(
                        invalidateScript,
                        List.of(versionKey, dataKey)
                    );
                }
            }
        );
    }
}
```

这里的 `getUser()` 应从新的读操作上下文访问主库，避免沿用旧的 RR 事务快照。

该代码是核心流程示例，生产环境还需要完善 Redis 异常重试、空值缓存、版本恢复与可靠失效机制。

## 七、必须注意的边界问题

### 7.1 Redis 缓存为空时，怎么校验 version？

因为 version 是独立的 Key：

```
user:{1}:data       → null
user:{1}:version    → 6
```

即使数据缓存被删除，version 仍然存在，因此可以校验。

但如果 version 也不存在，就不能无条件假设它是 0。

只有确认它从未初始化，且整个系统保证版本不会丢失时，这样处理才安全。

### 7.2 查询之前，其他线程先修改了 version 怎么办？

假设：

```
线程 A：读取 version = 5
线程 B：更新 MySQL
线程 B：INCR version → 6
线程 A：查询 MySQL
线程 A：Lua 校验 version
```

因为 `5 != 6`，所以拒绝回填。

即使线程 A 读到的是新数据，也可能因为版本变化而重新查询。这是安全优先的保守做法。

### 7.3 Lua 校验成功之后，其他线程再修改 version 呢？

```
线程 A：
    Lua 校验 version = 5
    SET data 成功

线程 B：
    MySQL 更新并提交
    INCR version → 6
    DEL data
```

线程 B 会删除 A 写入的缓存。

因此，只要更新后的失效操作成功执行，旧数据不会因为线程 A 的回填而长期残留。

### 7.4 MySQL 更新和 Redis 更新能保证原子性吗？

不能。

即使使用 Lua：

```
线程 B：MySQL 更新并提交
          ↓
       存在时间窗口
          ↓
线程 B：Redis INCR + DEL
```

如果线程 A 恰好在这个窗口里完成缓存读取或回填，仍可能短暂读取旧值。

另外，如果 MySQL 已经提交，但 Redis 更新失败，缓存可能持续不一致。

因此需要结合：

- 重试机制。
- Outbox 或 Binlog CDC 等可靠变更通知。
- 缓存 TTL 兜底。
- 对强一致性要求高的业务，采用更严格的数据访问与同步设计。

### 7.5 Version Key 会不会占用大量内存？

会。

独立的 version Key 通常需要长期保存，如果维护的业务对象很多，就会产生额外内存开销。

同时必须考虑 Redis 内存淘汰、持久化恢复与版本号重置问题。

对于强依赖版本顺序的场景，可以考虑将 MySQL 的 `version` 字段作为权威版本来源，Redis 维护独立版本水位。但依然需要解决数据库更新与 Redis 同步之间的可靠性问题。

## 八、面试高频问题总结

| 面试问题                               | 标准回答                                                     |
| -------------------------------------- | ------------------------------------------------------------ |
| 为什么缓存会出现旧值覆盖？             | 并发查询线程可能在数据库更新和缓存删除后，才将之前查到的旧值回填 |
| Version 有什么作用？                   | 标记数据变更，判断查询期间是否发生更新                       |
| Version 通过删除更新吗？               | 不是，一般使用 INCR 自增                                     |
| 缓存不存在还能校验吗？                 | 可以，version 使用独立 Key，不跟随 data 删除                 |
| 为什么需要 Lua？                       | 保证版本检查与缓存写入原子执行                               |
| Redis Lua 锁住的是哪个 Key？           | 不锁特定 Key，而是保证同实例的其他命令不能插入脚本执行过程   |
| Lua 会阻塞 Redis 吗？                  | 会阻塞其他命令执行，因此脚本必须尽量短                       |
| Redis I/O 会完全停止吗？               | 不会冻结内核网络 I/O，但主线程无法正常继续调度其他请求       |
| Redis I/O 多线程能并行执行 Lua 吗？    | 不能，常规 Redis Lua 命令执行仍是串行的                      |
| Lua 可以保证 MySQL 和 Redis 强一致吗？ | 不能，只保证 Redis 内部操作的原子性                          |

## 九、最终知识点串联

```
Redis Cache Aside
       │
       ▼
并发查询产生旧值回填问题
       │
       ▼
引入独立 version Key
       │
       ├── 查询：记录 version
       │
       └── 更新：INCR version + DEL data
       │
       ▼
回填前比较 version
       │
       ▼
发现 GET + SET 存在并发竞态
       │
       ▼
使用 Lua 保证原子性
       │
       ▼
Lua 独占 Redis 命令执行过程
       │
       ├── 不锁 Java 线程
       ├── 不锁特定 Key
       ├── 阻塞其他命令执行
       └── 不冻结底层网络 I/O
       │
       ▼
脚本应尽量简短
       │
       ▼
解决旧查询迟到回填问题
       │
       ▼
仍需处理 MySQL 与 Redis 跨系统一致性
```

核心结论：

1. Version 解决的是数据版本识别问题：判断查询期间是否发生数据变更。
2. Lua 解决的是原子性问题：防止版本校验和缓存写入之间被其他 Redis 命令插队。
3. Redis 命令执行与网络 I/O 是不同概念：Lua 阻塞命令执行，但不冻结整个网络系统。
4. Lua 的性能取决于执行时间：简单脚本一般开销较低，长脚本会拖慢 Redis。
5. Version + Lua 不等于强一致性：MySQL 与 Redis 之间仍存在提交窗口、异常失败以及版本信息丢失等问题。

最终可以将这套方案概括为：

使用 Redis 独立递增版本号检测并发数据变更，通过 Lua 原子完成版本校验和缓存回填，防止旧查询结果在版本更新后重新污染 Redis 缓存。