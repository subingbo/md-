# Kafka 消息一致性与消费幂等性——Java 后端面试笔记

> 场景：订单服务将订单写入 MySQL 后发送 `OrderCreated` 消息到 Kafka，库存服务消费消息并扣减库存。本文整理自一次逐层追问的面试练习，重点是 **数据库与消息队列最终一致性**、**重复消费的业务幂等性**，以及 **唯一约束、Redis 分布式锁、MySQL 悲观锁的区别和使用边界**。
>
> 主要讨论 MySQL InnoDB 和 Kafka；代码为原理演示，生产环境还需处理超时、失败重试、监控、死信和业务补偿。

## 一、先区分三个容易混淆的问题

| 问题 | 本质 | 主要手段 |
| --- | --- | --- |
| 订单已写库，但消息没发出去 | MySQL 与 Kafka **跨系统最终一致性** | Transactional Outbox（本地消息表）、CDC / Binlog、可靠重试 |
| 相同 Kafka 消息被消费多次，库存重复扣减 | **业务幂等性** | 业务唯一 ID + 数据库唯一约束 + 本地事务；或状态机条件更新 |
| 不同订单同时修改同一商品库存，产生并发冲突 | **并发控制** | 原子条件 `UPDATE`、行锁、必要时 `SELECT ... FOR UPDATE` |

**一条核心结论：互斥 ≠ 幂等 ≠ 原子性。** 互斥关注“是否同时执行”；幂等关注“同一业务操作重复执行是否产生额外效果”；原子性关注“同一事务内的多个数据库修改是否一起提交或回滚”。

## 二、问题一：MySQL 与 Kafka 如何保证最终一致性？

假设订单创建需要：（1）插入 `orders` 记录；（2）发送 `OrderCreated` 到 Kafka。面试官给出两种异常：

- **场景 A**：MySQL 已提交，应用在发 Kafka 之前宕机，出现“有订单、无消息”。
- **场景 B**：Kafka 消息发送成功，MySQL 却回滚，出现“有消息、无订单”。

仅将两步写在 `@Transactional` 方法中，**不能**使两者自动组成同一个事务：Spring 的常规 MySQL 本地事务不会自动将 Kafka broker 纳入原子提交范围。即使使用 Kafka 自身的事务能力，也不能直接把任意 MySQL 提交与 Kafka 提交变成一个跨系统原子事务。

### 方案 A：Transactional Outbox（本地消息表）+ 独立投递器

**推荐作为通用业务方案。** 在同一个 MySQL 本地事务中同时写订单和一条待发送消息：

```sql
BEGIN;

INSERT INTO orders (order_id, status)
VALUES (1001, 'CREATED');

INSERT INTO outbox_event (event_id, aggregate_id, event_type, payload, status)
VALUES ('evt-1001-created', 1001, 'OrderCreated', '{"orderId":1001}', 'PENDING');

COMMIT;
```

因为订单记录和 Outbox 记录在**同一个 MySQL 事务**里，数据库可以保证两者一起提交或回滚。提交之后，由独立投递器读取待发送事件，发布到 Kafka；成功拿到 broker 确认后再记录投递进度，失败就重试。投递器可采用定时扫描，也可通过 CDC 读取 Outbox 表的 Binlog，避免高频轮询。

```text
订单服务
   └─ MySQL 本地事务
       ├─ INSERT orders
       └─ INSERT outbox_event
              │ COMMIT 后两者同时存在
              ▼
        投递器 / CDC
              ▼
            Kafka
              ▼
         库存消费者
```

**注意“投递成功”和“标记已投递”之间仍有故障窗口。** Kafka 已收消息，但投递器还没保存进度就宕机，恢复后可能再次投递。因此 Outbox + Kafka 通常提供 **至少一次投递**，需要消费端幂等，不能单靠生产端认为“永不重复”。

### 方案 B：订阅订单表 Binlog（CDC）

面试时你提出“订阅 MySQL Binlog，在成功提交后发送消息”，方向正确。Debezium、Canal 等 CDC 组件可以持续读取数据库提交后的变更并转为消息，订单服务即使在提交后宕机，变更仍可由 CDC 继续处理。

不过，**直接把任意 `orders` 变更翻译成 `OrderCreated` 并不天然可靠**：需要定义哪些变更代表事件、怎样区分插入与状态更新、如何管理 schema 变更、如何处理重放和重复投递。实践中常采用 **Outbox 表 + CDC**，把“业务事件”明确记录在数据库中，再从 Binlog 发布到 Kafka。

