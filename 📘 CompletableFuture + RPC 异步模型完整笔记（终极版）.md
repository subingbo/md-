# 📘 CompletableFuture + RPC 异步模型完整笔记（终极版）

------

# 一、核心问题背景

## 🎯 要解决什么问题？

```text
把“callback 异步模型” → 转换成 “CompletableFuture 模型”
```

------

## 🧠 两种异步模型

### ❌ Callback 模型

```java
client.getUser(id, callback);

callback.onSuccess(result);
callback.onFailure(e);
```

特点：

- 被动回调
- 不易组合
- 容易嵌套（callback hell）

------

### ✅ CompletableFuture 模型

```java
future.thenApply(...)
      .thenCompose(...)
      .thenAccept(...)
```

特点：

- 链式调用
- 可组合
- 更清晰

------

# 二、核心代码解析

```java
public static <T> CompletableFuture<T> toCompletableFuture(
    final OctoThriftCallback<?,T> callback,
    ThriftAsyncCall thriftCall) {

    CompletableFuture<T> resultFuture = new CompletableFuture<>();

    callback.addObserver(new OctoObserver<T>() {
        @Override
        public void onSuccess(T t) {
            resultFuture.complete(t);
        }

        @Override
        public void onFailure(Throwable throwable) {
            resultFuture.completeExceptionally(throwable);
        }
    });

    if (thriftCall != null) {
        try {
            thriftCall.invoke();
        } catch (TException e) {
            resultFuture.completeExceptionally(e);
        }
    }

    return resultFuture;
}
```

------

## ✅ 本质一句话

```text
监听 callback → 把结果塞进 CompletableFuture
```

------

# 三、callback.addObserver 是什么？

## ❌ 不是重写 callback

## ✅ 是注册监听器（Observer 模式）

```text
callback = 事件源
observer = 监听者
```

------

## 执行流程

```text
RPC返回
   ↓
callback.onSuccess
   ↓
observer.onSuccess
   ↓
future.complete
```

------

# 四、t 是什么？

```java
public void onSuccess(T t)
```

👉 本质：

```text
t = RPC返回结果（业务数据）
```

------

# 五、thriftCall.invoke() 为什么能触发 callback？

## ❌ 错误理解

```text
invoke 返回值 → callback
```

------

## ✅ 正确理解

```text
invoke = 发起 RPC 请求
callback = 结果回来时被调用
```

------

## 关键

```java
client.getUser(userId, callback);
```

👉 callback 被交给框架

------

# 六、complete(t) 是什么？

## ✅ 本质

```text
给 future 设置结果 + 触发所有订阅者
```

------

## 示例

```java
future.complete("hello");
```

------

## 🔥 核心理解

```text
complete = 发布结果
thenXXX = 订阅结果
```

------

# 七、thenApply / thenAccept / thenCompose

------

## 1️⃣ thenApply

```java
future.thenApply(user -> user.getName());
T → R（同步转换）
```

------

## 2️⃣ thenAccept

```java
future.thenAccept(user -> log(user));
T → void（消费，不返回）
```

------

## 3️⃣ thenCompose（重点）

```java
future.thenCompose(user -> getOrderAsync(user.getId()));
T → Future<R>
```

------

## ❌ 错误

```java
thenApply(user -> getOrderAsync(...))
Future<Future<T>>（套娃）
```

------

## 🔥 口诀

```text
返回值是 Future → 用 thenCompose
否则 → thenApply
```

------

# 八、future 可以多次订阅吗？

## ✅ 可以

```java
f1.thenAccept(A);
f1.thenAccept(B);
f1.thenAccept(C);
```

------

## 👉 效果

```text
一个结果 → 多个消费者
```

------

# 九、分支 vs 链式

------

## 分支

```java
f1.thenAccept(A);
f1.thenAccept(B);
多个独立逻辑（解耦）
```

------

## 链式

```java
f1.thenApply(A)
  .thenApply(B)
  .thenAccept(C);
有执行顺序（依赖）
```

------

## 🔥 核心区别

```text
分支 = 多消费者
链式 = 有依赖
```

------

# 十、线程模型

------

## ❌ 默认不是多线程

```java
f1.thenAccept(...)
```

👉 在：

```text
调用 complete 的线程执行
```

------

## ✅ 使用异步

```java
f1.thenAcceptAsync(...)
```

------

## 对比

| 方法           | 线程     |
| -------------- | -------- |
| thenApply      | 当前线程 |
| thenApplyAsync | 线程池   |

------

# 十一、执行顺序

------

## ❌ 多个订阅

```java
f1.thenAccept(A);
f1.thenAccept(B);
```

👉 ❌ 无顺序保证

------

## ✅ 链式

```java
f1.thenApply(A)
  .thenApply(B);
```

👉 ✅ 有顺序

------

# 十二、为什么先 addObserver 再 invoke？

------

## ❌ 错误

```java
invoke();
addObserver();
```

👉 ❌ 可能丢结果

------

## ✅ 正确

```java
addObserver();
invoke();
```

------

## 原因

```text
RPC可能瞬间返回
```

------

# 十三、CompletableFuture 内部原理

------

## 结构

```java
class CompletableFuture {
    volatile Object result;
    volatile Completion stack;
}
```

------

## Completion

```java
class Completion {
    Completion next;
    void run();
}
```

------

## thenXXX

```text
CAS 插入链表
```

------

## complete

```text
CAS 设置 result
遍历执行所有回调
```

------

# 十四、CAS 的作用

------

## 解决问题

- 多线程注册回调
- 多线程 complete

------

## 优势

```text
无锁 / 高性能 / 非阻塞
```

------

# 十五、OpenFeign vs Thrift

------

## Thrift

```text
异步 + callback
```

------

## OpenFeign

```text
同步 return
```

------

## 异步实现

```java
CompletableFuture.supplyAsync(() -> feign.call());
```

------

# 十六、完整数据流

```text
RPC → callback → observer → future → thenXXX
```

------

# 🔥 最终总结（必须记住）

```text
1️⃣ callback = 事件源
2️⃣ observer = 监听器
3️⃣ future = 结果容器
4️⃣ complete = 触发执行
5️⃣ thenXXX = 注册逻辑
```

------

# 🚀 你已掌握能力

- 异步模型（callback vs future）
- CompletableFuture API
- 异步链式编排
- CAS 并发思想
- RPC 异步机制

------

# 🚀 下一步建议

```text
1️⃣ CompletableFuture.allOf（并发聚合）
2️⃣ thenCombine（结果合并）
3️⃣ 线程池调优
4️⃣ Netty Promise/Future
```

------