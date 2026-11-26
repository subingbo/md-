# Spring 事务管理与事务传播机制

## 一、Spring 声明式事务的实现原理

Spring 声明式事务主要通过 **AOP 动态代理 + TransactionInterceptor（事务拦截器）+ PlatformTransactionManager（事务管理器）** 实现。

当一个方法添加 `@Transactional` 注解后，在默认的代理模式下，Spring 会通过 AOP 为其创建代理对象。当外部调用这个方法时，实际调用的是代理对象，代理对象负责在目标方法执行前后进行事务管理。

### 1. 事务执行的基本流程

可以将事务拦截器的执行逻辑简化为：

```
// 伪代码，仅用于理解
public Object invoke() {
    // 1. 获取或创建事务
    TransactionStatus status =
        transactionManager.getTransaction();

    try {
        // 2. 执行目标方法
        Object result = targetMethod();

        // 3. 方法正常返回，尝试提交
        transactionManager.commit(status);

        return result;

    } catch (Throwable e) {
        // 4. 根据异常类型和回滚规则决定处理方式
        if (shouldRollback(e)) {
            transactionManager.rollback(status);
        } else {
            transactionManager.commit(status);
        }

        throw e;
    }
}
```

实际 Spring 实现比这个伪代码复杂，还需要处理事务挂起、保存点、事务同步等情况，但核心思路一致。

### 2. Spring 默认的事务回滚规则

| 异常类型                      | 默认行为 |
| ----------------------------- | -------- |
| RuntimeException              | 回滚     |
| Error                         | 回滚     |
| Checked Exception（受检异常） | 不回滚   |

如果希望受检异常也触发回滚，可以使用：

```
@Transactional(rollbackFor = Exception.class)
public void createOrder() throws Exception {
    orderMapper.insert();
    throw new Exception("创建订单失败");
}
```

也可以使用 `noRollbackFor` 指定某些异常不触发自动回滚。

注意：不回滚不代表异常不会继续向外传播，只是事务拦截器不会根据该异常执行回滚操作。

## 二、Java 异常捕获与 Spring 事务回滚的区别

这是理解 Spring 事务最重要的知识点之一。

**Java 的异常处理机制和 Spring 的事务管理机制是两个不同的机制。**

| 对比         | Java 异常处理  | Spring 事务管理                |
| ------------ | -------------- | ------------------------------ |
| 核心机制     | try-catch      | AOP 事务代理                   |
| 主要作用     | 控制异常传播   | 控制事务提交与回滚             |
| 关注内容     | 异常是否被捕获 | 事务是否被标记为 rollback-only |
| 能否相互替代 | 不能           | 不能                           |

### 1. 同一个事务方法内部捕获异常：默认不会回滚

```
@Transactional
public void createOrder() {
    orderMapper.insert();

    try {
        paymentMapper.insert();
        throw new RuntimeException("支付失败");
    } catch (Exception e) {
        System.out.println("支付失败");
    }

    logMapper.insert();
}
```

执行过程：

1. Spring 事务代理开启事务 T1。
2. 执行 `createOrder()`。
3. 插入订单记录。
4. 插入支付记录。
5. 抛出 RuntimeException。
6. 异常被当前方法内部的 catch 捕获。
7. 继续执行日志插入。
8. `createOrder()` 正常返回。
9. Spring 事务代理没有观察到异常，尝试提交事务。

**结果：正常提交。**

原因是异常没有逃出当前事务方法，因此事务拦截器没有根据这个异常触发默认回滚规则。

注意：如果代码显式调用 `setRollbackOnly()`，或者其他操作已经将事务标记为 rollback-only，即使异常被捕获，事务仍然可能回滚。

### 2. 内层事务方法抛出异常，外层捕获：仍然可能回滚

这是最容易混淆的情况。

假设存在两个 Service：

```
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        orderMapper.insert();

        try {
            paymentService.pay();
        } catch (Exception e) {
            System.out.println("支付失败");
        }
    }
}
@Service
public class PaymentService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void pay() {
        paymentMapper.insert();

        throw new RuntimeException("支付失败");
    }
}
```