**面试总结：**“我会把订单数据与 Outbox 事件放入同一 MySQL 事务，提交后使用可靠投递器或 CDC 发布 Kafka。投递采用重试和可恢复的进度管理，消费端负责幂等，从而实现最终一致性。”

## 三、问题二：Kafka 为什么会重复消费？

假设 CDC 或生产者已经成功把事件发到 Kafka，但还未保存自己的进度便崩溃，恢复后可能再次发出同一业务事件；另外，Kafka 消费者在**业务处理成功，但 offset 尚未提交**时宕机，恢复后也可能重新消费。

```text
消费者读取 event_id = evt-1001-deduct
             ↓
          扣库存成功
             ↓
       尚未确认消费进度
             ↓
           进程宕机
             ↓
         Kafka 重新投递
```

所以要区别 **消息传递次数** 与 **业务生效次数**：即便同一条业务事件到达多次，也应该只让扣库存动作生效一次。Kafka 的幂等生产者和事务机制能处理其特定范围内的重复与 Kafka 事务处理，但**不能自动保证外部 MySQL 业务写入恰好一次**。

> 一般情况下，同一个消费组中，同一分区会分配给一个消费者成员；面试中的“两实例同时处理同一事件”可以来自并发处理实现、重平衡时的重叠处理、重试机制等。数据库仍应有独立幂等防护。

## 四、首选：操作表 + 唯一索引 + MySQL 本地事务

这是本次面试中你首先提出、随后完善的消费端方案。给每一次**业务操作**分配稳定的唯一标识，比如 `order:1001:deduct`。它必须在重投递时保持不变，不能每次消费临时生成新 UUID；同一订单若存在不同合法操作，还要纳入操作类型或事件 ID 以避免误去重。

### 4.1 表设计

```sql
CREATE TABLE consumed_operation (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    operation_id VARCHAR(128) NOT NULL,
    UNIQUE KEY uk_operation_id (operation_id)
) ENGINE = InnoDB;

CREATE TABLE stock (
    product_id BIGINT PRIMARY KEY,
    quantity INT NOT NULL
) ENGINE = InnoDB;
```

### 4.2 消费时执行的事务

```sql
BEGIN;

-- 1. 先竞争唯一业务操作 ID
INSERT INTO consumed_operation (operation_id)
VALUES ('order:1001:deduct');

-- 2. 仅插入成功的事务继续扣库存
UPDATE stock
SET quantity = quantity - 1
WHERE product_id = 10 AND quantity >= 1;

-- 必须检查影响行数是否为 1；不是则 ROLLBACK 并按缺货/重试策略处理
COMMIT;
```

插入记录、扣库存必须同事务执行。若扣库存失败导致回滚，唯一操作记录也回滚，下一次消费仍有机会正确重试。若处理已成功并提交，再收到相同 `operation_id`，`INSERT` 会遇到唯一键冲突，可以将其视为“**这笔操作已经处理过**”，跳过库存修改并正常确认消息。需要区分重复键异常与数据库连接失败、死锁等普通异常，不能把所有错误都当成功。

### 4.3 为什么不用“先 SELECT，再 INSERT”？

```text
事务 A：SELECT 查询操作不存在
事务 B：SELECT 查询操作不存在
事务 A：执行扣减
事务 B：执行扣减
```

普通 `SELECT` 与后续扣减之间没有原子性保障，即使在 RR 隔离级别下，也不能用“我查不到”作为独占处理资格。**唯一索引**让两个并发插入不可能都成功提交：当 A 插入尚未提交时，B 对同一唯一键的插入通常要等待冲突结果；A 提交后 B 冲突，A 回滚后 B 可以继续成功。由此自然解决“两个消费者都看见不存在”的竞态。

### 4.4 简化的 Java 业务边界

```java
@Transactional
public void deductOnce(String operationId, long productId, int amount) {
    // 数据库必须有 UNIQUE(operation_id)
    operationMapper.insert(operationId); // 重复键 -> 已处理，交由调用层区别处理

    int updated = stockMapper.decreaseIfEnough(productId, amount);
    if (updated != 1) {
        throw new IllegalStateException("库存不足或商品不存在");
    }
}
```

对应 SQL：

```sql
UPDATE stock
SET quantity = quantity - #{amount}
WHERE product_id = #{productId}
  AND quantity >= #{amount};
```

调用层应将**明确识别的重复键冲突**作为已处理，且只在本地事务已成功提交或确认已经处理后提交 Kafka 消费进度；其他失败应保持可重试或进入明确的失败处理流程。

## 五、Redis：要区分“去重标记”和“分布式锁”

两者都可能使用 `SET NX`，但语义完全不同。

