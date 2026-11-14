# MySQL InnoDB 锁机制与死锁：完整面试笔记

> 适用范围：MySQL 8.x、InnoDB、Spring Boot + JdbcTemplate。重点理解 **S/X 锁的兼容性、`FOR UPDATE`、死锁形成、统一加锁顺序、转账代码和事务重试**。

## 一、先记住 6 个结论

1. **`SELECT ... FOR SHARE` 获取共享锁（S 锁）**；MySQL 旧语法 `LOCK IN SHARE MODE` 也表示共享锁。
2. **`SELECT ... FOR UPDATE` 获取排他锁（X 锁）**，不是共享锁。
3. **`UPDATE` 修改记录需要 X 锁**，不是“共享修改”。
4. 不同事务对**同一记录**的 **S + S 兼容**；**S + X、X + X 冲突**。
5. **转账方向与加锁顺序是两回事**：无论 A→B 还是 B→A，都可以先锁较小 ID，再锁较大 ID，之后再按实际方向扣款、入账。
6. **死锁 1213 通常回滚被选中的整个事务**；**锁等待超时 1205 在 InnoDB 默认配置下只回滚当前语句**。业务上应保证失败后结束/回滚事务，再进行有限次数的**完整事务重试**。

---

## 二、共享锁与排他锁是什么？

### 2.1 S 锁（Shared Lock，共享锁）

多个事务可以同时对同一条记录取得 S 锁，适合需要锁定读取、同时不允许其他事务修改该记录的场景。

```sql
-- 事务 T1
BEGIN;
SELECT balance FROM account WHERE id = 1 FOR SHARE;
-- 此时 T1 持有该记录的 S 锁
```

```sql
-- 事务 T2
BEGIN;
SELECT balance FROM account WHERE id = 1 FOR SHARE;
-- 与 T1 的 S 锁兼容，通常可以立即成功
```

**注意**：S 锁的“共享”是**共享读取时的锁权限**，并不允许两个事务共享写入权限。

### 2.2 X 锁（Exclusive Lock，排他锁）

一个事务对记录持有 X 锁时，其他事务不能在同一记录上再取得冲突的 S/X 锁。

```sql
-- 事务 T1
BEGIN;
SELECT balance FROM account WHERE id = 1 FOR UPDATE;
-- 此时 T1 持有该记录的 X 锁
```

```sql
-- 事务 T2
BEGIN;
UPDATE account SET balance = 500 WHERE id = 1;
-- 需要 X 锁，通常等待 T1 释放锁
```

### 2.3 不同事务之间的锁兼容矩阵

| 当前事务已持有 | 另一个事务请求 S 锁 | 另一个事务请求 X 锁 |
|---|---|---|
| **S 锁** | ✅ 兼容 | ❌ 冲突，需要等待 |
| **X 锁** | ❌ 冲突，需要等待 | ❌ 冲突，需要等待 |

这个矩阵描述的是**不同事务对同一锁定对象**的关系。同一事务可以在已有 S 锁的基础上申请 X 锁，但如果其他事务仍持有 S 锁，锁升级就可能等待甚至引起死锁。

### 2.4 普通 SELECT 会被 X 锁挡住吗？

通常不会：普通 `SELECT` 在 InnoDB 的 `READ COMMITTED` / `REPEATABLE READ` 隔离级别下通常是 **MVCC 快照读**，不主动申请记录 S 锁；即使另一个事务持有 X 锁，也通常能读到满足可见性规则的历史已提交版本。

```sql
SELECT balance FROM account WHERE id = 1;             -- 普通快照读（常见情况）
SELECT balance FROM account WHERE id = 1 FOR SHARE;   -- 当前读 + S 锁
SELECT balance FROM account WHERE id = 1 FOR UPDATE;  -- 当前读 + X 锁
```

> “普通 SELECT 不加行锁”不是无条件的全局规则。某些隔离级别、SQL 形式或数据库配置可能改变其行为。这里讨论最常用的 InnoDB RC/RR 场景。

---

## 三、为什么 `UPDATE` 与共享锁冲突？

`UPDATE` 在更改某条记录前必须获取相应的 **X 锁**。如果别的事务已经对该记录持有 S 锁，写入就要等待。