这里有两个事务方法：

- `createOrder()`：由 OrderService 的事务代理拦截。
- `pay()`：由 PaymentService 的事务代理拦截。

由于两个 Service 是不同的 Bean，正常通过 Spring 注入的代理对象调用，因此两个方法都能经过各自的事务拦截器。

但是，两个事务方法不代表两个物理事务。

因为 `pay()` 使用 REQUIRED，而当前已经存在事务 T1，所以它会加入 T1。

#### 实际执行流程

**第一步：OrderService 事务代理开启事务**

外部调用 `createOrder()`，首先经过 OrderService 的事务代理。

Spring 开启物理事务 T1，然后执行目标方法。

**第二步：调用 PaymentService**

`createOrder()` 调用：

```
paymentService.pay();
```

由于调用的是 PaymentService 的代理对象，因此会经过 PaymentService 的事务拦截器。

Spring 检测到当前已经存在事务 T1。

因为 `pay()` 使用 REQUIRED，所以加入 T1，不创建新的物理事务。

**第三步：pay() 抛出异常**

```
paymentMapper.insert();
throw new RuntimeException("支付失败");
```

异常离开 `pay()` 目标方法，首先到达 PaymentService 的事务代理。

事务代理发现 RuntimeException 符合默认回滚规则，于是调用事务管理器的 rollback。

但是，`pay()` 并没有创建独立事务，它只是参与了外层事务 T1。

因此，在 Spring 默认配置下，内层回滚会将共享事务 T1 标记为：

```
rollbackOnly = true;
```

这里需要注意：

**此时通常不是立即执行物理事务回滚，而是先标记共享事务为只能回滚。**

随后，事务代理重新抛出异常。

**第四步：外层 catch 捕获异常**

异常重新传播到 `createOrder()`：

```
try {
    paymentService.pay();
} catch (Exception e) {
    System.out.println("支付失败");
}
```

外层成功捕获异常，Java 程序可以继续执行。

但是，catch 只能控制异常传播，不能自动清除 Spring 已经设置的 rollback-only 标记。

**第五步：外层事务代理尝试提交**

`createOrder()` 正常结束，OrderService 的事务代理准备提交 T1。

但是，T1 已经被标记为 rollback-only。

因此 Spring 最终执行物理回滚，而不是正常提交。

通常还会向调用方抛出：

```
UnexpectedRollbackException
```

**最终结果：订单插入和支付插入都回滚。**

### 3. 两种情况的核心区别

| 对比                     | 当前方法内部 catch | 内层 REQUIRED 方法抛异常，外层 catch |
| ------------------------ | ------------------ | ------------------------------------ |
| 物理事务                 | 一个               | 一个                                 |
| 事务代理                 | 一个               | 两个                                 |
| 异常是否离开内层目标方法 | 不涉及             | 是                                   |
| 是否到达内层事务代理     | 不涉及             | 是                                   |
| 是否标记 rollback-only   | 默认不会           | 默认会                               |
| 最终结果                 | 正常提交           | 最终回滚                             |

核心结论：

**同一个物理事务可以被多个事务代理管理。异常被外层捕获，不代表之前内层事务代理设置的 rollback-only 标记会被清除。**

## 三、事务代理与物理事务的区别

这是理解上述过程的关键。

### 1. 什么是事务代理？

事务代理是 Spring AOP 创建的代理对象。

它负责拦截事务方法调用，执行事务开启、提交、回滚等管理逻辑。

例如：

```
@Transactional
public void createOrder() {
}
```

外部调用时，通常经过：

```
调用方
  |
  v
OrderService 代理对象
  |
  v
TransactionInterceptor
  |
  v
createOrder() 目标方法
```

### 2. 什么是物理事务？

物理事务指数据库层面真正开启的事务。

例如使用 JDBC 时，底层通常通过数据库连接的事务状态来管理事务：

```
connection.setAutoCommit(false);

// 执行 SQL

connection.commit();
```