### 5.1 Redis 去重标记（不推荐单独作为扣库存正确性依据）

```redis
SET processed:order:1001 1 NX EX 86400
```

它能原子地判断 Redis Key 是否首次出现，适合作为短期过滤。但如果 Redis 设置成功后，MySQL 还没扣库存就宕机，重试时会以为操作已经完成，造成**漏扣**；如果先扣 MySQL 后写 Redis，宕机又可能**重复扣**。此外 Redis Key 过期/丢失后，也无法保证永久幂等。因此 Redis 标记与 MySQL 更新**不能依靠两个独立命令保证原子提交**，关键业务应以 MySQL 状态或唯一约束为最终判定。

### 5.2 Redis 分布式互斥锁

```redis
SET lock:order:1001 <random-token> NX PX 10000
```

- `NX`：只有锁不存在才能加锁成功，同一时刻争夺有效锁的客户端只有一个成功。
- `PX 10000`：设置 10 秒过期，防止持锁者宕机导致无限等待。
- `<random-token>`：本次持锁的唯一身份，不是业务操作 ID；不能让不同持锁者共用固定 token。

获取锁成功的消费者进入业务逻辑；拿不到锁代表**别人正在处理或锁尚未释放**，并不代表业务已经处理成功，不能因此直接确认 Kafka 消息。

正确方向是：**拿锁 → 开启 MySQL 本地事务 → 检查业务是否已处理 → 扣库存并写幂等状态 → 提交事务 → 安全释放锁 → 确认 Kafka 消费**。即使有 Redis 锁，仍然需要 MySQL 幂等，因为 A 完成并解锁后，B 可以稍后重新拿锁，对同一业务事件重复执行。

#### Redis 锁的三个典型风险

**风险 1：业务超过锁 TTL。** A 持锁 10 秒，但执行 15 秒；第 10 秒锁过期，B 可能拿到新锁，与 A 并发运行。可以使用续期机制（如 Redisson watchdog），但在长时间停顿或网络分区等情况下仍不能绝对消除陈旧持锁者；要求更强保证时需要数据库约束，或让业务存储检查 fencing token。

**风险 2：误删别人的锁。** A 的锁过期，B 拿到同名锁；A 恢复执行 `DEL lock:order:1001` 会误删 B 的锁。应在 Redis 内原子地“验证 token → 删除”，常见 Lua：

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
```

**风险 3：Redis 锁与 MySQL 事务无原子关系。** Redis 的互斥不能替代数据库事务；Redis 主从故障切换等也可能影响单实例锁的互斥保障。对于扣库存这样的关键写入，数据库仍需兜底约束与幂等状态。

## 六、MySQL 悲观锁：`SELECT ... FOR UPDATE`

**悲观锁的主要作用是让并发事务有序地读取和修改同一份已有数据，而不是自动记录“某条消息是否已消费”。** 常用 InnoDB 锁定读：

```sql
BEGIN;

SELECT status
FROM stock_task
WHERE order_id = 1001
FOR UPDATE;
```

如果 `stock_task` 中确实已有 `order_id=1001` 记录，且通过唯一索引等值命中，事务会锁定该记录。A 锁住它后，B 对同一行的 `FOR UPDATE` 需要等待；A 在事务中检查 `PENDING`、扣库存、更新为 `DONE`，提交后 B 再获得锁，读取到 `DONE`，跳过扣减。

完整示意（前提：任务记录**事先存在**）：

```sql
BEGIN;

SELECT status
FROM stock_task
WHERE order_id = 1001
FOR UPDATE;

-- 应用层判断：仅当 status = 'PENDING' 才继续
UPDATE stock
SET quantity = quantity - 1
WHERE product_id = 10 AND quantity >= 1;
-- 检查影响行数是否为 1，否则 ROLLBACK

UPDATE stock_task
SET status = 'DONE'
WHERE order_id = 1001;