```text
T1: SELECT ... WHERE id=1 FOR SHARE;   → 获得 S 锁
T2: UPDATE account ... WHERE id=1;      → 请求 X 锁，等待 T1
T1: COMMIT;                            → 释放 S 锁
T2: UPDATE 继续执行                     → 获得 X 锁
```

另一个易考场景：**两个事务都先读共享锁，随后都要写入**。

| 时间 | 事务 T1 | 事务 T2 |
|---|---|---|
| ① | 对 id=1 获取 S 锁，成功 | |
| ② | | 对 id=1 获取 S 锁，成功 |
| ③ | 尝试 UPDATE，需要 X 锁；等待 T2 的 S 锁 | |
| ④ | | 尝试 UPDATE，需要 X 锁；等待 T1 的 S 锁 |
| ⑤ | **T1 等 T2，T2 等 T1：形成死锁** | |

这也是为什么**先读后改**的事务若明确知道要修改数据，常选用 `FOR UPDATE`，而不是先 `FOR SHARE` 后升级。

---

## 四、转账场景：死锁究竟怎么产生？

假设账户表：

```sql
CREATE TABLE account (
    id BIGINT PRIMARY KEY,
    balance DECIMAL(18, 2) NOT NULL
) ENGINE = InnoDB;

INSERT INTO account(id, balance)
VALUES (3, 1000.00), (8, 500.00);
```

### 4.1 错误示例：按转账方向加锁

**T1：账户 8 → 3，转 100 元**

```sql
BEGIN;
SELECT balance FROM account WHERE id = 8 FOR UPDATE; -- 先持有 id=8
SELECT balance FROM account WHERE id = 3 FOR UPDATE; -- 等待 id=3
```

**T2：账户 3 → 8，转 200 元**

```sql
BEGIN;
SELECT balance FROM account WHERE id = 3 FOR UPDATE; -- 先持有 id=3
SELECT balance FROM account WHERE id = 8 FOR UPDATE; -- 等待 id=8
```

如果两事务交错执行，形成等待环：

```text
T1 持有 id=8 的 X 锁 ──等待──> T2 持有 id=3 的 X 锁
          ^                                |
          └──────────── 等待 ─────────────┘
```

准确地说：T1 等 T2 释放 **id=3**，T2 等 T1 释放 **id=8**。它们彼此等待，InnoDB 检测到死锁后会选择其中一个事务作为牺牲者回滚。

### 4.2 正确思路：按照**资源 ID**，不按照转账方向

无论传入：

```java
transfer(8L, 3L, new BigDecimal("100")); // 8 → 3
transfer(3L, 8L, new BigDecimal("200")); // 3 → 8
```

先计算：

```java
long firstId = Math.min(fromId, toId);
long secondId = Math.max(fromId, toId);
```

结果都是：

```text
firstId = 3
secondId = 8

统一加锁顺序：id=3 → id=8
```

**转账方向并没有被修改**。只有**获取锁的次序**变成统一的；之后仍然按 `fromId` 扣钱，按 `toId` 加钱。

### 4.3 为什么统一顺序有用？

如果 T1 先锁定 id=3，那么 T2 在申请 id=3 时就必须等待，**无法先跑到 id=8 并持锁后反过来等 T1**。于是不存在由这两条记录按反序获取 X 锁构成的等待环。

```text
T1（8 → 3）                       T2（3 → 8）
获得 id=3 的 X 锁                  等待 id=3 的 X 锁
获得 id=8 的 X 锁                      ...
扣 id=8、加 id=3                     ...
COMMIT，释放锁                       ...
                                   获得 id=3 的 X 锁
                                   获得 id=8 的 X 锁
                                   扣 id=3、加 id=8
                                   COMMIT
```

**理论关键**：固定的全局资源获取顺序，能破坏针对这些资源的**循环等待**条件；但无法保证系统中所有 SQL、索引、外键和其他表都绝不会死锁。

---

## 五、Spring Boot：按 ID 升序加锁，再按转账方向更新

下面使用 `JdbcTemplate` 展示最小完整业务链路。假设 `account.id` 是主键，`balance` 非空，数据库事务管理器已正确配置。

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.Objects;

@Service
public class AccountService {

    private final JdbcTemplate jdbcTemplate;