多个事务方法可以共享同一个物理事务。

例如：

```
OrderService 事务代理
        |
        v
    开启 T1
        |
        v
    createOrder()
        |
        v
PaymentService 事务代理
        |
        v
    加入 T1
        |
        v
      pay()
```

这里有两个事务代理，但只有一个物理事务 T1。

**记忆：事务代理的数量不等于物理事务的数量。**

## 四、Spring 事务传播机制

事务传播机制用于控制一个事务方法调用另一个事务方法时，两个方法之间如何共享或创建事务。

Spring 常见的七种传播行为：

| 传播行为      | 当前存在事务                                   | 当前不存在事务 |
| ------------- | ---------------------------------------------- | -------------- |
| REQUIRED      | 加入当前事务                                   | 创建新事务     |
| REQUIRES_NEW  | 挂起当前事务，创建新事务                       | 创建新事务     |
| SUPPORTS      | 加入当前事务                                   | 不使用事务     |
| MANDATORY     | 加入当前事务                                   | 抛出异常       |
| NOT_SUPPORTED | 挂起当前事务，以非事务方式执行                 | 非事务方式执行 |
| NEVER         | 抛出异常                                       | 非事务方式执行 |
| NESTED        | 在当前事务中创建嵌套事务（通常通过保存点实现） | 创建新事务     |

### 1. REQUIRED：默认传播行为

```
@Transactional(propagation = Propagation.REQUIRED)
public void pay() {
}
```

特点：

- 当前存在事务：加入当前事务。
- 当前不存在事务：创建新事务。

例如：

```
@Transactional
public void createOrder() {
    orderMapper.insert();
    paymentService.pay();
}
```

如果 `pay()` 也使用 REQUIRED，那么两个方法共享同一个物理事务。

任何参与者按照默认规则将共享事务标记为 rollback-only，都可能导致整个事务最终回滚。

### 2. REQUIRES_NEW：创建独立事务

```
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void pay() {
}
```

特点：

- 当前存在事务：挂起当前事务，创建一个新的物理事务。
- 当前不存在事务：直接创建新事务。

例如：

```
@Transactional
public void createOrder() {
    orderMapper.insert();

    try {
        paymentService.pay();
    } catch (Exception e) {
        System.out.println("支付失败");
    }

    logService.saveLog();
}
```

如果 `pay()` 使用 REQUIRES_NEW：

```
外层事务 T1
    |
    | 插入订单
    |
    | 挂起 T1
    v
内层事务 T2
    |
    | 插入支付记录
    |
    | 抛出 RuntimeException
    |
    v
T2 回滚
    |
    | 恢复 T1
    v
外层继续执行
    |
    | 保存日志
    |
    v
T1 提交
```

**结果：内层支付事务回滚，不必然影响外层订单事务提交。**

原因是 T1 和 T2 是两个独立的物理事务。

但如果内层异常继续向外传播，外层事务代理也可能因为收到 RuntimeException 而回滚 T1。

因此，是否影响外层事务，还取决于异常是否被外层处理，以及外层自身的回滚规则。

### 3. NESTED：嵌套事务

```
@Transactional(propagation = Propagation.NESTED)
public void pay() {
}
```

特点：

- 当前存在事务：在当前物理事务中创建保存点。
- 当前不存在事务：创建新事务。

假设外层已经开启事务 T1：

```
物理事务 T1 开始
    |
    | 插入订单
    |
    | 创建保存点 Savepoint
    |
    v
执行 pay()
    |
    | 插入支付记录
    |
    | 发生异常
    |
    v
回滚到 Savepoint
    |
    | 支付记录回滚
    | 订单记录仍然保留
    |
    v
外层继续执行
    |
    | 插入日志
    |
    v
提交物理事务 T1
```

如果保存点回滚成功，外层事务可以继续执行并提交。

NESTED 与 REQUIRES_NEW 的核心区别：