COMMIT;
```

### 6.1 为什么不一定要提前有“操作记录”？

**被锁定的记录要有明确对象，但它可以是另一张表的已存在记录。** 比如幂等操作表还没有 `order_id=1001`，却已经存在 `orders.id=1001`，可以在事务里先 `SELECT ... FROM orders WHERE id=1001 FOR UPDATE` 锁住订单，再检查/写入操作表并扣库存。此时先存在的是**订单记录**，而非操作记录。

如果执行 `SELECT ... FOR UPDATE` 查询的目标行**不存在**，不能声称“已经锁住了该行”。在 InnoDB RR 下，依索引和谓词，可能有 Gap Lock 等范围锁；间隙锁本身并不按“同一订单号互斥”那样工作，多个事务都可能先查到不存在，之后插入竞争或死锁。所以对于“记录尚不存在”的业务幂等，优先使用 **唯一索引约束插入**，而不是依赖空结果的 `FOR UPDATE`。

### 6.2 悲观锁与 Kafka 幂等的关系

- **同一商品、不同订单**同时扣库存：需要确保数据并发修改正确。可以通过原子条件 `UPDATE` 实现，或在需要“先读后判断再写”时使用悲观锁。
- **同一订单消息重复投递**：即使两个消费者因悲观锁而先后执行，第二个也可能再次扣库存。必须有 `DONE` 状态或唯一操作记录，才能跳过重复业务。
- **消费者在事务提交前宕机**：MySQL 连接断开后，未提交事务会回滚并释放锁；如果事务已提交但 Kafka offset 未提交，消息会重投，此时应通过已提交的幂等状态跳过。

## 七、Redis、悲观锁、唯一索引：最后对比

| 维度 | Redis 分布式锁 | MySQL `FOR UPDATE` | MySQL 唯一操作表 |
| --- | --- | --- | --- |
| 核心目标 | 应用层短期互斥 | 数据库事务内并发互斥 | 同一业务操作只生效一次 |
| 锁/约束所在 | Redis Key | InnoDB 锁 | `UNIQUE(operation_id)` |
| 必须先有操作记录？ | 否 | 不一定，但需要明确可锁定的已有对象；不存在的行不能简单按行锁处理 | 否 |
| 业务成功后长期保留记忆？ | 否，锁会释放 | 否，事务结束锁会释放 | 是，已提交的操作记录持续保留 |
| 与 MySQL 库存更新同事务？ | 否 | 是 | 是 |
| 能否单独保证 Kafka 消费幂等？ | **不能** | **不能** | **可以作为可靠核心机制**（仍须正确事务与异常处理） |
| 常见风险 | 锁过期、误删、故障切换 | 锁等待、死锁、热点行 | 表增长、唯一键设计、重复异常区分 |

**推荐决策**：对于“Kafka 重复消费导致同一订单重复扣库存”，优先 **业务唯一 ID + MySQL 唯一索引 + 本地事务**；对于“不同订单同时修改同一库存”，优先 **原子条件 UPDATE**，复杂的先读后改场景再考虑 `FOR UPDATE`。Redis 锁可作为协调或减少并发进入的辅助方案，但不应该是关键数据库操作唯一的正确性保障。

## 八、面试官继续追问时的回答模板

**问：数据库提交成功、Kafka 消息未发出去怎么办？**

答：采用 Transactional Outbox，在一个 MySQL 事务里同时保存订单和业务事件；事务提交后由独立投递器或 CDC 订阅 Outbox Binlog 推送 Kafka。投递失败可重试，允许重复到达，因此消费端必须幂等。

**问：Kafka 消息重复投递，库存怎么只扣一次？**

答：每条业务操作携带稳定的 `operation_id`。在 MySQL 同一事务中先插入唯一操作记录，再进行带库存充足条件的 `UPDATE`，检查影响行数并提交。唯一键冲突意味着已经处理过，跳过业务并确认消费；真正失败回滚并重试。

**问：为什么 Redis `SET NX` 不能单独保证幂等？**

答：Redis 标记与 MySQL 扣库存不在一个本地事务中。先标记后扣库存可能漏扣，先扣库存后标记可能重复扣。Redis 锁还有过期并发、误删锁和故障切换问题，必须用 MySQL 业务状态或唯一约束兜底。

**问：悲观锁和操作表是不是同一回事？**

答：不是。`FOR UPDATE` 用于在事务持锁期间协调并发，锁结束后不会记住某操作做过；唯一操作表用持久唯一 ID 记录某业务已经生效。二者可以组合，但消息消费幂等不一定需要显式悲观锁。

**问：`FOR UPDATE` 必须要求操作表有记录吗？**

答：不必须。可以锁已经存在的订单行再处理操作表。但锁定读命中已有唯一索引记录，与查询不存在的行是不同情况；对不存在的行不能简单当作持有专属行锁，通常优先唯一索引竞争插入。

## 九、最后记忆

> **生产端防漏发：Outbox + CDC / 重试。消费端防重复：业务唯一 ID + 唯一索引 + 本地事务。业务数据防并发冲突：条件 UPDATE / 必要时 FOR UPDATE。Redis 锁只解决短期互斥，不等于幂等。**

复习时重点反问自己：消息重发、消费者重启、MySQL 回滚、Kafka offset 未提交、两个消费者同时执行、Redis 锁过期，这六种情况下，业务最终是否仍能正确？