    public AccountService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // ① 参数校验：避免空值、同账户转账、负数或零金额
        Objects.requireNonNull(fromId, "fromId");
        Objects.requireNonNull(toId, "toId");
        Objects.requireNonNull(amount, "amount");
        if (fromId.equals(toId)) {
            throw new IllegalArgumentException("不能向自己转账");
        }
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("转账金额必须为正数");
        }

        // ② 无论转账方向如何，统一按账号 ID 升序加锁
        long firstId = Math.min(fromId, toId);
        long secondId = Math.max(fromId, toId);

        BigDecimal firstBalance = jdbcTemplate.queryForObject(
                "SELECT balance FROM account WHERE id = ? FOR UPDATE",
                BigDecimal.class, firstId);

        BigDecimal secondBalance = jdbcTemplate.queryForObject(
                "SELECT balance FROM account WHERE id = ? FOR UPDATE",
                BigDecimal.class, secondId);

        // ③ 加锁顺序与转账方向独立，找到真正的付款方余额
        BigDecimal fromBalance = fromId.longValue() == firstId
                ? firstBalance : secondBalance;
        if (fromBalance.compareTo(amount) < 0) {
            throw new IllegalStateException("余额不足");
        }

        // ④ 已经持有两条记录的 X 锁，再按真实方向扣款、入账
        int debited = jdbcTemplate.update(
                "UPDATE account SET balance = balance - ? WHERE id = ?",
                amount, fromId);
        int credited = jdbcTemplate.update(
                "UPDATE account SET balance = balance + ? WHERE id = ?",
                amount, toId);

        if (debited != 1 || credited != 1) {
            throw new IllegalStateException("账户更新行数异常");
        }
        // ⑤ 方法正常返回时由 Spring 提交事务，释放锁；异常则回滚
    }
}
```

### 5.1 为什么这里可以保证先锁小 ID？

因为执行的是**两次顺序调用的 SQL**：

```sql
SELECT balance FROM account WHERE id = 3 FOR UPDATE;
SELECT balance FROM account WHERE id = 8 FOR UPDATE;
```

第一条 `JdbcTemplate.queryForObject(...)` 未成功返回前，当前执行线程不会调用第二条 SQL。若第一条锁被其他事务持有，就先等待，不能越过它去获取第二把锁。

### 5.2 为什么后面的 UPDATE 不必再升序执行？

这段代码的两个 `UPDATE` 之前，事务**已经取得这两个账户记录所需的 X 锁**。只要后续更新的确只涉及已锁定的记录，就不会因为对这两个账户记录的 X 锁获取顺序相反而出现先前所述的死锁。

这不排除 `UPDATE` 访问其他索引、触发器、外键或其他表时申请额外的锁。因此“提前锁了两行”不等于“后面绝不发生死锁”。

### 5.3 `@Transactional` 的必要条件

- `FOR UPDATE` 必须在有效的数据库事务中使用；否则自动提交模式下锁可能随语句结束很快释放。
- 同一方法中多次 `JdbcTemplate` 调用必须参与**同一个事务、同一个事务绑定连接**。
- Spring 默认声明式事务通常经由代理生效：**同类内部 `this.transfer(...)` 自调用不会触发该方法上的事务代理**。
- 失败时抛出的运行时异常要传播至事务边界；不要吞掉异常后继续提交。
- 将外部支付、消息发送等不可回滚副作用与数据库事务协调，防止重试导致重复执行。

---

## 六、`ORDER BY id FOR UPDATE` 是否等于按 ID 顺序拿锁？

不能简单画等号：

```sql
SELECT * FROM account
WHERE id IN (3, 8)
ORDER BY id
FOR UPDATE;
```

`ORDER BY` 保证的是**结果输出顺序**，并不构成“数据库必然严格按 ID 从小到大获取内部锁”的一般保证。实际访问和加锁顺序还可能受执行计划、所用索引等影响。

对少量关键账户，若要明确控制获取顺序，采用**排序后逐条执行主键等值 `FOR UPDATE`** 更容易理解和审核。大量账户操作还需平衡往返次数、并发成本和整体锁策略。

---

## 七、死锁（1213）和锁等待超时（1205）

| 项目 | 死锁 | 锁等待超时 |
|---|---|---|
| 典型 MySQL 错误码 | **1213** | **1205** |
| 原因 | 出现事务之间的循环等待 | 等待某锁超过配置阈值 |
| InnoDB 常见默认处理 | 选择一个受害事务，**回滚整个受害事务** | **只回滚超时的当前语句** |
| 应用侧建议 | 重试**完整事务**，次数有限 | 显式结束/回滚失败事务；按业务决定是否重试完整事务 |

**两个重要限定**：

1. `innodb_rollback_on_timeout=ON` 会改变 1205 的回滚行为，需以环境配置为准。
2. 即使 InnoDB 默认只回滚 1205 的单条语句，Spring 的 `@Transactional` 如果收到向外传播的运行时异常，**通常会将整个 Spring 事务标记为回滚**。不要把“InnoDB 内核默认行为”与“Spring 最终事务处理”混为一谈。

辅助排查命令：

```sql
SHOW ENGINE INNODB STATUS\G
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
SHOW VARIABLES LIKE 'innodb_rollback_on_timeout';
```

MySQL 8.x 还可结合 `performance_schema.data_locks` 与 `performance_schema.data_lock_waits` 观察锁与等待关系（需要相应权限）。

---

## 八、Spring Boot 如何正确重试死锁？

原则：**在事务外层循环，每一次都重新调用一个通过 Spring 代理开启事务的方法**。不要在已经失败的同一个事务内部重试部分 SQL。

以下写法将“事务业务”和“重试入口”拆成两个 Bean。第 5 节的 `AccountService.transfer()` 保留 `@Transactional`。

```java
import org.springframework.dao.DataAccessException;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.sql.SQLException;