| 对比               | NESTED       | REQUIRES_NEW         |
| ------------------ | ------------ | -------------------- |
| 物理事务数量       | 通常共享一个 | 两个独立事务         |
| 实现方式           | 保存点       | 挂起外层，创建新事务 |
| 内层回滚           | 回滚到保存点 | 回滚整个内层事务     |
| 外层能否继续       | 可以         | 可以                 |
| 内外层是否独立提交 | 不独立       | 独立                 |

注意：NESTED 依赖事务管理器和底层资源对保存点的支持，并非所有事务管理器都支持。它不等于创建完全独立的物理事务。

## 五、Spring 事务失效的常见情况

### 1. 同一个类内部通过 this 调用事务方法

```
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        this.pay();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void pay() {
        paymentMapper.insert();
    }
}
```

这里的 `this.pay()` 是直接调用当前对象的方法，没有经过 Spring AOP 代理。

因此，`pay()` 上的 `@Transactional(REQUIRES_NEW)` 不会被代理拦截，无法按预期创建独立事务。

如果 `createOrder()` 已经开启事务，`pay()` 的数据库操作通常仍然在外层事务中执行。

解决方式之一是通过另一个 Spring Bean 调用事务方法：

```
paymentService.pay();
```

这样可以经过代理对象。

### 2. 方法没有经过 Spring 代理

例如：

- 手动通过 `new` 创建对象。
- 调用同一个类中的方法，绕过代理。
- 对象没有交由 Spring 容器管理。

这些情况都可能导致 `@Transactional` 不生效。

### 3. 异常被内部捕获

如果异常在事务目标方法内部被捕获，没有继续抛出，事务代理通常会认为方法正常执行。

如果业务确实需要回滚，可以显式设置：

```
@Transactional
public void createOrder() {
    try {
        orderMapper.insert();
        throw new RuntimeException();
    } catch (Exception e) {
        TransactionAspectSupport
            .currentTransactionStatus()
            .setRollbackOnly();
    }
}
```

这样即使异常被捕获，事务最终仍然会回滚。

## 六、原始面试题总结

题目：`OrderService.createOrder()` 和 `PaymentService.pay()` 都使用 `@Transactional`，其中 `pay()` 使用 REQUIRED 并抛出 RuntimeException，外层通过 try-catch 捕获异常。

正确选项：**A、B、C、E（按照 Spring 默认事务配置及正常代理调用的前提）。**

| 选项 | 正误               | 理由                                                         |
| ---- | ------------------ | ------------------------------------------------------------ |
| A    | 正确               | REQUIRED 有事务则加入，没有则创建                            |
| B    | 正确（默认配置下） | 内层代理将共享事务标记为 rollback-only，外层捕获异常也不能清除标记 |
| C    | 正确               | REQUIRES_NEW 挂起外层事务，创建独立物理事务                  |
| D    | 错误               | NESTED 通常通过保存点实现，不会创建完全独立的物理事务        |
| E    | 正确               | REQUIRES_NEW 的内层回滚不必然影响外层事务提交                |

需要注意 B 中的「一定」有隐含前提：正常经过 Spring 事务代理、使用同一个事务管理器及共享事务资源，并且采用默认回滚规则和默认参与事务失败处理配置。严格来说，不能脱离这些前提说任何情况下都一定回滚。

## 七、面试速记

建议重点记住以下五句话：

1. **Spring 声明式事务通过 AOP 代理和 TransactionInterceptor 实现。**
2. **事务代理数量不等于物理事务数量，多个事务方法可以共享同一个物理事务。**
3. **Java catch 只能控制异常传播，不能自动清除 Spring 的 rollback-only 标记。**
4. **REQUIRED 共享事务，REQUIRES_NEW 创建独立事务，NESTED 通常通过保存点实现嵌套回滚。**
5. **Spring 默认对 RuntimeException 和 Error 回滚，对 Checked Exception 不回滚。**

最重要的理解：

```
Java 异常处理：
决定异常是否继续向外传播。

Spring 事务代理：
在事务方法调用边界处理异常和事务状态。

rollback-only：
表示当前共享事务只能回滚，不能正常提交。
```

这三个概念是理解 Spring 事务传播机制的基础。