@Service
public class TransferRetryService {
    private final AccountService accountService;

    public TransferRetryService(AccountService accountService) {
        this.accountService = accountService;
    }

    // 本方法不加 @Transactional，也不要从外层已有事务中调用
    public void transferWithRetry(Long fromId, Long toId, BigDecimal amount) {
        final int maxAttempts = 3;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                // 每次调用都通过另一个 Bean 的事务代理，开启新事务
                accountService.transfer(fromId, toId, amount);
                return;
            } catch (DataAccessException ex) {
                if (!isDeadlock1213(ex) || attempt == maxAttempts) {
                    throw ex;
                }
                // 简单退避：生产中可加抖动，并限制并发重试流量
                try {
                    Thread.sleep(30L * attempt);
                } catch (InterruptedException interrupted) {
                    Thread.currentThread().interrupt();
                    throw new IllegalStateException("重试等待被中断", interrupted);
                }
            }
        }
    }

    private boolean isDeadlock1213(Throwable error) {
        for (Throwable cause = error; cause != null; cause = cause.getCause()) {
            if (cause instanceof SQLException sql && sql.getErrorCode() == 1213) {
                return true;
            }
        }
        return false;
    }
}
```

**特别注意**：

- 上例假设 `transferWithRetry` **没有在一个已经开启的外层事务内执行**；否则默认传播行为 `REQUIRED` 可能加入外层事务，导致重试并没有真正获得一个独立、全新的事务。
- 在事务代理抛出异常后，确保事务已经完成回滚，再进入下一次尝试。
- `1213` 和 `1205` 的异常在 Spring 中可能被翻译为不同 `DataAccessException` 子类；以底层 SQL 错误码和当前数据库驱动为依据处理。示例只自动重试 **1213**。
- 最多 2～3 次、适度退避；高并发时避免重试风暴。
- **必须具备业务幂等性**：一个转账请求要有唯一业务流水号/幂等键以及数据库唯一约束，避免因网络超时、客户端重复请求等原因多次扣款。死锁重试不是请求级幂等性的替代品。
- 生产金融转账还需要流水表、对账、资金精度、审计和异常补偿等设计；上例是锁与事务教学示意。

---

## 九、按 ID 加锁是不是能彻底消灭死锁？

**不能。**只有当同一组资源的获取遵守一致的全局顺序，才能防止这一类“反向取得资源”的循环等待。其他死锁来源包括：

1. 不同业务路径对多张表的访问顺序不一致。
2. 使用非唯一条件或范围条件时涉及多个索引记录、间隙锁、临键锁。
3. `UPDATE` 改变二级索引、维护外键、执行触发器等引入额外锁。
4. 其他事务没有遵守相同的顺序，或实际执行计划带来不同锁范围。
5. 长事务、慢 SQL 使锁长期持有，扩大冲突窗口。

建议组合使用：**统一访问顺序 + 主键精准查询 + 合适索引 + 短事务 + 死锁监控 + 有限完整事务重试 + 业务幂等**。

---

## 十、面试常见追问

### Q1：`FOR UPDATE` 是共享锁吗？

不是。`FOR UPDATE` 是锁定读，通常对读到的目标索引记录申请排他锁（X 锁）；`FOR SHARE` 才是共享锁（S 锁）。

### Q2：两个事务能同时对同一条记录加共享锁吗？

能。不同事务之间 S + S 兼容，但 S + X、X + X 冲突。

### Q3：两个事务都拿到共享锁后同时执行 UPDATE 会怎样？

两个事务都需要升级为 X 锁，并且都要等待对方释放 S 锁，可能形成死锁。InnoDB 通常回滚其中一个事务。

### Q4：账户 8 → 3 和 3 → 8，按 ID 升序加锁会不会把转账方向改了？

不会。无论方向如何先锁 id=3，再锁 id=8；修改时仍按 `fromId` 扣款、`toId` 加款。

### Q5：既然先锁 3 后锁 8，为什么 UPDATE 可以先改 8？

因为两个账户记录所需的 X 锁已提前获取。在这两条记录上，更新动作不需要再按相反次序等待别人释放记录 X 锁。不过仍要警惕额外索引和表的锁。

### Q6：`ORDER BY id FOR UPDATE` 就能保证加锁顺序吗？

不能无条件保证。`ORDER BY` 约束结果排序，不是通用的内部锁获取顺序承诺。更明确的演示方法是升序逐条用主键等值 `FOR UPDATE`。

### Q7：死锁后重试 SQL 还是事务？

**重试完整事务**。在原事务已结束后重新开启新的事务，从业务校验、锁定读取、余额判断到扣款/入账全部重新执行，并且要控制次数和保证幂等。

### Q8：为什么 Spring 的 `@Transactional` 自调用会有问题？

常见的 Spring 代理事务通过代理对象拦截外部方法调用；同类中 `this.transfer()` 不经过代理，可能使 `@Transactional` 失效。事务调用应通过受 Spring 管理的代理对象进入。

---

## 十一、一段可直接背诵的面试回答

> InnoDB 中共享锁 S 和共享锁 S 兼容，但共享锁 S 与排他锁 X、排他锁 X 与 X 都冲突。`SELECT ... FOR SHARE` 获取共享锁，`SELECT ... FOR UPDATE` 获取排他锁，`UPDATE` 修改记录也需要排他锁。两个事务如果先各自取得同一记录的共享锁，再同时试图升级成排他锁，就可能互相等待而死锁。
>
> 对于转账等需要修改多行记录的业务，为防止两个事务反向获取资源，我会在同一个数据库事务内把涉及的账户 ID 按升序排列，先对小 ID 执行 `SELECT ... FOR UPDATE`，再锁大 ID。锁全部到手后才根据原始 `fromId` 和 `toId` 执行扣款与入账，锁顺序不会改变资金流向。除此之外还需要控制事务时长、建好索引、监控死锁，并在发生 1213 后重新开启事务进行有限次数的完整重试，同时保证业务幂等。

## 十二、一页速记

```text
普通 SELECT（RC/RR 常见情况） → MVCC 快照读，通常不加行级 S 锁
SELECT ... FOR SHARE          → S 锁
SELECT ... FOR UPDATE         → X 锁
UPDATE                        → 修改记录需要 X 锁

不同事务同一记录：
S + S 兼容；S + X 冲突；X + X 冲突

错误锁顺序：T1 先锁 8 再等 3，T2 先锁 3 再等 8 → 死锁
正确锁顺序：T1 先锁 3 后锁 8，T2 先锁 3 后锁 8 → 避免这一类等待环
业务方向：8→3 和 3→8 原样保持；加锁顺序不等于转账方向

1213 死锁：通常回滚被选中的整个事务
1205 超时：InnoDB 默认只回滚当前语句；Spring 事务最终结果看异常处理
重试：旧事务结束 → 新事务 → 从头执行业务 → 有限次数 + 幂等
```
